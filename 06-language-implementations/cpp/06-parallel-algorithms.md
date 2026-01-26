# C++17/20 병렬 알고리즘

## 📌 개요

C++17에서 도입된 **병렬 알고리즘(Parallel Algorithms)**은 기존 STL 알고리즘에 **실행 정책(Execution Policies)**을 추가하여 자동 병렬화를 제공합니다. 간단한 정책 파라미터 하나로 순차 알고리즘을 병렬 알고리즘으로 변환할 수 있습니다.

**주요 특징:**
- 기존 STL 알고리즘과 완벽한 호환성
- 실행 정책만 변경하여 병렬화
- 컴파일러/라이브러리가 최적화 담당
- 표준 C++로 이식성 높음

**헤더:**
```cpp
#include <algorithm>
#include <execution>  // C++17
```

## 1. 실행 정책 (Execution Policies)

### 정책 종류

```cpp
#include <algorithm>
#include <execution>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> vec = {5, 2, 8, 1, 9, 3, 7, 4, 6};

    // 1. std::execution::seq - 순차 실행
    std::sort(std::execution::seq, vec.begin(), vec.end());

    // 2. std::execution::par - 병렬 실행
    std::sort(std::execution::par, vec.begin(), vec.end());

    // 3. std::execution::par_unseq - 병렬 + 벡터화
    std::sort(std::execution::par_unseq, vec.begin(), vec.end());

    // 4. std::execution::unseq - 벡터화만 (C++20)
    std::sort(std::execution::unseq, vec.begin(), vec.end());

    return 0;
}
```

### 정책 비교

| 정책 | 병렬화 | 벡터화 | 스레드 안전 요구사항 | 사용 사례 |
|-----|-------|--------|-------------------|----------|
| **seq** | ❌ | ❌ | 없음 | 기본, 순차 실행 |
| **par** | ✅ | ❌ | 스레드 안전 필요 | CPU 집약적 작업 |
| **par_unseq** | ✅ | ✅ | 스레드 안전 + 데이터 레이스 없음 | 최고 성능 |
| **unseq** | ❌ | ✅ | 데이터 레이스 없음 | SIMD 최적화 |

### 실행 정책 선택 가이드

```cpp
// 안전하지만 느림
std::sort(std::execution::seq, v.begin(), v.end());

// 대부분의 경우 이것 사용
std::sort(std::execution::par, v.begin(), v.end());

// 매우 단순한 연산 (예: 산술 연산만)
std::sort(std::execution::par_unseq, v.begin(), v.end());
```

## 2. 병렬화 가능한 알고리즘

### 비수정 시퀀스 연산

```cpp
#include <algorithm>
#include <execution>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> numbers(10'000'000);
    std::iota(numbers.begin(), numbers.end(), 0);

    // all_of - 모든 요소가 조건 만족?
    bool all_positive = std::all_of(
        std::execution::par,
        numbers.begin(), numbers.end(),
        [](int n) { return n >= 0; }
    );
    std::cout << "All positive: " << all_positive << "\n";

    // any_of - 어떤 요소라도 조건 만족?
    bool has_large = std::any_of(
        std::execution::par,
        numbers.begin(), numbers.end(),
        [](int n) { return n > 5'000'000; }
    );

    // none_of - 어떤 요소도 조건 만족 안함?
    bool no_negative = std::none_of(
        std::execution::par,
        numbers.begin(), numbers.end(),
        [](int n) { return n < 0; }
    );

    // count_if - 조건 만족하는 요소 개수
    auto count = std::count_if(
        std::execution::par,
        numbers.begin(), numbers.end(),
        [](int n) { return n % 2 == 0; }
    );
    std::cout << "Even numbers: " << count << "\n";

    // find_if - 조건 만족하는 첫 요소
    auto it = std::find_if(
        std::execution::par,
        numbers.begin(), numbers.end(),
        [](int n) { return n > 1'000'000; }
    );

    return 0;
}
```

### 수정 시퀀스 연산

