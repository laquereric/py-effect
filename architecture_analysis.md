# Architecture Analysis: pyeffects Package

## Package Metrics

- Total lines of code: ~795
- Core modules: 5 (Monad, Option, Either, Try, Future)
- Version: 1.00.5
- Language: Python 3.6+

## Module Breakdown

| Module | Lines | Purpose | Key Classes |
|--------|-------|---------|-------------|
| Monad.py | 61 | Base abstraction | Monad |
| Option.py | 114 | Optional values | Option, Some, Empty |
| Either.py | 116 | Disjoint union | Either, Left, Right |
| Try.py | 211 | Exception handling | Try, Success, Failure |
| Future.py | 260 | Async computation | Future |

## Class Hierarchy

```
Monad[A] (Generic base class)
├── Option[A]
│   ├── Some[A]
│   └── Empty[A]
├── Either[A]
│   ├── Left[A]
│   └── Right[A]
├── Try[A]
│   ├── Success[A]
│   └── Failure[A]
└── Future[A]
```

## Design Patterns Identified

### 1. Template Method Pattern
The `Monad` base class defines the template for monadic operations:
- `map()` is implemented in base class using `flat_map()`
- Subclasses must implement `flat_map()` and `of()`
- Provides default implementations for utility methods

### 2. Factory Method Pattern
Each monad type provides static `of()` method:
- `Option.of(value)` - creates Some or Empty based on None check
- `Either.of(value)` - creates Right by default
- `Try.of(func_or_value)` - creates Success or Failure based on exception
- `Future.of(value)` - creates immediate Future
- `Future.run(func)` - creates async Future on new thread

### 3. Strategy Pattern
The `biased` boolean flag determines behavior:
- When `biased=True`: monad contains "success" value (Some, Right, Success)
- When `biased=False`: monad contains "failure" value (Empty, Left, Failure)
- Methods like `get()`, `get_or_else()`, `or_else()` use this flag

### 4. Chain of Responsibility Pattern
Monadic chaining through `map()` and `flat_map()`:
- Operations are chained without explicit conditional logic
- Each monad decides whether to apply transformation or short-circuit
- Enables fluent API design

### 5. Observer Pattern (Future)
Future implements observer pattern for async completion:
- Maintains list of subscribers
- Notifies subscribers when computation completes
- Thread-safe notification using semaphores

### 6. Singleton Pattern (Empty)
The `Empty` class uses singleton pattern:
- `empty = Empty()` - single instance created at module level
- All Empty references point to same instance
- Saves memory and comparison overhead

## Type System Architecture

### Generic Type Variables

```python
A = TypeVar("A", covariant=True)  # Covariant for return types
B = TypeVar("B")                   # Invariant for transformations
```

### Covariance Strategy
- `A` is covariant: allows `Monad[Derived]` to be used where `Monad[Base]` expected
- Enables flexible type composition
- Safe for immutable containers

### Type Signatures

**Monad operations:**
- `of(x: B) -> Monad[B]` - constructor
- `map(func: Callable[[A], B]) -> Monad[B]` - transformation
- `flat_map(func: Callable[[A], Monad[B]]) -> Monad[B]` - monadic bind
- `get() -> A` - extraction (may raise)
- `get_or_else(v: A) -> A` - safe extraction

## Bias-Based Polymorphism

The `biased` flag enables polymorphic behavior without explicit type checks:

| Type | Success (biased=True) | Failure (biased=False) |
|------|----------------------|------------------------|
| Option | Some(value) | Empty() |
| Either | Right(value) | Left(value) |
| Try | Success(value) | Failure(exception) |
| Future | Always True | N/A (uses wrapped Try) |

## Functional Programming Principles

### 1. Immutability
- All monad instances are immutable
- Operations return new instances
- No in-place modifications

### 2. Referential Transparency
- Pure functions in transformations
- Same input always produces same output
- Side-effects isolated in Future

### 3. Composition
- Monads compose through `flat_map()`
- Enables building complex operations from simple ones
- Type-safe composition

### 4. Lazy Evaluation (Partial)
- `or_else_supply()` takes function for lazy evaluation
- Future computations are lazy until executed
- Most operations are eager

## Concurrency Model (Future)

### Threading Architecture
```
Main Thread
    │
    ├─> Future.run(func)
    │       │
    │       └─> Worker Thread
    │               │
    │               ├─> Execute func()
    │               └─> Callback with Try[A]
    │
    └─> Subscribers notified on separate threads
```

### Thread Safety Mechanisms
1. **BoundedSemaphore**: Protects subscriber list
2. **Cache**: Option[Try[A]] stores result
3. **Atomic operations**: Subscriber notification

