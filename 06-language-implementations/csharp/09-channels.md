# Channels

## 📌 개요

`System.Threading.Channels`는 .NET Core 3.0 / .NET Standard 2.1에서 도입된 고성능 생산자-소비자 패턴 구현입니다. TPL Dataflow보다 **가볍고 빠르며**, **async/await**와 완벽하게 통합됩니다. Go 언어의 채널에서 영감을 받았습니다.

**주요 특징:**
- 고성능 비동기 큐
- 백프레셔 내장
- 다중 생산자/소비자 지원
- 단순하고 직관적인 API
- 메모리 효율적

**설치:** .NET Core 3.0+ 또는 .NET 5+에 기본 포함
```bash
# .NET Standard 2.0/2.1용
dotnet add package System.Threading.Channels
```

## 1. 기본 사용법

### Unbounded Channel

```csharp
using System;
using System.Threading.Channels;
using System.Threading.Tasks;

class UnboundedChannelExample
{
    static async Task Main()
    {
        // 무제한 채널 생성
        var channel = Channel.CreateUnbounded<int>();

        // 생산자
        var producer = Task.Run(async () =>
        {
            for (int i = 0; i < 10; i++)
            {
                await channel.Writer.WriteAsync(i);
                Console.WriteLine($"생산: {i}");
                await Task.Delay(100);
            }

            // 쓰기 완료
            channel.Writer.Complete();
            Console.WriteLine("생산자 완료");
        });

        // 소비자
        var consumer = Task.Run(async () =>
        {
            // 채널에서 읽기 (비동기 열거)
            await foreach (var item in channel.Reader.ReadAllAsync())
            {
                Console.WriteLine($"소비: {item}");
                await Task.Delay(200);
            }

            Console.WriteLine("소비자 완료");
        });

        await Task.WhenAll(producer, consumer);
    }
}

// 출력:
// 생산: 0
// 소비: 0
// 생산: 1
// 생산: 2
// 소비: 1
// ...
// 생산자 완료
// 소비자 완료
```

### Bounded Channel (백프레셔)

```csharp
using System;
using System.Diagnostics;
using System.Threading.Channels;
using System.Threading.Tasks;

class BoundedChannelExample
{
    static async Task Main()
    {
        // 용량이 3인 제한된 채널
        var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(3)
        {
            FullMode = BoundedChannelFullMode.Wait  // 가득 차면 대기
        });

        var sw = Stopwatch.StartNew();

        // 빠른 생산자
        var producer = Task.Run(async () =>
        {
            for (int i = 0; i < 10; i++)
            {
                Console.WriteLine($"[{sw.ElapsedMilliseconds}ms] 쓰기 시도: {i}");
                await channel.Writer.WriteAsync(i);  // 채널이 가득 차면 여기서 대기!
                Console.WriteLine($"[{sw.ElapsedMilliseconds}ms] 쓰기 성공: {i}");
            }

            channel.Writer.Complete();
        });

        // 느린 소비자
        var consumer = Task.Run(async () =>
        {
            await Task.Delay(500);  // 초기 지연

            await foreach (var item in channel.Reader.ReadAllAsync())
            {
                Console.WriteLine($"[{sw.ElapsedMilliseconds}ms] 읽기: {item}");
                await Task.Delay(300);  // 느린 처리
            }
        });

        await Task.WhenAll(producer, consumer);
    }
}

// 출력:
// [0ms] 쓰기 시도: 0
// [1ms] 쓰기 성공: 0
// [1ms] 쓰기 시도: 1
// [1ms] 쓰기 성공: 1
// [1ms] 쓰기 시도: 2
// [2ms] 쓰기 성공: 2
// [2ms] 쓰기 시도: 3
// [505ms] 읽기: 0  // 소비자 시작
// [505ms] 쓰기 성공: 3  // 생산자가 다시 쓸 수 있음
// [505ms] 쓰기 시도: 4
// ...
// 백프레셔가 자동으로 적용됨!
```

## 2. 채널 옵션

### BoundedChannelOptions

