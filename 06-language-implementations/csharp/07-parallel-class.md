# Parallel 클래스

## 📌 개요

`Parallel` 클래스는 .NET의 Task Parallel Library (TPL)의 핵심 구성 요소로, 명령형 데이터 병렬 처리를 제공합니다. PLINQ가 선언적 접근 방식을 제공하는 반면, Parallel 클래스는 명령형 루프와 메서드 호출을 병렬화하는 데 중점을 둡니다.

**주요 메서드:**
- `Parallel.For`: 병렬 for 루프
- `Parallel.ForEach`: 병렬 foreach 루프
- `Parallel.Invoke`: 병렬 메서드 실행

## 1. Parallel.For

### 기본 사용법

```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static void Main()
    {
        // 순차 for 루프
        for (int i = 0; i < 10; i++)
        {
            Console.WriteLine($"순차: {i}, Thread: {Thread.CurrentThread.ManagedThreadId}");
        }

        Console.WriteLine("\n--- 병렬 실행 ---\n");

        // 병렬 for 루프
        Parallel.For(0, 10, i =>
        {
            Console.WriteLine($"병렬: {i}, Thread: {Thread.CurrentThread.ManagedThreadId}");
        });
    }
}

// 출력 예시:
// 병렬: 0, Thread: 4
// 병렬: 2, Thread: 5
// 병렬: 1, Thread: 6
// 병렬: 3, Thread: 7
// 순서가 보장되지 않음!
```

### 실전 예제: 대용량 배열 처리

```csharp
using System;
using System.Diagnostics;
using System.Threading.Tasks;

class ParallelForExample
{
    static void Main()
    {
        const int size = 100_000_000;
        double[] numbers = new double[size];

        // 데이터 초기화
        for (int i = 0; i < size; i++)
        {
            numbers[i] = i * 0.5;
        }

        // 순차 처리
        var sw = Stopwatch.StartNew();
        double sequentialSum = 0;
        for (int i = 0; i < size; i++)
        {
            sequentialSum += Math.Sqrt(numbers[i]);
        }
        sw.Stop();
        Console.WriteLine($"순차 실행: {sw.ElapsedMilliseconds}ms, 합계: {sequentialSum:F2}");

        // 병렬 처리
        sw.Restart();
        object lockObj = new object();
        double parallelSum = 0;

        Parallel.For(0, size, i =>
        {
            double localSum = Math.Sqrt(numbers[i]);
            lock (lockObj)
            {
                parallelSum += localSum;
            }
        });
        sw.Stop();
        Console.WriteLine($"병렬 실행: {sw.ElapsedMilliseconds}ms, 합계: {parallelSum:F2}");
    }
}

// 출력 예시 (8코어 시스템):
// 순차 실행: 1200ms, 합계: 21081851083600.72
// 병렬 실행: 250ms, 합계: 21081851083600.72
// 약 5배 향상!
```

### Thread-Local State를 사용한 최적화

```csharp
using System;
using System.Diagnostics;
using System.Threading.Tasks;

class ThreadLocalStateExample
{
    static void Main()
    {
        const int size = 100_000_000;
        double[] numbers = Enumerable.Range(0, size)
            .Select(i => i * 0.5)
            .ToArray();

        // ❌ 비효율적: 모든 반복마다 lock
        var sw = Stopwatch.StartNew();
        object lockObj = new object();
        double inefficientSum = 0;

        Parallel.For(0, size, i =>
        {
            lock (lockObj)  // 매우 비효율적!
            {
                inefficientSum += Math.Sqrt(numbers[i]);
            }
        });
        sw.Stop();
        Console.WriteLine($"Lock 사용 (비효율적): {sw.ElapsedMilliseconds}ms");

        // ✅ 효율적: Thread-Local State 사용
        sw.Restart();
        double efficientSum = Parallel.For(
            fromInclusive: 0,
            toExclusive: size,

            // Thread-Local 초기화
            localInit: () => 0.0,

            // 각 반복 (thread-local 상태 사용)
            body: (i, loopState, localSum) =>
            {
                return localSum + Math.Sqrt(numbers[i]);
            },

            // 최종 집계 (각 스레드당 한 번만 lock)
            localFinally: localSum =>
            {
                lock (lockObj)
                {
                    efficientSum += localSum;
                }
            }
        ).IsCompleted ? efficientSum : 0;

        sw.Stop();
        Console.WriteLine($"Thread-Local State 사용: {sw.ElapsedMilliseconds}ms");
        Console.WriteLine($"합계: {efficientSum:F2}");
    }
}

// 출력 예시:
// Lock 사용 (비효율적): 3500ms
// Thread-Local State 사용: 180ms
// 약 20배 향상!
```

### 루프 제어: Break vs Stop

