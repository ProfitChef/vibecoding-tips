# Настройка Playwright MCP для отладки React-приложений в Claude Code

Этот гайд описывает как настроить автоматический запуск браузера через Playwright MCP для отладки React-приложений прямо из Claude Code.

## Что такое Playwright MCP?

Playwright MCP (Model Context Protocol) - это интеграция, которая позволяет Claude Code управлять браузером Playwright. Вы можете:

- Открывать страницы и навигироваться по приложению
- Получать DOM-дерево (accessibility snapshot) для анализа
- Читать консоль браузера (логи, ошибки, warnings)
- Просматривать сетевые запросы
- Кликать по элементам, заполнять формы
- Делать скриншоты

Это позволяет отлаживать UI без переключения между терминалом и браузером.

## Предварительные требования

1. **Claude Code** установлен и настроен
2. **Playwright MCP** добавлен в настройки Claude Code
3. **React-проект** (Create React App, Vite, Next.js и т.д.)

### Проверка Playwright MCP

Playwright MCP должен быть в списке MCP серверов. Проверить можно командой:
```bash
claude mcp list
```

Если Playwright MCP не установлен, добавьте его через настройки Claude Code.

## Шаг 1: Создание структуры команд

Создайте папку `.claude/commands/` в корне вашего проекта:

```bash
mkdir -p .claude/commands
```

## Шаг 2: Создание команды /start-web

Создайте файл `.claude/commands/start-web.md`:

```markdown
---
allowed-tools: Bash, mcp__playwright__browser_navigate, mcp__playwright__browser_snapshot, mcp__playwright__browser_wait_for, mcp__playwright__browser_console_messages, mcp__playwright__browser_network_requests
description: Запуск dev-сервера и открытие браузера через Playwright MCP
---

# Запуск веб-сервера с Playwright браузером

Запустить development сервер и открыть приложение в Playwright-браузере для отладки.

## Шаги выполнения

### 1. Проверить доступность порта
```bash
netstat -ano | findstr :3001
```
Если порт занят - завершить процесс или использовать другой порт.

### 2. Запустить dev-сервер в фоновом режиме
```bash
BROWSER=none PORT=3001 npm start
```
Запустить в background режиме. `BROWSER=none` отключает автоматическое открытие браузера (используем Playwright вместо этого).

### 3. Подождать запуска сервера
Подождать 8 секунд пока React dev server инициализируется.

### 4. Открыть браузер через Playwright MCP
Перейти на `http://localhost:3001` используя `mcp__playwright__browser_navigate`.

### 5. Получить начальное состояние
- Использовать `mcp__playwright__browser_snapshot` для получения DOM состояния
- Использовать `mcp__playwright__browser_console_messages` для проверки ошибок

### 6. Отчет о статусе
Сообщить пользователю:
- Статус сервера (работает/ошибка)
- Текущая страница
- Ошибки консоли если есть

## Доступные команды для отладки

После запуска можно использовать:
- `browser_snapshot` - Получить DOM состояние (accessibility tree)
- `browser_console_messages` - Получить логи консоли (info, warning, error)
- `browser_network_requests` - Просмотр сетевых запросов
- `browser_click` - Клик по элементам (по ref из snapshot)
- `browser_type` - Ввод текста в поля
- `browser_take_screenshot` - Сделать скриншот
- `browser_wait_for` - Ожидание текста/времени

## Примечания

