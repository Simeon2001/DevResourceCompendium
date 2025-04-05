# The Complete Guide to Go Goroutines

## Table of Contents
1. [Introduction to Goroutines](#introduction-to-goroutines)
2. [Understanding Concurrency vs. Parallelism](#understanding-concurrency-vs-parallelism)
3. [Basic Goroutine Usage](#basic-goroutine-usage)
4. [Synchronization Techniques](#synchronization-techniques)
   - [WaitGroups](#waitgroups)
   - [Channels](#channels)
   - [Mutexes](#mutexes)
5. [Managing Multiple Individual Goroutines](#managing-multiple-individual-goroutines)
6. [Long-Running Goroutines](#long-running-goroutines)
7. [Common Patterns and Best Practices](#common-patterns-and-best-practices)
8. [Pitfalls and How to Avoid Them](#pitfalls-and-how-to-avoid-them)
9. [Advanced Goroutine Patterns](#advanced-goroutine-patterns)
10. [Practical Examples](#practical-examples)
11. [Further Resources](#further-resources)

## Introduction to Goroutines

Goroutines are one of Go's most distinctive features. They are lightweight threads managed by the Go runtime, not operating system threads. This gives them several advantages:

- **Lightweight**: You can create thousands of goroutines without significant overhead
- **Low cost**: Starting a goroutine requires very little memory (around 2KB of stack space)
- **Managed by Go**: The Go runtime handles scheduling goroutines onto OS threads

Think of goroutines as independent paths of execution that can run concurrently with other goroutines. They allow you to write concurrent programs more easily than traditional threading models.

## Understanding Concurrency vs. Parallelism

Before diving deeper, it's important to understand the difference between concurrency and parallelism:

- **Concurrency** is about dealing with multiple things at once. It's a way to structure your program to handle multiple tasks that may start, run, and complete in overlapping time periods.
- **Parallelism** is about doing multiple things at the exact same time, which requires multiple CPUs or cores.

Go's concurrency model using goroutines is designed to make concurrent programming easier, and it may or may not involve parallelism, depending on the number of available CPU cores and how the Go scheduler works.

## Basic Goroutine Usage

Starting a goroutine is very simple - just use the `go` keyword before a function call:

```go
func main() {
    // Run sayHello as a goroutine
    go sayHello()
    
    // Main continues executing
    fmt.Println("Hello from main")
    
    // Wait to see the goroutine output 
    time.Sleep(100 * time.Millisecond)
}

func sayHello() {
    fmt.Println("Hello from goroutine")
}
```

However, using `time.Sleep()` to wait for goroutines is unreliable and not recommended for production code. Instead, we should use proper synchronization techniques.

## Synchronization Techniques

### WaitGroups

A `sync.WaitGroup` is used to wait for a collection of goroutines to finish. It's like a counter that can be incremented before starting goroutines and decremented as they complete.

```go
func main() {
    var wg sync.WaitGroup
    
    // Launch 5 goroutines
    for i := 1; i <= 5; i++ {
        wg.Add(1) // Increment counter BEFORE launching goroutine
        go worker(i, &wg)
    }
    
    // Wait for all goroutines to finish
    wg.Wait()
    fmt.Println("All workers done")
}

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done() // Decrement counter when goroutine completes
    fmt.Printf("Worker %d starting\n", id)
    time.Sleep(time.Second)
    fmt.Printf("Worker %d done\n", id)
}
```

Key points about WaitGroups:
- Call `wg.Add(n)` in the main goroutine before launching worker goroutines
- Pass the WaitGroup to goroutines by pointer
- Each goroutine calls `wg.Done()` when it completes (often using defer)
- The main goroutine calls `wg.Wait()` to block until all workers are done

### Channels

Channels are typed conduits through which you can send and receive values. They provide a way for goroutines to communicate with each other and synchronize their execution.

```go
func main() {
    // Create a channel of integers
    ch := make(chan int)
    
    // Start goroutine that sends data
    go func() {
        for i := 1; i <= 5; i++ {
            ch <- i // Send i to channel
            time.Sleep(time.Millisecond * 500)
        }
        close(ch) // Close channel when done sending
    }()
    
    // Receive from channel until it's closed
    for num := range ch {
        fmt.Println("Received:", num)
    }
    
    fmt.Println("Channel closed, program exits")
}
```

Key points about channels:
- Created with `make(chan Type)`
- Send values with `ch <- value`
- Receive values with `value := <-ch`
- Use `close(ch)` when done sending
- Range over a channel to receive until closed
- Unbuffered channels block until both sender and receiver are ready
- Buffered channels (`make(chan Type, size)`) block only when buffer is full

### Mutexes

When multiple goroutines need to access shared data, you need to protect it with a mutex to prevent race conditions:

```go
func main() {
    var counter int
    var mu sync.Mutex
    var wg sync.WaitGroup
    
    // Launch 1000 goroutines
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            
            mu.Lock() // Lock before accessing shared data
            counter++
            mu.Unlock() // Unlock when done
        }()
    }
    
    wg.Wait()
    fmt.Println("Final counter value:", counter)
}
```

Key points about mutexes:
- Use `sync.Mutex` to protect shared data
- Call `mu.Lock()` before accessing shared data
- Call `mu.Unlock()` when done (often with defer)
- Only one goroutine can hold the lock at a time
- For read-heavy workloads, consider using `sync.RWMutex`

## Managing Multiple Individual Goroutines

When you have multiple distinct goroutines (not in a loop), you can handle WaitGroups in two ways:

### Method 1: Add before each goroutine

```go
func main() {
    var wg sync.WaitGroup
    
    // For each distinct goroutine, add to the WaitGroup before launching
    wg.Add(1)
    go fetchData(&wg)
    
    wg.Add(1)
    go processImages(&wg)
    
    wg.Add(1)
    go updateDatabase(&wg)
    
    // Wait for all goroutines to finish
    wg.Wait()
    fmt.Println("All tasks completed")
}

func fetchData(wg *sync.WaitGroup) {
    defer wg.Done()
    // Fetch data logic...
    fmt.Println("Data fetched")
}

func processImages(wg *sync.WaitGroup) {
    defer wg.Done()
    // Process images logic...
    fmt.Println("Images processed")
}

func updateDatabase(wg *sync.WaitGroup) {
    defer wg.Done()
    // Update database logic...
    fmt.Println("Database updated")
}
```

### Method 2: Add all at once

```go
func main() {
    var wg sync.WaitGroup
    
    // Add the total number of goroutines you'll launch
    wg.Add(3)
    
    go fetchData(&wg)
    go processImages(&wg)
    go updateDatabase(&wg)
    
    // Wait for all goroutines to finish
    wg.Wait()
    fmt.Println("All tasks completed")
}
```

Either approach works well. The key is to add to the WaitGroup in the main goroutine before launching the worker goroutines.

## Long-Running Goroutines

For goroutines that run continuously (with infinite loops), a different approach is needed since they never naturally complete. In these cases, we need a graceful shutdown mechanism.

### Managing Continuously Running Goroutines

For these scenarios, we use a combination of context cancellation and channels:

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "sync"
    "syscall"
    "time"
)

func main() {
    // Create a context that can be canceled
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel() // Ensure it's called to release resources
    
    var wg sync.WaitGroup
    
    // Start continuously running goroutines
    wg.Add(1)
    go dataMonitor(ctx, &wg)
    
    wg.Add(1)
    go serviceHealthChecker(ctx, &wg)
    
    wg.Add(1)
    go messageProcessor(ctx, &wg)
    
    // Setup signal handling for graceful shutdown
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
    
    // Wait for termination signal
    <-sigChan
    fmt.Println("\nShutdown signal received, gracefully shutting down...")
    
    // Cancel context to notify all goroutines they should stop
    cancel()
    
    // Wait with timeout for all goroutines to finish cleanup
    waitCh := make(chan struct{})
    go func() {
        wg.Wait()
        close(waitCh)
    }()
    
    // Wait for either all goroutines to finish or timeout
    select {
    case <-waitCh:
        fmt.Println("All workers completed gracefully")
    case <-time.After(5 * time.Second):
        fmt.Println("Timeout waiting for workers, forcing shutdown")
    }
    
    fmt.Println("Application shutdown complete")
}

func dataMonitor(ctx context.Context, wg *sync.WaitGroup) {
    defer wg.Done()
    
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            // Context was canceled, time to clean up and exit
            fmt.Println("Data monitor shutting down...")
            // Do any cleanup needed
            return
        case <-ticker.C:
            // Do regular work
            fmt.Println("Monitoring data...")
        }
    }
}

func serviceHealthChecker(ctx context.Context, wg *sync.WaitGroup) {
    defer wg.Done()
    
    ticker := time.NewTicker(2 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            fmt.Println("Health checker shutting down...")
            return
        case <-ticker.C:
            fmt.Println("Checking service health...")
        }
    }
}

func messageProcessor(ctx context.Context, wg *sync.WaitGroup) {
    defer wg.Done()
    
    ticker := time.NewTicker(1500 * time.Millisecond)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            fmt.Println("Message processor shutting down...")
            return
        case <-ticker.C:
            fmt.Println("Processing messages...")
        }
    }
}
```

This approach provides several important benefits:

1. **Context Cancellation**: The context provides a standard way to signal that all operations should stop.

2. **Graceful Shutdown**: Each goroutine can perform cleanup operations when it detects that the context has been canceled.

3. **Signal Handling**: The application properly responds to termination signals (like Ctrl+C).

4. **Timeout Mechanism**: If some goroutines get stuck during shutdown, the application will exit after a timeout rather than hanging indefinitely.

The key pattern is the `select` statement inside each goroutine's infinite loop, which allows the goroutine to:
- Do its normal work when the ticker sends a signal
- Detect when the context has been canceled and exit gracefully

### When to Use This Pattern

You should use this pattern when:
- Your goroutines are intended to run for the entire lifetime of the application
- You need a way to gracefully shut down services and free resources
- Your application needs to handle OS signals properly
- You're building a daemon, service, or long-running application

## Common Patterns and Best Practices

### Worker Pools

A worker pool is a collection of goroutines that process tasks from a shared channel:

```go
func main() {
    const numJobs = 100
    const numWorkers = 5
    
    jobs := make(chan int, numJobs)
    results := make(chan int, numJobs)
    
    // Start workers
    for w := 1; w <= numWorkers; w++ {
        go worker(w, jobs, results)
    }
    
    // Send jobs
    for j := 1; j <= numJobs; j++ {
        jobs <- j
    }
    close(jobs)
    
    // Collect results
    for a := 1; a <= numJobs; a++ {
        <-results
    }
}

func worker(id int, jobs <-chan int, results chan<- int) {
    for j := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, j)
        time.Sleep(time.Millisecond * 100) // Simulate work
        results <- j * 2
    }
}
```

### Fan-out, Fan-in

This pattern involves "fanning out" work to multiple goroutines, then "fanning in" the results:

```go
func main() {
    input := make(chan int, 100)
    
    // Fan out to 3 workers
    c1 := squarer(input)
    c2 := squarer(input)
    c3 := squarer(input)
    
    // Fan in the results
    merged := merge(c1, c2, c3)
    
    // Send input
    for i := 0; i < 10; i++ {
        input <- i
    }
    close(input)
    
    // Read results
    for result := range merged {
        fmt.Println(result)
    }
}

func squarer(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

func merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)
    
    // Start an output goroutine for each input channel
    output := func(c <-chan int) {
        defer wg.Done()
        for n := range c {
            out <- n
        }
    }
    
    wg.Add(len(cs))
    for _, c := range cs {
        go output(c)
    }
    
    // Start a goroutine to close out once all the output goroutines are done
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}
```

### Context for Cancellation

Use the `context` package to handle timeouts and cancellation:

```go
func main() {
    // Create a context with a timeout
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel() // Always call cancel to release resources
    
    // Start a worker that respects the context
    go worker(ctx)
    
    // Wait for the worker to finish or timeout
    select {
    case <-ctx.Done():
        fmt.Println("Main:", ctx.Err())
    case <-time.After(3 * time.Second):
        fmt.Println("Worker completed successfully")
    }
}

func worker(ctx context.Context) {
    // Simulate a slow operation
    select {
    case <-time.After(3 * time.Second):
        fmt.Println("Worker: completed work")
    case <-ctx.Done():
        fmt.Println("Worker:", ctx.Err())
    }
}
```

## Pitfalls and How to Avoid Them

### Goroutine Leaks

Goroutines that are never terminated can lead to memory leaks:

```go
// BAD: This goroutine may never terminate if the channel is never closed
go func() {
    for {
        data, ok := <-ch
        if !ok {
            return // Only exit if channel is closed
        }
        process(data)
    }
}()
```

**Solution**: Always ensure goroutines can terminate, using context, timeouts, or done channels:

```go
// GOOD: This goroutine will terminate when the context is canceled
go func(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        case data, ok := <-ch:
            if !ok {
                return
            }
            process(data)
        }
    }
}(ctx)
```

### Race Conditions

Race conditions occur when multiple goroutines access shared data without proper synchronization:

```go
// BAD: Race condition
counter := 0
go func() { counter++ }()
go func() { counter++ }()
```

**Solution**: Use proper synchronization with mutexes, channels, or atomic operations:

```go
// GOOD: Using mutex
var mu sync.Mutex
counter := 0
go func() { mu.Lock(); counter++; mu.Unlock() }()
go func() { mu.Lock(); counter++; mu.Unlock() }()

// OR using atomic operations
var counter int32 = 0
go func() { atomic.AddInt32(&counter, 1) }()
go func() { atomic.AddInt32(&counter, 1) }()
```

### Incorrect WaitGroup Usage

One common mistake is calling `wg.Add()` inside a goroutine:

```go
// BAD: wg.Add inside goroutine
for i := 0; i < 10; i++ {
    go func() {
        wg.Add(1) // WRONG: May not be executed before wg.Wait()
        defer wg.Done()
        // work
    }()
}
wg.Wait() // May return immediately if no Add has executed
```

**Solution**: Always call `wg.Add()` in the parent goroutine before launching workers:

```go
// GOOD: wg.Add before launching goroutine
for i := 0; i < 10; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        // work
    }()
}
wg.Wait()
```

### Capturing Loop Variables

A classic gotcha in Go is capturing loop variables in goroutines:

```go
// BAD: All goroutines see the same value of i
for i := 0; i < 10; i++ {
    go func() {
        fmt.Println(i) // All might print the same value (10)
    }()
}
```

**Solution**: Pass the loop variable as a parameter to the goroutine:

```go
// GOOD: Each goroutine gets its own copy of i
for i := 0; i < 10; i++ {
    go func(val int) {
        fmt.Println(val) // Prints 0 to 9
    }(i)
}

// OR using a local variable
for i := 0; i < 10; i++ {
    i := i // Create a new i in this scope
    go func() {
        fmt.Println(i) // Prints 0 to 9
    }()
}
```

## Advanced Goroutine Patterns

### Rate Limiting

Control how quickly goroutines process work with a rate limiter:

```go
func main() {
    // Create a rate limiter that allows 2 operations per second
    limiter := time.Tick(500 * time.Millisecond)
    
    for i := 1; i <= 10; i++ {
        <-limiter // Wait for permission from rate limiter
        go func(id int) {
            fmt.Printf("Worker %d starting\n", id)
            // Do work
        }(i)
    }
    
    // Wait to see the output
    time.Sleep(6 * time.Second)
}
```

### Bounded Parallelism

Limit the number of active goroutines using a semaphore pattern:

```go
func main() {
    var wg sync.WaitGroup
    
    // Create a channel to limit concurrent goroutines
    semaphore := make(chan struct{}, 3) // Allow max 3 concurrent operations
    
    for i := 1; i <= 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            
            semaphore <- struct{}{} // Acquire a token
            defer func() { <-semaphore }() // Release the token
            
            fmt.Printf("Worker %d starting\n", id)
            time.Sleep(time.Second) // Simulate work
            fmt.Printf("Worker %d done\n", id)
        }(i)
    }
    
    wg.Wait()
}
```

### Error Handling

Propagate errors from goroutines using error channels:

```go
func main() {
    jobs := make(chan int, 10)
    results := make(chan result, 10)
    
    // Start workers
    var wg sync.WaitGroup
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go worker(i, jobs, results, &wg)
    }
    
    // Send jobs
    for j := 0; j < 10; j++ {
        jobs <- j
    }
    close(jobs)
    
    // Wait for all workers to finish
    go func() {
        wg.Wait()
        close(results)
    }()
    
    // Process results and errors
    for r := range results {
        if r.err != nil {
            fmt.Printf("Error: %v\n", r.err)
        } else {
            fmt.Printf("Result: %d\n", r.value)
        }
    }
}

