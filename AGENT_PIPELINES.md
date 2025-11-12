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