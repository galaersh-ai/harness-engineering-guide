---
author: Nexu
---

# Проектирование харнеса для длительных задач

> **Ключевая идея:** Агенты коротких задач падают корректно — либо завершаются, либо истекают по таймауту. Агенты длительных задач падают незаметно. Они производят раздутый контекст, молча деградируют и убеждают себя, что делают отличную работу, пока не сворачивают с курса. Проектирование харнеса для длительных задач — это проектирование против этих режимов сбоя.

## Почему агенты длительных задач сложны

Агент «короткой задачи» — ответить на вопрос, написать функцию, просуммировать документ — живёт и умирает в пределах одного контекстного окна. Он завершается или падает видимо.

Агент «длительной задачи» работает часами или днями: рефакторинг кодовой базы, написание 50-страничного отчёта, запуск многоэтапного пайплайна. Эти агенты сталкиваются с проблемами, которых короткие задачи никогда не встречают:

1. **Контекст накапливается.** Каждый вызов инструмента, каждый промежуточный результат, каждый шаг рассуждения добавляет токены. 200K окно заполняется быстрее, чем кажется.
2. **Качество молча деградирует.** Агент не падает — он просто становится хуже. Ответы становятся расплывчатее, инструкции забываются, ранний контекст вытесняется.
3. **Самооценка врёт.** Спросите агента «хороша ли Ваша работа?» — и он скажет да. Всегда. Это нормально для 30-секундной задачи, которую можно визуально оценить. Это катастрофа для 4-часового пайплайна, за которым Вы не следите.

## Режим сбоя №1: Контекстная тревога

Когда агент длительной задачи заполняет своё контекстное окно, происходит контринтуитивная вещь: модель начинает торопиться. Она завершает преждевременно, срезает углы и объявляет «готово» до фактического завершения работы.

Это **контекстная тревога** — неявное осознание модели, что у неё заканчивается место. Это проявляется как:

- Пропуск шагов, которые она обычно выполняет
- Более короткие, менее тщательные выводы
- Преждевременное объявление завершения с «я охватил основные моменты»
- Избегание вызовов инструментов, которые добавят больше контекста

Контекстная тревога возникает архитектурно. Модель выучила, что разговоры заканчиваются, и по мере сокращения места стремится к завершению.

**Большее окно откладывает проблему; не решает её.** Решение архитектурное: управляйте жизненным циклом контекста явно.

## Режим сбоя №2: Предвзятость самооценки

Попросите генератора оценить собственный вывод. Он поставит себе 8/10 или выше — стабильно, независимо от фактического качества. Это **предвзятость самооценки**, и это второй молчаливый убийца агентов длительных задач.

Почему? Модель имеет полный контекст своих собственных рассуждений — каждый выбор кажется оправданным. Признать неудачу значит противоречить предыдущим выводам, чему LLM сопротивляются. А данные обучения поощряют уверенность вместо самоанализа.

В короткой задаче человек ловит проблемы. В длительной задаче агент работает автономно. Если он оценивает собственные выводы и всегда говорит «выглядит хорошо», ошибки копятся без контроля.

```
Short task:   Agent produces → Human reviews → Feedback
Long-running: Agent produces → Agent reviews → "Looks great!" → Errors compound
```

Вывод из проектирования соревновательных сетей применим здесь: **никогда не позволяйте генератору проверять свой собственный экзамен.**

## Управление контекстом: сброс vs. сжатие

Когда контекст заполняется, у Вас два варианта. У каждого реальные компромиссы.

### Сброс контекста

Стереть диалог и начать с чистого листа. Передать сводку предыдущей работы в новый контекст как «брифинг».

```
Turn 1-50:  [full conversation history]
            ↓ context 80% full
Turn 51:    [system prompt + summary of turns 1-50 + current task]
            ↓ fresh start, ~10% context used
Turn 51-100: [continues from summary]
```

**Плюсы:** Чистый лист, предсказуемый бюджет токенов, устранение контекстной тревоги для нового сегмента.

**Минусы:** С потерями — суммаризации упускают нюансы и неудачные подходы. «Суммаризация суммаризации» деградирует при нескольких сбросах. Агент может вернуться к тупиковым путям.

### Сжатие контекста

Избирательное сжатие старых ходов с сохранением последних нетронутыми. Сворачивание многходовых рассуждений в суммаризации, отбрасывание многословных выводов инструментов.

```
Turn 1-20:  [compressed: 3-line summary of early exploration]
Turn 21-40: [compressed: key decisions and outcomes]
Turn 41-50: [full detail: recent work in progress]
```

**Плюсы:** Сохраняет преемственность. Ступенчатое — последние ходы остаются детальными, старые сжимаются. Агент сохраняет осведомлённость о том, что он пробовал.

**Минусы:** Качество сжатия варьируется. Сложнее в реализации. Сжатый контекст может запутать модель, если суммаризации конфликтуют с недавним состоянием.

