# Go Implementation Guide

[This guide provides detailed implementation patterns that align with the Core Go Standards]

## Component-Based Development

### Service Pattern
```go
// Service pattern with explicit dependencies
type UserService struct {
    repo    Repository
    logger  Logger
    monitor ResourceMonitor
}

func NewUserService(repo Repository, logger Logger, monitor ResourceMonitor) *UserService {
    return &UserService{
        repo:    repo,
        logger:  logger,
        monitor: monitor,
    }
}

func (s *UserService) Create(ctx context.Context, user *User) error {
    defer s.monitor.Track("user.create")()
    
    if err := user.Validate(); err != nil {
        s.logger.Error("user.create.validation_failed", err, map[string]interface{}{
            "user": user.Email,
        })
        return err
    }
    
    return s.repo.Create(ctx, user)
}
```

### Repository Pattern
```go
type Repository interface {
    Create(ctx context.Context, user *User) error
    FindByEmail(ctx context.Context, email string) (*User, error)
}

type PostgresRepository struct {
    db     *sql.DB
    logger Logger
}

func (r *PostgresRepository) FindByEmail(ctx context.Context, email string) (*User, error) {
    var user User
    err := r.db.QueryRowContext(ctx, 
        "SELECT id, email, name FROM users WHERE email = $1",
        email,
    ).Scan(&user.ID, &user.Email, &user.Name)
    
    if err == sql.ErrNoRows {
        return nil, nil
    }
    if err != nil {
        return nil, fmt.Errorf("finding user: %w", err)
    }
    return &user, nil
}
```

## When to Use Design Patterns

### Factory Functions (Not Factory Pattern)
```go
// GOOD - Simple factory function
func NewUserFromRequest(req *http.Request) (*User, error) {
    email := req.FormValue("email")
    if !validateEmail(email) {
        return nil, errors.New("invalid email")
    }
    
    return &User{
        Email: email,
        Name:  req.FormValue("name"),
    }, nil
}

// BAD - Overcomplicated factory pattern
type UserFactory struct {
    validator *Validator
    enricher  *DataEnricher
}

func (f *UserFactory) CreateUser(req *http.Request) (*User, error) {
    // Don't do this - overly complex for simple object creation
}
```

### Strategy Pattern - Good Use Case
```go
// When you genuinely need runtime behavior switching
type PaymentProcessor interface {
    Process(ctx context.Context, amount int64) error
}

type StripeProcessor struct {
    client *stripe.Client
    logger Logger
}

func (p *StripeProcessor) Process(ctx context.Context, amount int64) error {
    // Stripe-specific implementation
    return nil
}

type PayPalProcessor struct {
    client *paypal.Client
    logger Logger
}

func (p *PayPalProcessor) Process(ctx context.Context, amount int64) error {
    // PayPal-specific implementation
    return nil
}

// Usage - strategy makes sense here
type PaymentService struct {
    processor PaymentProcessor
    monitor   ResourceMonitor
}

func (s *PaymentService) ChargeUser(ctx context.Context, amount int64) error {
    defer s.monitor.Track("payment.charge")()
    return s.processor.Process(ctx, amount)
}
```

### When NOT to Use Strategy Pattern
```go
// BAD - Overcomplicated for simple validation
type Validator interface {
    Validate(data interface{}) error
}

type EmailValidator struct{}

func (v *EmailValidator) Validate(data interface{}) error {
    // Overcomplicated validation through interface
    return nil
}

// GOOD - Simple function
func validateEmail(email string) bool {
    return regexp.MustCompile(`^[^@]+@[^@]+\.[^@]+$`).MatchString(email)
}
```

## Pattern Usage Guidelines

### 1. Start Simple
- Begin with simple functions
- Only use interfaces when you need behavior swapping
- If a function works, don't make it a method
- Keep dependencies explicit

### 2. Good Reasons to Use Patterns
- Runtime behavior switching (Strategy)
- Complex object creation with validation (Factory Function)
- Clear abstraction boundaries (Repository)
- Cross-cutting concerns (Middleware)

### 3. Bad Reasons to Use Patterns
- "Future flexibility"
- "Best practices say so"
- "To make it more object-oriented" when a function would suffice

### 4. Questions to Ask Before Using a Pattern
- Could this be a simple function?
- Do I really need runtime behavior switching?
- Am I adding complexity without benefit?
- Will this make the code harder to understand?

## Concurrent Patterns

### Worker Pool Pattern
```go
type Job struct {
    ID     string
    Data   []byte
    Result chan<- error
}

func StartWorkerPool(ctx context.Context, numWorkers int, jobs <-chan Job) {
    for i := 0; i < numWorkers; i++ {
        go func() {
            for {
                select {
                case job := <-jobs:
                    job.Result <- processJob(job.Data)
                case <-ctx.Done():
                    return
                }
            }
        }()
    }
}
```

### Fan-Out Pattern
```go
func FanOut(ctx context.Context, input <-chan int, numWorkers int) []<-chan int {
    outputs := make([]<-chan int, numWorkers)
    for i := 0; i < numWorkers; i++ {
        outputs[i] = worker(ctx, input)
    }
    return outputs
}

func worker(ctx context.Context, input <-chan int) <-chan int {
    output := make(chan int)
    go func() {
        defer close(output)
        for {
            select {
            case n, ok := <-input:
                if !ok {
                    return
                }
                output <- process(n)
            case <-ctx.Done():
                return
            }
        }
    }()
    return output
}
```

## Error Handling Patterns

### Wrapping Errors
```go
// GOOD - Error wrapping with context
func (s *UserService) UpdateProfile(ctx context.Context, userID string, profile *Profile) error {
    user, err := s.repo.Find(ctx, userID)
    if err != nil {
        return fmt.Errorf("finding user %s: %w", userID, err)
    }
    
    if err := profile.Validate(); err != nil {
        return fmt.Errorf("validating profile: %w", err)
    }
    
    return s.repo.Update(ctx, user.ID, profile)
}

// BAD - Lost context
func (s *UserService) UpdateProfile(ctx context.Context, userID string, profile *Profile) error {
    user, err := s.repo.Find(ctx, userID)
    if err != nil {
        return err // Don't do this - context is lost
    }
    // ...
}
```

### Custom Error Types
```go
type ValidationError struct {
    Field string
    Error error
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed for %s: %v", e.Field, e.Error)
}

func (e *ValidationError) Unwrap() error {
    return e.Error
}
```

## Testing Patterns

### Table-Driven Tests
```go
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name    string
        email   string
        wantErr bool
    }{
        {
            name:    "valid email",
            email:   "user@example.com",
            wantErr: false,
        },
        {
            name:    "invalid email",
            email:   "not-an-email",
            wantErr: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := validateEmail(tt.email)
            if (err != nil) != tt.wantErr {
                t.Errorf("validateEmail() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

### Integration Tests
```go
func TestUserService_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }
    
    // Setup test database
    db, cleanup := setupTestDB(t)
    defer cleanup()
    
    // Create service with real dependencies
    repo := NewPostgresRepository(db)
    logger := NewTestLogger(t)