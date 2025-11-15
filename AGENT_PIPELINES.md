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

## Reasoning Mode Pipeline (Manager-Executor)

**When:** `reasoning=True` (for complex, multi-step tasks requiring strategic planning)

**Purpose:** Separates high-level planning (Manager) from low-level execution (Executor) for better control and error handling in complex scenarios.

### Pipeline Flow

```
┌───────────────────────────────────────────────────────────────────┐
│                           DroidAgent                              │
│                         (Orchestrator)                            │
└───────┬───────────────────────────────────────────────────────────┘
        │
        │ StartEvent(instruction)
        ▼
┌───────────────────────────────────────────────────────────────────┐
│                        ManagerAgent                               │
│                      (Strategic Planner)                          │
│                                                                   │
│  1. Receives user instruction                                    │
│  2. Analyzes current device state + screenshot                   │
│  3. Creates high-level plan with numbered subgoals               │
│  4. Tracks progress and memory                                   │
│  5. Decides: continue, delegate, or complete                     │
│                                                                   │
│  Output XML Tags:                                                │
│  - <thought>: Reasoning about current state                      │
│  - <plan>: List of subgoals or <script> tags                     │
│  - <add_memory>: Information to remember                         │
│  - <request_accomplished>: Task completion signal                │
└───────┬───────────────────────────────────────────────────────────┘
        │
        │ ManagerPlanEvent
        │ (plan, subgoals, scripts, text_tasks)
        ▼
┌───────────────────────────────────────────────────────────────────┐
│                        DroidAgent Router                          │
│                                                                   │
│  Examines first subgoal and routes to appropriate handler:       │
│  - Regular subgoal → ExecutorAgent                               │
│  - <script>...</script> → ScripterAgent                          │
│  - TEXT_TASK: ... → TextManipulatorAgent                         │
└───────┬───────────────────────────────────────────────────────────┘
        │
        ├─── Regular Subgoal ───────────────────────────────────────┐
        │                                                           │
        │ ExecutorInputEvent(subgoal)                              │
        ▼                                                           │
┌───────────────────────────────────────────────────────────────┐   │
│                       ExecutorAgent                           │   │
│                    (Action Executor)                          │   │
│                                                               │   │
│  1. Receives current subgoal from Manager                    │   │
│  2. Parses subgoal to identify:                              │   │
│     - Action verb (click, swipe, type, open)                 │   │
│     - Target element                                         │   │
│     - Location/parameters                                    │   │
│  3. Executes atomic action mechanically                      │   │
│  4. Does NOT make strategic decisions                        │   │
│                                                               │   │
│  Output JSON:                                                │   │
│  {                                                           │   │
│    "action": "click",                                        │   │
│    "index": 5                                                │   │
│  }                                                           │   │
└───────┬───────────────────────────────────────────────────────┘   │
        │                                                           │
        │ ExecutorResultEvent(outcome, error)                      │
        ▼                                                           │
┌───────────────────────────────────────────────────────────────┐   │
│                        DroidAgent                             │   │
│                                                               │   │
│  1. Executes atomic action via tools                         │   │
│  2. Records outcome (success/failure)                        │   │
│  3. Updates shared state                                     │   │
│  4. Returns to Manager with results                          │   │
└───────┬───────────────────────────────────────────────────────┘   │
        │                                                           │
        │ ManagerInputEvent(continue)                               │
        │                                                           │
        └───────────────────────────────────────────────────────────┘
                                ▲
                                │ Loop until task complete
                                │
```

### Prompts Used

| Agent | Prompt | Purpose |
|-------|--------|---------|
| **ManagerAgent** | `manager/system.jinja2` or `manager/rev1.jinja2` | System prompt: Create plans, track progress, decide completion |
| **ExecutorAgent** | `executor/system.jinja2` or `executor/rev1.jinja2` | System prompt: Parse subgoals and execute atomic actions mechanically |

### Data Flow

**Input to ManagerAgent:**
- `instruction`: User's goal
- `device_state`: Current device info (app, date, etc.)
- `screenshots`: Visual screenshot (if vision enabled)
- `app_card`: App-specific guidance (optional)
- `important_notes`: Contextual notes (optional)
- `error_history`: Recent failed actions (optional)
- `custom_tools_descriptions`: Custom actions available
- `available_secrets`: Secret IDs for credentials
- `plan`: Previous plan (for updates)
- `action_history`: Recent action outcomes
- `memory`: Remembered information from previous steps

**ManagerAgent generates:**
```xml
<thought>
The Settings app is now open. I need to navigate to Wi-Fi settings next.
</thought>

<add_memory>
At step 1, Settings app was successfully opened.
</add_memory>

<plan>
1. Click on "Wi-Fi" option to enter Wi-Fi settings
2. Enable Wi-Fi if it's currently disabled
3. DONE
</plan>
```

