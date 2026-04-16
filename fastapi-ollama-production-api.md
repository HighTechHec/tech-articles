# Build a Production-Ready AI API with FastAPI and Ollama

Running AI inference locally has become practical. With Ollama handling model serving and FastAPI wrapping it in a clean HTTP interface, you can ship a production-grade AI API in an afternoon — no cloud costs, no rate limits, no data leaving your infrastructure.

This guide builds a real API: streaming responses, async inference, model routing, request validation, rate limiting, and health checks. Everything you need before you go to production.

## What We're Building

A FastAPI service that:
- Wraps Ollama inference with a clean REST API
- Streams responses for real-time output
- Routes requests to different models based on task type
- Enforces rate limits per client
- Exposes health checks and model metrics
- Validates requests with Pydantic

## Prerequisites

```bash
ollama pull llama3.2:3b      # fast, for simple tasks
ollama pull gemma3:latest    # balanced quality/speed
pip install fastapi uvicorn httpx slowapi pydantic
```

## Project Structure

```
ai-api/
├── main.py           # FastAPI app + routes
├── ollama_client.py  # Ollama wrapper with async streaming
├── models.py         # Pydantic schemas
├── limiter.py        # Rate limiting
└── config.py         # Settings
```

## Step 1: Configuration

```python
# config.py
from pydantic_settings import BaseSettings
from typing import Literal


class Settings(BaseSettings):
    ollama_base_url: str = "http://localhost:11434"
    default_model: str = "llama3.2:3b"

    # Route task types to specific models
    model_routing: dict[str, str] = {
        "code": "qwen2.5-coder:7b",
        "analysis": "gemma3:latest",
        "chat": "llama3.2:3b",
        "summarize": "llama3.2:3b",
    }

    max_tokens: int = 2048
    request_timeout: float = 120.0
    rate_limit: str = "20/minute"  # slowapi format

    class Config:
        env_prefix = "AI_API_"


settings = Settings()
```

## Step 2: Pydantic Schemas

```python
# models.py
from pydantic import BaseModel, Field, field_validator
from typing import Literal, Optional


class GenerateRequest(BaseModel):
    prompt: str = Field(..., min_length=1, max_length=10000)
    task_type: Literal["chat", "code", "analysis", "summarize"] = "chat"
    model: Optional[str] = None           # override routing
    temperature: float = Field(0.7, ge=0.0, le=2.0)
    max_tokens: int = Field(512, ge=1, le=4096)
    stream: bool = False
    system: Optional[str] = None

    @field_validator("prompt")
    @classmethod
    def strip_prompt(cls, v: str) -> str:
        return v.strip()


class GenerateResponse(BaseModel):
    response: str
    model: str
    task_type: str
    prompt_tokens: int
    completion_tokens: int
    duration_ms: float


class ModelInfo(BaseModel):
    name: str
    size: str
    task_types: list[str]
    status: str


class HealthResponse(BaseModel):
    status: str
    ollama_connected: bool
    models_available: list[str]
    uptime_seconds: float
```

## Step 3: Ollama Client

```python
# ollama_client.py
import httpx
import json
import time
from typing import AsyncIterator
from config import settings


class OllamaClient:
    def __init__(self):
        self.base_url = settings.ollama_base_url
        self._client = httpx.AsyncClient(timeout=settings.request_timeout)

    async def generate(
        self,
        model: str,
        prompt: str,
        system: str | None = None,
        temperature: float = 0.7,
        max_tokens: int = 512,
    ) -> tuple[str, dict]:
        """Non-streaming generation. Returns (text, stats)."""
        payload = {
            "model": model,
            "prompt": prompt,
            "stream": False,
            "options": {"temperature": temperature, "num_predict": max_tokens},
        }
        if system:
            payload["system"] = system

        start = time.perf_counter()
        r = await self._client.post(f"{self.base_url}/api/generate", json=payload)
        r.raise_for_status()
        elapsed = (time.perf_counter() - start) * 1000

        data = r.json()
        return data["response"], {
            "prompt_tokens": data.get("prompt_eval_count", 0),
            "completion_tokens": data.get("eval_count", 0),
            "duration_ms": elapsed,
        }

    async def generate_stream(
        self,
        model: str,
        prompt: str,
        system: str | None = None,
        temperature: float = 0.7,
        max_tokens: int = 512,
    ) -> AsyncIterator[str]:
        """Streaming generation. Yields text chunks."""
        payload = {
            "model": model,
            "prompt": prompt,
            "stream": True,
            "options": {"temperature": temperature, "num_predict": max_tokens},
        }
        if system:
            payload["system"] = system

        async with self._client.stream(
            "POST", f"{self.base_url}/api/generate", json=payload
        ) as response:
            response.raise_for_status()
            async for line in response.aiter_lines():
                if not line:
                    continue
                try:
                    chunk = json.loads(line)
                    if token := chunk.get("response"):
                        yield token
                    if chunk.get("done"):
                        break
                except json.JSONDecodeError:
                    continue

    async def list_models(self) -> list[str]:
        """List available models."""
        try:
            r = await self._client.get(f"{self.base_url}/api/tags")
            r.raise_for_status()
            return [m["name"] for m in r.json().get("models", [])]
        except Exception:
            return []

    async def close(self):
        await self._client.aclose()
```