### Callback Flow
1. Future created with callback function
2. Computation runs on worker thread
3. Result wrapped in Try (Success/Failure)
4. Subscribers notified via `_callback()`
5. Each subscriber runs on separate thread

## Error Handling Strategy

### Try Monad
- **Eager exception catching**: `Try.of()` catches at construction
- **Typed recovery**: `recover(err_type, value)` for specific exceptions
- **Multiple recovery**: `recovers([err_types], value)` for multiple types
- **Error extraction**: `error()` method for Failure inspection

### Future Monad
- **Wraps in Try**: All Future results are Try[A]
- **Separate callbacks**: `on_success()` and `on_failure()`
- **Thread-safe error propagation**: Errors propagate through callbacks

### Option and Either
- **No exceptions**: Use Empty or Left for absence/failure
- **Type-safe**: Compiler/type checker enforces handling

## Method Categories

### Construction Methods
- `of()` - static factory
- `__init__()` - direct construction

### Transformation Methods
- `map()` - functor mapping
- `flat_map()` - monadic bind
- `foreach()` - side-effect iteration

### Extraction Methods
- `get()` - unsafe extraction (may raise)
- `get_or_else()` - safe with default
- `or_else()` - alternative monad
- `or_else_supply()` - lazy alternative

### Predicate Methods
- `is_defined()` / `is_empty()` - Option
- `is_right()` / `is_left()` - Either
- `is_success()` / `is_failure()` - Try
- `is_done()` - Future

### Recovery Methods (Try)
- `recover()` - single exception type
- `recovers()` - multiple exception types
- `error()` - extract exception

### Callback Methods (Future)
- `on_complete()` - any completion
- `on_success()` - successful completion
- `on_failure()` - failed completion

### Utility Methods (Future)
- `traverse()` - sequence list of Futures

## Relationships Between Types

### Compositional Relationships
1. **Future wraps Try**: `Future[A]` internally uses `Try[A]`
2. **Future uses Option**: Cache is `Option[Try[A]]`
3. **Try can wrap any type**: Including Option and Either
4. **All extend Monad**: Uniform interface

### Conversion Possibilities
- Option → Either: `Some(x) → Right(x)`, `Empty → Left(error)`
- Try → Either: `Success(x) → Right(x)`, `Failure(e) → Left(e)`
- Try → Option: `Success(x) → Some(x)`, `Failure(_) → Empty`
- Any → Future: Wrap in `Future.of()`

## Key Algorithms

### map() Implementation (Monad base class)
```python
def map(self, func: Callable[[A], B]) -> Monad[B]:
    def wrapped(x: A) -> Monad[B]:
        return self.of(func(x))
    return self.flat_map(wrapped)
```
- Wraps function result in monad using `of()`
- Delegates to `flat_map()` for actual execution
- Ensures consistent behavior across all monad types

### Future._callback() Algorithm
```python
def _callback(self, value: Try[A]) -> None:
    self.value = value
    self.semaphore.acquire()
    self.cache = Some(value)
    while len(self.subscribers) > 0:
        sub = self.subscribers.pop(0)
        t = threading.Thread(target=sub, args=[value])
        t.start()
    self.semaphore.release()
```
- Thread-safe result storage
- Notifies all subscribers
- Each subscriber runs on separate thread

### Future.traverse() Algorithm
```python
@staticmethod
def traverse(arr):
    return reduce(
        lambda acc, elem: acc.flat_map(
            lambda values: elem.map(lambda value: values + [value])
        ),
        arr,
        Future.of([]),
    )
```
- Sequences array of Futures into Future of array
- Uses reduce with flat_map for composition
- Maintains order of results

## Extensibility Points

1. **New Monad Types**: Extend `Monad` base class
2. **Custom Recovery**: Add recovery methods to Try
3. **Future Executors**: Customize thread execution
4. **Type Constraints**: Add validation in `of()` methods

## Performance Considerations

1. **Thread Overhead**: Each Future.run() creates new thread
2. **Semaphore Contention**: Subscriber management may block
3. **Memory**: Each monad instance allocates object
4. **Singleton Empty**: Reduces memory for Option
5. **No Tail Call Optimization**: Deep recursion may stack overflow

## Comparison with Effect-TS Concepts

| Concept | pyeffects | Effect-TS |
|---------|-----------|-----------|
| Error Tracking | Try monad | Effect[R, E, A] with typed errors |
| Async | Future with threads | Effect with fibers |
| Retry | Manual | Built-in retry combinators |
| Interruption | Thread termination | Cooperative interruption |
| Dependency Injection | Manual | Layer system |
| Observability | None | Built-in tracing/metrics |
| Resource Management | Manual | Scope-based |
| Concurrency | Thread-based | Fiber-based |

