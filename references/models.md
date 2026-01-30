# Models Reference

## Table of Contents
- [Model Classes](#model-classes)
- [Configuration Options](#configuration-options)
- [Provider Examples](#provider-examples)
- [Advanced Patterns](#advanced-patterns)

## Model Classes

| Class | Use For |
|-------|---------|
| `InferenceClientModel` | HuggingFace Inference API, serverless endpoints |
| `OpenAIModel` | OpenAI, OpenRouter, Together AI, any OpenAI-compatible API |
| `LiteLLMModel` | Anthropic, Ollama, Cohere, 100+ providers via LiteLLM |
| `TransformersModel` | Local HuggingFace models |
| `VLLMModel` | vLLM inference server |
| `MLXModel` | Apple Silicon local models |
| `AzureOpenAIModel` | Azure OpenAI deployments |
| `AmazonBedrockModel` | AWS Bedrock models |
| `LiteLLMRouterModel` | Load balancing across multiple providers |

## Configuration Options

Common parameters across model classes:

```python
model = SomeModel(
    model_id="model-name",           # Model identifier
    temperature=0.7,                  # Sampling temperature
    max_tokens=4096,                  # Max output tokens
    stop_sequences=["END"],           # Stop generation sequences
)
```

## Provider Examples

### HuggingFace Inference API

```python
from smolagents import InferenceClientModel

# Free tier (rate limited)
model = InferenceClientModel(model_id="Qwen/Qwen2.5-Coder-32B-Instruct")

# With API token
model = InferenceClientModel(
    model_id="meta-llama/Llama-3.3-70B-Instruct",
    token="hf_xxx",
)

# Serverless endpoint
model = InferenceClientModel(
    model_id="https://xxx.endpoints.huggingface.cloud",
    token="hf_xxx",
)
```

### OpenAI

```python
from smolagents import OpenAIModel

model = OpenAIModel(model_id="gpt-4o")
model = OpenAIModel(model_id="gpt-4o-mini")
model = OpenAIModel(model_id="o1-preview")
```

### Anthropic (via LiteLLM)

```python
from smolagents import LiteLLMModel

model = LiteLLMModel(model_id="anthropic/claude-sonnet-4-20250514")
model = LiteLLMModel(model_id="anthropic/claude-opus-4-20250514")
```

### Local with Ollama

```python
from smolagents import LiteLLMModel

# Ensure Ollama is running: ollama serve
model = LiteLLMModel(model_id="ollama/llama3.2")
model = LiteLLMModel(model_id="ollama/codellama")
model = LiteLLMModel(model_id="ollama/mistral")
```

### Local with Transformers

```python
from smolagents import TransformersModel

model = TransformersModel(
    model_id="Qwen/Qwen2.5-Coder-7B-Instruct",
    device_map="auto",
    torch_dtype="auto",
)
```

### Local with MLX (Apple Silicon)

```python
from smolagents import MLXModel

model = MLXModel(model_id="mlx-community/Qwen2.5-Coder-32B-Instruct-4bit")
```

### vLLM Server

```python
from smolagents import VLLMModel

# Start vLLM: python -m vllm.entrypoints.openai.api_server --model Qwen/Qwen2.5-7B-Instruct
model = VLLMModel(
    model_id="Qwen/Qwen2.5-7B-Instruct",
    base_url="http://localhost:8000/v1",
)
```

### Azure OpenAI

```python
from smolagents import AzureOpenAIModel

model = AzureOpenAIModel(
    model_id="gpt-4o",
    azure_endpoint="https://xxx.openai.azure.com",
    api_version="2024-02-15-preview",
)
```

### AWS Bedrock

```python
from smolagents import AmazonBedrockModel

model = AmazonBedrockModel(model_id="anthropic.claude-3-sonnet-20240229-v1:0")
model = AmazonBedrockModel(model_id="amazon.titan-text-express-v1")
```

### OpenRouter

```python
from smolagents import OpenAIModel

model = OpenAIModel(
    model_id="anthropic/claude-3.5-sonnet",
    api_base="https://openrouter.ai/api/v1",
    api_key="sk-or-xxx",
)
```

### Together AI

```python
from smolagents import OpenAIModel

model = OpenAIModel(
    model_id="meta-llama/Llama-3.3-70B-Instruct-Turbo",
    api_base="https://api.together.xyz/v1",
    api_key="xxx",
)
```

## Advanced Patterns

### Load Balancing with Router

```python
from smolagents import LiteLLMRouterModel
import os

model_list = [
    {
        "model_name": "main",
        "litellm_params": {
            "model": "gpt-4o",
            "api_key": os.getenv("OPENAI_API_KEY"),
        },
    },
    {
        "model_name": "main",
        "litellm_params": {
            "model": "anthropic/claude-sonnet-4-20250514",
            "api_key": os.getenv("ANTHROPIC_API_KEY"),
        },
    },
]

model = LiteLLMRouterModel(
    model_id="main",
    model_list=model_list,
    client_kwargs={"routing_strategy": "simple-shuffle"},  # or "latency-based-routing"
)
```

### Custom Model Class

```python
from smolagents.models import Model, ChatMessage

class CustomModel(Model):
    def __init__(self, model_id: str, **kwargs):
        super().__init__(**kwargs)
        self.model_id = model_id

    def generate(
        self,
        messages: list[ChatMessage],
        stop_sequences: list[str] | None = None,
        **kwargs,
    ) -> ChatMessage:
        # Your implementation
        response_text = self._call_api(messages)
        return ChatMessage(role="assistant", content=response_text)
```

### Structured Output

```python
from pydantic import BaseModel

class WeatherResponse(BaseModel):
    temperature: float
    conditions: str
    humidity: int

# Models with native structured output support
model = OpenAIModel(model_id="gpt-4o")
response = model.generate(
    messages=[...],
    response_format=WeatherResponse,
)
# response.content will be parsed WeatherResponse
```
