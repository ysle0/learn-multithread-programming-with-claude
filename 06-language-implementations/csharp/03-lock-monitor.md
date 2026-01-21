# Lock and Monitor in C#

C# provides the `lock` statement and `Monitor` class for mutual exclusion, protecting shared data from concurrent access.

## Table of Contents
- [Lock Statement](#lock-statement)
- [Monitor Class](#monitor-class)
- [Reader-Writer Locks](#reader-writer-locks)
- [Comparison with Other Languages](#comparison-with-other-languages)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)

## Lock Statement

### Basic Lock Usage

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

public class Counter
{
    private readonly object _lock = new object();
    private int _count = 0;

    public void Increment()
    {
        lock (_lock)
        {
            _count++;
        }
    }

    public int GetCount()
    {
        lock (_lock)
        {
            return _count;
        }
    }
}

// Usage
var counter = new Counter();
var tasks = Enumerable.Range(0, 10).Select(_ => 
    Task.Run(() => {
        for (int i = 0; i < 1000; i++)
            counter.Increment();
    })
);

await Task.WhenAll(tasks);
Console.WriteLine($"Count: {counter.GetCount()}");  // Always 10000
```

### Lock Object Guidelines

```csharp
public class GoodLocking
{
    // GOOD: Private lock object
    private readonly object _lock = new object();
    
    // GOOD: Multiple locks for different data
    private readonly object _lock1 = new object();
    private readonly object _lock2 = new object();
    
    private int _data1;
    private int _data2;
    
    public void Update1()
    {
        lock (_lock1)
        {
            _data1++;
        }
    }
    
    public void Update2()
    {
        lock (_lock2)
        {
            _data2++;
        }
    }
}

public class BadLocking
{
    // BAD: Locking on this
    public void Method1()
    {
        lock (this)  // External code can lock on this!
        {
            // Work
        }
    }
    
    // BAD: Locking on public object
    public object LockObject = new object();
    
    // BAD: Locking on string (interned!)
    public void Method2()
    {
        lock ("MyLock")  // All instances share same lock!
        {
            // Work
        }
    }
    
    // BAD: Locking on Type
    public void Method3()
    {
        lock (typeof(BadLocking))
        {
            // Work
        }
    }
}
```

### Lock with Exception Handling

```csharp
public class SafeResource
{
    private readonly object _lock = new object();
    
    public void ProcessWithLock()
    {
        lock (_lock)
        {
            try
            {
                // Work that might throw
                MightThrow();
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error: {ex.Message}");
                // Lock automatically released even on exception
            }
        }
    }
}
```

## Monitor Class

### Basic Monitor Usage

```csharp
public class MonitorExample
{
    private readonly object _lock = new object();
    private int _value;
    
    public void UpdateWithMonitor()
    {
        Monitor.Enter(_lock);
        try
        {
            _value++;
        }
        finally
        {
            Monitor.Exit(_lock);  // Always release
        }
    }
    
    // Equivalent to lock statement
    public void UpdateWithLock()
    {
        lock (_lock)
        {
            _value++;
        }
    }
}
```

### Monitor.TryEnter with Timeout

```csharp
public class TimeoutLock
{
    private readonly object _lock = new object();
    
    public bool TryUpdate(int timeout)
    {
        if (Monitor.TryEnter(_lock, timeout))
        {
            try
            {
                // Do work
                Thread.Sleep(100);
                return true;
            }
            finally
            {
                Monitor.Exit(_lock);
            }
        }
        
        Console.WriteLine("Couldn't acquire lock");
        return false;
    }
}

// Usage
var obj = new TimeoutLock();
var task1 = Task.Run(() => obj.TryUpdate(1000));
var task2 = Task.Run(() => obj.TryUpdate(1000));
```

### Monitor Wait and Pulse

```csharp
public class ProducerConsumer
{
    private readonly object _lock = new object();
    private readonly Queue<int> _queue = new Queue<int>();
    
    public void Produce(int value)
    {
        lock (_lock)
        {
            _queue.Enqueue(value);
            Monitor.Pulse(_lock);  // Wake one waiter
        }
    }
    
    public int Consume()
    {
        lock (_lock)
        {
            while (_queue.Count == 0)
            {
                Monitor.Wait(_lock);  // Release lock and wait
            }
            
            return _queue.Dequeue();
        }
    }
}
```

## Reader-Writer Locks

### ReaderWriterLockSlim

```csharp
public class ThreadSafeCache
{
    private readonly ReaderWriterLockSlim _lock = new ReaderWriterLockSlim();
    private readonly Dictionary<string, string> _cache = new();
    
    public string Get(string key)
    {
        _lock.EnterReadLock();
        try
        {
            return _cache.TryGetValue(key, out var value) ? value : null;
        }
        finally
        {
            _lock.ExitReadLock();
        }
    }
    
    public void Set(string key, string value)
    {
        _lock.EnterWriteLock();
        try
        {
            _cache[key] = value;
        }
        finally
        {
            _lock.ExitWriteLock();
        }
    }
    
    public void Remove(string key)
    {
        _lock.EnterWriteLock();
        try
        {
            _cache.Remove(key);
        }
        finally
        {
            _lock.ExitWriteLock();
        }
    }
}
```

## Comparison with Other Languages

### C# vs. C++
```csharp
// C#
private readonly object _lock = new object();
lock (_lock)
{
    // Critical section
}

// C++ equivalent:
// std::mutex mtx;
// {
//     std::lock_guard<std::mutex> lock(mtx);
//     // Critical section
// }
```

## Best Practices

1. **Use private lock objects**
2. **Keep critical sections small**
3. **Prefer SemaphoreSlim for async**
4. **Use ReaderWriterLockSlim for read-heavy scenarios**
5. **Avoid locking on this, typeof, or strings**

## Common Pitfalls

### Deadlock
```csharp
// BAD: Can deadlock
object lock1 = new(), lock2 = new();

Task.Run(() => {
    lock (lock1) {
        Thread.Sleep(100);
        lock (lock2) { }
    }
});

Task.Run(() => {
    lock (lock2) {
        Thread.Sleep(100);
        lock (lock1) { }
    }
});

// GOOD: Lock in same order
```

## Navigation

- [Back to C# Overview](./README.md)
- Previous: [Async/Await](./02-async-await.md)
- Next: [SemaphoreSlim](./04-semaphore-slim.md)
