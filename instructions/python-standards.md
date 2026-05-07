# Python Coding Standards for Infrastructure Scripts

This document defines the Python coding standards for infrastructure automation scripts, CLI tools, and utility modules developed by the HSBC Infrastructure team. The team's Copilot agents reference this document during code reviews and generation.

## Language Version and Style

- Target Python 3.10 or later. Do not use features deprecated in 3.10 or later.
- Follow PEP 8 for all code. Use a maximum line length of 100 characters.
- Use 4 spaces for indentation. No tabs.
- Use double quotes for strings by default. Single quotes are acceptable for dictionary keys and short identifiers.
- Imports follow this order, separated by blank lines: standard library, third-party packages, local modules.
- Remove unused imports. Do not use wildcard imports (`from module import *`).

## Type Hints

Type hints are required on all function signatures - both parameters and return types. This includes private functions, class methods, and lambda replacements.

```python
from typing import Any

def load_inventory(path: str, environment: str) -> dict[str, Any]:
    """Load and parse an Ansible inventory file."""
    ...

def validate_host_entries(
    hosts: list[dict[str, str]],
    required_fields: set[str],
) -> list[str]:
    """Validate host entries and return a list of error messages."""
    ...

class InventoryManager:
    def __init__(self, config_path: str, cache_ttl: int = 300) -> None:
        ...

    def get_hosts_by_group(self, group_name: str) -> list[str]:
        ...
```

For complex types, use the built-in generic syntax (Python 3.10+) rather than `typing` module generics where possible:

```python
# Preferred (3.10+)
def process_results(data: list[dict[str, str | int]]) -> dict[str, list[str]]:
    ...

# Acceptable when targeting older runtimes
from typing import Union
def process_results(data: list[dict[str, Union[str, int]]]) -> dict[str, list[str]]:
    ...
```

Use `typing.Protocol` for structural subtyping when you need to define interfaces without inheritance:

```python
from typing import Protocol

class HostProvider(Protocol):
    def get_hosts(self, group: str) -> list[str]: ...
    def get_host_vars(self, hostname: str) -> dict[str, str]: ...
```

## Logging

Use `structlog` for all operational logging. Do not use `print()` for output in library code. CLI tools may use `click.echo()` for user-facing output that is not operational logging.

### Configuration

Configure structlog at the application entry point:

```python
import structlog

def configure_logging(log_level: str = "INFO", json_output: bool = False) -> None:
    """Configure structured logging for the application."""
    processors: list[structlog.types.Processor] = [
        structlog.stdlib.add_log_level,
        structlog.stdlib.add_logger_name,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
    ]

    if json_output:
        processors.append(structlog.processors.JSONRenderer())
    else:
        processors.append(structlog.dev.ConsoleRenderer())

    structlog.configure(
        processors=processors,
        wrapper_class=structlog.stdlib.BoundLogger,
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
    )
```

### Usage

Include contextual key-value pairs in every log call:

```python
import structlog

logger = structlog.get_logger()

def deploy_configuration(hostname: str, config_path: str) -> bool:
    """Deploy configuration to a remote host."""
    logger.info(
        "deploying_configuration",
        hostname=hostname,
        config_path=config_path,
    )

    try:
        result = push_config(hostname, config_path)
        logger.info(
            "configuration_deployed",
            hostname=hostname,
            changed=result.changed,
            duration_ms=result.duration_ms,
        )
        return True
    except ConnectionError as exc:
        logger.error(
            "configuration_deployment_failed",
            hostname=hostname,
            error_type=type(exc).__name__,
            error_message=str(exc),
        )
        raise
```

### Rules

- Never log sensitive data: passwords, tokens, API keys, certificate contents, or personally identifiable information.
- Use present participle for in-progress actions (`deploying_configuration`) and past tense for completed actions (`configuration_deployed`).
- Include the operation name, relevant identifiers, and timing information in log entries.
- Log at `debug` level for function entry/exit and intermediate state.
- Log at `info` level for significant operations and their outcomes.
- Log at `warning` level for unexpected but recoverable situations.
- Log at `error` level for failures that prevent the operation from completing.

