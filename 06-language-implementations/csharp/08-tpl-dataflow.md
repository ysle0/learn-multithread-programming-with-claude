# TPL Dataflow

## 📌 개요

TPL Dataflow는 동시성이 높은 애플리케이션을 위한 강력한 라이브러리로, **액터 모델(Actor Model)**과 **데이터 흐름 프로그래밍**을 .NET에 구현한 것입니다. 복잡한 데이터 처리 파이프라인을 블록 단위로 구성할 수 있으며, 각 블록은 독립적으로 실행되고 메시지를 전달합니다.

**설치:**
```bash
dotnet add package System.Threading.Tasks.Dataflow
```

**핵심 개념:**
- **블록(Blocks)**: 데이터를 처리하는 독립적인 단위
- **링크(Links)**: 블록 간 데이터 흐름 연결
- **버퍼링(Buffering)**: 자동 큐 관리
- **병렬화(Parallelization)**: 자동 병렬 처리
- **백프레셔(Backpressure)**: 자동 흐름 제어

## 1. 기본 블록 타입

### ActionBlock<T> - 데이터 소비

```csharp
using System;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

class ActionBlockExample
{
    static async Task Main()
    {
        // 간단한 ActionBlock
        var actionBlock = new ActionBlock<int>(number =>
        {
            Console.WriteLine($"처리 중: {number} (Thread: {Thread.CurrentThread.ManagedThreadId})");
            Thread.Sleep(100);  // 처리 시뮬레이션
        });

        // 데이터 게시
        for (int i = 0; i < 10; i++)
        {
            await actionBlock.SendAsync(i);
        }

        // 입력 완료 신호
        actionBlock.Complete();

        // 모든 처리 완료 대기
        await actionBlock.Completion;
        Console.WriteLine("모든 작업 완료!");
    }
}

// 출력:
// 처리 중: 0 (Thread: 4)
// 처리 중: 1 (Thread: 4)
// ...
// 모든 작업 완료!
```

### TransformBlock<TInput, TOutput> - 데이터 변환

```csharp
using System;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

class TransformBlockExample
{
    static async Task Main()
    {
        // 입력을 제곱으로 변환
        var transformBlock = new TransformBlock<int, int>(number =>
        {
            Console.WriteLine($"변환 중: {number} → {number * number}");
            return number * number;
        });

        // 결과 소비
        var actionBlock = new ActionBlock<int>(result =>
        {
            Console.WriteLine($"결과: {result}");
        });

        // 블록 연결
        transformBlock.LinkTo(actionBlock, new DataflowLinkOptions
        {
            PropagateCompletion = true  // 완료 신호 전파
        });

        // 데이터 전송
        for (int i = 1; i <= 5; i++)
        {
            await transformBlock.SendAsync(i);
        }

        transformBlock.Complete();
        await actionBlock.Completion;
    }
}

// 출력:
// 변환 중: 1 → 1
// 결과: 1
// 변환 중: 2 → 4
// 결과: 4
// ...
```

### BufferBlock<T> - 데이터 큐

```csharp
using System;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

class BufferBlockExample
{
    static async Task Main()
    {
        var bufferBlock = new BufferBlock<string>();

        // 생산자: 데이터 추가
        var producer = Task.Run(async () =>
        {
            for (int i = 0; i < 5; i++)
            {
                var message = $"Message {i}";
                await bufferBlock.SendAsync(message);
                Console.WriteLine($"생산: {message}");
                await Task.Delay(500);
            }
            bufferBlock.Complete();
        });

        // 소비자: 데이터 읽기
        var consumer = Task.Run(async () =>
        {
            while (await bufferBlock.OutputAvailableAsync())
            {
                var message = await bufferBlock.ReceiveAsync();
                Console.WriteLine($"소비: {message}");
                await Task.Delay(1000);
            }
        });

        await Task.WhenAll(producer, consumer);
    }
}

// 출력:
// 생산: Message 0
// 소비: Message 0
// 생산: Message 1
// 생산: Message 2
// 소비: Message 1
// ...
```

### BroadcastBlock<T> - 데이터 브로드캐스트

