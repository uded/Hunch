<a href="https://github.com/uded/Hunch">
	<img width="160" src="https://user-images.githubusercontent.com/4630940/59684078-ea9d8c80-920b-11e9-9c99-85051dcb8a04.jpg" alt="Housekeeper" title="Hunch" align="right"/>
</a>

![GitHub tag (latest SemVer)](https://img.shields.io/github/tag/aaronjan/hunch.svg)
![Build status](https://github.com/uded/hunch/workflows/Go/badge.svg?branch=master)
[![codecov](https://codecov.io/gh/AaronJan/Hunch/branch/master/graph/badge.svg)](https://codecov.io/gh/AaronJan/Hunch)
[![Go Report Card](https://goreportcard.com/badge/github.com/uded/hunch)](https://goreportcard.com/report/github.com/uded/hunch)
![GitHub](https://img.shields.io/github/license/aaronjan/hunch.svg)
[![GoDoc](https://godoc.org/github.com/uded/hunch?status.svg)](https://godoc.org/github.com/uded/hunch)

# Hunch

Hunch provides functions like: `All`, `First`, `Retry`, `Waterfall` etc., that makes asynchronous flow control more intuitive.

This is a up-to-date repo for the original one available at https://github.com/AaronJan/Hunch
I will be more than happy to fix any errors, merge pull-requests or just move it forward.

## About Hunch

Go have several concurrency patterns, here are some articles:

* https://blog.golang.org/pipelines
* https://blog.golang.org/context
* https://blog.golang.org/go-concurrency-patterns-timing-out-and
* https://blog.golang.org/advanced-go-concurrency-patterns

But nowadays, using the `context` package is the most powerful pattern.

So base on `context`, Hunch provides functions that can help you deal with complex asynchronous logics with ease.

## Usage

### Installation

#### `go get`

```shell
$ go get -u -v github.com/uded/hunch
```

#### `go mod` (Recommended)

```go
import "github.com/uded/hunch"
```

```shell
$ go mod tidy
```

### Types

```go
type Executable[T interface{} func(context.Context) (T, error)

type ExecutableInSequence[T interface{} func(context.Context, T) (T, error)
```

### API

#### All

```go
func All[T any](parentCtx context.Context, execs ...Executable[T]) ([]T, error) 
```

All returns all the outputs from all Executables, order guaranteed.

##### Examples

```go
ctx := context.Background()
r, err := hunch.All(
    ctx,
    func(ctx context.Context) (int64, error) {
        time.Sleep(300 * time.Millisecond)
        return 1, nil
    },
    func(ctx context.Context) (int64, error) {
        time.Sleep(200 * time.Millisecond)
        return 2, nil
    },
    func(ctx context.Context) (int64, error) {
        time.Sleep(100 * time.Millisecond)
        return 3, nil
    },
)

fmt.Println(r, err)
// Output:
// [1 2 3] <nil>
```

#### Take

```go
func Take[T interface{}](parentCtx context.Context, num int, execs ...Executable[T]) ([]T, error)
```

Take returns the first `num` values outputted by the Executables.

##### Examples

```go
ctx := context.Background()
r, err := hunch.Take(
    ctx,
    // Only need the first 2 values.
    2,
    func(ctx context.Context) (int64, error) {
        time.Sleep(300 * time.Millisecond)
        return 1, nil
    },
    func(ctx context.Context) (int64, error) {
        time.Sleep(200 * time.Millisecond)
        return 2, nil
    },
    func(ctx context.Context) (int64, error) {
        time.Sleep(100 * time.Millisecond)
        return 3, nil
    },
)

fmt.Println(r, err)
// Output:
// [3 2] <nil>
```

#### Last

```go
func Last[T any](parentCtx context.Context, num int, execs ...Executable[T]) ([]T, error)
```

Last returns the last `num` values outputted by the Executables.

##### Examples

```go
ctx := context.Background()
r, err := hunch.Last(
    ctx,
    // Only need the last 2 values.
    2,
    func(ctx context.Context) (int64, error) {
        time.Sleep(300 * time.Millisecond)
        return 1, nil
    },
    func(ctx context.Context) (int64, error) {
        time.Sleep(200 * time.Millisecond)
        return 2, nil
    },
    func(ctx context.Context) (int64, error) {
        time.Sleep(100 * time.Millisecond)
        return 3, nil
    },
)

fmt.Println(r, err)
// Output:
// [2 1] <nil>
```

#### Waterfall

```go
func Waterfall[T any](parentCtx context.Context, execs ...ExecutableInSequence[T]) (T, error)
```

Waterfall runs `ExecutableInSequence`s one by one, passing previous result to next Executable as input. When an error occurred, it stop the process then returns the error. When the parent Context canceled, it returns the `Err()` of it immediately.

##### Examples

```go
ctx := context.Background()
r, err := hunch.Waterfall(
    ctx,
    func(ctx context.Context, n int) (int, error) {
        return 1, nil
    },
    func(ctx context.Context, n int) (int, error) {
        return n + 1, nil
    },
    func(ctx context.Context, n int) (int, error) {
        return n + 1, nil
    },
)

fmt.Println(r, err)
// Output:
// 3 <nil>
```

#### Retry

```go
func Retry[T any](parentCtx context.Context, retries int, fn Executable[T]) (T, error)
```

Retry attempts to get a value from an Executable instead of an Error. It will keeps re-running the Executable when failed no more than `retries` times. Also, when the parent Context canceled, it returns the `Err()` of it immediately.

##### Examples

```go
count := 0
getStuffFromAPI := func() (int, error) {
    if count == 5 {
        return 1, nil
    }
    count++

    return 0, fmt.Errorf("timeout")
}

ctx := context.Background()
r, err := hunch.Retry(
    ctx,
    10,
    func(ctx context.Context) (int, error) {
        rs, err := getStuffFromAPI()

        return rs, err
    },
)

fmt.Println(r, err, count)
// Output:
// 1 <nil> 5
```

## Credits

Heavily inspired by [Async](https://github.com/caolan/async/) and [ReactiveX](http://reactivex.io/).

## Licence

[Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0)
