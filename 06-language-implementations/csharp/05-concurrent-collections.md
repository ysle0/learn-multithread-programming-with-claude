# C#의 Concurrent Collections

C#은 `System.Collections.Concurrent` 네임스페이스에서 내부적으로 동기화를 처리하는 스레드 안전 컬렉션 클래스를 제공하여 동시성 프로그래밍을 더 쉽고 안전하게 만듭니다.

## 목차
- [개요](#개요)
- [ConcurrentQueue](#concurrentqueue)
- [ConcurrentStack](#concurrentstack)
- [ConcurrentDictionary](#concurrentdictionary)
- [ConcurrentBag](#concurrentbag)
- [BlockingCollection](#blockingcollection)
- [모범 사례](#모범-사례)

## 개요

### 왜 Concurrent Collections인가?

```csharp
// 나쁜 예: 수동 잠금
var queue = new Queue<int>();
var lockObj = new object();

void Enqueue(int item)
{
    lock (lockObj)
    {
        queue.Enqueue(item);
    }
}

// 좋은 예: 동시성 컬렉션 (lock-free)
var queue = new ConcurrentQueue<int>();

void Enqueue(int item)
{
    queue.Enqueue(item);  // 스레드 안전, 잠금 불필요
}
```

## ConcurrentQueue

### 기본 연산

```csharp
using System.Collections.Concurrent;

var queue = new ConcurrentQueue<int>();

// 항목 추가
queue.Enqueue(1);
queue.Enqueue(2);
queue.Enqueue(3);

// 꺼내기 시도
if (queue.TryDequeue(out int result))
{
    Console.WriteLine($"Dequeued: {result}");
}

// 엿보기 시도
if (queue.TryPeek(out int peeked))
{
    Console.WriteLine($"Front: {peeked}");
}
```

### ConcurrentQueue를 사용한 생산자-소비자

```csharp
var queue = new ConcurrentQueue<int>();

// 생산자
var producers = Enumerable.Range(0, 3).Select(id => Task.Run(() => {
    for (int i = 0; i < 10; i++)
    {
        queue.Enqueue(id * 100 + i);
        Console.WriteLine($"Producer {id} added {id * 100 + i}");
    }
}));

// 소비자
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

### 기본 연산

```csharp
var dict = new ConcurrentDictionary<string, int>();

// 추가 또는 업데이트
dict.TryAdd("apple", 1);
dict.TryAdd("banana", 2);

// 가져오기 또는 추가
int value = dict.GetOrAdd("cherry", 3);

// 팩토리를 사용한 추가 또는 업데이트
dict.AddOrUpdate("apple",
    addValue: 10,  // 키가 존재하지 않는 경우
    updateValueFactory: (key, oldValue) => oldValue + 1);  // 존재하는 경우

// 값 가져오기 시도
if (dict.TryGetValue("apple", out int appleCount))
{
    Console.WriteLine($"Apples: {appleCount}");
}

// 제거 시도
if (dict.TryRemove("banana", out int removed))
{
    Console.WriteLine($"Removed: {removed}");
}
```

### 스레드 안전 카운터

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

### 비정렬 컬렉션

```csharp
var bag = new ConcurrentBag<int>();

// 여러 스레드에서 항목 추가
Parallel.For(0, 100, i => {
    bag.Add(i);
});

// 항목 꺼내기 (순서 보장 없음)
while (bag.TryTake(out int item))
{
    Console.WriteLine(item);
}
```

## BlockingCollection

### 생산자-소비자 패턴

```csharp
var collection = new BlockingCollection<int>(boundedCapacity: 10);

// 생산자
var producer = Task.Run(() => {
    for (int i = 0; i < 100; i++)
    {
        collection.Add(i);
        Console.WriteLine($"Produced: {i}");
    }
    collection.CompleteAdding();
});

// 소비자
var consumer = Task.Run(() => {
    foreach (var item in collection.GetConsumingEnumerable())
    {
        Console.WriteLine($"Consumed: {item}");
        Thread.Sleep(50);
    }
});

await Task.WhenAll(producer, consumer);
```

### 다수의 생산자와 소비자

```csharp
var collection = new BlockingCollection<string>(100);

// 다수의 생산자
var producers = Enumerable.Range(0, 3).Select(id => Task.Run(() => {
    for (int i = 0; i < 10; i++)
    {
        collection.Add($"Producer{id}-Item{i}");
    }
}));

// 다수의 소비자
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

## 모범 사례

1. **lock + 일반 컬렉션 대신 동시성 컬렉션 사용**
2. **성능 특성 이해** (lock-free vs 차단)
3. **생산자-소비자에는 BlockingCollection 사용**
4. **순서를 가정하지 않기** (ConcurrentQueue/Stack 제외)
5. **예외 대신 TryXxx 메서드 선호**

## 전체 예제: 웹 크롤러

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

                            // 새 URL 추출 및 큐에 추가 (간소화)
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

## 탐색

- [C# 개요로 돌아가기](./README.md)
- 이전: [SemaphoreSlim](./04-semaphore-slim.md)
- [언어 구현으로 돌아가기](../)