```csharp
using System;
using System.Threading.Tasks;

class LoopControlExample
{
    static void DemoBreak()
    {
        Console.WriteLine("=== Parallel.For with Break ===");

        Parallel.For(0, 100, (i, loopState) =>
        {
            Console.WriteLine($"처리 중: {i}");

            if (i == 15)
            {
                Console.WriteLine($"Break 호출: {i}");
                loopState.Break();  // 15 이하의 모든 반복은 완료 보장
            }

            Thread.Sleep(10);
        });
    }

    static void DemoStop()
    {
        Console.WriteLine("\n=== Parallel.For with Stop ===");

        Parallel.For(0, 100, (i, loopState) =>
        {
            Console.WriteLine($"처리 중: {i}");

            if (i == 15)
            {
                Console.WriteLine($"Stop 호출: {i}");
                loopState.Stop();  // 즉시 중단 (다른 반복 무시 가능)
            }

            Thread.Sleep(10);
        });
    }

    static void Main()
    {
        DemoBreak();
        DemoStop();
    }
}

// Break vs Stop 차이:
// - Break: 현재 인덱스 이전의 모든 반복 완료 보장
// - Stop: 가능한 빨리 중단, 순서 보장 없음
```

## 2. Parallel.ForEach

### 기본 사용법

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

class ParallelForEachExample
{
    static void Main()
    {
        var files = new List<string>
        {
            "file1.txt", "file2.txt", "file3.txt",
            "file4.txt", "file5.txt", "file6.txt"
        };

        // 순차 처리
        Console.WriteLine("=== 순차 처리 ===");
        foreach (var file in files)
        {
            ProcessFile(file);
        }

        // 병렬 처리
        Console.WriteLine("\n=== 병렬 처리 ===");
        Parallel.ForEach(files, file =>
        {
            ProcessFile(file);
        });
    }

    static void ProcessFile(string filename)
    {
        Console.WriteLine($"처리 시작: {filename} (Thread: {Thread.CurrentThread.ManagedThreadId})");
        Thread.Sleep(500);  // 파일 처리 시뮬레이션
        Console.WriteLine($"처리 완료: {filename}");
    }
}
```

### 실전 예제: 이미지 배치 처리

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Drawing;
using System.Drawing.Imaging;
using System.IO;
using System.Threading.Tasks;

class ImageBatchProcessor
{
    static void Main()
    {
        var imageFiles = Directory.GetFiles(@"C:\Images", "*.jpg");
        var outputDir = @"C:\Images\Thumbnails";
        Directory.CreateDirectory(outputDir);

        Console.WriteLine($"총 {imageFiles.Length}개 이미지 처리 중...");

        var sw = Stopwatch.StartNew();
        int processedCount = 0;
        object lockObj = new object();

        Parallel.ForEach(
            imageFiles,
            new ParallelOptions
            {
                MaxDegreeOfParallelism = Environment.ProcessorCount
            },
            imageFile =>
            {
                try
                {
                    // 이미지 로드
                    using (var image = Image.FromFile(imageFile))
                    {
                        // 썸네일 생성 (200x200)
                        var thumbnail = image.GetThumbnailImage(
                            200, 200, null, IntPtr.Zero);

                        // 저장
                        var outputPath = Path.Combine(
                            outputDir,
                            Path.GetFileName(imageFile));
                        thumbnail.Save(outputPath, ImageFormat.Jpeg);
                        thumbnail.Dispose();
                    }

                    // 진행률 업데이트 (thread-safe)
                    lock (lockObj)
                    {
                        processedCount++;
                        if (processedCount % 10 == 0)
                        {
                            Console.WriteLine($"진행률: {processedCount}/{imageFiles.Length}");
                        }
                    }
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"오류 ({imageFile}): {ex.Message}");
                }
            }
        );

        sw.Stop();
        Console.WriteLine($"\n완료! {processedCount}개 이미지 처리됨");
        Console.WriteLine($"소요 시간: {sw.Elapsed.TotalSeconds:F2}초");
    }
}

// 출력 예시 (1000개 이미지, 8코어):
// 진행률: 10/1000
// 진행률: 20/1000
// ...
// 진행률: 1000/1000
// 완료! 1000개 이미지 처리됨
// 소요 시간: 12.34초
// (순차 처리 시 약 80초)
```

### Partitioner를 사용한 고급 분할

