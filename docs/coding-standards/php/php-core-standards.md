# PHP Core Standards

[For detailed implementation examples and patterns, see the Pythonic Implementation Guide]

## Core Philosophy

### Primary Goals
- Make PHP behave and feel as Pythonic as possible for cross-language project compatibility
- Create maintainable, secure, and testable code
- Keep systems simple enough to "fit in your head"

### Foundational Principles
- **Type Safety is Security**: Strict typing is mandatory as a security measure, not just a preference
- **Simplicity Over Patterns**: Use the simplest solution that solves the problem properly
- **Test-First Development**: Design and test are one process, not separate steps
- **Fits in Head Rule**: If you can't reason about a component in one sitting, it's too complex

## Implementation Rules
- NO namespaces (maintains Python-like imports)
- NO complex autoloading
- NO deep class hierarchies
- Use relative requires for imports
- Always use parentheses for function-like calls
- Strict type declarations required in ALL files
- Flat file organization
- Test coverage required BEFORE implementation

## Code Organization

### File Setup
```php
// REQUIRED at the top of EVERY PHP file - Type safety is security
declare(strict_types=1);

// Imports use relative paths with parentheses
require_once('/../lib/Logger.php');
require_once('/../lib/Config.php');

// INCORRECT - No type safety
include 'some_file.php';  // Don't do this
```

### Type Safety Examples
```php
// CORRECT - Full type hints as security measure
function process_payment(float $amount, string $currency): bool {
    return process_secure_transaction($amount, $currency);
}

// INCORRECT - Security vulnerability through loose types
function process_payment($amount, $currency) {
    return process_secure_transaction($amount, $currency);
}
```

### Function Size and Complexity
```php
// CORRECT - Single responsibility, fits in head
function validate_email(string $email): bool {
    return filter_var($email, FILTER_VALIDATE_EMAIL) !== false;
}

// INCORRECT - "Come to Jesus" function, too complex
function process_user($userData) {
    // 100 lines of validation
    // 50 lines of business logic
    // 75 lines of email processing
    // 80 lines of logging
    // Don't do this!
}
```

### Error Handling
```php
class ValidationError extends Exception {
    public function __construct(
        string $message,
        private array $fields = [],
        int $code = 0
    ) {
        parent::__construct($message, $code);
    }
}

// CORRECT - Clear error handling with types
function validate_input(array $data): bool {
    if (!isset($data['email'])) {
        throw new ValidationError(
            message: 'Email is required',
            fields: ['email']
        );
    }
    return true;
}
```

### Testing First Approach
```php
// CORRECT - Test written before implementation
class PaymentProcessorTest {
    public function testPaymentValidation(): void {
        $processor = new PaymentProcessor();
        
        // Define expected behavior
        $this->expectException(ValidationError::class);
        
        // Test invalid input
        $processor->processPayment(-50.00);
    }
}

// Then implement to match test
class PaymentProcessor {
    public function processPayment(float $amount): bool {
        if ($amount <= 0) {
            throw new ValidationError('Invalid amount');
        }
        return true;
    }
}
```

## Project Structure
```
project/
  ├── tests/           # Tests FIRST - matches src structure
  │   ├── Unit/
  │   └── Integration/
  ├── src/
  │   ├── Services/    # Business logic
  │   ├── Types/       # Type definitions
  │   └── Util/        # Utilities
  └── public/          # Entry points
```

## Implementation Patterns

### Simple Over Complex
```php
// CORRECT - Simple and direct
class UserService {
    public function find_by_email(string $email): ?User {
        return $this->db->query(
            'SELECT * FROM users WHERE email = ?',
            [$email]
        )->fetch();
    }
}

// INCORRECT - Overengineered
class AbstractUserServiceFactoryProvider {
    private UserStrategyInterface $strategy;
    // Don't do this!
}
```

### Modular Design
```php
// CORRECT - Clear dependencies, easy to test
class OrderProcessor {
    public function __construct(
        private PaymentService $payment,
        private InventoryService $inventory,
        private Logger $logger
    ) {}
    
    public function process(Order $order): bool {
        // Process order
    }
}
```

## Testing Requirements

### Test Coverage Rules
1. Tests must be written BEFORE implementation
2. 100% coverage required for new code
3. No untested code in production
4. Tests must be clear and readable

### Test Structure
```php
class OrderTest {
    public function setUp(): void {
        // Setup test dependencies
        $this->payment = new MockPaymentService();
        $this->inventory = new MockInventoryService();
        $this->logger = new MockLogger();
        
        $this->processor = new OrderProcessor(
            $this->payment,
            $this->inventory,
            $this->logger
        );
    }
    
    public function testOrderProcessing(): void {
        $order = new Order(['item' => 'test']);
        $result = $this->processor->process($order);
        $this->assertTrue($result);
    }
}
```

## Success Metrics
1. All files pass strict type checking (security)
2. No type coercion warnings
3. All new code has tests BEFORE implementation
4. Test coverage 100% for new code
5. No "Come to Jesus" functions
6. Components "fit in head"
7. No unnecessary abstractions
8. Clear dependency chains

## Implementation Notes
- Start with tests
- Keep components small and focused
- Type safety is a security requirement
- Don't refactor working code without reason
- Monitor resource usage
- Log all errors with context
- Test with type checking enabled
- Profile performance regularly