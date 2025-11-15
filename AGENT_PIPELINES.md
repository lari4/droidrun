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

## Scripter Delegation Pipeline

**When:** Manager detects `<script>` tags in its plan (only in Reasoning Mode)

**Purpose:** Execute off-device Python operations like API calls, web requests, data processing, file downloads, and webhooks on the host machine.

### Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        ManagerAgent                             │
│                                                                 │
│  Detects need for off-device operation and includes in plan:   │
│                                                                 │
│  <plan>                                                         │
│  <script>                                                       │
│  Fetch weather data from https://api.weather.com/london        │
│  and extract the temperature in Celsius                        │
│  </script>                                                      │
│  1. Continue with device actions...                            │
│  </plan>                                                        │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 │ ManagerPlanEvent (contains script)
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                      DroidAgent Router                          │
│                                                                 │
│  Detects <script> tag at start of plan                         │
│  → Routes to ScripterAgent                                     │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 │ ScripterExecutorInputEvent(script_task)
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                      ScripterAgent                              │
│                   (Off-device Python Executor)                  │
│                                                                 │
│  Execution Model: Like Jupyter Notebook                        │
│  - Variables persist across executions                         │
│  - Iterative problem solving                                   │
│  - Max steps limit                                             │
│                                                                 │
│  Step 1: Generate Python code                                  │
│  ```python                                                      │
│  import requests                                               │
│  response = requests.get("https://api.weather.com/london")     │
│  data = response.json()                                        │
│  temp = data['temperature']                                    │
│  print(f"Temperature: {temp}°C")                               │
│  ```                                                            │
│                                                                 │
│  Step 2: Execute code                                          │
│  Output: "Temperature: 22°C"                                   │
│                                                                 │
│  Step 3: Final message (NO code block)                         │
│  "Successfully fetched weather data. Temperature is 22°C"      │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 │ ScripterExecutorResultEvent
                 │ (status=SUCCESS/FAILED, message)
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                        DroidAgent                               │
│                                                                 │
│  Wraps result in XML format for Manager:                       │
│  <script_result status="SUCCESS">                              │
│  Successfully fetched weather data. Temperature is 22°C        │
│  </script_result>                                              │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 │ ManagerInputEvent (with script result)
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                        ManagerAgent                             │
│                                                                 │
│  Receives script result and continues planning:                │
│                                                                 │
│  <thought>                                                      │
│  Great! I got the temperature (22°C). Now I need to            │
│  display it on the device screen.                              │
│  </thought>                                                     │
│                                                                 │
│  <add_memory>                                                   │
│  At step 5, scripter successfully fetched weather: 22°C        │
│  </add_memory>                                                  │
│                                                                 │
│  <plan>                                                         │
│  1. Open Notes app                                             │
│  2. Type "Today's temperature: 22°C"                           │
│  3. DONE                                                        │
│  </plan>                                                        │
└─────────────────────────────────────────────────────────────────┘
```

### Prompts Used

| Agent | Prompt | Purpose |
|-------|--------|---------|
| **ScripterAgent** | `scripter/system.jinja2` | System prompt: Execute Python code iteratively like Jupyter |

### Data Flow

**Input to ScripterAgent:**
- `task`: Description of what needs to be accomplished (from `<script>` tag)
- `available_libraries`: List of Python libraries (requests, json, etc.)
- `max_steps`: Maximum execution steps allowed

**ScripterAgent iterative execution:**

**Iteration 1:**
```
Thought: I need to fetch data from the API first.
```python
import requests
response = requests.get("https://api.weather.com/london")
data = response.json()
print(f"Fetched: {data}")
```

**Iteration 2:**
```
Thought: Now extract the temperature.
```python
temperature = data['main']['temp']
print(f"Temperature: {temperature}°C")
```

**Final iteration (NO CODE BLOCK):**
```
Successfully fetched weather data. The temperature in London is 22°C.
```

**Output from ScripterAgent:**
- `status`: "SUCCESS" or "FAILED"
- `message`: Final message explaining what happened

### Error Handling

If scripter encounters errors:

```
ScripterAgent Step 1:
```python
import requests
response = requests.get("https://invalid-url.com")
```

Output: Connection error

ScripterAgent Final Message (NO CODE BLOCK):
"Error: Connection timeout. The API at https://invalid-url.com might be down. Please verify the URL."
```

Manager receives:
```xml
<script_result status="FAILED">
Error: Connection timeout. The API at https://invalid-url.com might be down. Please verify the URL.
</script_result>
```

Manager response:
```xml
<thought>
The scripter failed due to connection timeout. I should try an alternative approach or inform the user.
</thought>

