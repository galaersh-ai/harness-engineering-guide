---
author: Nexu
---

# Ваш первый харнес

> **Ключевая идея:** Харнес — это просто цикл: вызов модели, выполнение вызовов инструментов, возврат результатов, повтор. Вы можете построить работающий менее чем из 50 строк Python. Понимание этого цикла развеивает мистику любого фреймворка для агентов.

Большинство туториалов для агентов начинают с фреймворка — LangChain, CrewAI, AutoGen. Но фреймворки скрывают механизм. Построение харнеса с нуля учит Вас точно тому, что происходит: агентный цикл, сборка контекста и процесс принятия решений моделью. Когда Вы это поймёте, каждый фреймворк станет прозрачным.

## Полный харнес

Вот полностью рабочий харнес с двумя инструментами (чтение файла и запись файла). Скопируйте и запустите.

### Предварительные требования

```bash
pip install openai
export OPENAI_API_KEY="***"
```

### Код

```python
#!/usr/bin/env python3
"""A complete agent harness in ~50 lines. Run: python harness.py"""

import json
import os
from openai import OpenAI

client = OpenAI()
MODEL = "gpt-4o-mini"  # Cheap and fast for learning
MAX_TURNS = 15

# --- System prompt ---
SYSTEM = """You are a helpful file assistant. You can read and write files.
When asked to work with files, use the tools provided.
Always confirm what you did after completing a task."""

# --- Tool definitions ---
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "Read the contents of a file at the given path",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "Path to the file"}
                },
                "required": ["path"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "write_file",
            "description": "Write content to a file (creates or overwrites)",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "Path to the file"},
                    "content": {"type": "string", "description": "Content to write"}
                },
                "required": ["path", "content"]
            }
        }
    }
]

# --- Tool execution ---
def execute_tool(name: str, args: dict) -> str:
    try:
        if name == "read_file":
            with open(args["path"], "r") as f:
                return f.read()
        elif name == "write_file":
            os.makedirs(os.path.dirname(args["path"]) or ".", exist_ok=True)
            with open(args["path"], "w") as f:
                f.write(args["content"])
            return f"Wrote {len(args['content'])} chars to {args['path']}"
        else:
            return f"Error: Unknown tool '{name}'"
    except Exception as e:
        return f"Error: {e}"

# --- The tool loop ---
def run(user_message: str) -> str:
    messages = [
        {"role": "system", "content": SYSTEM},
        {"role": "user", "content": user_message}
    ]

    for turn in range(MAX_TURNS):
        response = client.chat.completions.create(
            model=MODEL, messages=messages, tools=TOOLS
        )
        msg = response.choices[0].message
        messages.append(msg)

        # No tool calls → model is done
        if not msg.tool_calls:
            return msg.content

        # Execute each tool call
        for tc in msg.tool_calls:
            args = json.loads(tc.function.arguments)
            print(f"  🔧 {tc.function.name}({args})")
            result = execute_tool(tc.function.name, args)
            messages.append({
                "role": "tool",
                "tool_call_id": tc.id,
                "content": result
            })

    return "Max turns reached."

# --- Main ---
if __name__ == "__main__":
    print("🤖 File Agent (type 'quit' to exit)")
    while True:
        user_input = input("\nYou: ").strip()
        if user_input.lower() in ("quit", "exit"):
            break
        response = run(user_input)
        print(f"\nAgent: {response}")
```

### Попробуйте

```bash
python harness.py
```

```
🤖 File Agent (type 'quit' to exit)

You: Create a file called hello.txt with a haiku about programming

  🔧 write_file({'path': 'hello.txt', 'content': 'Semicolons fall\nLike rain upon the server\nCompile error: none'})

Agent: I've created hello.txt with a programming haiku!

You: Read it back to me

  🔧 read_file({'path': 'hello.txt'})

Agent: Here's the content of hello.txt:
"Semicolons fall / Like rain upon the server / Compile error: none"
```

## Структура харнеса

Весь харнес состоит из четырёх компонентов:

```
┌────────────────────────────────┐
│         System Prompt          │  ← Who the agent is
├────────────────────────────────┤
│        Tool Definitions        │  ← What it can do (JSON schema)
├────────────────────────────────┤
│        Tool Execution          │  ← How tools actually run
├────────────────────────────────┤
│          Tool Loop             │  ← The cycle: think → act → observe
└────────────────────────────────┘
```

**Системный промпт**: Задаёт личность и ограничения агента. Это самая дешёвая, самая высокоэффективная деталь — изменение одной фразы здесь может полностью изменить поведение.

**Определения инструментов**: JSON-схемы, которые модель читает, чтобы понять, какие инструменты существуют. Модель никогда не видит Ваш Python-код — только описания и схемы параметров.

**Выполнение инструментов**: Ваш код, который фактически выполняет действия. Модель выводит структурированный JSON; Вы его парсите и делаете реальную работу.

**Цикл инструментов**: Оркестратор. Вызов модели, проверка вызовов инструментов, их выполнение, возврат результатов. Повтор, пока модель не ответит простым текстом.

## Добавление третьего инструмента

Хотите добавить shell-команды? Просто добавьте определение инструмента и обработчик:

```python
# Add to TOOLS list:
{
    "type": "function",
    "function": {
        "name": "run_shell",
        "description": "Run a shell command and return stdout/stderr",
        "parameters": {
            "type": "object",
            "properties": {
                "command": {"type": "string", "description": "Shell command to run"}
            },
            "required": ["command"]
        }
    }
}

# Add to execute_tool():
elif name == "run_shell":
    import subprocess
    r = subprocess.run(args["command"], shell=True, capture_output=True, text=True, timeout=30)
    return r.stdout + r.stderr
```

Цикл не меняется. Модель автоматически обнаруживает и использует новый инструмент.

## Замена моделей

Харнес агностичен к модели. Переключитесь на Claude от Anthropic, изменив клиент:

```python
from anthropic import Anthropic

client = Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=4096,
    system=SYSTEM,
    messages=messages,
    tools=[{
        "name": t["function"]["name"],
        "description": t["function"]["description"],
        "input_schema": t["function"]["parameters"]
    } for t in TOOLS]
)

# Parse tool calls from response.content blocks
for block in response.content:
    if block.type == "tool_use":
        result = execute_tool(block.name, block.input)
```

Тот же цикл. Те же инструменты. Другая модель.

## Чего не хватает (и что дальше)

Этот харнес работает, но production-агентам нужно больше:

| Функция | Этот харнес | Production-харнес |
|---------|-------------|-------------------|
| Память | Нет (бесостоятельный) | MEMORY.md + ежедневные логи |
| Управление контекстом | Вся история | Приоритетное окно |
| Восстановление ошибок | Базовый try/catch | Повтор + эскалация |
| Безопасность | Нет | Песочничное выполнение |
| Загрузка инструментов | Все сразу | Навыки по запросу |

Каждый из этих пунктов описан в остальном руководстве.

## Частые ошибки

- **Забыть добавить сообщение ассистента** — Если Вы не добавите `msg` в `messages` перед результатами инструментов, модель потеряет нить того, что она запрашивала. Всегда сначала добавляйте полный ответ ассистента.
- **Неправильное преобразование результатов инструментов в строки** — Результаты инструментов должны быть строками. Если Ваш инструмент возвращает dict, используйте `json.dumps()`. Возврат сыного Python-объекта приведёт к краху.
- **Нет лимита итераций** — Без `MAX_TURNS` запутавшаяся модель может зациклиться навечно, сжирая токены. Всегда ограничивайте.

## Дополнительные материалы

- [Руководство по function calling OpenAI](https://platform.openai.com/docs/guides/function-calling) — официальная документация по определениям инструментов
- [Руководство по использованию инструментов Anthropic](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) — аналог для Claude
- [Статья ReAct](https://arxiv.org/abs/2210.03629) — академическая основа для циклов инструментов
