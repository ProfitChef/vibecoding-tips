# Session Analyzer Agent: Мета-анализ сессий Claude Code

> Пошаговый гайд по созданию агента, который анализирует ваши сессии работы с Claude Code, находит паттерны и предлагает автоматизации.

## Что делает этот агент?

Session Analyzer — это мета-агент, который:

1. **Собирает данные** о каждом tool call, промпте и запуске агентов (0 токенов)
2. **Анализирует паттерны** — частые команды, grep patterns, повторяющиеся задачи
3. **Семантически группирует промпты** — находит похожие запросы и предлагает команды
4. **Отслеживает делегирование** — находит тяжёлые задачи, которые можно вынести в агенты
5. **Мониторит агентов** — success rate, время выполнения, ошибки
6. **Считает эффективность** — токены, tool calls, тренды

## Архитектура

```
┌─────────────────────────────────────────────────────────────┐
│           Layer 1: Data Collection (Hooks, 0 токенов)       │
├─────────────────────────────────────────────────────────────┤
│  PostToolUse → session-collector.js → session-data.jsonl    │
│  UserPromptSubmit → session-collector.js → user-prompts.jsonl│
│  SessionEnd  → session-collector.js → sessions-summary.jsonl│
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│           Layer 2: Analysis Agent (LLM, по триггеру)        │
├─────────────────────────────────────────────────────────────┤
│  session-analyzer agent                                     │
│  - Читает собранные данные                                  │
│  - Анализирует паттерны                                     │
│  - Генерирует рекомендации                                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│           Layer 3: Output                                   │
├─────────────────────────────────────────────────────────────┤
│  - Markdown отчет                                           │
│  - Предложения команд и агентов                             │
│  - Quick Actions checklist                                  │
└─────────────────────────────────────────────────────────────┘
```

## Структура файлов

После настройки у вас будет:

```
~/.claude/
├── hooks/
│   └── session-collector.js      # Hook для сбора данных
├── session-data/
│   ├── session-data.jsonl        # Все tool calls
│   ├── sessions-summary.jsonl    # Summary сессий
│   ├── agent-runs.jsonl          # Запуски агентов
│   ├── user-prompts.jsonl        # Промпты пользователя
│   └── token-tracker.json        # Счётчик токенов
├── agents/
│   └── session-analyzer.md       # Агент анализа
└── commands/
    └── analyze-sessions.md       # Команда /analyze-sessions
```

---

## Шаг 1: Создание директории для данных

```bash
mkdir -p ~/.claude/session-data ~/.claude/hooks
```

---

## Шаг 2: Создание hook-скрипта

Создайте файл `~/.claude/hooks/session-collector.js`:

```javascript
#!/usr/bin/env node
/**
 * Session Collector Hook
 * Collects data for session analysis: tool calls, prompts, agent runs, tokens
 *
 * Usage:
 *   PostToolUse: node session-collector.js < hook_input.json
 *   UserPromptSubmit: node session-collector.js --prompt < hook_input.json
 *   SessionEnd: node session-collector.js --end < hook_input.json
 */

const fs = require('fs');
const path = require('path');
const os = require('os');
const readline = require('readline');

// Paths
const CLAUDE_DIR = path.join(os.homedir(), '.claude');
const DATA_DIR = path.join(CLAUDE_DIR, 'session-data');
const SESSION_DATA_FILE = path.join(DATA_DIR, 'session-data.jsonl');
const SESSIONS_SUMMARY_FILE = path.join(DATA_DIR, 'sessions-summary.jsonl');
const AGENT_RUNS_FILE = path.join(DATA_DIR, 'agent-runs.jsonl');
const USER_PROMPTS_FILE = path.join(DATA_DIR, 'user-prompts.jsonl');
const TOKEN_TRACKER_FILE = path.join(DATA_DIR, 'token-tracker.json');

// Token threshold for analysis suggestion (настройте под себя)
const TOKEN_THRESHOLD = 500000;

// Ensure data directory exists
if (!fs.existsSync(DATA_DIR)) {
  fs.mkdirSync(DATA_DIR, { recursive: true });
}

/**
 * Append JSON line to file
 */
function appendJsonl(filePath, data) {
  fs.appendFileSync(filePath, JSON.stringify(data) + '\n');
}

/**
 * Read JSON file safely
 */
function readJsonFile(filePath, defaultValue = {}) {
  try {
    if (fs.existsSync(filePath)) {
      return JSON.parse(fs.readFileSync(filePath, 'utf8'));
    }
  } catch (e) {
    // Ignore parse errors
  }
  return defaultValue;
}

/**
 * Write JSON file
 */
function writeJsonFile(filePath, data) {
  fs.writeFileSync(filePath, JSON.stringify(data, null, 2));
}

/**
 * Parse transcript file to extract token usage
 */
function parseTranscriptForTokens(transcriptPath) {
  let totalTokens = 0;

  try {
    if (!fs.existsSync(transcriptPath)) {
      return 0;
    }

    const content = fs.readFileSync(transcriptPath, 'utf8');
    const lines = content.split('\n').filter(line => line.trim());

    for (const line of lines) {
      try {
        const entry = JSON.parse(line);
        // Look for usage field in API responses
        if (entry.usage) {
          totalTokens += (entry.usage.input_tokens || 0) + (entry.usage.output_tokens || 0);
        }
        // Also check for nested usage in message
        if (entry.message?.usage) {
          totalTokens += (entry.message.usage.input_tokens || 0) + (entry.message.usage.output_tokens || 0);
        }
      } catch (e) {
        // Skip invalid JSON lines
      }
    }
  } catch (e) {
    // Ignore file read errors
  }

  return totalTokens;
}

/**
 * Extract project name from cwd
 */
function extractProjectName(cwd) {
  if (!cwd) return 'unknown';
  return path.basename(cwd);
}

/**
 * Handle PostToolUse event
 */
function handlePostToolUse(input) {
  const timestamp = new Date().toISOString();
  const sessionId = input.session_id || 'unknown';
  const project = extractProjectName(input.cwd);
  const toolName = input.tool_name;
  const toolInput = input.tool_input || {};

  // Log tool call
  const toolData = {
    timestamp,
    session_id: sessionId,
    project,
    tool: toolName,
    params: toolInput,
    success: !input.tool_response?.error
  };

  // Extract specific data for different tools
  if (toolName === 'Bash' && toolInput.command) {
    toolData.bash_command = toolInput.command;
  }
  if (toolName === 'Grep' && toolInput.pattern) {
    toolData.grep_pattern = toolInput.pattern;
  }
  if (toolName === 'Glob' && toolInput.pattern) {
    toolData.glob_pattern = toolInput.pattern;
  }

  appendJsonl(SESSION_DATA_FILE, toolData);

  // Track agent runs (Task tool with subagent_type)
  if (toolName === 'Task' && toolInput.subagent_type) {
    const agentData = {
      timestamp,
      session_id: sessionId,
      agent: toolInput.subagent_type,
      success: !input.tool_response?.error,
      error: input.tool_response?.error || null
    };
    appendJsonl(AGENT_RUNS_FILE, agentData);
  }

  // Update token tracker
  if (input.transcript_path) {
    const tracker = readJsonFile(TOKEN_TRACKER_FILE, {
      current_session: null,
      tokens_total: 0,
      threshold_reached: false
    });

    // Reset if new session
    if (tracker.current_session !== sessionId) {
      tracker.current_session = sessionId;
      tracker.tokens_total = 0;
      tracker.threshold_reached = false;
    }

    // Parse tokens from transcript
    const currentTokens = parseTranscriptForTokens(input.transcript_path);
    tracker.tokens_total = currentTokens;
    tracker.last_check = timestamp;

    // Check threshold
    if (currentTokens >= TOKEN_THRESHOLD && !tracker.threshold_reached) {
      tracker.threshold_reached = true;
      // Output suggestion to stdout (visible to user)
      console.log(`\n[Session Analyzer] ${Math.round(currentTokens / 1000)}K tokens spent. Run /analyze-sessions for insights.\n`);
    }

    writeJsonFile(TOKEN_TRACKER_FILE, tracker);
  }
}

/**
 * Handle UserPromptSubmit event
 */
function handleUserPrompt(input) {
  const timestamp = new Date().toISOString();
  const sessionId = input.session_id || 'unknown';
  const project = extractProjectName(input.cwd);
  const prompt = input.prompt || '';

  if (prompt.trim()) {
    const promptData = {
      timestamp,
      session_id: sessionId,
      project,
      prompt: prompt.substring(0, 500) // Limit length
    };
    appendJsonl(USER_PROMPTS_FILE, promptData);
  }
}

/**
 * Handle SessionEnd event
 */
function handleSessionEnd(input) {
  const timestamp = new Date().toISOString();
  const sessionId = input.session_id || 'unknown';
  const project = extractProjectName(input.cwd);

  // Read session data to compile summary
  let toolCalls = 0;
  const uniqueTools = new Set();
  const bashCommands = {};
  const grepPatterns = {};
  let errors = 0;

  try {
    if (fs.existsSync(SESSION_DATA_FILE)) {
      const content = fs.readFileSync(SESSION_DATA_FILE, 'utf8');
      const lines = content.split('\n').filter(line => line.trim());

      for (const line of lines) {
        try {
          const entry = JSON.parse(line);
          if (entry.session_id === sessionId) {
            toolCalls++;
            uniqueTools.add(entry.tool);

            if (!entry.success) errors++;

            if (entry.bash_command) {
              // Normalize command (take first part)
              const cmdKey = entry.bash_command.split(' ')[0];
              bashCommands[cmdKey] = (bashCommands[cmdKey] || 0) + 1;
            }
            if (entry.grep_pattern) {
              grepPatterns[entry.grep_pattern] = (grepPatterns[entry.grep_pattern] || 0) + 1;
            }
          }
        } catch (e) {
          // Skip invalid lines
        }
      }
    }
  } catch (e) {
    // Ignore errors
  }

  // Get token count
  const tracker = readJsonFile(TOKEN_TRACKER_FILE, { tokens_total: 0 });

  const summary = {
    session_id: sessionId,
    project,
    end_time: timestamp,
    tokens_total: tracker.tokens_total,
    tool_calls: toolCalls,
    unique_tools: Array.from(uniqueTools),
    bash_commands: bashCommands,
    grep_patterns: grepPatterns,
    errors,
    reason: input.reason || 'unknown'
  };

  appendJsonl(SESSIONS_SUMMARY_FILE, summary);

  // Reset token tracker
  writeJsonFile(TOKEN_TRACKER_FILE, {
    current_session: null,
    tokens_total: 0,
    threshold_reached: false
  });
}

/**
 * Main entry point
 */
async function main() {
  const args = process.argv.slice(2);
  const isSessionEnd = args.includes('--end');
  const isPrompt = args.includes('--prompt');

  // Read JSON from stdin
  let inputData = '';

  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
    terminal: false
  });

  for await (const line of rl) {
    inputData += line;
  }

  let input = {};
  try {
    input = JSON.parse(inputData);
  } catch (e) {
    // If no valid JSON, exit silently
    process.exit(0);
  }

  try {
    if (isSessionEnd) {
      handleSessionEnd(input);
    } else if (isPrompt) {
      handleUserPrompt(input);
    } else {
      handlePostToolUse(input);
    }
  } catch (e) {
    // Silent failure - don't break the session
    fs.appendFileSync(path.join(DATA_DIR, 'errors.log'),
      `${new Date().toISOString()} - ${e.message}\n${e.stack}\n\n`);
  }

  process.exit(0);
}

main();
```

