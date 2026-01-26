# PLINQ (Parallel LINQ)

## 📌 개요

**PLINQ (Parallel LINQ)**는 LINQ to Objects의 병렬 구현으로, 데이터 병렬 처리를 선언적으로 표현할 수 있게 해줍니다. .NET 4.0에서 도입되었으며, 멀티코어 프로세서를 활용하여 LINQ 쿼리를 자동으로 병렬화합니다.

**핵심 개념**: LINQ 쿼리에 `.AsParallel()`을 추가하면 자동으로 병렬 실행

---

## 🎯 기본 사용법

### 간단한 예제

```csharp
using System;
using System.Linq;
using System.Collections.Generic;

// 순차 LINQ
var numbers = Enumerable.Range(1, 10000);
var result = numbers
    .Where(n => IsPrime(n))
    .Select(n => n * n)
    .ToList();

// 병렬 PLINQ - .AsParallel()만 추가!
var resultParallel = numbers
    .AsParallel()
    .Where(n => IsPrime(n))
    .Select(n => n * n)
    .ToList();

bool IsPrime(int number)
{
    if (number < 2) return false;
    for (int i = 2; i <= Math.Sqrt(number); i++)
    {
        if (number % i == 0) return false;
    }
    return true;
}
```

### AsParallel() 확장 메서드

```csharp
// IEnumerable<T>를 ParallelQuery<T>로 변환
IEnumerable<int> numbers = Enumerable.Range(1, 1000);
ParallelQuery<int> parallelNumbers = numbers.AsParallel();
```

---

## ⚡ 병렬화 제어

### 1. 병렬화 정도 (Degree of Parallelism) 설정

```csharp
// 최대 4개의 스레드만 사용
var result = numbers
    .AsParallel()
    .WithDegreeOfParallelism(4)
    .Where(n => IsPrime(n))
    .ToList();

// CPU 코어 수만큼 사용 (기본값)
var result2 = numbers
    .AsParallel()
    .WithDegreeOfParallelism(Environment.ProcessorCount)
    .Where(n => IsPrime(n))
    .ToList();
```

### 2. 실행 모드 설정

```csharp
// 강제 병렬 실행
var result = numbers
    .AsParallel()
    .WithExecutionMode(ParallelExecutionMode.ForceParallelism)
    .Where(n => n % 2 == 0)
    .ToList();

// 기본값: PLINQ가 자동으로 판단
var result2 = numbers
    .AsParallel()
    .WithExecutionMode(ParallelExecutionMode.Default)
    .Where(n => n % 2 == 0)
    .ToList();
```

### 3. 순서 보존

```csharp
// 순서 보존 (성능 저하 가능)
var ordered = numbers
    .AsParallel()
    .AsOrdered()  // 원본 순서 유지
    .Where(n => IsPrime(n))
    .ToList();

// 순서 무시 (더 빠름, 기본값)
var unordered = numbers
    .AsParallel()
    .AsUnordered()  // 순서 보장 안함
    .Where(n => IsPrime(n))
    .ToList();
```

### 4. 병합 옵션

```csharp
// 즉시 병합 (낮은 지연시간)
var result = numbers
    .AsParallel()
    .WithMergeOptions(ParallelMergeOptions.NotBuffered)
    .Where(n => IsPrime(n))
    .ToList();

// 완전 버퍼링 (높은 처리량)
var result2 = numbers
    .AsParallel()
    .WithMergeOptions(ParallelMergeOptions.FullyBuffered)
    .Where(n => IsPrime(n))
    .ToList();

// 자동 버퍼링 (기본값)
var result3 = numbers
    .AsParallel()
    .WithMergeOptions(ParallelMergeOptions.AutoBuffered)
    .Where(n => IsPrime(n))
    .ToList();
```

---

## 🔍 PLINQ 연산자

### 1. ForAll - 병렬 반복

```csharp
// ForAll: 각 요소에 대해 병렬 실행 (순서 무관)
numbers
    .AsParallel()
    .Where(n => IsPrime(n))
    .ForAll(n => Console.WriteLine($"Prime: {n}"));  // 순서 보장 안됨

// vs 순차 foreach
foreach (var n in numbers.AsParallel().Where(n => IsPrime(n)))
{
    Console.WriteLine($"Prime: {n}");  // 순차 실행
}
```

### 2. Aggregate - 병렬 집계