```csharp
using System;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

class BroadcastBlockExample
{
    static async Task Main()
    {
        var broadcastBlock = new BroadcastBlock<int>(x => x);

        // 여러 소비자 연결
        var consumer1 = new ActionBlock<int>(x =>
            Console.WriteLine($"소비자1: {x}"));

        var consumer2 = new ActionBlock<int>(x =>
            Console.WriteLine($"소비자2: {x}"));

        var consumer3 = new ActionBlock<int>(x =>
            Console.WriteLine($"소비자3: {x}"));

        broadcastBlock.LinkTo(consumer1, new DataflowLinkOptions { PropagateCompletion = true });
        broadcastBlock.LinkTo(consumer2, new DataflowLinkOptions { PropagateCompletion = true });
        broadcastBlock.LinkTo(consumer3, new DataflowLinkOptions { PropagateCompletion = true });

        // 데이터 브로드캐스트
        for (int i = 0; i < 3; i++)
        {
            await broadcastBlock.SendAsync(i);
            await Task.Delay(100);  // 출력 확인용
        }

        broadcastBlock.Complete();
        await Task.WhenAll(
            consumer1.Completion,
            consumer2.Completion,
            consumer3.Completion);
    }
}

// 출력:
// 소비자1: 0
// 소비자2: 0
// 소비자3: 0
// 소비자1: 1
// 소비자2: 1
// 소비자3: 1
// ...
```

### BatchBlock<T> - 데이터 일괄 처리

```csharp
using System;
using System.Linq;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

class BatchBlockExample
{
    static async Task Main()
    {
        // 3개씩 묶어서 처리
        var batchBlock = new BatchBlock<int>(batchSize: 3);

        var actionBlock = new ActionBlock<int[]>(batch =>
        {
            Console.WriteLine($"배치 처리: [{string.Join(", ", batch)}] (크기: {batch.Length})");
        });

        batchBlock.LinkTo(actionBlock, new DataflowLinkOptions
        {
            PropagateCompletion = true
        });

        // 10개 데이터 전송
        for (int i = 1; i <= 10; i++)
        {
            await batchBlock.SendAsync(i);
        }

        // 완료 신호 (마지막 불완전한 배치도 전송됨)
        batchBlock.Complete();
        await actionBlock.Completion;
    }
}

// 출력:
// 배치 처리: [1, 2, 3] (크기: 3)
// 배치 처리: [4, 5, 6] (크기: 3)
// 배치 처리: [7, 8, 9] (크기: 3)
// 배치 처리: [10] (크기: 1)  // 마지막 불완전한 배치
```

### JoinBlock<T1, T2> - 데이터 조인

```csharp
using System;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

class JoinBlockExample
{
    static async Task Main()
    {
        // 두 개의 입력을 조인
        var joinBlock = new JoinBlock<string, int>();

        var actionBlock = new ActionBlock<Tuple<string, int>>(tuple =>
        {
            Console.WriteLine($"조인 결과: {tuple.Item1} = {tuple.Item2}");
        });

        joinBlock.LinkTo(actionBlock, new DataflowLinkOptions
        {
            PropagateCompletion = true
        });

        // 비동기로 데이터 전송
        var task1 = Task.Run(async () =>
        {
            await joinBlock.Target1.SendAsync("Alpha");
            await Task.Delay(100);
            await joinBlock.Target1.SendAsync("Beta");
            await Task.Delay(100);
            await joinBlock.Target1.SendAsync("Gamma");
            joinBlock.Target1.Complete();
        });

        var task2 = Task.Run(async () =>
        {
            await Task.Delay(50);
            await joinBlock.Target2.SendAsync(1);
            await Task.Delay(100);
            await joinBlock.Target2.SendAsync(2);
            await Task.Delay(100);
            await joinBlock.Target2.SendAsync(3);
            joinBlock.Target2.Complete();
        });

        await Task.WhenAll(task1, task2);
        await actionBlock.Completion;
    }
}

// 출력:
// 조인 결과: Alpha = 1
// 조인 결과: Beta = 2
// 조인 결과: Gamma = 3
```

## 2. 실전 예제: 이미지 처리 파이프라인

