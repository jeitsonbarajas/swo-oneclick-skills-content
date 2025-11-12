# Python GitHub Copilot Instructions

**Generated:** 2025-11-09  
**Version:** 1.0  
**Technology:** Python  
**Author:** SWO Team

## Overview
These instructions configure GitHub Copilot to generate Python code following PEP 8, PEP 257, and modern Python best practices (3.10+).

## Python Best Practices

### Code Style (PEP 8)

```python
# Good naming conventions
class UserRepository:
    """Repository for user data operations."""
    
    MAX_RETRIES = 3
    DEFAULT_TIMEOUT = 30
    
    def __init__(self, database_url: str) -> None:
        self.database_url = database_url
        self._connection = None
    
    def get_user_by_id(self, user_id: str) -> dict | None:
        """
        Retrieve a user by their unique identifier.
        
        Args:
            user_id: The unique identifier of the user
            
        Returns:
            User data dictionary or None if not found
            
        Raises:
            DatabaseError: If database connection fails
        """
        pass

# Constants in UPPER_CASE
API_BASE_URL = "https://api.example.com"
MAX_CONNECTIONS = 100

# Functions and variables in snake_case
def calculate_total_price(items: list[dict]) -> float:
    total_price = sum(item['price'] for item in items)
    return total_price

# Classes in PascalCase
class PaymentProcessor:
    pass
```

### Type Hints (PEP 484)

```python
from typing import Optional, Union, List, Dict, Any, Callable, TypeVar, Generic
from collections.abc import Sequence, Mapping

# Basic type hints
def greet(name: str) -> str:
    return f"Hello, {name}!"

# Optional types (Python 3.10+)
def find_user(user_id: str) -> dict | None:
    return database.get(user_id)

# Union types
def process_input(value: int | str | float) -> str:
    return str(value)

# Complex types
def aggregate_data(
    items: Sequence[dict],
    filters: Mapping[str, Any]
) -> list[dict]:
    return [item for item in items if matches_filters(item, filters)]

# Generic types
T = TypeVar('T')

class DataStore(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []
    
    def add(self, item: T) -> None:
        self._items.append(item)
    
    def get_all(self) -> list[T]:
        return self._items.copy()

# Callable types
def retry_operation(
    func: Callable[[str], dict],
    arg: str,
    max_retries: int = 3
) -> dict:
    for attempt in range(max_retries):
        try:
            return func(arg)
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            continue
```

### Dataclasses and Pydantic

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional

# Simple dataclass
@dataclass
class User:
    id: str
    name: str
    email: str
    created_at: datetime = field(default_factory=datetime.now)
    is_active: bool = True
    
    def __post_init__(self) -> None:
        """Validate after initialization."""
        if not self.email or '@' not in self.email:
            raise ValueError(f"Invalid email: {self.email}")

# Frozen (immutable) dataclass
@dataclass(frozen=True)
class Config:
    api_key: str
    base_url: str
    timeout: int = 30

# Pydantic for validation
from pydantic import BaseModel, EmailStr, validator

class UserCreate(BaseModel):
    name: str
    email: EmailStr
    age: int
    
    @validator('age')
    def validate_age(cls, v: int) -> int:
        if v < 18:
            raise ValueError('Must be 18 or older')
        return v
    
    class Config:
        str_strip_whitespace = True
```

### Async/Await

```python
import asyncio
from typing import Any

# Async function
async def fetch_user_data(user_id: str) -> dict:
    """Fetch user data asynchronously."""
    async with aiohttp.ClientSession() as session:
        async with session.get(f'/api/users/{user_id}') as response:
            response.raise_for_status()
            return await response.json()

# Async context manager
class AsyncDatabaseConnection:
    async def __aenter__(self) -> 'AsyncDatabaseConnection':
        await self.connect()
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb) -> None:
        await self.close()
    
    async def connect(self) -> None:
        """Establish database connection."""
        pass
    
    async def close(self) -> None:
        """Close database connection."""
        pass

# Usage
async def process_users(user_ids: list[str]) -> list[dict]:
    """Process multiple users concurrently."""
    tasks = [fetch_user_data(user_id) for user_id in user_ids]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    # Filter out exceptions
    return [r for r in results if isinstance(r, dict)]

# Run async code
if __name__ == "__main__":
    user_ids = ["1", "2", "3"]
    users = asyncio.run(process_users(user_ids))
```

### Error Handling

```python
from typing import NoReturn

# Custom exceptions
class ValidationError(Exception):
    """Raised when validation fails."""
    
    def __init__(self, message: str, field: str) -> None:
        self.message = message
        self.field = field
        super().__init__(f"{field}: {message}")

class NotFoundError(Exception):
    """Raised when resource is not found."""
    pass

# Proper exception handling
def get_user(user_id: str) -> dict:
    """
    Get user by ID.
    
    Raises:
        ValidationError: If user_id is invalid
        NotFoundError: If user doesn't exist
    """
    if not user_id or not user_id.strip():
        raise ValidationError("User ID cannot be empty", "user_id")
    
    user = database.find_by_id(user_id)
    if user is None:
        raise NotFoundError(f"User {user_id} not found")
    
    return user

# Context manager for resources
from contextlib import contextmanager

