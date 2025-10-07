# Understanding Pydantic `BaseSettings` for FastAPI Config

When you inherit from `pydantic.BaseSettings`, you gain a typed configuration object that loads values straight from environment variables (and optional `.env` files) while validating them.

## Why Use `BaseSettings`?

- **Centralized config**: define every required secret or toggle in one place.
- **Type safety**: data is parsed into Python types (`str`, `int`, etc.) and validated.
- **Multiple sources**: environment variables, `.env`, and defaults all work together.

## Example

```python
from pydantic import BaseSettings


class Settings(BaseSettings):
    slack_bot_token: str
    ai_provider: str = "openai"
    openai_api_key: str | None = None
    anthropic_api_key: str | None = None

    model_config = {
        "env_file": ".env",
        "env_prefix": "",
        "case_sensitive": False,
    }
```

### Field Mapping

- `slack_bot_token` → expects `SLACK_BOT_TOKEN` in the environment.
- `ai_provider` defaults to `"openai"` unless `AI_PROVIDER` is set.
- `openai_api_key` / `anthropic_api_key` are optional (`None` if missing).

### Configuration Options

- `env_file": ".env"` loads variables from a local `.env` file before looking at the real environment.
- `env_prefix": ""` means names are used as-is (`SLACK_BOT_TOKEN`).
- `case_sensitive": False` allows `slack_bot_token` or `SLACK_BOT_TOKEN`; uppercase is still recommended.

## Using the Settings

```python
settings = Settings()
print(settings.slack_bot_token)
```

If a required field like `SLACK_BOT_TOKEN` is missing, Pydantic raises a validation error immediately. That fails fast during app startup and ensures secrets are in place before any endpoints run.

## Takeaways

1. Declare config once; use strongly-typed fields everywhere.
2. Document required environment variables via `.env.example` and your `Settings` class.
3. On startup, instantiate `Settings()` so misconfiguration is caught early.
