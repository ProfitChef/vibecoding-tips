# Звуковые уведомления для Claude Code (Windows)

Полный гайд по настройке аудио-оповещений когда Claude Code требует вашего внимания.

## Зачем это нужно

Когда работаешь с Claude Code в фоне (переключился в браузер, другое окно), легко пропустить момент когда:
- Claude закончил работу и ждёт ввода
- Нужно разрешение на действие
- Произошла ошибка

Звуковые уведомления решают эту проблему — ты сразу слышишь что пора вернуться в терминал.

## Обзор решения

Claude Code поддерживает **хуки** — кастомные команды, которые выполняются на определённые события. Мы используем их для проигрывания системных звуков Windows:

| Событие | Звук | Когда срабатывает |
|---------|------|-------------------|
| **Stop** | `tada.wav` | Claude закончил отвечать, ждёт ввода |
| **SubagentStop** | `Windows Battery Critical.wav` | Фоновый агент (Task) завершился |
| **PermissionRequest** | `Windows Error.wav` | Claude нужно разрешение |
| **PostToolUse** (ошибка) | `Windows Critical Stop.wav` | Инструмент завершился с ошибкой |

---

## Быстрая настройка

### 1. Создай скрипт обнаружения ошибок

Создай файл `~/.claude/scripts/error-sound.js`:

```javascript
/**
 * Проигрывает звук ошибки при неудачном выполнении инструмента
 * Используется как PostToolUse хук
 */
const { execSync } = require('child_process');

let input = '';
process.stdin.setEncoding('utf8');
process.stdin.on('data', chunk => input += chunk);
process.stdin.on('end', () => {
  try {
    const data = JSON.parse(input);

    // Проверяем индикаторы ошибки
    const hasError =
      data.tool_error ||
      (data.tool_result && (
        data.tool_result.includes('Error:') ||
        data.tool_result.includes('error:') ||
        data.tool_result.includes('ENOENT') ||
        data.tool_result.includes('Permission denied') ||
        data.tool_result.includes('failed') ||
        data.tool_result.includes('FAILED')
      ));

    if (hasError) {
      execSync(`powershell -NoProfile -Command "(New-Object System.Media.SoundPlayer 'C:\\Windows\\Media\\Windows Critical Stop.wav').PlaySync()"`, {
        stdio: 'ignore'
      });
    }
  } catch (e) {
    // Игнорируем ошибки парсинга
  }
});
```

### 2. Добавь хуки в settings.json