```csharp
using System;
using System.Diagnostics;
using System.Drawing;
using System.Drawing.Imaging;
using System.IO;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

class ImageProcessingPipeline
{
    class ImageData
    {
        public string FilePath { get; set; }
        public Image Image { get; set; }
        public Image Thumbnail { get; set; }
    }

    static async Task Main()
    {
        var sw = Stopwatch.StartNew();

        // 1. 이미지 로드 블록
        var loadBlock = new TransformBlock<string, ImageData>(
            async filePath =>
            {
                Console.WriteLine($"로드 중: {Path.GetFileName(filePath)}");
                await Task.Delay(100);  // I/O 시뮬레이션
                return new ImageData
                {
                    FilePath = filePath,
                    Image = Image.FromFile(filePath)
                };
            },
            new ExecutionDataflowBlockOptions
            {
                MaxDegreeOfParallelism = 4  // 병렬 로드
            });

        // 2. 썸네일 생성 블록
        var thumbnailBlock = new TransformBlock<ImageData, ImageData>(
            data =>
            {
                Console.WriteLine($"썸네일 생성: {Path.GetFileName(data.FilePath)}");
                data.Thumbnail = data.Image.GetThumbnailImage(200, 200, null, IntPtr.Zero);
                return data;
            },
            new ExecutionDataflowBlockOptions
            {
                MaxDegreeOfParallelism = 8  // CPU 집약적, 더 많은 병렬화
            });

        // 3. 워터마크 추가 블록
        var watermarkBlock = new TransformBlock<ImageData, ImageData>(
            data =>
            {
                Console.WriteLine($"워터마크 추가: {Path.GetFileName(data.FilePath)}");
                using (var graphics = Graphics.FromImage(data.Thumbnail))
                {
                    graphics.DrawString(
                        "© 2026",
                        new Font("Arial", 12),
                        Brushes.White,
                        new PointF(10, 10));
                }
                return data;
            },
            new ExecutionDataflowBlockOptions
            {
                MaxDegreeOfParallelism = 4
            });

        // 4. 저장 블록
        var saveBlock = new ActionBlock<ImageData>(
            async data =>
            {
                var outputPath = Path.Combine(
                    @"C:\Output",
                    Path.GetFileName(data.FilePath));

                Console.WriteLine($"저장 중: {Path.GetFileName(data.FilePath)}");
                data.Thumbnail.Save(outputPath, ImageFormat.Jpeg);

                // 리소스 정리
                data.Thumbnail.Dispose();
                data.Image.Dispose();

                await Task.Delay(50);  // I/O 시뮬레이션
            },
            new ExecutionDataflowBlockOptions
            {
                MaxDegreeOfParallelism = 2  // I/O 제한
            });

        // 파이프라인 연결
        var linkOptions = new DataflowLinkOptions { PropagateCompletion = true };
        loadBlock.LinkTo(thumbnailBlock, linkOptions);
        thumbnailBlock.LinkTo(watermarkBlock, linkOptions);
        watermarkBlock.LinkTo(saveBlock, linkOptions);

        // 이미지 파일 목록
        var imageFiles = Directory.GetFiles(@"C:\Images", "*.jpg");
        Console.WriteLine($"총 {imageFiles.Length}개 이미지 처리 시작\n");

        // 파이프라인에 데이터 전송
        foreach (var file in imageFiles)
        {
            await loadBlock.SendAsync(file);
        }

        // 파이프라인 완료 대기
        loadBlock.Complete();
        await saveBlock.Completion;

        sw.Stop();
        Console.WriteLine($"\n완료! 소요 시간: {sw.Elapsed.TotalSeconds:F2}초");
    }
}

// 출력 예시 (100개 이미지):
// 총 100개 이미지 처리 시작
//
// 로드 중: image001.jpg
// 로드 중: image002.jpg
// 로드 중: image003.jpg
// 로드 중: image004.jpg
// 썸네일 생성: image001.jpg
// 썸네일 생성: image002.jpg
// ...
// 워터마크 추가: image001.jpg
// 저장 중: image001.jpg
// ...
//
// 완료! 소요 시간: 8.45초
// (순차 처리 시 약 60초)
```

## 3. 실전 예제: 웹 크롤러

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net.Http;
using System.Text.RegularExpressions;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

class WebCrawler
{
    static readonly HttpClient client = new HttpClient();
    static readonly HashSet<string> visitedUrls = new HashSet<string>();
    static readonly object lockObj = new object();

    class CrawlResult
    {
        public string Url { get; set; }
        public string Content { get; set; }
        public List<string> Links { get; set; }
    }

    static async Task Main()
    {
        var startUrl = "https://example.com";
        int maxDepth = 3;

        // 1. URL 다운로드 블록
        var downloadBlock = new TransformBlock<(string Url, int Depth), CrawlResult>(
            async tuple =>
            {
                var (url, depth) = tuple;

                // 방문 확인
                lock (lockObj)
                {
                    if (visitedUrls.Contains(url))
                        return null;
                    visitedUrls.Add(url);
                }

                try
                {
                    Console.WriteLine($"다운로드 중 (깊이 {depth}): {url}");
                    var content = await client.GetStringAsync(url);

                    return new CrawlResult
                    {
                        Url = url,
                        Content = content,
                        Links = ExtractLinks(content, url)
                    };
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"오류 ({url}): {ex.Message}");
                    return null;
                }
            },
            new ExecutionDataflowBlockOptions
            {
                MaxDegreeOfParallelism = 10  // 동시 다운로드 수
            });

        // 2. 링크 추출 및 필터링 블록
        var extractBlock = new TransformManyBlock<CrawlResult, (string, int)>(
            result =>
            {
                if (result == null) return Enumerable.Empty<(string, int)>();

                Console.WriteLine($"링크 추출: {result.Url} → {result.Links.Count}개 발견");

                // 다음 깊이로 링크 전달
                return result.Links.Select(link => (link, result.Depth + 1));
            });