```csharp
using System;
using System.Threading.Channels;
using System.Threading.Tasks;

class ChannelOptionsExample
{
    static async Task Main()
    {
        await TestFullModeWait();
        await TestFullModeDropNewest();
        await TestFullModeDropOldest();
        await TestFullModeDropWrite();
    }

    // FullMode: Wait - 가득 차면 대기
    static async Task TestFullModeWait()
    {
        Console.WriteLine("=== FullMode.Wait ===");

        var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(3)
        {
            FullMode = BoundedChannelFullMode.Wait
        });

        // 4개 쓰기 시도 (마지막은 블로킹됨)
        await channel.Writer.WriteAsync(1);
        await channel.Writer.WriteAsync(2);
        await channel.Writer.WriteAsync(3);

        var writeTask = channel.Writer.WriteAsync(4);  // 블로킹!

        Console.WriteLine("채널 가득참, 4번째 쓰기 대기 중...");

        // 하나 읽어서 공간 확보
        var item = await channel.Reader.ReadAsync();
        Console.WriteLine($"읽기: {item}");

        // 이제 4번째 쓰기 완료됨
        await writeTask;
        Console.WriteLine("4번째 쓰기 완료!\n");
    }

    // FullMode: DropNewest - 새 항목 버림
    static async Task TestFullModeDropNewest()
    {
        Console.WriteLine("=== FullMode.DropNewest ===");

        var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(3)
        {
            FullMode = BoundedChannelFullMode.DropNewest
        });

        await channel.Writer.WriteAsync(1);
        await channel.Writer.WriteAsync(2);
        await channel.Writer.WriteAsync(3);
        await channel.Writer.WriteAsync(4);  // 버려짐!
        await channel.Writer.WriteAsync(5);  // 버려짐!

        Console.WriteLine("5개 쓰기 완료 (마지막 2개는 버려짐)");

        channel.Writer.Complete();

        Console.Write("읽은 항목: ");
        await foreach (var item in channel.Reader.ReadAllAsync())
        {
            Console.Write($"{item} ");
        }
        Console.WriteLine("\n");
    }

    // FullMode: DropOldest - 가장 오래된 항목 버림
    static async Task TestFullModeDropOldest()
    {
        Console.WriteLine("=== FullMode.DropOldest ===");

        var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(3)
        {
            FullMode = BoundedChannelFullMode.DropOldest
        });

        await channel.Writer.WriteAsync(1);
        await channel.Writer.WriteAsync(2);
        await channel.Writer.WriteAsync(3);
        await channel.Writer.WriteAsync(4);  // 1이 버려지고 4가 추가됨
        await channel.Writer.WriteAsync(5);  // 2가 버려지고 5가 추가됨

        Console.WriteLine("5개 쓰기 완료 (처음 2개는 버려짐)");

        channel.Writer.Complete();

        Console.Write("읽은 항목: ");
        await foreach (var item in channel.Reader.ReadAllAsync())
        {
            Console.Write($"{item} ");
        }
        Console.WriteLine("\n");
    }

    // FullMode: DropWrite - 쓰기 실패 반환
    static async Task TestFullModeDropWrite()
    {
        Console.WriteLine("=== FullMode.DropWrite ===");

        var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(3)
        {
            FullMode = BoundedChannelFullMode.DropWrite
        });

        Console.WriteLine($"쓰기 1: {await channel.Writer.WriteAsync(1)}");
        Console.WriteLine($"쓰기 2: {await channel.Writer.WriteAsync(2)}");
        Console.WriteLine($"쓰기 3: {await channel.Writer.WriteAsync(3)}");

        // TryWrite로 실패 확인
        if (!channel.Writer.TryWrite(4))
        {
            Console.WriteLine("쓰기 4: 실패 (채널 가득참)");
        }

        channel.Writer.Complete();

        Console.Write("읽은 항목: ");
        await foreach (var item in channel.Reader.ReadAllAsync())
        {
            Console.Write($"{item} ");
        }
        Console.WriteLine();
    }
}

// 출력:
// === FullMode.Wait ===
// 채널 가득참, 4번째 쓰기 대기 중...
// 읽기: 1
// 4번째 쓰기 완료!
//
// === FullMode.DropNewest ===
// 5개 쓰기 완료 (마지막 2개는 버려짐)
// 읽은 항목: 1 2 3
//
// === FullMode.DropOldest ===
// 5개 쓰기 완료 (처음 2개는 버려짐)
// 읽은 항목: 3 4 5
//
// === FullMode.DropWrite ===
// 쓰기 1: True
// 쓰기 2: True
// 쓰기 3: True
// 쓰기 4: 실패 (채널 가득참)
// 읽은 항목: 1 2 3
```

### SingleWriter/SingleReader 최적화

```csharp
using System;
using System.Diagnostics;
using System.Threading.Channels;
using System.Threading.Tasks;

class SingleOptimizationExample
{
    const int MessageCount = 1_000_000;

    static async Task Main()
    {
        // 일반 채널
        await TestNormalChannel();

        // 단일 생산자/소비자 최적화
        await TestOptimizedChannel();
    }

    static async Task TestNormalChannel()
    {
        var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(100)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleWriter = false,  // 다중 생산자
            SingleReader = false   // 다중 소비자
        });

        var sw = Stopwatch.StartNew();
        await RunTest(channel);
        sw.Stop();

        Console.WriteLine($"일반 채널: {sw.ElapsedMilliseconds}ms");
    }

    static async Task TestOptimizedChannel()
    {
        var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(100)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleWriter = true,   // 단일 생산자 최적화
            SingleReader = true    // 단일 소비자 최적화
        });

        var sw = Stopwatch.StartNew();
        await RunTest(channel);
        sw.Stop();

        Console.WriteLine($"최적화 채널: {sw.ElapsedMilliseconds}ms");
    }

    static async Task RunTest(Channel<int> channel)
    {
        var producer = Task.Run(async () =>
        {
            for (int i = 0; i < MessageCount; i++)
            {
                await channel.Writer.WriteAsync(i);
            }
            channel.Writer.Complete();
        });

        var consumer = Task.Run(async () =>
        {
            var sum = 0;
            await foreach (var item in channel.Reader.ReadAllAsync())
            {
                sum += item;
            }
        });

        await Task.WhenAll(producer, consumer);
    }
}

// 출력 예시:
// 일반 채널: 450ms
// 최적화 채널: 280ms
// 약 1.6배 빠름!
```

