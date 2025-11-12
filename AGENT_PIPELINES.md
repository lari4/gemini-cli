# Agent Pipelines Documentation

Этот документ описывает все возможные схемы работы агента в Gemini CLI, включая пайплайны, последовательности вызовов промтов и передачу данных между компонентами.

---

## 1. Main Agent Pipeline (Основной рабочий процесс)

**Назначение:** Основной пайплайн обработки пользовательских запросов.

**Компоненты:**
- Router (Classifier Strategy)
- Model Selection (Flash/Pro)
- Main Agent (с Main System Prompt)
- Tools (edit, read, write, shell, etc.)

**Схема:**
```
┌─────────────────┐
│  User Request   │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Router (Classifier Strategy)                           │
│  Prompt: Task Router Classifier Prompt                  │
│  Input: User request + последние 4 сообщения            │
│  Output: { model_choice: "flash" | "pro", reasoning }   │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Model Selection                                        │
│  Flash: gemini-2.0-flash (простые задачи)              │
│  Pro: gemini-2.5-pro (сложные задачи)                  │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Main Agent                                             │
│  Prompt: Main System Prompt                            │
│  - Preamble                                             │
│  - Core Mandates                                        │
│  - Primary Workflows                                    │
│  - Operational Guidelines                               │
│  - Sandbox Configuration                                │
│  - Git Repository Guidelines                            │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Agent decides action based on workflow:                │
│  1. Understand (grep, glob, read_file)                  │
│  2. Plan (internal or with write_todos)                 │
│  3. Implement (edit, write_file, shell)                 │
│  4. Verify (shell: tests)                               │
│  5. Verify (shell: lint, build)                         │
│  6. Finalize                                            │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Tool Execution                                         │
│  - Tools execute with user confirmation if needed       │
│  - Results returned to agent                            │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Loop Detection Service (каждую итерацию)               │
│  - Tool call repetition detection                       │
│  - Content loop detection                               │
│  - LLM-based loop detection (after 30+ turns)           │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Response Generation                                    │
│  - Agent formulates response                            │
│  - Next Speaker Checker determines continuation         │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│  User Response  │
│  or Continue    │
└─────────────────┘
```

**Передача данных:**
1. User Request → Classifier: Текст запроса + история (последние 4 сообщения без tool calls)
2. Classifier → Model Selection: `{ model_choice: "flash" | "pro", reasoning }`
3. Model → Tools: Function calls с параметрами
4. Tools → Model: Results (может быть суммаризован)
5. Loop Detector: Анализирует каждое tool call и content streaming

---

## 2. Codebase Investigation Pipeline

**Назначение:** Глубокое исследование кодовой базы для понимания архитектуры, поиска багов или планирования имплементации.

**Когда активируется:**
- Сложные задачи требующие обширного анализа кода
- Main Agent вызывает инструмент `codebase_investigator`
- Типичные сценарии: рефакторинг, поиск причин багов, анализ зависимостей

**Компоненты:**
- Main Agent
- Codebase Investigator Agent (субагент)
- Read-only tools (ls, read_file, glob, grep)

**Схема:**
```
┌─────────────────────────────────────────────────────────┐
│  Main Agent                                             │
│  Определяет что задача требует глубокого исследования   │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Invoke: codebase_investigator tool                     │
│  Input: { objective: "detailed description" }           │
│  - Оригинальный запрос пользователя                    │
│  - Дополнительный контекст и вопросы от Main Agent      │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Codebase Investigator Agent (субагент)                 │
│  System Prompt: Codebase Investigator Prompt            │
│  Query Prompt: Investigation task with objective        │
│                                                         │
│  Возможности:                                           │
│  - Максимум 15 итераций, 5 минут                       │
│  - Температура 0.1 (детерминизм)                       │
│  - Только read-only tools                              │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Investigation Loop (до 15 итераций)                    │
│                                                         │
│  Каждая итерация:                                       │
│  1. Update <scratchpad>                                 │
│     - Checklist of investigation goals                  │
│     - Questions to Resolve                              │
│     - Key Findings                                      │
│     - Irrelevant Paths to Ignore                        │
│                                                         │
│  2. Use tools to gather information:                    │
│     - ls: explore directory structure                   │
│     - grep: search for symbols/patterns                 │
│     - glob: find files by patterns                      │
│     - read_file: understand implementations             │
│     - web_fetch: research libraries/concepts            │
│                                                         │
│  3. Analyze findings:                                   │
│     - Understand WHY code is written this way           │
│     - Trace dependencies and call chains                │
│     - Identify ripple effects of potential changes      │
│     - Answer all questions in "Questions to Resolve"    │
│                                                         │
│  4. Check termination criteria:                         │
│     - Questions to Resolve list is empty?               │
│     - All relevant files identified?                    │
│     - Complete understanding achieved?                  │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Generate Investigation Report                          │
│  Format: JSON                                           │
│  {                                                      │
│    "SummaryOfFindings": "Root cause analysis...",      │
│    "ExplorationTrace": [                               │
│      "Used grep to search for X",                      │
│      "Read file Y to understand Z"                     │
│    ],                                                  │
│    "RelevantLocations": [                              │
│      {                                                 │
│        "FilePath": "src/foo.ts",                       │
│        "Reasoning": "Contains the bug...",             │
│        "KeySymbols": ["funcA", "classB"]               │
│      }                                                 │
│    ]                                                   │
│  }                                                     │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Return to Main Agent                                   │
│  Main Agent receives full investigation report          │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Main Agent uses findings for implementation            │
│  - Reads relevant files identified in report            │
│  - Makes changes based on insights                      │
│  - Considers ripple effects mentioned in report         │
└─────────────────────────────────────────────────────────┘
```

