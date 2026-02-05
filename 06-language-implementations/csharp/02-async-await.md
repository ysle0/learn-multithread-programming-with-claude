# C#의 Async/Await

async/await 패턴은 비동기 프로그래밍을 위한 C#의 대표적인 기능으로, 스레드를 차단하지 않으면서 동시성 작업을 위한 깔끔하고 읽기 쉬운 코드를 제공합니다.

## 목차
- [기본 개념](#기본-개념)
- [Async 메서드](#async-메서드)
- [Task 대기](#task-대기)
- [ConfigureAwait](#configureawait)
- [오류 처리](#오류-처리)
- [다른 언어와의 비교](#다른-언어와의-비교)
- [모범 사례](#모범-사례)
- [일반적인 실수](#일반적인-실수)

## 기본 개념

### Async/Await란?

```csharp
using System;
using System.Net.Http;
using System.Threading.Tasks;

class Program
{
    // 동기 방식 - 스레드를 차단합니다
    static string FetchDataSync(string url)
    {
        using var client = new HttpClient();
        return client.GetStringAsync(url).Result;  // 차단!
    }

    // 비동기 방식 - 스레드를 차단하지 않습니다
    static async Task<string> FetchDataAsync(string url)
    {
        using var client = new HttpClient();
        return await client.GetStringAsync(url);  // 차단하지 않음!
    }

    static async Task Main()
    {
        var data = await FetchDataAsync("https://api.github.com");
        Console.WriteLine($"Received {data.Length} characters");
    }
}
```

### Async/Await의 작동 방식

```csharp
public async Task<int> ComputeAsync()
{
    Console.WriteLine("1. Starting");

    await Task.Delay(1000);  // 여기서 제어를 양보합니다

    Console.WriteLine("2. After delay");
    return 42;
}

// 컴파일러가 이것을 상태 머신으로 변환합니다
```

## Async 메서드

### 기본 Async 메서드

```csharp
public async Task DoWorkAsync()
{
    await Task.Delay(1000);
    Console.WriteLine("Work done");
}

// 사용법
await DoWorkAsync();
```

### 반환 값이 있는 Async 메서드

```csharp
public async Task<int> GetValueAsync()
{
    await Task.Delay(1000);
    return 42;
}

// 사용법
int value = await GetValueAsync();
```

### Async Void (이벤트 핸들러에만 사용)

```csharp
// 이벤트 핸들러에만 사용
private async void Button_Click(object sender, EventArgs e)
{
    try
    {
        await ProcessAsync();
    }
    catch (Exception ex)
    {
        // 여기서 예외를 처리해야 합니다 - 전파할 수 없습니다!
        MessageBox.Show(ex.Message);
    }
}

// 다른 곳에서는 async void를 사용하지 마세요
public async void BadMethod()  // 나쁨!
{
    await Task.Delay(1000);
}

// 좋은 예: Task 반환
public async Task GoodMethod()
{
    await Task.Delay(1000);
}
```

### 성능을 위한 ValueTask

```csharp
public ValueTask<int> GetCachedValueAsync(string key)
{
    // 캐시에 값이 있으면 동기적으로 반환 (할당 없음)
    if (_cache.TryGetValue(key, out int value))
    {
        return new ValueTask<int>(value);
    }

    // 그렇지 않으면 실제 비동기 작업 반환
    return new ValueTask<int>(FetchFromDatabaseAsync(key));
}

// 사용법은 Task와 동일
int value = await GetCachedValueAsync("key");
```

## Task 대기

### 여러 Task 대기

```csharp
public async Task ProcessMultipleAsync()
{
    // 순차 실행 - 느림
    var result1 = await FetchData1Async();
    var result2 = await FetchData2Async();
    var result3 = await FetchData3Async();

    // 병렬 실행 - 빠름
    var task1 = FetchData1Async();
    var task2 = FetchData2Async();
    var task3 = FetchData3Async();

    await Task.WhenAll(task1, task2, task3);

    var r1 = task1.Result;
    var r2 = task2.Result;
    var r3 = task3.Result;
}
```

### 결과가 있는 Task.WhenAll

```csharp
public async Task<string[]> FetchAllAsync(string[] urls)
{
    var tasks = urls.Select(url => FetchDataAsync(url));
    return await Task.WhenAll(tasks);
}

// 사용법
var results = await FetchAllAsync(new[] { "url1", "url2", "url3" });
```

### 타임아웃을 위한 Task.WhenAny

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

### 취소 지원

```csharp
public async Task ProcessAsync(CancellationToken cancellationToken)
{
    for (int i = 0; i < 100; i++)
    {
        // 취소 확인
        cancellationToken.ThrowIfCancellationRequested();

        await DoWorkAsync(cancellationToken);
    }
}

// 사용법
var cts = new CancellationTokenSource();
var task = ProcessAsync(cts.Token);

// 5초 후 취소
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

### SynchronizationContext 이해하기

```csharp
// UI 애플리케이션 (WPF/WinForms)에서
private async void Button_Click(object sender, EventArgs e)
{
    // UI SynchronizationContext를 캡처합니다
    var data = await FetchDataAsync();

    // UI 스레드에서 재개 - UI 업데이트 가능
    textBox.Text = data;
}
```

### ConfigureAwait(false)

```csharp
// 라이브러리 코드에서 - 컨텍스트를 캡처하지 않음
public async Task<string> GetDataAsync()
{
    // 원래 컨텍스트에서 재개할 필요 없음
    var data = await httpClient.GetStringAsync(url)
                               .ConfigureAwait(false);

    // 다른 스레드에서 재개될 수 있음 - 괜찮습니다
    return ProcessData(data);
}

// UI 업데이트가 있는 애플리케이션 코드에서
private async void Button_Click(object sender, EventArgs e)
{
    // 기본값 유지 (ConfigureAwait(true))
    var data = await GetDataAsync();

    // UI 스레드로 복귀 - UI 업데이트 가능
    textBox.Text = data;
}
```

### ConfigureAwait(false)를 사용해야 하는 경우

```csharp
// 라이브러리 코드 - 항상 ConfigureAwait(false) 사용
public async Task<string> LibraryMethodAsync()
{
    var result = await SomeOperationAsync().ConfigureAwait(false);
    var processed = await ProcessAsync(result).ConfigureAwait(false);
    return processed;
}

// UI 업데이트가 없는 애플리케이션 코드 - ConfigureAwait(false) 사용 가능
public async Task BackgroundProcessAsync()
{
    await Task.Delay(1000).ConfigureAwait(false);
    // UI 스레드가 필요 없는 작업 수행
}

// UI 업데이트가 있는 애플리케이션 코드 - ConfigureAwait(false) 사용하지 않음
private async void UpdateUI()
{
    var data = await FetchAsync();  // 기본값으로 충분
    textBox.Text = data;  // UI 스레드 필요
}
```

## 오류 처리

### Async에서 Try-Catch

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

### 여러 Task의 예외

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
        // 첫 번째 예외만 가져옴
        Console.WriteLine($"First exception: {ex.Message}");
    }

    // 모든 예외를 가져오려면
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

### Async에서 Using 문

```csharp
public async Task ProcessFileAsync(string path)
{
    using var stream = File.OpenRead(path);
    var content = await ReadStreamAsync(stream);
    return content;
}

// 비동기 해제 (C# 8.0+)
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
        // 정리
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

## 다른 언어와의 비교

### C# vs. JavaScript

```csharp
// C# async/await (매우 유사!)
public async Task<string> FetchDataAsync(string url)
{
    var response = await httpClient.GetAsync(url);
    return await response.Content.ReadAsStringAsync();
}

// JavaScript 동등 코드:
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

// Python 동등 코드:
// async def fetch(url):
//     async with aiohttp.ClientSession() as session:
//         async with session.get(url) as response:
//             return await response.text()
```

### C# vs. C++

```csharp
// C#: 내장 async/await
var result = await ComputeAsync();

// C++: 코루틴 (C++20, 더 복잡함)
// co_await compute();
```

## 모범 사례

### 1. 끝까지 Async 사용

```csharp
// 좋은 예: 끝까지 async
public async Task<string> GetDataAsync()
{
    return await FetchAsync();
}

// 나쁜 예: 동기와 비동기 혼합
public string GetData()
{
    return FetchAsync().Result;  // 교착 상태 가능!
}
```

### 2. Async Void 피하기

```csharp
// 좋은 예: Task 반환
public async Task ProcessAsync()
{
    await DoWorkAsync();
}

// 나쁜 예: Async void (이벤트 핸들러 제외)
public async void ProcessBad()
{
    await DoWorkAsync();
}
```

### 3. CancellationToken 사용

```csharp
// 좋은 예: 취소 지원
public async Task ProcessAsync(CancellationToken ct)
{
    await DoWorkAsync(ct);
}

// 사용법
var cts = new CancellationTokenSource();
await ProcessAsync(cts.Token);
```

### 4. 라이브러리에서 ConfigureAwait

```csharp
// 라이브러리 코드에서
public async Task<string> LibraryMethodAsync()
{
    return await FetchAsync().ConfigureAwait(false);
}

// 애플리케이션 코드에서 (컨텍스트 필요)
private async void Button_Click(object sender, EventArgs e)
{
    var data = await LibraryMethodAsync();  // 기본값으로 충분
    textBox.Text = data;
}
```

### 5. 병렬 처리에 Task.WhenAll 선호

```csharp
// 좋은 예: 병렬 실행
var task1 = FetchAsync(url1);
var task2 = FetchAsync(url2);
await Task.WhenAll(task1, task2);

// 나쁜 예: 순차 실행
var data1 = await FetchAsync(url1);
var data2 = await FetchAsync(url2);
```

## 일반적인 실수

### 1. .Result 또는 .Wait()로 인한 교착 상태

```csharp
// UI/ASP.NET에서 교착 상태!
public void BadMethod()
{
    var result = GetDataAsync().Result;  // 차단
}

public async Task<string> GetDataAsync()
{
    return await FetchAsync();  // 차단된 컨텍스트에서 재개 시도
}

// 좋은 예: 끝까지 async 사용
public async Task GoodMethod()
{
    var result = await GetDataAsync();
}
```

### 2. Async Void 예외

```csharp
// 나쁜 예: 예외를 잡을 수 없음
public async void BadAsync()
{
    await Task.Delay(100);
    throw new Exception("Lost!");
}

// 호출자가 잡을 수 없습니다!
try
{
    BadAsync();
}
catch (Exception)
{
    // 절대 잡히지 않음!
}

// 좋은 예: Task 반환
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
// 나쁜 예: 예외 손실
public void StartBackground()
{
    Task.Run(async () => {
        await DoWorkAsync();
        throw new Exception("Lost!");
    });
}

// 좋은 예: await 또는 예외 처리
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

### 4. Async 메서드에서 차단

```csharp
// 나쁜 예: async 메서드에서 차단
public async Task<string> BadAsync()
{
    Thread.Sleep(1000);  // 스레드를 차단합니다!
    return "result";
}

// 좋은 예: async API 사용
public async Task<string> GoodAsync()
{
    await Task.Delay(1000);  // 차단하지 않음
    return "result";
}
```

### 5. CancellationToken 미사용

```csharp
// 나쁜 예: 취소할 방법이 없음
public async Task LongProcessAsync()
{
    for (int i = 0; i < 1000; i++)
    {
        await DoWorkAsync();
    }
}

// 좋은 예: 취소 지원
public async Task LongProcessAsync(CancellationToken ct)
{
    for (int i = 0; i < 1000; i++)
    {
        ct.ThrowIfCancellationRequested();
        await DoWorkAsync(ct);
    }
}
```

## 고급 패턴

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

// 사용
await foreach (var number in GenerateNumbersAsync())
{
    Console.WriteLine(number);
}
```

### Async 지연 초기화

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

// 사용법
private readonly AsyncLazy<string> _data = new AsyncLazy<string>(LoadDataAsync);

public async Task UseDataAsync()
{
    var data = await _data.Value;
}
```

### 진행률 보고

```csharp
public async Task ProcessWithProgressAsync(IProgress<int> progress)
{
    for (int i = 0; i < 100; i++)
    {
        await DoWorkAsync();
        progress?.Report(i + 1);
    }
}

// 사용법
var progress = new Progress<int>(percent => {
    Console.WriteLine($"Progress: {percent}%");
});

await ProcessWithProgressAsync(progress);
```

## 성능 고려 사항

### ValueTask vs Task

```csharp
// 대부분의 경우 Task 사용
public async Task<int> GetValueAsync()
{
    await SomeOperationAsync();
    return 42;
}

// 자주 동기적으로 완료되는 경우 ValueTask 사용
public ValueTask<int> GetCachedAsync(string key)
{
    if (_cache.TryGetValue(key, out int value))
        return new ValueTask<int>(value);  // 할당 없음

    return new ValueTask<int>(FetchAsync(key));
}
```

### 불필요한 Async 피하기

```csharp
// 나쁜 예: 불필요한 async/await
public async Task<int> GetValueAsync()
{
    return await Task.FromResult(42);
}

// 좋은 예: Task를 그대로 반환
public Task<int> GetValueAsync()
{
    return Task.FromResult(42);
}

// 또는 값을 알고 있으면 더 나은 방법
public ValueTask<int> GetValueAsync()
{
    return new ValueTask<int>(42);
}
```

## 전체 예제: Async 웹 스크래퍼

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
    private readonly SemaphoreSlim _semaphore = new(5); // 최대 5개 동시 실행

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

// 사용법
var scraper = new WebScraper();
var urls = new[] { "https://example.com", "https://example.org" };
var results = await scraper.FetchAllAsync(urls);

foreach (var (url, length) in results)
{
    Console.WriteLine($"{url}: {length} bytes");
}
```

## 추가 자료

- [Thread와 Task](./01-thread-task.md)
- [SemaphoreSlim](./04-semaphore-slim.md)
- Stephen Cleary's Blog: https://blog.stephencleary.com/

## 탐색

- [C# 개요로 돌아가기](./README.md)
- 이전: [Thread와 Task](./01-thread-task.md)
- 다음: [Lock과 Monitor](./03-lock-monitor.md)