## 3. 다중 생산자/소비자 패턴

### 다중 생산자, 단일 소비자

```csharp
using System;
using System.Linq;
using System.Threading.Channels;
using System.Threading.Tasks;

class MultipleProducersSingleConsumer
{
    static async Task Main()
    {
        var channel = Channel.CreateUnbounded<string>();

        // 3개의 생산자
        var producers = Enumerable.Range(1, 3).Select(id =>
            Task.Run(async () =>
            {
                for (int i = 0; i < 5; i++)
                {
                    var message = $"Producer{id}-Message{i}";
                    await channel.Writer.WriteAsync(message);
                    Console.WriteLine($"[P{id}] 생산: {message}");
                    await Task.Delay(100);
                }
                Console.WriteLine($"[P{id}] 완료");
            })
        ).ToArray();

        // 단일 소비자
        var consumer = Task.Run(async () =>
        {
            await foreach (var message in channel.Reader.ReadAllAsync())
            {
                Console.WriteLine($"[C] 소비: {message}");
                await Task.Delay(50);
            }
        });

        // 모든 생산자 완료 대기
        await Task.WhenAll(producers);
        channel.Writer.Complete();

        await consumer;
    }
}

// 출력:
// [P1] 생산: Producer1-Message0
// [P2] 생산: Producer2-Message0
// [P3] 생산: Producer3-Message0
// [C] 소비: Producer1-Message0
// [C] 소비: Producer2-Message0
// [C] 소비: Producer3-Message0
// ...
```

### 단일 생산자, 다중 소비자

```csharp
using System;
using System.Linq;
using System.Threading.Channels;
using System.Threading.Tasks;

class SingleProducerMultipleConsumers
{
    static async Task Main()
    {
        var channel = Channel.CreateUnbounded<int>();

        // 단일 생산자
        var producer = Task.Run(async () =>
        {
            for (int i = 0; i < 20; i++)
            {
                await channel.Writer.WriteAsync(i);
                Console.WriteLine($"[P] 생산: {i}");
                await Task.Delay(100);
            }

            channel.Writer.Complete();
            Console.WriteLine("[P] 완료");
        });

        // 3개의 소비자 (경쟁적으로 읽기)
        var consumers = Enumerable.Range(1, 3).Select(id =>
            Task.Run(async () =>
            {
                await foreach (var item in channel.Reader.ReadAllAsync())
                {
                    Console.WriteLine($"[C{id}] 소비: {item}");
                    await Task.Delay(200);
                }
                Console.WriteLine($"[C{id}] 완료");
            })
        ).ToArray();

        await Task.WhenAll(producer);
        await Task.WhenAll(consumers);
    }
}

// 출력:
// [P] 생산: 0
// [C1] 소비: 0
// [P] 생산: 1
// [P] 생산: 2
// [C2] 소비: 1
// [C3] 소비: 2
// ...
// 각 소비자가 경쟁적으로 항목 가져감
```

### Fan-Out / Fan-In 패턴

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Channels;
using System.Threading.Tasks;

class FanOutFanInPattern
{
    static async Task Main()
    {
        // Fan-Out: 하나의 입력을 여러 워커로 분배
        var inputChannel = Channel.CreateUnbounded<int>();
        var workerChannels = Enumerable.Range(0, 3)
            .Select(_ => Channel.CreateUnbounded<int>())
            .ToArray();

        // Fan-Out: 라운드 로빈 분배
        var fanOut = Task.Run(async () =>
        {
            int workerIndex = 0;
            await foreach (var item in inputChannel.Reader.ReadAllAsync())
            {
                await workerChannels[workerIndex].Writer.WriteAsync(item);
                Console.WriteLine($"Fan-Out: {item} → Worker{workerIndex}");
                workerIndex = (workerIndex + 1) % workerChannels.Length;
            }

            // 모든 워커 채널 종료
            foreach (var channel in workerChannels)
            {
                channel.Writer.Complete();
            }
        });

        // 워커들: 병렬 처리
        var outputChannel = Channel.CreateUnbounded<int>();
        var workers = workerChannels.Select((channel, index) =>
            Task.Run(async () =>
            {
                await foreach (var item in channel.Reader.ReadAllAsync())
                {
                    Console.WriteLine($"[W{index}] 처리 중: {item}");
                    await Task.Delay(200);  // 처리 시뮬레이션

                    var result = item * item;
                    await outputChannel.Writer.WriteAsync(result);
                    Console.WriteLine($"[W{index}] 결과: {result}");
                }
            })
        ).ToArray();

        // Fan-In: 여러 워커의 결과를 하나로 수집
        var fanIn = Task.Run(async () =>
        {
            await Task.WhenAll(workers);
            outputChannel.Writer.Complete();
        });

        // 결과 수집
        var collector = Task.Run(async () =>
        {
            var results = new List<int>();
            await foreach (var result in outputChannel.Reader.ReadAllAsync())
            {
                results.Add(result);
                Console.WriteLine($"Fan-In 수집: {result}");
            }
            Console.WriteLine($"\n총 {results.Count}개 결과 수집됨");
        });

        // 데이터 생산
        for (int i = 0; i < 9; i++)
        {
            await inputChannel.Writer.WriteAsync(i);
            await Task.Delay(50);
        }
        inputChannel.Writer.Complete();

        await Task.WhenAll(fanOut, fanIn, collector);
    }
}

