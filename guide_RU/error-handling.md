---
author: Nexu
---

# Обработка ошибок

> **Ключевая идея:** В традиционной программе необработанная ошибка приводит к краху процесса. В агентной системе модель *является* обработчиком ошибок — если Вы ясно покажете ошибку, модель сможет адаптироваться, повторить попытку или попробовать альтернативный подход. Ваша задача — классифицировать ошибки, применять правильную стратегию восстановления и эскалировать к человеку только когда автоматическое восстановление не помогает.

## Классификация ошибок

Не все ошибки одинаковы. Стратегия восстановления зависит от класса ошибки:

| Класс | Описание | Восстановление |
|-------|----------|----------------|
| **Транзитная** | Таймаут сети, лимит, временная недоступность | Повтор с отступом |
| **Постоянная** | Файл не найден, доступ запрещён, некорректный ввод | Сообщить модели, попробовать альтернативу |
| **Модельная** | Некорректный вызов инструмента, сгаллюцинированное имя функции, невалидный JSON | Пере-промпт с исправлением |
| **Ресурсная** | Не хватает памяти, диск полон, превышен бюджет токенов | Контрольная точка и эскалация |

```python
from enum import Enum

class ErrorClass(Enum):
    TRANSIENT = "transient"
    PERMANENT = "permanent"
    MODEL = "model"
    RESOURCE = "resource"

def classify_error(error: Exception, context: dict | None = None) -> ErrorClass:
    """Classify an error to determine the recovery strategy."""
    error_type = type(error).__name__
    message = str(error).lower()

    # Transient: network and rate limit errors
    transient_signals = [
        "timeout", "connection", "rate limit", "429", "503",
        "502", "504", "temporary", "retry",
    ]
    if any(signal in message for signal in transient_signals):
        return ErrorClass.TRANSIENT

    # Model errors: bad tool calls, JSON parsing failures
    model_signals = [
        "unknown tool", "invalid json", "missing required",
        "unexpected argument", "malformed",
    ]
    if any(signal in message for signal in model_signals):
        return ErrorClass.MODEL

    # Resource errors: system-level exhaustion
    resource_signals = [
        "out of memory", "disk full", "no space left",
        "token limit", "context length exceeded",
    ]
    if any(signal in message for signal in resource_signals):
        return ErrorClass.RESOURCE

    # Default: permanent (file not found, permission denied, etc.)
    return ErrorClass.PERMANENT
```

## Повтор с экспоненциальным отступом

Транзитные ошибки следует повторять автоматически. Ключ — экспоненциальный отступ с джиттером; без него множественные повторы могут вызвать громоподобную стаду при восстановлении сервиса:

```python
import time
import random
import functools
from typing import TypeVar, Callable, Any

T = TypeVar("T")

class RetryExhausted(Exception):
    """All retry attempts failed."""
    def __init__(self, last_error: Exception, attempts: int):
        self.last_error = last_error
        self.attempts = attempts
        super().__init__(f"Failed after {attempts} attempts: {last_error}")

def retry(
    max_attempts: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    retryable: tuple[type[Exception], ...] = (Exception,),
) -> Callable:
    """Decorator: retry with exponential backoff and jitter."""

    def decorator(func: Callable[..., T]) -> Callable[..., T]:
        @functools.wraps(func)
        def wrapper(*args: Any, **kwargs: Any) -> T:
            last_error = None
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except retryable as e:
                    last_error = e
                    if classify_error(e) != ErrorClass.TRANSIENT:
                        raise  # Don't retry non-transient errors
                    if attempt < max_attempts - 1:
                        delay = min(
                            base_delay * (2 ** attempt) + random.uniform(0, 1),
                            max_delay,
                        )
                        time.sleep(delay)
            raise RetryExhausted(last_error, max_attempts)
        return wrapper
    return decorator

# Usage
@retry(max_attempts=3, base_delay=2.0)
def call_llm(messages: list, tools: list) -> dict:
    """Make an LLM API call with automatic retry on transient failures."""
    response = httpx.post(
        "https://api.openai.com/v1/chat/completions",
        json={"messages": messages, "tools": tools, "model": "gpt-4o"},
        headers={"Authorization": f"Bearer {API_KEY}"},
        timeout=30.0,
    )
    response.raise_for_status()
    return response.json()
```