### Какой выбрать?

| Сценарий | Предпочтение |
|----------|--------------|
| Задачи с чёткими фазами (исследование → написание → проверка) | Сброс между фазами |
| Непрерывная итерация над одним артефактом | Сжатие |
| Агент часто возвращается к ранним решениям | Сжатие (сохраняет историю решений) |
| Контекст накопил много выводов инструментов | Сброс (выводы инструментов плохо сжимаются) |

На практике многие харнесы используют гибрид: сжатие внутри фазы, сброс между фазами.

## Архитектура генератор-оценщик

Заимствуем из GAN: генератор создаёт, дискриминатор судит — раздельные сети с противоположными целями. Применим тот же принцип к агентам:

```
┌─────────────┐         ┌──────────────┐
│  Generator  │────────►│  Evaluator   │
│  (Agent A)  │         │  (Agent B)   │
│             │◄────────│              │
│  Produces   │ feedback│  Judges      │
│  output     │         │  output      │
└─────────────┘         └──────────────┘
        │                       │
        │    Separate context   │
        │    Separate prompt    │
        │    Separate criteria  │
```

**Ключевые правила проектирования:**

1. **Раздельные контексты.** Оценщик видит только вывод, а не рассуждения генератора. Предотвращает предвзятость симпатии.
2. **Явный рубрикатор.** Оценивайте по чек-листу, а не по вайбу. «Обрабатывает ли код граничный случай X?» лучше, чем «хороший ли код?».
3. **Действенная обратная связь.** Возвращайте конкретные проблемы, а не оценки. «Функция `parse_input` не обрабатывает пустые строки» полезно. «7/10» — нет.
4. **Бюджет итераций.** Ограничьте цикл. Без лимита идеалистичный оценщик + ретивый генератор = бесконечный цикл.

```python
def generator_evaluator_loop(task, max_iterations=3):
    output = None
    for i in range(max_iterations):
        # Generator: produce or revise
        if output is None:
            output = generator.run(task)
        else:
            output = generator.revise(task, output, feedback)

        # Evaluator: judge with fresh eyes
        evaluation = evaluator.judge(task, output)  # no generator context!

        if evaluation.passes:
            return output

        feedback = evaluation.issues

    return output  # best effort after max iterations
```

## Трёхагентная архитектура: Планировщик → Генератор → Оценщик

Для сложных длительных задач добавьте **Планировщик** для декомпозиции, выполнения и контроля качества.

```
                    ┌─────────────┐
                    │   Planner   │
                    │             │
                    │ Decomposes  │
                    │ task into   │
                    │ subtasks    │
                    └──────┬──────┘
                           │
                           ▼
              ┌─── subtask list ───┐
              │                    │
              ▼                    ▼
     ┌─────────────┐      ┌─────────────┐
     │  Generator  │      │  Generator  │   (parallel or sequential)
     │  subtask 1  │      │  subtask 2  │
     └──────┬──────┘      └──────┬──────┘
            │                    │
            ▼                    ▼
     ┌─────────────┐      ┌─────────────┐
     │  Evaluator  │      │  Evaluator  │
     │  subtask 1  │      │  subtask 2  │
     └──────┬──────┘      └──────┬──────┘
            │                    │
            └────────┬───────────┘
                     ▼
              ┌─────────────┐
              │   Planner   │
              │  (reviews   │
              │   results,  │
              │   re-plans  │
              │   if needed)│
              └─────────────┘
```

**Планировщик** — Декомпозирует цель на подзадачи с критериями успеха. Перепланирует, когда оценщики сигнализируют о проблемах. Держит видение, но не выполняет.

**Генератор** — Выполняет одну подзадачу за раз со свежим контекстом. Имеет инструменты, файлы, среды выполнения. Не оценивает собственную работу.

**Оценщик** — Видит только вывод генератора (не рассуждения). Оценивает по критериям планировщика. Возвращает прохождение/непрохождение плюс конкретные проблемы.

Ключевое свойство: **каждый агент работает в своём контекстном окне.** Генератор может заполнить своё 200K окно исследованием кода и всё равно выдать чистый вывод. Оценщик начинает с чистого листа. Планировщик поддерживает обзорный вид без деталей реализации.

```python
def three_agent_pipeline(goal, max_replans=2):
    plan = planner.decompose(goal)

    for replan in range(max_replans + 1):
        results = {}
        for subtask in plan.subtasks:
            # Generator: fresh context per subtask
            output = generator.execute(subtask)

            # Evaluator: fresh context, only sees output + criteria
            evaluation = evaluator.judge(
                subtask=subtask,
                output=output,
                criteria=subtask.success_criteria
            )

            results[subtask.id] = {
                "output": output,
                "evaluation": evaluation
            }

        # Check if all subtasks pass
        failures = [r for r in results.values() if not r["evaluation"].passes]
        if not failures:
            return assemble_results(results)

        # Re-plan: planner sees which subtasks failed and why
        plan = planner.replan(goal, results)

    return assemble_results(results)  # best effort
```