<request_accomplished success="false">
Unable to fetch weather data due to API connection timeout.
</request_accomplished>
```

### Example Flow

**User Request:** "Download the latest release info from GitHub API and save it to clipboard"

**Manager generates:**
```xml
<plan>
<script>
Fetch the latest release from GitHub API https://api.github.com/repos/owner/repo/releases/latest and extract the tag_name and published_at fields
</script>
1. Open any app with a text field
2. Paste the release info into the text field
3. Copy to clipboard
4. DONE
</plan>
```

**Scripter Execution:**

**Step 1:**
```python
import requests
url = "https://api.github.com/repos/owner/repo/releases/latest"
response = requests.get(url)
data = response.json()
print(f"Status: {response.status_code}")
```
Output: `Status: 200`

**Step 2:**
```python
tag = data['tag_name']
published = data['published_at']
release_info = f"Latest Release: {tag}\nPublished: {published}"
print(release_info)
```
Output:
```
Latest Release: v1.2.3
Published: 2024-01-15T10:30:00Z
```

**Final (NO CODE BLOCK):**
```
Successfully fetched latest release info:
Latest Release: v1.2.3
Published: 2024-01-15T10:30:00Z
```

**Manager receives result and continues with device actions to save to clipboard.**

---

## Text Manipulation Pipeline

**When:** Manager detects `TEXT_TASK:` prefix in plan (only in Reasoning Mode)

**Purpose:** Edit or transform text in currently focused Android text input fields using constrained Python code generation.

### Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        ManagerAgent                             │
│                                                                 │
│  Detects need for text manipulation in focused text field:     │
│                                                                 │
│  <plan>                                                         │
│  1. Click on the message text field                            │
│  TEXT_TASK: Add "Hello World" at the beginning of the text     │
│  2. Send the message                                           │
│  3. DONE                                                        │
│  </plan>                                                        │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 │ ManagerPlanEvent (contains TEXT_TASK)
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                      DroidAgent Router                          │
│                                                                 │
│  Detects TEXT_TASK: prefix in first subgoal                    │
│  → Routes to TextManipulatorAgent                              │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 │ TextManipulatorInputEvent(task, current_text)
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                   TextManipulatorAgent                          │
│              (Constrained Code Generator)                       │
│                                                                 │
│  Input:                                                         │
│  - Task: "Add 'Hello World' at the beginning"                  │
│  - ORIGINAL: "This is existing text"                           │
│  - Overall plan context                                        │
│  - Current subgoal                                             │
│                                                                 │
│  LLM generates constrained Python code:                        │
│  ```python                                                      │
│  new_text = "Hello World\n" + ORIGINAL                         │
│  input_text(new_text)                                          │
│  ```                                                            │
│                                                                 │
│  Sandbox Execution:                                            │
│  - Environment: {                                              │
│      "__builtins__": {},                                       │
│      "input_text": <function>,                                 │
│      "ORIGINAL": "This is existing text"                       │
│    }                                                            │
│  - Executes code safely                                        │
│  - Captures result from input_text() call                      │
│                                                                 │
│  If execution fails:                                           │
│  → Send error back to LLM for correction                       │
│  → Retry (up to 4 times)                                       │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 │ TextManipulatorResultEvent
                 │ (text_to_type, code_ran)
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                        DroidAgent                               │
│                                                                 │
│  1. Receives final text: "Hello World\nThis is existing text"  │
│  2. Executes input_text() to clear field and type new text     │
│  3. Records outcome                                            │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 │ ManagerInputEvent (with result)
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                        ManagerAgent                             │
│                                                                 │
│  Receives confirmation and continues:                           │
│                                                                 │
│  <thought>                                                      │
│  Text has been modified successfully. Now send the message.    │
│  </thought>                                                     │
│                                                                 │
│  <plan>                                                         │
│  1. Send the message                                           │
│  2. DONE                                                        │
│  </plan>                                                        │
└─────────────────────────────────────────────────────────────────┘
```

### Prompts Used

| Component | Prompt | Purpose |
|-----------|--------|---------|
| **TextManipulator** | Inline system prompt | Constrained Python code generator for text editing |
| **TextManipulator** | Error correction prompt | Fixes code execution errors |
| **TextManipulator** | User prompt | Presents task and current text |

### Data Flow

**Input to TextManipulatorAgent:**
- `instruction`: User's overall goal
- `current_subgoal`: The TEXT_TASK description
- `current_text`: Current content of focused text field (ORIGINAL variable)
- `overall_plan`: Manager's full plan for context
- `llm`: LLM instance for code generation
- `max_retries`: Maximum retry attempts (default: 4)

**LLM generates:**
```python
new_text = """Hello World
""" + ORIGINAL
input_text(new_text)
```

**Sandbox execution:**
- Restricted environment (no imports, no builtins)
- Only `input_text()` function and `ORIGINAL` variable available
- Executes code safely
- Captures final text from `input_text()` call

**Output from TextManipulatorAgent:**
- `text_to_type`: Final text to input (e.g., "Hello World\nThis is existing text")
- `code_ran`: The generated Python code

### Error Correction Flow

**Attempt 1 - Code with error:**
```python
new_text = "Hello World" + ORIGINAL  # Missing newline
input_text(new_text)
```

**Execution succeeds but may not be what user wants**

**Attempt 2 - Corrected code:**
```python
new_text = "Hello World\n" + ORIGINAL  # Correct with newline
input_text(new_text)
```

