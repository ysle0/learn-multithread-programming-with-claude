# Concurrent Collections in C#

C# provides thread-safe collection classes in `System.Collections.Concurrent` that handle synchronization internally, making concurrent programming easier and safer.

## Table of Contents
- [Overview](#overview)
- [ConcurrentQueue](#concurrentqueue)
- [ConcurrentStack](#concurrentstack)
- [ConcurrentDictionary](#concurrentdictionary)
- [ConcurrentBag](#concurrentbag)
- [BlockingCollection](#blockingcollection)
- [Best Practices](#best-practices)

## Overview

### Why Concurrent Collections?

```csharp
// BAD: Manual locking
var queue = new Queue<int>();
var lockObj = new object();

void Enqueue(int item)
{
    lock (lockObj)
    {
        queue.Enqueue(item);
    }
}

// GOOD: Concurrent collection (lock-free)
var queue = new ConcurrentQueue<int>();

void Enqueue(int item)
{
    queue.Enqueue(item);  // Thread-safe, no lock needed
}
```

## ConcurrentQueue

### Basic Operations

```csharp
using System.Collections.Concurrent;

var queue = new ConcurrentQueue<int>();

// Enqueue items
queue.Enqueue(1);
queue.Enqueue(2);
queue.Enqueue(3);

// Try dequeue
if (queue.TryDequeue(out int result))
{
    Console.WriteLine($"Dequeued: {result}");
}

// Try peek
if (queue.TryPeek(out int peeked))
{
    Console.WriteLine($"Front: {peeked}");
}
```

### Producer-Consumer with ConcurrentQueue

```csharp
var queue = new ConcurrentQueue<int>();

// Producers
var producers = Enumerable.Range(0, 3).Select(id => Task.Run(() => {
    for (int i = 0; i < 10; i++)
    {
        queue.Enqueue(id * 100 + i);
        Console.WriteLine($"Producer {id} added {id * 100 + i}");
    }
}));

// Consumers
var consumers = Enumerable.Range(0, 2).Select(id => Task.Run(() => {
    while (!queue.IsEmpty || !Task.WhenAll(producers).IsCompleted)
    {
        if (queue.TryDequeue(out int item))
        {
            Console.WriteLine($"Consumer {id} got {item}");
        }
    }
}));

await Task.WhenAll(producers.Concat(consumers));
```

## ConcurrentDictionary

### Basic Operations

```csharp
var dict = new ConcurrentDictionary<string, int>();

// Add or update
dict.TryAdd("apple", 1);
dict.TryAdd("banana", 2);

// Get or add
int value = dict.GetOrAdd("cherry", 3);

// Add or update with factory
dict.AddOrUpdate("apple", 
    addValue: 10,  // If key doesn't exist
    updateValueFactory: (key, oldValue) => oldValue + 1);  // If exists

// Try get value
if (dict.TryGetValue("apple", out int appleCount))
{
    Console.WriteLine($"Apples: {appleCount}");
}

// Try remove
if (dict.TryRemove("banana", out int removed))
{
    Console.WriteLine($"Removed: {removed}");
}
```

### Thread-Safe Counter

```csharp
public class ConcurrentCounter
{
    private readonly ConcurrentDictionary<string, int> _counts = new();
    
    public void Increment(string key)
    {
        _counts.AddOrUpdate(key, 1, (_, count) => count + 1);
    }
    
    public int GetCount(string key)
    {
        return _counts.GetOrAdd(key, 0);
    }
    
    public Dictionary<string, int> GetAllCounts()
    {
        return new Dictionary<string, int>(_counts);
    }
}
```

## ConcurrentBag

### Unordered Collection

```csharp
var bag = new ConcurrentBag<int>();

// Add items from multiple threads
Parallel.For(0, 100, i => {
    bag.Add(i);
});

// Take items (no guaranteed order)
while (bag.TryTake(out int item))
{
    Console.WriteLine(item);
}
```

## BlockingCollection

### Producer-Consumer Pattern

```csharp
var collection = new BlockingCollection<int>(boundedCapacity: 10);

// Producer
var producer = Task.Run(() => {
    for (int i = 0; i < 100; i++)
    {
        collection.Add(i);
        Console.WriteLine($"Produced: {i}");
    }
    collection.CompleteAdding();
});

// Consumer
var consumer = Task.Run(() => {
    foreach (var item in collection.GetConsumingEnumerable())
    {
        Console.WriteLine($"Consumed: {item}");
        Thread.Sleep(50);
    }
});

await Task.WhenAll(producer, consumer);
```

### Multiple Producers and Consumers

```csharp
var collection = new BlockingCollection<string>(100);

// Multiple producers
var producers = Enumerable.Range(0, 3).Select(id => Task.Run(() => {
    for (int i = 0; i < 10; i++)
    {
        collection.Add($"Producer{id}-Item{i}");
    }
}));

// Multiple consumers
var consumers = Enumerable.Range(0, 2).Select(id => Task.Run(() => {
    foreach (var item in collection.GetConsumingEnumerable())
    {
        Console.WriteLine($"Consumer {id}: {item}");
    }
}));

await Task.WhenAll(producers);
collection.CompleteAdding();
await Task.WhenAll(consumers);
```

## Best Practices

1. **Use concurrent collections instead of lock + regular collection**
2. **Understand performance characteristics** (lock-free vs blocking)
3. **Use BlockingCollection for producer-consumer**
4. **Don't assume ordering** (except ConcurrentQueue/Stack)
5. **Prefer TryXxx methods** over exceptions

## Complete Example: Web Crawler

```csharp
using System;
using System.Collections.Concurrent;
using System.Linq;
using System.Net.Http;
using System.Threading.Tasks;

public class WebCrawler
{
    private readonly HttpClient _client = new();
    private readonly ConcurrentDictionary<string, bool> _visited = new();
    private readonly ConcurrentBag<string> _results = new();
    private readonly SemaphoreSlim _semaphore = new(10);
    
    public async Task<string[]> CrawlAsync(string[] seedUrls)
    {
        var queue = new ConcurrentQueue<string>(seedUrls);
        var tasks = new List<Task>();
        
        for (int i = 0; i < 20; i++)
        {
            tasks.Add(Task.Run(async () => {
                while (queue.TryDequeue(out string url))
                {
                    if (_visited.TryAdd(url, true))
                    {
                        await _semaphore.WaitAsync();
                        try
                        {
                            var content = await _client.GetStringAsync(url);
                            _results.Add(url);
                            
                            // Extract and queue new URLs (simplified)
                            // var newUrls = ExtractUrls(content);
                            // foreach (var newUrl in newUrls)
                            //     queue.Enqueue(newUrl);
                        }
                        finally
                        {
                            _semaphore.Release();
                        }
                    }
                }
            }));
        }
        
        await Task.WhenAll(tasks);
        return _results.ToArray();
    }
}
```

## Navigation

- [Back to C# Overview](./README.md)
- Previous: [SemaphoreSlim](./04-semaphore-slim.md)
- [Back to Language Implementations](../)
