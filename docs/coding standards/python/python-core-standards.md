# Python Core Standards

[For detailed implementation examples and patterns, see the Implementation Guide]

## Core Philosophy

### Primary Goals
- Create clear, maintainable, and secure Python code
- Keep systems simple enough to "fit in your head"
- Use type hints consistently for safety and clarity
- Design for testing from the start

### Foundational Principles
- **Type Safety**: Type hints are mandatory, not optional
- **Simplicity Over Patterns**: Explicit is better than implicit
- **Test-First Development**: Tests drive design, not validate it
- **Fits in Head Rule**: If you can't reason about a component in one sitting, it's too complex

## Implementation Rules
- NO global state
- NO metaclass magic in production code
- NO empty protocols
- Type hints required for all functions and methods
- All imports must be explicit
- Keep modules focused and small
- Tests required before implementation
- Resource monitoring in place

## Code Organization

### Module Structure
```python
# CORRECT - Explicit imports, type hints
from typing import Optional, List
from datetime import datetime
from .logger import Logger
from .types import User, UserData

# INCORRECT - Star imports, no types
from datetime import *  # Don't do this
```

### Type Safety
```python
# CORRECT - Full type hints
def process_user(user_id: int, name: str) -> Optional[User]:
    if not name:
        return None
    return User(id=user_id, name=name)

# INCORRECT - Missing type hints
def process_user(user_id, name):  # Don't do this
    return User(id=user_id, name=name)

# CORRECT - Type hints for data classes
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class UserProfile:
    id: int
    name: str
    email: str
    tags: List[str] = field(default_factory=list)
    deleted_at: Optional[datetime] = None

# INCORRECT - Dictionary abuse
user = {  # Don't do this
    "id": 1,
    "name": "test",
    "email": "test@example.com"
}
```

### Error Handling
```python
# CORRECT - Custom exceptions with context
class ValidationError(Exception):
    def __init__(self, message: str, field: str):
        self.field = field
        super().__init__(f"{field}: {message}")

# CORRECT - Error handling pattern
def validate_email(email: str) -> bool:
    if not isinstance(email, str):
        raise ValidationError("Invalid type", "email")
    if "@" not in email:
        raise ValidationError("Invalid format", "email")
    return True

# INCORRECT - Bare except
try:
    process_data()
except:  # Don't do this
    pass
```

### Size and Complexity
```python
# CORRECT - Simple, focused function
def get_user_by_email(email: str) -> Optional[User]:
    """Retrieve user by email.
    
    Args:
        email: User's email address
        
    Returns:
        User if found, None otherwise
    """
    return db.query(User).filter_by(email=email).first()

# INCORRECT - "Come to Jesus" function
def process_user_data(data: dict):  # Don't do this
    # 100 lines of validation
    # 50 lines of business logic
    # 75 lines of database operations
```

### Testing First
```python
# CORRECT - Test driven development
from unittest import TestCase

class TestUserService(TestCase):
    def setUp(self) -> None:
        self.service = UserService(
            db=MockDatabase(),
            logger=MockLogger()
        )

    def test_create_user(self) -> None:
        data = {"name": "Test", "email": "test@example.com"}
        user = self.service.create(data)
        self.assertEqual(user.name, "Test")
        self.assertEqual(user.email, "test@example.com")

    def test_invalid_email(self) -> None:
        data = {"name": "Test", "email": "not-an-email"}
        with self.assertRaises(ValidationError) as context:
            self.service.create(data)
        self.assertEqual(context.exception.field, "email")
```

### Resource Management
```python
# CORRECT - Resource monitoring
from contextlib import contextmanager
from typing import Generator
import time
import psutil

@contextmanager
def monitor_resources() -> Generator[None, None, None]:
    start_time = time.time()
    start_memory = psutil.Process().memory_info().rss
    
    try:
        yield
    finally:
        end_memory = psutil.Process().memory_info().rss
        duration = time.time() - start_time
        logger.info("resource_usage", {
            "duration_ms": duration * 1000,
            "memory_delta": end_memory - start_memory
        })
```

## Project Structure
```
project/
  ├── tests/           # Tests FIRST - mirrors src
  │   ├── unit/
  │   └── integration/
  ├── src/
  │   ├── services/    # Business logic
  │   ├── models/      # Data models
  │   └── utils/       # Utilities
  ├── scripts/         # Maintenance scripts
  └── mypy.ini         # Type checking config
```

## Implementation Patterns

### Simple Over Complex
```python
# CORRECT - Simple and direct
class UserService:
    def __init__(self, db: Database, logger: Logger):
        self.db = db
        self.logger = logger
    
    def find_by_email(self, email: str) -> Optional[User]:
        return self.db.query(User).filter_by(email=email).first()

# INCORRECT - Overengineered
class AbstractUserServiceFactoryProvider:  # Don't do this
    def __init__(self, strategy: UserStrategyInterface):
        self.strategy = strategy
```

### Modular Design
```python
# CORRECT - Clear dependencies, easy to test
class OrderProcessor:
    def __init__(
        self,
        payment: PaymentService,
        inventory: InventoryService,
        logger: Logger
    ):
        self.payment = payment
        self.inventory = inventory
        self.logger = logger
```

## Success Metrics
1. All type hints in place
2. MyPy passes with strict mode
3. Tests written BEFORE implementation
4. Test coverage 100% for new code
5. No "Come to Jesus" functions
6. Components "fit in head"
7. Resource monitoring in place
8. No unnecessary abstractions

## Implementation Notes
- Start with tests
- Keep components small and focused
- Type hints are mandatory
- Don't refactor working code without reason
- Monitor resource usage
- Log structured data
- Use dataclasses for data structures
- Run type checker in strict mode