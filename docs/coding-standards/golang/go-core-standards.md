# Go Core Standards

[For detailed implementation examples and patterns, see the Implementation Guide]

## Core Philosophy

### Primary Goals
- Create clear, maintainable, and secure Go code
- Keep systems simple enough to "fit in your head"
- Maintain explicit behavior over implicit magic
- Design for testing from the start

### Foundational Principles
- **Explicit Over Implicit**: Code should be clear and obvious rather than clever
- **Error Handling as Flow**: Errors are values and part of the normal flow
- **Test-First Development**: Tests drive design, not validate it
- **Resource Awareness**: Monitor and manage resources proactively
- **Fits in Head Rule**: If you can't reason about a component in one sitting, it's too complex

## Implementation Rules
- NO global state (makes testing impossible)
- NO panic in production code (breaks flow control)
- NO empty interface parameters (defeats type safety)
- Use direct dependencies only (makes reasoning easier)
- Keep package structure flat (supports "fits in head")
- Always handle errors explicitly
- Monitor resource usage consistently
- Write tests before implementation

## Code Organization

### Package Structure
```go
// CORRECT - Small focused interfaces
type Parser interface {
    // Only methods that are needed, nothing more
    Parse(input string) (Command, error)
}

// CORRECT - Clear dependencies, easy to test
type CommandProcessor struct {
    parser   Parser           // Required for parsing
    monitor  ResourceMonitor  // Required for monitoring
    logger   Logger          // Required for logging
}

// INCORRECT - Empty interfaces
type Handler interface{} // Don't do this

// INCORRECT - Global state
var globalCache map[string]interface{} // Don't do this
```

### Error Handling
```go
// CORRECT - Errors with context
type CommandError struct {
    Command string
    Context map[string]interface{}
    Err     error
}

func (e *CommandError) Error() string {
    return fmt.Sprintf("%s: %v", e.Command, e.Err)
}

// CORRECT - Error handling pattern
func ProcessCommand(cmd string) (*Result, error) {
    if cmd == "" {
        return nil, &CommandError{
            Command: "process",
            Context: map[string]interface{}{"input": cmd},
            Err:     errors.New("empty command"),
        }
    }
    
    // Processing...
    return result, nil
}

// INCORRECT - Panic in production
func ProcessOrPanic(cmd string) *Result {
    if cmd == "" {
        panic("empty command") // Don't do this
    }
    return result
}
```

### Size and Complexity
```go
// CORRECT - Simple, focused function
func validateEmail(email string) error {
    if !strings.Contains(email, "@") {
        return errors.New("invalid email format")
    }
    return nil
}

// INCORRECT - "Come to Jesus" function
func ProcessUserData(data map[string]interface{}) error {
    // 100 lines of validation
    // 50 lines of business logic
    // 75 lines of database operations
    // Don't do this!
}
```

### Testing First
```go
// CORRECT - Test driven development
func TestCommandProcessing(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    *Result
        wantErr bool
    }{
        {
            name:    "empty command",
            input:   "",
            want:    nil,
            wantErr: true,
        },
        {
            name:    "valid command",
            input:   "status",
            want:    &Result{Status: "ok"},
            wantErr: false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            monitor := NewTestMonitor(t)
            monitor.Start()
            defer monitor.Stop()

            got, err := ProcessCommand(tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("ProcessCommand() error = %v, wantErr %v", err, tt.wantErr)
                return
            }
            if !reflect.DeepEqual(got, tt.want) {
                t.Errorf("ProcessCommand() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

### Resource Management
```go
type Metrics struct {
    AllocBytes   int64
    NumGC        uint32
    NumGoroutine int
    Uptime       time.Duration
}

// CORRECT - Resource monitoring
type Monitor struct {
    startTime time.Time
    logger    Logger
}

func (m *Monitor) Start() {
    m.startTime = time.Now()
    m.logger.Info("monitor.started", nil)
}

func (m *Monitor) GetMetrics() Metrics {
    var stats runtime.MemStats
    runtime.ReadMemStats(&stats)
    
    return Metrics{
        AllocBytes:   stats.Alloc,
        NumGC:        stats.NumGC,
        NumGoroutine: runtime.NumGoroutine(),
        Uptime:       time.Since(m.startTime),
    }
}
```

## Project Structure
```
project/
  ├── cmd/              # Command-line applications
  │   ├── api/          # API server
  │   └── worker/       # Background worker
  ├── internal/         # Private packages
  │   ├── platform/     # Platform-specific code
  │   └── auth/         # Authentication
  ├── pkg/              # Public packages
  │   ├── monitor/      # Resource monitoring
  │   └── errors/       # Error types
  └── tests/            # Integration tests
```

## Implementation Patterns

### Simple Over Complex
```go
// CORRECT - Simple and direct
type UserService struct {
    db *sql.DB
}

func (s *UserService) FindByEmail(ctx context.Context, email string) (*User, error) {
    var user User
    err := s.db.QueryRowContext(ctx, 
        "SELECT * FROM users WHERE email = $1", 
        email,
    ).Scan(&user.ID, &user.Email)
    return &user, err
}

// INCORRECT - Overengineered
type AbstractUserServiceFactoryProvider struct {
    strategy UserStrategyInterface
    // Don't do this!
}
```

### Modular Design
```go
// CORRECT - Clear dependencies, easy to test
type OrderProcessor struct {
    payment   PaymentService
    inventory InventoryService
    logger    Logger
}

func NewOrderProcessor(
    payment PaymentService,
    inventory InventoryService,
    logger Logger,
) *OrderProcessor {
    return &OrderProcessor{
        payment:   payment,
        inventory: inventory,
        logger:    logger,
    }
}
```

## Success Metrics
1. All errors properly handled and logged
2. No global state
3. Tests written BEFORE implementation
4. Test coverage > 90% (100% for new code)
5. No data races (verified by race detector)
6. All public APIs documented
7. No "Come to Jesus" functions
8. Components "fit in head"

## Implementation Notes
- Start with tests
- Use channels for concurrency control
- Keep components small and focused
- Monitor goroutine count
- Use structured logging
- Profile performance regularly
- Document all exported identifiers
- Don't refactor working code without reason
- Run tests with race detector