Математика: при `base_delay=2.0` повторы происходят примерно через ~2с, ~5с, ~9с. Джиттер предотвращает синхронизированные повторы нескольких агентов, обращющихся к одному API.

## Корректная деградация

Когда инструмент терпит окончательную неудачу, модель должна попробовать альтернативы, а не сдаваться. Харнес обеспечивает это, возвращая чёткие сообщения об ошибках:

```python
class ToolExecutor:
    """Execute tools with graceful degradation."""

    def __init__(self):
        self.fallbacks: dict[str, list[str]] = {
            "web_search": ["web_fetch"],
            "read_file": ["shell_exec"],      # fallback: cat via shell
            "git_push": ["git_diff"],          # fallback: show diff instead
        }

    def execute(self, tool_name: str, arguments: dict) -> str:
        """Execute a tool, falling back to alternatives on failure."""
        try:
            result = self._dispatch(tool_name, arguments)
            return result
        except Exception as primary_error:
            error_class = classify_error(primary_error)

            if error_class == ErrorClass.TRANSIENT:
                # Let the retry decorator handle transient errors
                raise

            # Try fallbacks for permanent errors
            fallback_chain = self.fallbacks.get(tool_name, [])
            for fallback_name in fallback_chain:
                try:
                    result = self._dispatch(fallback_name, arguments)
                    return f"[Used fallback: {fallback_name}]\n{result}"
                except Exception:
                    continue

            # All fallbacks failed — return error to the model
            return (
                f"Error in {tool_name}: {primary_error}\n"
                f"Tried fallbacks: {fallback_chain or 'none available'}\n"
                f"All failed. Consider an alternative approach."
            )

    def _dispatch(self, name: str, arguments: dict) -> str:
        """Dispatch to the actual tool handler."""
        handler = tool_registry.get_handler(name)
        if not handler:
            raise ValueError(f"Unknown tool: {name}")
        return str(handler(**arguments))
```

Ключевое архитектурное решение: **всегда возвращайте ошибки как результаты инструментов, никогда не бросайте исключения через агентный цикл**. Модель должна видеть ошибку, чтобы адаптироваться. Поглощённое исключение приводит к молчаливой неудаче или сгаллюцинированному успеху.

## Эскалация с участием человека

Некоторые ошибки требуют суждения человека. Паттерн эскалации:

```python
class EscalationLevel(Enum):
    AUTO = "auto"         # Fully automated recovery
    INFORM = "inform"     # Recover automatically, notify human
    CONFIRM = "confirm"   # Ask human before proceeding
    BLOCK = "block"       # Stop and wait for human input

def determine_escalation(
    error: Exception,
    error_class: ErrorClass,
    attempt: int,
    context: dict,
) -> EscalationLevel:
    """Determine how much human involvement is needed."""
    # Resource errors always block — human needs to provision more
    if error_class == ErrorClass.RESOURCE:
        return EscalationLevel.BLOCK

    # Repeated transient failures suggest a real outage
    if error_class == ErrorClass.TRANSIENT and attempt >= 3:
        return EscalationLevel.INFORM

    # Model errors on destructive operations require confirmation
    if error_class == ErrorClass.MODEL and context.get("destructive"):
        return EscalationLevel.CONFIRM

    # Permanent errors on critical paths should inform
    if error_class == ErrorClass.PERMANENT and context.get("critical"):
        return EscalationLevel.INFORM

    return EscalationLevel.AUTO

def escalate(level: EscalationLevel, message: str) -> str | None:
    """Execute the escalation action. Returns human response if blocking."""
    if level == EscalationLevel.AUTO:
        return None

    if level == EscalationLevel.INFORM:
        notify_human(f"⚠️ Agent issue (auto-resolved): {message}")
        return None

    if level == EscalationLevel.CONFIRM:
        return ask_human(f"⚠️ Agent needs confirmation: {message}\nProceed? (yes/no)")

    if level == EscalationLevel.BLOCK:
        return ask_human(f"🛑 Agent blocked: {message}\nPlease resolve and reply.")
```

## Контрольные точки/возобновление для длительных задач

