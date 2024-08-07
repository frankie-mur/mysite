+++
title = 'Go: some context about context'
date = 2024-08-06T22:48:54-07:00
draft = true
tage = ["golang"]
+++
## In its simpliest form, context is an interface
from the [Go docs](https://pkg.go.dev/context)
```
     Package context defines the Context type, which carries deadlines, cancellation signals, and other request-scoped values across API boundaries and between processes.
```

#### This may seem confusing but in essense context serves two purposes within a Go application
1. Timeouts
2. Storing values

In general to craete a context you will call the `context.Background()` function.
when you are unsure of what context to use you will use `context.TODO()` function

# Timeouts
```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()

    val, err := doSomething(ctx)
    ...
}
```
In this example we are creating a context with a timeout of 1 second, this means that if the function `doSomething` does not return within 1 second the context will be canceled.


# Storing Values
```go
func main() {
    ctx := context.WithValue(context.Background(), "key", "value")
    val := ctx.Value("key")
    val, err := doSomething(ctx)
    ...
}
```
In this example we are creating a context with a value of "key" and "value". We can then access this value later in our code by calling `ctx.Value("key")`, for example:

```go
func doSomething(ctx context.Context) (string, error) {
    val := ctx.Value("key").(string)
    // do something with val    
    ...
}
```
## Conclusion
Context is a powerful tool in Go, it is used to pass values and timeouts between functions. It is used in many places in the Go standard library, such as the `net/http` package and the `database/sql` package. Have fun using it!