// 출력:
// Fan-Out: 0 → Worker0
// Fan-Out: 1 → Worker1
// Fan-Out: 2 → Worker2
// Fan-Out: 3 → Worker0
// [W0] 처리 중: 0
// [W1] 처리 중: 1
// [W2] 처리 중: 2
// [W0] 결과: 0
// [W1] 결과: 1
// [W2] 결과: 4
// Fan-In 수집: 0
// Fan-In 수집: 1
// Fan-In 수집: 4
// ...
```

## 4. 실전 예제: 병렬 HTTP 요청 처리

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Net.Http;
using System.Threading;
using System.Threading.Channels;
using System.Threading.Tasks;

class ParallelHttpProcessor
{
    static readonly HttpClient httpClient = new HttpClient();

    class Request
    {
        public string Url { get; set; }
        public int Id { get; set; }
    }

    class Response
    {
        public int RequestId { get; set; }
        public string Url { get; set; }
        public int StatusCode { get; set; }
        public long ContentLength { get; set; }
        public TimeSpan Duration { get; set; }
    }

    static async Task Main()
    {
        var urls = new[]
        {
            "https://www.google.com",
            "https://www.github.com",
            "https://www.microsoft.com",
            "https://www.stackoverflow.com",
            "https://www.reddit.com",
            "https://www.wikipedia.org",
            "https://www.amazon.com",
            "https://www.twitter.com"
        };

        var requestChannel = Channel.CreateBounded<Request>(new BoundedChannelOptions(10)
        {
            FullMode = BoundedChannelFullMode.Wait
        });

        var responseChannel = Channel.CreateUnbounded<Response>();

        var totalSw = Stopwatch.StartNew();

        // 생산자: 요청 생성
        var producer = Task.Run(async () =>
        {
            for (int i = 0; i < urls.Length; i++)
            {
                await requestChannel.Writer.WriteAsync(new Request
                {
                    Id = i,
                    Url = urls[i]
                });
                Console.WriteLine($"[생산자] 요청 큐잉: {urls[i]}");
            }

            requestChannel.Writer.Complete();
            Console.WriteLine("[생산자] 모든 요청 큐잉 완료");
        });

        // 워커들: HTTP 요청 처리 (병렬 처리)
        const int workerCount = 4;
        var workers = Enumerable.Range(0, workerCount).Select(workerId =>
            Task.Run(async () =>
            {
                await foreach (var request in requestChannel.Reader.ReadAllAsync())
                {
                    Console.WriteLine($"[워커{workerId}] 처리 시작: {request.Url}");

                    var sw = Stopwatch.StartNew();
                    try
                    {
                        var response = await httpClient.GetAsync(request.Url);
                        var content = await response.Content.ReadAsByteArrayAsync();
                        sw.Stop();

                        await responseChannel.Writer.WriteAsync(new Response
                        {
                            RequestId = request.Id,
                            Url = request.Url,
                            StatusCode = (int)response.StatusCode,
                            ContentLength = content.Length,
                            Duration = sw.Elapsed
                        });

                        Console.WriteLine($"[워커{workerId}] 완료: {request.Url} ({sw.ElapsedMilliseconds}ms)");
                    }
                    catch (Exception ex)
                    {
                        Console.WriteLine($"[워커{workerId}] 오류: {request.Url} - {ex.Message}");
                    }
                }

                Console.WriteLine($"[워커{workerId}] 종료");
            })
        ).ToArray();

        // 응답 채널 종료 처리
        var completionTask = Task.Run(async () =>
        {
            await Task.WhenAll(workers);
            responseChannel.Writer.Complete();
        });

        // 소비자: 응답 수집 및 통계
        var responses = new List<Response>();
        var consumer = Task.Run(async () =>
        {
            await foreach (var response in responseChannel.Reader.ReadAllAsync())
            {
                responses.Add(response);
                Console.WriteLine($"[소비자] 응답 수집: {response.Url} " +
                    $"(상태: {response.StatusCode}, 크기: {response.ContentLength} bytes, " +
                    $"시간: {response.Duration.TotalMilliseconds:F0}ms)");
            }
        });

        await Task.WhenAll(producer, completionTask, consumer);
        totalSw.Stop();

        // 통계 출력
        Console.WriteLine("\n=== 통계 ===");
        Console.WriteLine($"총 요청 수: {responses.Count}");
        Console.WriteLine($"총 실행 시간: {totalSw.ElapsedMilliseconds}ms");
        Console.WriteLine($"평균 응답 시간: {responses.Average(r => r.Duration.TotalMilliseconds):F2}ms");
        Console.WriteLine($"최소 응답 시간: {responses.Min(r => r.Duration.TotalMilliseconds):F2}ms");
        Console.WriteLine($"최대 응답 시간: {responses.Max(r => r.Duration.TotalMilliseconds):F2}ms");
        Console.WriteLine($"총 다운로드 크기: {responses.Sum(r => r.ContentLength):N0} bytes");
    }
}

// 출력 예시:
// [생산자] 요청 큐잉: https://www.google.com
// [생산자] 요청 큐잉: https://www.github.com
// ...
// [워커0] 처리 시작: https://www.google.com
// [워커1] 처리 시작: https://www.github.com
// [워커2] 처리 시작: https://www.microsoft.com
// [워커3] 처리 시작: https://www.stackoverflow.com
// [워커0] 완료: https://www.google.com (245ms)
// [소비자] 응답 수집: https://www.google.com (상태: 200, 크기: 15234 bytes, 시간: 245ms)
// ...
//
// === 통계 ===
// 총 요청 수: 8
// 총 실행 시간: 1250ms
// 평균 응답 시간: 312.50ms
// ...
```