```csharp
using System;
using System.Collections.Concurrent;
using System.Diagnostics;
using System.Linq;
using System.Threading.Tasks;

class PartitionerExample
{
    static void Main()
    {
        var numbers = Enumerable.Range(0, 10_000_000).ToList();

        // 기본 분할 (각 요소마다 태스크 생성 가능 - 오버헤드 큼)
        var sw = Stopwatch.StartNew();
        long sum1 = 0;
        Parallel.ForEach(numbers, num =>
        {
            Interlocked.Add(ref sum1, num);
        });
        sw.Stop();
        Console.WriteLine($"기본 분할: {sw.ElapsedMilliseconds}ms");

        // 청크 분할 (효율적인 배치 처리)
        sw.Restart();
        long sum2 = 0;
        var partitioner = Partitioner.Create(numbers, loadBalance: true);

        Parallel.ForEach(partitioner, chunk =>
        {
            long localSum = 0;
            foreach (var num in chunk)
            {
                localSum += num;
            }
            Interlocked.Add(ref sum2, localSum);
        });
        sw.Stop();
        Console.WriteLine($"청크 분할: {sw.ElapsedMilliseconds}ms");

        // 범위 분할 (가장 효율적)
        sw.Restart();
        long sum3 = 0;
        var rangePartitioner = Partitioner.Create(0, numbers.Count);

        Parallel.ForEach(rangePartitioner, (range, loopState) =>
        {
            long localSum = 0;
            for (int i = range.Item1; i < range.Item2; i++)
            {
                localSum += numbers[i];
            }
            Interlocked.Add(ref sum3, localSum);
        });
        sw.Stop();
        Console.WriteLine($"범위 분할: {sw.ElapsedMilliseconds}ms");

        Console.WriteLine($"\n합계: {sum1} (모두 동일해야 함)");
    }
}

// 출력 예시:
// 기본 분할: 850ms
// 청크 분할: 180ms
// 범위 분할: 120ms
```

## 3. Parallel.Invoke

### 기본 사용법

```csharp
using System;
using System.Diagnostics;
using System.Threading.Tasks;

class ParallelInvokeExample
{
    static void Main()
    {
        var sw = Stopwatch.StartNew();

        // 순차 실행
        Task1();
        Task2();
        Task3();

        sw.Stop();
        Console.WriteLine($"순차 실행: {sw.ElapsedMilliseconds}ms");

        // 병렬 실행
        sw.Restart();
        Parallel.Invoke(
            Task1,
            Task2,
            Task3
        );
        sw.Stop();
        Console.WriteLine($"병렬 실행: {sw.ElapsedMilliseconds}ms");
    }

    static void Task1()
    {
        Console.WriteLine("Task 1 시작");
        Thread.Sleep(1000);
        Console.WriteLine("Task 1 완료");
    }

    static void Task2()
    {
        Console.WriteLine("Task 2 시작");
        Thread.Sleep(1000);
        Console.WriteLine("Task 2 완료");
    }

    static void Task3()
    {
        Console.WriteLine("Task 3 시작");
        Thread.Sleep(1000);
        Console.WriteLine("Task 3 완료");
    }
}

// 출력:
// 순차 실행: 3000ms
// 병렬 실행: 1000ms
```

### 실전 예제: 데이터 파이프라인

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Linq;
using System.Threading.Tasks;

class DataPipeline
{
    static List<string> rawData = new List<string>();
    static List<ProcessedData> processedData = new List<ProcessedData>();
    static Dictionary<string, int> statistics = new Dictionary<string, int>();

    class ProcessedData
    {
        public string Value { get; set; }
        public DateTime Timestamp { get; set; }
        public string Category { get; set; }
    }

    static void Main()
    {
        var sw = Stopwatch.StartNew();

        // 세 가지 독립적인 작업을 병렬로 실행
        Parallel.Invoke(
            // 작업 1: 데이터 로드
            () =>
            {
                Console.WriteLine("데이터 로드 시작...");
                LoadData();
                Console.WriteLine($"데이터 로드 완료: {rawData.Count}개 항목");
            },

            // 작업 2: 설정 파일 읽기
            () =>
            {
                Console.WriteLine("설정 로드 시작...");
                var config = LoadConfiguration();
                Console.WriteLine($"설정 로드 완료: {config.Count}개 설정");
            },

            // 작업 3: 캐시 워밍업
            () =>
            {
                Console.WriteLine("캐시 워밍업 시작...");
                WarmupCache();
                Console.WriteLine("캐시 워밍업 완료");
            }
        );

        sw.Stop();
        Console.WriteLine($"\n초기화 완료: {sw.ElapsedMilliseconds}ms");

        // 데이터 처리 파이프라인
        sw.Restart();
        Parallel.Invoke(
            // 단계 1: 데이터 변환
            () =>
            {
                processedData = rawData.Select(data => new ProcessedData
                {
                    Value = data.ToUpper(),
                    Timestamp = DateTime.Now,
                    Category = ClassifyData(data)
                }).ToList();
                Console.WriteLine($"데이터 변환 완료: {processedData.Count}개");
            },

            // 단계 2: 통계 계산
            () =>
            {
                Thread.Sleep(500);  // 데이터 로드 대기
                statistics = rawData
                    .GroupBy(d => ClassifyData(d))
                    .ToDictionary(g => g.Key, g => g.Count());
                Console.WriteLine($"통계 계산 완료: {statistics.Count}개 카테고리");
            }
        );

        sw.Stop();
        Console.WriteLine($"처리 완료: {sw.ElapsedMilliseconds}ms");
    }

