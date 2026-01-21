# SemaphoreSlim in C#

`SemaphoreSlim` is a lightweight semaphore that limits the number of threads that can access a resource concurrently. It supports both synchronous and asynchronous waiting.

## Table of Contents
- [Basic Concepts](#basic-concepts)
- [Synchronous Usage](#synchronous-usage)
- [Asynchronous Usage](#asynchronous-usage)
- [Rate Limiting](#rate-limiting)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)

## Basic Concepts

### What is a Semaphore?

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

// Allow max 3 concurrent accesses
var semaphore = new SemaphoreSlim(3, 3);

var tasks = Enumerable.Range(0, 10).Select(i => Task.Run(async () => {
    Console.WriteLine($"Task {i} waiting...");
    
    await semaphore.WaitAsync();
    try
    {
        Console.WriteLine($"Task {i} entered");
        await Task.Delay(1000);
        Console.WriteLine($"Task {i} leaving");
    }
    finally
    {
        semaphore.Release();
    }
}));

await Task.WhenAll(tasks);
```

## Synchronous Usage

### Basic Pattern

```csharp
var semaphore = new SemaphoreSlim(initialCount: 2, maxCount: 2);

void AccessResource(int id)
{
    semaphore.Wait();  // Blocks until available
    try
    {
        Console.WriteLine($"Worker {id} accessing resource");
        Thread.Sleep(1000);
    }
    finally
    {
        semaphore.Release();
    }
}

Parallel.For(0, 10, i => AccessResource(i));
```

### With Timeout

```csharp
var semaphore = new SemaphoreSlim(1);

bool TryAccessResource(int timeoutMs)
{
    if (semaphore.Wait(timeoutMs))
    {
        try
        {
            // Access resource
            return true;
        }
        finally
        {
            semaphore.Release();
        }
    }
    
    Console.WriteLine("Timeout - couldn't access resource");
    return false;
}
```

## Asynchronous Usage

### Basic Async Pattern

```csharp
private readonly SemaphoreSlim _semaphore = new(5); // Max 5 concurrent

public async Task ProcessAsync(string item)
{
    await _semaphore.WaitAsync();
    try
    {
        await ProcessItemAsync(item);
    }
    finally
    {
        _semaphore.Release();
    }
}
```

### With Cancellation

```csharp
public async Task ProcessAsync(CancellationToken ct)
{
    await _semaphore.WaitAsync(ct);
    try
    {
        await DoWorkAsync(ct);
    }
    finally
    {
        _semaphore.Release();
    }
}
```

## Rate Limiting

### HTTP Request Throttling

```csharp
public class HttpThrottler
{
    private readonly HttpClient _client = new();
    private readonly SemaphoreSlim _semaphore = new(10); // Max 10 concurrent
    
    public async Task<string[]> FetchAllAsync(IEnumerable<string> urls)
    {
        var tasks = urls.Select(FetchAsync);
        return await Task.WhenAll(tasks);
    }
    
    private async Task<string> FetchAsync(string url)
    {
        await _semaphore.WaitAsync();
        try
        {
            return await _client.GetStringAsync(url);
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

### Database Connection Pool

```csharp
public class ConnectionPool
{
    private readonly SemaphoreSlim _semaphore;
    
    public ConnectionPool(int maxConnections)
    {
        _semaphore = new SemaphoreSlim(maxConnections);
    }
    
    public async Task<T> ExecuteAsync<T>(Func<Task<T>> query)
    {
        await _semaphore.WaitAsync();
        try
        {
            return await query();
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

## Best Practices

1. **Always Release in Finally**: Ensure `Release()` is called even on exceptions
2. **Use Async Version**: Prefer `WaitAsync()` over `Wait()` in async code
3. **Proper Disposal**: Dispose semaphore when done
4. **Match Wait/Release**: Every Wait must have corresponding Release
5. **Avoid Recursive Acquisition**: Don't wait on same semaphore in nested code

## Common Pitfalls

### Forgetting to Release

```csharp
// BAD: Exception causes lock leak
await _semaphore.WaitAsync();
await MightThrowAsync();  // If throws, never releases!
_semaphore.Release();

// GOOD: Always release in finally
await _semaphore.WaitAsync();
try
{
    await MightThrowAsync();
}
finally
{
    _semaphore.Release();
}
```

### Double Release

```csharp
// BAD: Releasing too many times
_semaphore.Release();
_semaphore.Release();  // Can exceed maxCount!

// GOOD: Track state
bool acquired = false;
try
{
    await _semaphore.WaitAsync();
    acquired = true;
    // Work
}
finally
{
    if (acquired)
        _semaphore.Release();
}
```

## Complete Example: Parallel Downloader

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;

public class ParallelDownloader
{
    private readonly HttpClient _client = new();
    private readonly SemaphoreSlim _semaphore;
    
    public ParallelDownloader(int maxConcurrent = 5)
    {
        _semaphore = new SemaphoreSlim(maxConcurrent);
    }
    
    public async Task<Dictionary<string, string>> DownloadAllAsync(
        IEnumerable<string> urls,
        IProgress<int> progress = null,
        CancellationToken ct = default)
    {
        var urlList = urls.ToList();
        var results = new Dictionary<string, string>();
        var completed = 0;
        var lockObj = new object();
        
        var tasks = urlList.Select(async url => {
            await _semaphore.WaitAsync(ct);
            try
            {
                var content = await _client.GetStringAsync(url, ct);
                
                lock (lockObj)
                {
                    results[url] = content;
                    completed++;
                    progress?.Report(completed * 100 / urlList.Count);
                }
            }
            finally
            {
                _semaphore.Release();
            }
        });
        
        await Task.WhenAll(tasks);
        return results;
    }
}

// Usage
var downloader = new ParallelDownloader(maxConcurrent: 10);
var progress = new Progress<int>(p => Console.WriteLine($"Progress: {p}%"));
var urls = new[] { "https://example.com", "https://example.org" };
var results = await downloader.DownloadAllAsync(urls, progress);
```

## Navigation

- [Back to C# Overview](./README.md)
- Previous: [Lock and Monitor](./03-lock-monitor.md)
- Next: [Concurrent Collections](./05-concurrent-collections.md)