```csharp
// 병렬 Sum
var sum = numbers
    .AsParallel()
    .Sum();

// 커스텀 Aggregate
var product = numbers
    .AsParallel()
    .Aggregate(
        seed: 1,                           // 초기값
        updateAccumulatorFunc: (acc, n) => acc * n,  // 각 스레드에서 실행
        combineAccumulatorsFunc: (acc1, acc2) => acc1 * acc2,  // 결과 병합
        resultSelector: result => result   // 최종 변환
    );
```

### 3. 그룹화 및 조인

```csharp
// 병렬 GroupBy
var groups = numbers
    .AsParallel()
    .GroupBy(n => n % 10)
    .ToList();

// 병렬 Join
var customers = GetCustomers().AsParallel();
var orders = GetOrders().AsParallel();

var result = customers
    .Join(
        orders,
        c => c.Id,
        o => o.CustomerId,
        (c, o) => new { Customer = c.Name, Order = o.Total }
    )
    .ToList();
```

---

## 💻 실전 예제

### 예제 1: 대용량 데이터 필터링

```csharp
using System;
using System.Linq;
using System.Diagnostics;

class Program
{
    static void Main()
    {
        var data = Enumerable.Range(1, 10_000_000).ToList();

        // 순차 처리
        var sw1 = Stopwatch.StartNew();
        var result1 = data
            .Where(n => IsExpensive(n))
            .ToList();
        sw1.Stop();
        Console.WriteLine($"순차: {sw1.ElapsedMilliseconds}ms, 결과: {result1.Count}");

        // 병렬 처리
        var sw2 = Stopwatch.StartNew();
        var result2 = data
            .AsParallel()
            .Where(n => IsExpensive(n))
            .ToList();
        sw2.Stop();
        Console.WriteLine($"병렬: {sw2.ElapsedMilliseconds}ms, 결과: {result2.Count}");
    }

    static bool IsExpensive(int n)
    {
        // 계산 비용이 높은 연산 시뮬레이션
        double result = 0;
        for (int i = 0; i < 100; i++)
        {
            result += Math.Sqrt(n * i);
        }
        return n % 2 == 0;
    }
}

// 출력 예:
// 순차: 2500ms, 결과: 5000000
// 병렬: 650ms, 결과: 5000000  (약 4배 빠름)
```

### 예제 2: 이미지 처리

```csharp
using System;
using System.Linq;
using System.Drawing;
using System.IO;

public class ImageProcessor
{
    public void ProcessImages(string[] imagePaths)
    {
        var results = imagePaths
            .AsParallel()
            .WithDegreeOfParallelism(Environment.ProcessorCount)
            .Select(path => new
            {
                Path = path,
                Image = ProcessImage(path)
            })
            .ToList();

        Console.WriteLine($"처리 완료: {results.Count}개 이미지");
    }

    private Bitmap ProcessImage(string path)
    {
        using var original = new Bitmap(path);
        var processed = new Bitmap(original.Width, original.Height);

        // 픽셀 단위 처리를 병렬화
        Enumerable.Range(0, original.Height)
            .AsParallel()
            .ForAll(y =>
            {
                for (int x = 0; x < original.Width; x++)
                {
                    var pixel = original.GetPixel(x, y);
                    var gray = (int)(pixel.R * 0.3 + pixel.G * 0.59 + pixel.B * 0.11);
                    processed.SetPixel(x, y, Color.FromArgb(gray, gray, gray));
                }
            });

        return processed;
    }
}
```

### 예제 3: 파일 검색

```csharp
using System;
using System.Linq;
using System.IO;
using System.Collections.Generic;

public class FileSearcher
{
    public List<string> FindFiles(string rootPath, string pattern)
    {
        var allFiles = Directory.GetFiles(rootPath, "*", SearchOption.AllDirectories);

        var results = allFiles
            .AsParallel()
            .Where(file => ContainsPattern(file, pattern))
            .OrderBy(file => file)  // 순서 보존
            .ToList();

        return results;
    }

    private bool ContainsPattern(string filePath, string pattern)
    {
        try
        {
            string content = File.ReadAllText(filePath);
            return content.Contains(pattern, StringComparison.OrdinalIgnoreCase);
        }
        catch
        {
            return false;
        }
    }
}

// 사용
var searcher = new FileSearcher();
var results = searcher.FindFiles(@"C:\Projects", "TODO");
Console.WriteLine($"찾은 파일: {results.Count}개");
```

### 예제 4: 통계 계산