## 5. 실전 예제: 실시간 로그 처리 시스템

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Channels;
using System.Threading.Tasks;

class LogProcessingSystem
{
    enum LogLevel { Debug, Info, Warning, Error, Critical }

    class LogEntry
    {
        public DateTime Timestamp { get; set; }
        public LogLevel Level { get; set; }
        public string Message { get; set; }
        public string Source { get; set; }
    }

    static async Task Main()
    {
        var cts = new CancellationTokenSource();

        // 로그 입력 채널
        var logChannel = Channel.CreateBounded<LogEntry>(new BoundedChannelOptions(1000)
        {
            FullMode = BoundedChannelFullMode.DropOldest  // 오래된 로그 버림
        });

        // 레벨별 채널
        var debugChannel = Channel.CreateUnbounded<LogEntry>();
        var infoChannel = Channel.CreateUnbounded<LogEntry>();
        var warningChannel = Channel.CreateUnbounded<LogEntry>();
        var errorChannel = Channel.CreateUnbounded<LogEntry>();
        var criticalChannel = Channel.CreateUnbounded<LogEntry>();

        // 라우터: 로그 레벨에 따라 분배
        var router = Task.Run(async () =>
        {
            await foreach (var log in logChannel.Reader.ReadAllAsync(cts.Token))
            {
                var targetChannel = log.Level switch
                {
                    LogLevel.Debug => debugChannel,
                    LogLevel.Info => infoChannel,
                    LogLevel.Warning => warningChannel,
                    LogLevel.Error => errorChannel,
                    LogLevel.Critical => criticalChannel,
                    _ => throw new ArgumentException("Unknown log level")
                };

                await targetChannel.Writer.WriteAsync(log, cts.Token);
            }

            // 모든 채널 종료
            debugChannel.Writer.Complete();
            infoChannel.Writer.Complete();
            warningChannel.Writer.Complete();
            errorChannel.Writer.Complete();
            criticalChannel.Writer.Complete();
        });

        // 로그 처리기들
        var debugProcessor = ProcessLogs("DEBUG", debugChannel, ConsoleColor.Gray, cts.Token);
        var infoProcessor = ProcessLogs("INFO", infoChannel, ConsoleColor.White, cts.Token);
        var warningProcessor = ProcessLogs("WARNING", warningChannel, ConsoleColor.Yellow, cts.Token);
        var errorProcessor = ProcessLogs("ERROR", errorChannel, ConsoleColor.Red, cts.Token);
        var criticalProcessor = ProcessLogs("CRITICAL", criticalChannel, ConsoleColor.DarkRed, cts.Token);

        // 로그 생성 시뮬레이션
        var logGenerator = Task.Run(async () =>
        {
            var random = new Random();
            var sources = new[] { "WebAPI", "Database", "Cache", "FileSystem", "Network" };

            for (int i = 0; i < 100; i++)
            {
                var log = new LogEntry
                {
                    Timestamp = DateTime.Now,
                    Level = (LogLevel)random.Next(5),
                    Message = $"Log message {i}",
                    Source = sources[random.Next(sources.Length)]
                };

                await logChannel.Writer.WriteAsync(log, cts.Token);
                await Task.Delay(50, cts.Token);
            }

            logChannel.Writer.Complete();
        });

        // 10초 후 자동 종료
        _ = Task.Run(async () =>
        {
            await Task.Delay(10000);
            cts.Cancel();
        });

        await Task.WhenAll(logGenerator, router);
        await Task.WhenAll(debugProcessor, infoProcessor, warningProcessor,
                          errorProcessor, criticalProcessor);

        Console.WriteLine("\n로그 처리 완료!");
    }

    static async Task ProcessLogs(
        string processorName,
        Channel<LogEntry> channel,
        ConsoleColor color,
        CancellationToken ct)
    {
        var count = 0;

        try
        {
            await foreach (var log in channel.Reader.ReadAllAsync(ct))
            {
                count++;

                lock (Console.Out)
                {
                    var originalColor = Console.ForegroundColor;
                    Console.ForegroundColor = color;
                    Console.WriteLine($"[{processorName}] {log.Timestamp:HH:mm:ss.fff} " +
                        $"[{log.Source}] {log.Message}");
                    Console.ForegroundColor = originalColor;
                }

                // 중요 로그는 더 신중히 처리
                if (log.Level >= LogLevel.Error)
                {
                    await Task.Delay(100, ct);  // 알림, DB 저장 등
                }
            }
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine($"[{processorName}] 취소됨");
        }

        Console.WriteLine($"[{processorName}] 총 {count}개 로그 처리됨");
    }
}

