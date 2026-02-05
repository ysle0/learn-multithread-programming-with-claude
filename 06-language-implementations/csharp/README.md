# C# 동시성

C#은 뛰어난 async/await 지원, 포괄적인 스레딩 라이브러리, 내장 동시성 컬렉션을 통해 풍부하고 고수준의 동시성 프로그래밍 접근 방식을 제공합니다. 개발자 생산성과 성능 간의 균형을 잘 맞추고 있습니다.

## 개요

C# 동시성은 크게 발전해 왔습니다:
- **.NET 1.0**: `Thread` 클래스를 사용한 기본 스레딩
- **.NET 2.0**: Thread pool, 비동기 프로그래밍 모델
- **.NET 4.0**: Task Parallel Library (TPL), PLINQ
- **.NET 4.5**: async/await 키워드
- **.NET Core/5+**: 성능 개선, Channels, ValueTask

## 핵심 구성 요소

### 1. [Thread와 Task](./01-thread-task.md)
- 저수준 스레딩을 위한 `Thread` 클래스
- 작업 기반 병렬 처리를 위한 `Task`와 TPL
- Thread pool 관리
- Task 스케줄러와 컨텍스트

### 2. [Async/Await](./02-async-await.md)
- async/await 패턴
- 비동기 프로그래밍 모델
- ConfigureAwait와 컨텍스트
- 성능을 위한 ValueTask

### 3. [Lock과 Monitor](./03-lock-monitor.md)
- 상호 배제를 위한 `lock` 문
- 고급 잠금을 위한 `Monitor` 클래스
- Reader-Writer lock
- 저지연 시나리오를 위한 SpinLock

### 4. [SemaphoreSlim](./04-semaphore-slim.md)
- 동시 접근 제한
- 비동기 대기
- 리소스 스로틀링
- 취소 지원

### 5. [Concurrent Collections](./05-concurrent-collections.md)
- `ConcurrentQueue<T>`
- `ConcurrentDictionary<K,V>`
- `ConcurrentBag<T>`
- `BlockingCollection<T>`

## 다른 언어와의 간단한 비교

| 기능 | C# | 비교 |
|------|-----|------|
| **Async 구문** | `async/await` | 가장 우아하며, JavaScript와 유사 |
| **스레딩** | Thread + Task | C++보다 고수준, Go보다 무거움 |
| **메시지 전달** | Channels (System.Threading.Channels) | Go channels와 유사 |
| **컬렉션** | 풍부한 동시성 컬렉션 | 다른 언어보다 포괄적 |
| **안전성** | 메모리 안전 (GC) | C++보다 안전, Go/JS와 유사 |

## 핵심 원칙

### 1. Thread보다 Async/Await 우선
```csharp
// 좋은 방법: 현대적인 async/await
public async Task<string> FetchDataAsync()
{
    return await httpClient.GetStringAsync(url);
}

// 이전 방식: 수동 스레드 관리
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

### 2. Task 기반 병렬 처리
```csharp
// Task가 스레드 관리를 추상화합니다
var task1 = Task.Run(() => Compute1());
var task2 = Task.Run(() => Compute2());
await Task.WhenAll(task1, task2);
```

### 3. 내장 동기화
```csharp
// C#은 풍부한 동기화 프리미티브를 제공합니다
lock (lockObject)
{
    // 임계 영역
}
```

## 일반적인 패턴

### 병렬 처리
```csharp
Parallel.For(0, 1000, i =>
{
    ProcessItem(i);
});
```

### BlockingCollection을 사용한 생산자-소비자
```csharp
var queue = new BlockingCollection<int>();

// 생산자
Task.Run(() => {
    for (int i = 0; i < 100; i++)
        queue.Add(i);
    queue.CompleteAdding();
});

// 소비자
foreach (var item in queue.GetConsumingEnumerable())
{
    Console.WriteLine(item);
}
```

## 모범 사례

1. **I/O에는 async/await 사용**: I/O 작업에서 스레드를 차단하지 마세요
2. **async void 피하기**: 이벤트 핸들러에만 사용하세요
3. **ConfigureAwait(false)**: 라이브러리 코드에서 컨텍스트 캡처를 피하기 위해 사용
4. **CancellationToken 사용**: 협력적 취소를 위해
5. **불변성 선호**: 동기화 필요성을 줄입니다
6. **동시성 컬렉션 사용**: 수동 잠금 대신 사용
7. **공개 타입에 대한 잠금 피하기**: `this`나 `typeof(Type)`에 대해 절대 lock하지 마세요

## 일반적인 실수

### 1. Async Void
```csharp
// 나쁜 예: 예외가 처리되지 않음
public async void ProcessData()
{
    await Task.Delay(1000);
    throw new Exception();  // 잡을 수 없습니다!
}