    static void LoadData()
    {
        // 대용량 데이터 로드 시뮬레이션
        Thread.Sleep(1000);
        rawData = Enumerable.Range(0, 100000)
            .Select(i => $"Data_{i}")
            .ToList();
    }

    static Dictionary<string, string> LoadConfiguration()
    {
        Thread.Sleep(800);
        return new Dictionary<string, string>
        {
            ["Setting1"] = "Value1",
            ["Setting2"] = "Value2"
        };
    }

    static void WarmupCache()
    {
        Thread.Sleep(600);
        // 캐시 초기화 로직
    }

    static string ClassifyData(string data)
    {
        int hash = data.GetHashCode();
        return hash % 3 == 0 ? "CategoryA" :
               hash % 3 == 1 ? "CategoryB" : "CategoryC";
    }
}

// 출력 예시:
// 데이터 로드 시작...
// 설정 로드 시작...
// 캐시 워밍업 시작...
// 캐시 워밍업 완료
// 설정 로드 완료: 2개 설정
// 데이터 로드 완료: 100000개 항목
//
// 초기화 완료: 1050ms (순차 실행 시 2400ms)
```

## 4. ParallelOptions

### 병렬화 수준 제어

```csharp
using System;
using System.Diagnostics;
using System.Threading;
using System.Threading.Tasks;

class ParallelOptionsExample
{
    static void Main()
    {
        const int iterations = 20;

        // 기본 설정 (모든 코어 사용)
        Console.WriteLine($"CPU 코어 수: {Environment.ProcessorCount}");

        var sw = Stopwatch.StartNew();
        Parallel.For(0, iterations, i =>
        {
            Thread.Sleep(100);
            Console.WriteLine($"기본: {i}, Thread: {Thread.CurrentThread.ManagedThreadId}");
        });
        sw.Stop();
        Console.WriteLine($"기본 설정: {sw.ElapsedMilliseconds}ms\n");

        // MaxDegreeOfParallelism 제한
        sw.Restart();
        Parallel.For(0, iterations, new ParallelOptions
        {
            MaxDegreeOfParallelism = 2  // 최대 2개 스레드만 사용
        }, i =>
        {
            Thread.Sleep(100);
            Console.WriteLine($"제한: {i}, Thread: {Thread.CurrentThread.ManagedThreadId}");
        });
        sw.Stop();
        Console.WriteLine($"병렬화 제한 (2): {sw.ElapsedMilliseconds}ms");
    }
}

// 출력 예시 (8코어 시스템):
// CPU 코어 수: 8
// 기본: 0, Thread: 4
// 기본: 1, Thread: 5
// 기본: 2, Thread: 6
// ...
// 기본 설정: 300ms
//
// 제한: 0, Thread: 7
// 제한: 1, Thread: 8
// 제한: 2, Thread: 7  // 2개 스레드만 번갈아 사용
// ...
// 병렬화 제한 (2): 1000ms
```

### CancellationToken 사용

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class CancellationExample
{
    static void Main()
    {
        var cts = new CancellationTokenSource();

        // 5초 후 자동 취소
        cts.CancelAfter(TimeSpan.FromSeconds(5));

        // 사용자 입력으로 취소
        Task.Run(() =>
        {
            Console.WriteLine("작업 취소하려면 아무 키나 누르세요...");
            Console.ReadKey();
            cts.Cancel();
        });

        try
        {
            Parallel.For(0, 1000, new ParallelOptions
            {
                CancellationToken = cts.Token
            }, i =>
            {
                // 취소 확인
                cts.Token.ThrowIfCancellationRequested();

                Console.WriteLine($"처리 중: {i}");
                Thread.Sleep(500);
            });

            Console.WriteLine("모든 작업 완료!");
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("\n작업이 취소되었습니다.");
        }
        finally
        {
            cts.Dispose();
        }
    }
}
```

### TaskScheduler 커스터마이징

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

class CustomSchedulerExample
{
    static void Main()
    {
        // 커스텀 스케줄러 생성 (최대 동시 작업 수 제한)
        var scheduler = new LimitedConcurrencyLevelTaskScheduler(2);

        var options = new ParallelOptions
        {
            TaskScheduler = scheduler
        };

        Console.WriteLine("커스텀 스케줄러 사용:");
        Parallel.For(0, 10, options, i =>
        {
            Console.WriteLine($"작업 {i} 시작 (Thread: {Thread.CurrentThread.ManagedThreadId})");
            Thread.Sleep(1000);
            Console.WriteLine($"작업 {i} 완료");
        });
    }
}

