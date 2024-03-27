# Go gRPC interceptors under the hood

The Go gRPC interceptor system is kind of neat. You hand it a list of functions, and it'll convert them into an onion of functions that are executed on each request. At first glance, it's easy to assume that it's a just list of functions that are executed in order, but it's more nuanced than that. 

Lets look at the simplest possible interceptor:
```go
func emptyInterceptor(ctx context.Context, req interface{}, _ *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	return handler(ctx, req)
}
```
All this does is run the handler that's passed into it. Conceptually it can be thought of as:

```
request -> emptyInterceptor -> handler
```

Lets say you're building a middleware handler from scratch. The simple usecase that springs to mind is handling auth. If the user isn't auth'd, reject the request. And while we're at it, lets add some logging.

```go
type MiddlewareFunction = func(req any) (error)

func AuthMiddleware(_ context.Context, req any) (error) {
    if req.Token != "some-secret-token" {
        return nil, fmt.Errorf("user not auth'd")
    }
}

func AddLog(req any) (error) {
    logger.Info("=== Got request: ", req)
    return nil
}
```

The simplest way to handle this is with a list of functions that are executed sequentially, and if none of them fail, execute the main handler:

```go
func executeRequest(req any, handler handlerFunc, middleware []MiddlewareFunc) (resp, error) {
    for _, mw := range(middleware) {
        err := mw(req)
        if err != nil {
            // We got an error, break out
            return nil, err
        }
    }
    
    return handler(req)
}
```

This is fine for really simple usecases, but what if we want to add tracing? What if we want to emit logs before _and_ after the request. What if we want to inject data into the context after the request has been handled? Enter, The Interceptor:

## Interceptor

If we look at the function signature for a gRPC interceptor, it hints at how it should be used:
```go
func emptyInterceptor(ctx context.Context, req interface{}, _ *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error)
```

If we strip out the bits we don't need for this example, we're left with:

```go
func emptyInterceptor(req any, handler grpc.UnaryHandler) (any, error)
```

Take in a `req` and `handler`, return `any` (response) or an error. What's the big deal with this, and why's it different from what we did above? Well we can achieve the same thing as above, lets give it a go.

```go
func LogMiddleware(req any, handler grpc.UnaryHandler) (any, error) {
    log.Info("=== Got request:", req)
	return handler(ctx, req)
}

func AuthMiddleware(req any, handler grpc.UnaryHandler) (any, error) {
    if req.Token != "some-secret-token" {
        return nil, fmt.Errorf("user not auth'd")
    }
    return handler(ctx, req)
}
```

But this _also_ lets you do some neat things like:

```go
func LogMiddleware(req any, handler grpc.UnaryHandler) (any, error) {
    start := time.Now()
    log.Info("=== Got request:", req)
    res, err := handler(ctx, req)
    log.Infof("=== Completed request, took %s seconds", time.Since(start))
    
	return res, err
}
```

