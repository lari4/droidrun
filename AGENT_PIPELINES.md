# Agent Pipelines and Workflows

This document describes all agent workflows and pipelines in the DroidRun application. It shows how different agents interact, what prompts they use, and how data flows between them.

## Table of Contents

1. [Overview](#overview)
2. [Direct Execution Pipeline (CodeAct Mode)](#direct-execution-pipeline-codeact-mode)
3. [Reasoning Mode Pipeline (Manager-Executor)](#reasoning-mode-pipeline-manager-executor)
4. [Scripter Delegation Pipeline](#scripter-delegation-pipeline)
5. [Text Manipulation Pipeline](#text-manipulation-pipeline)
6. [Structured Output Pipeline](#structured-output-pipeline)

---

## Overview

DroidRun has **two main execution modes**:

1. **Direct Execution Mode** (`reasoning=False`): Uses **CodeActAgent** directly
2. **Reasoning Mode** (`reasoning=True`): Uses **ManagerAgent** (planning) + **ExecutorAgent** (actions)

The **DroidAgent** class acts as the orchestrator that coordinates all sub-agents.

### Agent Hierarchy

```
DroidAgent (Orchestrator)
├── CodeActAgent (Direct execution mode)
├── ManagerAgent (Planning in reasoning mode)
├── ExecutorAgent (Action execution in reasoning mode)
├── ScripterAgent (Off-device operations)
└── TextManipulatorAgent (Text field editing)
```

---

## Direct Execution Pipeline (CodeAct Mode)

**When:** `reasoning=False` (default for simple tasks)

**Purpose:** Fast, direct code execution for Android UI automation without strategic planning overhead.

### Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                         DroidAgent                              │
│                       (Orchestrator)                            │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 │ StartEvent(instruction)
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                      CodeActAgent                               │
│                                                                 │
│  1. Receives user instruction                                  │
│  2. Gets device state (UI elements, screenshot, phone state)   │
│  3. Generates Python code to accomplish task                   │
│  4. Executes code (calls tools: click, type, swipe, etc.)      │
│  5. Observes results                                           │
│  6. Repeats until task complete or max_steps reached           │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 │ TaskExecutionResultEvent
                 │ (success, reason)
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                         DroidAgent                              │
│                    Returns final result                         │
└─────────────────────────────────────────────────────────────────┘
```

### Prompts Used

| Agent | Prompt | Purpose |
|-------|--------|---------|
| **CodeActAgent** | `codeact/system.jinja2` | System prompt: Instructs LLM to write Python code for UI automation |
| **CodeActAgent** | `codeact/user.jinja2` | User prompt: Presents current goal and asks for next step |

### Data Flow

**Input to CodeActAgent:**
- `instruction`: User's goal/task
- `ui_state`: List of visible UI elements with indices
- `screenshots`: Visual screenshot of current screen (if vision enabled)
- `phone_state`: Current app package name
- `chat_history`: Previous actions and observations
- `tool_descriptions`: Available Python functions (click, type, swipe, etc.)
- `available_secrets`: Secret IDs for credentials
- `output_schema`: Optional structured output requirements

**CodeActAgent generates:**
```python
# Example generated code
click(5)  # Click element at index 5
type(12, "Hello World")  # Type text into element 12
complete(success=True, reason="Task completed successfully")
```

**Output from CodeActAgent:**
- `success`: Boolean indicating task completion status
- `reason`: Explanation of outcome or final answer
- Execution results from each code block

### Example Flow

**User Request:** "Open Settings app and turn on Wi-Fi"

**Step 1:**
- **Prompt Input:** System + User prompt with current screen state
- **LLM Output:**
  ```python
  open_app("Settings")
  ```
- **Execution:** Settings app opens
- **Observation:** Screenshots and UI state updated

**Step 2:**
- **Prompt Input:** System + User prompt + previous action + new screen state
- **LLM Output:**
  ```python
  click(3)  # Wi-Fi option at index 3
  ```
- **Execution:** Navigates to Wi-Fi settings
- **Observation:** Wi-Fi settings screen visible

**Step 3:**
- **Prompt Input:** System + User prompt + action history + current state
- **LLM Output:**
  ```python
  click(1)  # Wi-Fi toggle at index 1
  complete(success=True, reason="Wi-Fi has been turned on successfully")
  ```
- **Execution:** Wi-Fi enabled
- **Result:** Task complete

---
