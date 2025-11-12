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

## 4. Loop Detection Pipeline

**Назначение:** Обнаружение зацикливания агента в непродуктивном состоянии.

**Когда активируется:**
- Каждую итерацию агента (проверка tool calls и content)
- После 30+ итераций в одном промпте (LLM-based проверка)

**Компоненты:**
- Loop Detection Service
- Tool call tracking
- Content streaming tracking
- LLM-based diagnostic agent

**Схема:**
```
┌─────────────────────────────────────────────────────────┐
│  Every Agent Turn / Stream Event                        │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Loop Detection Service                                 │
│  - Tool Call Loop Detection                             │
│  - Content Loop Detection                               │
│  - LLM-based Loop Detection (periodic)                  │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────┐
│  1. Tool Call Loop Detection     │
│                                  │
│  Track:                          │
│  - Hash of (tool_name + args)    │
│  - Repetition count              │
│                                  │
│  Threshold: 5 identical calls    │
│  → LOOP DETECTED                 │
└──────────────────────────────────┘

┌──────────────────────────────────┐
│  2. Content Loop Detection       │
│                                  │
│  Track:                          │
│  - Sliding window chunks (50ch)  │
│  - Hash map of chunk positions   │
│  - Reset on code blocks/tables   │
│                                  │
│  Threshold:                      │
│  - 10 occurrences of same chunk  │
│  - Within 5×chunk_size distance  │
│  → LOOP DETECTED                 │
└──────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│  3. LLM-based Loop Detection (after 30+ turns)           │
│                                                          │
│  Trigger: turnsInCurrentPrompt >= 30 AND                │
│          (currentTurn - lastCheckTurn) >= llmCheckInterval│
│                                                          │
│  Process:                                                │
│  1. Get last 20 messages from history                    │
│  2. Remove dangling function calls/responses             │
│  3. Send to LLM with Loop Detection Prompt              │
│     - Analyze for Repetitive Actions                     │
│     - Analyze for Cognitive Loop                         │
│     - Differentiate from legitimate progress             │
│  4. Get JSON response:                                   │
│     {                                                    │
│       "unproductive_state_analysis": "...",              │
│       "unproductive_state_confidence": 0.0-1.0           │
│     }                                                    │
│  5. If confidence > 0.9 → LOOP DETECTED                  │
│  6. Adjust llmCheckInterval (5-15) based on confidence   │
│     - High confidence → check more frequently (5)        │
│     - Low confidence → check less frequently (15)        │
└──────────────────────────────────────────────────────────┘
         │
         ▼
    [Loop Detected?] ────No───> [Continue normally]
         │
         Yes
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Stop Agent Execution                                   │
│  - Set loopDetected flag                                │
│  - Log telemetry event                                  │
│  - Return error to user                                 │
│  - User can disable loop detection for session          │
└─────────────────────────────────────────────────────────┘
```

**Передача данных:**
1. Stream Events → Loop Detector:
   - `ToolCallRequest`: `{ name, args }`
   - `Content`: текстовые chunks

2. LLM Loop Check → LLM:
   - Last 20 messages (trimmed)
   - Task prompt: "analyze for loops"
   - Return: `{ unproductive_state_analysis, unproductive_state_confidence }`

---

## 5. History Compression Pipeline

**Назначение:** Сжатие истории разговора когда она становится слишком большой.

**Когда активируется:**
- История превышает максимальный размер контекстного окна
- Настраивается автоматически системой

**Компоненты:**
- History manager
- Compression agent (с Compression Prompt)

**Схема:**
```
┌─────────────────────────────────────────────────────────┐
│  History grows too large                                │
│  - Approaching context window limit                     │
│  - System triggers compression                          │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Compression Agent                                      │
│  System Prompt: Compression Prompt                     │
│                                                         │
│  Input: Full conversation history                       │
│                                                         │
│  Task:                                                  │
│  1. Think in private <scratchpad>                       │
│     - Review user's goal                                │
│     - Review agent actions                              │
│     - Review tool outputs                               │
│     - Review file modifications                         │
│     - Identify essential information                    │
│                                                         │
│  2. Generate <state_snapshot> XML                       │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Generate State Snapshot (XML)                          │
│                                                         │
│  <state_snapshot>                                       │
│    <overall_goal>                                       │
│      User's high-level objective                        │
│    </overall_goal>                                      │
│                                                         │
│    <key_knowledge>                                      │
│      - Build command: npm run build                     │
│      - Test command: npm test                           │
│      - Key constraints and conventions                  │
│    </key_knowledge>                                     │
│                                                         │
│    <file_system_state>                                  │
│      - CWD: /path/to/project                            │
│      - READ: package.json - confirmed deps              │
│      - MODIFIED: src/foo.ts - refactored                │
│      - CREATED: tests/new.test.ts                       │
│    </file_system_state>                                 │
│                                                         │
│    <recent_actions>                                     │
│      - Ran grep 'function' → 5 results                  │
│      - Ran npm test → failed with error X               │
│    </recent_actions>                                    │
│                                                         │
│    <current_plan>                                       │
│      1. [DONE] Step 1                                   │
│      2. [IN PROGRESS] Step 2                            │
│      3. [TODO] Step 3                                   │
│    </current_plan>                                      │
│  </state_snapshot>                                      │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Replace Full History with Snapshot                     │
│  - Original history cleared                             │
│  - Snapshot becomes first user message                  │
│  - Agent continues from this compressed state           │
│  - ALL crucial details preserved in snapshot            │
└─────────────────────────────────────────────────────────┘
```