@contextmanager
def database_connection(url: str):
    """Context manager for database connections."""
    conn = None
    try:
        conn = connect_to_database(url)
        yield conn
    except Exception as e:
        if conn:
            conn.rollback()
        raise
    finally:
        if conn:
            conn.close()

# Usage
with database_connection(DB_URL) as conn:
    cursor = conn.execute("SELECT * FROM users")
    users = cursor.fetchall()
```

### Decorators

```python
from functools import wraps
from time import time
from typing import Callable, Any

# Simple decorator
def timer(func: Callable) -> Callable:
    """Time function execution."""
    @wraps(func)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        start = time()
        result = func(*args, **kwargs)
        end = time()
        print(f"{func.__name__} took {end - start:.4f}s")
        return result
    return wrapper

# Decorator with arguments
def retry(max_attempts: int = 3, delay: float = 1.0):
    """Retry decorator with configurable attempts."""
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args: Any, **kwargs: Any) -> Any:
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    time.sleep(delay)
            return None
        return wrapper
    return decorator

# Class method decorators
class Cache:
    def __init__(self):
        self._cache: dict[str, Any] = {}
    
    @property
    def size(self) -> int:
        """Get cache size."""
        return len(self._cache)
    
    @staticmethod
    def validate_key(key: str) -> bool:
        """Validate cache key format."""
        return bool(key and isinstance(key, str))
    
    @classmethod
    def create_default(cls) -> 'Cache':
        """Create cache with default settings."""
        return cls()

# Usage
@timer
@retry(max_attempts=3, delay=0.5)
def fetch_data(url: str) -> dict:
    """Fetch data with retry and timing."""
    pass
```

### List Comprehensions & Generators

```python
# List comprehension
squared_evens = [x**2 for x in range(10) if x % 2 == 0]

# Dict comprehension
user_map = {user['id']: user for user in users}

# Set comprehension
unique_tags = {tag.lower() for item in items for tag in item['tags']}

# Generator expression (memory efficient)
total = sum(x**2 for x in range(1000000))

# Generator function
def read_large_file(file_path: str):
    """Read file line by line (memory efficient)."""
    with open(file_path, 'r') as f:
        for line in f:
            yield line.strip()

# Usage
for line in read_large_file('large.txt'):
    process(line)
```

## Project Structure

```
project/
├── src/
│   ├── __init__.py
│   ├── main.py          # Entry point
│   ├── models/          # Data models
│   │   ├── __init__.py
│   │   └── user.py
│   ├── services/        # Business logic
│   │   ├── __init__.py
│   │   └── user_service.py
│   ├── repositories/    # Data access
│   │   ├── __init__.py
│   │   └── user_repository.py
│   └── utils/           # Utilities
│       ├── __init__.py
│       └── helpers.py
├── tests/
│   ├── __init__.py
│   ├── test_services.py
│   └── test_repositories.py
├── requirements.txt
├── pyproject.toml
└── README.md
```

## Testing

```python
import pytest
from unittest.mock import Mock, patch

# Simple test
def test_calculate_total():
    items = [{'price': 10}, {'price': 20}]
    assert calculate_total_price(items) == 30

# Parametrized test
@pytest.mark.parametrize("input,expected", [
    ("hello", "HELLO"),
    ("world", "WORLD"),
    ("", ""),
])
def test_uppercase(input: str, expected: str):
    assert input.upper() == expected

# Fixture
@pytest.fixture
def sample_user():
    return User(
        id="123",
        name="John Doe",
        email="john@example.com"
    )

def test_user_creation(sample_user):
    assert sample_user.name == "John Doe"
    assert sample_user.is_active is True

# Mocking
@patch('requests.get')
def test_fetch_data(mock_get):
    mock_response = Mock()
    mock_response.json.return_value = {'id': '1', 'name': 'Test'}
    mock_get.return_value = mock_response
    
    result = fetch_user_data('1')
    assert result['name'] == 'Test'
    mock_get.assert_called_once()

# Async test
@pytest.mark.asyncio
async def test_async_fetch():
    result = await fetch_user_data('123')
    assert 'id' in result
```

## Documentation

```python
def calculate_statistics(
    data: Sequence[float],
    exclude_outliers: bool = False
) -> dict[str, float]:
    """
    Calculate statistical metrics for a dataset.
    
    This function computes mean, median, and standard deviation.
    Optionally removes outliers before calculation.
    
    Args:
        data: Sequence of numeric values to analyze
        exclude_outliers: Whether to exclude outliers (> 2 std devs)
        
    Returns:
        Dictionary containing:
            - mean: Arithmetic mean
            - median: Middle value
            - std_dev: Standard deviation
            
    Raises:
        ValueError: If data is empty
        
    Examples:
        >>> calculate_statistics([1, 2, 3, 4, 5])
        {'mean': 3.0, 'median': 3.0, 'std_dev': 1.41}
        
        >>> calculate_statistics([1, 100], exclude_outliers=True)
        {'mean': 50.5, 'median': 50.5, 'std_dev': 49.5}
    """
    pass
```

## Performance Tips

1. Use generators for large datasets
2. Prefer `''.join()` over `+=` for strings
3. Use `sets` for membership testing
4. Profile with `cProfile` before optimizing
5. Consider `asyncio` for I/O-bound tasks
6. Use `__slots__` for memory optimization in classes

---

*This configuration is part of SWO OneClick Skills - Created by SoftwareOne Team*
