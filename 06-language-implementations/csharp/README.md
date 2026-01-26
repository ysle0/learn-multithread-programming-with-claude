# C# 동시성

C#은 뛰어난 async/await 지원, 포괄적인 스레딩 라이브러리, 내장 동시 컬렉션을 제공하는 풍부하고 고수준의 동시 프로그래밍 접근 방식을 제공합니다. 개발자 생산성과 성능의 균형을 잘 맞춥니다.

## 개요

C# 동시성은 크게 발전해왔습니다:
- **.NET 1.0**: `Thread` 클래스를 사용한 기본 스레딩
- **.NET 2.0**: 스레드 풀, 비동기 프로그래밍 모델
- **.NET 4.0**: Task Parallel Library (TPL), PLINQ
- **.NET 4.5**: async/await 키워드
- **.NET Core/5+**: 성능 개선, Channels, ValueTask

## 핵심 구성 요소

### 1. [스레드와 태스크](./01-thread-task.md)
- 저수준 스레딩을 위한 `Thread` 클래스
- 태스크 기반 병렬 처리를 위한 `Task`와 TPL
- 스레드 풀 관리
- 태스크 스케줄러와 컨텍스트

### 2. [Async/Await](./02-async-await.md)
- async/await 패턴
- 비동기 프로그래밍 모델
- ConfigureAwait와 컨텍스트
- 성능을 위한 ValueTask

### 3. [Lock과 Monitor](./03-lock-monitor.md)
- 상호 배제를 위한 `lock` 문
- 고급 잠금을 위한 `Monitor` 클래스
- Reader-writer 락
- 저지연 시나리오를 위한 SpinLock

### 4. [SemaphoreSlim](./04-semaphore-slim.md)
- 동시 접근 제한
- 비동기 대기
- 리소스 제한(throttling)
- 취소 지원

### 5. [동시 컬렉션](./05-concurrent-collections.md)
- `ConcurrentQueue<T>`
- `ConcurrentDictionary<K,V>`
- `ConcurrentBag<T>`
- `BlockingCollection<T>`

### 6. [PLINQ](./06-plinq.md)
- Parallel LINQ
- 선언적 병렬 처리
- 성능 최적화
- 집계 및 변환

### 7. [Parallel 클래스](./07-parallel-class.md)
- `Parallel.For`와 `Parallel.ForEach`
- `Parallel.Invoke`
- 명령형 병렬 처리
- Thread-Local State

### 8. [TPL Dataflow](./08-tpl-dataflow.md)
- 액터 모델 구현
- 데이터 흐름 파이프라인
- 블록 기반 아키텍처
- 복잡한 병렬 처리

### 9. [Channels](./09-channels.md)
- 고성능 생산자-소비자 패턴
- 비동기 큐
- 백프레셔 관리
- Go 스타일 채널

## 다른 언어와의 간단한 비교

| 기능 | C# | 비교 |
|------|---|------|
| **비동기 구문** | `async/await` | 가장 우아함, JavaScript와 유사 |
| **스레딩** | Thread + Task | C++보다 높은 수준, Go보다 무거움 |
| **메시지 전달** | Channels (System.Threading.Channels) | Go 채널과 유사 |
| **컬렉션** | 풍부한 동시 컬렉션 | 다른 언어보다 포괄적 |
| **안전성** | 메모리 안전 (GC) | C++보다 안전, Go/JS와 유사 |

## 핵심 원칙

### 1. 스레드보다 Async/Await
```csharp
// 좋음: 현대적인 async/await
public async Task<string> FetchDataAsync()
{
    return await httpClient.GetStringAsync(url);
}

// 옛날 방식: 수동 스레드 관리
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

### 2. 태스크 기반 병렬 처리
```csharp
// 태스크는 스레드 관리를 추상화합니다
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

1. **I/O에는 async/await 사용**: I/O 작업에서 스레드를 블록하지 마세요
2. **async void 피하기**: 이벤트 핸들러에만 사용하세요
3. **ConfigureAwait(false)**: 라이브러리 코드에서 컨텍스트 캡처를 피하세요
4. **CancellationToken 사용**: 협력적 취소를 위해
5. **불변성 선호**: 동기화 필요성 감소
6. **동시 컬렉션 사용**: 수동 잠금보다
7. **공용 타입에 대한 잠금 피하기**: `this`나 `typeof(Type)`에 잠금을 걸지 마세요

## 일반적인 함정

### 1. Async Void
```csharp
// 나쁨: 예외를 처리할 수 없음
public async void ProcessData()
{
    await Task.Delay(1000);
    throw new Exception();  // 캐치할 수 없음!
}

// 좋음: Task 반환
public async Task ProcessDataAsync()
{
    await Task.Delay(1000);
}
```