**Передача данных:**
1. Full History → Compression Agent:
   - Все сообщения в разговоре
   - Tool calls и responses
   - User messages и model responses

2. Compression Agent → Compressed State:
   - XML state_snapshot
   - Максимально плотная информация
   - Никакого conversational filler

---

## 6. Next Speaker Decision Pipeline

**Назначение:** Определение кто должен говорить следующим: пользователь или модель.

**Когда активируется:**
- После каждого ответа модели
- Определяет нужно ли продолжить автоматически или ждать пользователя

**Компоненты:**
- Next Speaker Checker
- Curated history (без invalid turns)

**Схема:**
```
┌─────────────────────────────────────────────────────────┐
│  Model generates response                               │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Pre-checks (fast path)                                 │
│                                                         │
│  1. Last message is function_response?                  │
│     → Model should speak next                           │
│                                                         │
│  2. Last model message has empty parts?                 │
│     → Model should speak next (filler message)          │
│                                                         │
│  3. History is empty?                                   │
│     → Cannot determine, return null                     │
└────────┬────────────────────────────────────────────────┘
         │
         No pre-check match
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Next Speaker Checker Agent                             │
│  Prompt: Next Speaker Checker Prompt                    │
│                                                         │
│  Input: Curated history (last model response visible)   │
│                                                         │
│  Analyze last model response ONLY:                      │
│  - Content and structure                                │
│  - Apply decision rules in order:                       │
│                                                         │
│    Rule 1: Model Continues                              │
│    - Explicit next action stated                        │
│    - Response incomplete/cut off                        │
│    - Intended tool call didn't execute                  │
│    → next_speaker = "model"                             │
│                                                         │
│    Rule 2: Question to User                             │
│    - Ends with direct question to user                  │
│    → next_speaker = "user"                              │
│                                                         │
│    Rule 3: Waiting for User                             │
│    - Completed thought/statement/task                   │
│    - Doesn't match Rule 1 or 2                          │
│    - Implies pause for user input                       │
│    → next_speaker = "user"                              │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Return Decision (JSON)                                 │
│  {                                                      │
│    "reasoning": "Explanation...",                       │
│    "next_speaker": "user" | "model"                     │
│  }                                                      │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Take Action                                            │
│  - If "model": Continue agent turn immediately          │
│  - If "user": Wait for user input                       │
└─────────────────────────────────────────────────────────┘
```

**Передача данных:**
1. Curated History → Next Speaker Checker:
   - История без invalid turns
   - Последнее сообщение модели видно

2. Next Speaker Checker → Decision:
   - `{ reasoning: string, next_speaker: "user" | "model" }`

---

## 7. Tool Output Summarization Pipeline

**Назначение:** Сжатие слишком длинных выводов инструментов для эффективной передачи модели.

**Когда активируется:**
- Вывод инструмента > maxOutputTokens (обычно 2000)
- Автоматически для всех tool outputs

**Компоненты:**
- Summarizer
- History context для контекстно-зависимой суммаризации

**Схема:**
```
┌─────────────────────────────────────────────────────────┐
│  Tool Execution Completes                               │
│  Output: string (может быть очень длинным)              │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Check Output Length                                    │
│  length > maxOutputTokens?                              │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
    [Too long?] ────No───> [Return original output]
         │
         Yes
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Summarizer Agent                                       │
│  Prompt: Tool Output Summarization Prompt              │
│                                                         │
│  Input:                                                 │
│  - textToSummarize: Full tool output                    │
│  - maxOutputTokens: Target length (e.g., 2000)          │
│  - Context: Full conversation history                   │
│                                                         │
│  Rules:                                                 │
│  1. Structural content (directory listing):             │
│     - Use history to understand what user needs         │
│     - Extract relevant information only                 │
│                                                         │
│  2. Text content:                                       │
│     - Summarize main points                             │
│                                                         │
│  3. Shell command output:                               │
│     - Summarize overall result                          │
│     - Extract FULL stack trace → <error>...</error>     │
│     - Extract warnings → <warning>...</warning>         │
│     - Do NOT truncate errors/warnings                   │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Generate Summary                                       │
│                                                         │
│  Format:                                                │
│  [Overall summarization of content]                     │
│                                                         │
│  <error>                                                │
│  [Complete stack trace if present]                      │
│  </error>                                               │
│                                                         │
│  <warning>                                              │
│  [Complete warnings if present]                         │
│  </warning>                                             │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Return Summarized Output to Model                      │
│  - Compressed to ~maxOutputTokens                       │
│  - Preserves critical information                       │
│  - Errors/warnings complete and accessible              │
└─────────────────────────────────────────────────────────┘
```

**Передача данных:**
1. Tool → Summarizer:
   - `textToSummarize`: Полный вывод
   - `maxOutputTokens`: Целевой размер
   - История разговора (для контекста)

2. Summarizer → Model:
   - Сжатый текст с сохранением:
     - Общей информации
     - Полных stack traces в `<error>`
     - Полных warnings в `<warning>`

---