Сделайте файл исполняемым (Linux/macOS):

```bash
chmod +x ~/.claude/hooks/session-collector.js
```

---

## Шаг 3: Настройка hooks в settings.json

Откройте `~/.claude/settings.json` и добавьте hooks:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.claude/hooks/session-collector.js",
            "timeout": 5000
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.claude/hooks/session-collector.js --prompt",
            "timeout": 3000
          }
        ]
      }
    ],
    "SessionEnd": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.claude/hooks/session-collector.js --end",
            "timeout": 5000
          }
        ]
      }
    ]
  }
}
```

> **Важно**: Если у вас уже есть другие hooks в PostToolUse, добавьте новый объект в массив, а не заменяйте существующие.

---

## Шаг 4: Создание агента-анализатора

Создайте файл `~/.claude/agents/session-analyzer.md`:

````markdown
---
name: session-analyzer
description: Analyzes Claude Code session data to find patterns, suggest automations, and monitor agent performance
tools: Read, Glob, Grep, Bash
model: sonnet
---

# Session Analyzer Agent

You are a meta-analysis agent that examines Claude Code usage patterns across sessions. Your goal is to help optimize the development workflow by identifying automation opportunities and improving existing agents.

## Data Sources

All data is stored in `~/.claude/session-data/`:

1. **session-data.jsonl** - All tool calls with parameters
2. **sessions-summary.jsonl** - Session summaries with aggregated stats
3. **agent-runs.jsonl** - Agent (subagent) execution logs
4. **user-prompts.jsonl** - User prompts for semantic analysis
5. **token-tracker.json** - Current session token count

## Analysis Tasks

When invoked, perform ALL of the following analyses:

### 1. Semantic Prompt Analysis

Read `user-prompts.jsonl` and:
- Group similar prompts by semantic meaning (not exact match)
- Identify recurring task types (e.g., "TypeScript fixes", "API debugging", "UI tweaks")
- For each category with 3+ occurrences, suggest a slash command

Output format:
```
## 1. Semantic Prompt Analysis