**If code has syntax error:**
```python
new_text = "Hello World + ORIGINAL  # Missing quote
input_text(new_text)
```

**Error:** `SyntaxError: EOL while scanning string literal`

**Error correction prompt sent to LLM:**
```
You are CODEACT_TEXT_AGENT, correcting your previous code that had execution errors.

The code you generated previously failed with this error:
SyntaxError: EOL while scanning string literal

Please fix the code...
```

**LLM generates corrected code:**
```python
new_text = "Hello World\n" + ORIGINAL
input_text(new_text)
```

### Constraints and Security

**Allowed:**
- Reference `ORIGINAL` variable
- Call `input_text(text)` function
- String operations and concatenation
- Triple-quoted strings for multi-line text

**NOT Allowed:**
- `import` statements
- Function/class definitions
- `print()` or `input()` calls
- File/network/system access
- Any builtins

### Example Flows

**Example 1: Add signature**

**Current text:** `"Meeting at 3pm tomorrow"`

**Task:** `TEXT_TASK: Add my email signature at the end`

**Generated code:**
```python
signature = "\n\nBest regards,\nJohn Doe\njohn@example.com"
new_text = ORIGINAL + signature
input_text(new_text)
```

**Result:** `"Meeting at 3pm tomorrow\n\nBest regards,\nJohn Doe\njohn@example.com"`

---

**Example 2: Replace text**

**Current text:** `"The quick brown fox"`

**Task:** `TEXT_TASK: Replace "brown" with "red"`

**Generated code:**
```python
new_text = ORIGINAL.replace("brown", "red")
input_text(new_text)
```

**Result:** `"The quick red fox"`

---

**Example 3: Format as bullet points**

**Current text:** `"Item 1, Item 2, Item 3"`

**Task:** `TEXT_TASK: Convert to bullet points`

**Generated code:**
```python
items = ORIGINAL.split(", ")
bullet_list = "\n".join([f"• {item}" for item in items])
new_text = bullet_list
input_text(new_text)
```

**Result:**
```
• Item 1
• Item 2
• Item 3
```

---

## Structured Output Pipeline

**When:** `output_model` is provided (Pydantic BaseModel for structured extraction)

**Purpose:** Extract structured data from the final answer and return it in a typed format.

### Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                   Agent completes task                          │
│          (CodeActAgent or ManagerAgent)                         │
│                                                                 │
│  Final answer: "The weather in London is 22°C and sunny"       │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 │ FinalizeEvent(result)
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                    DroidAgent Finalize                          │
│                                                                 │
│  If output_model is defined:                                   │
│  → Route to StructuredOutputAgent                              │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 │ (final_answer, output_model)
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                  StructuredOutputAgent                          │
│                                                                 │
│  Prompt Template:                                              │
│  "Extract the following information from the text:             │
│   {output_schema}                                              │
│   Text: {final_answer}                                         │
│   Return JSON matching the schema."                            │
│                                                                 │
│  LLM extracts structured data:                                 │
│  {                                                             │
│    "temperature": 22,                                          │
│    "unit": "celsius",                                          │
│    "condition": "sunny",                                       │
│    "location": "London"                                        │
│  }                                                             │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 │ Parsed Pydantic model instance
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                        DroidAgent                               │
│                                                                 │
│  Returns ResultEvent with:                                     │
│  - success: True                                               │
│  - result: "The weather in London is 22°C and sunny"           │
│  - data: WeatherInfo(                                          │
│            temperature=22,                                     │
│            unit="celsius",                                     │
│            condition="sunny",                                  │
│            location="London"                                   │
│          )                                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Prompts Used

| Agent | Prompt | Purpose |
|-------|--------|---------|
| **StructuredOutputAgent** | Dynamic prompt template | Extracts structured data based on Pydantic schema |

### Example

**User code:**
```python
from pydantic import BaseModel

class WeatherInfo(BaseModel):
    temperature: int
    unit: str
    condition: str
    location: str

agent = DroidAgent(
    goal="Check the weather in London",
    output_model=WeatherInfo,
    ...
)
```

**Agent completes task:**
- Final answer: "The current weather in London is 22 degrees Celsius and sunny."

**StructuredOutputAgent extracts:**
```python
WeatherInfo(
    temperature=22,
    unit="celsius",
    condition="sunny",
    location="London"
)
```

**User receives:**
```python
result = await agent.run()
print(result.success)  # True
print(result.result)   # "The current weather in London is 22 degrees Celsius and sunny."
print(result.data)     # WeatherInfo(temperature=22, unit="celsius", condition="sunny", location="London")
```

---

## Summary

This document has covered all agent pipelines in DroidRun:

1. **Direct Execution (CodeAct)**: Fast, single-agent code execution
2. **Reasoning Mode (Manager-Executor)**: Strategic planning with atomic actions
3. **Scripter Delegation**: Off-device Python operations
4. **Text Manipulation**: Constrained code generation for text editing
5. **Structured Output**: Typed data extraction from results

Each pipeline is optimized for specific use cases, working together to provide a flexible and powerful Android automation system.