        // 3. 깊이 필터 블록
        var filterBlock = new TransformBlock<(string Url, int Depth), (string, int)>(
            tuple =>
            {
                var (url, depth) = tuple;
                if (depth > maxDepth)
                {
                    Console.WriteLine($"깊이 제한 초과: {url} (깊이 {depth})");
                    return default;
                }
                return tuple;
            });

        // 4. 데이터 저장 블록
        var saveBlock = new ActionBlock<CrawlResult>(
            result =>
            {
                if (result == null) return;

                // 데이터베이스나 파일에 저장
                Console.WriteLine($"저장: {result.Url} ({result.Content.Length} bytes)");
            },
            new ExecutionDataflowBlockOptions
            {
                MaxDegreeOfParallelism = 4
            });

        // 파이프라인 연결
        var linkOptions = new DataflowLinkOptions { PropagateCompletion = false };

        downloadBlock.LinkTo(extractBlock, linkOptions, result => result != null);
        downloadBlock.LinkTo(saveBlock, linkOptions, result => result != null);

        extractBlock.LinkTo(filterBlock, linkOptions);
        filterBlock.LinkTo(downloadBlock, linkOptions, tuple => tuple != default);

        // 크롤링 시작
        await downloadBlock.SendAsync((startUrl, 0));

        // 완료 대기 (실제로는 타임아웃이나 조건 필요)
        await Task.Delay(10000);

        downloadBlock.Complete();
        await saveBlock.Completion;

        Console.WriteLine($"\n크롤링 완료! 총 {visitedUrls.Count}개 URL 방문");
    }

    static List<string> ExtractLinks(string html, string baseUrl)
    {
        var links = new List<string>();
        var regex = new Regex(@"href=""(https?://[^""]+)""", RegexOptions.IgnoreCase);

        foreach (Match match in regex.Matches(html))
        {
            var url = match.Groups[1].Value;
            if (Uri.IsWellFormedUriString(url, UriKind.Absolute))
            {
                links.Add(url);
            }
        }

        return links.Take(10).ToList();  // 링크 수 제한
    }
}

// 출력 예시:
// 다운로드 중 (깊이 0): https://example.com
// 저장: https://example.com (1256 bytes)
// 링크 추출: https://example.com → 8개 발견
// 다운로드 중 (깊이 1): https://example.com/about
// 다운로드 중 (깊이 1): https://example.com/contact
// ...
// 크롤링 완료! 총 47개 URL 방문
```

## 4. 고급 옵션

### ExecutionDataflowBlockOptions

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

class AdvancedOptionsExample
{
    static async Task Main()
    {
        var cts = new CancellationTokenSource();

        var options = new ExecutionDataflowBlockOptions
        {
            // 병렬화 수준
            MaxDegreeOfParallelism = 4,

            // 버퍼 크기 제한 (백프레셔)
            BoundedCapacity = 100,

            // 취소 토큰
            CancellationToken = cts.Token,

            // 작업 스케줄러
            TaskScheduler = TaskScheduler.Default,

            // 단일 생산자 최적화
            SingleProducerConstrained = false,

            // 메시지당 최대 메시지 수
            MaxMessagesPerTask = 1,

            // 이름 (디버깅용)
            NameFormat = "MyBlock {0}"
        };

        var actionBlock = new ActionBlock<int>(
            async number =>
            {
                Console.WriteLine($"처리 중: {number}");
                await Task.Delay(500);
            },
            options);

        // 데이터 전송
        for (int i = 0; i < 10; i++)
        {
            var sent = await actionBlock.SendAsync(i);
            if (!sent)
            {
                Console.WriteLine($"전송 실패: {i} (버퍼 가득참)");
            }
        }

        // 5초 후 취소
        cts.CancelAfter(5000);

        actionBlock.Complete();

        try
        {
            await actionBlock.Completion;
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("작업 취소됨");
        }
    }
}
```

### 백프레셔(Backpressure) 관리