```cpp
#include <algorithm>
#include <execution>
#include <vector>
#include <iostream>
#include <random>

int main() {
    std::vector<int> data(1'000'000);
    std::vector<int> result(1'000'000);

    // fill - 값으로 채우기
    std::fill(std::execution::par, data.begin(), data.end(), 42);

    // generate - 생성 함수로 채우기
    std::mt19937 gen;
    std::generate(std::execution::par, data.begin(), data.end(),
        [&gen]() { return gen() % 100; }
    );

    // transform - 변환
    std::transform(
        std::execution::par,
        data.begin(), data.end(),
        result.begin(),
        [](int x) { return x * x; }
    );

    // replace_if - 조건부 치환
    std::replace_if(
        std::execution::par,
        data.begin(), data.end(),
        [](int x) { return x < 50; },
        0
    );

    // copy_if - 조건부 복사
    std::vector<int> filtered;
    std::copy_if(
        std::execution::par,
        data.begin(), data.end(),
        std::back_inserter(filtered),
        [](int x) { return x % 2 == 0; }
    );

    return 0;
}
```

### 정렬 및 파티셔닝

```cpp
#include <algorithm>
#include <execution>
#include <vector>
#include <chrono>
#include <iostream>

int main() {
    const int N = 10'000'000;
    std::vector<int> data(N);

    // 랜덤 데이터 생성
    std::mt19937 gen;
    std::generate(data.begin(), data.end(), [&gen]() { return gen() % 1000; });

    // 순차 정렬
    auto data_seq = data;
    auto start = std::chrono::high_resolution_clock::now();
    std::sort(std::execution::seq, data_seq.begin(), data_seq.end());
    auto end = std::chrono::high_resolution_clock::now();
    auto seq_time = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << "순차 정렬: " << seq_time.count() << "ms\n";

    // 병렬 정렬
    auto data_par = data;
    start = std::chrono::high_resolution_clock::now();
    std::sort(std::execution::par, data_par.begin(), data_par.end());
    end = std::chrono::high_resolution_clock::now();
    auto par_time = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << "병렬 정렬: " << par_time.count() << "ms\n";
    std::cout << "가속: " << (double)seq_time.count() / par_time.count() << "x\n";

    // stable_sort - 안정 정렬
    std::stable_sort(std::execution::par, data.begin(), data.end());

    // partial_sort - 부분 정렬
    std::partial_sort(
        std::execution::par,
        data.begin(), data.begin() + 100, data.end()
    );

    // nth_element - N번째 요소 찾기
    std::nth_element(
        std::execution::par,
        data.begin(), data.begin() + N/2, data.end()
    );

    // partition - 파티셔닝
    std::partition(
        std::execution::par,
        data.begin(), data.end(),
        [](int x) { return x % 2 == 0; }
    );

    return 0;
}

// 출력 예시 (8코어 시스템):
// 순차 정렬: 1250ms
// 병렬 정렬: 280ms
// 가속: 4.46x
```

### 집계 연산

```cpp
#include <numeric>
#include <execution>
#include <vector>
#include <iostream>
#include <chrono>

int main() {
    const int N = 100'000'000;
    std::vector<double> data(N);

    // 데이터 초기화
    std::generate(data.begin(), data.end(),
        [n = 0]() mutable { return std::sin(n++ * 0.001); });

    // reduce - 병렬 축약
    auto start = std::chrono::high_resolution_clock::now();
    double sum_par = std::reduce(
        std::execution::par,
        data.begin(), data.end(),
        0.0
    );
    auto end = std::chrono::high_resolution_clock::now();
    auto par_time = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    std::cout << "병렬 합계: " << sum_par << " (" << par_time.count() << "ms)\n";

    // transform_reduce - 변환 후 축약
    double sum_squares = std::transform_reduce(
        std::execution::par,
        data.begin(), data.end(),
        0.0,
        std::plus<>(),
        [](double x) { return x * x; }
    );
    std::cout << "제곱의 합: " << sum_squares << "\n";

    // inclusive_scan - 누적 합 (prefix sum)
    std::vector<double> prefix_sum(N);
    std::inclusive_scan(
        std::execution::par,
        data.begin(), data.end(),
        prefix_sum.begin()
    );

    // exclusive_scan
    std::vector<double> exclusive_sum(N);
    std::exclusive_scan(
        std::execution::par,
        data.begin(), data.end(),
        exclusive_sum.begin(),
        0.0
    );

    // transform_inclusive_scan - 변환 후 누적
    std::vector<double> transformed_scan(N);
    std::transform_inclusive_scan(
        std::execution::par,
        data.begin(), data.end(),
        transformed_scan.begin(),
        std::plus<>(),
        [](double x) { return x * 2; }
    );

    return 0;
}
```