## Error Handling

### Specific Exceptions

Catch specific exceptions. Never use bare `except` clauses:

```python
# Correct
try:
    response = requests.get(api_url, timeout=30)
    response.raise_for_status()
except requests.exceptions.ConnectionError as exc:
    logger.error("api_connection_failed", url=api_url, error=str(exc))
    raise
except requests.exceptions.HTTPError as exc:
    logger.error("api_request_failed", url=api_url, status_code=exc.response.status_code)
    raise

# Incorrect
try:
    response = requests.get(api_url)
except Exception:
    print("Something went wrong")
```

### Custom Exceptions

Define custom exception classes for domain-specific errors. Place them in a dedicated `exceptions.py` module:

```python
class InfraError(Exception):
    """Base exception for infrastructure automation errors."""

class InventoryError(InfraError):
    """Raised when inventory operations fail."""

class VaultDecryptionError(InfraError):
    """Raised when vault decryption fails."""

class HostUnreachableError(InfraError):
    """Raised when a target host cannot be reached."""
    def __init__(self, hostname: str, reason: str) -> None:
        self.hostname = hostname
        self.reason = reason
        super().__init__(f"Host {hostname} is unreachable: {reason}")
```

### Context Managers

Use context managers for resource management. Implement `__enter__` and `__exit__` or use `contextlib.contextmanager` for cleanup patterns:

```python
from contextlib import contextmanager
from typing import Generator

@contextmanager
def temporary_vault_decryption(vault_path: str, password: str) -> Generator[str, None, None]:
    """Decrypt a vault file temporarily and clean up after use."""
    decrypted_path = decrypt_vault(vault_path, password)
    try:
        yield decrypted_path
    finally:
        secure_delete(decrypted_path)
```

## CLI Tools

### Framework

Use `click` for CLI argument parsing. Fall back to `argparse` only when click is not available in the target environment.

### Structure

```python
import click
import structlog

logger = structlog.get_logger()

@click.group()
@click.option("--verbose", "-v", is_flag=True, help="Enable verbose output.")
@click.option("--config", "-c", type=click.Path(exists=True), help="Path to configuration file.")
@click.pass_context
def cli(ctx: click.Context, verbose: bool, config: str | None) -> None:
    """Infrastructure automation utilities for the HSBC Infrastructure team."""
    ctx.ensure_object(dict)
    log_level = "DEBUG" if verbose else "INFO"
    configure_logging(log_level=log_level)
    ctx.obj["config"] = load_config(config) if config else default_config()

@cli.command()
@click.argument("inventory_path", type=click.Path(exists=True))
@click.option("--environment", "-e", required=True, help="Target environment name.")
@click.pass_context
def validate(ctx: click.Context, inventory_path: str, environment: str) -> None:
    """Validate an Ansible inventory file against team standards."""
    logger.info("validating_inventory", path=inventory_path, environment=environment)
    errors = run_validation(inventory_path, environment, ctx.obj["config"])
    if errors:
        for error in errors:
            click.echo(f"  {error}", err=True)
        raise SystemExit(1)
    click.echo("Inventory validation passed.")

if __name__ == "__main__":
    cli()
```

### Requirements

- Every command and option must have a `help` string.
- Use `click.Path(exists=True)` for file path arguments that must exist.
- Use `click.Choice()` for arguments with a fixed set of valid values.
- Use `click.pass_context` to share configuration between commands.
- Exit with code 0 on success and non-zero on failure.
- Write user-facing messages to stderr for errors and stdout for normal output.

## Configuration

### Sources

Configuration should be loaded from these sources in order of increasing precedence:

1. Default values defined in code
2. Configuration file (YAML format, path specified via CLI option or environment variable)
3. Environment variables

