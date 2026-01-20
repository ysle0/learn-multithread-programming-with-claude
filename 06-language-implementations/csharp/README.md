# C# Concurrency

C# provides a rich, high-level approach to concurrent programming with excellent async/await support, comprehensive threading libraries, and built-in concurrent collections. It balances developer productivity with performance.

## Overview

C# concurrency has evolved significantly:
- **.NET 1.0**: Basic threading with `Thread` class
- **.NET 2.0**: Thread pool, asynchronous programming model
- **.NET 4.0**: Task Parallel Library (TPL), PLINQ
- **.NET 4.5**: async/await keywords
- **.NET Core/5+**: Improved performance, Channels, ValueTask

## Core Components

### 1. [Thread and Task](./01-thread-task.md)
- `Thread` class for low-level threading
- `Task` and TPL for task-based parallelism
- Thread pool management
- Task schedulers and contexts

### 2. [Async/Await](./02-async-await.md)
- The async/await pattern
- Asynchronous programming model
- ConfigureAwait and context
- ValueTask for performance

### 3. [Lock and Monitor](./03-lock-monitor.md)
- `lock` statement for mutual exclusion
- `Monitor` class for advanced locking
- Reader-writer locks
- SpinLock for low-latency scenarios

### 4. [SemaphoreSlim](./04-semaphore-slim.md)
- Limiting concurrent access
- Asynchronous waiting
- Resource throttling
- Cancellation support

### 5. [Concurrent Collections](./05-concurrent-collections.md)
- `ConcurrentQueue<T>`
- `ConcurrentDictionary<K,V>`
- `ConcurrentBag<T>`
- `BlockingCollection<T>`

## Quick Comparison with Other Languages

| Feature | C# | Comparison |
|---------|----|-----------|
| **Async Syntax** | `async/await` | Most elegant, similar to JavaScript |
| **Threading** | Thread + Task | Higher level than C++, heavier than Go |
| **Message Passing** | Channels (System.Threading.Channels) | Similar to Go channels |
| **Collections** | Rich concurrent collections | More comprehensive than other languages |
| **Safety** | Memory-safe (GC) | Safer than C++, similar to Go/JS |

## Key Principles

### 1. Async/Await Over Threads
```csharp
// GOOD: Modern async/await
public async Task<string> FetchDataAsync()
{
    return await httpClient.GetStringAsync(url);
}

// OLD: Manual thread management
public string FetchData()
{
    string result = null;
    var thread = new Thread(() => {
        result = httpClient.GetString(url);
    });
    thread.Start();
    thread.Join();
    return result;
}
```

### 2. Task-Based Parallelism
```csharp
// Tasks abstract away thread management
var task1 = Task.Run(() => Compute1());
var task2 = Task.Run(() => Compute2());
await Task.WhenAll(task1, task2);
```

### 3. Built-in Synchronization
```csharp
// C# provides rich synchronization primitives
lock (lockObject)
{
    // Critical section
}
```

## Common Patterns

### Parallel Processing
```csharp
Parallel.For(0, 1000, i =>
{
    ProcessItem(i);
});
```

### Producer-Consumer with BlockingCollection
```csharp
var queue = new BlockingCollection<int>();

// Producer
Task.Run(() => {
    for (int i = 0; i < 100; i++)
        queue.Add(i);
    queue.CompleteAdding();
});

// Consumer
foreach (var item in queue.GetConsumingEnumerable())
{
    Console.WriteLine(item);
}
```

## Best Practices

1. **Use async/await for I/O**: Don't block threads on I/O operations
2. **Avoid async void**: Only use for event handlers
3. **ConfigureAwait(false)**: In library code to avoid context capture
4. **Use CancellationToken**: For cooperative cancellation
5. **Prefer immutability**: Reduce need for synchronization
6. **Use concurrent collections**: Over manual locking
7. **Avoid locks on public types**: Never lock on `this` or `typeof(Type)`

## Common Pitfalls

### 1. Async Void
```csharp
// BAD: Exceptions unhandled
public async void ProcessData()
{
    await Task.Delay(1000);
    throw new Exception();  // Can't catch!
}

// GOOD: Return Task
public async Task ProcessDataAsync()
{
    await Task.Delay(1000);
}
```

### 2. Deadlock with .Result
```csharp
// BAD: Deadlocks in UI/ASP.NET contexts
public void Button_Click(object sender, EventArgs e)
{
    var result = GetDataAsync().Result;  // Deadlock!
}

// GOOD: Use await
public async void Button_Click(object sender, EventArgs e)
{
    var result = await GetDataAsync();
}
```

