# C#의 스레드와 태스크

C#은 저수준 `Thread` 클래스와 고수준 `Task` 추상화를 모두 제공합니다. 현대 C#은 대부분의 시나리오에서 스레드보다 태스크를 강력히 선호합니다.

## 목차
- [Thread 클래스](#thread-클래스)
- [Task Parallel Library](#task-parallel-library)
- [Task vs Thread](#task-vs-thread)
- [Parallel 클래스](#parallel-클래스)
- [다른 언어와의 비교](#다른-언어와의-비교)
- [모범 사례](#모범-사례)
- [일반적인 함정](#일반적인-함정)

## Thread 클래스

### 스레드 생성

```csharp
using System;
using System.Threading;

class Program
{
    static void PrintNumbers()
    {
        for (int i = 0; i < 5; i++)
        {
            Console.WriteLine($"Number: {i}");
            Thread.Sleep(100);
        }
    }

    static void Main()
    {
        // 스레드 생성 및 시작
        var thread = new Thread(PrintNumbers);
        thread.Start();

        // 완료 대기
        thread.Join();

        Console.WriteLine("Thread completed");
    }
}
```

### 매개변수 전달

```csharp
using System;
using System.Threading;

class Program
{
    static void PrintMessage(object data)
    {
        string message = (string)data;
        Console.WriteLine(message);
    }

    static void Main()
    {
        var thread = new Thread(PrintMessage);
        thread.Start("Hello from thread!");
        thread.Join();

        // 람다 사용 (더 좋음)
        string msg = "Hello from lambda!";
        var thread2 = new Thread(() => Console.WriteLine(msg));
        thread2.Start();
        thread2.Join();
    }
}
```

### 스레드 속성

```csharp
using System;
using System.Threading;

var thread = new Thread(() => {
    Console.WriteLine($"Thread ID: {Thread.CurrentThread.ManagedThreadId}");
    Console.WriteLine($"Is Background: {Thread.CurrentThread.IsBackground}");
    Console.WriteLine($"Priority: {Thread.CurrentThread.Priority}");
});

// 속성 설정
thread.Name = "Worker Thread";
thread.IsBackground = true;  // 프로세스 종료를 막지 않음
thread.Priority = ThreadPriority.AboveNormal;

thread.Start();
thread.Join();
```

### 백그라운드 vs. 포그라운드

```csharp
// 포그라운드 스레드 - 프로세스를 유지
var foregroundThread = new Thread(() => {
    Thread.Sleep(5000);
    Console.WriteLine("Foreground done");
});
foregroundThread.IsBackground = false;
foregroundThread.Start();

// 백그라운드 스레드 - 프로세스를 유지하지 않음
var backgroundThread = new Thread(() => {
    Thread.Sleep(5000);
    Console.WriteLine("Background done");  // 출력되지 않을 수 있음
});
backgroundThread.IsBackground = true;
backgroundThread.Start();

// 모든 포그라운드 스레드가 완료되면 프로세스 종료
```

## Task Parallel Library

### 기본 태스크 생성

```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        // 방법 1: Task.Run
        var task1 = Task.Run(() => {
            Console.WriteLine("Task 1 running");
            return 42;
        });

        // 방법 2: Task.Factory.StartNew (더 많은 옵션)
        var task2 = Task.Factory.StartNew(() => {
            Console.WriteLine("Task 2 running");
            return 100;
        });

        // 방법 3: 별도로 생성 및 시작
        var task3 = new Task(() => {
            Console.WriteLine("Task 3 running");
        });
        task3.Start();

        // 모두 대기
        await Task.WhenAll(task1, task2, task3);

        Console.WriteLine($"Results: {task1.Result}, {task2.Result}");
    }
}
```

### 반환 값이 있는 태스크

```csharp
using System;
using System.Threading.Tasks;

static async Task Main()
{
    Task<int> task = Task.Run(() => {
        Thread.Sleep(1000);
        return 42;
    });

    Console.WriteLine("Task started");

    // 결과 대기
    int result = await task;
    Console.WriteLine($"Result: {result}");

    // 또는 .Result 사용 (블록, 데드락 가능)
    // int result = task.Result;  // 가능하면 하지 마세요
}
```

### 태스크 연속

```csharp
using System;
using System.Threading.Tasks;

var task = Task.Run(() => {
    Thread.Sleep(1000);
    return 42;
})
.ContinueWith(prevTask => {
    int result = prevTask.Result;
    Console.WriteLine($"Previous result: {result}");
    return result * 2;
})
.ContinueWith(prevTask => {
    int result = prevTask.Result;
    Console.WriteLine($"Final result: {result}");
});

await task;
```

### 태스크 취소

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

static async Task Main()
{
    var cts = new CancellationTokenSource();

    var task = Task.Run(async () => {
        for (int i = 0; i < 100; i++)
        {
            // 취소 확인
            cts.Token.ThrowIfCancellationRequested();

            Console.WriteLine($"Working: {i}");
            await Task.Delay(100, cts.Token);
        }
    }, cts.Token);

    // 1초 후 취소
    await Task.Delay(1000);
    cts.Cancel();

    try
    {
        await task;
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("Task was cancelled");
    }
}
```

### Task.WhenAll과 Task.WhenAny

```csharp
using System;
using System.Threading.Tasks;

static async Task Main()
{
    var task1 = Task.Delay(1000).ContinueWith(_ => "Task 1");
    var task2 = Task.Delay(2000).ContinueWith(_ => "Task 2");
    var task3 = Task.Delay(1500).ContinueWith(_ => "Task 3");

    // 모든 태스크 대기
    string[] results = await Task.WhenAll(task1, task2, task3);
    Console.WriteLine($"All done: {string.Join(", ", results)}");

    // 또는 첫 번째 완료 대기
    var firstTask = await Task.WhenAny(task1, task2, task3);
    Console.WriteLine($"First done: {firstTask.Result}");
}
```

### 예외 처리

```csharp
using System;
using System.Threading.Tasks;

static async Task Main()
{
    var task = Task.Run(() => {
        throw new InvalidOperationException("Something went wrong!");
    });

    try
    {
        await task;
    }
    catch (InvalidOperationException ex)
    {
        Console.WriteLine($"Caught: {ex.Message}");
    }

    // 예외가 있는 여러 태스크
    var task1 = Task.Run(() => throw new Exception("Error 1"));
    var task2 = Task.Run(() => throw new Exception("Error 2"));

    try
    {
        await Task.WhenAll(task1, task2);
    }
    catch (Exception ex)
    {
        // 첫 번째 예외만 캐치됨
        Console.WriteLine($"First exception: {ex.Message}");

        // 태스크에서 모든 예외 가져오기
        if (task1.IsFaulted)
        {
            foreach (var inner in task1.Exception!.InnerExceptions)
                Console.WriteLine($"Task1 exception: {inner.Message}");
        }
    }
}
```

## Task vs Thread

### 비교

```csharp
// Thread: 저수준, 수동 관리
var thread = new Thread(() => {
    // 작업
});
thread.Start();
thread.Join();

// Task: 고수준, 스레드 풀 사용
var task = Task.Run(() => {
    // 작업
});
await task;
```

### Thread를 사용해야 하는 경우

```csharp
// Thread를 사용하는 경우:
// 1. 장기 실행 작업 (스레드 풀에 적합하지 않음)
var longRunningThread = new Thread(() => {
    // 장기 실행 작업
});
longRunningThread.IsBackground = true;
longRunningThread.Start();

// 또는 LongRunning 힌트와 함께 Task 사용
var longRunningTask = Task.Factory.StartNew(() => {
    // 장기 실행 작업
}, TaskCreationOptions.LongRunning);

// 2. 특정 스레드 구성
var thread = new Thread(() => {
    // 작업
});
thread.Priority = ThreadPriority.Highest;
thread.SetApartmentState(ApartmentState.STA);  // COM interop용
thread.Start();
```

### 스레드 풀

```csharp
using System;
using System.Threading;

// 스레드 풀에 직접 작업 큐잉
ThreadPool.QueueUserWorkItem(state => {
    Console.WriteLine("Work item executing");
});

// 스레드 풀 정보 가져오기
ThreadPool.GetMinThreads(out int minWorker, out int minIO);
ThreadPool.GetMaxThreads(out int maxWorker, out int maxIO);
ThreadPool.GetAvailableThreads(out int availWorker, out int availIO);

Console.WriteLine($"Thread pool: min={minWorker}, max={maxWorker}, avail={availWorker}");
```

## Parallel 클래스

### Parallel.For

```csharp
using System;
using System.Threading.Tasks;

static void Main()
{
    // 순차
    for (int i = 0; i < 100; i++)
    {
        ProcessItem(i);
    }

    // 병렬
    Parallel.For(0, 100, i => {
        ProcessItem(i);
    });

    // 옵션 포함
    var options = new ParallelOptions {
        MaxDegreeOfParallelism = Environment.ProcessorCount
    };

    Parallel.For(0, 100, options, i => {
        ProcessItem(i);
    });
}

static void ProcessItem(int i)
{
    Console.WriteLine($"Processing {i} on thread {Thread.CurrentThread.ManagedThreadId}");
}
```

### Parallel.ForEach

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

var items = new List<string> { "apple", "banana", "cherry", "date" };

// 병렬 foreach
Parallel.ForEach(items, item => {
    Console.WriteLine($"Processing {item}");
    // 항목 처리
});

// 병렬 처리 정도 지정
var options = new ParallelOptions {
    MaxDegreeOfParallelism = 2
};

Parallel.ForEach(items, options, item => {
    ProcessItem(item);
});
```

### Parallel.Invoke

```csharp
using System;
using System.Threading.Tasks;

// 메서드를 병렬로 실행
Parallel.Invoke(
    () => Method1(),
    () => Method2(),
    () => Method3()
);

Console.WriteLine("All methods completed");

static void Method1() => Console.WriteLine("Method 1");
static void Method2() => Console.WriteLine("Method 2");
static void Method3() => Console.WriteLine("Method 3");
```

### 병렬 루프 중단

```csharp
using System;
using System.Threading.Tasks;

Parallel.For(0, 1000, (i, state) => {
    if (i == 100)
    {
        Console.WriteLine("Breaking at 100");
        state.Break();  // 또는 state.Stop()
        return;
    }

    ProcessItem(i);
});
```

## 다른 언어와의 비교

### C# vs. C++
```csharp
// C#: 태스크 기반
var task = Task.Run(() => Compute());
var result = await task;

// C++ 동등 (async):
// auto future = std::async(compute);
// auto result = future.get();
```

### C# vs. Go
```csharp
// C#: 태스크
var task = Task.Run(() => Work());
await task;

// Go 동등 (고루틴):
// go work()
// (직접적인 await 없음; 채널 또는 sync.WaitGroup 사용)
```

### C# vs. JavaScript
```csharp
// C#: async/await (매우 유사!)
var result = await FetchDataAsync();

// JavaScript:
// const result = await fetchData();
```

## 모범 사례

### 1. Thread보다 Task 선호

```csharp
// 좋음: Task 사용
var task = Task.Run(() => Work());
await task;

// 덜 좋음: 수동 스레드 (필요할 때만)
var thread = new Thread(() => Work());
thread.Start();
thread.Join();
```

### 2. 끝까지 비동기 사용

```csharp
// 좋음: 끝까지 비동기
public async Task<string> GetDataAsync()
{
    return await httpClient.GetStringAsync(url);
}

// 나쁨: 비동기 메서드에서 블록
public async Task<string> GetDataBad()
{
    return httpClient.GetString(url);  // 블록!
}
```

### 3. 라이브러리 코드에서 ConfigureAwait

```csharp
// 라이브러리 코드에서
public async Task<string> GetDataAsync()
{
    return await httpClient.GetStringAsync(url)
                           .ConfigureAwait(false);
}

// 애플리케이션 코드(UI)에서는 기본값이 괜찮음
public async Task Button_Click(object sender, EventArgs e)
{
    var data = await GetDataAsync();  // UI 스레드에서 재개
    textBox.Text = data;
}
```

### 4. 취소 처리

```csharp
public async Task ProcessAsync(CancellationToken ct)
{
    for (int i = 0; i < 100; i++)
    {
        ct.ThrowIfCancellationRequested();
        await DoWorkAsync(ct);
    }
}
```

### 5. 태스크를 블록하지 마세요

```csharp
// 나쁨: 데드락 가능
public void BadMethod()
{
    var result = GetDataAsync().Result;  // UI/ASP.NET에서 데드락!
}

// 좋음: 끝까지 비동기
public async Task GoodMethod()
{
    var result = await GetDataAsync();
}

// 반드시 블록해야 하는 경우 (콘솔 앱), GetAwaiter().GetResult() 사용
public void ConsoleMain()
{
    var result = GetDataAsync().GetAwaiter().GetResult();
}
```

## 일반적인 함정

### 1. 스레드 풀 기아

```csharp
// 나쁨: 스레드 풀 스레드 블록
Parallel.For(0, 1000, i => {
    Thread.Sleep(10000);  // 스레드 풀 스레드 블록!
});

// 좋음: async 사용
await Task.WhenAll(Enumerable.Range(0, 1000).Select(async i => {
    await Task.Delay(10000);
}));
```

### 2. 태스크를 Dispose하지 않음

```csharp
// 일반적으로 괜찮음: 태스크는 해제가 필요 없음
var task = Task.Run(() => Work());
await task;

// 예외: TaskCompletionSource와 취소 토큰
using var cts = new CancellationTokenSource();
var task = LongRunningWorkAsync(cts.Token);
```

### 3. Fire and Forget

```csharp
// 나쁨: 예외가 손실됨
public void BadMethod()
{
    Task.Run(() => {
        throw new Exception("Lost!");
    });
}

// 좋음: 대기 또는 처리
public async Task GoodMethod()
{
    await Task.Run(() => {
        throw new Exception("Caught!");
    });
}

// 정말 fire-and-forget이라면, 최소한 예외 로깅
public void FireAndForget()
{
    Task.Run(async () => {
        try
        {
            await DoWorkAsync();
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Background work failed");
        }
    });
}
```

### 4. 클로저 캡처 문제

```csharp
// 나쁨: 루프 변수를 잘못 캡처 (C# 5 이전)
for (int i = 0; i < 10; i++)
{
    Task.Run(() => Console.WriteLine(i));  // 잘못된 값 출력 가능
}

// 좋음: 복사본 캡처
for (int i = 0; i < 10; i++)
{
    int copy = i;
    Task.Run(() => Console.WriteLine(copy));
}

// 참고: C# 5+에서 foreach는 올바르게 캡처
foreach (var item in items)
{
    Task.Run(() => Console.WriteLine(item));  // 괜찮음
}
```

### 5. ASP.NET에서 Task.Run

```csharp
// 나쁨: ASP.NET에서 Task.Run 사용하지 마세요
public async Task<IActionResult> Index()
{
    var data = await Task.Run(() => GetData());  // 스레드 낭비!
    return View(data);
}

// 좋음: GetData를 비동기로 만들기
public async Task<IActionResult> Index()
{
    var data = await GetDataAsync();
    return View(data);
}
```

## 성능 고려사항

### 태스크 오버헤드

```csharp
// 태스크 오버헤드: ~1 μs
// 작업이 100 μs 이상일 때만 가치 있음

// 너무 작음: 오버헤드가 지배적
Parallel.For(0, 1000000, i => {
    result[i] = i * 2;  // 너무 단순
});

// 더 나음: 더 큰 청크
var chunkSize = 10000;
Parallel.For(0, 1000000 / chunkSize, chunk => {
    for (int i = chunk * chunkSize; i < (chunk + 1) * chunkSize; i++)
    {
        result[i] = i * 2;
    }
});
```

### 핫 패스에 ValueTask

```csharp
// 결과가 자주 동기적으로 사용 가능할 때 ValueTask 사용
public ValueTask<int> GetCachedAsync(string key)
{
    if (cache.TryGetValue(key, out int value))
        return new ValueTask<int>(value);  // 할당 없음

    return new ValueTask<int>(FetchFromDbAsync(key));
}
```

## 완전한 예제: 병렬 합계

```csharp
using System;
using System.Linq;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        int[] numbers = Enumerable.Range(1, 10_000_000).ToArray();

        // 순차
        var sw = System.Diagnostics.Stopwatch.StartNew();
        long sum1 = numbers.Sum(x => (long)x);
        Console.WriteLine($"Sequential: {sum1} in {sw.ElapsedMilliseconds}ms");

        // Parallel.For를 사용한 병렬
        sw.Restart();
        long sum2 = 0;
        object lockObj = new object();
        Parallel.For(0, numbers.Length, () => 0L, (i, state, subtotal) => {
            return subtotal + numbers[i];
        }, subtotal => {
            lock (lockObj) sum2 += subtotal;
        });
        Console.WriteLine($"Parallel.For: {sum2} in {sw.ElapsedMilliseconds}ms");

        // 태스크를 사용한 병렬
        sw.Restart();
        int numTasks = Environment.ProcessorCount;
        var tasks = new Task<long>[numTasks];
        int chunkSize = numbers.Length / numTasks;

        for (int t = 0; t < numTasks; t++)
        {
            int start = t * chunkSize;
            int end = (t == numTasks - 1) ? numbers.Length : (t + 1) * chunkSize;

            tasks[t] = Task.Run(() => {
                long subtotal = 0;
                for (int i = start; i < end; i++)
                    subtotal += numbers[i];
                return subtotal;
            });
        }

        var results = await Task.WhenAll(tasks);
        long sum3 = results.Sum();
        Console.WriteLine($"Tasks: {sum3} in {sw.ElapsedMilliseconds}ms");

        // PLINQ (가장 쉬움!)
        sw.Restart();
        long sum4 = numbers.AsParallel().Sum(x => (long)x);
        Console.WriteLine($"PLINQ: {sum4} in {sw.ElapsedMilliseconds}ms");
    }
}
```

## 추가 읽을거리

- [Async/Await](./02-async-await.md)
- [Lock과 Monitor](./03-lock-monitor.md)
- Microsoft Docs: Task Parallel Library

## 네비게이션

- [C# 개요로 돌아가기](./README.md)
- 다음: [Async/Await](./02-async-await.md)
