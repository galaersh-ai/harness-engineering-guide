---
author: Nexu
---

# Инженерия контекста

> **Ключевая идея:** Модель не знает того, что Вы ей не сообщили. Инженерия контекста — это дисциплина, определяющая, что попадает в контекстное окно, в каком порядке и что отбрасывается, когда место заканчивается. Это самая высокоэффективная работа в инженерии харнесов — более значимая, чем выбор модели, настройка промптов или проектирование инструментов.

## Проблема

Контекстное окно в 128K токенов кажется огромным, пока Вы не начнёте его заполнять. Один большой файл может занять 10K токенов. Двадцать схем инструментов «съедают» 3K. История диалога растёт линейно с каждым ходом. В течение дюжины ходов сложной задачи по программированию Вы уже делаете непростой выбор, что сохранить, а что отбросить.

Инженерия контекста — это искусство этих выборов. У неё три столпа: **сборка** (что попадает внутрь), **сжатие** (что уменьшается) и **бюджетирование** (как распределяется ёмкость).

## Система приоритетов сборки контекста

Не весь контекст одинаков. Система приоритетов гарантирует, что наиболее критичная информация выживает, когда места мало:

| Приоритет | Категория | Типичные токены | Примечания |
|-----------|-----------|-----------------|------------|
| 0 (высший) | Системный промпт | 300–800 | Идентичность, правила поведения, ограничения безопасности |
| 1 | Схемы активных инструментов | 1 000–3 000 | Только загруженные навыки, не все инструменты |
| 2 | Инструкция задачи | 200–1 000 | Текущий запрос пользователя + закреплённые цели |
| 3 | Сводка памяти | 500–2 000 | Сжатый MEMORY.md + сегодняшний ежедневный лог |
| 4 | Инжектируемые файлы | 2 000–20 000 | AGENTS.md, SKILL.md, релевантные файлы исходников |
| 5 | Недавний диалог | 5 000–50 000 | Последние N ходов сообщений + результаты инструментов |
| 6 (низший) | Старый диалог | остаток | Ранние ходы, первыми сжимаются или отбрасываются |

Сборщик проходит этот список сверху вниз, упаковывая контент, пока бюджет не исчерпан. Контент с низким приоритетом обрезается или исключается полностью.

```python
import tiktoken

encoder = tiktoken.encoding_for_model("gpt-4o")

def estimate_tokens(text: str) -> int:
    """Fast token estimation using tiktoken."""
    return len(encoder.encode(text))

class ContextAssembler:
    """Assemble context with priority-based token budgeting."""

    def __init__(self, max_tokens: int = 128_000, reserve: int = 4_096):
        self.max_tokens = max_tokens
        self.reserve = reserve  # Leave room for the model's response
        self.budget = max_tokens - reserve
        self.sections: list[tuple[int, str, str]] = []

    def add(self, priority: int, name: str, content: str):
        """Add a section. Lower priority number = higher importance."""
        self.sections.append((priority, name, content))

    def build(self) -> list[dict]:
        """Pack sections into messages within the token budget."""
        self.sections.sort(key=lambda s: s[0])
        messages = []
        used = 0

        for priority, name, content in self.sections:
            tokens = estimate_tokens(content)
            if used + tokens <= self.budget:
                messages.append({
                    "role": "system",
                    "content": f"[{name}]\n{content}",
                })
                used += tokens
            elif priority <= 2:
                # Critical sections get truncated rather than dropped
                remaining = self.budget - used
                truncated = self._truncate_to_tokens(content, remaining)
                if truncated:
                    messages.append({
                        "role": "system",
                        "content": f"[{name} (truncated)]\n{truncated}",
                    })
                    used += estimate_tokens(truncated)
            # Priority > 2: silently dropped when over budget

        return messages

    def _truncate_to_tokens(self, text: str, max_tokens: int) -> str:
        """Truncate text to fit within a token limit."""
        tokens = encoder.encode(text)
        if len(tokens) <= max_tokens:
            return text
        return encoder.decode(tokens[:max_tokens]) + "\n[...truncated]"
```

Параметр `reserve` легко упустить, но он критичен — нужно оставить запас для ответа модели. Если Вы упакуете контекст на 100%, у модели не будет места для ответа.

## Сжатие контекста: три линии обороны

По мере продвижения сессии сырая история диалога растёт без ограничений. Три техники не дают ей заполнить всё контекстное окно:

### Линия 1: Автоматическое затухание

Старые сообщения естественно теряют актуальность. Простая стратегия затухания отбрасывает сообщения за пределами фиксированного окна, сохраняя только последние N ходов:

```python
def apply_decay(messages: list[dict], max_turns: int = 20) -> list[dict]:
    """Keep the system prompt and the last max_turns exchanges."""
    system = [m for m in messages if m["role"] == "system"]
    conversation = [m for m in messages if m["role"] != "system"]
    # Each "turn" is roughly a user + assistant + tool cycle
    if len(conversation) > max_turns * 3:
        conversation = conversation[-(max_turns * 3):]
    return system + conversation
```

### Линия 2: Пороговое сжатие

Когда общее количество токенов пересекает порог (например, 70% бюджета), старые ходы диалога сжимаются в сводку, а последние ходы сохраняются дословно:

```python
def threshold_compress(
    messages: list[dict],
    budget: int,
    threshold: float = 0.7,
    keep_recent: int = 10,
) -> list[dict]:
    """Compress older messages when token usage exceeds threshold."""
    total = sum(estimate_tokens(m["content"]) for m in messages)
    if total < budget * threshold:
        return messages  # Under threshold, no compression needed

    system = [m for m in messages if m["role"] == "system"]
    conversation = [m for m in messages if m["role"] != "system"]

    old = conversation[:-keep_recent]
    recent = conversation[-keep_recent:]

    summary = summarize_with_llm(old)  # Use a fast, cheap model
    compressed = system + [{
        "role": "system",
        "content": f"[Conversation summary]\n{summary}",
    }] + recent

    return compressed
```

### Линия 3: Активная суммаризация

Для чрезвычайно длительных задач периодически извлекайте ключевые факты и решения в документ-сводку. Это не автоматически — харнес явно просит модель создать контрольную точку:

```python
SUMMARIZE_PROMPT = """Summarize the key decisions, findings, and current state 
from this conversation. Include: files modified, tests run, errors encountered, 
and the current plan. Be concise — under 500 words."""

def active_summarize(messages: list[dict]) -> str:
    """Ask the model to produce a checkpoint summary."""
    response = llm.chat(
        messages=messages + [{"role": "user", "content": SUMMARIZE_PROMPT}],
        max_tokens=1024,
    )
    return response.choices[0].message.content
```

## Бюджетирование токенов на практике

Реальная арифметика токенов для контекстного окна в 128K:

```
Total capacity:              128,000 tokens
Response reserve:             -4,096
System prompt:                  -500
Tool schemas (12 tools):      -2,400
MEMORY.md:                    -1,200
AGENTS.md:                      -800
─────────────────────────────────────
Available for conversation:  119,004 tokens

At ~3 tokens/word, that's ~39,600 words of conversation.
A 50-turn coding session with tool results: ~60,000 tokens.
→ You'll hit the budget around turn 35 without compression.
```

Вывод: сжатие обязательно для любой нетривиальной сессии.

## Паттерны инжекции контекста

Контекст поступает не только из истории диалога. Пять распространённых паттернов инжекции:

| Паттерн | Когда | Пример |
|---------|-------|--------|
| **Инжекция файлов** | Запуск сессии | Загрузка AGENTS.md, MEMORY.md, релевантных файлов исходников |
| **Инжекция памяти** | Запуск сессии | Сжатая долгосрочная память + недавние ежедневные логи |
| **Инжекция результатов инструментов** | В цикле | Добавление выводов инструментов как сообщений с ролью tool |
| **Инжекция навыков** | По запросу | Загрузка SKILL.md при активации навыка |
| **Инжекция извлечения** | За запрос | RAG-результаты из векторного хранилища |

Каждая точка инжекции имеет стоимость. Файл исходников на 200 строк — это ~800 токенов. Спекулятивная инжекция десяти файлов стоит 8K токенов ещё до начала диалога. Будьте осознанными: инжектируйте то, что нужно, а не то, что *может* понадобиться.

## Реализация скользящего окна

Скользящее окно сохраняет последние ходы нетронутыми и сжимает всё до границы окна. Это самая практичная стратегия для production-харнесов:

```python
class SlidingWindowContext:
    """Maintain a sliding window over conversation history."""

    def __init__(self, window_size: int = 15, max_tokens: int = 128_000):
        self.window_size = window_size
        self.max_tokens = max_tokens
        self.summary = ""
        self.messages: list[dict] = []

    def add(self, message: dict):
        self.messages.append(message)
        conversation = [m for m in self.messages if m["role"] != "system"]
        if len(conversation) > self.window_size * 3:
            self._compress()

    def _compress(self):
        """Move older messages into a rolling summary."""
        conversation = [m for m in self.messages if m["role"] != "system"]
        system = [m for m in self.messages if m["role"] == "system"]

        old = conversation[:-(self.window_size * 3)]
        recent = conversation[-(self.window_size * 3):]

        new_summary = summarize_with_llm(
            [{"role": "system", "content": self.summary}] + old
        )
        self.summary = new_summary
        self.messages = system + recent

    def get_messages(self) -> list[dict]:
        """Return context-ready message list."""
        result = [m for m in self.messages if m["role"] == "system"]
        if self.summary:
            result.append({
                "role": "system",
                "content": f"[Conversation history summary]\n{self.summary}",
            })
        result.extend(m for m in self.messages if m["role"] != "system")
        return result
```

## Частые ошибки

- **Рассматривать весь контекст как равноприоритетный** — Системный промпт и инструкции задачи должны выжить; старый диалог можно сжать. Без приоритетов Вы либо тратите место на устаревшие сообщения, либо отбрасываете критичные инструкции.
- **Сжимать слишком агрессивно** — Суммаризация с потерями. Если Вы сожмёте результат инструмента, содержавший путь к файлу, который модели понадобится позже, она сгаллюцинирует путь. Сохраняйте последние ходы дословно.
- **Игнорировать подсчёт токенов** — «Кажется, это коротко» быстро ломается. Используйте фактический подсчёт токенов (tiktoken, специфичные для модели токенизаторы) для бюджетирования.
- **Одноразовая сборка контекста** — Собрать контекст один раз при запуске сессии и никогда не обновлять означает, что модель работает с устаревшими данными после первого вызова инструмента. Пересобирайте контекст каждый ход.

## Дополнительные материалы

- [Karpathy: "Context Engineering is the New Prompt Engineering"](https://x.com/karpathy/status/1937902263428948034) — почему сборка контекста важнее трюков с промптами
- [Letta: MemGPT и будущее памяти агентов](https://www.letta.com/blog/memgpt) — управление памятью по аналогии с ОС с виртуальным контекстом
- [OpenAI: Managing Tokens](https://platform.openai.com/docs/guides/text-generation#managing-tokens) — основы подсчёта токенов
