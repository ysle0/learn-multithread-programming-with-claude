# Thread and Task in C#

C# provides both low-level `Thread` class and high-level `Task` abstraction. Modern C# strongly favors tasks over threads for most scenarios.

## Table of Contents
- [Thread Class](#thread-class)
- [Task Parallel Library](#task-parallel-library)
- [Task vs Thread](#task-vs-thread)
- [Parallel Class](#parallel-class)
- [Comparison with Other Languages](#comparison-with-other-languages)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)

## Thread Class

### Creating Threads

```csharp
using System;
using System.Threading;

class Program
{
    static void PrintNumbers()
    {
        for (int i = 0; i < 5; i++)
        {
            Console.WriteLine($"Number: {i}");
            Thread.Sleep(100);
        }
    }

    static void Main()
    {
        // Create and start thread
        var thread = new Thread(PrintNumbers);
        thread.Start();

        // Wait for completion
        thread.Join();

        Console.WriteLine("Thread completed");
    }
}
```

### Passing Parameters

```csharp
using System;
using System.Threading;

class Program
{
    static void PrintMessage(object data)
    {
        string message = (string)data;
        Console.WriteLine(message);
    }

    static void Main()
    {
        var thread = new Thread(PrintMessage);
        thread.Start("Hello from thread!");
        thread.Join();

        // With lambda (better)
        string msg = "Hello from lambda!";
        var thread2 = new Thread(() => Console.WriteLine(msg));
        thread2.Start();
        thread2.Join();
    }
}
```

### Thread Properties

```csharp
using System;
using System.Threading;

var thread = new Thread(() => {
    Console.WriteLine($"Thread ID: {Thread.CurrentThread.ManagedThreadId}");
    Console.WriteLine($"Is Background: {Thread.CurrentThread.IsBackground}");
    Console.WriteLine($"Priority: {Thread.CurrentThread.Priority}");
});

// Set properties
thread.Name = "Worker Thread";
thread.IsBackground = true;  // Won't prevent process from exiting
thread.Priority = ThreadPriority.AboveNormal;

thread.Start();
thread.Join();
```

### Background vs. Foreground

```csharp
// Foreground thread - keeps process alive
var foregroundThread = new Thread(() => {
    Thread.Sleep(5000);
    Console.WriteLine("Foreground done");
});
foregroundThread.IsBackground = false;
foregroundThread.Start();

// Background thread - doesn't keep process alive
var backgroundThread = new Thread(() => {
    Thread.Sleep(5000);
    Console.WriteLine("Background done");  // May not print
});
backgroundThread.IsBackground = true;
backgroundThread.Start();

// Process exits when all foreground threads complete
```

## Task Parallel Library

### Basic Task Creation

```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        // Method 1: Task.Run
        var task1 = Task.Run(() => {
            Console.WriteLine("Task 1 running");
            return 42;
        });

        // Method 2: Task.Factory.StartNew (more options)
        var task2 = Task.Factory.StartNew(() => {
            Console.WriteLine("Task 2 running");
            return 100;
        });

        // Method 3: Create and start separately
        var task3 = new Task(() => {
            Console.WriteLine("Task 3 running");
        });
        task3.Start();

        // Wait for all
        await Task.WhenAll(task1, task2, task3);

        Console.WriteLine($"Results: {task1.Result}, {task2.Result}");
    }
}
```

### Task with Return Value

```csharp
using System;
using System.Threading.Tasks;

static async Task Main()
{
    Task<int> task = Task.Run(() => {
        Thread.Sleep(1000);
        return 42;
    });

    Console.WriteLine("Task started");

    // Await result
    int result = await task;
    Console.WriteLine($"Result: {result}");

    // Or use .Result (blocks, can deadlock)
    // int result = task.Result;  // Don't do this if avoidable
}
```

### Task Continuation

```csharp
using System;
using System.Threading.Tasks;

var task = Task.Run(() => {
    Thread.Sleep(1000);
    return 42;
})
.ContinueWith(prevTask => {
    int result = prevTask.Result;
    Console.WriteLine($"Previous result: {result}");
    return result * 2;
})
.ContinueWith(prevTask => {
    int result = prevTask.Result;
    Console.WriteLine($"Final result: {result}");
});

await task;
```

### Task Cancellation

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

static async Task Main()
{
    var cts = new CancellationTokenSource();

    var task = Task.Run(async () => {
        for (int i = 0; i < 100; i++)
        {
            // Check for cancellation
            cts.Token.ThrowIfCancellationRequested();

            Console.WriteLine($"Working: {i}");
            await Task.Delay(100, cts.Token);
        }
    }, cts.Token);

    // Cancel after 1 second
    await Task.Delay(1000);
    cts.Cancel();

    try
    {
        await task;
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("Task was cancelled");
    }
}
```

### Task.WhenAll and Task.WhenAny

```csharp
using System;
using System.Threading.Tasks;

static async Task Main()
{
    var task1 = Task.Delay(1000).ContinueWith(_ => "Task 1");
    var task2 = Task.Delay(2000).ContinueWith(_ => "Task 2");
    var task3 = Task.Delay(1500).ContinueWith(_ => "Task 3");

    // Wait for all tasks
    string[] results = await Task.WhenAll(task1, task2, task3);
    Console.WriteLine($"All done: {string.Join(", ", results)}");

    // Or wait for first to complete
    var firstTask = await Task.WhenAny(task1, task2, task3);
    Console.WriteLine($"First done: {firstTask.Result}");
}
```

### Exception Handling

```csharp
using System;
using System.Threading.Tasks;

static async Task Main()
{
    var task = Task.Run(() => {
        throw new InvalidOperationException("Something went wrong!");
    });

    try
    {
        await task;
    }
    catch (InvalidOperationException ex)
    {
        Console.WriteLine($"Caught: {ex.Message}");
    }

    // Multiple tasks with exceptions
    var task1 = Task.Run(() => throw new Exception("Error 1"));
    var task2 = Task.Run(() => throw new Exception("Error 2"));

    try
    {
        await Task.WhenAll(task1, task2);
    }
    catch (Exception ex)
    {
        // Only first exception caught
        Console.WriteLine($"First exception: {ex.Message}");

        // Get all exceptions from task
        if (task1.IsFaulted)
        {
            foreach (var inner in task1.Exception!.InnerExceptions)
                Console.WriteLine($"Task1 exception: {inner.Message}");
        }
    }
}
```

## Task vs Thread

### Comparison

```csharp
// Thread: Low-level, manual management
var thread = new Thread(() => {
    // Work
});
thread.Start();
thread.Join();

// Task: High-level, uses thread pool
var task = Task.Run(() => {
    // Work
});
await task;
```

### When to Use Thread

```csharp
// Use Thread when you need:
// 1. Long-running operation (not suitable for thread pool)
var longRunningThread = new Thread(() => {
    // Long-running work
});
longRunningThread.IsBackground = true;
longRunningThread.Start();

// Or use Task with LongRunning hint
var longRunningTask = Task.Factory.StartNew(() => {
    // Long-running work
}, TaskCreationOptions.LongRunning);

// 2. Specific thread configuration
var thread = new Thread(() => {
    // Work
});
thread.Priority = ThreadPriority.Highest;
thread.SetApartmentState(ApartmentState.STA);  // For COM interop
thread.Start();
```

### Thread Pool

```csharp
using System;
using System.Threading;

// Queue work to thread pool directly
ThreadPool.QueueUserWorkItem(state => {
    Console.WriteLine("Work item executing");
});

// Get thread pool info
ThreadPool.GetMinThreads(out int minWorker, out int minIO);
ThreadPool.GetMaxThreads(out int maxWorker, out int maxIO);
ThreadPool.GetAvailableThreads(out int availWorker, out int availIO);

Console.WriteLine($"Thread pool: min={minWorker}, max={maxWorker}, avail={availWorker}");
```

## Parallel Class

### Parallel.For

```csharp
using System;
using System.Threading.Tasks;

static void Main()
{
    // Sequential
    for (int i = 0; i < 100; i++)
    {
        ProcessItem(i);
    }

    // Parallel
    Parallel.For(0, 100, i => {
        ProcessItem(i);
    });

    // With options
    var options = new ParallelOptions {
        MaxDegreeOfParallelism = Environment.ProcessorCount
    };

    Parallel.For(0, 100, options, i => {
        ProcessItem(i);
    });
}

static void ProcessItem(int i)
{
    Console.WriteLine($"Processing {i} on thread {Thread.CurrentThread.ManagedThreadId}");
}
```

### Parallel.ForEach

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

var items = new List<string> { "apple", "banana", "cherry", "date" };

// Parallel foreach
Parallel.ForEach(items, item => {
    Console.WriteLine($"Processing {item}");
    // Process item
});

// With degree of parallelism
var options = new ParallelOptions {
    MaxDegreeOfParallelism = 2
};

Parallel.ForEach(items, options, item => {
    ProcessItem(item);
});
```

### Parallel.Invoke

```csharp
using System;
using System.Threading.Tasks;

// Execute methods in parallel
Parallel.Invoke(
    () => Method1(),
    () => Method2(),
    () => Method3()
);

Console.WriteLine("All methods completed");

static void Method1() => Console.WriteLine("Method 1");
static void Method2() => Console.WriteLine("Method 2");
static void Method3() => Console.WriteLine("Method 3");
```

### Breaking Parallel Loops

```csharp
using System;
using System.Threading.Tasks;

Parallel.For(0, 1000, (i, state) => {
    if (i == 100)
    {
        Console.WriteLine("Breaking at 100");
        state.Break();  // Or state.Stop()
        return;
    }

    ProcessItem(i);
});
```

## Comparison with Other Languages

### C# vs. C++
```csharp
// C#: Task-based
var task = Task.Run(() => Compute());
var result = await task;

// C++ equivalent (async):
// auto future = std::async(compute);
// auto result = future.get();
```

### C# vs. Go
```csharp
// C#: Tasks
var task = Task.Run(() => Work());
await task;

// Go equivalent (goroutines):
// go work()
// (No direct await; use channels or sync.WaitGroup)
```

### C# vs. JavaScript
```csharp
// C#: async/await (very similar!)
var result = await FetchDataAsync();

// JavaScript:
// const result = await fetchData();
```

## Best Practices

### 1. Prefer Task Over Thread

```csharp
// GOOD: Use Task
var task = Task.Run(() => Work());
await task;

// LESS GOOD: Manual thread (only when necessary)
var thread = new Thread(() => Work());
thread.Start();
thread.Join();
```

### 2. Use Async All the Way

```csharp
// GOOD: Async all the way
public async Task<string> GetDataAsync()
{
    return await httpClient.GetStringAsync(url);
}

// BAD: Blocking in async method
public async Task<string> GetDataBad()
{
    return httpClient.GetString(url);  // Blocking!
}
```

### 3. ConfigureAwait in Library Code

```csharp
// In library code
public async Task<string> GetDataAsync()
{
    return await httpClient.GetStringAsync(url)
                           .ConfigureAwait(false);
}

// In application code (UI), default is fine
public async Task Button_Click(object sender, EventArgs e)
{
    var data = await GetDataAsync();  // Resumes on UI thread
    textBox.Text = data;
}
```

### 4. Handle Cancellation

```csharp
public async Task ProcessAsync(CancellationToken ct)
{
    for (int i = 0; i < 100; i++)
    {
        ct.ThrowIfCancellationRequested();
        await DoWorkAsync(ct);
    }
}
```

### 5. Don't Block on Tasks

```csharp
// BAD: Can deadlock
public void BadMethod()
{
    var result = GetDataAsync().Result;  // Deadlock in UI/ASP.NET!
}

// GOOD: Async all the way
public async Task GoodMethod()
{
    var result = await GetDataAsync();
}

// If you MUST block (console app), use GetAwaiter().GetResult()
public void ConsoleMain()
{
    var result = GetDataAsync().GetAwaiter().GetResult();
}
```

## Common Pitfalls

### 1. Thread Pool Starvation

```csharp
// BAD: Blocking thread pool threads
Parallel.For(0, 1000, i => {
    Thread.Sleep(10000);  // Blocks thread pool thread!
});

// GOOD: Use async
await Task.WhenAll(Enumerable.Range(0, 1000).Select(async i => {
    await Task.Delay(10000);
}));
```

### 2. Not Disposing Tasks

```csharp
// Generally OK: Tasks don't need disposal
var task = Task.Run(() => Work());
await task;

// Exception: TaskCompletionSource and cancellation tokens
using var cts = new CancellationTokenSource();
var task = LongRunningWorkAsync(cts.Token);
```

### 3. Fire and Forget

```csharp
// BAD: Exceptions lost
public void BadMethod()
{
    Task.Run(() => {
        throw new Exception("Lost!");
    });
}

// GOOD: Await or handle
public async Task GoodMethod()
{
    await Task.Run(() => {
        throw new Exception("Caught!");
    });
}

// If truly fire-and-forget, at least log exceptions
public void FireAndForget()
{
    Task.Run(async () => {
        try
        {
            await DoWorkAsync();
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Background work failed");
        }
    });
}
```

### 4. Closure Capture Issues

```csharp
// BAD: Captures loop variable incorrectly (before C# 5)
for (int i = 0; i < 10; i++)
{
    Task.Run(() => Console.WriteLine(i));  // May print wrong values
}

// GOOD: Capture copy
for (int i = 0; i < 10; i++)
{
    int copy = i;
    Task.Run(() => Console.WriteLine(copy));
}

// Note: In C# 5+, foreach captures correctly
foreach (var item in items)
{
    Task.Run(() => Console.WriteLine(item));  // OK
}
```

### 5. Task.Run in ASP.NET

```csharp
// BAD: Don't use Task.Run in ASP.NET
public async Task<IActionResult> Index()
{
    var data = await Task.Run(() => GetData());  // Wastes threads!
    return View(data);
}

// GOOD: Just make GetData async
public async Task<IActionResult> Index()
{
    var data = await GetDataAsync();
    return View(data);
}
```

## Performance Considerations

### Task Overhead

```csharp
// Task overhead: ~1 μs
// Only worth it if work > 100 μs

// TOO SMALL: Overhead dominates
Parallel.For(0, 1000000, i => {
    result[i] = i * 2;  // Too simple
});

// BETTER: Larger chunks
var chunkSize = 10000;
Parallel.For(0, 1000000 / chunkSize, chunk => {
    for (int i = chunk * chunkSize; i < (chunk + 1) * chunkSize; i++)
    {
        result[i] = i * 2;
    }
});
```

### ValueTask for Hot Paths

```csharp
// Use ValueTask when result often available synchronously
public ValueTask<int> GetCachedAsync(string key)
{
    if (cache.TryGetValue(key, out int value))
        return new ValueTask<int>(value);  // No allocation

    return new ValueTask<int>(FetchFromDbAsync(key));
}
```

## Complete Example: Parallel Sum

```csharp
using System;
using System.Linq;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        int[] numbers = Enumerable.Range(1, 10_000_000).ToArray();

        // Sequential
        var sw = System.Diagnostics.Stopwatch.StartNew();
        long sum1 = numbers.Sum(x => (long)x);
        Console.WriteLine($"Sequential: {sum1} in {sw.ElapsedMilliseconds}ms");

        // Parallel with Parallel.For
        sw.Restart();
        long sum2 = 0;
        object lockObj = new object();
        Parallel.For(0, numbers.Length, () => 0L, (i, state, subtotal) => {
            return subtotal + numbers[i];
        }, subtotal => {
            lock (lockObj) sum2 += subtotal;
        });
        Console.WriteLine($"Parallel.For: {sum2} in {sw.ElapsedMilliseconds}ms");

        // Parallel with Tasks
        sw.Restart();
        int numTasks = Environment.ProcessorCount;
        var tasks = new Task<long>[numTasks];
        int chunkSize = numbers.Length / numTasks;

        for (int t = 0; t < numTasks; t++)
        {
            int start = t * chunkSize;
            int end = (t == numTasks - 1) ? numbers.Length : (t + 1) * chunkSize;

            tasks[t] = Task.Run(() => {
                long subtotal = 0;
                for (int i = start; i < end; i++)
                    subtotal += numbers[i];
                return subtotal;
            });
        }

        var results = await Task.WhenAll(tasks);
        long sum3 = results.Sum();
        Console.WriteLine($"Tasks: {sum3} in {sw.ElapsedMilliseconds}ms");

        // PLINQ (easiest!)
        sw.Restart();
        long sum4 = numbers.AsParallel().Sum(x => (long)x);
        Console.WriteLine($"PLINQ: {sum4} in {sw.ElapsedMilliseconds}ms");
    }
}
```

## Further Reading

- [Async/Await](./02-async-await.md)
- [Lock and Monitor](./03-lock-monitor.md)
- Microsoft Docs: Task Parallel Library

## Navigation

- [Back to C# Overview](./README.md)
- Next: [Async/Await](./02-async-await.md)
