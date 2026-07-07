---
author: Nexu
---

# Система инструментов

> **Ключевая идея:** Инструменты — это руки агента. Модель рассуждает; инструменты действуют. Но проектирование системы инструментов — как они регистрируются, описываются, диспетчизируются и управляются — оказывает большее влияние на качество агента, чем сама модель.

## Что такое инструмент?

Инструмент — это функция, которую модель может вызвать по имени со структурированными аргументами. Модель видит **схему** (имя, описание, типы параметров); харнес обеспечивает **выполнение** (непосредственный вызов функции и возврат результата).

```python
# What the model sees (tool schema)
{
    "name": "read_file",
    "description": "Read the contents of a file at the given path",
    "parameters": {
        "type": "object",
        "properties": {
            "path": {"type": "string", "description": "File path to read"}
        },
        "required": ["path"]
    }
}

# What the harness executes (tool implementation)
def read_file(path: str) -> str:
    with open(path, 'r') as f:
        return f.read()
```

Модель никогда не видит и не выполняет реализацию. Она знает только схему. Это разделение фундаментально — оно означает, что Вы можете изменить способ работы инструмента, не меняя поведение модели, и ограничить возможности инструмента, не зная об этом модели.

## Реестр инструментов

Реестр инструментов — это компонент харнеса, который сопоставляет имена инструментов с их схемами и реализациями:

```python
class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, Tool] = {}

    def register(self, name: str, schema: dict, handler: Callable):
        self._tools[name] = Tool(name=name, schema=schema, handler=handler)

    def get_schemas(self) -> list[dict]:
        """Return schemas for the LLM API call."""
        return [t.schema for t in self._tools.values()]

    def dispatch(self, name: str, arguments: dict) -> str:
        """Execute a tool call and return the result as a string."""
        tool = self._tools.get(name)
        if not tool:
            return f"Error: Unknown tool '{name}'"
        try:
            result = tool.handler(**arguments)
            return str(result)
        except Exception as e:
            return f"Error: {type(e).__name__}: {e}"
```

Обратите внимание, что `dispatch` всегда возвращает строку, даже для ошибок. Это намеренно — модели нужно видеть сообщения об ошибках, чтобы адаптировать подход, а не молча падать.

## Статические vs. динамические инструменты

**Статические инструменты** загружаются при запуске и всегда доступны. Это работает для небольших наборов инструментов (5–15), но масштабируется плохо — 100 инструментов означают 100 схем в каждом API-запросе, потребляя токены и запутывая модель.

**Динамические инструменты** (также называемые **загрузкой навыков**) решают эту проблему, представляя модели меню доступных категорий инструментов и загружая конкретные инструменты только по запросу:

```python
# Instead of loading all 100 tools, show a menu
SKILL_MENU = """
Available skills (use load_skill to activate):
- file_ops: Read, write, search files
- git: Git operations (status, diff, commit, push)
- web: HTTP requests, web search
- database: SQL queries, schema inspection
"""

# The model calls load_skill("git") and then gets git-specific tools
def load_skill(skill_name: str) -> str:
    tools = skill_registry.load(skill_name)
    active_tools.extend(tools)
    return f"Loaded {len(tools)} tools: {[t.name for t in tools]}"
```

Экономия токенов впечатляет. Меню навыков может стоить 200 токенов; загрузка всех инструментов сразу — более 5 000.

## Качество описаний инструментов

Способность модели правильно использовать инструменты почти полностью зависит от качества описаний инструментов. Расплывчатое описание ведёт к неправильному использованию; точное — направляет к правильному поведению:

```python
# Bad — the model will guess at behavior
{"name": "search", "description": "Search for things"}

# Good — unambiguous, includes format and constraints
{
    "name": "search_files",
    "description": "Search for files matching a glob pattern in the workspace. "
                   "Returns a list of relative file paths, one per line. "
                   "Max 100 results. Use '**/*.py' for recursive Python file search.",
    "parameters": {
        "type": "object",
        "properties": {
            "pattern": {
                "type": "string",
                "description": "Glob pattern (e.g., '*.md', 'src/**/*.ts')"
            }
        },
        "required": ["pattern"]
    }
}
```

Ключевые принципы описаний инструментов:
- Описывайте, что инструмент **делает**, а не что он **представляет**
- Указывайте **формат вывода** (JSON, простой текст, по одному на строку)
- Включайте **ограничения** (максимум результатов, лимиты размера файлов)
- Добавляйте **примеры** для неочевидных параметров

## Паттерны композиции инструментов

Сложные возможности агента часто возникают из композиции простых инструментов, а не из построения сложных:

| Паттерн | Пример |
|---------|--------|
| **Последовательный** | `read_file` → `edit_file` → `run_tests` |
| **Развёртывание (fan-out)** | Чтение 5 файлов параллельно, затем синтез |
| **Условный** | `list_files` → решение, какой `read_file` |
| **Итеративный** | `run_tests` → `edit_file` → `run_tests` (до прохождения) |

Харнесу не нужно реализовывать эти паттерны — модель обнаруживает их естественно через агентный цикл. Ваша задача — предоставить правильные атомарные инструменты и позволить модели композировать их.

## MCP: Model Context Protocol

[MCP](https://modelcontextprotocol.io/) — это открытый стандарт для предоставления инструментов агентам через транспортный слой (stdio, HTTP SSE). Вместо хардкода инструментов в харнес, MCP позволяет подключаться к внешним серверам инструментов:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    }
  }
}
```

MCP значим тем, что отделяет реализацию инструментов от харнеса. Инструмент, написанный для одного харнеса, работает в любом MCP-совместимом харнесе — Claude Desktop, OpenClaw, Cursor и других.

## Частые ошибки

- **Слишком много инструментов одновременно** — Более ~20 активных инструментов ухудшают работу модели. Используйте динамическую загрузку.
- **Молчаливые сбои** — Инструменты, возвращающие пустые строки при ошибке, заставляют модель угадывать. Всегда возвращайте явные сообщения об ошибках.
- **Отсутствие результатов инструментов** — Если Вы забыли добавить результат инструмента в историю сообщений, API-запрос завершится ошибкой. Каждый вызов инструмента должен иметь соответствующий результат.
- **Несогласованные типы возврата** — Если `read_file` иногда возвращает контент, а иногда словарь ошибки, модель не может надёжно разобрать вывод. Стандартизируйте формат результатов.

## Дополнительные материалы

- [Anthropic: Tool Use Guide](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) — производственные паттерны использования инструментов
- [Model Context Protocol](https://modelcontextprotocol.io/) — открытый стандарт для инструментов агентов