This is basically the building blocks for things like [request tracing](https://github.com/tel-io/otelgrpc?tab=readme-ov-file#server-side).

How would this be implemented, though? Lets try loop over the list of functions and see what happens:

```go
type middlewareFn = func(req any, handler handlerFn) (any, error)
type handlerFn = func(req any) (any, error)
middlewareFunctions := []middlewareFn{
    LogMiddleware,
    AuthMiddleware
}

func executeHandlerWithMiddleware(req any, baseHandler handlerFn, middlewareFunctions []middlewareFn) (any, error) {
    for _, mw := range(middlewareFunctions) {
        res, err := mw(req, ???)
    }
}

```
We get stuck here. What if we try this:

```go
func executeHandlerWithMiddleware(req any, baseHandler handlerFn, middlewareFunctions []middlewareFn) (any, error) {
	for idx, mw := range middlewareFunctions {
		if idx == len(middlewareFunctions)-1 {
			res, err := baseHandler(req)
		} else {
			res, err := mw(req, middlewareFunctions[idx+1])
		}
	}
}
```

This still doesn't work, as `middlewareFn` isn't compatible with `handlerFn`. Also what do we do with the result from each of the handlers? If we make any modifications to the request, we'll need to keep track of that when passing it into the next handler. What about the result? How do we know if we should overwrite the result?

We're going to need a different approach. What if we could wrap the functions over eachother, so that they're executed like:

```go
res, err := LoggerMiddleware(
    req,
    AuthMiddleware
)
```
But this doesn't work, either. `AuthMiddleware` (of type `func(any, handlerFn) (any, error)` is the wrong type. It's expecting something of `func(any) (any, error)` (aka `handlerFn`). What if we could 'bind' `AuthMiddleware` to whatever the next function should be? Let's try that:

```go
boundAuthMiddleware := func(handler handlerFn) handlerFn {
	return func(req any) (any, error) {
		return AuthMiddleware(req, handler)
	}
}
```

All we're doing here is returning something in the `handlerFn` shape that has `AuthMiddleware` 'bound' to whatever handler is provided. This works because the only difference in 'shape' of `handlerFn` and `middlewareFn` is that `middlewareFn` has a second parameter for the handler. If that can be pre-populated, the signature is (effectively) the same.

Now we're getting somewhere. The code below fits the expected shape.

```go
res, err := LogMiddleware(
	req,
	wrappedAuthMiddleware(baseFn),
)
```

Lets make that wrapper generic:

```go
wrapMiddleware := func(middleware middlewareFn, handler handlerFn) handlerFn {
	return func(req any) (any, error) {
		return middleware(req, handler)
	}
}
```

Now we can run:

```go
res, err := LogMiddleware(
	req,
	wrapMiddleware(AuthMiddleware, baseFn),
)
```

Lets add another piece of middleware and see how it fits in:

```go
func TraceMiddleware(req any, handler grpc.UnaryHandler) (any, error) {
    trace.Start()
    res, err := handler(ctx, req)
    trace.End()
	return res, err
}

res, err := LogMiddleware(
	req,
	wrapMiddleware(
	    AuthMiddleware,
	    wrapMiddleware(TraceMiddleware, baseFn)
    ),
)
```

It seems like we're getting deeper and deeper as we add more middleware, we'll come back to this in a moment. Going back to the original problem that we're trying to solve, we have:

```go
middlewareFns := middlewareFn{
    LogMiddleware,
    AuthMiddleware,
    TraceMiddleware,
}

// A sample baseFn that just returns the request
baseFn := func(req any) (any, error) {
    fmt.Println("Got to the base function")
    return req, nil
}
```

How can we combine the various middleware functions _and_ the `baseFn` into a single function that has the `handlerFn` signature?

We know that we'll likely need a function that takes in both and returns a `handlerFn`, so lets start with that:

```go
function combineMiddlewareAndHandler(middleware []middlewareFn, handler handlerFn) handlerFn {}
```

We saw above that as we added more middleware, the function calls got deeper. This hints at the need for some recursion. Lets add something to chain the current function with the next one. We'll need to keep track of _some_ index to know which is the current function, and which is the next function:

```go
function chainFunction(middleware []middlewareFn, current int) handlerFn {
    return wrapMiddleware(middleware[current], middleware[current+1])
}
```

This won't recurse, though. It'll run once and not keep iterating through the list of middleware. Lets reimplement `wrapMiddleware`

```go
function chainFunction(middleware []middlewareFn, current int) handlerFn {
    return func(req any) (any, error) {
        currentFunction := middleware[current]
        return currentFunction(
            req,
            chainFunction(middleware, current+1)
        )
    }
}
```
Note that this is nearly identical to what we were doing above when we first wrote `wrapMiddleware`.

This'll quickly run into an out of bounds error, though. What do we want to run on the last iteration? The `baseFn`, of course! Lets add some guards in:

```go
function chainFunction(middleware []middlewareFn, baseFn handlerFn, current int) handlerFn {
	if current == len(middleware) {
		return baseFn
	}

    return func(req any) (any, error) {
        currentFunction := middleware[current]
        return currentFunction(
            req,
            chainFunction(middleware, current+1)
        )
    }
}
```

At this point, we could just run:
```go
middlewareFns := middlewareFn{
    LogMiddleware,
    AuthMiddleware,
    TraceMiddleware,
}

baseFn := func(req any) (any, error) {
    fmt.Println("Got to the base function")
    return req, nil
}

chainedFunction := chainFunction(middlewareFns, baseFn, 0)
```
and it'd all work. But that's a leaky interface, so lets take the `combineMiddlewareAndHandler` function above and call what we've called above in it:

```go
function combineMiddlewareAndHandler(middleware []middlewareFn, handler handlerFn) handlerFn {
    return chainFunction(middlewareFns, baseFn, 0)
}
```

Putting it all together, we get:

```go
type middlewareFn = func(req any, handler handlerFn) (any, error)
type handlerFn = func(req any) (any, error)

func TraceMiddleware(req any, handler handlerFn) (any, error) {
	fmt.Println("Starting trace")
	res, err := handler(req)
	fmt.Println("Ending trace")
	return res, err
}

func LogMiddleware(req any, handler handlerFn) (any, error) {
	fmt.Println("Hit log middleware")
	return handler(req)
}

func AuthMiddleware(req any, handler handlerFn) (any, error) {
	fmt.Println("Hit auth middleware")
	if false {
		return nil, fmt.Errorf("user not auth'd")
	}
	return handler(req)
}

baseFn := func(req any) (any, error) {
	fmt.Println("Made it to the base function")
	return req, nil
}

middlewareFunctions := []middlewareFn{
	LogMiddleware,
	AuthMiddleware,
	TraceMiddleware,
}

chainedFunction := combineMiddlewareAndHandler(middlewareFunctions, baseFn)

res, err := chainedFunction("hello")
if err != nil {
	fmt.Println("Got err from the chained functions:", err)
	return
}

fmt.Println("Got res from the chained functions:", res)
```

And if we execute this, we see:

```
Hit log middleware
Hit auth middleware
Starting trace
Made it to the base function
Ending trace
Got res from the chained functions: hello
```

It works! It might not look like much, but we've just implemented a basic version of `grpc-go`'s middleware functionality.

If you look at the [grpc-go](https://github.com/grpc/grpc-go/blob/v1.62.1/server.go#L1198) source, this is effectively what they have! 


