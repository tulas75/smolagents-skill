# Tools Reference

## Table of Contents
- [Tool Types](#tool-types)
- [Input Types](#input-types)
- [Output Types](#output-types)
- [Advanced Patterns](#advanced-patterns)
- [Built-in Tools](#built-in-tools)
- [Tool Integrations](#tool-integrations)

## Tool Types

### Decorator-based (`@tool`)

Best for simple, stateless tools:

```python
from smolagents import tool

@tool
def calculate_area(length: float, width: float) -> float:
    """Calculate rectangle area.

    Args:
        length: Rectangle length in meters.
        width: Rectangle width in meters.
    """
    return length * width
```

### Class-based (`Tool`)

Best for tools with state, initialization, or complex logic:

```python
from smolagents import Tool

class APITool(Tool):
    name = "api_client"
    description = "Call external API. Returns JSON response as string."
    inputs = {
        "endpoint": {"type": "string", "description": "API endpoint path"},
        "method": {"type": "string", "description": "HTTP method", "nullable": True},
    }
    output_type = "string"

    def __init__(self, base_url: str, api_key: str, **kwargs):
        super().__init__(**kwargs)
        self.base_url = base_url
        self.headers = {"Authorization": f"Bearer {api_key}"}

    def forward(self, endpoint: str, method: str = "GET") -> str:
        import requests
        resp = requests.request(method, f"{self.base_url}/{endpoint}", headers=self.headers)
        return resp.text
```

## Input Types

Valid types for `inputs` dict:

| Type | Python | Description |
|------|--------|-------------|
| `string` | `str` | Text input |
| `integer` | `int` | Whole numbers |
| `number` | `float` | Decimal numbers |
| `boolean` | `bool` | True/False |
| `array` | `list` | List of items |
| `object` | `dict` | Key-value pairs |
| `image` | `AgentImage` | Image data |
| `audio` | `AgentAudio` | Audio data |
| `any` | `Any` | Any type |

### Optional parameters

Use `nullable: True` for optional inputs:

```python
inputs = {
    "query": {"type": "string", "description": "Search query"},
    "limit": {"type": "integer", "description": "Max results", "nullable": True},
}
```

## Output Types

Valid `output_type` values: `string`, `integer`, `number`, `boolean`, `image`, `audio`, `object`, `array`, `any`

### Returning images

```python
from smolagents import Tool
from smolagents.agent_types import AgentImage

class ChartTool(Tool):
    name = "create_chart"
    description = "Create a bar chart from data."
    inputs = {"data": {"type": "object", "description": "Chart data dict"}}
    output_type = "image"

    def forward(self, data: dict) -> AgentImage:
        import matplotlib.pyplot as plt
        import io

        fig, ax = plt.subplots()
        ax.bar(data.keys(), data.values())
        buf = io.BytesIO()
        fig.savefig(buf, format='png')
        buf.seek(0)
        return AgentImage(buf.read())
```

## Advanced Patterns

### Tool with file handling

```python
class FileProcessorTool(Tool):
    name = "process_file"
    description = "Process uploaded file. Returns processing result."
    inputs = {
        "file_path": {"type": "string", "description": "Path to file"},
        "operation": {"type": "string", "description": "Operation: count, summarize, extract"},
    }
    output_type = "string"

    def forward(self, file_path: str, operation: str) -> str:
        with open(file_path, 'r') as f:
            content = f.read()

        if operation == "count":
            return f"Words: {len(content.split())}, Lines: {len(content.splitlines())}"
        # ... other operations
```

### Tool with async support

```python
class AsyncAPITool(Tool):
    name = "async_api"
    description = "Async API call."
    inputs = {"url": {"type": "string", "description": "URL to fetch"}}
    output_type = "string"

    async def forward(self, url: str) -> str:
        import aiohttp
        async with aiohttp.ClientSession() as session:
            async with session.get(url) as resp:
                return await resp.text()
```

### Tool validation

```python
class ValidatedTool(Tool):
    name = "validated_tool"
    description = "Tool with input validation."
    inputs = {
        "email": {"type": "string", "description": "User email address"},
    }
    output_type = "string"

    def forward(self, email: str) -> str:
        import re
        if not re.match(r"[^@]+@[^@]+\.[^@]+", email):
            raise ValueError(f"Invalid email format: {email}")
        return f"Validated: {email}"
```

## Built-in Tools

Import from `smolagents.default_tools`:

| Tool | Description |
|------|-------------|
| `WebSearchTool` | Search the web (uses DuckDuckGo) |
| `VisitWebpageTool` | Fetch and parse webpage content |
| `DuckDuckGoSearchTool` | DuckDuckGo search |
| `GoogleSearchTool` | Google search (requires API key) |
| `WikipediaSearchTool` | Search Wikipedia |
| `PythonInterpreterTool` | Execute Python code |
| `FinalAnswerTool` | Return final answer (auto-included) |

```python
from smolagents.default_tools import WebSearchTool, VisitWebpageTool

agent = CodeAgent(
    tools=[WebSearchTool(), VisitWebpageTool()],
    model=model,
)
```

## Tool Integrations

### From Hugging Face Hub

```python
from smolagents import Tool

tool = Tool.from_hub("username/tool-name")
agent = CodeAgent(tools=[tool], model=model)
```

### From LangChain

```python
from smolagents import Tool
from langchain_community.tools import WikipediaQueryRun

langchain_tool = WikipediaQueryRun(...)
smolagent_tool = Tool.from_langchain(langchain_tool)
```

### From MCP Server

```python
from smolagents import ToolCollection

# Load all tools from MCP server
tools = ToolCollection.from_mcp("http://localhost:8000")
agent = CodeAgent(tools=tools.tools, model=model)
```

### Push tool to Hub

```python
my_tool = MyCustomTool()
my_tool.push_to_hub("username/my-tool")
```