// LimitedConcurrencyLevelTaskScheduler 구현
public class LimitedConcurrencyLevelTaskScheduler : TaskScheduler
{
    private readonly int _maxDegreeOfParallelism;
    private readonly LinkedList<Task> _tasks = new LinkedList<Task>();
    private int _delegatesQueuedOrRunning = 0;

    public LimitedConcurrencyLevelTaskScheduler(int maxDegreeOfParallelism)
    {
        if (maxDegreeOfParallelism < 1)
            throw new ArgumentOutOfRangeException(nameof(maxDegreeOfParallelism));

        _maxDegreeOfParallelism = maxDegreeOfParallelism;
    }

    protected override void QueueTask(Task task)
    {
        lock (_tasks)
        {
            _tasks.AddLast(task);
            if (_delegatesQueuedOrRunning < _maxDegreeOfParallelism)
            {
                ++_delegatesQueuedOrRunning;
                NotifyThreadPoolOfPendingWork();
            }
        }
    }

    private void NotifyThreadPoolOfPendingWork()
    {
        ThreadPool.UnsafeQueueUserWorkItem(_ =>
        {
            try
            {
                while (true)
                {
                    Task item;
                    lock (_tasks)
                    {
                        if (_tasks.Count == 0)
                        {
                            --_delegatesQueuedOrRunning;
                            break;
                        }

                        item = _tasks.First.Value;
                        _tasks.RemoveFirst();
                    }

                    TryExecuteTask(item);
                }
            }
            finally { }
        }, null);
    }

    protected override bool TryExecuteTaskInline(Task task, bool taskWasPreviouslyQueued)
    {
        return false;
    }

    protected override IEnumerable<Task> GetScheduledTasks()
    {
        bool lockTaken = false;
        try
        {
            Monitor.TryEnter(_tasks, ref lockTaken);
            if (lockTaken) return _tasks.ToArray();
            else throw new NotSupportedException();
        }
        finally
        {
            if (lockTaken) Monitor.Exit(_tasks);
        }
    }

    public override int MaximumConcurrencyLevel => _maxDegreeOfParallelism;
}
```

## 5. 예외 처리

### AggregateException 처리

```csharp
using System;
using System.Threading.Tasks;

class ExceptionHandlingExample
{
    static void Main()
    {
        try
        {
            Parallel.For(0, 100, i =>
            {
                if (i == 10)
                    throw new InvalidOperationException($"작업 {i}에서 오류 발생!");

                if (i == 25)
                    throw new ArgumentException($"작업 {i}에서 인수 오류!");

                Console.WriteLine($"처리 중: {i}");
            });
        }
        catch (AggregateException ae)
        {
            Console.WriteLine($"\n총 {ae.InnerExceptions.Count}개의 예외 발생:");

            foreach (var ex in ae.InnerExceptions)
            {
                Console.WriteLine($"- {ex.GetType().Name}: {ex.Message}");
            }

            // 특정 예외 타입만 처리
            ae.Handle(ex =>
            {
                if (ex is InvalidOperationException)
                {
                    Console.WriteLine($"InvalidOperationException 처리됨: {ex.Message}");
                    return true;  // 처리됨
                }
                return false;  // 재발생
            });
        }
    }
}

// 출력 예시:
// 처리 중: 0
// 처리 중: 1
// ...
// 처리 중: 9
//
// 총 2개의 예외 발생:
// - InvalidOperationException: 작업 10에서 오류 발생!
// - ArgumentException: 작업 25에서 인수 오류!
```

### 예외 발생 시 계속 진행

```csharp
using System;
using System.Collections.Concurrent;
using System.Threading.Tasks;

class ContinueOnExceptionExample
{
    static void Main()
    {
        var errors = new ConcurrentBag<Exception>();
        var successCount = 0;
        var totalCount = 100;

        Parallel.For(0, totalCount, i =>
        {
            try
            {
                ProcessItem(i);
                Interlocked.Increment(ref successCount);
            }
            catch (Exception ex)
            {
                errors.Add(ex);
                Console.WriteLine($"항목 {i} 처리 실패: {ex.Message}");
                // 예외를 기록하고 계속 진행
            }
        });

        Console.WriteLine($"\n처리 완료:");
        Console.WriteLine($"- 성공: {successCount}/{totalCount}");
        Console.WriteLine($"- 실패: {errors.Count}/{totalCount}");

        if (errors.Count > 0)
        {
            Console.WriteLine("\n오류 목록:");
            foreach (var error in errors)
            {
                Console.WriteLine($"- {error.Message}");
            }
        }
    }

