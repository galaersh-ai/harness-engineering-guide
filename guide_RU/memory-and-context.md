---
author: Nexu
---

# Память и контекст

> **Ключевая идея:** Модель знает только то, что находится в её контекстном окне. Память — это способ соединить то, что модели *нужно знать*, с тем, что она *видит* в одном API-запросе. Правильное решение этой задачи — самый высокоэффективный вопрос в инженерии харнесов.

## Три различных понятия

Эти термины часто смешивают, но они служат разным целям:

| Понятие | Область | Персистентность | Пример |
|---------|---------|-----------------|--------|
| **Контекст** | Один API-запрос | Нет — пересобирается каждый ход | Системный промпт + инструменты + недавние сообщения + релевантные файлы |
| **Сессия** | Один диалог или задача | В оперативной памяти, теряется при перезапуске | История сообщений, результаты вызовов инструментов, рабочее состояние |
| **Память** | Кросс-сессионная, бессрочная | Записывается на диск | MEMORY.md, ежедневные логи, изучённые предпочтения |

**Контекст** — это «рабочая память» модели — всё, что собрано в один промпт. **Сессия** — это состояние текущего взаимодействия. **Память** — это то, что переживает завершение сессии.

## Сборка контекста

Каждый ход агентного цикла начинается со сборки контекста. Это задача приоритетной упаковки — у Вас фиксированный бюджет токенов, и нужно решить, что туда попадёт:

```
Context Window (e.g., 128K tokens)
┌─────────────────────────────────┐
│  System Prompt        (~500)    │  ← Always included, highest priority
│  Tool Schemas         (~2000)   │  ← Active tools only
│  Memory Summary       (~1000)   │  ← Compressed long-term memory
│  Relevant Files       (~5000)   │  ← Task-specific context
│  Conversation History (~varies) │  ← Grows over time, needs pruning
│  [Remaining Budget]             │  ← Available for new content
└─────────────────────────────────┘
```

Система приоритетов определяет, что включается, когда места мало:

```python
class ContextAssembler:
    def __init__(self, max_tokens: int = 128_000):
        self.max_tokens = max_tokens
        self.sections = []  # (priority, name, content)

    def add(self, priority: int, name: str, content: str):
        self.sections.append((priority, name, content))

    def build(self) -> list[dict]:
        # Sort by priority (lower = higher priority)
        self.sections.sort(key=lambda x: x[0])
        messages = []
        used_tokens = 0
        for priority, name, content in self.sections:
            token_count = estimate_tokens(content)
            if used_tokens + token_count > self.max_tokens:
                break  # Budget exceeded — skip lower-priority content
            messages.append({"role": "system", "content": f"[{name}]\n{content}"})
            used_tokens += token_count
        return messages
```

## Управление сессиями

Сессия — это граница одного запуска агента. Она хранит:

- **Историю сообщений** — полный диалог, включая вызовы инструментов и результаты
- **Рабочее состояние** — какие файлы открыты, какие навыки загружены, текущий прогресс задачи
- **Временное хранилище** — данные, которые агент сгенерировал, но ещё не зафиксировал

Ключевой выбор при проектировании сессии — **когда её очищать**. Варианты:

| Стратегия | Поведение | Применение |
|-----------|-----------|------------|
| **На задачу** | Новая сессия на каждый запрос пользователя | Без состояния, ассистент |
| **На диалог** | Сессия сохраняется между ходами в одном чате | Интерактивное программирование |
| **Постоянная** | Сессия переживает перезапуск процесса | Долговременный фоновый агент |

Постоянные сессии требуют сериализации — записи состояния сессии на диск для восстановления. Здесь сессия и память пересекаются: всё, что стоит сохранять между перезапусками, лучше записывать в файл памяти, а не хранить в состоянии сессии.

## Архитектура памяти

Проверенная архитектура памяти использует двухуровневую систему:

### Уровень 1: Ежедневные логи

Сырые, хронологические записи о том, что произошло. Пишутся во время сессии, не отбираются:

```markdown
<!-- memory/2026-04-15.md -->
# 2026-04-15

## 14:30 — Refactored auth module
- Moved JWT validation from middleware to dedicated service
- Tests passing (23/23)
- User prefers explicit error messages over error codes

## 16:00 — Deploy to staging
- Used blue-green deployment
- Rollback plan: revert commit abc123
```

### Уровень 2: Долгосрочная память

Отобранные, выжатые знания. Обновляются периодически (не каждую сессию):

```markdown
<!-- MEMORY.md -->
# Long-term Memory

## User Preferences
- Prefers explicit error messages over error codes
- Uses pytest, not unittest
- Deploy strategy: blue-green with rollback plan

## Project Knowledge
- Auth module: JWT validation in /src/services/auth.py
- Database: PostgreSQL 15, migrations in /db/migrations/
- CI: GitHub Actions, ~3min build time

## Lessons Learned
- Always run tests before committing (broke build on 4/10)
- User dislikes verbose output — keep summaries under 5 lines
```

Ключевая идея: ежедневные логи дешёвы в записи (просто дописывайте). Долгосрочная память требует суждения (что стоит сохранять?). Production-харнесы пишут ежедневные логи автоматически и периодически отбирают MEMORY.md — по расписанию или когда агент обнаруживает значимые выводы.

## Цикл чтения/записи памяти

```python
def session_startup(memory_dir: str) -> str:
    """Read memory at session start."""
    sections = []
    # Always read long-term memory
    memory_path = os.path.join(memory_dir, "MEMORY.md")
    if os.path.exists(memory_path):
        sections.append(open(memory_path).read())
    # Read recent daily logs (today + yesterday)
    for days_ago in [0, 1]:
        date = (datetime.now() - timedelta(days=days_ago)).strftime("%Y-%m-%d")
        daily_path = os.path.join(memory_dir, f"memory/{date}.md")
        if os.path.exists(daily_path):
            sections.append(open(daily_path).read())
    return "\n---\n".join(sections)
```

## Паттерн AGENTS.md

Связанный, но отличный файл — AGENTS.md: текстовый файл, определяющий, как агент должен *вести себя* (а не что он *помнит*). Поместите его в любую директорию, и совместимый харнес прочитает его автоматически:

```markdown
<!-- AGENTS.md -->
# Behavior

- You are a Python backend engineer
- Use pytest for all tests
- Follow Google style docstrings
- Never modify files in /config/ without asking

# Tools

- Prefer `ruff` over `pylint` for linting
- Use `uv` for package management
```

AGENTS.md — **декларативный** (что делать), а MEMORY.md — **эксперенциальный** (что произошло). Оба вставляются в контекст при запуске сессии, но служат разным целям.

## Частые ошибки

- **Рассматривать контекст как бесконечный** — Даже 128K токенов быстро заполняются схемами инструментов, содержимым файлов и историей диалогов. Планируйте бюджет токенов явно.
- **Никогда не очищать историю сессий** — 50-ходовой диалог копит избыточный контент. Сжимайте или суммаризируйте старые ходы, чтобы освободить место.
- **Слишком ревностно писать память** — Не каждый ход генерирует знания, которые стоит персистировать. Перезапись создаёт шум, разбавляющий полезную информацию.
- **Забывать читать память при запуске** — Агент без чтения памяти фактически амнестик. Это самая распространённая ошибка конфигурации.

## Дополнительные материалы

- [Letta: MemGPT и будущее памяти агентов](https://www.letta.com/blog/memgpt) — управление памятью для агентов по аналогии с операционными системами
- [Anthropic: Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — паттерны памяти в продакшене
