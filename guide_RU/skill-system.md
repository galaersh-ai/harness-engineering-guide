---
author: Nexu
---

# Система навыков

> **Ключевая идея:** Навык — это не инструмент; это *набор* связанных инструментов, документации и правил поведения, упакованных как единая возможность. Система навыков превращает «100 инструментов, запихнутых в каждый промпт» в «меню возможностей, загружаемых по запросу», экономя тысячи токенов и значительно повышая точность выбора инструментов.

## Что такое навык?

Инструмент — это одна функция, которую модель может вызвать. **Навык** — это упакованная возможность, объединяющая:

- **Инструменты** — одну или несколько связанных схем функций и обработчиков
- **Документацию** — файл SKILL.md, объясняющий когда и как использовать навык
- **Правила поведения** — ограничения, паттерны и соглашения для модели

```
skill/
├── SKILL.md          # Documentation: when to use, how to use, constraints
├── tools.py          # Tool implementations
└── schema.json       # Tool schemas (or generated from code)
```

Например, навык `git` не предоставляет один инструмент `git` — он объединяет `git_status`, `git_diff`, `git_commit`, `git_push`, `git_log` и включает документацию о соглашениях для коммит-сообщений, именовании веток и когда спрашивать перед пушем.

## Навык vs. инструмент

| | Инструмент | Навык |
|---|-----------|-------|
| **Область** | Одна функция | Набор связанных функций |
| **Документация** | Описание параметров | Полный SKILL.md с примерами, соглашениями |
| **Загрузка** | Всегда присутствует или отсутствует | Загружается по запросу из меню |
| **Стоимость контекста** | ~100–200 токенов на схему | ~200 токенов запись меню + ~1 000 токенов при загрузке |
| **Правила поведения** | Нет | Может включать ограничения, рабочие процессы, паттерны |

Различие важно для экономии токенов. Харнес с 80 инструментами тратит ~12 000 токенов на каждый API-запрос только на схемы. Система навыков с 15 навыками и 300-токенным меню загружает только то, что нужно.

## Формат SKILL.md

Файл SKILL.md — это руководство по эксплуатации навыка. Модель читает его при загрузке навыка:

```markdown
# Git Operations

## When to Use
- User asks to check, commit, or push code changes
- You need to inspect file history or diffs
- Resolving merge conflicts

## Available Tools
- `git_status` — Show working tree status
- `git_diff` — Show changes (staged or unstaged)
- `git_commit` — Commit staged changes with a message
- `git_push` — Push commits to remote
- `git_log` — Show recent commit history

## Conventions
- Always run `git_status` before committing
- Use conventional commit messages (feat:, fix:, docs:)
- Never force-push without explicit user approval
- Commit message should be under 72 characters

## Examples
To commit and push:
1. `git_status` → review what's changed
2. `git_diff` → verify the changes are correct
3. `git_commit("feat: add user auth middleware")`
4. `git_push`
```

Этот формат даёт модели достаточно контекста для правильного использования навыка без встраивания всех знаний в описания инструментов.

## Паттерн меню навыков

Вместо загрузки всех инструментов при запуске представьте модели компактное меню доступных навыков. Модель читает меню, решает, какой навык ей нужен, и загружает его:

```python
SKILL_MENU = """Available skills (use load_skill to activate):

- file_ops: Read, write, search, and edit files in the workspace
- git: Version control — status, diff, commit, push, log
- web: HTTP requests, web search, URL fetching
- shell: Execute shell commands in a sandbox
- database: SQL queries, schema inspection, migrations
- calendar: Create events, check availability, manage schedules
- email: Read inbox, send emails, search messages
- image: Generate and analyze images
"""
```

Меню стоит ~150 токенов. Загрузка навыка добавляет его SKILL.md (~500–1 000 токенов) и схемы инструментов (~200–800 токенов). По сравнению с загрузкой всех инструментов сразу:

```
Strategy                    Tokens (8 skills, ~60 tools)
────────────────────────────────────────────────────────
All tools upfront:          ~12,000 tokens (always)
Skill menu + 2 loaded:      ~150 + ~2,400 = ~2,550 tokens
────────────────────────────────────────────────────────
Savings:                    ~9,450 tokens per turn (78%)
```

За 30-ходовую сессию это ~280K сэкономленных токенов — реальные деньги по тарифам API.

## Реализация реестра навыков

