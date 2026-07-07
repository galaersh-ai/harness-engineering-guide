---
author: Nexu
---

# Харнес vs. фреймворк

> **Ключевая идея:** Фреймворк добавляет сотни зависимостей и слоёв абстракций для задачи, которая может потребовать 50 строк Python. Но писать мультиагентную оркестрацию с нуля, когда CrewAI уже это решает, — потеря недель. Ключ — соотнести инструмент с задачей, а не выбирать самый популярный по умолчанию.

Харнес — это код, который Вы пишете с нуля, чтобы обернуть модель инструментами, памятью и контекстом. **Фреймворк** — это библиотека, предоставляющая абстракции для построения агентов — LangChain, CrewAI, AutoGen и другие. Выбор между ними — не о том, что «лучше», а о том, когда каждый из них окупается.

## Дерево решений

```
Need an agent? 
│
├── Is it a single-model loop with < 5 tools?
│   └── YES → Write a raw harness (50-200 lines)
│
├── Do you need multi-agent orchestration out of the box?
│   └── YES → Consider CrewAI or AutoGen
│
├── Do you need complex RAG pipelines with vector stores?
│   └── YES → Consider LangChain
│
├── Is this a production product where you need full control?
│   └── YES → Write a raw harness (own every line)
│
├── Are you prototyping / exploring quickly?
│   └── YES → Framework is fine, expect to rewrite later
│
└── Do you need to understand what's actually happening?
    └── YES → Write a raw harness first, then decide
```

## Одна задача: три способа

**Задача**: Прочитать CSV-файл, проанализировать его и записать сводку в Markdown-файл.

### Харнес с нуля (~60 строк)

```python
import json, csv, io
from openai import OpenAI

client = OpenAI()

TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "Read a file's contents",
            "parameters": {
                "type": "object",
                "properties": {"path": {"type": "string"}},
                "required": ["path"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "write_file",
            "description": "Write content to a file",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string"},
                    "content": {"type": "string"}
                },
                "required": ["path", "content"]
            }
        }
    }
]

def execute(name, args):
    if name == "read_file":
        return open(args["path"]).read()
    elif name == "write_file":
        open(args["path"], "w").write(args["content"])
        return f"Written to {args['path']}"

def run(task):
    messages = [
        {"role": "system", "content": "You analyze data files and write reports."},
        {"role": "user", "content": task}
    ]
    for _ in range(10):
        resp = client.chat.completions.create(
            model="gpt-4o-mini", messages=messages, tools=TOOLS
        )
        msg = resp.choices[0].message
        messages.append(msg)
        if not msg.tool_calls:
            return msg.content
        for tc in msg.tool_calls:
            result = execute(tc.function.name, json.loads(tc.function.arguments))
            messages.append({"role": "tool", "tool_call_id": tc.id, "content": result})
    return "Done"

run("Read data.csv, analyze the trends, and write a summary to report.md")
```

**Зависимости**: `openai` (1 пакет)
**Строк кода**: ~60
**Контроль**: полный

### LangChain (~40 строк, но...)

```python
from langchain_openai import ChatOpenAI
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain.tools import tool

@tool
def read_file(path: str) -> str:
    """Read a file's contents"""
    return open(path).read()

@tool
def write_file(path: str, content: str) -> str:
    """Write content to a file"""
    open(path, "w").write(content)
    return f"Written to {path}"

llm = ChatOpenAI(model="gpt-4o-mini")
prompt = ChatPromptTemplate.from_messages([
    ("system", "You analyze data files and write reports."),
    ("human", "{input}"),
    MessagesPlaceholder("agent_scratchpad"),
])

agent = create_openai_tools_agent(llm, [read_file, write_file], prompt)
executor = AgentExecutor(agent=agent, tools=[read_file, write_file], verbose=True)

executor.invoke({"input": "Read data.csv, analyze trends, write summary to report.md"})
```

**Зависимости**: `langchain`, `langchain-openai`, `langchain-core` плюс их транзитивные зависимости (~50+ пакетов)
**Строк кода**: ~40 (но слои абстракций под капотом исчисляются тысячами)
**Контроль**: определения инструментов, шаблон промпта. Всё остальное принадлежит LangChain.

### CrewAI (~35 строк)