### 3. Locking on Wrong Object
```csharp
// BAD: Locking on public object
public class BadClass
{
    public void Method()
    {
        lock (this)  // BAD: Others can lock on this!
        {
        }
    }
}

// GOOD: Private lock object
public class GoodClass
{
    private readonly object _lock = new object();

    public void Method()
    {
        lock (_lock)
        {
        }
    }
}
```

### 4. Capturing SynchronizationContext Unnecessarily
```csharp
// In library code (BAD):
await Task.Delay(1000);  // Captures context

// GOOD: Don't capture context
await Task.Delay(1000).ConfigureAwait(false);
```

### 5. Not Handling Cancellation
```csharp
// BAD: No cancellation support
public async Task ProcessAsync()
{
    await Task.Delay(10000);
}

// GOOD: Support cancellation
public async Task ProcessAsync(CancellationToken ct)
{
    await Task.Delay(10000, ct);
}
```

## Performance Considerations

### Task vs. Thread
- **Thread**: ~100 μs to create, ~2MB memory
- **Task**: Runs on thread pool, minimal overhead
- **Rule**: Use tasks unless you need dedicated thread

### ValueTask vs. Task
```csharp
// Use ValueTask when result often available synchronously
public ValueTask<int> GetCachedValue(string key)
{
    if (cache.TryGetValue(key, out int value))
        return new ValueTask<int>(value);  // No allocation

    return new ValueTask<int>(FetchFromDbAsync(key));
}
```

### Thread Pool Sizing
```csharp
// Get thread pool info
ThreadPool.GetMinThreads(out int minWorker, out int minIO);
ThreadPool.GetMaxThreads(out int maxWorker, out int maxIO);

// Adjust if needed (rarely necessary)
ThreadPool.SetMinThreads(Environment.ProcessorCount * 2, minIO);
```

## Modern C# Features

### Async Streams (C# 8.0)
```csharp
public async IAsyncEnumerable<int> GetNumbersAsync()
{
    for (int i = 0; i < 10; i++)
    {
        await Task.Delay(100);
        yield return i;
    }
}

// Consume
await foreach (var number in GetNumbersAsync())
{
    Console.WriteLine(number);
}
```

### Channels (System.Threading.Channels)
```csharp
var channel = Channel.CreateUnbounded<int>();

// Producer
await channel.Writer.WriteAsync(42);

// Consumer
while (await channel.Reader.WaitToReadAsync())
{
    if (channel.Reader.TryRead(out var item))
        Console.WriteLine(item);
}
```

### IAsyncDisposable (C# 8.0)
```csharp
public class AsyncResource : IAsyncDisposable
{
    public async ValueTask DisposeAsync()
    {
        await FlushAsync();
        // Cleanup
    }
}

await using var resource = new AsyncResource();
```

## Tools and Debugging

### Visual Studio Debugger
- Threads window
- Parallel Stacks window
- Tasks window
- Concurrency Visualizer

### Performance Profiling
- dotTrace
- PerfView
- Visual Studio Profiler
- BenchmarkDotNet for micro-benchmarks

### Diagnostic Tools
```csharp
// Detect deadlocks
ThreadPool.GetAvailableThreads(out int available, out int _);
if (available == 0)
{
    // Potential thread pool starvation
}
```

## Platform Considerations

### .NET Framework vs. .NET Core/5+
- .NET Core has better async performance
- .NET 5+ has improved thread pool
- Some APIs differ between platforms

### ASP.NET Core
- Don't use `Task.Run` in controllers
- Don't block on async code
- Use async all the way down

### WPF/WinForms
- Marshal to UI thread: `Dispatcher.Invoke` / `Control.Invoke`
- Or use async/await (captures SynchronizationContext automatically)

## Further Reading

- **C# in Depth** by Jon Skeet
- **Concurrency in C# Cookbook** by Stephen Cleary
- Microsoft Docs: https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/
- Stephen Cleary's Blog: https://blog.stephencleary.com/

## Navigation

- [Back to Language Implementations](../)
- Next Topics:
  - [Thread and Task](./01-thread-task.md)
  - [Async/Await](./02-async-await.md)
  - [Lock and Monitor](./03-lock-monitor.md)
  - [SemaphoreSlim](./04-semaphore-slim.md)
  - [Concurrent Collections](./05-concurrent-collections.md)