## 3. 실전 예제

### 예제 1: 대용량 이미지 처리

```cpp
#include <algorithm>
#include <execution>
#include <vector>
#include <cmath>
#include <iostream>
#include <chrono>

struct Pixel {
    uint8_t r, g, b, a;
};

class Image {
    std::vector<Pixel> pixels;
    size_t width, height;

public:
    Image(size_t w, size_t h) : width(w), height(h), pixels(w * h) {}

    // 그레이스케일 변환 (병렬)
    void toGrayscale() {
        std::transform(
            std::execution::par_unseq,
            pixels.begin(), pixels.end(),
            pixels.begin(),
            [](const Pixel& p) {
                uint8_t gray = static_cast<uint8_t>(
                    0.299 * p.r + 0.587 * p.g + 0.114 * p.b
                );
                return Pixel{gray, gray, gray, p.a};
            }
        );
    }

    // 밝기 조정 (병렬)
    void adjustBrightness(float factor) {
        std::transform(
            std::execution::par_unseq,
            pixels.begin(), pixels.end(),
            pixels.begin(),
            [factor](const Pixel& p) {
                return Pixel{
                    static_cast<uint8_t>(std::min(255.0f, p.r * factor)),
                    static_cast<uint8_t>(std::min(255.0f, p.g * factor)),
                    static_cast<uint8_t>(std::min(255.0f, p.b * factor)),
                    p.a
                };
            }
        );
    }

    // 임계값 적용 (병렬)
    void threshold(uint8_t value) {
        std::transform(
            std::execution::par,
            pixels.begin(), pixels.end(),
            pixels.begin(),
            [value](const Pixel& p) {
                uint8_t gray = static_cast<uint8_t>(
                    0.299 * p.r + 0.587 * p.g + 0.114 * p.b
                );
                uint8_t binary = gray >= value ? 255 : 0;
                return Pixel{binary, binary, binary, p.a};
            }
        );
    }

    size_t getPixelCount() const { return pixels.size(); }
};

int main() {
    // 4K 이미지 (3840 x 2160)
    Image img(3840, 2160);

    auto start = std::chrono::high_resolution_clock::now();

    // 이미지 처리 파이프라인
    img.adjustBrightness(1.2f);
    img.toGrayscale();
    img.threshold(128);

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    std::cout << "처리 완료: " << img.getPixelCount() << " pixels\n";
    std::cout << "소요 시간: " << duration.count() << "ms\n";

    return 0;
}

// 출력 예시 (8코어):
// 처리 완료: 8294400 pixels
// 소요 시간: 35ms (순차: 180ms)
```

### 예제 2: 통계 계산

```cpp
#include <algorithm>
#include <numeric>
#include <execution>
#include <vector>
#include <cmath>
#include <iostream>

class Statistics {
    std::vector<double> data;

public:
    explicit Statistics(std::vector<double> d) : data(std::move(d)) {}

    // 평균
    double mean() const {
        return std::reduce(
            std::execution::par,
            data.begin(), data.end(),
            0.0
        ) / data.size();
    }

    // 분산
    double variance() const {
        double m = mean();
        return std::transform_reduce(
            std::execution::par,
            data.begin(), data.end(),
            0.0,
            std::plus<>(),
            [m](double x) { return (x - m) * (x - m); }
        ) / data.size();
    }

    // 표준편차
    double stddev() const {
        return std::sqrt(variance());
    }

    // 중앙값 (병렬 정렬 후)
    double median() {
        auto copy = data;
        std::sort(std::execution::par, copy.begin(), copy.end());

        size_t n = copy.size();
        if (n % 2 == 0) {
            return (copy[n/2 - 1] + copy[n/2]) / 2.0;
        } else {
            return copy[n/2];
        }
    }

    // 최솟값과 최댓값
    std::pair<double, double> minmax() const {
        auto [min_it, max_it] = std::minmax_element(
            std::execution::par,
            data.begin(), data.end()
        );
        return {*min_it, *max_it};
    }

    // 범위 (range)
    double range() const {
        auto [min_val, max_val] = minmax();
        return max_val - min_val;
    }
};

int main() {
    // 대용량 데이터 생성 (1억 개)
    std::vector<double> data(100'000'000);
    std::mt19937 gen(42);
    std::normal_distribution<double> dist(100.0, 15.0);
    std::generate(data.begin(), data.end(), [&]() { return dist(gen); });

    Statistics stats(std::move(data));

    auto start = std::chrono::high_resolution_clock::now();

    std::cout << "=== 통계 분석 ===\n";
    std::cout << "평균: " << stats.mean() << "\n";
    std::cout << "표준편차: " << stats.stddev() << "\n";
    std::cout << "중앙값: " << stats.median() << "\n";

    auto [min_val, max_val] = stats.minmax();
    std::cout << "최솟값: " << min_val << "\n";
    std::cout << "최댓값: " << max_val << "\n";
    std::cout << "범위: " << stats.range() << "\n";

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    std::cout << "\n소요 시간: " << duration.count() << "ms\n";

    return 0;
}

// 출력 예시:
// === 통계 분석 ===
// 평균: 99.9987
// 표준편차: 14.9991
// 중앙값: 99.9965
// 최솟값: 34.5621
// 최댓값: 165.432
// 범위: 130.870
//
// 소요 시간: 850ms (순차: 3200ms)
```