```python
import os
from pathlib import Path
from typing import Any

import yaml


def load_config(config_path: str | None = None) -> dict[str, Any]:
    """Load configuration from file and environment variables."""
    config: dict[str, Any] = {
        "api_url": "https://cmdb.example.internal/api/v1",
        "api_timeout": 30,
        "cache_ttl": 300,
        "log_level": "INFO",
    }

    if config_path:
        file_path = Path(config_path)
        if file_path.exists():
            with file_path.open() as f:
                file_config = yaml.safe_load(f)
                if file_config:
                    config.update(file_config)

    env_overrides = {
        "INFRA_API_URL": "api_url",
        "INFRA_API_TIMEOUT": "api_timeout",
        "INFRA_CACHE_TTL": "cache_ttl",
        "INFRA_LOG_LEVEL": "log_level",
    }
    for env_var, config_key in env_overrides.items():
        value = os.environ.get(env_var)
        if value is not None:
            config[config_key] = value

    return config
```

### Rules

- Never hardcode environment-specific values (hostnames, ports, URLs, credentials).
- Use `yaml.safe_load()` for parsing YAML configuration. Never use `yaml.load()` without a safe loader.
- Validate configuration at startup. Fail early with a clear message if required values are missing.
- Document all configuration options in the project README or a dedicated configuration reference.

## Unit Tests

### Framework

Use `pytest` as the test framework. Use `pytest-mock` for mocking external dependencies.

### Requirements

- Every module with business logic must have a corresponding test module: `src/inventory.py` should have `tests/test_inventory.py`.
- Test functions use the naming convention `test_<function_name>_<scenario>`:

```python
def test_load_inventory_returns_hosts_for_valid_file() -> None:
    ...

def test_load_inventory_raises_error_for_missing_file() -> None:
    ...

def test_load_inventory_handles_empty_host_group() -> None:
    ...
```

- Use `pytest.raises` for testing exception behavior:

```python
import pytest

def test_validate_host_raises_for_missing_hostname() -> None:
    with pytest.raises(InventoryError, match="hostname is required"):
        validate_host({"ip": "10.0.0.1"})
```

- Use fixtures for shared test setup:

```python
import pytest
from pathlib import Path

@pytest.fixture
def sample_inventory_path(tmp_path: Path) -> Path:
    inventory_file = tmp_path / "hosts.yml"
    inventory_file.write_text(
        "all:\n  hosts:\n    app-01:\n      ansible_host: 10.0.0.1\n"
    )
    return inventory_file
```

- Mock external dependencies (API calls, file system operations, SSH connections) in unit tests. Use real resources only in integration tests.
- Tests must not depend on execution order. Each test must set up its own state.
- Tests must not depend on network access or external services.

### Coverage

Aim for meaningful test coverage that validates business logic. The team requires a minimum of 80% line coverage as enforced by `pytest --cov` with `fail_under = 80`. Focus test effort on:
- Input validation and edge cases
- Error handling paths
- Business logic with conditional branches
- Data transformation functions

## File Headers

Every Python file must begin with a module docstring:

```python
"""
Ansible inventory management utilities.

Provides functions for loading, validating, and transforming Ansible
inventory files in YAML format. Used by the inventory validation CLI
tool and the dynamic inventory generator.

Author: HSBC Infrastructure Team
"""
```

Do not include license headers, encoding declarations (`# -*- coding: utf-8 -*-`), or shebang lines unless the file is intended to be executed directly.

## Dependencies

- Pin all dependencies in `requirements.txt` or `pyproject.toml` with exact versions for reproducible builds.
- Separate runtime dependencies from development dependencies.
- Use `uv` for dependency installation and virtual environment management. Do not use `pip` directly.
- Document why each dependency is needed with a comment in the requirements file if the purpose is not obvious from the package name.

```
# requirements.txt
click==8.1.7                  # CLI framework
structlog==24.1.0             # Structured logging
pyyaml==6.0.1                 # YAML parsing for configuration and inventory
requests==2.31.0              # HTTP client for API interactions
```