Добавь в `~/.claude/settings.json` (объедини с существующими хуками если есть):

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "powershell -NoProfile -Command \"(New-Object System.Media.SoundPlayer 'C:\\Windows\\Media\\tada.wav').PlaySync()\"",
            "timeout": 3000
          }
        ]
      }
    ],
    "SubagentStop": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "powershell -NoProfile -Command \"(New-Object System.Media.SoundPlayer 'C:\\Windows\\Media\\Windows Battery Critical.wav').PlaySync()\"",
            "timeout": 3000
          }
        ]
      }
    ],
    "PermissionRequest": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "powershell -NoProfile -Command \"(New-Object System.Media.SoundPlayer 'C:\\Windows\\Media\\Windows Error.wav').PlaySync()\"",
            "timeout": 3000
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.claude/scripts/error-sound.js",
            "timeout": 3000
          }
        ]
      }
    ]
  }
}
```

### 3. Перезапусти Claude Code

Закрой и открой Claude Code заново — хуки загружаются при старте.

---

## Доступные системные звуки Windows

Расположены в `C:\Windows\Media\`:

| Файл | Описание | Подходит для |
|------|----------|--------------|
| `tada.wav` | Весёлая фанфара | Завершение задачи |
| `chimes.wav` | Приятный перезвон | Уведомления |
| `chord.wav` | Простой аккорд | Общие алерты |
| `ding.wav` | Одиночный динь | Быстрое уведомление |
| `notify.wav` | Мягкое уведомление | Ненавязчивые алерты |
| `Ring01.wav` | Звонок телефона | Нужно внимание |
| `Ring02.wav` | Другой звонок | Альтернативный звонок |
| `Alarm01.wav` | Будильник | Срочные алерты |
| `Alarm02.wav` | Другой будильник | Альтернативный |
| `Windows Error.wav` | Звук ошибки | Запросы разрешений |
| `Windows Critical Stop.wav` | Критическая ошибка | Сбои |
| `Windows Exclamation.wav` | Предупреждение | Предупреждения |
| `Windows Notify.wav` | Системное уведомление | Общее |
| `Windows Battery Critical.wav` | Батарея разряжена | Фоновые задачи |
| `Windows Unlock.wav` | Разблокировка | Старт сессии |

---

## Все доступные хуки Claude Code

| Хук | Когда срабатывает | Пример использования |
|-----|-------------------|----------------------|
| **Stop** | Claude закончил отвечать | Звук завершения |
| **SubagentStop** | Task агент завершился | Уведомление о фоновой работе |
| **PermissionRequest** | Показан диалог разрешения | Алерт "нужно внимание" |
| **PreToolUse** | Перед выполнением инструмента | Блокировка опасных команд |
| **PostToolUse** | После выполнения инструмента | Проверка ошибок, логирование |
| **Notification** | Системные уведомления | Кастомные алерты |
| **UserPromptSubmit** | Пользователь отправил сообщение | Валидация ввода |
| **PreCompact** | Перед сжатием контекста | Сохранение важной инфы |
| **SessionStart** | Начало сессии | Настройка окружения |
| **SessionEnd** | Конец сессии | Cleanup, логирование |

### Матчеры для Notification

Для хука `Notification` можно использовать специфичные матчеры:

```json
{
  "Notification": [
    {
      "matcher": "permission_prompt",
      "hooks": [{ "type": "command", "command": "..." }]
    },
    {
      "matcher": "idle_prompt",
      "hooks": [{ "type": "command", "command": "..." }]
    }
  ]
}
```

- `permission_prompt` — Запросы разрешений
- `idle_prompt` — Claude ждёт 60+ секунд
- `auth_success` — Успешная авторизация
- `""` или пропустить — Все уведомления

---

## Примеры кастомизации

### Разные звуки для разных инструментов

```json
{
  "PostToolUse": [
    {
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "powershell -NoProfile -Command \"(New-Object System.Media.SoundPlayer 'C:\\Windows\\Media\\ding.wav').PlaySync()\""
      }]
    },
    {
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "powershell -NoProfile -Command \"(New-Object System.Media.SoundPlayer 'C:\\Windows\\Media\\chimes.wav').PlaySync()\""
      }]
    }
  ]
}
```

### Использование кастомных WAV файлов

```json
{
  "Stop": [{
    "matcher": "*",
    "hooks": [{
      "type": "command",
      "command": "powershell -NoProfile -Command \"(New-Object System.Media.SoundPlayer 'D:\\Sounds\\my-custom-sound.wav').PlaySync()\""
    }]
  }]
}
```

### Простой beep (без WAV файла)

```json
{
  "Stop": [{
    "matcher": "*",
    "hooks": [{
      "type": "command",
      "command": "powershell -NoProfile -Command \"[System.Console]::Beep(1000, 500)\""
    }]
  }]
}
```

Параметры: `Beep(частота_hz, длительность_ms)`

### Паттерн из нескольких beep'ов

```json
{
  "PermissionRequest": [{
    "matcher": "*",
    "hooks": [{
      "type": "command",
      "command": "powershell -NoProfile -Command \"[System.Console]::Beep(800, 200); Start-Sleep -m 100; [System.Console]::Beep(1000, 200); Start-Sleep -m 100; [System.Console]::Beep(1200, 200)\""
    }]
  }]
}
```

---

## Устранение неполадок

### Звук не проигрывается

1. **Проверь что файл существует:**
   ```powershell
   Test-Path "C:\Windows\Media\tada.wav"
   ```

2. **Протестируй звук вручную:**
   ```powershell
   (New-Object System.Media.SoundPlayer 'C:\Windows\Media\tada.wav').PlaySync()
   ```

3. **Проверь громкость системы** — звуки Windows могут быть заглушены

4. **Проверь синтаксис settings.json** — используй JSON валидатор

### Хук не срабатывает

1. **Перезапусти Claude Code** — хуки загружаются при старте

2. **Проверь имя хука** — должно быть точным: `Stop`, не `stop` или `STOP`

3. **Протестируй с простой командой:**
   ```json
   {
     "Stop": [{
       "matcher": "*",
       "hooks": [{
         "type": "command",
         "command": "echo Hook fired >> C:\\Users\\PC\\hook-test.log"
       }]
     }]
   }
   ```

4. **Проверь логи Claude Code** на ошибки хуков

### Звук играет слишком часто

- Используй более специфичный `matcher` вместо `"*"`
- Добавь условия в скрипт (как в error-sound.js)

### Звук задерживает ответ

- Уменьши значение `timeout`
- `PlaySync()` блокирует, `Play()` нет
- Для неблокирующего: используй `Start-Process`

---

## macOS / Linux

### macOS

```json
{
  "Stop": [{
    "matcher": "*",
    "hooks": [{
      "type": "command",
      "command": "afplay /System/Library/Sounds/Glass.aiff"
    }]
  }]
}
```

Доступные звуки: `/System/Library/Sounds/` (Basso, Blow, Bottle, Frog, Funk, Glass, Hero, Morse, Ping, Pop, Purr, Sosumi, Submarine, Tink)

### Linux

```json
{
  "Stop": [{
    "matcher": "*",
    "hooks": [{
      "type": "command",
      "command": "paplay /usr/share/sounds/freedesktop/stereo/complete.oga"
    }]
  }]
}
```

Или с `beep`:
```json
{
  "command": "beep -f 1000 -l 500"
}
```

---

## Структура файлов

После настройки у тебя должно быть:

```
~/.claude/
├── settings.json              # Конфигурация хуков
└── scripts/
    └── error-sound.js         # Скрипт обнаружения ошибок
```

---

## Быстрый промпт для Claude

Чтобы настроить это, дай Claude такой промпт:

```
Настрой звуковые уведомления для Claude Code на Windows:
- tada.wav когда закончил работу (Stop)
- Windows Battery Critical.wav когда субагент завершился (SubagentStop)
- Windows Error.wav когда нужно разрешение (PermissionRequest)
- Windows Critical Stop.wav при ошибках (PostToolUse)

Создай скрипт error-sound.js и обнови settings.json.
Объедини с существующими хуками если есть.
```
