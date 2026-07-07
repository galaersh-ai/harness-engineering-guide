<p align="center">
  <a href="https://harness-guide.com">
    <img src="site/public/banner.png" alt="Harness Engineering Guide" width="100%" />
  </a>
</p>

<p align="center">
  <em>Практическое руководство по построению ИИ-агентных харнесов — с реальными примерами кода, которые можно скопировать и запустить.</em>
</p>

<p align="center">
  <a href="https://github.com/nexu-io/harness-engineering-guide/stargazers"><img src="https://img.shields.io/github/stars/nexu-io/harness-engineering-guide?style=social" alt="Stars"></a>
  <a href="https://github.com/nexu-io/harness-engineering-guide/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="License"></a>
</p>

<p align="center">
  🌐 <b><a href="https://harness-guide.com">harness-guide.com</a></b> | <a href="https://harness-guide.com/zh/">中文站</a>
</p>

<p align="center">
  <b>English</b> | <a href="README.zh-CN.md">中文</a> | <a href="README.ru.md">Русский</a>
</p>

---

**Харнес** — это обёртка среды выполнения, которая превращает голую языковую модель в **агента** — автономную систему, способную воспринимать среду, принимать решения и выполнять действия за несколько шагов. Харнес берёт на себя всё, что модель не может сделать сама: выполнение инструментов, управление памятью, сборка контекста и обеспечение границ безопасности.

Это руководство охватывает инженерию харнесов от первых принципов до production-паттернов, с реальным кодом в каждой статье.

---

## Начало работы

| Тема | Описание |
|------|----------|
| [Что такое харнес?](guide/what-is-harness.md) | Концепция за 3 минуты. Как он превращает модель в агента. Харнес vs. фреймворк vs. среда выполнения. |
| [Ваш первый харнес](guide/your-first-harness.md) | Постройте работающий харнес из 50 строк Python. Полный код для копирования и запуска. |
| [Харнес vs. фреймворк](guide/harness-vs-framework.md) | Когда использовать голый харнес vs. LangChain/CrewAI. Дерево решений + сравнение кода. |

## Основные концепции

| Тема | Описание |
|------|----------|
| [Агентный цикл](guide/agentic-loop.md) | Цикл «подумай → действуй → наблюдай». Бюджеты ходов, параллельные вызовы инструментов, обнаружение циклов, стриминг. |
| [Система инструментов](guide/tool-system.md) | Реестр инструментов, статическая vs. динамическая загрузка, протокол MCP, паттерны качества описаний. |
| [Память и контекст](guide/memory-and-context.md) | Сборка контекста, управление сессиями, двухуровневая память (ежедневные логи + долгосрочная). Паттерны AGENTS.md и MEMORY.md. |
| [Защитные ограничения](guide/guardrails.md) | Модели разрешений, границы доверия, песочница, защита от prompt-injection. |

## Практика

| Тема | Описание |
|------|----------|
| [Инженерия контекста](guide/context-engineering.md) | Сборка на основе приоритетов, три линии обороны для сжатия, бюджетирование токенов. |
| [Песочница](guide/sandbox.md) | Настройка Docker и Firecracker, изоляция сети, ограничения файловой системы. |
| [Система навыков](guide/skill-system.md) | Упаковка навыков, загрузка по запросу, формат SKILL.md, тонкий харнес + толстые навыки. |
| [Субагент](guide/sub-agent.md) | Паттерн лидер-исполнитель, файловая коммуникация, изоляция сессий, параллельное выполнение. |
| [Обработка ошибок](guide/error-handling.md) | Классификация ошибок, стратегии повтора, корректная деградация, контрольные точки/возобновление. |
| [Оркестрация мультиагентов](guide/multi-agent-orchestration.md) | Паттерны оркестрации (пайплайн, fan-out, супервайзер), изоляция контекста, реальные примеры (Multica, Paseo, OpenClaw). |
| [Планирование и автоматизация](guide/scheduling-and-automation.md) | Cron, heartbeat'и, триггеры по событиям. Назначение сессий, доставка, сравнение LangSmith vs. нативные харнесы. |
| [Проектирование харнеса для длительных задач](guide/long-running-harness.md) | Контекстная тревога, предвзятость самооценки, сброс vs. сжатие контекста, архитектура генератор-оценщик, вдохновлённая GAN. |
| [Управляемые агенты](guide/managed-agents-architecture.md) | Разделение мозга/рук/сессии, питомцы vs. скот, изоляция учётных данных, улучшения TTFT. |
| [Инфраструктурный шум в оценках](guide/eval-infrastructure.md) | Ресурсные конфигурации влияют на баллы бенчмарков на 6 пп. Стратегия floor+ceiling. |
| [Классификаторные разрешения](guide/classifier-permissions.md) | Замена усталости от одобрения классификаторами на основе моделей. Двухслойная защита, четыре модели угроз, дизайн без учёта рассуждений. |
| [Осведомлённость об оценках](guide/eval-awareness.md) | Когда агенты узнают, что их тестируют. Новый паттерн загрязнения, мультиагентное усиление, защита харнесов. |
| [Команды агентов](guide/agent-teams.md) | 16 параллельных Claude построили 100K-строчный C-компилятор. Ralph-loop, координация через git, бисекция через GCC-оракул. |
| [Паттерн инициализатор + coding-агент](guide/initializer-coding-pattern.md) | Двухфазный харнес для длительных задач. JSON-список функций, стартовый ритуал, коммиты чистого состояния. |

## Справочник

| Тема | Описание |
|------|----------|
| [Сравнение реализаций](guide/comparison.md) | Сравнение OpenClaw, Claude Code, Codex, Cline, Aider, Cursor. |
| [Глоссарий](guide/glossary.md) | Определения ключевых терминов. |

## Шоукейс

| Тема | Описание |
|------|----------|
| [Выпуск нашего Windows-клиента](guide/nexu-windows-packaging.md) | Время сборки 15мин→4мин, время установки 10мин→2мин. Как мы перестроили пайплайн упаковки Electron. |
| [Охота на призрачных аккаунтов](guide/ghost-account-hunting.md) | 1000+ призрачных аккаунтов опустошили нашу платформу за 15 дней. Полный постмортем. |

---

## Как внести вклад

1. Перейдите в [**Issues → New Issue**](https://github.com/nexu-io/harness-engineering-guide/issues/new/choose)
2. Выберите **"📬 Submit a Resource"**
3. Заполните название, URL и почему это релевантно

Или отправьте PR напрямую — см. [CONTRIBUTING.md](CONTRIBUTING.md).

---

## Сообщество

- 💬 **GitHub Discussions** — [Присоединяйтесь к обсуждению](https://github.com/nexu-io/harness-engineering-guide/discussions)
- 🐦 **Twitter** — [@nexudotio](https://x.com/nexudotio)
- 💬 **飞书群** — [加入 Harness Engineering 话题群](https://applink.feishu.cn/client/chat/chatter/add_by_link?link_token=717g465a-0bc8-4242-9281-12b23953491a)

---

## О проекте

Поддерживается [Nexu](https://github.com/nexu-io) — open-source платформой Claude Co-worker & Managed Agent.

## Лицензия

[MIT License](LICENSE)

---

Если Вы觉得 это руководство полезным, пожалуйста, поставьте нам ⭐

```
@misc{nexu_harness-engineering-guide_2026,
  author = {Nexu Team},
  title = {Harness Engineering Guide},
  year = {2026},
  publisher = {GitHub},
  howpublished = {\url{https://github.com/nexu-io/harness-engineering-guide}}
}
```