type result struct {
    value int
    err   error
}

func worker(id int, jobs <-chan int, results chan<- result, wg *sync.WaitGroup) {
    defer wg.Done()
    
    for j := range jobs {
        // Simulate some errors
        if j%3 == 0 {
            results <- result{err: fmt.Errorf("error processing job %d", j)}
            continue
        }
        
        // Process job
        time.Sleep(time.Millisecond * 100)
        results <- result{value: j * j}
    }
}
```

## Practical Examples

### Web Scraper

```go
func main() {
    urls := []string{
        "https://golang.org",
        "https://github.com",
        "https://stackoverflow.com",
        // Add more URLs
    }
    
    results := make(chan string)
    var wg sync.WaitGroup
    
    // Set up a semaphore to limit concurrent requests
    semaphore := make(chan struct{}, 3)
    
    for _, url := range urls {
        wg.Add(1)
        go func(url string) {
            defer wg.Done()
            
            semaphore <- struct{}{} // Acquire token
            defer func() { <-semaphore }() // Release token
            
            // Fetch the URL
            resp, err := http.Get(url)
            if err != nil {
                results <- fmt.Sprintf("Error fetching %s: %v", url, err)
                return
            }
            defer resp.Body.Close()
            
            // Read the response body
            body, err := io.ReadAll(resp.Body)
            if err != nil {
                results <- fmt.Sprintf("Error reading %s: %v", url, err)
                return
            }
            
            results <- fmt.Sprintf("Fetched %s: %d bytes", url, len(body))
        }(url)
    }
    
    // Close results channel when all goroutines are done
    go func() {
        wg.Wait()
        close(results)
    }()
    
    // Process results
    for result := range results {
        fmt.Println(result)
    }
}
```

### Long-Running Service Example

Here's a more complete example of a service that runs continuously and handles graceful shutdown:

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "os"
    "os/signal"
    "sync"
    "syscall"
    "time"
)

func main() {
    // Create a cancellable context
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Create wait group to track all goroutines
    var wg sync.WaitGroup

    // Shared data that needs protection
    var (
        stats    = make(map[string]int)
        statsMux sync.RWMutex
    )

    // Start web server
    server := &http.Server{
        Addr: ":8080",
        Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            statsMux.Lock()
            stats[r.URL.Path]++
            statsMux.Unlock()
            fmt.Fprintf(w, "Hello, you've requested: %s\n", r.URL.Path)
        }),
    }

    // Start HTTP server in a goroutine
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println("Starting HTTP server on :8080")
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("HTTP server error: %v", err)
        }
        fmt.Println("HTTP server stopped")
    }()

    // Start stats reporter goroutine
    wg.Add(1)
    go func() {
        defer wg.Done()
        ticker := time.NewTicker(5 * time.Second)
        defer ticker.Stop()

        for {
            select {
            case <-ctx.Done():
                fmt.Println("Stats reporter shutting down...")
                return
            case <-ticker.C:
                statsMux.RLock()
                fmt.Println("--- Current Stats ---")
                for path, count := range stats {
                    fmt.Printf("%s: %d requests\n", path, count)
                }
                statsMux.RUnlock()
            }
        }
    }()

    // Start background processor goroutine
    wg.Add(1)
    go func() {
        defer wg.Done()
        ticker := time.NewTicker(2 * time.Second)
        defer ticker.Stop()

        for {
            select {
            case <-ctx.Done():
                fmt.Println("Background processor shutting down...")
                return
            case <-ticker.C:
                fmt.Println("Performing background processing...")
                time.Sleep(500 * time.Millisecond)
                fmt.Println("Background processing complete")
            }
        }
    }()

    // Set up signal handling
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

    // Wait for termination signal
    sig := <-sigChan
    fmt.Printf("\nReceived signal: %v, initiating graceful shutdown...\n", sig)

    // Start the shutdown process
    cancel() // Signal all goroutines to stop via context

    // Create a timeout context for server shutdown
    shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer shutdownCancel()

    // Shutdown the HTTP server gracefully
    if err := server.Shutdown(shutdownCtx); err != nil {
        fmt.Printf("HTTP server shutdown error: %v\n", err)
    }

    // Wait for goroutines to finish with timeout
    waitCh := make(chan struct{})
    go func() {
        wg.Wait()
        close(waitCh)
    }()

    // Wait with timeout for all goroutines to finish
    select {
    case <-waitCh:
        fmt.Println("All goroutines have completed gracefully")
    case <-time.After(15 * time.Second):
        fmt.Println("Timeout waiting for some goroutines, forcing exit")
    }

    fmt.Println("Service has shut down gracefully")
}
```

This example demonstrates:
1. Multiple continuous goroutines for different service components
2. Shared data protection with mutexes
3. Proper signal handling
4. Graceful HTTP server shutdown
5. Timeout mechanism for cleanup operations
6. Coordinated termination of all components

## Further Resources

1. [Go Concurrency Patterns](https://blog.golang.org/pipelines) - Official Go blog post
2. [Concurrency is not Parallelism](https://blog.golang.org/waza-talk) - Talk by Rob Pike
3. [Advanced Go Concurrency Patterns](https://blog.golang.org/advanced-go-concurrency-patterns) - Google I/O talk
4. [Go by Example: Goroutines](https://gobyexample.com/goroutines) - Practical examples
5. [The Go Memory Model](https://golang.org/ref/mem) - Official documentation
6. [Context Package](https://golang.org/pkg/context/) - Official documentation for context
7. [Concurrency in Go](https://www.oreilly.com/library/view/concurrency-in-go/9781491941294/) - Book by Katherine Cox-Buday

Remember, concurrent programming is powerful but comes with its own set of challenges. Start simple, thoroughly test your code (including with the race detector `-race` flag), and gradually build up your understanding of these patterns.