### Recurring Task Types
| Category | Count | Example Prompts |
|----------|-------|-----------------|
| ... | ... | ... |

### Command Suggestions
- **Create `/command-name`** - N similar prompts about X
```

### 2. Delegation Opportunities

Read `session-data.jsonl` and find tasks that:
- Had >20 tool calls in sequence
- Show repetitive patterns (Read → Edit → Read → Edit)
- Took significant time

For each, suggest an agent that could handle this task type.

### 3. Agent Performance

Read `agent-runs.jsonl` and analyze:
- Success/failure rates per agent
- Escalation patterns (e.g., qa-quick → qa-deep)
- Timeout or error patterns

### 4. Pattern Analysis

Read `session-data.jsonl` and identify:
- Most frequent bash commands
- Most frequent grep/glob patterns
- Most frequently edited files

### 5. Efficiency Metrics

Read `sessions-summary.jsonl` and calculate:
- Average tokens per session
- Average tool calls per session
- Error rate trend
- Compare with previous period if data available

### 6. Quick Actions

At the end, provide a checklist of actionable items.

## Important Guidelines

1. **Be specific** - Don't suggest vague improvements, give concrete examples
2. **Be practical** - Only suggest automations that would save real time
3. **Show evidence** - Include actual data from logs to support suggestions
4. **Prioritize** - Most impactful suggestions first
5. **Generate code** - When suggesting a command or agent, provide the actual markdown file content

## Parameters

The command may include parameters:
- `--days N` - Analyze last N days only (filter by timestamp)
- `--project NAME` - Filter by project name
- `--focus patterns|delegation|agents|efficiency|all` - Focus on specific section

If no parameters, analyze all available data with all sections.
````

---

## Шаг 5: Создание slash-команды

Создайте файл `~/.claude/commands/analyze-sessions.md`:

```markdown
---
allowed-tools: Task
description: Analyze Claude Code sessions for patterns and optimization opportunities
---

# Analyze Sessions

Run the session-analyzer agent to analyze collected session data and provide insights.

## What it does

1. **Semantic Prompt Analysis** - Groups similar prompts, suggests commands
2. **Delegation Opportunities** - Finds heavy tasks that could be agents
3. **Agent Performance** - Monitors existing agents for issues
4. **Pattern Analysis** - Identifies frequent commands and patterns
5. **Efficiency Metrics** - Tracks tokens, tool calls, error rates

## Parameters

$ARGUMENTS

Parse the following optional parameters:
- `--days N` - Analyze last N days only (default: 7)
- `--project NAME` - Filter by specific project
- `--focus X` - Focus on: patterns, delegation, agents, efficiency, or all (default: all)

## Execution

Use the Task tool to invoke the session-analyzer agent:

Task tool with:
  subagent_type: "session-analyzer"
  prompt: "Analyze session data with parameters: [parsed parameters].
           Read data from ~/.claude/session-data/ and generate a complete analysis report."

After the agent returns, display the full report to the user.
```

---

## Шаг 6: Проверка установки

Проверьте что все файлы на месте:

```bash
ls -la ~/.claude/hooks/session-collector.js
ls -la ~/.claude/agents/session-analyzer.md
ls -la ~/.claude/commands/analyze-sessions.md
ls -la ~/.claude/session-data/
```

Протестируйте hook вручную:

```bash
# Тест PostToolUse
echo '{"session_id":"test","cwd":"/test","tool_name":"Bash","tool_input":{"command":"git status"}}' | node ~/.claude/hooks/session-collector.js

# Проверка что данные записались
cat ~/.claude/session-data/session-data.jsonl
```

---

## Использование

### Сбор данных

Данные начнут собираться автоматически с момента настройки hooks. Каждый tool call, промпт и завершение сессии будут логироваться.

### Триггер по токенам

Когда вы потратите 500K токенов в сессии (настраивается в `TOKEN_THRESHOLD`), hook выведет напоминание:

```
[Session Analyzer] 523K tokens spent. Run /analyze-sessions for insights.
```

### Запуск анализа

```bash
# Полный анализ
/analyze-sessions

# Анализ за последние 3 дня
/analyze-sessions --days 3

# Только для конкретного проекта
/analyze-sessions --project my-app

# Только паттерны
/analyze-sessions --focus patterns
```

---

## Пример вывода агента

```markdown
# Session Analysis Report
Period: Dec 21-28, 2025 | Sessions: 12 | Tokens: 1.2M | Project: tsfc_admin_panel

