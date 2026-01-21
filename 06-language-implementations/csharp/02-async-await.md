# Async/Await in C#

The async/await pattern is C#'s flagship feature for asynchronous programming, providing clean, readable code for concurrent operations without blocking threads.

## Table of Contents
- [Basic Concepts](#basic-concepts)
- [Async Methods](#async-methods)
- [Awaiting Tasks](#awaiting-tasks)
- [ConfigureAwait](#configureawait)
- [Error Handling](#error-handling)
- [Comparison with Other Languages](#comparison-with-other-languages)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)

## Basic Concepts

### What is Async/Await?

```csharp
using System;
using System.Net.Http;
using System.Threading.Tasks;

class Program
{
    // Synchronous - blocks thread
    static string FetchDataSync(string url)
    {
        using var client = new HttpClient();
        return client.GetStringAsync(url).Result;  // BLOCKS!
    }

    // Asynchronous - doesn't block thread
    static async Task<string> FetchDataAsync(string url)
    {
        using var client = new HttpClient();
        return await client.GetStringAsync(url);  // Doesn't block!
    }

    static async Task Main()
    {
        var data = await FetchDataAsync("https://api.github.com");
        Console.WriteLine($"Received {data.Length} characters");
    }
}
```

### How Async/Await Works

```csharp
public async Task<int> ComputeAsync()
{
    Console.WriteLine("1. Starting");

    await Task.Delay(1000);  // Yields control here

    Console.WriteLine("2. After delay");
    return 42;
}

// Compiler transforms this into a state machine
```

## Async Methods

### Basic Async Method

```csharp
public async Task DoWorkAsync()
{
    await Task.Delay(1000);
    Console.WriteLine("Work done");
}

// Usage
await DoWorkAsync();
```

### Async Method with Return Value

```csharp
public async Task<int> GetValueAsync()
{
    await Task.Delay(1000);
    return 42;
}

// Usage
int value = await GetValueAsync();
```

### Async Void (Event Handlers Only)

```csharp
// ONLY for event handlers
private async void Button_Click(object sender, EventArgs e)
{
    try
    {
        await ProcessAsync();
    }
    catch (Exception ex)
    {
        // Must handle exceptions here - they can't propagate!
        MessageBox.Show(ex.Message);
    }
}

// DON'T use async void elsewhere
public async void BadMethod()  // BAD!
{
    await Task.Delay(1000);
}

// GOOD: Return Task
public async Task GoodMethod()
{
    await Task.Delay(1000);
}
```

### ValueTask for Performance

```csharp
public ValueTask<int> GetCachedValueAsync(string key)
{
    // If value in cache, return synchronously (no allocation)
    if (_cache.TryGetValue(key, out int value))
    {
        return new ValueTask<int>(value);
    }

    // Otherwise, return actual async operation
    return new ValueTask<int>(FetchFromDatabaseAsync(key));
}

// Usage is same as Task
int value = await GetCachedValueAsync("key");
```

## Awaiting Tasks

### Awaiting Multiple Tasks

```csharp
public async Task ProcessMultipleAsync()
{
    // Sequential - slow
    var result1 = await FetchData1Async();
    var result2 = await FetchData2Async();
    var result3 = await FetchData3Async();

    // Parallel - fast
    var task1 = FetchData1Async();
    var task2 = FetchData2Async();
    var task3 = FetchData3Async();

    await Task.WhenAll(task1, task2, task3);

    var r1 = task1.Result;
    var r2 = task2.Result;
    var r3 = task3.Result;
}
```

### Task.WhenAll with Results

```csharp
public async Task<string[]> FetchAllAsync(string[] urls)
{
    var tasks = urls.Select(url => FetchDataAsync(url));
    return await Task.WhenAll(tasks);
}

// Usage
var results = await FetchAllAsync(new[] { "url1", "url2", "url3" });
```

### Task.WhenAny for Timeout

```csharp
public async Task<string> FetchWithTimeoutAsync(string url, TimeSpan timeout)
{
    var fetchTask = FetchDataAsync(url);
    var timeoutTask = Task.Delay(timeout);

    var completedTask = await Task.WhenAny(fetchTask, timeoutTask);

    if (completedTask == timeoutTask)
    {
        throw new TimeoutException("Request timed out");
    }

    return await fetchTask;
}
```

### Cancellation Support

```csharp
public async Task ProcessAsync(CancellationToken cancellationToken)
{
    for (int i = 0; i < 100; i++)
    {
        // Check for cancellation
        cancellationToken.ThrowIfCancellationRequested();

        await DoWorkAsync(cancellationToken);
    }
}

// Usage
var cts = new CancellationTokenSource();
var task = ProcessAsync(cts.Token);

// Cancel after 5 seconds
cts.CancelAfter(TimeSpan.FromSeconds(5));

try
{
    await task;
}
catch (OperationCanceledException)
{
    Console.WriteLine("Operation was cancelled");
}
```

## ConfigureAwait

### Understanding SynchronizationContext

```csharp
// In UI application (WPF/WinForms)
private async void Button_Click(object sender, EventArgs e)
{
    // Captures UI SynchronizationContext
    var data = await FetchDataAsync();

    // Resumes on UI thread - can update UI
    textBox.Text = data;
}
```

### ConfigureAwait(false)

```csharp
// In library code - don't capture context
public async Task<string> GetDataAsync()
{
    // Don't need to resume on original context
    var data = await httpClient.GetStringAsync(url)
                               .ConfigureAwait(false);

    // May resume on different thread - that's OK
    return ProcessData(data);
}

// In application code with UI updates
private async void Button_Click(object sender, EventArgs e)
{
    // Keep default (ConfigureAwait(true))
    var data = await GetDataAsync();

    // Back on UI thread - can update UI
    textBox.Text = data;
}
```

### When to Use ConfigureAwait(false)

```csharp
// Library code - always use ConfigureAwait(false)
public async Task<string> LibraryMethodAsync()
{
    var result = await SomeOperationAsync().ConfigureAwait(false);
    var processed = await ProcessAsync(result).ConfigureAwait(false);
    return processed;
}

// Application code with no UI updates - can use ConfigureAwait(false)
public async Task BackgroundProcessAsync()
{
    await Task.Delay(1000).ConfigureAwait(false);
    // Do work that doesn't need UI thread
}

// Application code with UI updates - don't use ConfigureAwait(false)
private async void UpdateUI()
{
    var data = await FetchAsync();  // Default is fine
    textBox.Text = data;  // Needs UI thread
}
```

## Error Handling

### Try-Catch in Async

```csharp
public async Task ProcessWithErrorHandlingAsync()
{
    try
    {
        await RiskyOperationAsync();
    }
    catch (HttpRequestException ex)
    {
        Console.WriteLine($"Network error: {ex.Message}");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Unexpected error: {ex.Message}");
    }
    finally
    {
        Console.WriteLine("Cleanup");
    }
}
```

### Multiple Task Exceptions

```csharp
public async Task ProcessMultipleWithErrorsAsync()
{
    var task1 = Task.Run(() => throw new InvalidOperationException("Error 1"));
    var task2 = Task.Run(() => throw new ArgumentException("Error 2"));
    var task3 = Task.Run(() => 42);

    try
    {
        await Task.WhenAll(task1, task2, task3);
    }
    catch (Exception ex)
    {
        // Only gets first exception
        Console.WriteLine($"First exception: {ex.Message}");
    }

    // To get all exceptions
    try
    {
        await Task.WhenAll(task1, task2, task3);
    }
    catch
    {
        if (task1.IsFaulted)
            Console.WriteLine($"Task1: {task1.Exception?.InnerException?.Message}");
        if (task2.IsFaulted)
            Console.WriteLine($"Task2: {task2.Exception?.InnerException?.Message}");
    }
}
```

### Using Statement in Async

```csharp
public async Task ProcessFileAsync(string path)
{
    using var stream = File.OpenRead(path);
    var content = await ReadStreamAsync(stream);
    return content;
}

// Async disposal (C# 8.0+)
public async Task ProcessResourceAsync()
{
    await using var resource = new AsyncDisposableResource();
    await resource.ProcessAsync();
}

class AsyncDisposableResource : IAsyncDisposable
{
    public async ValueTask DisposeAsync()
    {
        await FlushAsync();
        // Cleanup
    }

    public async Task FlushAsync()
    {
        await Task.Delay(100);
    }

    public async Task ProcessAsync()
    {
        await Task.Delay(1000);
    }
}
```

## Comparison with Other Languages

### C# vs. JavaScript

```csharp
// C# async/await (very similar!)
public async Task<string> FetchDataAsync(string url)
{
    var response = await httpClient.GetAsync(url);
    return await response.Content.ReadAsStringAsync();
}

// JavaScript equivalent:
// async function fetchData(url) {
//     const response = await fetch(url);
//     return await response.text();
// }
```

### C# vs. Python

```csharp
// C#
public async Task<string> FetchAsync(string url)
{
    return await httpClient.GetStringAsync(url);
}

// Python equivalent:
// async def fetch(url):
//     async with aiohttp.ClientSession() as session:
//         async with session.get(url) as response:
//             return await response.text()
```

### C# vs. C++

```csharp
// C#: Built-in async/await
var result = await ComputeAsync();

// C++: Coroutines (C++20, more complex)
// co_await compute();
```

## Best Practices

### 1. Async All the Way

```csharp
// GOOD: Async all the way down
public async Task<string> GetDataAsync()
{
    return await FetchAsync();
}

// BAD: Mixing sync and async
public string GetData()
{
    return FetchAsync().Result;  // Can deadlock!
}
```

### 2. Avoid Async Void

```csharp
// GOOD: Return Task
public async Task ProcessAsync()
{
    await DoWorkAsync();
}

// BAD: Async void (except event handlers)
public async void ProcessBad()
{
    await DoWorkAsync();
}
```

### 3. Use CancellationToken

```csharp
// GOOD: Support cancellation
public async Task ProcessAsync(CancellationToken ct)
{
    await DoWorkAsync(ct);
}

// Usage
var cts = new CancellationTokenSource();
await ProcessAsync(cts.Token);
```

### 4. ConfigureAwait in Libraries

```csharp
// In library code
public async Task<string> LibraryMethodAsync()
{
    return await FetchAsync().ConfigureAwait(false);
}

// In application code (context needed)
private async void Button_Click(object sender, EventArgs e)
{
    var data = await LibraryMethodAsync();  // Default is fine
    textBox.Text = data;
}
```

### 5. Prefer Task.WhenAll for Parallelism

```csharp
// GOOD: Parallel execution
var task1 = FetchAsync(url1);
var task2 = FetchAsync(url2);
await Task.WhenAll(task1, task2);

// BAD: Sequential execution
var data1 = await FetchAsync(url1);
var data2 = await FetchAsync(url2);
```

## Common Pitfalls

### 1. Deadlock with .Result or .Wait()

```csharp
// DEADLOCK in UI/ASP.NET!
public void BadMethod()
{
    var result = GetDataAsync().Result;  // Blocks
}

public async Task<string> GetDataAsync()
{
    return await FetchAsync();  // Tries to resume on blocked context
}

// GOOD: Use async all the way
public async Task GoodMethod()
{
    var result = await GetDataAsync();
}
```

### 2. Async Void Exceptions

```csharp
// BAD: Exception can't be caught
public async void BadAsync()
{
    await Task.Delay(100);
    throw new Exception("Lost!");
}

// Caller can't catch this!
try
{
    BadAsync();
}
catch (Exception)
{
    // Never caught!
}

// GOOD: Return Task
public async Task GoodAsync()
{
    await Task.Delay(100);
    throw new Exception("Can be caught");
}

try
{
    await GoodAsync();
}
catch (Exception ex)
{
    Console.WriteLine($"Caught: {ex.Message}");
}
```

### 3. Fire and Forget

```csharp
// BAD: Exceptions lost
public void StartBackground()
{
    Task.Run(async () => {
        await DoWorkAsync();
        throw new Exception("Lost!");
    });
}

// GOOD: Await or handle exceptions
public async Task StartBackgroundGood()
{
    try
    {
        await Task.Run(async () => {
            await DoWorkAsync();
        });
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Background work failed");
    }
}
```

### 4. Blocking in Async Method

```csharp
// BAD: Blocking in async method
public async Task<string> BadAsync()
{
    Thread.Sleep(1000);  // Blocks thread!
    return "result";
}

// GOOD: Use async APIs
public async Task<string> GoodAsync()
{
    await Task.Delay(1000);  // Doesn't block
    return "result";
}
```

### 5. Not Using CancellationToken

```csharp
// BAD: No way to cancel
public async Task LongProcessAsync()
{
    for (int i = 0; i < 1000; i++)
    {
        await DoWorkAsync();
    }
}

// GOOD: Support cancellation
public async Task LongProcessAsync(CancellationToken ct)
{
    for (int i = 0; i < 1000; i++)
    {
        ct.ThrowIfCancellationRequested();
        await DoWorkAsync(ct);
    }
}
```

## Advanced Patterns

### Async Streams (C# 8.0+)

```csharp
public async IAsyncEnumerable<int> GenerateNumbersAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < 100; i++)
    {
        ct.ThrowIfCancellationRequested();
        await Task.Delay(100, ct);
        yield return i;
    }
}

// Consume
await foreach (var number in GenerateNumbersAsync())
{
    Console.WriteLine(number);
}
```

### Async Lazy Initialization

```csharp
public class AsyncLazy<T>
{
    private readonly Lazy<Task<T>> _instance;

    public AsyncLazy(Func<Task<T>> factory)
    {
        _instance = new Lazy<Task<T>>(() => Task.Run(factory));
    }

    public Task<T> Value => _instance.Value;
}

// Usage
private readonly AsyncLazy<string> _data = new AsyncLazy<string>(LoadDataAsync);

public async Task UseDataAsync()
{
    var data = await _data.Value;
}
```

### Progress Reporting

```csharp
public async Task ProcessWithProgressAsync(IProgress<int> progress)
{
    for (int i = 0; i < 100; i++)
    {
        await DoWorkAsync();
        progress?.Report(i + 1);
    }
}

// Usage
var progress = new Progress<int>(percent => {
    Console.WriteLine($"Progress: {percent}%");
});

await ProcessWithProgressAsync(progress);
```

## Performance Considerations

### ValueTask vs Task

```csharp
// Use Task for most cases
public async Task<int> GetValueAsync()
{
    await SomeOperationAsync();
    return 42;
}

// Use ValueTask when often completing synchronously
public ValueTask<int> GetCachedAsync(string key)
{
    if (_cache.TryGetValue(key, out int value))
        return new ValueTask<int>(value);  // No allocation

    return new ValueTask<int>(FetchAsync(key));
}
```

### Avoid Unnecessary Async

```csharp
// BAD: Unnecessary async/await
public async Task<int> GetValueAsync()
{
    return await Task.FromResult(42);
}

// GOOD: Just return the task
public Task<int> GetValueAsync()
{
    return Task.FromResult(42);
}

// Or even better if value is known
public ValueTask<int> GetValueAsync()
{
    return new ValueTask<int>(42);
}
```

## Complete Example: Async Web Scraper

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;

public class WebScraper
{
    private readonly HttpClient _client = new();
    private readonly SemaphoreSlim _semaphore = new(5); // Max 5 concurrent

    public async Task<Dictionary<string, int>> FetchAllAsync(
        IEnumerable<string> urls,
        CancellationToken ct = default)
    {
        var tasks = urls.Select(url => FetchWithSemaphoreAsync(url, ct));
        var results = await Task.WhenAll(tasks);

        return results
            .Where(r => r.success)
            .ToDictionary(r => r.url, r => r.length);
    }

    private async Task<(string url, int length, bool success)> FetchWithSemaphoreAsync(
        string url,
        CancellationToken ct)
    {
        await _semaphore.WaitAsync(ct);
        try
        {
            var content = await _client.GetStringAsync(url, ct);
            return (url, content.Length, true);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Failed to fetch {url}: {ex.Message}");
            return (url, 0, false);
        }
        finally
        {
            _semaphore.Release();
        }
    }
}

// Usage
var scraper = new WebScraper();
var urls = new[] { "https://example.com", "https://example.org" };
var results = await scraper.FetchAllAsync(urls);

foreach (var (url, length) in results)
{
    Console.WriteLine($"{url}: {length} bytes");
}
```

## Further Reading

- [Thread and Task](./01-thread-task.md)
- [SemaphoreSlim](./04-semaphore-slim.md)
- Stephen Cleary's Blog: https://blog.stephencleary.com/

## Navigation

- [Back to C# Overview](./README.md)
- Previous: [Thread and Task](./01-thread-task.md)
- Next: [Lock and Monitor](./03-lock-monitor.md)
