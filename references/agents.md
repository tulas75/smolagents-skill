# Agents Reference

## Table of Contents
- [Agent Types](#agent-types)
- [Configuration Options](#configuration-options)
- [Multi-Agent Systems](#multi-agent-systems)
- [Memory System](#memory-system)
- [Custom Prompts](#custom-prompts)
- [Callbacks and Monitoring](#callbacks-and-monitoring)

## Agent Types

### CodeAgent

Writes and executes Python code. Best for complex reasoning.

```python
from smolagents import CodeAgent

agent = CodeAgent(
    tools=[...],
    model=model,
    executor_type="local",  # or "docker", "e2b"
    additional_authorized_imports=["pandas", "numpy"],
)
```

### ToolCallingAgent

Uses JSON-based tool calls. Simpler, no code execution.

```python
from smolagents import ToolCallingAgent

agent = ToolCallingAgent(
    tools=[...],
    model=model,
)
```

## Configuration Options

```python
agent = CodeAgent(
    # Required
    tools=[tool1, tool2],
    model=model,

    # Behavior
    max_steps=20,                    # Max reasoning steps
    planning_interval=None,          # Steps between planning (None=disabled)

    # Output
    verbosity_level=1,               # 0=errors, 1=info, 2=debug
    stream_outputs=True,             # Stream as generated

    # Execution (CodeAgent only)
    executor_type="local",           # local, docker, e2b, modal, blaxel
    additional_authorized_imports=["requests"],
    max_print_outputs_length=10000,  # Truncate long outputs

    # Prompts
    instructions="You are a helpful assistant...",  # Added to system prompt
    prompt_templates={...},          # Custom prompt templates
)
```

### Run Options

```python
# Basic run
result = agent.run("What is 2+2?")

# With full result object
result = agent.run("task", return_full_result=True)
print(result.output)       # Final answer
print(result.steps)        # List of ActionStep, FinalAnswerStep, etc.
print(result.token_usage)  # TokenUsage(input_tokens, output_tokens, total_tokens)
print(result.timing)       # Timing(start_time, end_time, duration)
print(result.state)        # "success" or "max_steps_error"

# Reset between runs
agent.run("task 1")
agent.run("task 2", reset=True)  # Clear memory between tasks
```

## Multi-Agent Systems

### Managed Agents Pattern

Manager agent delegates to specialist agents:

```python
from smolagents import CodeAgent, ToolCallingAgent

# Specialist: web research
web_agent = ToolCallingAgent(
    tools=[WebSearchTool(), VisitWebpageTool()],
    model=model,
    name="web_researcher",  # Required for managed agents
    description="Searches and reads web pages for information",
)

# Specialist: data analysis
data_agent = CodeAgent(
    tools=[],
    model=model,
    name="data_analyst",
    description="Analyzes data and creates visualizations",
    additional_authorized_imports=["pandas", "matplotlib"],
)

# Manager coordinates specialists
manager = CodeAgent(
    tools=[],  # Manager uses managed_agents instead
    model=model,
    managed_agents=[web_agent, data_agent],
)

result = manager.run("Research AI trends and create a summary chart")
```

### Agent as Tool

Use any agent as a tool in another agent:

```python
from smolagents import CodeAgent

specialist = CodeAgent(
    tools=[...],
    model=model,
    name="specialist",
    description="Does specialized work",
)

# Parent treats specialist as managed agent
parent = CodeAgent(
    tools=[other_tools],
    model=model,
    managed_agents=[specialist],
)
```

## Memory System

Agents maintain memory through structured steps:

```python
from smolagents.memory import (
    AgentMemory,
    SystemPromptStep,
    TaskStep,
    ActionStep,
    PlanningStep,
    FinalAnswerStep,
)

# Access memory
agent.memory  # AgentMemory instance

# Get all messages (for debugging)
messages = agent.memory.get_messages()
for msg in messages:
    print(msg["role"], msg.get("content", "")[:100])

# Memory steps after run
result = agent.run("task", return_full_result=True)
for step in result.steps:
    if isinstance(step, ActionStep):
        print(f"Action: {step.tool_calls}")
        print(f"Observation: {step.observations}")
    elif isinstance(step, FinalAnswerStep):
        print(f"Answer: {step.answer}")
```

### Step Types

| Step | Description |
|------|-------------|
| `SystemPromptStep` | Initial system prompt |
| `TaskStep` | User task/request |
| `PlanningStep` | Agent's planning output |
| `ActionStep` | Tool calls and observations |
| `FinalAnswerStep` | Final answer |

## Custom Prompts

Override default prompts:

```python
agent = CodeAgent(
    tools=[...],
    model=model,
    prompt_templates={
        "system_prompt": """You are a specialized assistant.
Your job is to help with {specific_domain}.
Available tools: {{tool_descriptions}}
{{managed_agents_descriptions}}
""",
        "planning": {
            "initial_plan": "Create a plan for: {{task}}",
            "update_plan_pre_messages": "Review progress...",
            "update_plan_post_messages": "Update your plan...",
        },
        "managed_agent": {
            "task": "You are a managed agent. Your task: {{task}}",
            "report": "Report your findings...",
        },
        "final_answer": {
            "pre_messages": "Now provide your final answer.",
            "post_messages": "Remember to be concise.",
        },
    },
)
```

### Prompt Variables

Available variables in templates:
- `{{tool_descriptions}}` - Formatted tool list
- `{{managed_agents_descriptions}}` - Managed agent descriptions
- `{{task}}` - Current task
- `{{memory}}` - Conversation history

## Callbacks and Monitoring

### Step Callbacks

```python
from smolagents.memory import ActionStep, FinalAnswerStep

def on_action(step: ActionStep):
    print(f"Step {step.step_number}")
    print(f"Tools: {step.tool_calls}")
    print(f"Tokens: {step.token_usage}")

def on_final(step: FinalAnswerStep):
    print(f"Final answer: {step.answer}")

agent = CodeAgent(
    tools=[...],
    model=model,
    step_callbacks={
        ActionStep: on_action,
        FinalAnswerStep: on_final,
    },
)
```

### Logging Levels

```python
from smolagents.monitoring import LogLevel

agent = CodeAgent(
    tools=[...],
    model=model,
    verbosity_level=LogLevel.DEBUG,  # Most verbose
)

# LogLevel options:
# - ERROR (0): Only errors
# - WARNING (1): Warnings and errors
# - INFO (2): General info
# - DEBUG (3): Detailed debug output
```

### Token Tracking

```python
result = agent.run("task", return_full_result=True)

# Total usage
print(result.token_usage)
# TokenUsage(input_tokens=1234, output_tokens=567, total_tokens=1801)

# Per-step usage
for step in result.steps:
    if hasattr(step, 'token_usage') and step.token_usage:
        print(f"Step {step.step_number}: {step.token_usage}")
```