Длительные задачи (20+ ходов) уязвимы к сбоям на полпути. Контрольные точки позволяют агенту возобновить работу без потери прогресса:

```python
import json
import os
from datetime import datetime

class Checkpoint:
    """Save and restore agent progress for long-running tasks."""

    def __init__(self, checkpoint_dir: str = "/tmp/agent-checkpoints"):
        self.checkpoint_dir = checkpoint_dir
        os.makedirs(checkpoint_dir, exist_ok=True)

    def save(self, task_id: str, state: dict):
        """Save current progress."""
        checkpoint = {
            "task_id": task_id,
            "timestamp": datetime.now().isoformat(),
            "state": state,
        }
        path = os.path.join(self.checkpoint_dir, f"{task_id}.json")
        # Write atomically (write to temp, then rename)
        tmp_path = path + ".tmp"
        with open(tmp_path, "w") as f:
            json.dump(checkpoint, f, indent=2)
        os.rename(tmp_path, path)

    def load(self, task_id: str) -> dict | None:
        """Load the last checkpoint for a task."""
        path = os.path.join(self.checkpoint_dir, f"{task_id}.json")
        if not os.path.exists(path):
            return None
        with open(path) as f:
            return json.load(f)["state"]

    def clear(self, task_id: str):
        """Remove checkpoint after task completes."""
        path = os.path.join(self.checkpoint_dir, f"{task_id}.json")
        if os.path.exists(path):
            os.unlink(path)

# Usage in the agentic loop
checkpoint = Checkpoint()

def agentic_loop_with_checkpoint(task_id: str, messages: list, tools: list):
    """Agentic loop that can resume from a checkpoint."""
    # Try to resume from checkpoint
    saved_state = checkpoint.load(task_id)
    if saved_state:
        messages = saved_state["messages"]
        turn = saved_state["turn"]
        print(f"Resumed from checkpoint at turn {turn}")
    else:
        turn = 0

    for turn in range(turn, 50):
        try:
            response = call_llm(messages, tools)
            assistant_msg = response["choices"][0]["message"]
            messages.append(assistant_msg)

            if not assistant_msg.get("tool_calls"):
                checkpoint.clear(task_id)
                return assistant_msg["content"]

            for tc in assistant_msg["tool_calls"]:
                result = execute_tool(tc)
                messages.append({
                    "role": "tool",
                    "tool_call_id": tc["id"],
                    "content": result,
                })

            # Checkpoint every 5 turns
            if turn % 5 == 0:
                checkpoint.save(task_id, {
                    "messages": messages,
                    "turn": turn,
                })

        except RetryExhausted as e:
            # Save progress and escalate
            checkpoint.save(task_id, {"messages": messages, "turn": turn})
            escalate(
                EscalationLevel.BLOCK,
                f"Task {task_id} failed at turn {turn}: {e}",
            )
            break
```

Паттерн атомарной записи (запись в `.tmp` → `rename`) предотвращает повреждение контрольных точек, если процесс падает во время записи.

## Частые ошибки

- **Повтор постоянных ошибок** — Повтор «файл не найден» 3 раза не заставит файл появиться. Сначала классифицируйте, затем выбирайте стратегию.
- **Молчаливое поглощение ошибок** — Если инструмент возвращает пустую строку при ошибке, модель думает, что всё прошло успешно. Всегда включайте тип и сообщение ошибки в результат инструмента.
- **Нет джиттера при отступе** — Экспоненциальный отступ без джиттера создаёт синхронизированные штормы повторов. Всегда добавляйте случайность.
- **Контрольная точка каждый ход** — Запись контрольной точки после каждого вызова инструмента добавляет задержку и дисковый ввод-вывод. Каждые 3–5 ходов — правильный баланс между детализацией восстановления и производительностью.
- **Слишком ревностная эскалация** — Запрос помощи человека при каждой транзитной ошибке подрывает доверие. Восстанавливайте автоматически; эскалируйте только когда автоматическое восстановление исчерпано.

## Дополнительные материалы

- [AWS: Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) — исчерпывающее руководство по стратегиям повторов
- [Anthropic: Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — паттерны восстановления ошибок в production-агентах
- [Microsoft: Retry Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry) — cloud-паттерны проектирования для обработки транзитных сбоев
