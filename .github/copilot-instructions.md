# Copilot Instructions for OpenAI Python SDK

## Architecture Overview

This SDK is **generated code** from OpenAPI specs using [Stainless](https://stainlessapi.com/). Most files in `src/openai/` include the header comment "File generated from our OpenAPI spec by Stainless."

- **Generated code**: Core client (`_client.py`), resources, type definitions, exceptions
- **Preserved across regeneration**: `src/openai/lib/`, `examples/`, manual patches in generated files
- **Manual extensions**: `src/openai/lib/azure.py` (Azure support), `src/openai/lib/_tools.py` (Pydantic tool parsing)

### Key Entry Points

- **Sync client**: `OpenAI(api_key=...)` in [src/openai/_client.py](src/openai/_client.py)
- **Async client**: `AsyncOpenAI(api_key=...)` for async operations
- **Base HTTP layer**: [src/openai/_base_client.py](src/openai/_base_client.py) (httpx-based, retry logic, auth headers)
- **Resources pattern**: [src/openai/resources/](src/openai/resources/) organizes API endpoints (e.g., `chat/completions`, `images`, `audio`)

## Resource/API Structure

Resources inherit from `SyncAPIResource` or `AsyncAPIResource` (in [_resource.py](src/openai/_resource.py)) which provide:
- `self._get()`, `self._post()`, `self._patch()`, `self._put()`, `self._delete()` methods
- `self._get_api_list()` for pagination support

Example resource pattern:
```python
# src/openai/resources/chat/completions/completions.py
class Completions(SyncAPIResource):
    def create(self, **params) -> ChatCompletion:
        return self._post("/chat/completions", cast_to=ChatCompletion, **params)
```

## Type System & Validation

- **Pydantic-based**: All types in [src/openai/types/](src/openai/types/) use Pydantic for validation
- **Support both v1 & v2**: `pyproject.toml` declares `pydantic>=1.9.0,<3`
- **Structured outputs**: Use `client.chat.completions.parse(response_format=MyModel)` for JSON schema extraction (see [helpers.md](helpers.md))
- **Tool parsing**: `openai.pydantic_function_tool()` for strict tool schemas; see [src/openai/lib/_tools.py](src/openai/lib/_tools.py)
- **Pydantic validation**: Manual patching in generated files via comments to preserve across regeneration

## Development Workflow

### Setup
```bash
./scripts/bootstrap  # Uses Rye to set up Python environment
# OR manually
pip install -r requirements-dev.lock
```

### Linting & Formatting
```bash
./scripts/lint       # Check ruff, run type checks (pyright/mypy)
./scripts/format     # Format with ruff + black, fix issues
```

### Testing

**Key test setup** ([tests/conftest.py](tests/conftest.py)):
- Base URL: `http://127.0.0.1:4010` (default mock server)
- Tests use **respx** to mock HTTP responses (not actual API calls)
- Async tests auto-marked with `@pytest.mark.asyncio(loop_scope="session")`
- Fixtures: `client` (sync), `async_client` (async) with optional `http_client="aiohttp"`

```bash
./scripts/test       # Run pytest (requires mock OpenAPI server)
```

Mock server setup:
```bash
npx prism mock path/to/openapi.yml
```

### Code Generation

**Do NOT manually edit generated code extensively** without understanding merge conflict implications:
- Generator overwrites most files
- Patch manual changes into existing files (use comments to mark edits)
- Use `examples/` and `src/openai/lib/` for new, non-generated features

## Common Patterns

### Sync/Async Duality
Every resource has sync and async variants:
- Sync: `Completions` class with methods
- Async: `AsyncCompletions` class with `async def` methods
- Both share the same parameter types and response types

### Pagination
Use `_get_api_list()` for cursor-based pagination:
```python
# Returns SyncCursorPage[T] or AsyncCursorPage[T]
page = client.chat.completions.list(**params)
for item in page.data:
    ...
```

### Streaming
- **Sync**: `Stream[T]` class wraps iterator logic ([_streaming.py](src/openai/_streaming.py))
- **Async**: `AsyncStream[T]` for async iteration
- Example: `client.chat.completions.create(..., stream=True)` returns `Stream[ChatCompletionChunk]`

### Error Handling
Standard exception hierarchy in [_exceptions.py](src/openai/_exceptions.py):
- `OpenAIError` (base)
- `APIError` → `APIStatusError`, `APIConnectionError`, `APITimeoutError`, `APIResponseValidationError`
- Specific errors: `AuthenticationError`, `RateLimitError`, `BadRequestError`, etc.

## Files to Modify for New Features

- **Azure support**: [src/openai/lib/azure.py](src/openai/lib/azure.py) (not overwritten by generator)
- **Parsing/tool helpers**: [src/openai/lib/_tools.py](src/openai/lib/_tools.py) (manually maintained)
- **Examples**: [examples/](examples/) (safe zone, never regenerated)
- **Tests**: [tests/](tests/) may be regenerated; preserve manual tests in subdirectories
- **Streaming helpers**: [src/openai/lib/streaming/](src/openai/lib/streaming/) for custom streaming behavior

## Import Patterns

- **Relative imports configured**: `.vscode/settings.json` sets `python.analysis.importFormat: "relative"`
- **Backward compatibility**: [src/openai/lib/_old_api.py](src/openai/lib/_old_api.py) re-exports deprecated APIs
- **Type hints**: Import from `openai.types.*` (Pydantic models) and `typing_extensions` (for `Literal`, `override`)

## Dependency Notes

- **httpx**: HTTP client (supports sync/async, connection pooling, proxies)
- **pydantic**: Data validation (v1 & v2 compatible)
- **anyio**: Async abstraction layer
- **respx**: HTTP mocking for tests (read-only responses)
- **Rye**: Python dependency manager (auto-provisions Python version from `.python-version`)

## Documentation References

- **API Reference**: [api.md](api.md) (generated from OpenAPI spec)
- **Parsing Guide**: [helpers.md](helpers.md) (Pydantic model + structured outputs patterns)
- **Contributing**: [CONTRIBUTING.md](CONTRIBUTING.md) (detailed setup & release process)
