# Research Findings: py-effect (pyeffects) Package

## Package Overview

**Name:** pyeffects  
**Author:** Vic Kumar  
**License:** Apache 2.0  
**Repository:** https://github.com/vickumar1981/pyeffects  
**Documentation:** https://pyeffects.readthedocs.io/  
**PyPI:** https://pypi.org/project/pyeffects/  

## Description

pyeffects is a Python package that implements functional monadic types for handling side-effects explicitly. It provides implementations for:
- **Option** (Some/Empty) - for handling optional values
- **Try** (Success/Failure) - for exception handling
- **Either** (Left/Right) - for representing values with two possibilities
- **Future** - for asynchronous computations

## Relationship to Effect-TS

While the package description mentions "following the model set by Effect-TS", pyeffects is actually a traditional monad library similar to Scala's effect types, rather than a direct port of Effect-TS concepts. Effect-TS is a TypeScript library that provides:
- Maximum type-safety including error handling
- Composable, reusable code patterns
- Built-in retry, interruption, and concurrency primitives
- Observability with tracing and metrics
- A comprehensive standard library for TypeScript

## Core Architecture

### Base Class: Monad

The `Monad` class is the abstract base class that all effect types inherit from:

**Key Properties:**
- `value: A` - the wrapped value
- `biased: bool` - indicates if the monad contains a "success" value

**Key Methods:**
- `of(x)` - static constructor
- `map(func)` - transforms the value
- `flat_map(func)` - monadic bind operation
- `foreach(func)` - side-effect iteration
- `get()` - extracts the value
- `get_or_else(v)` - provides default value
- `or_else(other)` - alternative monad
- `or_else_supply(func)` - lazy alternative

### Option Type

**Classes:** `Option`, `Some`, `Empty`

**Purpose:** Represents optional values (similar to Scala's Option or Haskell's Maybe)

**Key Methods:**
- `is_defined()` - returns True if Some
- `is_empty()` - returns True if Empty
- `flat_map(func)` - chains operations

**Usage Pattern:**
```python
Some(5).map(lambda v: v * v)  # Some(25)
Option.of(None)  # Empty()
```

### Try Type

**Classes:** `Try`, `Success`, `Failure`

**Purpose:** Encapsulates computations that may throw exceptions

**Key Methods:**
- `is_success()` - returns True if Success
- `is_failure()` - returns True if Failure
- `recover(err, recover)` - recovers from specific exception
- `recovers(errs, recover)` - recovers from multiple exceptions
- `error()` - extracts the exception
- `on_success(func)` - callback on success
- `on_failure(func)` - callback on failure

**Usage Pattern:**
```python
Try.of(lambda: 5/0)  # Failure(ZeroDivisionError)
Success(5).map(lambda v: v * v)  # Success(25)
```

### Either Type

**Classes:** `Either`, `Left`, `Right`

**Purpose:** Represents a value of one of two possible types (a disjoint union)

**Key Methods:**
- `is_right()` - returns True if Right
- `is_left()` - returns True if Left
- `right()` - extracts right value
- `left()` - extracts left value

**Usage Pattern:**
```python
Right(5).map(lambda v: v * v)  # Right(25)
Left("error").map(lambda v: v * v)  # Left("error")
```

### Future Type

**Classes:** `Future`

**Purpose:** Represents asynchronous computations with thread-based execution

**Key Features:**
- Thread-based asynchronous execution
- Callback-based completion handling
- Caching of results
- Thread-safe subscriber management using semaphores

**Key Methods:**
- `run(func)` - executes function on new thread
- `is_done()` - checks if computation completed
- `is_success()` / `is_failure()` - checks result status
- `on_complete(subscriber)` - registers callback
- `on_success(subscriber)` - callback on success
- `on_failure(subscriber)` - callback on failure
- `traverse(arr)` - sequences array of Futures

**Implementation Details:**
- Uses Python's `threading` module
- Maintains subscriber list for callbacks
- Uses `BoundedSemaphore` for thread safety
- Wraps results in Try (Success/Failure)

## Type System

The package uses Python's typing module with:
- Generic types: `Generic[A]`
- Type variables: `TypeVar("A", covariant=True)`
- Callable types for function parameters
- Union types for multiple possibilities

## Design Patterns

1. **Monad Pattern**: All types implement monadic interface (map, flat_map)
2. **Bias Pattern**: Uses `biased` flag to distinguish success/failure cases
3. **Factory Pattern**: Static `of()` methods for construction
4. **Callback Pattern**: Future uses callbacks for async completion
5. **Thread Safety**: Semaphores for concurrent access in Future

## Key Differences from Effect-TS

1. **Scope**: pyeffects focuses on core monadic types; Effect-TS is a comprehensive ecosystem
2. **Error Handling**: pyeffects uses traditional monads; Effect-TS has advanced error tracking
3. **Concurrency**: pyeffects uses basic threading; Effect-TS has sophisticated concurrency primitives
4. **Observability**: pyeffects lacks built-in tracing/metrics that Effect-TS provides
5. **Type Safety**: Effect-TS leverages TypeScript's type system more extensively
6. **Standard Library**: Effect-TS provides extensive utilities; pyeffects is focused on effects

## Source Code Structure

```
pyeffects/
├── Monad.py         # Base monad class
├── Option.py        # Option, Some, Empty
├── Either.py        # Either, Left, Right
├── Try.py           # Try, Success, Failure
├── Future.py        # Future with async support
├── __init__.py      # Package initialization
└── __version__.py   # Version information
```