## Step 4: Rate Limiter

```python
# limiter.py
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
```

## Step 5: Main Application

```python
# main.py
import time
import asyncio
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request, HTTPException, Depends
from fastapi.responses import StreamingResponse
from slowapi.errors import RateLimitExceeded
from slowapi import _rate_limit_exceeded_handler

from config import settings
from models import GenerateRequest, GenerateResponse, HealthResponse, ModelInfo
from ollama_client import OllamaClient
from limiter import limiter

START_TIME = time.time()
ollama: OllamaClient = None


@asynccontextmanager
async def lifespan(app: FastAPI):
    global ollama
    ollama = OllamaClient()
    yield
    await ollama.close()


app = FastAPI(
    title="JAX AI API",
    description="Local AI inference API powered by Ollama",
    version="1.0.0",
    lifespan=lifespan,
)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)


def resolve_model(request: GenerateRequest) -> str:
    """Route to the right model based on task type."""
    if request.model:
        return request.model
    return settings.model_routing.get(request.task_type, settings.default_model)


@app.get("/health", response_model=HealthResponse)
async def health():
    models = await ollama.list_models()
    return HealthResponse(
        status="healthy" if models else "degraded",
        ollama_connected=bool(models),
        models_available=models[:10],
        uptime_seconds=time.time() - START_TIME,
    )


@app.get("/models", response_model=list[ModelInfo])
async def list_models():
    models = await ollama.list_models()
    routing_reverse: dict[str, list[str]] = {}
    for task, model in settings.model_routing.items():
        routing_reverse.setdefault(model, []).append(task)

    result = []
    for name in models:
        result.append(ModelInfo(
            name=name,
            size="unknown",
            task_types=routing_reverse.get(name, ["general"]),
            status="available",
        ))
    return result


@app.post("/generate", response_model=GenerateResponse)
@limiter.limit(settings.rate_limit)
async def generate(request: Request, body: GenerateRequest):
    model = resolve_model(body)

    if body.stream:
        raise HTTPException(
            status_code=400,
            detail="Use /generate/stream for streaming responses"
        )

    try:
        text, stats = await ollama.generate(
            model=model,
            prompt=body.prompt,
            system=body.system,
            temperature=body.temperature,
            max_tokens=body.max_tokens,
        )
    except Exception as e:
        raise HTTPException(status_code=502, detail=f"Ollama error: {e}")

    return GenerateResponse(
        response=text,
        model=model,
        task_type=body.task_type,
        **stats,
    )


@app.post("/generate/stream")
@limiter.limit(settings.rate_limit)
async def generate_stream(request: Request, body: GenerateRequest):
    model = resolve_model(body)

    async def event_stream():
        try:
            async for chunk in ollama.generate_stream(
                model=model,
                prompt=body.prompt,
                system=body.system,
                temperature=body.temperature,
                max_tokens=body.max_tokens,
            ):
                yield f"data: {chunk}\n\n"
        except Exception as e:
            yield f"data: [ERROR: {e}]\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(
        event_stream(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",
        },
    )


@app.post("/v1/chat/completions")
@limiter.limit(settings.rate_limit)
async def openai_compat(request: Request):
    """OpenAI-compatible endpoint for drop-in replacement."""
    body = await request.json()
    messages = body.get("messages", [])
    if not messages:
        raise HTTPException(status_code=400, detail="No messages provided")

    system = next((m["content"] for m in messages if m["role"] == "system"), None)
    user_msgs = [m["content"] for m in messages if m["role"] == "user"]
    prompt = "\n".join(user_msgs)

    model = body.get("model", settings.default_model)
    temperature = body.get("temperature", 0.7)
    max_tokens = body.get("max_tokens", 512)
    stream = body.get("stream", False)

    if stream:
        async def sse_stream():
            import json as _json
            async for chunk in ollama.generate_stream(model, prompt, system, temperature, max_tokens):
                data = {
                    "choices": [{"delta": {"content": chunk}, "finish_reason": None}]
                }
                yield f"data: {_json.dumps(data)}\n\n"
            yield "data: [DONE]\n\n"

        return StreamingResponse(sse_stream(), media_type="text/event-stream")

    text, stats = await ollama.generate(model, prompt, system, temperature, max_tokens)
    return {
        "choices": [{"message": {"role": "assistant", "content": text}, "finish_reason": "stop"}],
        "model": model,
        "usage": {
            "prompt_tokens": stats["prompt_tokens"],
            "completion_tokens": stats["completion_tokens"],
        }
    }
```

