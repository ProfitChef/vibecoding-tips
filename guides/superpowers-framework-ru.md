# Superpowers Framework: Полный workflow для Claude Code

> Фреймворк от Jesse Vincent, который превращает хаотичную работу агента в дисциплинированный процесс разработки с brainstorming, планированием, TDD и code review.

## Что это такое?

**Superpowers** — это не просто набор скиллов, а полная методология разработки для coding agents. Фреймворк заставляет Claude работать как senior developer: сначала уточнить требования, потом спланировать, потом писать код с тестами, потом ревью.

**GitHub:** https://github.com/obra/superpowers (26k+ stars)
**Блог автора:** https://blog.fsck.com/2025/10/09/superpowers/

## Проблема которую решает

### Без Superpowers

```
Ты: Добавь авторизацию

Claude: *сразу начинает писать код*
        *пишет 500 строк*
        *забыл половину требований*
        *нет тестов*
        *код работает "как-то"*
        *к 5-й задаче забыл что делал в 1-й*
```

### С Superpowers

```
Ты: Добавь авторизацию

Claude: [using brainstorming skill]
        "Прежде чем писать код, уточним:
        - OAuth или email/password?
        - Какие роли нужны?
        - Где хранить сессии?"

        [показывает дизайн по частям]

        [using writing-plans skill]
        [разбивает на 2-5 минутные задачи]

        [using subagent-driven-development]
        [fresh subagent на каждую задачу]
        [spec review + code quality review]

        [using test-driven-development]
        [RED → GREEN → REFACTOR]
```

---

## Архитектура

```
superpowers/
├── .claude-plugin/     # Метаданные плагина
├── skills/             # 14 скиллов (главное)
│   ├── brainstorming/
│   ├── writing-plans/
│   ├── executing-plans/
│   ├── subagent-driven-development/
│   ├── dispatching-parallel-agents/
│   ├── test-driven-development/
│   ├── systematic-debugging/
│   ├── requesting-code-review/
│   ├── receiving-code-review/
│   ├── using-git-worktrees/
│   ├── finishing-a-development-branch/
│   ├── verification-before-completion/
│   ├── writing-skills/
│   └── using-superpowers/          # Мета-скилл
├── agents/             # Subagent definitions
├── commands/           # Slash commands
├── hooks/              # Event handlers
└── tests/              # Тесты для скиллов
```

---

## Полный workflow