```csharp
using System;
using System.Linq;

public class Statistics
{
    public static void ComputeStatistics(double[] data)
    {
        // 평균 (병렬)
        var mean = data.AsParallel().Average();

        // 표준편차 (병렬)
        var variance = data
            .AsParallel()
            .Select(x => Math.Pow(x - mean, 2))
            .Average();
        var stdDev = Math.Sqrt(variance);

        // 중앙값 (정렬 필요, 순서 보존)
        var sorted = data
            .AsParallel()
            .AsOrdered()
            .OrderBy(x => x)
            .ToArray();
        var median = sorted[sorted.Length / 2];

        // 최빈값 (병렬 GroupBy)
        var mode = data
            .AsParallel()
            .GroupBy(x => x)
            .OrderByDescending(g => g.Count())
            .First()
            .Key;

        Console.WriteLine($"평균: {mean:F2}");
        Console.WriteLine($"표준편차: {stdDev:F2}");
        Console.WriteLine($"중앙값: {median:F2}");
        Console.WriteLine($"최빈값: {mode:F2}");
    }
}

// 사용
var data = Enumerable.Range(1, 1000000)
    .Select(_ => Random.Shared.NextDouble() * 100)
    .ToArray();
Statistics.ComputeStatistics(data);
```

---

## ⚠️ 주의사항 및 제약

### 1. 언제 PLINQ를 사용하지 말아야 하는가

```csharp
// ❌ 나쁜 예: 연산이 매우 가볍고 데이터가 적음
var result = Enumerable.Range(1, 10)
    .AsParallel()  // 오버헤드가 이득보다 큼
    .Select(x => x * 2)
    .ToList();

// ✅ 좋은 예: 순차 처리가 더 빠름
var result = Enumerable.Range(1, 10)
    .Select(x => x * 2)
    .ToList();

// ❌ 나쁜 예: 순서가 중요하고 병렬화 이득이 적음
var result = data
    .AsParallel()
    .AsOrdered()  // 순서 보존은 성능 저하
    .Select(x => x + 1)
    .ToList();

// ❌ 나쁜 예: 공유 상태 수정 (Race Condition)
int count = 0;
data.AsParallel().ForAll(x =>
{
    count++;  // Race Condition!
});

// ✅ 좋은 예: 스레드 안전 집계
int count = data.AsParallel().Count();
```

### 2. PLINQ 사용 권장 기준

| 조건 | 권장 |
|------|------|
| **데이터 크기** | > 1,000개 |
| **연산 복잡도** | 중간~높음 (각 항목당 >100μs) |
| **순서 중요** | 순서 무관 (AsOrdered는 성능 저하) |
| **부작용** | 없음 (순수 함수) |
| **공유 상태** | 없음 (스레드 안전) |

### 3. 취소 및 예외 처리

```csharp
using System;
using System.Linq;
using System.Threading;

// 취소 지원
var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromSeconds(5));

try
{
    var result = data
        .AsParallel()
        .WithCancellation(cts.Token)  // 취소 토큰
        .Where(n => IsExpensive(n))
        .ToList();
}
catch (OperationCanceledException)
{
    Console.WriteLine("연산이 취소되었습니다.");
}

// 예외 처리
try
{
    var result = data
        .AsParallel()
        .Select(n => ProcessItem(n))  // 예외 발생 가능
        .ToList();
}
catch (AggregateException ae)
{
    // PLINQ는 여러 예외를 AggregateException으로 래핑
    foreach (var ex in ae.InnerExceptions)
    {
        Console.WriteLine($"예외: {ex.Message}");
    }
}
```

---

## 📊 성능 비교

### LINQ vs PLINQ 벤치마크