```csharp
using System;
using System.Diagnostics;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

class BackpressureExample
{
    static async Task Main()
    {
        // 빠른 생산자, 느린 소비자 시나리오

        // ❌ 버퍼 무제한 (메모리 폭발 가능)
        Console.WriteLine("=== 버퍼 무제한 ===");
        await TestUnbounded();

        Console.WriteLine("\n=== 버퍼 제한 (백프레셔) ===");
        await TestBounded();
    }

    static async Task TestUnbounded()
    {
        var actionBlock = new ActionBlock<int>(
            async number =>
            {
                await Task.Delay(100);  // 느린 처리
            });

        var sw = Stopwatch.StartNew();

        // 빠르게 1000개 전송 (모두 버퍼에 쌓임)
        for (int i = 0; i < 1000; i++)
        {
            await actionBlock.SendAsync(i);
        }

        sw.Stop();
        Console.WriteLine($"전송 완료: {sw.ElapsedMilliseconds}ms");
        Console.WriteLine($"버퍼링된 항목: ~1000개 (메모리 사용량 증가!)");

        actionBlock.Complete();
        await actionBlock.Completion;
    }

    static async Task TestBounded()
    {
        var actionBlock = new ActionBlock<int>(
            async number =>
            {
                await Task.Delay(100);  // 느린 처리
            },
            new ExecutionDataflowBlockOptions
            {
                BoundedCapacity = 10  // 최대 10개만 버퍼링
            });

        var sw = Stopwatch.StartNew();

        // 전송 시 버퍼가 가득 차면 대기 (자동 백프레셔)
        for (int i = 0; i < 100; i++)
        {
            await actionBlock.SendAsync(i);  // 버퍼 가득 차면 블로킹
            if (i % 10 == 0)
            {
                Console.WriteLine($"{i}개 전송 완료 (경과: {sw.ElapsedMilliseconds}ms)");
            }
        }

        sw.Stop();
        Console.WriteLine($"전송 완료: {sw.ElapsedMilliseconds}ms");
        Console.WriteLine($"메모리 사용량: 제한됨 (최대 10개 버퍼링)");

        actionBlock.Complete();
        await actionBlock.Completion;
    }
}

// 출력:
// === 버퍼 무제한 ===
// 전송 완료: 15ms
// 버퍼링된 항목: ~1000개 (메모리 사용량 증가!)
//
// === 버퍼 제한 (백프레셔) ===
// 0개 전송 완료 (경과: 0ms)
// 10개 전송 완료 (경과: 1050ms)
// 20개 전송 완료 (경과: 2100ms)
// ...
// 전송 완료: 10500ms
// 메모리 사용량: 제한됨 (최대 10개 버퍼링)
```

### 조건부 링크

```csharp
using System;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

class ConditionalLinkExample
{
    static async Task Main()
    {
        var inputBlock = new BufferBlock<int>();

        // 짝수 처리 블록
        var evenBlock = new ActionBlock<int>(x =>
            Console.WriteLine($"짝수: {x}"));

        // 홀수 처리 블록
        var oddBlock = new ActionBlock<int>(x =>
            Console.WriteLine($"홀수: {x}"));

        // 큰 수 처리 블록
        var largeBlock = new ActionBlock<int>(x =>
            Console.WriteLine($"큰 수 (>50): {x}"));

        // 조건부 링크
        inputBlock.LinkTo(evenBlock, new DataflowLinkOptions
        {
            PropagateCompletion = true
        }, x => x % 2 == 0 && x <= 50);

        inputBlock.LinkTo(oddBlock, new DataflowLinkOptions
        {
            PropagateCompletion = true
        }, x => x % 2 != 0 && x <= 50);

        inputBlock.LinkTo(largeBlock, new DataflowLinkOptions
        {
            PropagateCompletion = true
        }, x => x > 50);

        // Null 타겟 (매칭되지 않는 항목 버림)
        inputBlock.LinkTo(DataflowBlock.NullTarget<int>());

        // 데이터 전송
        for (int i = 0; i < 100; i += 7)
        {
            await inputBlock.SendAsync(i);
        }

        inputBlock.Complete();
        await Task.WhenAll(
            evenBlock.Completion,
            oddBlock.Completion,
            largeBlock.Completion);
    }
}

// 출력:
// 짝수: 0
// 홀수: 7
// 짝수: 14
// 홀수: 21
// 짝수: 28
// 홀수: 35
// 짝수: 42
// 홀수: 49
// 큰 수 (>50): 56
// 큰 수 (>50): 63
// ...
```

## 5. 커스텀 블록

```csharp
using System;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

class CustomBlockExample
{
    // 커스텀 블록: 입력을 두 배로 만들고 필터링
    static IPropagatorBlock<int, int> CreateDoubleAndFilterBlock()
    {
        var transformBlock = new TransformBlock<int, int>(x => x * 2);
        var filterBlock = new TransformBlock<int, int>(x =>
        {
            if (x > 100)
            {
                Console.WriteLine($"필터링됨: {x}");
                return -1;  // 센티넬 값
            }
            return x;
        });

        var outputBlock = new BufferBlock<int>();

        // 내부 연결
        transformBlock.LinkTo(filterBlock, new DataflowLinkOptions
        {
            PropagateCompletion = true
        });

        filterBlock.LinkTo(outputBlock, new DataflowLinkOptions
        {
            PropagateCompletion = true
        }, x => x != -1);

        // Null 타겟으로 필터링된 항목 버림
        filterBlock.LinkTo(DataflowBlock.NullTarget<int>());

        // 커스텀 블록 래핑
        return DataflowBlock.Encapsulate(transformBlock, outputBlock);
    }

    static async Task Main()
    {
        var customBlock = CreateDoubleAndFilterBlock();

        var printBlock = new ActionBlock<int>(x =>
            Console.WriteLine($"출력: {x}"));

        customBlock.LinkTo(printBlock, new DataflowLinkOptions
        {
            PropagateCompletion = true
        });

        // 테스트 데이터
        for (int i = 0; i < 100; i += 10)
        {
            await customBlock.SendAsync(i);
        }

        customBlock.Complete();
        await printBlock.Completion;
    }
}

// 출력:
// 출력: 0
// 출력: 20
// 출력: 40
// 출력: 60
// 출력: 80
// 출력: 100
// 필터링됨: 120
// 필터링됨: 140
// 필터링됨: 160
// 필터링됨: 180
```