    static void ProcessItem(int i)
    {
        if (i % 10 == 0)
            throw new InvalidOperationException($"항목 {i} 처리 불가");

        // 정상 처리
        Thread.Sleep(10);
    }
}

// 출력:
// 항목 0 처리 실패: 항목 0 처리 불가
// 항목 10 처리 실패: 항목 10 처리 불가
// ...
//
// 처리 완료:
// - 성공: 90/100
// - 실패: 10/100
```

## 6. 성능 비교

### Parallel vs PLINQ vs Sequential

```csharp
using System;
using System.Diagnostics;
using System.Linq;
using System.Threading.Tasks;

class PerformanceComparison
{
    static void Main()
    {
        const int size = 10_000_000;
        var data = Enumerable.Range(0, size).ToArray();

        // 1. 순차 처리
        var sw = Stopwatch.StartNew();
        long sum1 = 0;
        for (int i = 0; i < size; i++)
        {
            if (IsPrime(data[i]))
                sum1 += data[i];
        }
        sw.Stop();
        Console.WriteLine($"순차 (for):        {sw.ElapsedMilliseconds,6}ms, 합계: {sum1}");

        // 2. LINQ
        sw.Restart();
        var sum2 = data.Where(IsPrime).Sum(x => (long)x);
        sw.Stop();
        Console.WriteLine($"LINQ:              {sw.ElapsedMilliseconds,6}ms, 합계: {sum2}");

        // 3. PLINQ
        sw.Restart();
        var sum3 = data.AsParallel().Where(IsPrime).Sum(x => (long)x);
        sw.Stop();
        Console.WriteLine($"PLINQ:             {sw.ElapsedMilliseconds,6}ms, 합계: {sum3}");

        // 4. Parallel.For
        sw.Restart();
        long sum4 = 0;
        Parallel.For(0, size, () => 0L, (i, state, localSum) =>
        {
            if (IsPrime(data[i]))
                return localSum + data[i];
            return localSum;
        }, localSum =>
        {
            Interlocked.Add(ref sum4, localSum);
        });
        sw.Stop();
        Console.WriteLine($"Parallel.For:      {sw.ElapsedMilliseconds,6}ms, 합계: {sum4}");

        // 5. Parallel.ForEach
        sw.Restart();
        long sum5 = 0;
        Parallel.ForEach(data, () => 0L, (num, state, localSum) =>
        {
            if (IsPrime(num))
                return localSum + num;
            return localSum;
        }, localSum =>
        {
            Interlocked.Add(ref sum5, localSum);
        });
        sw.Stop();
        Console.WriteLine($"Parallel.ForEach:  {sw.ElapsedMilliseconds,6}ms, 합계: {sum5}");
    }

    static bool IsPrime(int number)
    {
        if (number < 2) return false;
        if (number == 2) return true;
        if (number % 2 == 0) return false;

        int sqrt = (int)Math.Sqrt(number);
        for (int i = 3; i <= sqrt; i += 2)
        {
            if (number % i == 0) return false;
        }
        return true;
    }
}

// 출력 예시 (8코어 시스템):
// 순차 (for):        12450ms, 합계: 3203324994356
// LINQ:              12680ms, 합계: 3203324994356
// PLINQ:              1850ms, 합계: 3203324994356
// Parallel.For:       1720ms, 합계: 3203324994356
// Parallel.ForEach:   1740ms, 합계: 3203324994356
//
// 결론: CPU 집약적 작업에서는 병렬 처리가 약 7배 빠름
```

### 오버헤드 측정

```csharp
using System;
using System.Diagnostics;
using System.Linq;
using System.Threading.Tasks;

class OverheadMeasurement
{
    static void Main()
    {
        // 간단한 작업 (병렬화 오버헤드가 이득보다 큼)
        var smallData = Enumerable.Range(0, 1000).ToArray();

        var sw = Stopwatch.StartNew();
        var sum1 = smallData.Sum();
        sw.Stop();
        Console.WriteLine($"순차 (작은 데이터):   {sw.ElapsedTicks,8} ticks");

        sw.Restart();
        var sum2 = 0;
        Parallel.For(0, smallData.Length, i =>
        {
            Interlocked.Add(ref sum2, smallData[i]);
        });
        sw.Stop();
        Console.WriteLine($"병렬 (작은 데이터):   {sw.ElapsedTicks,8} ticks (더 느림!)\n");

        // 복잡한 작업 (병렬화 이득이 오버헤드보다 큼)
        var largeData = Enumerable.Range(0, 10_000_000).ToArray();

        sw.Restart();
        long sum3 = 0;
        for (int i = 0; i < largeData.Length; i++)
        {
            sum3 += (long)Math.Sqrt(largeData[i]);
        }
        sw.Stop();
        Console.WriteLine($"순차 (큰 데이터):     {sw.ElapsedMilliseconds,8}ms");

        sw.Restart();
        long sum4 = 0;
        Parallel.For(0, largeData.Length, () => 0L, (i, state, local) =>
        {
            return local + (long)Math.Sqrt(largeData[i]);
        }, local =>
        {
            Interlocked.Add(ref sum4, local);
        });
        sw.Stop();
        Console.WriteLine($"병렬 (큰 데이터):     {sw.ElapsedMilliseconds,8}ms (훨씬 빠름!)");
    }
}