### 예제 3: 텍스트 처리

```cpp
#include <algorithm>
#include <execution>
#include <string>
#include <vector>
#include <cctype>
#include <iostream>
#include <fstream>
#include <sstream>

class TextProcessor {
public:
    // 모든 라인을 대문자로 변환
    static void toUpperCase(std::vector<std::string>& lines) {
        std::for_each(
            std::execution::par,
            lines.begin(), lines.end(),
            [](std::string& line) {
                std::transform(line.begin(), line.end(), line.begin(),
                    [](char c) { return std::toupper(c); });
            }
        );
    }

    // 특정 단어 포함하는 라인 찾기
    static size_t countLinesContaining(
        const std::vector<std::string>& lines,
        const std::string& word)
    {
        return std::count_if(
            std::execution::par,
            lines.begin(), lines.end(),
            [&word](const std::string& line) {
                return line.find(word) != std::string::npos;
            }
        );
    }

    // 라인 길이 통계
    static std::vector<size_t> getLineLengths(
        const std::vector<std::string>& lines)
    {
        std::vector<size_t> lengths(lines.size());
        std::transform(
            std::execution::par,
            lines.begin(), lines.end(),
            lengths.begin(),
            [](const std::string& line) { return line.length(); }
        );
        return lengths;
    }

    // 빈 라인 제거
    static void removeEmptyLines(std::vector<std::string>& lines) {
        auto new_end = std::remove_if(
            std::execution::par,
            lines.begin(), lines.end(),
            [](const std::string& line) {
                return line.empty() ||
                       std::all_of(line.begin(), line.end(), ::isspace);
            }
        );
        lines.erase(new_end, lines.end());
    }

    // 라인 정렬
    static void sortLines(std::vector<std::string>& lines) {
        std::sort(std::execution::par, lines.begin(), lines.end());
    }
};

int main() {
    // 대용량 텍스트 데이터 시뮬레이션
    std::vector<std::string> lines;
    for (int i = 0; i < 1'000'000; ++i) {
        lines.push_back("This is line number " + std::to_string(i) + " with some text");
        if (i % 10 == 0) {
            lines.push_back("");  // 빈 라인 추가
        }
    }

    std::cout << "초기 라인 수: " << lines.size() << "\n";

    auto start = std::chrono::high_resolution_clock::now();

    // 텍스트 처리 파이프라인
    TextProcessor::removeEmptyLines(lines);
    TextProcessor::toUpperCase(lines);

    size_t count = TextProcessor::countLinesContaining(lines, "NUMBER");
    std::cout << "'NUMBER' 포함 라인: " << count << "\n";

    auto lengths = TextProcessor::getLineLengths(lines);
    double avg_length = std::reduce(
        std::execution::par,
        lengths.begin(), lengths.end(),
        0.0
    ) / lengths.size();
    std::cout << "평균 라인 길이: " << avg_length << "\n";

    TextProcessor::sortLines(lines);

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    std::cout << "최종 라인 수: " << lines.size() << "\n";
    std::cout << "소요 시간: " << duration.count() << "ms\n";

    return 0;
}
```

## 4. 성능 비교

### 벤치마크

