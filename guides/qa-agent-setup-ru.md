# Создание глобального QA-агента с Playwright MCP

Подробный гайд по созданию автоматического QA-агента, который верифицирует UI после каждого изменения кода.

## Содержание

1. [Концепция](#концепция)
2. [Архитектура](#архитектура)
3. [Пошаговая установка](#пошаговая-установка)
4. [Полные промпты агентов](#полные-промпты-агентов)
5. [Hook скрипт](#hook-скрипт)
6. [Конфигурация проекта](#конфигурация-проекта)
7. [Команды](#команды)
8. [Тестирование](#тестирование)
9. [Troubleshooting](#troubleshooting)

---

## Концепция

### Что делает QA-агент

После редактирования UI файла (`.js`, `.jsx`, `.css` и т.д.):

1. **Hook** ловит событие Edit/Write
2. **Скрипт** фильтрует (только UI файлы) и ждёт паузу в редактировании
3. **qa-quick (Sonnet)** быстро проверяет:
   - Console errors
   - Network failures
   - DOM snapshot
4. **Если проблемы** → эскалация на **qa-deep (Opus)**
5. **Отчёт** с рекомендациями по исправлению

### Почему два агента?

| Агент | Модель | Когда используется | Стоимость |
|-------|--------|-------------------|-----------|
| qa-quick | Sonnet | Всегда (90% случаев) | Низкая |
| qa-deep | Opus | Только при проблемах (10%) | Высокая |

Экономия: Opus запускается только когда действительно нужен глубокий анализ.

### Принципы безопасности

1. **READ-ONLY** — агенты только наблюдают, не взаимодействуют
2. **Только отчёт** — рекомендации без автофикса
3. **Нет бесконечных циклов** — агент не может редактировать код

---

## Архитектура

```
~/.claude/                          # Глобально (для всех проектов)
├── agents/
│   ├── qa-quick.md                 # Быстрая проверка (Sonnet)
│   ├── qa-deep.md                  # Глубокий анализ (Opus)
│   └── qa-setup.md                 # Авто-создание конфига
├── scripts/
│   └── qa-hook.js                  # PostToolUse hook
├── commands/
│   ├── qa-verify.md                # /qa-verify
│   └── qa-setup.md                 # /qa-setup
├── templates/
│   └── qa-config.template.json     # Шаблон конфига
└── settings.json                   # Глобальный hook

<project>/.claude/                  # Локально (для конкретного проекта)
└── qa-config.json                  # Порт, роуты, фильтры
```

### Как работает адаптация

1. Hook скрипт проверяет `<project>/.claude/qa-config.json`
2. Если есть → использует настройки проекта
3. Если нет → defaults + предложение запустить `/qa-setup`
4. `/qa-setup` анализирует проект и создаёт конфиг автоматически

---

## Пошаговая установка

### Шаг 1: Создать директории

```bash
mkdir -p ~/.claude/agents
mkdir -p ~/.claude/scripts
mkdir -p ~/.claude/commands
mkdir -p ~/.claude/templates
```

### Шаг 2: Создать агентов

Создайте файлы агентов (полные промпты ниже в разделе [Полные промпты агентов](#полные-промпты-агентов)).

### Шаг 3: Создать hook скрипт

Создайте `~/.claude/scripts/qa-hook.js` (полный код ниже).

### Шаг 4: Обновить settings.json

Добавьте в `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.claude/scripts/qa-hook.js",
            "timeout": 10000
          }
        ]
      }
    ]
  }
}
```

### Шаг 5: Создать конфиг проекта

В каждом проекте создайте `.claude/qa-config.json` или запустите `/qa-setup`.

---

## Полные промпты агентов

### qa-quick.md (Sonnet)

Создайте файл `~/.claude/agents/qa-quick.md`:

```markdown
---
name: qa-quick
description: Fast QA check after UI changes. Runs automatically via PostToolUse hook. Checks console, network, DOM. Escalates to qa-deep if issues found. Adapts to project config.
tools: mcp__playwright__browser_snapshot, mcp__playwright__browser_console_messages, mcp__playwright__browser_network_requests, mcp__playwright__browser_navigate, mcp__playwright__browser_wait_for, Read, Task
model: sonnet
---

# QA Quick Check Agent (Adaptive)

You are a fast QA checker. Your job: quick verification, escalate if problems.
You adapt to each project by reading its local config.

## Step 0: Load Project Config

First, try to read `.claude/qa-config.json` in the current project directory.
If it exists, use its settings. If not, use defaults:

**Defaults:**
- Port: 3000
- No file-to-route mapping (check current page)
- Standard noise filters

## Pre-check (ALWAYS FIRST)

1. Try `browser_snapshot`
   - If fails: try `browser_navigate` to http://localhost:{port}
   - If still fails: report "SERVER_DOWN" and stop

2. Check current URL from snapshot
   - If contains "/login" or "/signin": report "NEEDS_LOGIN" and stop

3. If config has fileToRoute mapping and you know the changed file:
   - Navigate to the mapped route

## Quick Check

1. `browser_console_messages` level="error"
   - Filter out (from config or defaults):
     - chrome-extension://
     - React DevTools
     - [HMR]
     - WebSocket connection
     - Download the React DevTools

2. `browser_network_requests`
   - Look for: 4xx, 5xx errors
   - Ignore successful requests (2xx, 304)
   - Ignore webpack/hot-update requests

3. `browser_snapshot`
   - Check for error boundaries
   - Check for "undefined" or "null" text in UI
   - Check for missing key elements
   - Check for "Error" or "Something went wrong" text

## Decision

**IF all clean:**
```
QA Status: PASS
No issues detected.
```

**IF issues found:**
```
QA Status: ISSUES DETECTED
Escalating to deep analysis...
```
Then call Task tool with subagent_type="qa-deep" for root cause analysis.
Pass: error details, changed file path, project config.

## CRITICAL RULES
- Be FAST - basic checks only
- Filter noise aggressively
- Escalate to qa-deep for complex issues
- You are READ-ONLY - never click, type, or edit files
- Adapt to project config when available
```

### qa-deep.md (Opus)

Создайте файл `~/.claude/agents/qa-deep.md`:

```markdown
---
name: qa-deep
description: Deep QA analysis for complex issues. Called by qa-quick when problems detected. Analyzes root cause, reads code, provides detailed fix recommendations. Adapts to project structure.
tools: mcp__playwright__browser_snapshot, mcp__playwright__browser_console_messages, mcp__playwright__browser_network_requests, mcp__playwright__browser_navigate, Read, Glob, Grep
model: opus
---

# QA Deep Analysis Agent (Adaptive)

You are an expert QA analyst. You're called when qa-quick found issues.
You adapt to each project's structure and conventions.

## Your Mission

Analyze the issue deeply and provide:
1. Root cause analysis
2. Connection between code change and error
3. Specific fix recommendations

## Step 0: Understand Project Context

Try to read `.claude/qa-config.json` for project-specific info:
- File to route mappings
- Project structure hints
- Framework (React, Vue, Next.js, etc.)

Also check for common config files to understand the stack:
- `package.json` - dependencies, scripts
- `tsconfig.json` - TypeScript project
- `vite.config.js` - Vite project
- `next.config.js` - Next.js project

## Deep Analysis Workflow

1. **Understand the error**
   - Read full error message and stack trace from console
   - Identify which component/function failed
   - Note the exact error type (TypeError, ReferenceError, etc.)

2. **Read the changed code**
   - Use Read tool to see the file that was modified
   - Understand what changed recently
   - Look for obvious mistakes (typos, missing imports, wrong props)

3. **Trace the connection**
   - How does the change relate to the error?
   - Is it direct cause or side effect?
   - Check the component hierarchy

4. **Check related code**
   - Look at imports and dependencies
   - Check if change broke a contract/interface
   - Verify prop types and data flow

5. **Formulate fix strategy**
   - What exactly needs to change?
   - Are there multiple valid approaches?
   - What's the safest fix?

## Report Format

### Deep QA Analysis

**Project:** [detected framework/stack]

**Error Summary:**
[One line description of the error]

**Root Cause:**
[Detailed explanation of why this happened]

**Code Connection:**
[How the change in {file} caused this error]

**Stack Trace Analysis:**
```
[Key parts of stack trace with explanation]
```

**Recommended Fix:**
```javascript
// Specific code suggestion
// Show exactly what to change
```

**Alternative Approaches:**
1. [Option A with pros/cons]
2. [Option B with pros/cons]

**Prevention:**
[How to avoid this type of error in future]

---

## CRITICAL RULES
- Be THOROUGH - this is deep analysis
- Read actual code, don't guess
- Provide SPECIFIC fix recommendations with code examples
- Explain WHY the fix works
- You are READ-ONLY - recommend, never fix directly
- Always include file paths and line numbers when possible
- Adapt analysis to the project's framework and conventions
```

### qa-setup.md (Sonnet)

Создайте файл `~/.claude/agents/qa-setup.md`:

```markdown
---
name: qa-setup
description: Analyzes project structure and creates .claude/qa-config.json. Called automatically when config is missing. Detects framework, routes, port, and project conventions.
tools: Read, Glob, Grep, Bash, Write
model: sonnet
---

# QA Setup Agent

You analyze projects and create QA configuration automatically.
Called when `.claude/qa-config.json` doesn't exist.

## Your Mission

1. Analyze project structure thoroughly
2. Detect framework and conventions
3. Find all routes/pages
4. Determine dev server port
5. Create `.claude/qa-config.json`

## Analysis Workflow

### Step 1: Detect Framework

Check for config files:
```
package.json → dependencies (react, vue, angular, next, nuxt, svelte)
vite.config.js → Vite project
next.config.js → Next.js
nuxt.config.js → Nuxt
angular.json → Angular
```

Read `package.json` scripts to find:
- Dev server command (start, dev, serve)
- Port configuration

### Step 2: Find Dev Server Port

Check in order:
1. `package.json` scripts (look for PORT=, --port)
2. `.env` or `.env.local` (PORT=, VITE_PORT=)
3. Framework defaults:
   - Create React App: 3000
   - Vite: 5173
   - Next.js: 3000
   - Angular: 4200
   - Vue CLI: 8080

### Step 3: Map Files to Routes

**React/CRA/Vite React:**
- Look in `src/pages/`, `src/routes/`, `src/views/`
- Check router config: `App.js`, `routes.js`, `router.js`
- Pattern: `src/pages/DashboardPage/` → `/dashboard`

**Next.js:**
- `pages/` or `app/` directory = routes
- `pages/dashboard.js` → `/dashboard`
- `app/dashboard/page.js` → `/dashboard`

**Vue:**
- Check `src/router/index.js`
- Look in `src/views/`

### Step 4: Identify Noise Patterns

Standard filters for all projects:
```json
{
  "consoleIgnore": [
    "chrome-extension://",
    "React DevTools",
    "[HMR]",
    "WebSocket connection",
    "Download the React DevTools",
    "Manifest:",
    "favicon.ico",
    "[vite]",
    "hmr"
  ]
}
```

### Step 5: Create Config

Create `.claude/` directory if needed, then write `.claude/qa-config.json`:

```json
{
  "framework": "react|next|vue|angular|svelte",
  "serverPort": 3000,
  "fileToRoute": {
    "src/pages/ExamplePage/": "/example"
  },
  "ignorePatterns": [
    "*.test.js",
    "*.spec.js",
    "node_modules"
  ],
  "noiseFilters": {
    "consoleIgnore": [...],
    "networkIgnoreStatus": [200, 201, 204, 304]
  },
  "batchDelayMs": 3000
}
```

## Output

After creating config, report:

```
QA Config Created: .claude/qa-config.json

Framework: [detected]
Port: [port]
Routes found: [count]

File-to-Route Mapping:
- src/pages/Dashboard/ → /dashboard
- src/pages/Users/ → /users
...

You can customize this config as needed.
```

## CRITICAL RULES
- Be THOROUGH in analysis
- Check multiple sources for port
- Map ALL page directories to routes
- Create the config file using Write tool
- Report what was detected and created
```

---

## Hook скрипт

Создайте файл `~/.claude/scripts/qa-hook.js`:

```javascript
#!/usr/bin/env node
/**
 * Global QA Hook Script (Adaptive)
 *
 * Features:
 * - Detects project-local .claude/qa-config.json
 * - Falls back to defaults if no config
 * - Triggers qa-setup agent to create config if missing
 * - Filters UI file changes
 * - Implements batch mode (waits for editing pause)
 * - Checks server availability
 */

const fs = require('fs');
const net = require('net');
const path = require('path');

// Get project directory from environment or current working directory
const PROJECT_DIR = process.env.CLAUDE_PROJECT_DIR || process.cwd();
const CONFIG_PATH = path.join(PROJECT_DIR, '.claude', 'qa-config.json');
const BATCH_FILE = path.join(process.env.TEMP || '/tmp', 'claude-qa-batch.json');

// Default configuration (used when no project config exists)
const DEFAULT_CONFIG = {
  serverPort: 3000,
  batchDelayMs: 3000,
  ignorePatterns: [
    '.test.js', '.spec.js', '.test.jsx', '.spec.jsx',
    '.test.ts', '.spec.ts', '.test.tsx', '.spec.tsx',
    'node_modules', 'setupTests'
  ],
  uiFilePatterns: ['.js', '.jsx', '.ts', '.tsx', '.css', '.scss', '.vue', '.svelte']
};

// Load project config or use defaults
function loadConfig() {
  try {
    if (fs.existsSync(CONFIG_PATH)) {
      const projectConfig = JSON.parse(fs.readFileSync(CONFIG_PATH, 'utf8'));
      return { ...DEFAULT_CONFIG, ...projectConfig, hasLocalConfig: true };
    }
  } catch (e) {
    // Config exists but invalid - will trigger setup
  }
  return { ...DEFAULT_CONFIG, hasLocalConfig: false };
}

// Check if file matches UI patterns
function isUIFile(filePath, config) {
  const patterns = config.uiFilePatterns || DEFAULT_CONFIG.uiFilePatterns;
  return patterns.some(ext => filePath.endsWith(ext));
}

// Check if file should be ignored
function shouldIgnore(filePath, config) {
  const patterns = config.ignorePatterns || DEFAULT_CONFIG.ignorePatterns;
  return patterns.some(pattern => filePath.includes(pattern));
}

// Check server availability
function checkServer(port) {
  return new Promise(resolve => {
    const socket = new net.Socket();
    socket.setTimeout(1000);
    socket.on('connect', () => { socket.destroy(); resolve(true); });
    socket.on('error', () => resolve(false));
    socket.on('timeout', () => { socket.destroy(); resolve(false); });
    socket.connect(port, 'localhost');
  });
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// Main logic
let input = '';
process.stdin.on('data', chunk => input += chunk);
process.stdin.on('end', async () => {
  try {
    const data = JSON.parse(input);
    const filePath = data.tool_input?.file_path || '';
    const config = loadConfig();

    // Filter: only UI files, not ignored
    if (!isUIFile(filePath, config) || shouldIgnore(filePath, config)) {
      process.exit(0);
    }

    // Batch mode: collect files
    let batch = { files: [], timestamp: Date.now() };
    if (fs.existsSync(BATCH_FILE)) {
      try {
        batch = JSON.parse(fs.readFileSync(BATCH_FILE, 'utf8'));
      } catch (e) {}
    }

    if (!batch.files.includes(filePath)) {
      batch.files.push(filePath);
    }
    batch.timestamp = Date.now();
    fs.writeFileSync(BATCH_FILE, JSON.stringify(batch));

    // Wait for editing pause
    const batchDelay = config.batchDelayMs || DEFAULT_CONFIG.batchDelayMs;
    await sleep(batchDelay);

    // Re-check batch
    if (!fs.existsSync(BATCH_FILE)) {
      process.exit(0);
    }

    const currentBatch = JSON.parse(fs.readFileSync(BATCH_FILE, 'utf8'));
    if (Date.now() - currentBatch.timestamp < batchDelay - 100) {
      process.exit(0); // Still editing
    }

    // Check server
    const port = config.serverPort || DEFAULT_CONFIG.serverPort;
    const serverUp = await checkServer(port);

    if (!serverUp) {
      try { fs.unlinkSync(BATCH_FILE); } catch (e) {}
      process.exit(0); // Silent skip if server down
    }

    // Output for Claude
    console.log('[QA] UI files changed:');
    currentBatch.files.forEach(f => {
      const relativePath = f.replace(/.*[\\\/]src[\\\/]/, 'src/').replace(/\\/g, '/');
      console.log(`  - ${relativePath}`);
    });

    // If no local config, suggest creating one
    if (!config.hasLocalConfig) {
      console.log('[QA] No project config found. Run /qa-setup to create .claude/qa-config.json');
      console.log('[QA] Using defaults (port ' + port + ')');
    }

    console.log('[QA] Run qa-quick agent to verify changes.');

    // Clear batch
    try { fs.unlinkSync(BATCH_FILE); } catch (e) {}
    process.exit(0);

  } catch (e) {
    process.exit(0);
  }
});
```

---

## Конфигурация проекта

### Шаблон qa-config.json

Создайте `~/.claude/templates/qa-config.template.json`:

```json
{
  "_comment": "QA Agent Configuration - copy to .claude/qa-config.json",

  "framework": "react",
  "serverPort": 3000,

  "fileToRoute": {
    "src/pages/HomePage/": "/",
    "src/pages/DashboardPage/": "/dashboard",
    "src/pages/LoginPage/": "/login",
    "src/pages/UsersPage/": "/users",
    "src/components/": null
  },

  "ignorePatterns": [
    "*.test.js",
    "*.spec.js",
    "*.test.jsx",
    "*.spec.jsx",
    "*.test.ts",
    "*.spec.ts",
    "*.test.tsx",
    "*.spec.tsx",
    "node_modules",
    "setupTests",
    "__tests__",
    "__mocks__"
  ],

  "noiseFilters": {
    "consoleIgnore": [
      "chrome-extension://",
      "React DevTools",
      "[HMR]",
      "WebSocket connection",
      "Download the React DevTools",
      "Manifest:",
      "favicon.ico",
      "[vite]",
      "hmr update"
    ],
    "networkIgnoreStatus": [200, 201, 204, 304],
    "networkIgnoreUrls": [
      "chrome-extension://",
      "webpack",
      "hot-update",
      ".map",
      "sockjs-node"
    ]
  },

  "batchDelayMs": 3000
}
```

### Примеры конфигов для разных фреймворков

**Create React App:**
```json
{
  "framework": "react",
  "serverPort": 3000,
  "fileToRoute": {
    "src/pages/": null
  }
}
```

**Vite React:**
```json
{
  "framework": "vite-react",
  "serverPort": 5173,
  "fileToRoute": {
    "src/pages/": null
  }
}
```

**Next.js (Pages Router):**
```json
{
  "framework": "nextjs",
  "serverPort": 3000,
  "fileToRoute": {
    "pages/dashboard": "/dashboard",
    "pages/users/": "/users",
    "components/": null
  }
}
```

**Next.js (App Router):**
```json
{
  "framework": "nextjs-app",
  "serverPort": 3000,
  "fileToRoute": {
    "app/dashboard/": "/dashboard",
    "app/users/": "/users",
    "components/": null
  }
}
```

---

## Команды

### /qa-verify

Создайте `~/.claude/commands/qa-verify.md`:

```markdown
---
allowed-tools: Task
description: Manually trigger QA verification on current UI state
---

# Manual QA Verification

Run qa-quick agent to verify current UI state.

## When to Use

- Check UI without making code changes
- Hook didn't trigger (server was down)
- Verify specific page
- After login to check dashboard

## Steps

1. Use Task tool with subagent_type="qa-quick"
2. Include in prompt:
   - Current page to check
   - Any specific concerns
3. Report results

## Example Prompts

**Basic check:**
```
Verify current UI state. Check console for errors, network for failed requests, and DOM for issues.
```

**Specific page:**
```
Navigate to /dashboard and verify UI state.
```
```

### /qa-setup

Создайте `~/.claude/commands/qa-setup.md`:

```markdown
---
allowed-tools: Task
description: Analyze project and create .claude/qa-config.json
---

# QA Setup

Run qa-setup agent to analyze project and create configuration.

## When to Use

- New project without config
- Regenerate config after changes
- Adding QA to existing project

## Steps

1. Use Task tool with subagent_type="qa-setup"
2. Prompt: "Analyze this project and create .claude/qa-config.json"
3. Review created config
4. Customize if needed
```

---

## Тестирование

### Проверить установку

1. Убедитесь что файлы созданы:
```bash
ls ~/.claude/agents/
# qa-quick.md  qa-deep.md  qa-setup.md

ls ~/.claude/scripts/
# qa-hook.js

ls ~/.claude/commands/
# qa-verify.md  qa-setup.md
```

2. Проверьте settings.json:
```bash
cat ~/.claude/settings.json | grep -A 10 "hooks"
```

### Тест с реальным проектом

1. Откройте проект в Claude Code
2. Запустите dev server
3. Отредактируйте любой UI файл
4. Подождите 3 секунды
5. Должно появиться сообщение `[QA] UI files changed...`
6. Claude запустит qa-quick

### Ручная проверка

```
/qa-verify
```

---

## Troubleshooting

### Hook не срабатывает

1. Проверьте что hooks добавлены в `~/.claude/settings.json`
2. Перезапустите Claude Code (hooks читаются при старте)
3. Проверьте что файл - UI файл (.js, .jsx, .css)
4. Проверьте что не тестовый файл (.test.js)

### SERVER_DOWN

1. Запустите dev server
2. Проверьте порт в `.claude/qa-config.json`
3. Убедитесь что сервер доступен на localhost

### NEEDS_LOGIN

1. Залогиньтесь в приложении в Playwright браузере
2. Или добавьте тестовые credentials в агента

### Слишком много проверок

Увеличьте `batchDelayMs` в конфиге:
```json
{
  "batchDelayMs": 5000
}
```

### Ложные срабатывания (шум)

Добавьте паттерны в `noiseFilters.consoleIgnore`:
```json
{
  "noiseFilters": {
    "consoleIgnore": [
      "ваш-паттерн",
      ...
    ]
  }
}
```

---

## Итого

Вы создали глобального QA-агента который:

1. **Автоматически** проверяет UI после каждого изменения
2. **Адаптируется** к каждому проекту через локальный конфиг
3. **Экономит ресурсы** - Opus только для сложных случаев
4. **Безопасен** - READ-ONLY, без автофикса
5. **Масштабируется** - работает с любым проектом

Команды:
- `/qa-verify` — ручная проверка
- `/qa-setup` — создать конфиг для нового проекта