---

## 1. Semantic Prompt Analysis

### Recurring Task Types
| Category | Count | Example Prompts |
|----------|-------|-----------------|
| TypeScript fixes | 5 | "Fix TS errors", "Resolve type issues" |
| API debugging | 4 | "Debug authFetch", "Fix 401 error" |
| UI tweaks | 3 | "Adjust button style", "Fix modal" |

### Command Suggestions
- **Create `/fix-types`** - 5 similar prompts about TypeScript
- **Create `/debug-api`** - 4 prompts about API issues

---

## 2. Delegation Opportunities

### Heavy Task: "Refactor UserDetailsPage" (Dec 25)
- Tool calls: 67
- Duration: 52 min
- Pattern: Read → Grep → Read → Edit (×15 iterations)

**Suggested Agent: `page-refactorer`**
```yaml
name: page-refactorer
tools: Read, Grep, Edit, Bash
model: sonnet
---
Refactors a React page component systematically
```

---

## 3. Agent Performance

| Agent | Runs | Success | Issues |
|-------|------|---------|--------|
| qa-quick | 23 | 87% | 3 escalations to qa-deep |
| qa-deep | 5 | 100% | - |
| explore | 41 | 93% | 3 timeouts |

### Recommendations
1. **qa-quick → qa-deep escalation 13%** - Consider adding more DOM checks

---

## 4. Frequent Patterns

### Bash Commands
| Command | Count | Suggestion |
|---------|-------|------------|
| git | 45 | Already optimized |
| npm | 23 | Create /run-tests |
| node | 12 | - |

### Grep Patterns
- `authFetch` - 12 times
- `useState` - 8 times

---

## 5. Efficiency Metrics

| Metric | This Week | Prev Week | Trend |
|--------|-----------|-----------|-------|
| Avg session | 42 min | 38 min | ↑ |
| Tokens/session | 100K | 115K | ↓ Better |
| Tool calls | 89 | 102 | ↓ Better |
| Error rate | 3.2% | 5.1% | ↓ Better |

---

## Quick Actions

1. [ ] Create `/fix-types` command
2. [ ] Create `/debug-api` command
3. [ ] Generate `page-refactorer` agent
4. [ ] Update qa-quick with additional checks
```

---

## Кастомизация

### Изменение порога токенов

В файле `session-collector.js` измените константу:

```javascript
const TOKEN_THRESHOLD = 500000; // Измените на нужное значение
```

### Добавление своих метрик

В функции `handlePostToolUse` добавьте логику для отслеживания специфичных инструментов:

```javascript
if (toolName === 'YourTool' && toolInput.someParam) {
  toolData.your_metric = toolInput.someParam;
}
```

### Фильтрация данных

В функции `handleSessionEnd` можно добавить фильтры для summary:

```javascript
// Исключить короткие сессии
if (toolCalls < 10) return;
```

---

## Оценка стоимости

| Компонент | Токены | Частота |
|-----------|--------|---------|
| Data collection | 0 | Каждый tool call |
| Full analysis | ~10,000-15,000 | По запросу |

**Ожидаемая стоимость**: ~$0.15-0.25 за анализ (Sonnet)

При триггере 500K токенов = **~2-3% overhead**

---

## Troubleshooting

### Hook не работает

1. Проверьте синтаксис JSON в `settings.json`
2. Убедитесь что Node.js установлен: `node --version`
3. Проверьте права на файл: `ls -la ~/.claude/hooks/session-collector.js`

### Данные не записываются

1. Проверьте что директория существует: `ls ~/.claude/session-data/`
2. Посмотрите лог ошибок: `cat ~/.claude/session-data/errors.log`

### Агент не находит данные

1. Убедитесь что путь правильный (на Windows это может быть `C:\Users\USERNAME\.claude\`)
2. Проверьте что jsonl файлы не пустые

---

## Что дальше?

После накопления данных за несколько сессий агент начнёт находить реальные паттерны. Рекомендации:

1. **Поработайте неделю** с включённым сбором данных
2. **Запустите `/analyze-sessions`** и посмотрите отчёт
3. **Создайте предложенные команды** — это сэкономит время
4. **Итерируйте** — добавляйте свои метрики по мере необходимости

---

## Связанные гайды

- [QA Agent Setup](qa-agent-setup-ru.md) — автоматический QA с Playwright
- [Playwright MCP Browser Debugging](playwright-mcp-browser-debugging-ru.md) — отладка в браузере