// 출력 (색상 포함):
// [INFO] 14:32:15.123 [WebAPI] Log message 0
// [DEBUG] 14:32:15.174 [Cache] Log message 1
// [ERROR] 14:32:15.225 [Database] Log message 2
// [WARNING] 14:32:15.276 [Network] Log message 3
// ...
// [DEBUG] 총 23개 로그 처리됨
// [INFO] 총 19개 로그 처리됨
// [WARNING] 총 21개 로그 처리됨
// [ERROR] 총 18개 로그 처리됨
// [CRITICAL] 총 19개 로그 처리됨
//
// 로그 처리 완료!
```

## 6. Channels vs TPL Dataflow

### 비교

```csharp
using System;
using System.Diagnostics;
using System.Threading.Channels;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

class ChannelsVsDataflow
{
    const int MessageCount = 1_000_000;

    static async Task Main()
    {
        Console.WriteLine("=== 성능 비교: Channels vs TPL Dataflow ===\n");

        await TestChannels();
        await TestDataflow();
    }

    static async Task TestChannels()
    {
        var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(100)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleWriter = true,
            SingleReader = true
        });

        var sw = Stopwatch.StartNew();

        var producer = Task.Run(async () =>
        {
            for (int i = 0; i < MessageCount; i++)
            {
                await channel.Writer.WriteAsync(i);
            }
            channel.Writer.Complete();
        });

        var consumer = Task.Run(async () =>
        {
            var sum = 0L;
            await foreach (var item in channel.Reader.ReadAllAsync())
            {
                sum += item;
            }
        });

        await Task.WhenAll(producer, consumer);
        sw.Stop();

        Console.WriteLine($"Channels:        {sw.ElapsedMilliseconds,6}ms");
    }

    static async Task TestDataflow()
    {
        var actionBlock = new ActionBlock<int>(
            item => { var x = item; },  // 최소 처리
            new ExecutionDataflowBlockOptions
            {
                BoundedCapacity = 100,
                SingleProducerConstrained = true
            });

        var sw = Stopwatch.StartNew();

        for (int i = 0; i < MessageCount; i++)
        {
            await actionBlock.SendAsync(i);
        }

        actionBlock.Complete();
        await actionBlock.Completion;

        sw.Stop();

        Console.WriteLine($"TPL Dataflow:    {sw.ElapsedMilliseconds,6}ms");
    }
}

// 출력 예시:
// === 성능 비교: Channels vs TPL Dataflow ===
//
// Channels:           280ms
// TPL Dataflow:       420ms
//
// Channels가 약 1.5배 빠름!
```

### 선택 가이드

| 특징 | Channels | TPL Dataflow |
|------|----------|--------------|
| **성능** | 더 빠름 (~1.5배) | 느림 |
| **메모리** | 더 적음 | 더 많음 |
| **API 복잡도** | 단순 | 복잡 |
| **기능** | 기본적 | 풍부함 (변환, 배치, 조인 등) |
| **async/await 통합** | 완벽 | 양호 |
| **학습 곡선** | 낮음 | 높음 |
| **사용 사례** | 단순 큐, 생산자-소비자 | 복잡한 파이프라인 |
| **.NET 버전** | .NET Core 3.0+ | .NET Framework 4.5+ |

**권장 사항:**
- **단순한 생산자-소비자 패턴**: ✅ Channels 사용
- **복잡한 데이터 변환 파이프라인**: ✅ TPL Dataflow 사용
- **최고 성능 필요**: ✅ Channels 사용
- **풍부한 블록 타입 필요**: ✅ TPL Dataflow 사용

## 7. 고급 패턴

### 타임아웃 처리

```csharp
using System;
using System.Threading;
using System.Threading.Channels;
using System.Threading.Tasks;

class TimeoutPattern
{
    static async Task Main()
    {
        var channel = Channel.CreateUnbounded<string>();

        // 생산자
        var producer = Task.Run(async () =>
        {
            for (int i = 0; i < 5; i++)
            {
                await channel.Writer.WriteAsync($"Message {i}");
                await Task.Delay(1000);
            }
            channel.Writer.Complete();
        });

        // 소비자 (타임아웃 적용)
        var consumer = Task.Run(async () =>
        {
            using var cts = new CancellationTokenSource();

            try
            {
                await foreach (var message in channel.Reader.ReadAllAsync(cts.Token))
                {
                    Console.WriteLine($"수신: {message}");

                    // 각 메시지마다 2초 타임아웃
                    cts.CancelAfter(TimeSpan.FromSeconds(2));

                    await Task.Delay(500);  // 처리 시뮬레이션
                }
            }
            catch (OperationCanceledException)
            {
                Console.WriteLine("타임아웃! 메시지 수신 중단");
            }
        });

        await Task.WhenAll(producer, consumer);
    }
}