```python
from crewai import Agent, Task, Crew
from crewai_tools import FileReadTool, FileWriterTool

analyst = Agent(
    role="Data Analyst",
    goal="Analyze CSV data and produce insightful reports",
    backstory="You are an expert data analyst.",
    tools=[FileReadTool(), FileWriterTool()],
    verbose=True
)

task = Task(
    description="Read data.csv, analyze the trends, write a summary to report.md",
    expected_output="A markdown report with key findings",
    agent=analyst
)

crew = Crew(agents=[analyst], tasks=[task], verbose=True)
crew.kickoff()
```

**Зависимости**: `crewai`, `crewai-tools` плюс их зависимости (~80+ пакетов)
**Строк кода**: ~35
**Контроль**: роли агентов, описания задач. Поток выполнения принадлежит CrewAI.

## Матрица компромиссов

| Измерение | Харнес с нуля | LangChain | CrewAI |
|-----------|---------------|-----------|--------|
| Строк кода | Больше | Меньше | Меньше всего |
| Зависимости | 1 | ~50 | ~80 |
| Отладка | Легко (это Ваш код) | Сложно (глубокие стек-трейсы) | Средне |
| Гибкость | Полная | Ограничена абстракциями | Только на основе ролей |
| Мультиагентность | Своими руками | Возможна, но сложно | Встроена |
| Кривая обучения | Понять API модели | Изучить концепции LangChain | Изучить концепции CrewAI |
| Путь обновления | Меняйте что хотите | Ждите обновлений LangChain | Ждите обновлений CrewAI |
| Production-готовность | Решаете Вы | Зависит от стабильности версий | Новее, меньше проверено на практике |

## Скрытая стоимость фреймворков

### 1. Отладка чёрных ящиков

Когда что-то ломается в харнесе с нуля, Вы смотрите свои 60 строк. Когда что-то ломается в LangChain:

```
File "langchain/agents/openai_tools/base.py", line 147, in _plan
File "langchain_core/runnables/base.py", line 534, in invoke
File "langchain/chains/base.py", line 89, in __call__
File "langchain_core/callbacks/manager.py", line 442, in _handle_event
...
```

Вы отлаживаете чужую архитектуру.

### 2. Привязка к абстракциям

Хотите добавить стриминг? Пользовательскую память? Нестандартный паттерн вызова инструментов? В харнесе с нуля Вы просто пишете это. Во фреймворке Вы работаете в рамках его точек расширения — или форкаете библиотеку.

### 3. Смена версий

LangChain пережил несколько крупных переработок API. Код, написанный 6 месяцев назад, может не работать сегодня. Харнес с нуля, использующий только пакет `openai`, стабилен уже много лет.

## Когда фреймворки выигрывают

Фреймворки — не плохо. Они действительно помогают, когда:

- **Вы прототипируете** — получите работающий результат за день, чтобы проверить идею. Перепишете позже.
- **Оркестрация мультиагентов** — модель агент-задача CrewAI действительно хороша для сложных многоуровневых рабочих процессов.
- **RAG-пайплайны** — загрузчики документов, сплитеры и интеграции с векторными хранилищами LangChain экономят реальную работу.
- **Вам не важна под капотом** — если агент — малая часть большего продукта и Вам просто нужно, чтобы он работал.

## Гибридный подход

Многие продакшен-команды начинают с фреймворка и мигрируют на харнес с нуля:

```
Week 1:  LangChain prototype → "It works!"
Week 4:  Hit a limitation → "Why can't I do X?"
Week 8:  Fork/override half the framework → "I'm fighting the framework"
Week 12: Rewrite as raw harness → "This is 200 lines and does exactly what I need"
```

Это нормально. Фреймворк научил вас, что вам нужно. Харнес даёт контроль.

## Частые ошибки

- **Начинать с фреймворка, не понимая основ** — Вы не можете отлаживать то, чего не понимаете. Постройте харнес с нуля хотя бы раз, даже если никогда не будете использовать его в продакшене.
- **Выбирать по звёздам на GitHub** — Звёзды ≠ соответствие. Фреймворк с 80K звёзд, предназначенный для RAG-пайплайнов, не поможет Вам построить coding-агента.
- **Бояться «изобретать велосипед»** — Велосипед здесь — это 50 строк Python. Это не такой уж большой велосипед.

## Дополнительные материалы

- [Документация LangChain](https://python.langchain.com/) — самый популярный фреймворк
- [Документация CrewAI](https://docs.crewai.com/) — оркестрация мультиагентов
- [AutoGen](https://microsoft.github.io/autogen/) — мультиагентный фреймворк Microsoft
- [Ваш первый харнес](/guide/your-first-harness) — постройте версию с нуля