## 6. 성능 모니터링

```csharp
using System;
using System.Diagnostics;
using System.Threading;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

class PerformanceMonitoring
{
    class MonitoredBlock<T>
    {
        private readonly ActionBlock<T> _block;
        private long _processedCount;
        private long _totalProcessingTime;
        private readonly Stopwatch _overallTimer = Stopwatch.StartNew();

        public MonitoredBlock(Func<T, Task> action, ExecutionDataflowBlockOptions options)
        {
            _block = new ActionBlock<T>(async item =>
            {
                var sw = Stopwatch.StartNew();
                await action(item);
                sw.Stop();

                Interlocked.Increment(ref _processedCount);
                Interlocked.Add(ref _totalProcessingTime, sw.ElapsedMilliseconds);
            }, options);
        }

        public Task<bool> SendAsync(T item) => _block.SendAsync(item);
        public void Complete() => _block.Complete();
        public Task Completion => _block.Completion;

        public void PrintStatistics()
        {
            _overallTimer.Stop();

            Console.WriteLine("\n=== 성능 통계 ===");
            Console.WriteLine($"총 처리 항목: {_processedCount}");
            Console.WriteLine($"총 실행 시간: {_overallTimer.ElapsedMilliseconds}ms");
            Console.WriteLine($"평균 처리 시간: {_totalProcessingTime / (double)_processedCount:F2}ms");
            Console.WriteLine($"처리율: {_processedCount / _overallTimer.Elapsed.TotalSeconds:F2} items/sec");
            Console.WriteLine($"병렬 효율성: {(_totalProcessingTime / (double)_overallTimer.ElapsedMilliseconds):F2}x");
        }
    }

    static async Task Main()
    {
        var monitoredBlock = new MonitoredBlock<int>(
            async item =>
            {
                await Task.Delay(100);  // 처리 시뮬레이션
                Console.WriteLine($"처리됨: {item}");
            },
            new ExecutionDataflowBlockOptions
            {
                MaxDegreeOfParallelism = 4
            });

        // 데이터 전송
        for (int i = 0; i < 40; i++)
        {
            await monitoredBlock.SendAsync(i);
        }

        monitoredBlock.Complete();
        await monitoredBlock.Completion;

        monitoredBlock.PrintStatistics();
    }
}

// 출력:
// 처리됨: 0
// 처리됨: 1
// ...
// 처리됨: 39
//
// === 성능 통계 ===
// 총 처리 항목: 40
// 총 실행 시간: 1050ms
// 평균 처리 시간: 100.25ms
// 처리율: 38.10 items/sec
// 병렬 효율성: 3.81x  // 4배 병렬화 중 3.81배 달성
```

## 7. 오류 처리 패턴