// 출력 예시:
// 순차 (작은 데이터):        1250 ticks
// 병렬 (작은 데이터):       45000 ticks (더 느림!)
//
// 순차 (큰 데이터):          1200ms
// 병렬 (큰 데이터):           180ms (훨씬 빠름!)
//
// 교훈: 작은 데이터나 간단한 작업은 순차 처리가 더 빠름
```

## 7. 모범 사례

### ✅ 권장 사항

```csharp
// 1. Thread-Local State 사용으로 lock 최소화
Parallel.For(0, size, () => 0.0, (i, state, local) =>
{
    return local + ExpensiveCalculation(i);
}, local =>
{
    lock (lockObj) { total += local; }
});

// 2. Partitioner로 오버헤드 감소
var partitioner = Partitioner.Create(0, size, size / Environment.ProcessorCount);
Parallel.ForEach(partitioner, range =>
{
    for (int i = range.Item1; i < range.Item2; i++)
    {
        ProcessItem(i);
    }
});

// 3. CancellationToken으로 중단 가능하게
var cts = new CancellationTokenSource();
var options = new ParallelOptions { CancellationToken = cts.Token };
Parallel.For(0, size, options, i =>
{
    options.CancellationToken.ThrowIfCancellationRequested();
    ProcessItem(i);
});

// 4. 예외 처리로 안정성 확보
try
{
    Parallel.ForEach(items, item =>
    {
        try
        {
            ProcessItem(item);
        }
        catch (Exception ex)
        {
            LogError(ex);
            // 계속 진행
        }
    });
}
catch (AggregateException ae)
{
    HandleAggregateException(ae);
}
```

### ❌ 피해야 할 패턴

```csharp
// ❌ 1. 모든 반복마다 lock (매우 비효율적)
Parallel.For(0, size, i =>
{
    lock (lockObj)  // 병렬화 이득을 모두 상쇄!
    {
        sharedData += ProcessItem(i);
    }
});

// ❌ 2. 작은 데이터에 병렬화 (오버헤드 > 이득)
var small = new[] { 1, 2, 3, 4, 5 };
Parallel.ForEach(small, x => Process(x));  // 순차가 더 빠름

// ❌ 3. 공유 상태에 직접 쓰기 (Race Condition)
int counter = 0;
Parallel.For(0, size, i =>
{
    counter++;  // ⚠️ Thread-unsafe!
});

// ❌ 4. ForEach에서 컬렉션 수정
var list = new List<int> { 1, 2, 3, 4, 5 };
Parallel.ForEach(list, item =>
{
    list.Add(item * 2);  // ⚠️ InvalidOperationException!
});

// ❌ 5. UI 스레드 블로킹
// WinForms/WPF에서:
private void Button_Click(object sender, EventArgs e)
{
    Parallel.For(0, 1000, i =>  // ⚠️ UI 프리즈!
    {
        Thread.Sleep(100);
    });
}
```

## 8. 언제 Parallel 클래스를 사용할까?

### 결정 트리

```
작업이 CPU 집약적인가?
├─ 예 → 데이터 크기가 충분히 큰가? (>10,000)
│        ├─ 예 → 명령형 루프인가?
│        │        ├─ 예 → ✅ Parallel.For/ForEach 사용
│        │        └─ 아니오 → ✅ PLINQ 사용
│        └─ 아니오 → ❌ 순차 처리 사용
│
└─ 아니오 → I/O 집약적인가?
           ├─ 예 → ✅ async/await + Task 사용
           └─ 아니오 → ❌ 순차 처리 사용
```

### 사용 사례

| 시나리오 | 권장 방법 | 이유 |
|---------|----------|------|
| 대용량 배열 처리 | `Parallel.For` | 인덱스 기반 접근, 최고 성능 |
| 컬렉션 반복 | `Parallel.ForEach` | 편리한 열거, IEnumerable 지원 |
| 독립적인 작업 실행 | `Parallel.Invoke` | 간단한 병렬 실행 |
| LINQ 쿼리 병렬화 | `PLINQ` | 선언적, 함수형 스타일 |
| 파일 I/O | `async/await` | 스레드 차단 없음 |
| 네트워크 요청 | `Task.WhenAll` | 비동기 I/O 최적화 |
| 소량 데이터 (<1000) | 순차 처리 | 오버헤드 회피 |
| 간단한 연산 | 순차 처리 | 오버헤드 회피 |

## 9. Parallel vs Task

### 비교

```csharp
using System;
using System.Diagnostics;
using System.Linq;
using System.Threading.Tasks;

