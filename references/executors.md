# Executors Reference

## Table of Contents
- [Executor Types](#executor-types)
- [Security Considerations](#security-considerations)
- [Configuration](#configuration)
- [Choosing an Executor](#choosing-an-executor)

## Executor Types

CodeAgent executes Python code. Choose executor based on security needs:

| Executor | Security | Setup | Best For |
|----------|----------|-------|----------|
| `local` | Low | None | Development, trusted code |
| `docker` | High | Docker installed | Self-hosted production |
| `e2b` | High | E2B account | Cloud production |
| `modal` | High | Modal account | Serverless production |
| `blaxel` | High | Blaxel account | Cloud production |

### Local Executor (Default)

Runs in-process with restricted imports. **Not fully sandboxed.**

```python
from smolagents import CodeAgent

agent = CodeAgent(
    tools=[...],
    model=model,
    executor_type="local",
    additional_authorized_imports=["pandas", "numpy", "requests"],
)
```

Default authorized imports:
- `math`, `random`, `datetime`, `time`, `json`, `re`
- `collections`, `itertools`, `functools`
- `statistics`, `unicodedata`, `queue`

### Docker Executor

Runs code in isolated Docker container.

```python
agent = CodeAgent(
    tools=[...],
    model=model,
    executor_type="docker",
)

# Or with context manager for cleanup
with CodeAgent(tools=[...], model=model, executor_type="docker") as agent:
    result = agent.run("task")
```

Requirements:
- Docker installed and running
- Docker SDK: `pip install docker`

### E2B Executor

Runs in E2B cloud sandbox.

```python
import os
os.environ["E2B_API_KEY"] = "your-key"

agent = CodeAgent(
    tools=[...],
    model=model,
    executor_type="e2b",
)
```

Requirements:
- E2B account and API key
- `pip install e2b-code-interpreter`

### Modal Executor

Runs on Modal serverless platform.

```python
agent = CodeAgent(
    tools=[...],
    model=model,
    executor_type="modal",
)
```

Requirements:
- Modal account configured
- `pip install modal`
- `modal token new` to authenticate

### Blaxel Executor

Runs in Blaxel cloud sandbox.

```python
agent = CodeAgent(
    tools=[...],
    model=model,
    executor_type="blaxel",
)
```

Requirements:
- Blaxel account
- Blaxel SDK configured

## Security Considerations

### Local Executor Risks

The local executor has restrictions but is **not a security boundary**:

- Restricted imports can be bypassed
- File system access possible
- Network access possible
- No process isolation

**Never use local executor with untrusted input in production.**

### Sandboxed Executor Benefits

Docker, E2B, Modal, Blaxel provide:

- Process isolation
- Network restrictions (configurable)
- File system isolation
- Resource limits
- Timeout enforcement

### Best Practices

1. **Development**: Local executor is fine for testing
2. **Production with trusted users**: Docker recommended
3. **Production with untrusted input**: E2B, Modal, or Blaxel
4. **Limit imports**: Only allow needed packages
5. **Set timeouts**: Prevent runaway execution

## Configuration

### Local Executor Options

```python
agent = CodeAgent(
    tools=[...],
    model=model,
    executor_type="local",

    # Authorized imports
    additional_authorized_imports=[
        "pandas",
        "numpy",
        "requests",
        "sklearn",
    ],

    # Output limits
    max_print_outputs_length=10000,  # Truncate long outputs
)
```

### Docker Executor Options

```python
from smolagents.remote_executors import DockerExecutor

executor = DockerExecutor(
    image="python:3.11-slim",  # Base image
    pip_packages=["pandas", "numpy"],  # Install packages
)

agent = CodeAgent(
    tools=[...],
    model=model,
    executor=executor,
)
```

### Timeout Configuration

All executors enforce execution timeout (default 30 seconds):

```python
# Timeout is enforced at executor level
# Long-running code will be terminated
```

## Choosing an Executor

### Decision Flow

```
Is this development/testing?
├── Yes → Use "local"
└── No → Is input from trusted users?
    ├── Yes → Use "docker" (self-hosted) or "e2b" (cloud)
    └── No → Use "e2b", "modal", or "blaxel" (fully sandboxed cloud)
```

### Comparison

| Factor | local | docker | e2b/modal/blaxel |
|--------|-------|--------|------------------|
| Setup | None | Docker install | Account + API key |
| Cost | Free | Free (self-hosted) | Pay per use |
| Security | Low | High | High |
| Latency | Lowest | Low | Higher |
| Scaling | No | Manual | Automatic |

### Migration Path

Start with local for development:

```python
# Development
agent = CodeAgent(tools=[...], model=model, executor_type="local")
```

Switch to Docker for production:

```python
# Production
agent = CodeAgent(tools=[...], model=model, executor_type="docker")
```

Or cloud sandbox:

```python
# Production (cloud)
agent = CodeAgent(tools=[...], model=model, executor_type="e2b")
```