```csharp
using System;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

class ErrorHandlingPatterns
{
    static async Task Main()
    {
        await DemoRetryPattern();
        await DemoCircuitBreakerPattern();
    }

    // 패턴 1: 재시도 (Retry)
    static async Task DemoRetryPattern()
    {
        Console.WriteLine("=== 재시도 패턴 ===");

        var processBlock = new TransformBlock<int, int>(
            async number =>
            {
                const int maxRetries = 3;
                int attempt = 0;

                while (attempt < maxRetries)
                {
                    try
                    {
                        return await ProcessWithFailure(number);
                    }
                    catch (Exception ex)
                    {
                        attempt++;
                        if (attempt >= maxRetries)
                        {
                            Console.WriteLine($"최대 재시도 초과: {number}");
                            throw;
                        }

                        Console.WriteLine($"재시도 {attempt}/{maxRetries}: {number}");
                        await Task.Delay(100 * attempt);  // 지수 백오프
                    }
                }

                throw new InvalidOperationException("Unreachable");
            },
            new ExecutionDataflowBlockOptions
            {
                MaxDegreeOfParallelism = 4
            });

        var outputBlock = new ActionBlock<int>(result =>
            Console.WriteLine($"성공: {result}"));

        processBlock.LinkTo(outputBlock, new DataflowLinkOptions
        {
            PropagateCompletion = true
        });

        for (int i = 0; i < 10; i++)
        {
            await processBlock.SendAsync(i);
        }

        processBlock.Complete();

        try
        {
            await outputBlock.Completion;
        }
        catch (Exception ex)
        {
            Console.WriteLine($"파이프라인 오류: {ex.Message}");
        }
    }

    // 패턴 2: 회로 차단기 (Circuit Breaker)
    static async Task DemoCircuitBreakerPattern()
    {
        Console.WriteLine("\n=== 회로 차단기 패턴 ===");

        var circuitBreaker = new CircuitBreaker(failureThreshold: 3, timeout: TimeSpan.FromSeconds(5));

        var processBlock = new TransformBlock<int, int>(
            async number =>
            {
                if (circuitBreaker.IsOpen)
                {
                    Console.WriteLine($"회로 열림, 처리 건너뜀: {number}");
                    throw new InvalidOperationException("Circuit breaker is open");
                }

                try
                {
                    var result = await ProcessWithFailure(number);
                    circuitBreaker.RecordSuccess();
                    return result;
                }
                catch (Exception)
                {
                    circuitBreaker.RecordFailure();
                    throw;
                }
            });

        var outputBlock = new ActionBlock<int>(result =>
            Console.WriteLine($"성공: {result}"));

        processBlock.LinkTo(outputBlock, new DataflowLinkOptions
        {
            PropagateCompletion = false  // 오류 전파 방지
        });

        for (int i = 0; i < 20; i++)
        {
            try
            {
                await processBlock.SendAsync(i);
            }
            catch { }

            await Task.Delay(500);
        }
    }

    static async Task<int> ProcessWithFailure(int number)
    {
        await Task.Delay(50);

        // 30% 실패 확률
        if (Random.Shared.Next(100) < 30)
        {
            throw new InvalidOperationException($"처리 실패: {number}");
        }

        return number * 2;
    }

    class CircuitBreaker
    {
        private int _failureCount;
        private readonly int _failureThreshold;
        private readonly TimeSpan _timeout;
        private DateTime _lastFailureTime;

        public bool IsOpen =>
            _failureCount >= _failureThreshold &&
            DateTime.UtcNow - _lastFailureTime < _timeout;

        public CircuitBreaker(int failureThreshold, TimeSpan timeout)
        {
            _failureThreshold = failureThreshold;
            _timeout = timeout;
        }

        public void RecordSuccess()
        {
            _failureCount = 0;
        }

        public void RecordFailure()
        {
            _failureCount++;
            _lastFailureTime = DateTime.UtcNow;

            if (_failureCount == _failureThreshold)
            {
                Console.WriteLine($"⚠️ 회로 차단기 열림! (실패 {_failureCount}회)");
            }
        }
    }
}
```

## 8. 모범 사례

### ✅ 권장 사항

```csharp
// 1. PropagateCompletion으로 완료 신호 전파
block1.LinkTo(block2, new DataflowLinkOptions
{
    PropagateCompletion = true
});

// 2. MaxDegreeOfParallelism으로 병렬화 제어
var options = new ExecutionDataflowBlockOptions
{
    MaxDegreeOfParallelism = Environment.ProcessorCount
};

// 3. BoundedCapacity로 메모리 관리
var options = new ExecutionDataflowBlockOptions
{
    BoundedCapacity = 100  // 백프레셔 적용
};

// 4. 리소스 정리
using var cts = new CancellationTokenSource();
var options = new ExecutionDataflowBlockOptions
{
    CancellationToken = cts.Token
};

// 5. 예외 처리
try
{
    await block.Completion;
}
catch (Exception ex)
{
    Console.WriteLine($"블록 실패: {ex.Message}");
}
```

### ❌ 피해야 할 패턴

```csharp
// ❌ 1. PropagateCompletion 없이 연결
block1.LinkTo(block2);  // 완료 신호가 전파되지 않음!

// ❌ 2. 무제한 버퍼 (메모리 폭발)
var block = new ActionBlock<int>(SlowProcess);  // BoundedCapacity 미설정

// ❌ 3. Complete() 호출 누락
await block.Completion;  // 영원히 대기!

// ❌ 4. 공유 상태에 동기화 없이 접근
var counter = 0;
var block = new ActionBlock<int>(x =>
{
    counter++;  // ⚠️ Race Condition!
});

// ❌ 5. 예외 무시
await block.Completion;  // 예외가 발생해도 처리하지 않음
```

## 9. 성능 비교