```cpp
#include <algorithm>
#include <numeric>
#include <execution>
#include <vector>
#include <chrono>
#include <iostream>
#include <iomanip>

template<typename Func>
auto benchmark(const std::string& name, Func func) {
    auto start = std::chrono::high_resolution_clock::now();
    func();
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    std::cout << std::setw(30) << std::left << name
              << std::setw(10) << std::right << duration.count() << "ms\n";

    return duration;
}

int main() {
    const size_t N = 50'000'000;
    std::vector<int> data(N);
    std::mt19937 gen(42);
    std::generate(data.begin(), data.end(), [&gen]() { return gen() % 1000; });

    std::cout << "=== 병렬 알고리즘 성능 비교 ===\n";
    std::cout << "데이터 크기: " << N << "\n\n";

    // Sort 비교
    std::cout << "## Sort\n";
    auto data_copy = data;
    auto seq_time = benchmark("순차 정렬 (seq)", [&]() {
        std::sort(std::execution::seq, data_copy.begin(), data_copy.end());
    });

    data_copy = data;
    auto par_time = benchmark("병렬 정렬 (par)", [&]() {
        std::sort(std::execution::par, data_copy.begin(), data_copy.end());
    });

    std::cout << "가속비: " << (double)seq_time.count() / par_time.count() << "x\n\n";

    // Reduce 비교
    std::cout << "## Reduce (합계)\n";
    seq_time = benchmark("순차 reduce", [&]() {
        volatile auto sum = std::reduce(std::execution::seq, data.begin(), data.end(), 0LL);
    });

    par_time = benchmark("병렬 reduce", [&]() {
        volatile auto sum = std::reduce(std::execution::par, data.begin(), data.end(), 0LL);
    });

    std::cout << "가속비: " << (double)seq_time.count() / par_time.count() << "x\n\n";

    // Transform 비교
    std::cout << "## Transform (제곱)\n";
    std::vector<int> result(N);
    seq_time = benchmark("순차 transform", [&]() {
        std::transform(std::execution::seq, data.begin(), data.end(),
                      result.begin(), [](int x) { return x * x; });
    });

    par_time = benchmark("병렬 transform", [&]() {
        std::transform(std::execution::par, data.begin(), data.end(),
                      result.begin(), [](int x) { return x * x; });
    });

    std::cout << "가속비: " << (double)seq_time.count() / par_time.count() << "x\n\n";

    // Count_if 비교
    std::cout << "## Count_if (짝수)\n";
    seq_time = benchmark("순차 count_if", [&]() {
        volatile auto count = std::count_if(std::execution::seq, data.begin(), data.end(),
                                           [](int x) { return x % 2 == 0; });
    });

    par_time = benchmark("병렬 count_if", [&]() {
        volatile auto count = std::count_if(std::execution::par, data.begin(), data.end(),
                                           [](int x) { return x % 2 == 0; });
    });

    std::cout << "가속비: " << (double)seq_time.count() / par_time.count() << "x\n";

    return 0;
}

// 출력 예시 (8코어 시스템):
// === 병렬 알고리즘 성능 비교 ===
// 데이터 크기: 50000000
//
// ## Sort
// 순차 정렬 (seq)                  5420ms
// 병렬 정렬 (par)                  1180ms
// 가속비: 4.59x
//
// ## Reduce (합계)
// 순차 reduce                       124ms
// 병렬 reduce                        25ms
// 가속비: 4.96x
//
// ## Transform (제곱)
// 순차 transform                    186ms
// 병렬 transform                     42ms
// 가속비: 4.43x
//
// ## Count_if (짝수)
// 순차 count_if                     168ms
// 병렬 count_if                      35ms
// 가속비: 4.80x
```

## 5. 주의사항 및 모범 사례

### ✅ 권장 사항

```cpp
// 1. 충분히 큰 데이터에만 병렬화 사용
if (data.size() > 10000) {
    std::sort(std::execution::par, data.begin(), data.end());
} else {
    std::sort(data.begin(), data.end());  // 순차가 더 빠름
}

// 2. 스레드 안전한 연산만 사용
std::for_each(std::execution::par, vec.begin(), vec.end(), [](int& x) {
    x = x * x;  // OK: 각 요소 독립적
});

// 3. 간단한 연산에 par_unseq 사용
std::transform(std::execution::par_unseq, v.begin(), v.end(), result.begin(),
    [](int x) { return x + 1; }  // 매우 간단한 연산
);

// 4. 예외 안전성 고려
try {
    std::sort(std::execution::par, data.begin(), data.end());
} catch (const std::exception& e) {
    // 병렬 실행 중 예외 처리
}
```

