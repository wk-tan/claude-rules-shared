# Development: Python Coding Standards

**When to use this file:** Reference this when writing Python code, including style guidelines, docstring formats, type hints, and logging patterns.

**Related documentation:**
- For tool configuration (ruff, pyright), see workspace root `pyproject.toml`
- For Polylith structure, see [01-setup.md](01-setup.md)
- For infrastructure/application separation and architecture patterns, see [01-setup.md](01-setup.md#1-infrastructure-and-application-separation)

---

## Table of Contents

1. [General Principles](#1-general-principles)
2. [Code Style and Formatting](#2-code-style-and-formatting)
3. [Naming Conventions](#3-naming-conventions)
4. [Import Organization](#4-import-organization)
5. [Type Hints](#5-type-hints)
6. [Docstring Standards](#6-docstring-standards)
7. [Logging Standards](#7-logging-standards)
8. [Complete Code Examples](#8-complete-code-examples)

---

## 1. General Principles

### Core Standards

- **Style Guide:** All Python code MUST adhere to [PEP 8](https://peps.python.org/pep-0008/)
- **Automation:** Use `ruff` formatter for consistent styling
- **Clarity:** Write code to be clear and readable
- **Type Safety:** All functions, methods, and class attributes MUST include type annotations
- **Documentation:** All public APIs MUST have Google-style docstrings

### Code Quality Tools

**ruff** - Linter and formatter
- Configured in workspace root `pyproject.toml`
- Line length: 88 characters
- Target: Python 3.13
- Auto-fixes many issues

**pyright** - Type checker
- Configured in workspace root `pyproject.toml`
- Type checking mode: standard
- Verifies all type hints

**pre-commit** - Git hooks
- Runs ruff and pyright before commits
- Ensures code quality before it reaches CI/CD

---

## 2. Code Style and Formatting

### Indentation

- **Use 4 spaces** per indentation level
- **Never use tabs**
- Configure editor to insert spaces

```python
# ✅ Correct
def calculate_total(items: list[int]) -> int:
    total = 0
    for item in items:
        total += item
    return total

# ❌ Wrong - uses tabs or inconsistent spacing
```

### Line Length

- **Maximum 88 characters** per line
- Wrap to multiple lines if needed
- Break after opening parenthesis/bracket or before binary operator

```python
# ✅ Correct - Breaking after opening parenthesis
response = requests.get(
    f"{API_ENDPOINT}/{item_id}",
    headers={"Authorization": f"Bearer {self.api_key}"},
    timeout=10,
)

# ✅ Correct - Breaking before operator
total_cost = (
    base_price
    + tax_amount
    + shipping_cost
    - discount
)
```

### Whitespace

**Around operators:**
- Single space around most operators (`=`, `+=`, `==`, `<`, `>`)

```python
# ✅ Correct
x = 5
y = x + 10
is_valid = y > 15

# ❌ Wrong
x=5
y=x+10
is_valid=y>15
```

**After commas:**
- Single space after commas in lists, dictionaries, function arguments

```python
# ✅ Correct
items = [1, 2, 3, 4]
config = {"host": "localhost", "port": 8080}

# ❌ Wrong
items = [1,2,3,4]
config = {"host":"localhost","port":8080}
```

**Inside parentheses/brackets:**
- Avoid extraneous whitespace immediately inside

```python
# ✅ Correct
result = function(arg1, arg2)
items = [1, 2, 3]

# ❌ Wrong
result = function( arg1, arg2 )
items = [ 1, 2, 3 ]
```

### Blank Lines

- **Two blank lines** to separate top-level functions and class definitions
- **One blank line** to separate method definitions inside a class

```python
import os


def first_function():
    pass


def second_function():
    pass


class MyClass:
    def method_one(self):
        pass

    def method_two(self):
        pass
```

---

## 3. Naming Conventions

### Naming Styles

| Element | Style | Example |
|---------|-------|---------|
| **Modules** | `lowercase_snake_case` | `api_client.py`, `data_processor.py` |
| **Functions** | `lowercase_snake_case` | `calculate_total()`, `fetch_data()` |
| **Variables** | `lowercase_snake_case` | `user_count`, `api_key` |
| **Classes** | `CamelCase` | `DataProcessor`, `UserManager` |
| **Constants** | `UPPER_SNAKE_CASE` | `MAX_RETRIES`, `API_ENDPOINT` |
| **Private** | `_leading_underscore` | `_internal_helper()`, `_cache` |

### Examples

```python
# Module: user_manager.py

MAX_LOGIN_ATTEMPTS = 3
DEFAULT_TIMEOUT = 30


class UserManager:
    """Manage user authentication and sessions."""

    def __init__(self, api_key: str):
        self.api_key = api_key
        self._session_cache = {}

    def authenticate_user(self, username: str, password: str) -> bool:
        """Authenticate user with credentials."""
        return self._verify_credentials(username, password)

    def _verify_credentials(self, username: str, password: str) -> bool:
        """Internal helper for credential verification."""
        # Implementation
        pass
```

---

## 4. Import Organization

### Import Order

Imports MUST be grouped in three sections, with blank lines separating each:

1. **Standard library imports**
2. **Third-party library imports**
3. **Local application/library imports**

Within each group, sort alphabetically.

### Import Style

- **Prefer absolute imports** over relative imports
- One import per line (unless importing multiple items from same module)

```python
# ✅ Correct
import os
import sys
from collections import namedtuple
from typing import Any

import requests
from google.cloud import pubsub_v1

from my_workspace.components.logging.core import log_event
from my_workspace.components.settings.core import Settings

# ❌ Wrong - mixed groups, no separation
import requests
import os
from my_workspace.components.logging.core import log_event
import sys
```

### Tool-Specific Imports

```python
# Standard library
import logging
import traceback
from datetime import datetime
from typing import Any, Optional

# Third-party
import requests
from pydantic import BaseModel
from pydantic_settings import BaseSettings

# Local
from pipeline.components.gcp.pubsub.core import publish_message
from pipeline.components.logging.core import log_event
```

---

## 5. Type Hints

### Requirements

- **All functions and methods** MUST have type hints for parameters and return values
- **All class attributes** MUST have type hints
- Use modern Python 3.13 syntax (e.g., `list[str]` not `List[str]`)

### Basic Type Hints

```python
from typing import Any

# Function with type hints
def calculate_total(prices: list[float], tax_rate: float = 0.1) -> float:
    """Calculate total with tax."""
    subtotal = sum(prices)
    return subtotal * (1 + tax_rate)

# Multiple return types
def fetch_user(user_id: int) -> dict[str, Any] | None:
    """Fetch user data or None if not found."""
    # Implementation
    pass

# Class with type hints
class User:
    """User data model."""

    name: str
    age: int
    email: str | None

    def __init__(self, name: str, age: int, email: str | None = None):
        self.name = name
        self.age = age
        self.email = email
```

### Advanced Type Hints

```python
from collections.abc import Callable, Iterator
from typing import Any, TypeVar

T = TypeVar("T")


def process_items(
    items: list[T],
    processor: Callable[[T], T],
) -> list[T]:
    """Process each item using provided function."""
    return [processor(item) for item in items]


def generate_numbers(start: int, end: int) -> Iterator[int]:
    """Generate numbers in range."""
    for i in range(start, end):
        yield i
```

---

## 6. Docstring Standards

### Format Requirements

- **Style:** Google Python Style Guide
- **Quotes:** Always use triple double quotes (`"""`)
- **Line length:** Maximum 88 characters per line
- **Structure:** Summary line, blank line, detailed description, sections

### Docstring Sections

1. **Summary** (required) - One-line description in imperative mood
2. **Extended description** (optional) - Detailed explanation
3. **Note** (optional) - Important considerations, side effects
4. **Args** (required if parameters exist) - Parameter descriptions
5. **Returns** (required if returns value) - Return value description
6. **Yields** (required if generator) - Yielded value description
7. **Raises** (required if raises exceptions) - Exception descriptions

### Function Docstrings

```python
def fetch_and_process_data(
    endpoint: str,
    timeout: int = 10,
    retries: int = 3
) -> list[dict[str, Any]]:
    """Fetch and process data from API endpoint.

    Send GET request to specified endpoint, parse JSON response,
    and return processed data items.

    Note:
    - This function loads all data into memory
    - Retries automatically on network errors

    Args:
        endpoint: The API endpoint URL to fetch data from
        timeout: Request timeout in seconds. Defaults to 10.
        retries: Number of retry attempts on failure. Defaults to 3.

    Returns:
        list[dict[str, Any]]: Processed data items from the API

    Raises:
        requests.exceptions.RequestException: For connection errors
        ValueError: If response is not valid JSON
    """
    # Implementation
    pass
```

### Class Docstrings

```python
class DataProcessor:
    """Process and transform data from multiple sources.

    This class handles fetching, transforming, and loading data
    from various API endpoints into a standardized format.

    Attributes:
        api_key: Authentication key for API access
        timeout: Default request timeout in seconds
        retry_count: Number of automatic retries on failure
    """

    def __init__(self, api_key: str, timeout: int = 30):
        """Initialize the DataProcessor.

        Args:
            api_key: Authentication key for API access
            timeout: Request timeout in seconds. Defaults to 30.
        """
        self.api_key = api_key
        self.timeout = timeout
        self.retry_count = 3
```

### Generator Docstrings

```python
def read_large_file(file_path: str) -> Iterator[str]:
    """Read large file line by line.

    Efficiently process large files by yielding one line at a time
    instead of loading entire file into memory.

    Args:
        file_path: Path to the file to read

    Yields:
        str: Each line from the file, with trailing newline removed

    Raises:
        FileNotFoundError: If file doesn't exist
        PermissionError: If file can't be read
    """
    with open(file_path) as f:
        for line in f:
            yield line.rstrip("\n")
```

---

## 7. Logging Standards

### Structured Logging Format

All logs MUST use structured dictionary format with the `msg` parameter.

### Key Fields

- **event** (required) - Past-tense verb + object describing what happened
- **payload** (optional) - Contextual data relevant to the event
- **traceback** (error logs only) - Exception traceback for debugging

### Event Naming Convention

- **Start with verb in past tense** followed by object
- ✅ Correct: `"Fetched user data"`, `"Published message"`, `"Started pipeline"`
- ❌ Wrong: `"User data fetched"`, `"Message published"`, `"Pipeline started"`

### Info Logs

Use for successful or expected operations:

```python
import logging

logging.info(
    msg={
        "event": "Fetched user profile",
        "payload": {
            "user_id": user_id,
            "profile_data": profile_data,
        },
    }
)

logging.info(
    msg={
        "event": "Published message to Pub/Sub",
        "payload": {
            "topic": topic_name,
            "message_id": message_id,
        },
    }
)
```

### Error Logs

Use for errors, exceptions, or unexpected failures. MUST include traceback:

```python
import logging
import traceback

try:
    result = risky_operation()
except Exception as e:
    logging.error(
        msg={
            "event": str(e),
            "payload": {
                "traceback": traceback.format_exc(),
                "operation": "risky_operation",
            },
        }
    )
```

### Logging Best Practices

**DO:**
- ✅ Use structured dictionary format
- ✅ Start event with past-tense verb
- ✅ Include relevant context in payload
- ✅ Include traceback in error logs
- ✅ Keep payloads concise

**DON'T:**
- ❌ Log plain strings: `logging.info("User logged in")`
- ❌ Use present tense: `"Fetching user data"`
- ❌ Include sensitive data (passwords, tokens, PII)
- ❌ Include full request/response bodies (unless necessary)
- ❌ Concatenate messages: `logging.info(f"User {user_id} logged in")`

---

## 8. Complete Code Examples

### Example 1: Data Fetcher with Full Standards

```python
"""Data fetching module for external API integration."""

import logging
import sys
import traceback
from typing import Any

import requests

from my_workspace.components.settings.core import Settings

# Constants
MAX_RETRIES = 3
DEFAULT_TIMEOUT = 30
API_ENDPOINT = "https://api.example.com/data"


class DataFetcher:
    """Fetch and process data from external API.

    This class handles authentication, request retries, and error handling
    for API data fetching operations.

    Attributes:
        api_key: Authentication key for API access
        timeout: Request timeout in seconds
        settings: Application settings instance
    """

    def __init__(self, api_key: str, timeout: int = DEFAULT_TIMEOUT):
        """Initialize the DataFetcher.

        Args:
            api_key: Authentication key for API access
            timeout: Request timeout in seconds. Defaults to 30.
        """
        self.api_key = api_key
        self.timeout = timeout
        self.settings = Settings()

    def fetch_item(self, item_id: int) -> dict[str, Any] | None:
        """Fetch specific item data from API.

        Send GET request to API endpoint with authentication, parse JSON
        response, and return the data.

        Note:
        - This function performs a network request and may raise exceptions
        - Automatically retries on connection errors

        Args:
            item_id: Unique identifier of the item to fetch

        Returns:
            dict[str, Any] | None: Item data as dictionary, or None if error

        Raises:
            requests.exceptions.RequestException: For network errors
        """
        try:
            response = requests.get(
                f"{API_ENDPOINT}/{item_id}",
                headers={"Authorization": f"Bearer {self.api_key}"},
                timeout=self.timeout,
            )
            response.raise_for_status()
            data = response.json()

            logging.info(
                msg={
                    "event": "Fetched item data",
                    "payload": {
                        "item_id": item_id,
                        "status_code": response.status_code,
                    },
                }
            )

            return data

        except requests.exceptions.RequestException as e:
            logging.error(
                msg={
                    "event": str(e),
                    "payload": {
                        "traceback": traceback.format_exc(),
                        "item_id": item_id,
                        "endpoint": API_ENDPOINT,
                    },
                }
            )
            return None

    def fetch_multiple(self, item_ids: list[int]) -> list[dict[str, Any]]:
        """Fetch multiple items from API.

        Args:
            item_ids: List of item IDs to fetch

        Returns:
            list[dict[str, Any]]: List of successfully fetched items
        """
        results = []
        for item_id in item_ids:
            item = self.fetch_item(item_id)
            if item is not None:
                results.append(item)

        logging.info(
            msg={
                "event": "Fetched multiple items",
                "payload": {
                    "requested": len(item_ids),
                    "successful": len(results),
                },
            }
        )

        return results


def calculate_metrics(data: list[dict[str, Any]]) -> dict[str, float]:
    """Calculate aggregate metrics from data items.

    Args:
        data: List of data items with numeric values

    Returns:
        dict[str, float]: Calculated metrics (total, average, max, min)

    Raises:
        ValueError: If data list is empty
    """
    if not data:
        raise ValueError("Cannot calculate metrics from empty data list")

    values = [item.get("value", 0) for item in data]

    return {
        "total": sum(values),
        "average": sum(values) / len(values),
        "max": max(values),
        "min": min(values),
    }
```

### Example 2: Settings Component

```python
"""Application settings using Pydantic."""

from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """Application configuration settings.

    Load settings from environment variables with validation.

    Attributes:
        api_key: Authentication key for external API
        api_endpoint: Base URL for API requests
        timeout: Default request timeout in seconds
        max_retries: Maximum retry attempts for failed requests
        log_level: Logging level (DEBUG, INFO, WARNING, ERROR)
    """

    model_config = SettingsConfigDict(
        env_prefix="APP_",
        env_file=".env",
        env_file_encoding="utf-8",
    )

    api_key: str = Field(..., description="API authentication key")
    api_endpoint: str = Field(
        default="https://api.example.com",
        description="Base API endpoint URL",
    )
    timeout: int = Field(
        default=30,
        ge=1,
        le=300,
        description="Request timeout in seconds",
    )
    max_retries: int = Field(
        default=3,
        ge=0,
        le=10,
        description="Maximum retry attempts",
    )
    log_level: str = Field(
        default="INFO",
        description="Logging level",
    )
```

### Example 3: Simple Utility Function

```python
"""Utility functions for data processing."""

import os


def get_system_load(default_value: float = 0.0) -> float:
    """Get current system load average over 1 minute.

    Retrieve 1-minute load average of the system. If the operating
    system doesn't support getloadavg, return default value.

    Args:
        default_value: Value to return if getloadavg not available.
            Defaults to 0.0.

    Returns:
        float: 1-minute system load average, or default_value if unsupported
    """
    if hasattr(os, "getloadavg"):
        return os.getloadavg()[0]
    return default_value
```

---

## Quick Reference

### Code Style Checklist

- [ ] 4 spaces for indentation (no tabs)
- [ ] Max 88 characters per line
- [ ] Proper spacing around operators and after commas
- [ ] Two blank lines between top-level definitions
- [ ] One blank line between class methods

### Naming Checklist

- [ ] Modules: `lowercase_snake_case.py`
- [ ] Functions: `lowercase_snake_case()`
- [ ] Variables: `lowercase_snake_case`
- [ ] Classes: `CamelCase`
- [ ] Constants: `UPPER_SNAKE_CASE`

### Import Checklist

- [ ] Standard library imports first
- [ ] Third-party imports second
- [ ] Local imports last
- [ ] Blank lines between groups
- [ ] Sorted alphabetically within groups

### Docstring Checklist

- [ ] Triple double quotes (`"""`)
- [ ] Imperative mood summary
- [ ] Args section with types and descriptions
- [ ] Returns section with type
- [ ] Raises section if applicable
- [ ] Note section for important details

### Logging Checklist

- [ ] Structured dictionary format
- [ ] Past-tense verb in event
- [ ] Relevant context in payload
- [ ] Traceback included in error logs
- [ ] No sensitive data in logs