class ParallelVsTask
{
    static async Task Main()
    {
        const int count = 10;

        // Parallel.For - 블로킹, CPU 집약적 작업에 적합
        Console.WriteLine("=== Parallel.For ===");
        var sw = Stopwatch.StartNew();
        Parallel.For(0, count, i =>
        {
            CpuIntensiveWork(i);
        });
        sw.Stop();
        Console.WriteLine($"소요 시간: {sw.ElapsedMilliseconds}ms\n");

        // Task - 논블로킹, I/O 작업에 적합
        Console.WriteLine("=== Task (I/O) ===");
        sw.Restart();
        var tasks = Enumerable.Range(0, count)
            .Select(i => IoWork(i))
            .ToArray();
        await Task.WhenAll(tasks);
        sw.Stop();
        Console.WriteLine($"소요 시간: {sw.ElapsedMilliseconds}ms");
    }

    static void CpuIntensiveWork(int id)
    {
        // CPU 집약적 작업 (계산)
        var sum = 0.0;
        for (int i = 0; i < 10_000_000; i++)
        {
            sum += Math.Sqrt(i);
        }
        Console.WriteLine($"CPU 작업 {id} 완료");
    }

    static async Task IoWork(int id)
    {
        // I/O 작업 (네트워크, 파일 등)
        await Task.Delay(1000);  // 비동기 대기 (스레드 차단 없음)
        Console.WriteLine($"I/O 작업 {id} 완료");
    }
}

// 출력:
// === Parallel.For ===
// CPU 작업 0 완료
// CPU 작업 1 완료
// ...
// 소요 시간: 1200ms (병렬 실행으로 단축)
//
// === Task (I/O) ===
// I/O 작업 0 완료
// I/O 작업 1 완료
// ...
// 소요 시간: 1000ms (모두 동시 대기)
```

### 선택 가이드

```csharp
// CPU 집약적 작업 → Parallel
Parallel.For(0, size, i =>
{
    data[i] = ExpensiveCpuCalculation(data[i]);
});

// I/O 집약적 작업 → async/await + Task
var tasks = urls.Select(url => DownloadAsync(url));
await Task.WhenAll(tasks);

// 혼합 작업 → Task + 내부 Parallel
await Task.Run(() =>
{
    Parallel.For(0, size, i =>
    {
        ProcessData(i);
    });
});
```

## 10. 요약

### 핵심 포인트

1. **Parallel.For**: 인덱스 기반 병렬 루프, 최고 성능
2. **Parallel.ForEach**: 컬렉션 병렬 반복, 편리함
3. **Parallel.Invoke**: 독립적인 메서드 병렬 실행
4. **Thread-Local State**: lock 최소화로 성능 향상
5. **ParallelOptions**: 세밀한 제어 (취소, 병렬화 수준 등)
6. **예외 처리**: AggregateException으로 모든 예외 수집
7. **성능**: CPU 집약적, 대용량 데이터에 효과적
8. **오버헤드**: 작은 데이터나 간단한 작업은 순차 처리가 빠름

### 빠른 참조

```csharp
// 기본 사용
Parallel.For(0, 100, i => Process(i));
Parallel.ForEach(items, item => Process(item));
Parallel.Invoke(Method1, Method2, Method3);

// Thread-Local State
Parallel.For(0, size,
    () => 0,                           // 초기화
    (i, state, local) => local + i,    // 반복
    local => Interlocked.Add(ref total, local)  // 최종화
);

// ParallelOptions
var options = new ParallelOptions
{
    MaxDegreeOfParallelism = 4,
    CancellationToken = cts.Token
};
Parallel.For(0, 100, options, i => Process(i));

// 루프 제어
Parallel.For(0, 100, (i, state) =>
{
    if (condition) state.Break();  // 순서 보장
    if (emergency) state.Stop();   // 즉시 중단
});

// 예외 처리
try
{
    Parallel.For(0, 100, i => Process(i));
}
catch (AggregateException ae)
{
    foreach (var ex in ae.InnerExceptions)
        Console.WriteLine(ex.Message);
}
```

## 다음 단계

- **08-tpl-dataflow.md**: 복잡한 데이터 처리 파이프라인 구축
- **09-channels.md**: 비동기 생산자-소비자 패턴
- **10-concurrent-collections.md**: Thread-safe 컬렉션 심화

## 참고 자료

- [Microsoft Docs: Parallel Class](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.parallel)
- [Parallel Programming in .NET](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/)
- [Task Parallel Library (TPL)](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/task-parallel-library-tpl)
