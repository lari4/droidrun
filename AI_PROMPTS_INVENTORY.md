# DroidRun AI Prompts - Comprehensive Inventory

## Overview
This document provides a complete inventory of all AI prompts and instructions used in the DroidRun application for controlling Android devices through AI agents.

---

## 1. PROMPT TEMPLATE FILES (Jinja2 Templates)

### Location: `/home/user/droidrun/droidrun/config/prompts/`

These are the core system prompts used by different agents. They use Jinja2 templating with dynamic variable injection.

---

### 1.1 CodeAct Agent Prompts

#### File: `codeact/system.jinja2`
**Purpose**: System prompt for CodeActAgent - a ReAct-style agent that writes and executes Python code to solve problems on Android devices.

**Key Instructions**:
- Agent role: Helpful AI assistant that can write and execute Python code
- Output format: Python code wrapped in ``` tags
- Context provided: UI state, screenshots, phone state, chat history, execution results
- Must check preconditions before executing tasks
- Use `complete(success=bool, reason:str)` to mark tasks as finished
- Has access to available secrets (credentials)
- Must describe thoughts and explain key plan parts in comments (saved in history)
- Supports output schema for structured data collection

**Dynamic Variables**:
- `tool_descriptions`: Available functions/tools
- `available_secrets`: Available credential IDs
- `output_schema`: Expected output structure (optional)

---

#### File: `codeact/user.jinja2`
**Purpose**: User message template for CodeActAgent

**Content**:
- Current request/goal
- Asks: "Is the precondition met? What is your reasoning and next step?"
- Instructs to provide code in ```python ... ``` tags if needed

**Dynamic Variables**:
- `goal`: The user's request or current task

---

### 1.2 Executor Agent Prompts

#### File: `executor/system.jinja2`
**Purpose**: System prompt for ExecutorAgent - a low-level action executor for atomic Android operations

**Key Instructions**:
- Role: LOW-LEVEL ACTION EXECUTOR for Android phone
- Does NOT answer questions or provide results
- ONLY performs individual atomic actions from subgoals
- Part of larger system (Manager provides plans)
- EXECUTION MODE: "You are a dumb robot" - find exact text/element and perform specified action
- Subgoal parsing: Extract action words (tap, click, swipe, type, press, open), target elements, and locations
- General guidelines:
  - Close pop-ups/permission requests before proceeding
  - Use `open_app` action (not app drawer)
  - Use `swipe` with different directions to reveal content
  - For text input: click to activate box first, then use `type` action
  - Use `type_secret` for passwords/sensitive data without exposing values

**Output Format**:
- Three parts: ### Thought ###, ### Action ###, ### Description ###
- Action must be valid JSON: `{"action":"action_name", "argument":"value"}`

**Dynamic Variables**:
- `instruction`: User request
- `app_card`: App operation information
- `device_state`: Current device state
- `plan`: Overall plan context
- `subgoal`: Current subgoal to execute
- `progress_status`: Current progress
- `atomic_actions`: Available actions with descriptions
- `action_history`: Recent action history (last 5)
- `available_secrets`: Available credential IDs

---

#### File: `executor/rev1.jinja2`
**Purpose**: Alternative/revised version of executor prompt (simpler format)

**Key Instructions**:
- Similar role but simpler formatting
- Markdown list format instead of XML
- Focus on: Read subgoal → Identify action verb → Identify target → Execute action
- Do NOT make decisions about what to do next
- Do NOT optimize or substitute actions
- Do NOT repeat failed actions more than once

**Output Format**:
- ### Thought ###: What action and target does the subgoal specify?
- ### Action ###: `{"action": "action_name", "argument": "value"}`
- ### Description ###: One sentence describing the action

**Dynamic Variables**:
- Similar to system.jinja2

---

### 1.3 Manager Agent Prompts

#### File: `manager/system.jinja2`
**Purpose**: System prompt for ManagerAgent - high-level planning and reasoning agent

**Key Instructions**:
- Role: Agent that can operate Android phone on behalf of user
- Goal: Track progress and devise high-level plans
- Analyzes current state and creates plans with specific subgoals
- Tracks progress and decides when tasks are complete
- Uses XML-style tags for structured output

**Key Guidelines**:
- Use `open_app` action to open apps (not app drawer)
- Use search to find files/entries with specific names
- Only copy to clipboard when task requires it
- Store information in Memory section with step context
- File names and dates must match user request exactly
- Check full names (not truncated) in notifications
- Don't do more than what user asks for

**Special Features**:
- Memory usage: Append-only system with step context (e.g., "At step 5, I obtained recipe from source X")
- Text manipulation: Can use `TEXT_TASK:` prefix for modifying text in input fields
- Script execution: Can use `<script>` tags for off-device operations
- Custom tools: Can reference custom actions beyond standard actions
- Available secrets: Can reference secret IDs for sensitive operations

**Output Format**:
XML-style structured output with:
- `<thought>`: Explanation and rationale
- `<add_memory>`: Store important information with step context
- `<plan>`: Updated plan with subgoals
- `<request_accomplished success="true|false">`: Task completion status

**Dynamic Variables**:
- `instruction`: User request
- `device_date`: Current device date
- `app_card`: App operation information
- `important_notes`: Important context
- `error_history`: List of failed attempts
- `custom_tools_descriptions`: Custom action descriptions
- `available_secrets`: Available credential IDs
- `scripter_execution_enabled`: Whether script execution is available
- `text_manipulation_enabled`: Whether text manipulation is available
- `output_schema`: Expected output structure (optional)

---

#### File: `manager/rev1.jinja2`
**Purpose**: Alternative/simpler version of manager prompt

**Key Instructions**:
- Simplified format for planning
- Same core functionality but cleaner structure
- Uses markdown for formatting instead of verbose XML
- Focus on: Assess → Decide → Handle errors → Make assumptions

**Special Sections**:
- Planning guidelines
- Memory usage guidelines
- Text operations section
- Scripter operations section
- Output requirements for structured data

**Output Format**:
- `<thought>`: Explain reasoning
- `<add_memory>`: Store information with context
- `<plan>`: Numbered list of subgoals
- `<request_accomplished success="true|false">`: Completion message

**Dynamic Variables**:
- Same as manager/system.jinja2

---

### 1.4 Scripter Agent Prompt

#### File: `scripter/system.jinja2`
**Purpose**: System prompt for ScripterAgent - Python code execution for off-device operations

**Key Instructions**:
- Role: Python scripter agent for off-device operations
- No device control functions available
- Works like Jupyter notebook:
  1. Variables persist across code executions
  2. Can iteratively build solutions
  3. Previous results available
  4. Has max steps limit

**Available Libraries**:
- requests library
- json library
- Standard Python libraries

**Key Guidelines**:
1. Always explain thought process first before code
2. Write code incrementally (can execute multiple times)
3. Use print() to see intermediate results
4. Store results in variables for reuse
5. When finished, provide final message WITHOUT code block

**Tasks Include**:
- Downloading files from internet
- Making HTTP API calls
- Processing data (JSON, XML, CSV)
- Sending webhooks
- Any operation that doesn't involve device UI

**Output Format**:
- **Thought**: Explain what needs to be done
- ```python ... ```: Code block (optional, can skip final message)
- Final message without code block to signal completion

**Dynamic Variables**:
- `task`: The specific task to accomplish
- `available_libraries`: Available Python libraries
- `max_steps`: Maximum execution steps allowed

---

## 2. INLINE PROMPTS IN PYTHON FILES

### 2.1 Text Manipulation Agent
**File**: `/home/user/droidrun/droidrun/agent/oneflows/text_manipulator.py`

**Purpose**: Agent for modifying text in Android text input fields using constrained Python execution

**System Prompt**:
- Role: CODEACT_TEXT_AGENT - constrained Python code generator
- Constraints:
  - ONLY single Python code block
  - NO new functions, classes, or imports
  - Uses ONLY provided function: `input_text(text: str)`
  - Builds final content in triple-quoted string
  - Includes ORIGINAL variable if needed
  - Calls `input_text(new_text)` exactly once
- Strict format: Only fenced Python code block, NO printing
- Has access to ORIGINAL variable containing current text

**Error Correction Prompt**:
- Used when generated code fails
- Provides error stack trace and requests code fix
- Same rules as initial prompt
- Up to 3 retry attempts

**Variables Used**:
- `current_text`: Current content of focused text box
- `task_instruction`: Natural language instruction
- `ORIGINAL`: Current text in text box (in execution sandbox)
- `input_text`: Function to capture final text

---

### 2.2 Response Parsing Functions
**File**: `/home/user/droidrun/droidrun/agent/executor/prompts.py`

**Function**: `parse_executor_response()`
- Extracts structured data from executor's LLM response
- Parses sections: "### Thought", "### Action", "### Description"
- Returns: thought, action, description

---

### 2.3 Response Parsing Functions
**File**: `/home/user/droidrun/droidrun/agent/manager/prompts.py`

**Function**: `parse_manager_response()`
- Extracts structured data from manager's LLM response
- Parses XML-style tags:
  - `<thought>...</thought>`: Manager's reasoning
  - `<add_memory>...</add_memory>`: Memory to store
  - `<plan>...</plan>`: Task plan
  - `<request_accomplished success="true|false">...</request_accomplished>`: Task answer and success status
- Derives:
  - `current_subgoal`: First line of plan (cleaned)
  - `success`: Boolean parsed from success attribute
- Returns: thought, memory, plan, current_subgoal, answer, success

---

## 3. PROMPT RESOLVER & LOADER

### File: `/home/user/droidrun/droidrun/agent/utils/prompt_resolver.py`

**Class**: `PromptResolver`

**Purpose**: Resolves prompts from custom dict or falls back to file paths

**Valid Prompt Keys**:
- `codeact_system`: CodeAct system prompt
- `codeact_user`: CodeAct user prompt
- `manager_system`: Manager system prompt
- `executor_system`: Executor system prompt
- `scripter_system`: Scripter system prompt

**Features**:
- Allows custom Jinja2 template strings to override defaults
- Supports both file paths and inline templates
- Method: `get_prompt(prompt_key, fallback_path)` - returns custom template or None

---

### File: `/home/user/droidrun/droidrun/config_manager/prompt_loader.py`

**Class**: `PromptLoader`

**Purpose**: Loads and renders Jinja2 templates

**Features**:
- Loads from absolute file paths (resolved by AgentConfig + PathResolver)
- Conditional rendering: `{% if variable %}...{% endif %}`
- Loops with slicing: `{% for item in items[-5:] %}...{% endfor %}`
- Filters: `{{ variable|default("fallback") }}`
- Missing variables: silently ignored (render as empty string)
- Extra variables: silently ignored

**Methods**:
- `load_prompt(file_path, variables)`: Load and render from file
- `render_template(template_string, variables)`: Render from string

---

## 4. LLM CONFIGURATION & SELECTION

### File: `/home/user/droidrun/droidrun/agent/utils/llm_picker.py`

**Determines Required LLM Profiles**:

**For reasoning=True mode**:
- manager
- executor
- text_manipulator
- app_opener
- scripter (if enabled)
- structured_output (if output_model provided)

**For reasoning=False (direct execution) mode**:
- codeact
- app_opener
- structured_output (if output_model provided)

---

## 5. AGENT INTERACTION FLOW

### DroidAgent Architecture
**File**: `/home/user/droidrun/droidrun/agent/droid/droid_agent.py`

**Two Modes**:

1. **Direct Execution Mode (reasoning=False)**:
   - Uses CodeActAgent directly
   - Single LLM calls prompt and executes code
   - Fast but less strategic

2. **Reasoning Mode (reasoning=True)**:
   - Manager (Planning) → Executor (Actions) workflow
   - Manager creates plan and subgoals
   - Executor performs specific actions
   - Can delegate to ScripterAgent for off-device work
   - Can use TextManipulatorAgent for text editing
   - More strategic but slower

**Agents Involved**:
- CodeActAgent: Direct code execution
- ManagerAgent: Planning and reasoning
- ExecutorAgent: Action execution
- ScripterAgent: Off-device operations
- TextManipulatorAgent: Text field manipulation
- StructuredOutputAgent: Extract structured data from answers

---

## 6. SUMMARY TABLE

| Component | Type | Purpose | Key Variables |
|-----------|------|---------|---------------|
| codeact/system.jinja2 | Template | Main CodeAct system instructions | tool_descriptions, available_secrets, output_schema |
| codeact/user.jinja2 | Template | CodeAct user message | goal |
| executor/system.jinja2 | Template | Executor action instructions | instruction, device_state, plan, subgoal, atomic_actions |
| executor/rev1.jinja2 | Template | Simplified executor prompt | instruction, device_state, plan, subgoal, atomic_actions |
| manager/system.jinja2 | Template | Manager planning instructions | instruction, device_state, app_card, error_history, custom_tools_descriptions |
| manager/rev1.jinja2 | Template | Simplified manager prompt | instruction, device_state, app_card, error_history |
| scripter/system.jinja2 | Template | Scripter Python instructions | task, available_libraries, max_steps |
| text_manipulator.py | Inline | Text editing constraints and rules | current_text, task_instruction, ORIGINAL |
| prompt_resolver.py | Utility | Custom prompt resolution | custom_prompts dict |
| prompt_loader.py | Utility | Jinja2 template rendering | template_string, variables |
| executor/prompts.py | Parser | Response parsing for executor | response string |
| manager/prompts.py | Parser | Response parsing for manager | response string |

---

## 7. KEY CONCEPTS

### Jinja2 Template Features Used
- Conditional blocks: `{% if variable %}`
- Loops: `{% for item in collection %}`
- Filters: `{{ variable|default("fallback") }}`
- Variable interpolation: `{{ variable_name }}`

### Dynamic Variable System
All prompts use dynamic variable injection:
- Tool descriptions
- Device state formatting
- App card information
- Action history
- Secret credentials
- Custom tools
- Output schemas

### Structured Output
- Manager uses XML-style tags
- Executor uses JSON format
- CodeAct uses Python code with function calls
- All support structured data extraction

### Security Features
- Text Manipulator: Sandboxed Python execution
- Credentials: Secrets passed via `type_secret()` without exposing values
- ScripterAgent: Limited library access
- Safe execution modes with module whitelisting

---

## 8. CONFIGURATION HIERARCHY

**Prompt Loading Priority**:
1. Custom prompts passed to DroidAgent (highest priority)
2. File paths specified in agent config
3. Default file paths in `/config/prompts/`

**LLM Assignment**:
1. Agent-specific LLM from config profiles (manager, executor, codeact, etc.)
2. Single LLM for all agents (if only one provided)
3. Custom provider/model override

---

## 9. KEY FILE PATHS

- Prompt templates: `/home/user/droidrun/droidrun/config/prompts/`
- Prompt utilities: `/home/user/droidrun/droidrun/config_manager/`
- Agent prompts: `/home/user/droidrun/droidrun/agent/*/prompts.py`
- Agent implementations: `/home/user/droidrun/droidrun/agent/*/`
- LLM utilities: `/home/user/droidrun/droidrun/agent/utils/`

