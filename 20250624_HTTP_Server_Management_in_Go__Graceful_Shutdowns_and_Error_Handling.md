Running a web server isn’t just about handling requests—it's about lifecycle control. You want predictable startup, clear error handling, and graceful shutdown.

```go
func main() {
    // 1. Initialize server
    srv := webapi.NewServer() // Preconfigured *http.Server

    // 2. Error channel (buffer size 1 prevents leaks)
    serverErrCh := make(chan error, 1)

    // 3. Start server in goroutine
    go func() {
        log.Printf("Starting server on http://%s ...", srv.Addr)
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            serverErrCh <- err  // Report startup errors
        }
    }()

    // 4. Shutdown signal listener
    shutdownCh := make(chan os.Signal, 1)
    signal.Notify(shutdownCh, os.Interrupt, syscall.SIGTERM) // Capture Ctrl+C/kill

    // 5. Wait for either error or shutdown signal
    select {
    case err := <-serverErrCh:
        log.Fatalf("Server error: %v", err) // Startup failed

    case <-shutdownCh:
        log.Println("Shutting down server...")
        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel() // Always release context resources!

        // 6. Attempt graceful shutdown
        if err := srv.Shutdown(ctx); err != nil {
            log.Printf("Graceful shutdown failed: %v", err)
            // 7. Force close if graceful fails
            if err = srv.Close(); err != nil {
                log.Fatalf("Forced shutdown error: %v", err)
            }
        }
        log.Println("Server exited gracefully")
    }
}
```

---

### What This Code Really Does

**1. Server Initialization**

```go
srv := webapi.NewServer()
```

This creates a preconfigured HTTP server—potentially with address, routes, and timeouts already set. It keeps the code clean and reusable.

**2. Error Channel for Server Runtime**

```go
serverErrCh := make(chan error, 1)
```

We set up a buffered error channel to listen for startup or runtime issues. The buffer of 1 ensures no goroutines get stuck if an error isn’t read immediately.

**3. Starting the Server Concurrently**

```go
go func() {
    log.Printf("Starting server on http://%s ...", srv.Addr)
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        serverErrCh <- err
    }
}()
```

By starting the server in a goroutine, we avoid blocking the main execution. It listens and serves, and if something goes wrong, we send the error to the channel—unless it's an intentional shutdown.

**4. Listening for OS Signals**

```go
shutdownCh := make(chan os.Signal, 1)
signal.Notify(shutdownCh, os.Interrupt, syscall.SIGTERM)
```

This sets up a signal listener to catch system interrupts (like `Ctrl+C` or a Docker stop event). This is key for real-world server deployments that need to shut down gracefully.

**5. Coordinating Shutdown or Errors**

```go
select {
case err := <-serverErrCh:
    log.Fatalf("Server error: %v", err)
case <-shutdownCh:
    log.Println("Shutting down server...")
    ...
}
```

We wait for either a server error or an external shutdown signal. Whichever happens first dictates the next step.

**6. Graceful Shutdown with Timeout**

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
```

This timeout context gives the server up to 5 seconds to shut down cleanly. Any ongoing requests are allowed to finish—ideal for preventing dropped traffic.

**7. Fallback to Forced Shutdown**

```go
if err := srv.Shutdown(ctx); err != nil {
    ...
    if err = srv.Close(); err != nil {
        log.Fatalf("Forced shutdown error: %v", err)
    }
}
```

If graceful shutdown fails (for example, due to a stuck connection), the server is forcefully closed to ensure the process exits and frees resources.

---

### Why This Pattern Matters

This isn’t just clean code—it’s operationally sound. It’s what you need when deploying Go services behind a load balancer, inside containers, or on bare metal:

* Startup and error visibility
* Responsiveness to OS signals
* Clean exit path with fallbacks

It’s a tested way to make your Go services production-ready—resilient, observable, and predictable.