**Передача данных:**
1. Main Agent → Investigator: `{ objective: string }`
   - Оригинальный запрос пользователя
   - Вопросы/контекст от Main Agent

2. Investigator → Tools: Параметры для grep, read_file, etc.

3. Tools → Investigator: Результаты чтения/поиска

4. Investigator → Main Agent: JSON report
   ```json
   {
     "SummaryOfFindings": "Детальный анализ...",
     "ExplorationTrace": ["Step 1", "Step 2", ...],
     "RelevantLocations": [{FilePath, Reasoning, KeySymbols}, ...]
   }
   ```

5. Main Agent использует report для планирования и имплементации

---

## 3. Edit Correction Pipeline

**Назначение:** Автоматическое исправление неудачных попыток редактирования кода.

**Когда активируется:**
- Edit tool не находит `old_string` в файле
- Проблемы с whitespace, indentation, или escape-символами
- Может быть вызван как Edit Corrector, так и LLM Edit Fixer

**Компоненты:**
- Edit Tool
- Edit Corrector (простые исправления)
- LLM Edit Fixer (сложные исправления с контекстом)

**Схема:**
```
┌─────────────────────────────────────────────────────────┐
│  Main Agent calls Edit Tool                             │
│  Input: { file_path, old_string, new_string }          │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Edit Tool attempts to find old_string in file          │
│  - Counts occurrences of old_string                     │
│  - Expected: exactly 1 match (or expected_replacements) │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
    [Found?] ────Yes───> [Execute edit successfully]
         │
         No
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Edit Corrector - Phase 1: Simple Fixes                │
│                                                         │
│  1. Try unescaping old_string                           │
│     - Remove extra backslashes (\\n → \n, \\" → ")      │
│     - Check if unescaped version matches                │
│                                                         │
│  2. If new_string is escaped, fix it too                │
│     - Correct escaping to match old_string changes      │
│                                                         │
│  3. Try trimming whitespace                             │
│     - old_string.trim() might match                     │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
    [Fixed?] ────Yes───> [Retry edit with corrected params]
         │
         No
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Edit Corrector - Phase 2: LLM-based Correction        │
│  Prompt: Code Correction System Prompt                 │
│  Input: old_string, file_content                       │
│  Output: corrected_target_snippet                      │
│                                                         │
│  LLM analyzes:                                          │
│  - What in file_content is closest to old_string?      │
│  - What escaping issues exist?                          │
│  - Returns EXACT literal text from file                │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
    [Fixed?] ────Yes───> [Retry edit with corrected params]
         │
         No
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Check if file was modified externally                  │
│  - Find last edit timestamp in history                  │
│  - Compare with file modification time                  │
│  - If file modified > 2 seconds after last edit:        │
│    → File changed externally, give up                   │
└────────┬────────────────────────────────────────────────┘
         │
         No external modification
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  LLM Edit Fixer - Advanced Context-Aware Correction    │
│  System Prompt: LLM Edit Fixer System Prompt           │
│  User Prompt: LLM Edit Fixer User Prompt               │
│                                                         │
│  Input:                                                 │
│  - instruction: High-level goal of the edit            │
│  - old_string: Failed search string                    │
│  - new_string: Replacement string                      │
│  - error: Error message                                │
│  - current_content: Full file content                  │
│                                                         │
│  Output (JSON):                                         │
│  {                                                      │
│    "search": "corrected search string",                │
│    "replace": "replacement (usually unchanged)",       │
│    "explanation": "why original failed",               │
│    "noChangesRequired": false                          │
│  }                                                     │
│                                                         │
│  Timeout: 40 seconds                                    │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
    [noChangesRequired?] ────Yes───> [Return success: changes already present]
         │
         No
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Retry Edit with LLM-corrected parameters               │
│  - Use corrected search string                          │
│  - Keep original or corrected replace string            │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
    [Success?] ────Yes───> [Edit completed]
         │
         No
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Return error to Main Agent                             │
│  Agent must handle manually or ask user                 │
└─────────────────────────────────────────────────────────┘
```

**Передача данных:**
1. Edit Tool → Edit Corrector:
   - `file_path`, `old_string`, `new_string`
   - `current_content` (файл целиком)
   - `expected_replacements` (обычно 1)

2. Edit Corrector → LLM (Code Correction):
   - `problematicSnippet`: old_string после простых исправлений
   - `fileContent`: содержимое файла
   - Возврат: `corrected_target_snippet`

3. LLM Edit Fixer → LLM:
   - `instruction`: цель правки
   - `old_string`, `new_string`: оригинальные параметры
   - `error`: сообщение об ошибке
   - `current_content`: содержимое файла
   - Возврат: `{ search, replace, explanation, noChangesRequired }`

4. Corrected params → Edit Tool (retry)

---