- Сессия браузера сохраняется до явного закрытия
- Сообщения консоли накапливаются - можно проверить в любой момент
- Используй `browser_wait_for` для ожидания загрузки
```

## Шаг 3: Понимание ключевых параметров

### BROWSER=none

Эта переменная окружения отключает автоматическое открытие браузера в Create React App. Без неё CRA откроет браузер по умолчанию, и у вас будет два открытых браузера.

**Для разных фреймворков:**

| Фреймворк | Команда запуска |
|-----------|-----------------|
| Create React App | `BROWSER=none PORT=3001 npm start` |
| Vite | `BROWSER=none PORT=3001 npm run dev` |
| Next.js | `PORT=3001 npm run dev` (не нужен BROWSER=none) |
| Angular | `ng serve --port 3001 --open=false` |
| Vue CLI | `VUE_CLI_NO_BROWSER=true PORT=3001 npm run serve` |

### Выбор порта

Порт 3001 выбран потому что:
- 3000 часто занят другими проектами
- Легко запомнить (стандартный React порт + 1)
- Можно изменить на любой свободный

### allowed-tools

Список инструментов в frontmatter определяет какие tools Claude Code может использовать при выполнении команды:

```yaml
allowed-tools: Bash, mcp__playwright__browser_navigate, mcp__playwright__browser_snapshot, ...
```

## Шаг 4: Использование команды

После создания файла команда автоматически доступна в Claude Code:

```
/start-web
```

Claude Code:
1. Проверит свободен ли порт
2. Запустит сервер в фоне
3. Подождёт инициализации
4. Откроет Playwright браузер
5. Покажет текущее состояние страницы

## Шаг 5: Отладка через Playwright

### Получение состояния страницы

После запуска можно запросить:

```
Покажи текущее состояние страницы
```

Claude использует `browser_snapshot` и покажет accessibility tree - структурированное представление DOM.

### Проверка ошибок

```
Есть ли ошибки в консоли?
```

Claude использует `browser_console_messages` с level="error".

### Проверка сетевых запросов

```
Покажи сетевые запросы
```

Claude использует `browser_network_requests` и покажет все HTTP запросы.

### Взаимодействие с UI

```
Кликни на кнопку "Login"
```

Claude найдёт элемент в snapshot по тексту и использует `browser_click`.

### Заполнение форм

```
Заполни форму логина: email test@test.com, password 123456
```

Claude использует `browser_type` или `browser_fill_form`.

## Продвинутые настройки

### Добавление в CLAUDE.md

Рекомендуется добавить секцию в CLAUDE.md проекта для документации:

```markdown
## Playwright MCP (Отладка браузера)

### Автоматический запуск браузера

Используй команду `/start-web` для запуска dev-сервера с Playwright браузером.

### Алгоритм выполнения:
1. Проверить свободен ли порт 3001
2. Запустить `BROWSER=none PORT=3001 npm start` в фоновом режиме
3. Подождать 8 секунд
4. Открыть `http://localhost:3001` через Playwright MCP
5. Получить snapshot и console logs

### Доступные инструменты:
- `browser_navigate` - переход по URL
- `browser_snapshot` - получение accessibility tree
- `browser_click` - клики по элементам
- `browser_type` - ввод текста
- `browser_console_messages` - логи консоли
- `browser_network_requests` - сетевые запросы
- `browser_take_screenshot` - скриншот
```

### Кастомные роуты

Добавьте список роутов вашего приложения в CLAUDE.md:

```markdown
### Роуты приложения:
- `/` - главная страница
- `/login` - авторизация
- `/dashboard` - панель управления
- `/users` - пользователи
- `/settings` - настройки
```

Это поможет Claude навигироваться по приложению.

## Типичные проблемы и решения

### Порт занят

```
Error: listen EADDRINUSE: address already in use :::3001
```

**Решение (Windows):**
```bash
netstat -ano | findstr :3001
taskkill /PID <PID> /F
```

**Решение (Linux/Mac):**
```bash
lsof -i :3001
kill -9 <PID>
```

### Браузер не открывается

Проверьте что Playwright MCP установлен:
```bash
claude mcp list
```

### Сервер не успевает запуститься

Увеличьте время ожидания в команде с 8 до 15 секунд для медленных проектов.

### Два браузера открываются

Убедитесь что используете `BROWSER=none` в команде запуска.

## Завершение работы

### Закрыть только браузер Playwright
Claude Code автоматически закроет браузер при завершении сессии, или можно явно попросить:
```
Закрой браузер
```

### Остановить dev-сервер

**Windows:**
```bash
netstat -ano | findstr :3001
taskkill /PID <PID> /F
```

**Linux/Mac:**
```bash
pkill -f "npm start"
# или
lsof -i :3001 | awk 'NR>1 {print $2}' | xargs kill
```

## Полезные сценарии использования

### 1. Отладка стилей
```
Открой страницу /dashboard и покажи snapshot. Найди элемент с классом .header
```

### 2. Проверка API интеграции
```
Открой /users и покажи все сетевые запросы к API
```

### 3. Тестирование формы
```
Перейди на /login, заполни форму тестовыми данными и нажми Submit. Покажи результат
```

### 4. Поиск ошибок
```
Открой приложение и покажи все ошибки консоли
```

### 5. Скриншот для документации
```
Перейди на /dashboard и сделай скриншот
```

---

## Заключение

Playwright MCP интеграция превращает Claude Code в полноценную среду разработки с визуальной отладкой. Вы можете:

- Видеть результат изменений сразу в браузере
- Отлаживать без переключения между окнами
- Автоматизировать рутинные проверки
- Быстро находить ошибки через консоль и сетевые запросы

Команда `/start-web` - это входная точка, но возможности Playwright MCP гораздо шире. Экспериментируйте!