// 출력:
// 수신: Message 0
// 수신: Message 1
// 타임아웃! 메시지 수신 중단
```

### Priority Queue 패턴

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Channels;
using System.Threading.Tasks;

class PriorityQueuePattern
{
    class Message : IComparable<Message>
    {
        public int Priority { get; set; }
        public string Content { get; set; }

        public int CompareTo(Message other) =>
            other.Priority.CompareTo(Priority);  // 높은 우선순위가 먼저
    }

    static async Task Main()
    {
        // 우선순위 큐 (내부적으로 SortedSet 사용)
        var priorityQueue = new SortedSet<Message>();
        var lockObj = new object();
        var channel = Channel.CreateUnbounded<Message>();

        // 생산자
        var producer = Task.Run(async () =>
        {
            var messages = new[]
            {
                new Message { Priority = 1, Content = "Low priority" },
                new Message { Priority = 10, Content = "High priority" },
                new Message { Priority = 5, Content = "Medium priority" },
                new Message { Priority = 15, Content = "Critical!" },
                new Message { Priority = 3, Content = "Low-medium priority" }
            };

            foreach (var msg in messages)
            {
                await channel.Writer.WriteAsync(msg);
                Console.WriteLine($"생산: [{msg.Priority}] {msg.Content}");
                await Task.Delay(100);
            }

            channel.Writer.Complete();
        });

        // 우선순위 정렬기
        var sorter = Task.Run(async () =>
        {
            await foreach (var message in channel.Reader.ReadAllAsync())
            {
                lock (lockObj)
                {
                    priorityQueue.Add(message);
                }
            }
        });

        await Task.WhenAll(producer, sorter);

        // 소비자 (우선순위 순서대로 처리)
        Console.WriteLine("\n우선순위 순서대로 처리:");
        foreach (var message in priorityQueue)
        {
            Console.WriteLine($"처리: [{message.Priority}] {message.Content}");
        }
    }
}

// 출력:
// 생산: [1] Low priority
// 생산: [10] High priority
// 생산: [5] Medium priority
// 생산: [15] Critical!
// 생산: [3] Low-medium priority
//
// 우선순위 순서대로 처리:
// 처리: [15] Critical!
// 처리: [10] High priority
// 처리: [5] Medium priority
// 처리: [3] Low-medium priority
// 처리: [1] Low priority
```

### Batching 패턴

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Threading;
using System.Threading.Channels;
using System.Threading.Tasks;

class BatchingPattern
{
    static async Task Main()
    {
        var inputChannel = Channel.CreateUnbounded<int>();
        var batchChannel = Channel.CreateUnbounded<List<int>>();

        const int batchSize = 5;
        var batchTimeout = TimeSpan.FromSeconds(2);

        // 생산자
        var producer = Task.Run(async () =>
        {
            for (int i = 0; i < 23; i++)
            {
                await inputChannel.Writer.WriteAsync(i);
                Console.WriteLine($"생산: {i}");
                await Task.Delay(300);
            }
            inputChannel.Writer.Complete();
        });

        // 배치 생성기
        var batcher = Task.Run(async () =>
        {
            var batch = new List<int>();
            var sw = Stopwatch.StartNew();

            await foreach (var item in inputChannel.Reader.ReadAllAsync())
            {
                batch.Add(item);

                // 배치 크기 도달 또는 타임아웃
                if (batch.Count >= batchSize || sw.Elapsed >= batchTimeout)
                {
                    if (batch.Count > 0)
                    {
                        await batchChannel.Writer.WriteAsync(new List<int>(batch));
                        Console.WriteLine($"배치 생성: {batch.Count}개 항목");
                        batch.Clear();
                        sw.Restart();
                    }
                }
            }

            // 마지막 불완전한 배치 전송
            if (batch.Count > 0)
            {
                await batchChannel.Writer.WriteAsync(batch);
                Console.WriteLine($"배치 생성 (마지막): {batch.Count}개 항목");
            }

            batchChannel.Writer.Complete();
        });

        // 소비자: 배치 처리
        var consumer = Task.Run(async () =>
        {
            int batchNumber = 1;
            await foreach (var batch in batchChannel.Reader.ReadAllAsync())
            {
                Console.WriteLine($"배치 #{batchNumber} 처리: [{string.Join(", ", batch)}]");
                await Task.Delay(500);  // 배치 처리 시뮬레이션
                batchNumber++;
            }
        });

        await Task.WhenAll(producer, batcher, consumer);
    }
}

// 출력:
// 생산: 0
// 생산: 1
// 생산: 2
// 생산: 3
// 생산: 4
// 배치 생성: 5개 항목
// 배치 #1 처리: [0, 1, 2, 3, 4]
// 생산: 5
// ...
// 배치 생성 (마지막): 3개 항목
// 배치 #5 처리: [20, 21, 22]
```

## 8. 모범 사례

### ✅ 권장 사항

```csharp
// 1. async/await와 함께 사용
await foreach (var item in channel.Reader.ReadAllAsync())
{
    await ProcessAsync(item);
}

// 2. Complete() 항상 호출
try
{
    // 데이터 쓰기
}
finally
{
    channel.Writer.Complete();
}

// 3. 백프레셔 적용 (메모리 관리)
var channel = Channel.CreateBounded<T>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait
});

// 4. SingleWriter/SingleReader 최적화
var channel = Channel.CreateBounded<T>(new BoundedChannelOptions(100)
{
    SingleWriter = true,
    SingleReader = true
});