## Антипаттерны

### Антипаттерн №1: Монолитный агент

Запихивание планирования, выполнения, оценки и управления контекстом в один агент.

```python
# DON'T DO THIS
response = llm.chat(
    system="""You are a planner, coder, reviewer, and project manager.
    First plan the work, then do the work, then review your own work.
    If the review finds issues, fix them and review again.""",
    messages=conversation  # 150K tokens of accumulated history
)
```

Это падает по всем причинам выше: контекст заполняется, самооценка ненадёжна, нет разделения ответственности. Работает для простых задач, рушится на сложных.

### Антипаттерн №2: Оценка без рубрикатора

```python
# DON'T DO THIS
evaluation = evaluator.judge(
    prompt=f"Is this output good? Rate 1-10.\n\n{output}"
)
# Result: always 8/10. Always.
```

Оценщик без критериев — это просто генератор с синдромом самозванца. Всегда предоставляйте рубрикатор:

```python
# DO THIS
evaluation = evaluator.judge(
    prompt=f"""Evaluate the following output against these criteria:
    1. Does every function have error handling for edge cases?
    2. Are all API calls wrapped in retry logic?
    3. Does the code match the spec in {spec_file}?
    4. Are there any hardcoded values that should be config?

    Output to evaluate:
    {output}

    For each criterion, answer PASS or FAIL with a one-line explanation.""")
```

### Антипаттерн №3: Бесконечное перепланирование

```python
# DON'T DO THIS
while not all_subtasks_pass:
    plan = planner.replan(goal, results)  # loops forever
    results = execute_plan(plan)
```

Всегда ограничивайте итерации. Три неудачных перепланирования — это проблема спецификации, а не выполнения. Покажите это человеку.

## Собираем вместе: минимальная реализация

```python
class LongRunningHarness:
    """Planner → Generator → Evaluator harness for long-running tasks."""

    def __init__(self, planner_model, generator_model, evaluator_model):
        self.planner = Agent(model=planner_model, role="planner")
        self.generator = Agent(model=generator_model, role="generator")
        self.evaluator = Agent(model=evaluator_model, role="evaluator")

    def run(self, goal, max_replans=2, max_gen_iterations=3):
        plan = self.planner.decompose(goal)

        for _ in range(max_replans + 1):
            results = {}

            for subtask in plan.subtasks:
                output = self._generate_with_eval(
                    subtask, max_iterations=max_gen_iterations
                )
                results[subtask.id] = output

            failures = {k: v for k, v in results.items() if not v["passed"]}
            if not failures:
                return self._assemble(results)

            plan = self.planner.replan(goal, plan, failures)

        return self._assemble(results, partial=True)  # best effort

    def _generate_with_eval(self, subtask, max_iterations):
        output = None
        for i in range(max_iterations):
            output = self.generator.execute(
                subtask=subtask,
                prior_feedback=output.get("feedback") if output else None
            )

            evaluation = self.evaluator.judge(
                output=output["result"],
                criteria=subtask.success_criteria
            )

            if evaluation["passes"]:
                return {"result": output["result"], "passed": True}

            output["feedback"] = evaluation["issues"]

        return {"result": output["result"], "passed": False,
                "feedback": evaluation["issues"]}

    def _assemble(self, results, partial=False):
        assembled = "\n\n".join(r["result"] for r in results.values())
        if partial:
            failed = [k for k, v in results.items() if not v["passed"]]
            assembled += f"\n\n⚠️ Incomplete subtasks: {failed}"
        return assembled
```

## Ключевые выводы

1. **Длительные ≠ короткие с большим временем.** Режимы сбоя качественно различны.
2. **Контекстная тревога реальна.** Управляйте жизненным циклом контекста через сбросы, сжатие или оба.
3. **Никогда не позволяйте генератору проверять свой экзамен.** Раздельные агенты, раздельные контексты, явные рубрикаторы.
4. **Ограничивайте всё.** Максимум ходов, максимум перепланирований, максимум итераций. Неограниченные циклы сжирают токены.
5. **Декомпозируйте сначала.** Куски размером с контекст предотвращают большинство контекстных проблем до их возникновения.

## Дополнительные материалы

- [Инженерия контекста](context-engineering.md) — углублённый разбор сборки, сжатия и бюджетирования контекста
- [Оркестрация мультиагентов](multi-agent-orchestration.md) — паттерны оркестрации за пределами трёхагентной архитектуры
- [Обработка ошибок](error-handling.md) — обработка сбоев, повторов и корректной деградации в циклах агентов
- [Anthropic: Building effective agents](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/building-effective-agents) — руководство Anthropic по паттернам проектирования агентов
