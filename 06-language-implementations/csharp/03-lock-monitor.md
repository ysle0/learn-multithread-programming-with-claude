# C#의 Lock과 Monitor

C#은 상호 배제를 위한 `lock` 문과 `Monitor` 클래스를 제공하여 공유 데이터를 동시 접근으로부터 보호합니다.

## 목차
- [Lock 문](#lock-문)
- [Monitor 클래스](#monitor-클래스)
- [Reader-Writer Lock](#reader-writer-lock)
- [다른 언어와의 비교](#다른-언어와의-비교)
- [모범 사례](#모범-사례)
- [일반적인 실수](#일반적인-실수)

## Lock 문

### 기본 Lock 사용법

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

// 사용법
var counter = new Counter();
var tasks = Enumerable.Range(0, 10).Select(_ =>
    Task.Run(() => {
        for (int i = 0; i < 1000; i++)
            counter.Increment();
    })
);

await Task.WhenAll(tasks);
Console.WriteLine($"Count: {counter.GetCount()}");  // 항상 10000
```

### Lock 객체 가이드라인

```csharp
public class GoodLocking
{
    // 좋은 예: 비공개 잠금 객체
    private readonly object _lock = new object();

    // 좋은 예: 다른 데이터에 대한 여러 잠금
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
    // 나쁜 예: this에 잠금
    public void Method1()
    {
        lock (this)  // 외부 코드도 이것에 잠금을 걸 수 있습니다!
        {
            // 작업
        }
    }

    // 나쁜 예: 공개 객체에 잠금
    public object LockObject = new object();

    // 나쁜 예: 문자열에 잠금 (인턴됨!)
    public void Method2()
    {
        lock ("MyLock")  // 모든 인스턴스가 같은 잠금을 공유합니다!
        {
            // 작업
        }
    }

    // 나쁜 예: Type에 잠금
    public void Method3()
    {
        lock (typeof(BadLocking))
        {
            // 작업
        }
    }
}
```

### 예외 처리와 Lock

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
                // 예외를 던질 수 있는 작업
                MightThrow();
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error: {ex.Message}");
                // 예외 발생 시에도 잠금은 자동으로 해제됩니다
            }
        }
    }
}
```

## Monitor 클래스

### 기본 Monitor 사용법

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
            Monitor.Exit(_lock);  // 항상 해제
        }
    }

    // lock 문과 동일
    public void UpdateWithLock()
    {
        lock (_lock)
        {
            _value++;
        }
    }
}
```

### 타임아웃이 있는 Monitor.TryEnter

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
                // 작업 수행
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

// 사용법
var obj = new TimeoutLock();
var task1 = Task.Run(() => obj.TryUpdate(1000));
var task2 = Task.Run(() => obj.TryUpdate(1000));
```

### Monitor Wait와 Pulse

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
            Monitor.Pulse(_lock);  // 대기 중인 하나를 깨움
        }
    }

    public int Consume()
    {
        lock (_lock)
        {
            while (_queue.Count == 0)
            {
                Monitor.Wait(_lock);  // 잠금을 해제하고 대기
            }

            return _queue.Dequeue();
        }
    }
}
```

## Reader-Writer Lock

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

## 다른 언어와의 비교

### C# vs. C++
```csharp
// C#
private readonly object _lock = new object();
lock (_lock)
{
    // 임계 영역
}

// C++ 동등 코드:
// std::mutex mtx;
// {
//     std::lock_guard<std::mutex> lock(mtx);
//     // 임계 영역
// }
```

## 모범 사례

1. **비공개 잠금 객체 사용**
2. **임계 영역을 작게 유지**
3. **async에서는 SemaphoreSlim 선호**
4. **읽기 작업이 많은 시나리오에서는 ReaderWriterLockSlim 사용**
5. **this, typeof, 문자열에 잠금을 걸지 않기**

## 일반적인 실수

### 교착 상태
```csharp
// 나쁜 예: 교착 상태 가능
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

// 좋은 예: 같은 순서로 잠금
```

## 탐색

- [C# 개요로 돌아가기](./README.md)
- 이전: [Async/Await](./02-async-await.md)
- 다음: [SemaphoreSlim](./04-semaphore-slim.md)