## Step 6: Run It

```bash
uvicorn main:app --host 0.0.0.0 --port 8080 --workers 4
```

## Testing

```bash
# Health check
curl http://localhost:8080/health

# Generate (non-streaming)
curl -X POST http://localhost:8080/generate \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Write a Python function to parse JWT tokens",
    "task_type": "code",
    "max_tokens": 300
  }'

# Streaming
curl -N -X POST http://localhost:8080/generate/stream \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Explain async/await in Python", "task_type": "chat"}'

# OpenAI-compatible (works with any OpenAI SDK client)
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.2:3b",
    "messages": [{"role": "user", "content": "What is a database index?"}]
  }'
```

## Dockerize It

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8080
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080", "--workers", "4"]
```

`requirements.txt`:
```
fastapi>=0.110.0
uvicorn[standard]>=0.29.0
httpx>=0.27.0
slowapi>=0.1.9
pydantic>=2.0.0
pydantic-settings>=2.0.0
```

```bash
docker build -t ai-api .
docker run -d \
  --network host \           # access host's Ollama
  --name ai-api \
  -e AI_API_OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  ai-api
```

## Production Hardening

**Add API key auth:**

```python
from fastapi.security import APIKeyHeader
from fastapi import Security

api_key_header = APIKeyHeader(name="X-API-Key")
API_KEYS = set(os.getenv("API_KEYS", "").split(","))

async def verify_api_key(api_key: str = Security(api_key_header)):
    if api_key not in API_KEYS:
        raise HTTPException(status_code=403, detail="Invalid API key")
    return api_key

# Add to routes: @app.post("/generate", dependencies=[Depends(verify_api_key)])
```

**Nginx reverse proxy:**

```nginx
upstream ai_api {
    server 127.0.0.1:8080;
    keepalive 32;
}

server {
    listen 443 ssl;
    server_name your-api.internal;

    location / {
        proxy_pass http://ai_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering off;           # critical for streaming
        proxy_read_timeout 120s;
    }
}
```

## What You Get

The OpenAI-compatible `/v1/chat/completions` endpoint is the key feature: any library that speaks OpenAI — LangChain, LlamaIndex, your custom code — works against your local Ollama without changing a line.

Model routing keeps your API clean: callers specify a task type (`code`, `analysis`, `chat`), and the API routes to the right model. Swap models by changing `config.py`, not client code.

The rate limiter (slowapi + Redis for distributed deployments) prevents any single client from flooding your GPU. Combined with async streaming, the API stays responsive under concurrent load.

This is the same pattern used in production AI gateways — LiteLLM, OpenRouter, and similar tools all follow this architecture. Building it yourself gives you full control over routing logic, cost tracking, and model selection without vendor lock-in.