```
┌─────────────────────────────────────────────────────────────────┐
│  1. BRAINSTORMING                                               │
│                                                                 │
│  Claude НЕ пишет код сразу!                                    │
│  → Задаёт уточняющие вопросы                                   │
│  → Исследует альтернативы                                      │
│  → Показывает дизайн по частям для approval                    │
│  → Сохраняет design document                                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓ (твой approval)
┌─────────────────────────────────────────────────────────────────┐
│  2. USING-GIT-WORKTREES                                        │
│                                                                 │
│  → Создаёт изолированный worktree (.worktrees/feature-name)    │
│  → Новая ветка для фичи                                        │
│  → npm install / cargo build / etc                             │
│  → Проверяет что тесты зелёные (baseline)                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  3. WRITING-PLANS                                              │
│                                                                 │
│  → Разбивает работу на bite-sized задачи (2-5 мин каждая)      │
│  → Каждая задача имеет:                                        │
│    • Exact file paths                                          │
│    • Complete code snippets                                    │
│    • Verification steps                                        │
│  → План понятен "junior engineer без контекста"                │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  4. EXECUTION (один из вариантов)                              │
│                                                                 │
│  A) subagent-driven-development:                               │
│     → Fresh subagent на каждую задачу                          │
│     → Spec review + Code quality review                        │
│     → Автономно, без human checkpoints                         │
│                                                                 │
│  B) executing-plans:                                           │
│     → Batch execution (по 3 задачи)                            │
│     → Human checkpoint между батчами                           │
│                                                                 │
│  C) dispatching-parallel-agents:                               │
│     → Параллельные subagents для независимых задач             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  5. TEST-DRIVEN-DEVELOPMENT (во время execution)               │
│                                                                 │
│  RED:    Написать failing test                                 │
│  GREEN:  Написать минимальный код чтобы тест прошёл           │
│  REFACTOR: Улучшить код, тесты остаются зелёными              │
│  COMMIT                                                        │
│                                                                 │
│  ⚠️ Если код написан ДО теста — УДАЛИТЬ и начать заново       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  6. CODE REVIEW                                                │
│                                                                 │
│  → Review против плана и требований                            │
│  → Issues по severity (Critical/Important/Minor)               │
│  → Critical issues блокируют progress                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  7. FINISHING-A-DEVELOPMENT-BRANCH                             │
│                                                                 │
│  → Verify все тесты зелёные                                    │
│  → Опции: merge / PR / keep / discard                          │
│  → Cleanup worktree                                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Все 14 скиллов

### Process Skills (как подходить к задаче)

| Скилл | Описание | Когда срабатывает |
|-------|----------|-------------------|
| `brainstorming` | Socratic design refinement | Перед написанием кода |
| `writing-plans` | Детальные implementation plans | После approval дизайна |
| `executing-plans` | Batch execution с checkpoints | Есть план, нужен human control |
| `subagent-driven-development` | Fresh subagent + двойное ревью | Есть план, автономная работа |
| `dispatching-parallel-agents` | Параллельные subagents | 3+ независимых задач |

### Implementation Skills (как делать)

| Скилл | Описание | Когда срабатывает |
|-------|----------|-------------------|
| `test-driven-development` | RED-GREEN-REFACTOR | Во время написания кода |
| `systematic-debugging` | 4-фазный root cause process | Баг, нужен анализ |
| `verification-before-completion` | Проверка перед "готово" | Перед завершением задачи |

### Collaboration Skills

| Скилл | Описание | Когда срабатывает |
|-------|----------|-------------------|
| `requesting-code-review` | Pre-review checklist | Между задачами / перед merge |
| `receiving-code-review` | Responding to feedback | Получил feedback |
| `using-git-worktrees` | Изолированные workspaces | Нужна изоляция |
| `finishing-a-development-branch` | Merge/PR decision | Задачи выполнены |

### Meta Skills

| Скилл | Описание | Когда срабатывает |
|-------|----------|-------------------|
| `using-superpowers` | Мета-скилл, управляет всеми | Начало любой сессии |
| `writing-skills` | Создание новых скиллов | Когда нужен новый скилл |

---

## Ключевая магия: using-superpowers

Это мета-скилл который инжектится через `SessionStart` hook. Он говорит Claude:

```markdown
<EXTREMELY-IMPORTANT>
Если есть хотя бы 1% шанс что скилл применим —
ТЫ ОБЯЗАН его использовать.

ЭТО НЕ ОПЦИОНАЛЬНО. ЭТО НЕ ОБСУЖДАЕТСЯ.
</EXTREMELY-IMPORTANT>
```

### Red flags которые ловит

| Мысль агента | Реальность |
|--------------|------------|
| "Это просто вопрос" | Вопросы = задачи. Проверь скиллы. |
| "Мне нужно больше контекста" | Проверка скиллов ПЕРЕД уточнениями |
| "Давай сначала посмотрю код" | Скиллы говорят КАК смотреть |
| "Скилл overkill для этого" | Простое становится сложным. Используй. |
| "Я помню этот скилл" | Скиллы меняются. Прочитай текущую версию. |
| "Сначала соберу информацию" | Скиллы говорят КАК собирать |
| "Это не считается задачей" | Действие = задача. Проверь скиллы. |

---

## Subagent-Driven Development (подробнее)

Самый мощный скилл для автономной работы.

### Процесс

```
Для каждой задачи из плана:

1. Dispatch Implementer Subagent
   → Полный текст задачи (не ссылка на файл!)
   → Контекст: где это в архитектуре
   → Может задавать вопросы перед началом

2. Implementer работает
   → Реализует задачу
   → Пишет тесты (TDD)
   → Коммитит
   → Self-review

3. Dispatch Spec Reviewer
   → "Построено ли ТО, что просили?"
   → НЕ доверяет отчёту implementer'а
   → Читает код напрямую
   → Ловит: missing requirements, extra work, misunderstandings

4. Если spec reviewer нашёл issues
   → Implementer фиксит
   → Re-review
   → Repeat до ✅

5. Dispatch Code Quality Reviewer
   → "Качественно ли построено?"
   → Strengths, Issues (Critical/Important/Minor)
   → Assessment

6. Если quality reviewer нашёл issues
   → Implementer фиксит
   → Re-review
   → Repeat до ✅