// 좋은 예: Task 반환
public async Task ProcessDataAsync()
{
    await Task.Delay(1000);
}
```

### 2. .Result로 인한 교착 상태
```csharp
// 나쁜 예: UI/ASP.NET 컨텍스트에서 교착 상태 발생
public void Button_Click(object sender, EventArgs e)
{
    var result = GetDataAsync().Result;  // 교착 상태!
}

// 좋은 예: await 사용
public async void Button_Click(object sender, EventArgs e)
{
    var result = await GetDataAsync();
}
```

### 3. 잘못된 객체에 대한 잠금
```csharp
// 나쁜 예: 공개 객체에 잠금
public class BadClass
{
    public void Method()
    {
        lock (this)  // 나쁨: 외부 코드도 이것에 잠금을 걸 수 있습니다!
        {
        }
    }
}

// 좋은 예: 비공개 잠금 객체
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

### 4. 불필요한 SynchronizationContext 캡처
```csharp
// 라이브러리 코드에서 (나쁜 예):
await Task.Delay(1000);  // 컨텍스트를 캡처함

// 좋은 예: 컨텍스트를 캡처하지 않음
await Task.Delay(1000).ConfigureAwait(false);
```

### 5. 취소 처리 미흡
```csharp
// 나쁜 예: 취소 지원 없음
public async Task ProcessAsync()
{
    await Task.Delay(10000);
}

// 좋은 예: 취소 지원
public async Task ProcessAsync(CancellationToken ct)
{
    await Task.Delay(10000, ct);
}
```

## 성능 고려 사항

### Task vs. Thread
- **Thread**: 생성에 ~100 μs, ~2MB 메모리
- **Task**: Thread pool에서 실행, 최소한의 오버헤드
- **원칙**: 전용 스레드가 필요한 경우가 아니면 Task를 사용하세요

### ValueTask vs. Task
```csharp
// 결과가 자주 동기적으로 사용 가능한 경우 ValueTask 사용
public ValueTask<int> GetCachedValue(string key)
{
    if (cache.TryGetValue(key, out int value))
        return new ValueTask<int>(value);  // 할당 없음

    return new ValueTask<int>(FetchFromDbAsync(key));
}
```

### Thread Pool 크기 조정
```csharp
// Thread pool 정보 가져오기
ThreadPool.GetMinThreads(out int minWorker, out int minIO);
ThreadPool.GetMaxThreads(out int maxWorker, out int maxIO);

// 필요시 조정 (드물게 필요)
ThreadPool.SetMinThreads(Environment.ProcessorCount * 2, minIO);
```

## 최신 C# 기능

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

// 사용
await foreach (var number in GetNumbersAsync())
{
    Console.WriteLine(number);
}
```

### Channels (System.Threading.Channels)
```csharp
var channel = Channel.CreateUnbounded<int>();

// 생산자
await channel.Writer.WriteAsync(42);

// 소비자
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
        // 정리
    }
}

await using var resource = new AsyncResource();
```

## 도구 및 디버깅

### Visual Studio 디버거
- Threads 창
- Parallel Stacks 창
- Tasks 창
- Concurrency Visualizer

### 성능 프로파일링
- dotTrace
- PerfView
- Visual Studio Profiler
- 마이크로 벤치마크를 위한 BenchmarkDotNet

### 진단 도구
```csharp
// 교착 상태 감지
ThreadPool.GetAvailableThreads(out int available, out int _);
if (available == 0)
{
    // Thread pool 기아 가능성
}
```

## 플랫폼 고려 사항

### .NET Framework vs. .NET Core/5+
- .NET Core는 더 나은 async 성능을 제공합니다
- .NET 5+는 개선된 Thread pool을 갖추고 있습니다
- 일부 API는 플랫폼 간에 차이가 있습니다

### ASP.NET Core
- 컨트롤러에서 `Task.Run`을 사용하지 마세요
- async 코드를 차단하지 마세요
- 끝까지 async를 사용하세요

### WPF/WinForms
- UI 스레드로 마샬링: `Dispatcher.Invoke` / `Control.Invoke`
- 또는 async/await 사용 (SynchronizationContext를 자동으로 캡처)

## 추가 자료

- **C# in Depth** by Jon Skeet
- **Concurrency in C# Cookbook** by Stephen Cleary
- Microsoft Docs: https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/
- Stephen Cleary's Blog: https://blog.stephencleary.com/

## 탐색

- [언어 구현으로 돌아가기](../)
- 다음 주제:
  - [Thread와 Task](./01-thread-task.md)
  - [Async/Await](./02-async-await.md)
  - [Lock과 Monitor](./03-lock-monitor.md)
  - [SemaphoreSlim](./04-semaphore-slim.md)
  - [Concurrent Collections](./05-concurrent-collections.md)