### 2. .Result로 인한 데드락
```csharp
// 나쁨: UI/ASP.NET 컨텍스트에서 데드락
public void Button_Click(object sender, EventArgs e)
{
    var result = GetDataAsync().Result;  // 데드락!
}

// 좋음: await 사용
public async void Button_Click(object sender, EventArgs e)
{
    var result = await GetDataAsync();
}
```

### 3. 잘못된 객체에 대한 잠금
```csharp
// 나쁨: 공용 객체에 잠금
public class BadClass
{
    public void Method()
    {
        lock (this)  // 나쁨: 다른 곳에서도 this에 잠금을 걸 수 있음!
        {
        }
    }
}

// 좋음: 전용 잠금 객체
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
// 라이브러리 코드에서 (나쁨):
await Task.Delay(1000);  // 컨텍스트 캡처

// 좋음: 컨텍스트를 캡처하지 않음
await Task.Delay(1000).ConfigureAwait(false);
```

### 5. 취소 처리하지 않음
```csharp
// 나쁨: 취소 지원 없음
public async Task ProcessAsync()
{
    await Task.Delay(10000);
}

// 좋음: 취소 지원
public async Task ProcessAsync(CancellationToken ct)
{
    await Task.Delay(10000, ct);
}
```

## 성능 고려사항

### Task vs. Thread
- **Thread**: 생성에 ~100 μs, ~2MB 메모리
- **Task**: 스레드 풀에서 실행, 최소 오버헤드
- **규칙**: 전용 스레드가 필요하지 않으면 태스크 사용

### ValueTask vs. Task
```csharp
// 결과가 자주 동기적으로 사용 가능할 때 ValueTask 사용
public ValueTask<int> GetCachedValue(string key)
{
    if (cache.TryGetValue(key, out int value))
        return new ValueTask<int>(value);  // 할당 없음

    return new ValueTask<int>(FetchFromDbAsync(key));
}
```

### 스레드 풀 크기 조정
```csharp
// 스레드 풀 정보 가져오기
ThreadPool.GetMinThreads(out int minWorker, out int minIO);
ThreadPool.GetMaxThreads(out int maxWorker, out int maxIO);

// 필요한 경우 조정 (거의 필요하지 않음)
ThreadPool.SetMinThreads(Environment.ProcessorCount * 2, minIO);
```

## 현대 C# 기능

### 비동기 스트림 (C# 8.0)
```csharp
public async IAsyncEnumerable<int> GetNumbersAsync()
{
    for (int i = 0; i < 10; i++)
    {
        await Task.Delay(100);
        yield return i;
    }
}

// 소비
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
- 스레드 창
- 병렬 스택 창
- 태스크 창
- 동시성 시각화 도구

### 성능 프로파일링
- dotTrace
- PerfView
- Visual Studio 프로파일러
- 마이크로 벤치마크를 위한 BenchmarkDotNet

### 진단 도구
```csharp
// 데드락 감지
ThreadPool.GetAvailableThreads(out int available, out int _);
if (available == 0)
{
    // 잠재적 스레드 풀 기아
}
```

## 플랫폼 고려사항

### .NET Framework vs. .NET Core/5+
- .NET Core가 더 나은 비동기 성능 제공
- .NET 5+가 개선된 스레드 풀 제공
- 플랫폼 간에 일부 API가 다름

### ASP.NET Core
- 컨트롤러에서 `Task.Run`을 사용하지 마세요
- 비동기 코드를 블록하지 마세요
- 끝까지 비동기 사용

### WPF/WinForms
- UI 스레드로 마샬링: `Dispatcher.Invoke` / `Control.Invoke`
- 또는 async/await 사용 (자동으로 SynchronizationContext 캡처)

## 추가 읽을거리

- **C# in Depth** by Jon Skeet
- **Concurrency in C# Cookbook** by Stephen Cleary
- Microsoft Docs: https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/
- Stephen Cleary's Blog: https://blog.stephencleary.com/

## 네비게이션

- [언어 구현으로 돌아가기](../)
- 주제 목록:
  - [스레드와 태스크](./01-thread-task.md)
  - [Async/Await](./02-async-await.md)
  - [Lock과 Monitor](./03-lock-monitor.md)
  - [SemaphoreSlim](./04-semaphore-slim.md)
  - [동시 컬렉션](./05-concurrent-collections.md)
  - [PLINQ](./06-plinq.md)
  - [Parallel 클래스](./07-parallel-class.md)
  - [TPL Dataflow](./08-tpl-dataflow.md)
  - [Channels](./09-channels.md)
