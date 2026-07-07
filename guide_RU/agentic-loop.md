---
author: Nexu
---

# Агентный цикл

> **Ключевая идея:** Любой агент — это цикл: подумай, действуй, наблюдай, повторяй. Сам цикл тривиален. Производственный уровень определяется тем, как Вы обрабатываете крайние случаи: когда останавливаться, что делать при сбоях инструментов и как предотвратить бесконечные циклы.

## Паттерн

Агентный цикл (также называемый паттерном ReAct — Reason + Act, «Рассуждение + Действие») — это базовый цикл выполнения любого ИИ-агента. Модель генерирует ответ, при необходимости вызывает один или несколько инструментов, анализирует результаты и повторяет цикл до завершения задачи.

```
┌─────────────┐
│   Reason    │◄──────────────────┐
│  (LLM call) │                   │
└──────┬──────┘                   │
       │                          │
       ▼                          │
  ┌─────────┐    No tools    ┌────┴─────┐
  │  Tools? ├───────────────►│  Output  │
  └────┬────┘                └──────────┘
       │ Yes
       ▼
  ┌─────────┐
  │ Execute │
  │  tools  │
  └────┬────┘
       │
       ▼
  ┌─────────┐
  │ Observe │
  │ results ├─────────────────────┘
  └─────────┘
```

Это принципиально отличается от простого вызова API инструментов. Одиночный вызов инструмента — это одноразовое действие: модель говорит «вызови эту функцию», Вы возвращаете результат. **Агентный цикл** запускает этот процесс многократно — модель видит результат, решает, что ей нужна дополнительная информация, вызывает другой инструмент, видит *его* результат и продолжает, пока не наберёт достаточно контекста для финального ответа.

## Реализация

Минимальный агентный цикл на Python:

```python
def agentic_loop(messages: list, tools: list, max_turns: int = 25) -> str:
    """Run the agentic loop until the model produces a final text response."""
    for turn in range(max_turns):
        response = llm.chat(messages=messages, tools=tools)
        assistant_msg = response.choices[0].message
        messages.append(assistant_msg)

        # Exit condition: no tool calls means the model is done
        if not assistant_msg.tool_calls:
            return assistant_msg.content

        # Execute each tool call and append results
        for tool_call in assistant_msg.tool_calls:
            result = dispatch_tool(tool_call)
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": str(result)
            })

    raise AgentLoopError(f"Agent did not complete within {max_turns} turns")
```

Параметр `max_turns` критически важен. Без него запутавшаяся модель будет зацикливаться —многократно вызывать один и тот же инструмент, получать одну и ту же ошибку и сжирать токены. Это простейшая защита, которая всегда должна присутствовать.

## Параллельные вызовы инструментов

Современные API поддерживают **параллельные вызовы инструментов** — модель может запросить несколько инструментов в одном ответе. Это не просто оптимизация — это меняет поведение агента. Модели, которой нужно прочитать три файла, запросит все три одновременно, а не последовательно:

```python
# A single assistant message might contain:
# tool_calls = [read_file("a.py"), read_file("b.py"), read_file("c.py")]

for tool_call in assistant_msg.tool_calls:
    result = dispatch_tool(tool_call)
    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": str(result)
    })
# All three results are appended, then the model sees them all at once
```

## Бюджет ходов и условия выхода

Циклу нужны чёткие условия выхода помимо `max_turns`:

| Условие | Действие |
|---------|----------|
| Нет вызовов инструментов в ответе | Вернуть текст — агент завершил работу |
| Достигнут максимум ходов | Выбросить ошибку или принудительно создать сводку |
| Превышен бюджет токенов | Запустить сжатие контекста, затем продолжить |
| Повторяющиеся одинаковые вызовы | Вероятно, зацикливание — eskейл или прервать |
| Сигнал прерывания от человека | Показать текущее состояние |

```python
def detect_loop(messages: list, window: int = 3) -> bool:
    """Detect if the agent is stuck calling the same tool repeatedly."""
    recent_calls = []
    for msg in messages[-window * 2:]:
        if hasattr(msg, 'tool_calls') and msg.tool_calls:
            recent_calls.extend(
                (tc.function.name, tc.function.arguments) for tc in msg.tool_calls
            )
    if len(recent_calls) >= window:
        return len(set(recent_calls[-window:])) == 1
    return False
```

## Стриминг в цикле

Production-харнесы стримят вывод модели по токенам во время работы цикла. Это важно для пользовательского опыта — человек видит, как агент «думает» в реальном времени, а не смотрит на пустой экран:

```python
for turn in range(max_turns):
    stream = llm.chat(messages=messages, tools=tools, stream=True)

    tool_calls = []
    text_chunks = []

    for chunk in stream:
        delta = chunk.choices[0].delta
        if delta.content:
            text_chunks.append(delta.content)
            emit_to_user(delta.content)  # Real-time streaming
        if delta.tool_calls:
            accumulate_tool_calls(tool_calls, delta.tool_calls)

    if not tool_calls:
        return "".join(text_chunks)

    # Execute tools and continue loop
    ...
```

## Частые ошибки

- **Нет лимита ходов** — самая распространённая ошибка харнесов. Всегда устанавливайте максимум.
- **Поглощение ошибок инструментов** — если инструмент молча терпит неудачу, модель будет повторять попытки или галлюцинировать успех. Всегда возвращайте сообщения об ошибках как результаты инструментов, чтобы модель могла адаптироваться.
- **Добавление сырых результатов** — большие выводы инструментов (целые файлы, ответы API) раздувают контекстное окно. Обрезайте или суммаризируйте перед добавлением.
- **Игнорирование параллельных вызовов** — если Ваш цикл обрабатывает вызовы инструментов последовательно, но модель выпустила их параллельно, Вы можете создать зависимости порядка, которых не существует.

## Дополнительные материалы

- [Yao et al., "ReAct: Synergizing Reasoning and Acting"](https://arxiv.org/abs/2210.03629) — оригинальная статья, формализующая паттерн Reason + Act
- [Anthropic: Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — практические паттерны для production-циклов