```csharp
using System;
using System.Linq;
using System.Diagnostics;

public class Benchmark
{
    public static void ComparePerformance()
    {
        var data = Enumerable.Range(1, 10_000_000).ToArray();

        // 테스트 1: 간단한 필터링 (LINQ 우세)
        BenchmarkSimple(data);

        // 테스트 2: 복잡한 계산 (PLINQ 우세)
        BenchmarkComplex(data);
    }

    static void BenchmarkSimple(int[] data)
    {
        var sw1 = Stopwatch.StartNew();
        var result1 = data.Where(x => x % 2 == 0).ToArray();
        sw1.Stop();

        var sw2 = Stopwatch.StartNew();
        var result2 = data.AsParallel().Where(x => x % 2 == 0).ToArray();
        sw2.Stop();

        Console.WriteLine($"간단한 필터링:");
        Console.WriteLine($"  LINQ:  {sw1.ElapsedMilliseconds}ms");
        Console.WriteLine($"  PLINQ: {sw2.ElapsedMilliseconds}ms");
    }

    static void BenchmarkComplex(int[] data)
    {
        var sw1 = Stopwatch.StartNew();
        var result1 = data.Where(x => IsPrime(x)).ToArray();
        sw1.Stop();

        var sw2 = Stopwatch.StartNew();
        var result2 = data.AsParallel().Where(x => IsPrime(x)).ToArray();
        sw2.Stop();

        Console.WriteLine($"복잡한 계산 (소수 판정):");
        Console.WriteLine($"  LINQ:  {sw1.ElapsedMilliseconds}ms");
        Console.WriteLine($"  PLINQ: {sw2.ElapsedMilliseconds}ms");
        Console.WriteLine($"  속도 향상: {(double)sw1.ElapsedMilliseconds / sw2.ElapsedMilliseconds:F2}x");
    }

    static bool IsPrime(int n)
    {
        if (n < 2) return false;
        for (int i = 2; i <= Math.Sqrt(n); i++)
            if (n % i == 0) return false;
        return true;
    }
}

// 예상 결과 (8코어 CPU):
// 간단한 필터링:
//   LINQ:  15ms
//   PLINQ: 25ms  (병렬화 오버헤드가 이득보다 큼)
//
// 복잡한 계산 (소수 판정):
//   LINQ:  12500ms
//   PLINQ: 1800ms  (약 7배 빠름)
```

---

## 🔑 핵심 원칙

### 1. PLINQ 사용 결정 플로우차트

```
데이터가 충분히 큰가? (>1000개)
    │
    NO → LINQ 사용
    │
    YES
    ↓
연산이 CPU 집약적인가?
    │
    NO → LINQ 사용
    │
    YES
    ↓
순서가 중요한가?
    │
    YES → AsOrdered() 사용 (성능 저하 감수)
    │
    NO
    ↓
PLINQ 사용 (AsParallel())
```

### 2. 최적화 팁

```csharp
// 1. 파티셔닝 전략 선택
var result = data
    .AsParallel()
    .WithDegreeOfParallelism(Environment.ProcessorCount)
    .Where(x => Process(x))
    .ToList();

// 2. 조기 필터링으로 데이터 축소
var result = data
    .Where(x => x > 100)  // 순차로 먼저 필터링
    .AsParallel()         // 그 다음 병렬화
    .Select(x => ExpensiveOperation(x))
    .ToList();

// 3. ForAll로 반복 최적화
data
    .AsParallel()
    .Where(x => IsValid(x))
    .ForAll(x => Process(x));  // 순서 무관하면 ForAll 사용
```

---

## 📚 추가 리소스

### 참고 자료
- [PLINQ 공식 문서](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/introduction-to-plinq)
- [Parallel Programming in .NET](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/)
- "Concurrency in C# Cookbook" - Stephen Cleary

### 관련 주제
- [Parallel 클래스](./07-parallel-class.md) - 명령형 병렬 처리
- [Task Parallel Library](./01-thread-task.md) - Task 기반 병렬 처리
- [Concurrent Collections](./05-concurrent-collections.md) - 스레드 안전 컬렉션

---

## 요약

### PLINQ의 장점
- ✅ **선언적**: LINQ 쿼리에 `.AsParallel()` 추가만으로 병렬화
- ✅ **자동 최적화**: 런타임이 자동으로 파티셔닝 및 스케줄링
- ✅ **함수형**: 부작용 없는 순수 함수에 최적
- ✅ **간단한 취소**: CancellationToken 지원

### PLINQ의 단점
- ❌ **오버헤드**: 작은 데이터셋에서는 느림
- ❌ **순서 보존 비용**: AsOrdered()는 성능 저하
- ❌ **디버깅 어려움**: 병렬 실행으로 인한 비결정적 동작

### 황금률
1. **측정하라**: 병렬화가 실제로 빠른지 벤치마크
2. **데이터가 클 때**: 최소 수천 개 이상의 항목
3. **연산이 무거울 때**: 각 항목당 처리 시간이 충분히 길 때
4. **순수 함수**: 공유 상태 없이 부작용이 없을 때

*PLINQ는 강력하지만 만능이 아닙니다. 적절한 상황에서 사용하세요!*