```csharp
using System;
using System.Collections.Concurrent;
using System.Diagnostics;
using System.Linq;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

class DataflowPerformance
{
    const int ItemCount = 10_000;

    static async Task Main()
    {
        // 1. 순차 처리
        await TestSequential();

        // 2. Task 기반 병렬 처리
        await TestTaskBased();

        // 3. Parallel.ForEach
        await TestParallelForEach();

        // 4. TPL Dataflow
        await TestDataflow();
    }

    static async Task TestSequential()
    {
        var sw = Stopwatch.StartNew();

        foreach (var i in Enumerable.Range(0, ItemCount))
        {
            await ProcessItem(i);
        }

        sw.Stop();
        Console.WriteLine($"순차 처리:           {sw.ElapsedMilliseconds,6}ms");
    }

    static async Task TestTaskBased()
    {
        var sw = Stopwatch.StartNew();

        var tasks = Enumerable.Range(0, ItemCount)
            .Select(i => Task.Run(() => ProcessItem(i)));

        await Task.WhenAll(tasks);

        sw.Stop();
        Console.WriteLine($"Task 기반:           {sw.ElapsedMilliseconds,6}ms");
    }

    static async Task TestParallelForEach()
    {
        var sw = Stopwatch.StartNew();

        await Task.Run(() =>
        {
            Parallel.ForEach(Enumerable.Range(0, ItemCount), i =>
            {
                ProcessItem(i).Wait();
            });
        });

        sw.Stop();
        Console.WriteLine($"Parallel.ForEach:    {sw.ElapsedMilliseconds,6}ms");
    }

    static async Task TestDataflow()
    {
        var sw = Stopwatch.StartNew();

        var actionBlock = new ActionBlock<int>(
            async i => await ProcessItem(i),
            new ExecutionDataflowBlockOptions
            {
                MaxDegreeOfParallelism = Environment.ProcessorCount,
                BoundedCapacity = 100
            });

        foreach (var i in Enumerable.Range(0, ItemCount))
        {
            await actionBlock.SendAsync(i);
        }

        actionBlock.Complete();
        await actionBlock.Completion;

        sw.Stop();
        Console.WriteLine($"TPL Dataflow:        {sw.ElapsedMilliseconds,6}ms");
    }

    static async Task ProcessItem(int item)
    {
        await Task.Delay(1);  // 간단한 I/O 작업 시뮬레이션
    }
}

// 출력 예시 (8코어 시스템):
// 순차 처리:           10250ms
// Task 기반:            1350ms  // 스레드 풀 오버헤드
// Parallel.ForEach:     1280ms
// TPL Dataflow:         1200ms  // 백프레셔로 메모리 효율적
```

## 10. 요약

### 핵심 포인트

1. **블록 기반 아키텍처**: 독립적인 처리 단위로 파이프라인 구성
2. **자동 버퍼링**: 블록 간 비동기 데이터 전달
3. **병렬화 제어**: MaxDegreeOfParallelism으로 세밀한 조정
4. **백프레셔**: BoundedCapacity로 메모리 관리
5. **유연한 연결**: 조건부 링크, 브로드캐스트, 배치 처리
6. **오류 처리**: 블록별 독립적인 예외 관리
7. **성능**: 복잡한 파이프라인에 최적화
8. **Actor 모델**: 메시지 전달 기반 동시성

### 블록 타입 요약

| 블록 타입 | 역할 | 사용 사례 |
|----------|------|-----------|
| `ActionBlock<T>` | 데이터 소비 | 최종 처리, 저장 |
| `TransformBlock<T, U>` | 1:1 변환 | 데이터 변환, 필터링 |
| `TransformManyBlock<T, U>` | 1:N 변환 | 분할, 확장 |
| `BufferBlock<T>` | 큐 | 버퍼링, 생산자-소비자 |
| `BroadcastBlock<T>` | 브로드캐스트 | 다중 소비자 |
| `BatchBlock<T>` | 일괄 처리 | 배치 작업 |
| `JoinBlock<T1, T2>` | 조인 | 다중 입력 동기화 |
| `WriteOnceBlock<T>` | 단일 쓰기 | 초기화, 설정 |

### 빠른 참조

```csharp
// 기본 블록 생성
var block = new ActionBlock<int>(x => Console.WriteLine(x));

// 블록 연결
block1.LinkTo(block2, new DataflowLinkOptions
{
    PropagateCompletion = true
});

// 데이터 전송
await block.SendAsync(42);

// 완료
block.Complete();
await block.Completion;

// 옵션
var options = new ExecutionDataflowBlockOptions
{
    MaxDegreeOfParallelism = 4,
    BoundedCapacity = 100,
    CancellationToken = cts.Token
};
```

## 다음 단계

- **09-channels.md**: 경량 대안으로 System.Threading.Channels 학습
- **10-advanced-patterns.md**: 고급 동시성 패턴

## 참고 자료

- [Microsoft Docs: TPL Dataflow](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/dataflow-task-parallel-library)
- [Introduction to TPL Dataflow](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/walkthrough-creating-a-dataflow-pipeline)
- [Dataflow (Task Parallel Library)](https://devblogs.microsoft.com/pfxteam/introduction-to-tpl-dataflow/)
