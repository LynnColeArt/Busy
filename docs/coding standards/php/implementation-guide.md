# Pythonic PHP Implementation Guide

[This guide provides detailed implementation patterns that align with the Core PHP Standards]

## Component-Based Development

### Service Pattern (Alternative to MVC)
```php
// [Aligns with Core Standards anti-MVC philosophy]
declare(strict_types=1);

require_once('/../services/UserService.php');
require_once('/../repositories/UserRepository.php');

class UserService {
    public function __construct(
        private UserRepository $repository,
        private Logger $logger
    ) {}

    public function register(array $data): ?array {
        try {
            // Validation
            if (!$this->validate_registration($data)) {
                return null;
            }

            // Processing
            $user = $this->repository->create($data);
            $this->logger->info('User registered', ['id' => $user['id']]);
            return $user;
        } catch (Exception $e) {
            $this->logger->error($e->getMessage());
            return null;
        }
    }

    private function validate_registration(array $data): bool {
        return isset($data['email']) && filter_var($data['email'], FILTER_VALIDATE_EMAIL);
    }
}
```

### Repository Pattern
```php
declare(strict_types=1);

class UserRepository {
    private Database $db;

    public function __construct(Database $db) {
        $this->db = $db;
    }

    public function find_by_email(string $email): ?array {
        $result = $this->db->query(
            'SELECT * FROM users WHERE email = ?',
            [$email]
        );
        return $result->fetch() ?: null;
    }
}
```

## When to Use Design Patterns

### Factory Functions (Not Factory Pattern)
```php
declare(strict_types=1);

class User {
    public function __construct(
        private int $id,
        private string $name,
        private string $email,
        private array $preferences = []
    ) {}

    // Simple factory function - good!
    public static function from_database(array $row): self {
        return new self(
            id: (int)$row['id'],
            name: $row['name'],
            email: $row['email'],
            preferences: json_decode($row['preferences'], true) ?? []
        );
    }
}

// Usage
$user = User::from_database($row);
```

### When NOT to Use Factory Pattern
```php
// DON'T DO THIS - overcomplicated!
class UserFactory {
    public function createBasicUser(string $name): User {
        return new User(0, $name, '', []);
    }
    
    public function createPremiumUser(string $name): User {
        return new User(0, $name, '', ['premium' => true]);
    }
}

// Just do this instead:
$basic_user = new User(0, $name, '', []);
$premium_user = new User(0, $name, '', ['premium' => true]);
```

### Strategy Pattern - Good Use Case
```php
declare(strict_types=1);

// When you genuinely need runtime swappable behaviors
interface PaymentProcessor {
    public function process(float $amount): bool;
}

class StripeProcessor implements PaymentProcessor {
    public function process(float $amount): bool {
        return true; // Stripe-specific implementation
    }
}

class PayPalProcessor implements PaymentProcessor {
    public function process(float $amount): bool {
        return true; // PayPal-specific implementation
    }
}

// Usage - strategy makes sense here
class PaymentService {
    public function __construct(
        private PaymentProcessor $processor
    ) {}

    public function charge(float $amount): bool {
        return $this->processor->process($amount);
    }
}
```

### When NOT to Use Strategy Pattern
```php
// DON'T DO THIS - overcomplicated!
interface UserValidator {
    public function validate(array $data): bool;
}

class EmailValidator implements UserValidator {
    public function validate(array $data): bool {
        return isset($data['email']) && 
               filter_var($data['email'], FILTER_VALIDATE_EMAIL);
    }
}

// Just do this instead:
class UserService {
    public function validate_email(string $email): bool {
        return filter_var($email, FILTER_VALIDATE_EMAIL) !== false;
    }
}
```

### Pattern Usage Guidelines

1. **Start Simple**
   - Begin with plain functions and simple classes
   - Only add patterns when complexity actually requires them
   - If you can solve it with a function, don't use a class

2. **Good Reasons to Use Patterns**
   - Runtime behavior switching (Strategy)
   - Complex object creation with validation (Factory Function)
   - Plugin/extension systems (Strategy)
   - Third-party API integration (Adapter)

3. **Bad Reasons to Use Patterns**
   - "We might need flexibility later"
   - "It's more object-oriented"
   - "That's how Java does it"
   - To make simple code look more "enterprise"

4. **Questions to Ask Before Using a Pattern**
   - Could this be a simple function?
   - Would a plain class work just as well?
   - Am I adding complexity without adding value?
   - Will this make the code harder to understand?