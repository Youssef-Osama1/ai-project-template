# AI Project Template

A production-shaped starting point for LLM applications: a layered, async FastAPI backend
with typed configuration, a single LLM client with timeouts, retries and usage logging,
structured output into Pydantic models, and unit tests that run without network access.

It holds only the parts every LLM app needs. The domain (schemas, prompts, services,
endpoints and evals) is added per project.

It talks to [Groq](https://groq.com) through the OpenAI SDK, so any OpenAI-compatible
provider works by changing `LLM_BASE_URL` and `LLM_MODEL`.

## Requirements

- Python 3.12
- [uv](https://docs.astral.sh/uv/)
- A Groq API key from [console.groq.com](https://console.groq.com/keys)

## Quick start

```bash
git clone <repo-url>
cd ai-project-template
uv sync
```

Create your environment file and add your API key:

```bash
cp .env.example .env
```

Check the key with a raw SDK call:

```bash
uv run --env-file .env playground/first_call.py
```

Start the server:

```bash
uv run uvicorn app.main:app --reload
```

Interactive docs are at http://localhost:8000/docs, and `GET /health` returns
`{"status": "ok"}`.

## Layers

```text
HTTP request
  → app/api/endpoints/      validate the request, map the response
  → app/services/<domain>   which schema, which prompt
  → app/services/llm_client.py   the only module that talks to the LLM SDK
```

Each layer only calls the one below it. Endpoints never import the SDK or a prompt, and the
LLM client knows nothing about the domain.

## Adding a use case

1. **Schema**: a Pydantic model in `app/schemas/`. Field descriptions are sent to the model,
   so they are part of the prompt.
2. **Prompt**: a system prompt constant in `app/prompts/`.
3. **Service**: a class in `app/services/` that takes an `LLMClient` and calls
   `extract(text, YourSchema, system=YOUR_PROMPT)`.
4. **Dependency**: a `get_<name>_service(llm: LLMClientDep)` function and an `Annotated`
   alias in `app/api/dependencies.py`.
5. **API contract**: request and response models in `app/api/schemas/`, with a length limit
   on every input that reaches the model.
6. **Endpoints**: a router in `app/api/endpoints/` that only calls the service, included in
   `app/api/router.py`.
7. **Tests**: mock the layer directly below. A fake `LLMClient` for the service, and
   `app.dependency_overrides` for the endpoints.
8. **Evals**: a labeled JSONL dataset and a runner in `evals/` that uses the same service as
   the API.

`LLMClient`, the settings, logging and `LLMOutputError` handling need no changes.

## Configuration

All settings are read from `.env` by `app/config/settings.py`. Only `LLM_API_KEY` is
required.

| Variable | Default |
| --- | --- |
| `LLM_API_KEY` | required |
| `LLM_BASE_URL` | `https://api.groq.com/openai/v1` |
| `LLM_MODEL` | `openai/gpt-oss-120b` |
| `LLM_TEMPERATURE` | `0.2` |
| `LLM_TIMEOUT_SECONDS` | `30.0` |
| `LLM_MAX_RETRIES` | `2` |
| `LOG_LEVEL` | `INFO` |

The app fails at startup if a required value is missing or a value is invalid. A model
refusal raises `LLMOutputError`, which the API returns as `502`.

## Development

Run the tests:

```bash
uv run pytest
```

Lint and format:

```bash
uv run ruff check --fix .
uv run ruff format .
```

## Project structure

```text
├── app/
│   ├── main.py                 FastAPI app, lifespan, error handler
│   ├── config/settings.py      Settings loaded and validated from .env
│   ├── core/logger.py          Logging setup
│   ├── core/exceptions.py      LLMOutputError
│   ├── schemas/                Contracts with the model
│   ├── prompts/                System prompts
│   ├── services/
│   │   └── llm_client.py       Async LLMClient (complete, extract, close)
│   └── api/
│       ├── router.py           Combines all endpoint routers
│       ├── dependencies.py     Provides the LLM client and services to endpoints
│       ├── endpoints/          One module per group of routes
│       └── schemas/            Request and response models: the contract with API callers
├── evals/                      Labeled datasets and eval runners
├── playground/                 Experiments, never imported by app/
├── tests/                      Unit and API tests, no network
├── docs/                       Project notes and decisions
├── docker/                     Container setup
├── .env.example                Environment variables template
├── pyproject.toml              Dependencies and tool config
└── uv.lock                     Locked dependency versions
```

## Dependency policy

Set in `pyproject.toml` under `[tool.uv]`:

- `add-bounds = "exact"` pins every dependency to an exact version.
- `exclude-newer = "7 days"` skips releases younger than a week, as a guard against
  supply-chain attacks.
- `package = false` treats the project as an application, not an installable package.

In CI and Docker, install with `uv sync --locked --no-dev` so the build fails if
`uv.lock` is out of date with `pyproject.toml`.