**Input to ExecutorAgent:**
- `instruction`: User's original request
- `app_card`: App-specific guidance (optional)
- `device_state`: Current device state
- `plan`: Overall plan from Manager
- `subgoal`: **Current subgoal to execute** (e.g., "Click on Wi-Fi option")
- `progress_status`: Current progress description
- `atomic_actions`: Available actions with signatures
- `available_secrets`: Secret IDs for credentials
- `action_history`: Last 5 actions with outcomes

**ExecutorAgent generates:**
```
### Thought ###
The subgoal is "Click on Wi-Fi option". This requires a click action on the element containing "Wi-Fi" text.

### Action ###
{"action": "click", "text": "Wi-Fi"}

### Description ###
Clicking on the Wi-Fi option to navigate to Wi-Fi settings.
```

### Manager Decision Logic

The Manager analyzes the current state and makes one of these decisions:

1. **Continue with next subgoal** → Update plan, send subgoal to Executor
2. **Delegate to Scripter** → Found `<script>` tag in plan → Route to ScripterAgent
3. **Delegate to Text Manipulator** → Found `TEXT_TASK:` prefix → Route to TextManipulatorAgent
4. **Task complete** → Output `<request_accomplished success="true">` tag
5. **Task failed** → Output `<request_accomplished success="false">` tag

### Error Handling

When Executor actions fail repeatedly (threshold: 2 failures):

```
Manager receives error_history:
- Attempt 1: Action: click | Description: Click Wi-Fi | Outcome: Failed | Feedback: Element not found
- Attempt 2: Action: click | Description: Click Wi-Fi | Outcome: Failed | Feedback: Element not found

Manager response:
<thought>
The Wi-Fi option is not visible. I should try swiping down to reveal more options in the Settings menu.
</thought>

<plan>
1. Swipe down to reveal more settings options
2. Look for Wi-Fi option and click it
3. Enable Wi-Fi if disabled
4. DONE
</plan>
```

### Memory Management

Manager uses `<add_memory>` tags to store information across steps:

```xml
<add_memory>
At step 3, I obtained the weather data from the API: Temperature is 22°C, Humidity is 65%, Condition is Sunny.
</add_memory>
```

This memory persists and is included in future Manager prompts, allowing multi-step data collection and processing.

### Example Flow

**User Request:** "Find the top trending video on YouTube and send its title to webhook.site"

**Manager Step 1:**
- **Input:** User instruction + current device state
- **Output:**
  ```xml
  <thought>
  I need to open YouTube app first, then find the trending section to get the top video title.
  </thought>

  <plan>
  1. Open YouTube app
  2. Navigate to Trending section
  3. Click on the first video to see its full title
  4. Remember the video title
  <script>
  Send the video title to webhook https://webhook.site/abc123 as JSON with format {"title": "video_title"}
  </script>
  5. DONE
  </plan>
  ```

**Executor Step 1:**
- **Input:** Subgoal: "Open YouTube app"
- **Output:** `{"action": "open_app", "text": "YouTube"}`
- **Execution:** YouTube app opens

**Manager Step 2:**
- **Input:** Previous plan + action outcome (success)
- **Output:**
  ```xml
  <thought>
  YouTube is now open. I need to navigate to the Trending section.
  </thought>

  <plan>
  1. Navigate to Trending section
  2. Click on the first video to see its full title
  3. Remember the video title
  <script>
  Send the video title to webhook https://webhook.site/abc123 as JSON with format {"title": "video_title"}
  </script>
  4. DONE
  </plan>
  ```

**Executor Step 2:**
- **Input:** Subgoal: "Navigate to Trending section"
- **Output:** `{"action": "click", "text": "Trending"}`
- **Execution:** Trending section opens

**Manager Step 3:**
- **Input:** Previous plan + action outcome + screenshot showing trending videos
- **Output:**
  ```xml
  <thought>
  I can see trending videos. The first video is titled "Amazing Cat Compilation 2024". I'll click on it to confirm the full title.
  </thought>

  <add_memory>
  At step 3, I observed the top trending video title: "Amazing Cat Compilation 2024"
  </add_memory>

  <plan>
  <script>
  Send the video title "Amazing Cat Compilation 2024" to webhook https://webhook.site/abc123 as JSON with format {"title": "Amazing Cat Compilation 2024"}
  </script>
  1. DONE
  </plan>
  ```

**DroidAgent Routes to ScripterAgent:**
- **Input:** Script task: Send video title to webhook
- **ScripterAgent executes Python code** (see Scripter Pipeline section)
- **Output:** Success/failure result

**Manager Step 4:**
- **Input:** Previous plan + scripter result (SUCCESS)
- **Output:**
  ```xml
  <thought>
  The video title has been successfully sent to the webhook. Task is complete.
  </thought>

  <request_accomplished success="true">
  Successfully found the top trending video on YouTube ("Amazing Cat Compilation 2024") and sent its title to the webhook.
  </request_accomplished>
  ```

---