### ❌ 피해야 할 패턴

```cpp
// ❌ 1. 공유 상태 수정 (데이터 레이스!)
int counter = 0;
std::for_each(std::execution::par, vec.begin(), vec.end(), [&counter](int x) {
    ++counter;  // ⚠️ 데이터 레이스!
});

// ❌ 2. mutex 사용 (par_unseq에서)
std::mutex mtx;
std::for_each(std::execution::par_unseq, vec.begin(), vec.end(), [&mtx](int x) {
    std::lock_guard lock(mtx);  // ⚠️ par_unseq에서 불법!
});

// ❌ 3. 작은 데이터에 병렬화
std::vector<int> small = {1, 2, 3, 4, 5};
std::sort(std::execution::par, small.begin(), small.end());  // 오버헤드 > 이득

// ❌ 4. I/O 작업
std::for_each(std::execution::par, files.begin(), files.end(), [](const auto& file) {
    std::ofstream out(file);  // ⚠️ I/O는 병렬화 부적합
});

// ❌ 5. 복잡한 동기화
std::condition_variable cv;
std::for_each(std::execution::par, vec.begin(), vec.end(), [&cv](int x) {
    cv.wait(...);  // ⚠️ 복잡한 동기화는 병렬 알고리즘에 부적합
});
```

## 6. 컴파일러 지원

### GCC/G++

```bash
# GCC 9+ 필요
g++ -std=c++17 -O3 program.cpp -ltbb

# Intel TBB 라이브러리 필요 (대부분의 구현에서)
sudo apt-get install libtbb-dev  # Ubuntu/Debian
```

### Clang

```bash
# Clang은 아직 병렬 알고리즘을 완전히 지원하지 않음
# libc++ 대신 libstdc++ 사용 권장
clang++ -std=c++17 -O3 -stdlib=libstdc++ program.cpp -ltbb
```

### MSVC

```bash
# Visual Studio 2017 15.7+ 지원
cl /std:c++17 /O2 /EHsc program.cpp
```

### 컴파일러별 지원 상태

| 기능 | GCC | Clang | MSVC |
|-----|-----|-------|------|
| 기본 병렬 알고리즘 | 9+ | 부분적 | VS2017 15.7+ |
| par_unseq | 9+ | 부분적 | VS2019+ |
| unseq (C++20) | 10+ | ❌ | VS2019+ |

## 7. 요약

### 핵심 포인트

1. **간단한 API**: 실행 정책만 추가하면 병렬화
2. **표준 C++**: 이식성 높고 표준 보장
3. **자동 최적화**: 컴파일러/라이브러리가 최적화 담당
4. **성능**: 4-5배 가속 (8코어 기준)
5. **안전성**: 스레드 안전성 필수
6. **제한사항**: 큰 데이터, 단순 연산에 적합

### 빠른 참조

```cpp
// 병렬 정렬
std::sort(std::execution::par, vec.begin(), vec.end());

// 병렬 변환
std::transform(std::execution::par, v.begin(), v.end(), result.begin(),
    [](int x) { return x * 2; });

// 병렬 축약
auto sum = std::reduce(std::execution::par, v.begin(), v.end(), 0);

// 병렬 카운트
auto count = std::count_if(std::execution::par, v.begin(), v.end(),
    [](int x) { return x > 50; });
```

### 언제 사용할까?

✅ **병렬 알고리즘 사용:**
- 대용량 데이터 (>10,000 요소)
- CPU 집약적 연산
- 독립적인 요소 처리
- 간단한 변환/집계

❌ **병렬 알고리즘 피하기:**
- 작은 데이터
- I/O 작업
- 복잡한 동기화 필요
- 순서 의존성 있음

## 참고 자료

- [C++17 Parallel Algorithms - cppreference](https://en.cppreference.com/w/cpp/algorithm)
- [P0024R2: The Parallelism TS Should be Standardized](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0024r2.html)
- **C++17 - The Complete Guide** by Nicolai M. Josuttis
- **C++ Concurrency in Action** (2nd edition) by Anthony Williams
