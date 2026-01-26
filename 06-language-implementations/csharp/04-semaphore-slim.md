# C#의 SemaphoreSlim

`SemaphoreSlim`은 리소스에 동시에 접근할 수 있는 스레드 수를 제한하는 경량 세마포어입니다. 동기 및 비동기 대기를 모두 지원합니다.

## 목차
- [기본 개념](#기본-개념)
- [동기 사용법](#동기-사용법)
- [비동기 사용법](#비동기-사용법)
- [Rate Limiting](#rate-limiting)
- [모범 사례](#모범-사례)
- [일반적인 함정](#일반적인-함정)

## 기본 개념

### 세마포어란?

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

// 최대 3개의 동시 접근 허용
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

## 동기 사용법

### 기본 패턴

```csharp
var semaphore = new SemaphoreSlim(initialCount: 2, maxCount: 2);

void AccessResource(int id)
{
    semaphore.Wait();  // 사용 가능할 때까지 블록
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

### 타임아웃 포함

```csharp
var semaphore = new SemaphoreSlim(1);

bool TryAccessResource(int timeoutMs)
{
    if (semaphore.Wait(timeoutMs))
    {
        try
        {
            // 리소스 접근
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

## 비동기 사용법

### 기본 비동기 패턴

```csharp
private readonly SemaphoreSlim _semaphore = new(5); // 최대 5개 동시

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

### 취소 포함

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

### HTTP 요청 제한

```csharp
public class HttpThrottler
{
    private readonly HttpClient _client = new();
    private readonly SemaphoreSlim _semaphore = new(10); // 최대 10개 동시

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

### 데이터베이스 연결 풀

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

## 모범 사례

1. **항상 Finally에서 해제**: 예외가 발생해도 `Release()`가 호출되도록 보장
2. **비동기 버전 사용**: 비동기 코드에서는 `Wait()` 대신 `WaitAsync()` 선호
3. **적절한 해제**: 사용이 끝나면 세마포어 해제
4. **Wait/Release 일치**: 모든 Wait에는 해당하는 Release가 있어야 함
5. **재귀적 획득 피하기**: 중첩된 코드에서 동일한 세마포어를 대기하지 마세요

## 일반적인 함정

### 해제를 잊어버림

```csharp
// 나쁨: 예외로 인한 잠금 누수
await _semaphore.WaitAsync();
await MightThrowAsync();  // 예외 발생 시 해제하지 않음!
_semaphore.Release();

// 좋음: 항상 finally에서 해제
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

### 이중 해제

```csharp
// 나쁨: 너무 많이 해제
_semaphore.Release();
_semaphore.Release();  // maxCount를 초과할 수 있음!

// 좋음: 상태 추적
bool acquired = false;
try
{
    await _semaphore.WaitAsync();
    acquired = true;
    // 작업
}
finally
{
    if (acquired)
        _semaphore.Release();
}
```

## 완전한 예제: 병렬 다운로더

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

// 사용법
var downloader = new ParallelDownloader(maxConcurrent: 10);
var progress = new Progress<int>(p => Console.WriteLine($"Progress: {p}%"));
var urls = new[] { "https://example.com", "https://example.org" };
var results = await downloader.DownloadAllAsync(urls, progress);
```

## 네비게이션

- [C# 개요로 돌아가기](./README.md)
- 이전: [Lock과 Monitor](./03-lock-monitor.md)
- 다음: [동시 컬렉션](./05-concurrent-collections.md)