```python
import json
from pathlib import Path
from dataclasses import dataclass, field

@dataclass
class Skill:
    name: str
    description: str
    doc: str                           # Contents of SKILL.md
    tools: list[dict] = field(default_factory=list)        # Tool schemas
    handlers: dict = field(default_factory=dict)            # name → callable

class SkillRegistry:
    """Registry with on-demand skill loading."""

    def __init__(self, skills_dir: str):
        self.skills_dir = Path(skills_dir)
        self._catalog: dict[str, Skill] = {}
        self._active: dict[str, Skill] = {}
        self._scan()

    def _scan(self):
        """Scan the skills directory and build the catalog."""
        for skill_dir in self.skills_dir.iterdir():
            if not skill_dir.is_dir():
                continue
            skill_md = skill_dir / "SKILL.md"
            schema_file = skill_dir / "schema.json"
            if not skill_md.exists():
                continue

            doc = skill_md.read_text()
            # Extract first line after "# " as description
            first_heading = ""
            for line in doc.splitlines():
                if line.startswith("# "):
                    first_heading = line[2:].strip()
                    break

            schemas = []
            if schema_file.exists():
                schemas = json.loads(schema_file.read_text())

            self._catalog[skill_dir.name] = Skill(
                name=skill_dir.name,
                description=first_heading,
                doc=doc,
                tools=schemas,
            )

    def get_menu(self) -> str:
        """Generate the skill menu for the model."""
        lines = ["Available skills (use load_skill to activate):\n"]
        for name, skill in self._catalog.items():
            status = " [loaded]" if name in self._active else ""
            lines.append(f"- {name}: {skill.description}{status}")
        return "\n".join(lines)

    def load_skill(self, name: str) -> str:
        """Load a skill, making its tools available."""
        if name not in self._catalog:
            return f"Error: Unknown skill '{name}'. Check the skill menu."
        if name in self._active:
            return f"Skill '{name}' is already loaded."

        skill = self._catalog[name]
        self._active[name] = skill
        tool_names = [t["name"] for t in skill.tools]
        return (
            f"Loaded skill '{name}' with {len(skill.tools)} tools: "
            f"{', '.join(tool_names)}\n\n"
            f"Documentation:\n{skill.doc}"
        )

    def unload_skill(self, name: str) -> str:
        """Unload a skill to free up context space."""
        if name not in self._active:
            return f"Skill '{name}' is not loaded."
        del self._active[name]
        return f"Unloaded skill '{name}'."

    def get_active_schemas(self) -> list[dict]:
        """Return tool schemas for all currently loaded skills."""
        schemas = []
        for skill in self._active.values():
            schemas.extend(skill.tools)
        # Always include the meta-tools
        schemas.append({
            "name": "load_skill",
            "description": "Load a skill by name to activate its tools",
            "parameters": {
                "type": "object",
                "properties": {
                    "name": {"type": "string", "description": "Skill name from the menu"}
                },
                "required": ["name"],
            },
        })
        return schemas

    def dispatch(self, tool_name: str, arguments: dict) -> str:
        """Dispatch a tool call to the appropriate skill handler."""
        if tool_name == "load_skill":
            return self.load_skill(arguments["name"])

        for skill in self._active.values():
            if tool_name in skill.handlers:
                try:
                    return str(skill.handlers[tool_name](**arguments))
                except Exception as e:
                    return f"Error: {type(e).__name__}: {e}"

        return f"Error: Tool '{tool_name}' not found. Is the skill loaded?"
```

## Тонкий харнес + толстые навыки

Архитектурный принцип: харнес должен быть тонким — только агентный цикл, сборка контекста и реестр навыков. Вся предметная интеллектуальность живёт в навыках:

```
┌─────────────────────────────────────────────────┐
│  Harness (thin)                                  │
│  ┌──────────┐  ┌───────────┐  ┌──────────────┐ │
│  │ Agentic  │  │  Context   │  │    Skill     │ │
│  │  Loop    │  │ Assembler  │  │  Registry    │ │
│  └──────────┘  └───────────┘  └──────────────┘ │
└─────────────────────┬───────────────────────────┘
                      │ loads on demand
     ┌────────────────┼────────────────┐
     ▼                ▼                ▼
┌─────────┐    ┌───────────┐    ┌───────────┐
│  git    │    │  file_ops  │    │  web      │
│  skill  │    │  skill     │    │  skill    │
└─────────┘    └───────────┘    └───────────┘
```

Это разделение даёт практические преимущества:
- **Навыки переносимы** — навык, написанный для одного харнеса, работает в другом
- **Навыки тестируемы** — тестирование инструментов и схем изолированно
- **Навыки композируемы** — модель естественно открывает, как комбинировать загруженные навыки
- **Харнес остаётся простым** — добавляйте возможности через навыки, а не через модификацию ядра

## Частые ошибки

- **Загружать все навыки при запуске** — сводит на нет цель системы навыков. Используйте паттерн меню и загружайте по запросу.
- **Монолитные навыки** — «делающий всё» навык с 30 инструментами — это просто проблема «все инструменты сразу» под другим именем. Держите навыки фокусированными: 3–8 инструментов каждый.
- **Отсутствие SKILL.md** — инструменты без документации заставляют модель угадывать соглашения. SKILL.md не опционален; это мозг навыка.
- **Нет механизма выгрузки** — если модель загружает пять навыков и не может их выгрузить, контекст быстро заполняется. Всегда предоставляйте `unload_skill` рядом с `load_skill`.
- **Путаница имён навыков и инструментов** — если навык назван `git` и содержит инструмент `git`, модель может попытаться вызвать имя навыка как инструмент. Используйте разные имена: навык `git`, инструменты `git_status`, `git_diff` и т.д.

## Дополнительные материалы

- [Архитектура навыков OpenClaw](https://docs.openclaw.ai) — production-система навыков с меню и загрузкой по запросу
- [Anthropic: Tool Use Guide](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) — лучшие практики проектирования инструментов, применимые к навыкам
- [Model Context Protocol: Tools](https://modelcontextprotocol.io/) — открытый стандарт для совместимости инструментов/навыков между харнесами