7. Mark task complete
```

### Почему это работает

| Проблема | Решение |
|----------|---------|
| Context pollution к 5-й задаче | Fresh subagent на каждую задачу |
| "Сделал быстро" = пропустил требования | Spec reviewer не доверяет отчёту |
| Over-engineering | Spec reviewer ловит лишнее |
| Низкое качество | Code quality reviewer |
| Баги проскакивают | Два этапа ревью |

---

## Установка

### Claude Code

```bash
# Добавить marketplace
/plugin marketplace add obra/superpowers-marketplace

# Установить плагин
/plugin install superpowers@superpowers-marketplace

# Проверить
/help
```

После установки появятся команды:
- `/superpowers:brainstorm` — интерактивный дизайн
- `/superpowers:write-plan` — создание плана
- `/superpowers:execute-plan` — выполнение плана

### Обновление

```bash
/plugin update superpowers
```

---

## Философия фреймворка

1. **Test-Driven Development** — тесты сначала, всегда
2. **Systematic over ad-hoc** — процесс важнее угадывания
3. **Complexity reduction** — простота как главная цель
4. **Evidence over claims** — проверяй перед "готово"
5. **YAGNI** — You Aren't Gonna Need It (не строй лишнего)
6. **DRY** — Don't Repeat Yourself

---

## Когда использовать?

### Да, если:

- Работаешь над сложными фичами (не quick fixes)
- Хочешь чтобы Claude работал автономно часами
- Важно качество кода и тесты
- Устал от "сделал быстро, но криво"
- Проект с тестами или планируешь их добавить

### Может быть overkill, если:

- Только quick fixes и мелкие правки
- Проект без тестов и не планируешь добавлять
- Нужна максимальная скорость, качество вторично
- Разовые скрипты и эксперименты

---

## Пример сессии

```
Ты: Добавь user authentication

Claude: I'm using the brainstorming skill to refine this design.

Before we start, let me understand:
1. OAuth, email/password, or both?
2. What roles do you need? (admin, user, guest?)
3. Session storage: JWT, cookies, or database sessions?
4. Do you need 2FA?

Ты: Email/password, admin and user roles, JWT

Claude: [показывает дизайн по частям]

## Authentication Design

### Part 1: Data Model
- User table with email, password_hash, role
- ...

Does this look correct? [approval needed]

Ты: да

Claude: I'm using the using-git-worktrees skill.

Creating worktree at .worktrees/auth...
Running npm install...
Running tests... 47 passing ✓

Baseline verified. Ready to implement.

I'm using the writing-plans skill.

## Implementation Plan

### Task 1: Create User model (3 min)
- File: src/models/User.ts
- Add: User interface, validation schema
- Test: src/models/User.test.ts
- Verify: npm test -- User.test.ts

### Task 2: Password hashing service (2 min)
...

[5 tasks total]

Ты: go

Claude: I'm using the subagent-driven-development skill.

═══ Task 1: Create User model ═══

[Dispatching implementer subagent]

Implementer: Implemented User model with validation.
Tests: 5/5 passing. Committed.

[Dispatching spec reviewer]

Spec reviewer: ✅ Spec compliant. All requirements met.

[Dispatching code quality reviewer]

Quality reviewer:
Strengths: Clean types, good validation
Issues: None
✅ Approved

[Task 1 complete]

═══ Task 2: Password hashing ═══
...

[После всех задач]

I'm using the finishing-a-development-branch skill.

All tests passing (23 tests).
Options:
1. Merge to main
2. Create PR
3. Keep branch for review
4. Discard

Ты: 2

Claude: Created PR #42: "Add user authentication"
```

---

## Troubleshooting

### Скиллы не срабатывают автоматически

1. Убедитесь что плагин установлен: `/plugin list`
2. Попробуйте явно вызвать: `/superpowers:brainstorm`
3. Проверьте что `using-superpowers` скилл активен

### Claude пропускает этапы

Добавьте в промпт:
```
Use the full superpowers workflow: brainstorming → plan → execute with TDD
```

### Слишком много вопросов в brainstorming

Это нормально для сложных задач. Для простых можно:
```
Skip brainstorming, I know exactly what I want: [детальное описание]
```

---

## Связанные гайды

- [QA Agent Setup](qa-agent-setup-ru.md) — автоматический QA
- [Session Analyzer](session-analyzer-agent-ru.md) — анализ сессий
- [Playwright MCP](playwright-mcp-browser-debugging-ru.md) — отладка в браузере