// 5. CancellationToken 사용
await foreach (var item in channel.Reader.ReadAllAsync(cancellationToken))
{
    // 처리
}
```

### ❌ 피해야 할 패턴

```csharp
// ❌ 1. Complete() 호출 누락
// reader.ReadAllAsync()가 영원히 대기!

// ❌ 2. 무제한 채널에서 빠른 생산자
var channel = Channel.CreateUnbounded<T>();
// 메모리 폭발 위험!

// ❌ 3. TryWrite 결과 무시
channel.Writer.TryWrite(item);  // 실패 가능성 무시!

// ❌ 4. 동기 블로킹
var item = channel.Reader.ReadAsync().Result;  // 데드락 위험!

// ❌ 5. 예외 처리 없음
await foreach (var item in channel.Reader.ReadAllAsync())
{
    // 예외 발생 시 채널 상태 불확실
}
```

## 9. 성능 최적화

### 벤치마크

```csharp
using System;
using System.Diagnostics;
using System.Threading.Channels;
using System.Threading.Tasks;

class ChannelBenchmark
{
    const int MessageCount = 10_000_000;

    static async Task Main()
    {
        Console.WriteLine("=== Channel 성능 벤치마크 ===\n");

        await BenchmarkUnbounded();
        await BenchmarkBoundedWait();
        await BenchmarkBoundedDrop();
        await BenchmarkOptimized();
    }

    static async Task BenchmarkUnbounded()
    {
        var channel = Channel.CreateUnbounded<int>();
        await RunBenchmark("Unbounded", channel);
    }

    static async Task BenchmarkBoundedWait()
    {
        var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(1000)
        {
            FullMode = BoundedChannelFullMode.Wait
        });
        await RunBenchmark("Bounded (Wait)", channel);
    }

    static async Task BenchmarkBoundedDrop()
    {
        var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(1000)
        {
            FullMode = BoundedChannelFullMode.DropOldest
        });
        await RunBenchmark("Bounded (DropOldest)", channel);
    }

    static async Task BenchmarkOptimized()
    {
        var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(1000)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleWriter = true,
            SingleReader = true
        });
        await RunBenchmark("Bounded (Optimized)", channel);
    }

    static async Task RunBenchmark(string name, Channel<int> channel)
    {
        var sw = Stopwatch.StartNew();

        var producer = Task.Run(async () =>
        {
            for (int i = 0; i < MessageCount; i++)
            {
                await channel.Writer.WriteAsync(i);
            }
            channel.Writer.Complete();
        });

        var consumer = Task.Run(async () =>
        {
            var sum = 0L;
            await foreach (var item in channel.Reader.ReadAllAsync())
            {
                sum += item;
            }
        });

        await Task.WhenAll(producer, consumer);
        sw.Stop();

        var throughput = MessageCount / sw.Elapsed.TotalSeconds;
        Console.WriteLine($"{name,-25} {sw.ElapsedMilliseconds,6}ms  " +
            $"({throughput / 1_000_000:F2}M msgs/sec)");
    }
}

// 출력 예시:
// === Channel 성능 벤치마크 ===
//
// Unbounded                   1250ms  (8.00M msgs/sec)
// Bounded (Wait)              1180ms  (8.47M msgs/sec)
// Bounded (DropOldest)         980ms  (10.20M msgs/sec)
// Bounded (Optimized)          820ms  (12.20M msgs/sec)
```

## 10. 요약

### 핵심 포인트

1. **고성능**: TPL Dataflow보다 빠르고 가벼움
2. **async/await 통합**: 완벽한 비동기 지원
3. **백프레셔**: 자동 흐름 제어로 메모리 관리
4. **단순한 API**: 배우기 쉽고 사용하기 간편
5. **유연성**: 다중 생산자/소비자, Fan-Out/Fan-In 등 다양한 패턴
6. **최적화**: SingleWriter/SingleReader로 성능 향상
7. **안정성**: Bounded 채널로 메모리 폭발 방지

### 빠른 참조

```csharp
// 무제한 채널
var channel = Channel.CreateUnbounded<T>();

// 제한된 채널 (권장)
var channel = Channel.CreateBounded<T>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait,
    SingleWriter = true,
    SingleReader = true
});

// 쓰기
await channel.Writer.WriteAsync(item);
channel.Writer.TryWrite(item);
channel.Writer.Complete();

// 읽기
await foreach (var item in channel.Reader.ReadAllAsync())
{
    // 처리
}

// 또는
while (await channel.Reader.WaitToReadAsync())
{
    if (channel.Reader.TryRead(out var item))
    {
        // 처리
    }
}
```

### 언제 사용할까?

✅ **Channels 사용:**
- 단순한 생산자-소비자 패턴
- 최고 성능 필요
- async/await 중심 코드
- 경량 솔루션 선호

✅ **TPL Dataflow 사용:**
- 복잡한 파이프라인 (변환, 배치, 조인 등)
- 풍부한 블록 타입 필요
- 기존 Dataflow 코드베이스

## 참고 자료

- [Microsoft Docs: System.Threading.Channels](https://docs.microsoft.com/en-us/dotnet/api/system.threading.channels)
- [An Introduction to System.Threading.Channels](https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/)
- [Channel<T> Class](https://docs.microsoft.com/en-us/dotnet/api/system.threading.channels.channel-1)
