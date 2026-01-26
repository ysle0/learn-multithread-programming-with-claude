# Intel TBB (Threading Building Blocks)

## 📌 개요

**Intel Threading Building Blocks (TBB)**는 C++를 위한 고성능 병렬 프로그래밍 라이브러리입니다. 스레드를 직접 관리하는 대신 **태스크 기반 병렬화**를 제공하여, 개발자가 병렬화 로직에 집중할 수 있게 합니다.

**주요 특징:**
- 태스크 기반 병렬화 (스레드 풀 자동 관리)
- 풍부한 병렬 알고리즘 (parallel_for, parallel_reduce 등)
- 동시성 컨테이너 (concurrent_vector, concurrent_hash_map 등)
- 메모리 할당자 (scalable_allocator)
- Work-stealing 스케줄러로 효율적인 로드 밸런싱

**설치:**
```bash
# Ubuntu/Debian
sudo apt-get install libtbb-dev

# macOS (Homebrew)
brew install tbb

# Windows (vcpkg)
vcpkg install tbb

# CMake에서
find_package(TBB REQUIRED)
target_link_libraries(your_target TBB::tbb)
```

**헤더:**
```cpp
#include <tbb/parallel_for.h>
#include <tbb/parallel_reduce.h>
#include <tbb/task_group.h>
#include <tbb/concurrent_vector.h>
```

## 1. parallel_for - 병렬 루프

### 기본 사용법

```cpp
#include <tbb/parallel_for.h>
#include <tbb/blocked_range.h>
#include <vector>
#include <iostream>

int main() {
    const size_t N = 1000;
    std::vector<int> data(N);

    // 순차 for 루프
    for (size_t i = 0; i < N; ++i) {
        data[i] = i * i;
    }

    // TBB parallel_for
    tbb::parallel_for(
        tbb::blocked_range<size_t>(0, N),
        [&](const tbb::blocked_range<size_t>& r) {
            for (size_t i = r.begin(); i != r.end(); ++i) {
                data[i] = i * i;
            }
        }
    );

    // 더 간단한 버전
    tbb::parallel_for(size_t(0), N, [&](size_t i) {
        data[i] = i * i;
    });

    return 0;
}
```

### 실전 예제: 이미지 처리

```cpp
#include <tbb/parallel_for.h>
#include <tbb/blocked_range2d.h>
#include <vector>
#include <cmath>
#include <iostream>
#include <chrono>

struct RGB {
    unsigned char r, g, b;
};

class Image {
    std::vector<RGB> pixels;
    size_t width, height;

public:
    Image(size_t w, size_t h) : width(w), height(h), pixels(w * h) {}

    RGB& at(size_t x, size_t y) {
        return pixels[y * width + x];
    }

    size_t get_width() const { return width; }
    size_t get_height() const { return height; }

    // 가우시안 블러 (병렬)
    void gaussian_blur(int radius) {
        Image temp(width, height);

        tbb::parallel_for(
            tbb::blocked_range2d<size_t>(0, height, 0, width),
            [&](const tbb::blocked_range2d<size_t>& r) {
                for (size_t y = r.rows().begin(); y != r.rows().end(); ++y) {
                    for (size_t x = r.cols().begin(); x != r.cols().end(); ++x) {
                        int r_sum = 0, g_sum = 0, b_sum = 0, count = 0;

                        // 주변 픽셀 평균
                        for (int dy = -radius; dy <= radius; ++dy) {
                            for (int dx = -radius; dx <= radius; ++dx) {
                                int nx = x + dx;
                                int ny = y + dy;

                                if (nx >= 0 && nx < width && ny >= 0 && ny < height) {
                                    auto& pixel = at(nx, ny);
                                    r_sum += pixel.r;
                                    g_sum += pixel.g;
                                    b_sum += pixel.b;
                                    ++count;
                                }
                            }
                        }

                        temp.at(x, y) = RGB{
                            static_cast<unsigned char>(r_sum / count),
                            static_cast<unsigned char>(g_sum / count),
                            static_cast<unsigned char>(b_sum / count)
                        };
                    }
                }
            }
        );

        pixels = std::move(temp.pixels);
    }

    // 밝기 조정 (병렬)
    void adjust_brightness(float factor) {
        tbb::parallel_for(size_t(0), height, [&](size_t y) {
            for (size_t x = 0; x < width; ++x) {
                auto& pixel = at(x, y);
                pixel.r = std::min(255, static_cast<int>(pixel.r * factor));
                pixel.g = std::min(255, static_cast<int>(pixel.g * factor));
                pixel.b = std::min(255, static_cast<int>(pixel.b * factor));
            }
        });
    }
};

int main() {
    // 4K 이미지
    Image img(3840, 2160);

    auto start = std::chrono::high_resolution_clock::now();

    img.adjust_brightness(1.2f);
    img.gaussian_blur(3);

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    std::cout << "이미지 처리 완료: " << duration.count() << "ms\n";

    return 0;
}

// 출력 예시:
// 이미지 처리 완료: 85ms (순차: 420ms)
```

## 2. parallel_reduce - 병렬 축약

### 기본 사용법

```cpp
#include <tbb/parallel_reduce.h>
#include <tbb/blocked_range.h>
#include <vector>
#include <iostream>

int main() {
    std::vector<double> data(10'000'000);
    for (size_t i = 0; i < data.size(); ++i) {
        data[i] = i * 0.5;
    }

    // parallel_reduce로 합계 계산
    double sum = tbb::parallel_reduce(
        tbb::blocked_range<size_t>(0, data.size()),
        0.0,  // 초기값
        // Reduction 함수
        [&](const tbb::blocked_range<size_t>& r, double init) -> double {
            for (size_t i = r.begin(); i != r.end(); ++i) {
                init += data[i];
            }
            return init;
        },
        // Join 함수
        [](double x, double y) -> double {
            return x + y;
        }
    );

    std::cout << "합계: " << sum << "\n";

    return 0;
}
```

### 실전 예제: 통계 계산

```cpp
#include <tbb/parallel_reduce.h>
#include <tbb/blocked_range.h>
#include <vector>
#include <cmath>
#include <iostream>
#include <algorithm>

struct Statistics {
    double sum = 0.0;
    double sum_sq = 0.0;
    double min_val = std::numeric_limits<double>::max();
    double max_val = std::numeric_limits<double>::lowest();
    size_t count = 0;

    // Join 연산
    Statistics& operator+=(const Statistics& other) {
        sum += other.sum;
        sum_sq += other.sum_sq;
        min_val = std::min(min_val, other.min_val);
        max_val = std::max(max_val, other.max_val);
        count += other.count;
        return *this;
    }

    double mean() const { return sum / count; }
    double variance() const {
        double m = mean();
        return (sum_sq / count) - (m * m);
    }
    double stddev() const { return std::sqrt(variance()); }
    double range() const { return max_val - min_val; }
};

Statistics compute_statistics(const std::vector<double>& data) {
    return tbb::parallel_reduce(
        tbb::blocked_range<size_t>(0, data.size()),
        Statistics{},  // 초기값
        // 로컬 축약
        [&](const tbb::blocked_range<size_t>& r, Statistics init) -> Statistics {
            for (size_t i = r.begin(); i != r.end(); ++i) {
                double val = data[i];
                init.sum += val;
                init.sum_sq += val * val;
                init.min_val = std::min(init.min_val, val);
                init.max_val = std::max(init.max_val, val);
                ++init.count;
            }
            return init;
        },
        // Join
        [](Statistics x, Statistics y) -> Statistics {
            x += y;
            return x;
        }
    );
}

int main() {
    std::vector<double> data(100'000'000);

    // 정규분포 데이터 생성
    std::mt19937 gen(42);
    std::normal_distribution<> dist(100.0, 15.0);
    for (auto& val : data) {
        val = dist(gen);
    }

    auto start = std::chrono::high_resolution_clock::now();

    auto stats = compute_statistics(data);

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    std::cout << "=== 통계 분석 ===\n";
    std::cout << "데이터 개수: " << stats.count << "\n";
    std::cout << "평균: " << stats.mean() << "\n";
    std::cout << "표준편차: " << stats.stddev() << "\n";
    std::cout << "최솟값: " << stats.min_val << "\n";
    std::cout << "최댓값: " << stats.max_val << "\n";
    std::cout << "범위: " << stats.range() << "\n";
    std::cout << "처리 시간: " << duration.count() << "ms\n";

    return 0;
}

// 출력 예시:
// === 통계 분석 ===
// 데이터 개수: 100000000
// 평균: 99.9991
// 표준편차: 14.9987
// 최솟값: 34.5612
// 최댓값: 165.421
// 범위: 130.860
// 처리 시간: 180ms (순차: 850ms)
```

## 3. parallel_scan - 병렬 스캔 (Prefix Sum)

```cpp
#include <tbb/parallel_scan.h>
#include <tbb/blocked_range.h>
#include <vector>
#include <iostream>

class PrefixSum {
    const std::vector<int>& input;
    std::vector<int>& output;
    int sum;

public:
    PrefixSum(const std::vector<int>& in, std::vector<int>& out)
        : input(in), output(out), sum(0) {}

    PrefixSum(PrefixSum& other, tbb::split)
        : input(other.input), output(other.output), sum(0) {}

    // Scan 연산
    template<typename Tag>
    void operator()(const tbb::blocked_range<size_t>& r, Tag) {
        int temp = sum;

        for (size_t i = r.begin(); i != r.end(); ++i) {
            temp += input[i];
            if (Tag::is_final_scan()) {
                output[i] = temp;
            }
        }

        sum = temp;
    }

    // Reverse join
    void reverse_join(PrefixSum& other) {
        sum += other.sum;
    }

    // Assign
    void assign(PrefixSum& other) {
        sum = other.sum;
    }
};

int main() {
    const size_t N = 10'000'000;
    std::vector<int> data(N);
    std::vector<int> prefix_sum(N);

    // 데이터 초기화
    for (size_t i = 0; i < N; ++i) {
        data[i] = 1;  // 모두 1로 설정 (결과는 1, 2, 3, ...)
    }

    auto start = std::chrono::high_resolution_clock::now();

    PrefixSum scanner(data, prefix_sum);
    tbb::parallel_scan(tbb::blocked_range<size_t>(0, N), scanner);

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    std::cout << "Prefix sum 계산 완료: " << duration.count() << "ms\n";
    std::cout << "처음 10개: ";
    for (int i = 0; i < 10; ++i) {
        std::cout << prefix_sum[i] << " ";
    }
    std::cout << "\n";

    return 0;
}

// 출력:
// Prefix sum 계산 완료: 25ms (순차: 120ms)
// 처음 10개: 1 2 3 4 5 6 7 8 9 10
```

## 4. task_group - 태스크 기반 병렬화

```cpp
#include <tbb/task_group.h>
#include <iostream>
#include <vector>
#include <chrono>

// Fibonacci (재귀, 병렬)
int fib_parallel(int n) {
    if (n < 2) return n;

    int x, y;
    tbb::task_group g;

    g.run([&] { x = fib_parallel(n - 1); });  // 태스크 1
    g.run([&] { y = fib_parallel(n - 2); });  // 태스크 2

    g.wait();  // 모든 태스크 완료 대기

    return x + y;
}

// 더 효율적인 버전 (작은 n은 순차)
int fib_optimized(int n, int cutoff = 20) {
    if (n < cutoff) {
        // 작은 문제는 순차 처리
        if (n < 2) return n;
        return fib_optimized(n - 1, cutoff) + fib_optimized(n - 2, cutoff);
    }

    int x, y;
    tbb::task_group g;

    g.run([&] { x = fib_optimized(n - 1, cutoff); });
    y = fib_optimized(n - 2, cutoff);  // 현재 스레드에서 처리

    g.wait();

    return x + y;
}

int main() {
    const int N = 40;

    auto start = std::chrono::high_resolution_clock::now();
    int result = fib_optimized(N);
    auto end = std::chrono::high_resolution_clock::now();

    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    std::cout << "fib(" << N << ") = " << result << "\n";
    std::cout << "시간: " << duration.count() << "ms\n";

    return 0;
}

// 출력:
// fib(40) = 102334155
// 시간: 85ms (순차: 850ms)
```

## 5. concurrent_vector - 동시성 벡터

```cpp
#include <tbb/concurrent_vector.h>
#include <tbb/parallel_for.h>
#include <iostream>
#include <vector>

int main() {
    tbb::concurrent_vector<int> vec;

    // 병렬로 요소 추가
    tbb::parallel_for(0, 10000, [&](int i) {
        vec.push_back(i);  // 스레드 안전!
    });

    std::cout << "벡터 크기: " << vec.size() << "\n";

    // grow_by로 여러 요소 추가
    auto it = vec.grow_by(100);  // 100개 공간 확보, 반복자 반환
    for (int i = 0; i < 100; ++i) {
        *(it + i) = i * 10;
    }

    // 안전한 반복
    tbb::parallel_for(
        tbb::blocked_range<size_t>(0, vec.size()),
        [&](const tbb::blocked_range<size_t>& r) {
            for (size_t i = r.begin(); i != r.end(); ++i) {
                vec[i] *= 2;  // 읽기/쓰기 안전
            }
        }
    );

    return 0;
}
```

## 6. concurrent_hash_map - 동시성 해시맵

```cpp
#include <tbb/concurrent_hash_map.h>
#include <tbb/parallel_for.h>
#include <string>
#include <iostream>

// 단어 빈도 카운터
class WordCounter {
    using Map = tbb::concurrent_hash_map<std::string, int>;
    Map word_counts;

public:
    void add_word(const std::string& word) {
        Map::accessor acc;

        if (word_counts.insert(acc, word)) {
            // 새 단어
            acc->second = 1;
        } else {
            // 기존 단어
            ++acc->second;
        }
        // acc 소멸 시 자동 언락
    }

    void process_documents(const std::vector<std::vector<std::string>>& docs) {
        tbb::parallel_for(size_t(0), docs.size(), [&](size_t i) {
            for (const auto& word : docs[i]) {
                add_word(word);
            }
        });
    }

    void print_top(int n) const {
        std::vector<std::pair<std::string, int>> items;

        for (auto it = word_counts.begin(); it != word_counts.end(); ++it) {
            items.emplace_back(it->first, it->second);
        }

        std::partial_sort(items.begin(), items.begin() + std::min(n, (int)items.size()),
                         items.end(),
                         [](const auto& a, const auto& b) { return a.second > b.second; });

        std::cout << "Top " << n << " words:\n";
        for (int i = 0; i < std::min(n, (int)items.size()); ++i) {
            std::cout << items[i].first << ": " << items[i].second << "\n";
        }
    }
};

int main() {
    std::vector<std::vector<std::string>> documents = {
        {"the", "quick", "brown", "fox"},
        {"the", "lazy", "dog"},
        {"quick", "brown", "fox", "jumps"},
        // ... 수천 개의 문서
    };

    WordCounter counter;
    counter.process_documents(documents);
    counter.print_top(10);

    return 0;
}
```

## 7. parallel_pipeline - 병렬 파이프라인

```cpp
#include <tbb/parallel_pipeline.h>
#include <tbb/tick_count.h>
#include <iostream>
#include <sstream>
#include <string>

struct Data {
    int id;
    std::string content;
};

int main() {
    const int NUM_ITEMS = 1000;

    auto start = tbb::tick_count::now();

    tbb::parallel_pipeline(
        8,  // 최대 동시 토큰 수

        // Stage 1: 데이터 생성 (순차)
        tbb::make_filter<void, Data>(
            tbb::filter_mode::serial_in_order,
            [count = 0](tbb::flow_control& fc) mutable -> Data {
                if (count >= NUM_ITEMS) {
                    fc.stop();
                    return {};
                }
                return Data{count++, "raw data " + std::to_string(count)};
            }
        ) &

        // Stage 2: 데이터 처리 (병렬)
        tbb::make_filter<Data, Data>(
            tbb::filter_mode::parallel,
            [](Data d) -> Data {
                // 무거운 처리 시뮬레이션
                std::this_thread::sleep_for(std::chrono::milliseconds(1));
                d.content = "processed: " + d.content;
                return d;
            }
        ) &

        // Stage 3: 결과 저장 (순차, 순서 유지)
        tbb::make_filter<Data, void>(
            tbb::filter_mode::serial_in_order,
            [](Data d) {
                // 파일 쓰기 등 (순서 중요)
                // std::cout << "Saving: " << d.id << "\n";
            }
        )
    );

    auto end = tbb::tick_count::now();

    std::cout << "파이프라인 완료: " << (end - start).seconds() << "초\n";

    return 0;
}

// 출력:
// 파이프라인 완료: 0.15초 (순차: 1.0초)
```

## 8. 성능 비교

```cpp
#include <tbb/parallel_for.h>
#include <tbb/parallel_reduce.h>
#include <algorithm>
#include <execution>
#include <vector>
#include <chrono>
#include <iostream>

template<typename Func>
auto benchmark(const std::string& name, Func func) {
    auto start = std::chrono::high_resolution_clock::now();
    func();
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    std::cout << name << ": " << duration.count() << "ms\n";
    return duration;
}

int main() {
    const size_t N = 100'000'000;
    std::vector<int> data(N);

    std::generate(data.begin(), data.end(), []() {
        static int i = 0;
        return i++;
    });

    std::cout << "=== 정렬 성능 비교 ===\n";

    // 순차 정렬
    auto data_copy = data;
    benchmark("순차 (std::sort)", [&]() {
        std::sort(data_copy.begin(), data_copy.end());
    });

    // C++17 병렬 정렬
    data_copy = data;
    benchmark("C++17 병렬", [&]() {
        std::sort(std::execution::par, data_copy.begin(), data_copy.end());
    });

    // TBB 병렬 정렬
    data_copy = data;
    benchmark("TBB parallel_sort", [&]() {
        tbb::parallel_sort(data_copy.begin(), data_copy.end());
    });

    std::cout << "\n=== Reduce 성능 비교 ===\n";

    // 순차 합계
    benchmark("순차 accumulate", [&]() {
        volatile auto sum = std::accumulate(data.begin(), data.end(), 0LL);
    });

    // C++17 병렬 reduce
    benchmark("C++17 reduce", [&]() {
        volatile auto sum = std::reduce(std::execution::par, data.begin(), data.end(), 0LL);
    });

    // TBB parallel_reduce
    benchmark("TBB parallel_reduce", [&]() {
        volatile auto sum = tbb::parallel_reduce(
            tbb::blocked_range<size_t>(0, N),
            0LL,
            [&](const tbb::blocked_range<size_t>& r, long long init) {
                for (size_t i = r.begin(); i != r.end(); ++i) {
                    init += data[i];
                }
                return init;
            },
            std::plus<long long>()
        );
    });

    return 0;
}

// 출력 예시 (8코어):
// === 정렬 성능 비교 ===
// 순차 (std::sort): 12500ms
// C++17 병렬: 2800ms
// TBB parallel_sort: 2600ms
//
// === Reduce 성능 비교 ===
// 순차 accumulate: 280ms
// C++17 reduce: 60ms
// TBB parallel_reduce: 55ms
```

## 9. 모범 사례

### ✅ 권장 사항

```cpp
// 1. 적절한 그레인 크기 선택
tbb::parallel_for(
    tbb::blocked_range<size_t>(0, N, 1000),  // 그레인 크기 1000
    [&](const tbb::blocked_range<size_t>& r) { /* ... */ }
);

// 2. task_group으로 동적 태스크
tbb::task_group g;
for (auto& task : tasks) {
    g.run([&task]() { task.execute(); });
}
g.wait();

// 3. concurrent 컨테이너 사용
tbb::concurrent_vector<int> vec;  // std::vector 대신
tbb::concurrent_hash_map<K, V> map;  // std::map 대신

// 4. auto_partitioner로 자동 분할
tbb::parallel_for(
    tbb::blocked_range<size_t>(0, N),
    [&](const auto& r) { /* ... */ },
    tbb::auto_partitioner()  // 자동 최적화
);
```

### ❌ 피해야 할 패턴

```cpp
// ❌ 너무 작은 그레인 크기
tbb::parallel_for(0, N, [&](int i) {
    data[i]++;  // 오버헤드 > 이득
});

// ❌ concurrent 컨테이너 오용
tbb::concurrent_vector<int> vec;
for (int i = 0; i < N; ++i) {
    vec.push_back(i);  // 순차적으로 추가하면 느림
}

// ❌ 과도한 태스크 생성
for (int i = 0; i < 1000000; ++i) {
    g.run([i]() { /* 매우 작은 작업 */ });  // 너무 많은 태스크!
}
```

## 10. 요약

### 핵심 포인트

1. **태스크 기반**: 스레드 대신 태스크로 관리
2. **Work-stealing**: 효율적인 로드 밸런싱
3. **풍부한 알고리즘**: parallel_for, parallel_reduce, parallel_scan 등
4. **동시성 컨테이너**: concurrent_vector, concurrent_hash_map
5. **성능**: C++17 병렬 알고리즘과 유사하거나 더 나음

### 빠른 참조

```cpp
// parallel_for
tbb::parallel_for(0, N, [&](int i) { /* work */ });

// parallel_reduce
auto sum = tbb::parallel_reduce(
    tbb::blocked_range<size_t>(0, N),
    0,
    [&](auto r, auto init) { /* reduce */ return init; },
    std::plus<>()
);

// task_group
tbb::task_group g;
g.run([]() { /* task 1 */ });
g.run([]() { /* task 2 */ });
g.wait();

// concurrent_vector
tbb::concurrent_vector<int> vec;
tbb::parallel_for(0, N, [&](int i) {
    vec.push_back(i);  // thread-safe
});
```

## 참고 자료

- [Intel TBB Documentation](https://www.intel.com/content/www/us/en/developer/tools/oneapi/onetbb.html)
- [TBB GitHub](https://github.com/oneapi-src/oneTBB)
- **Pro TBB: C++ Parallel Programming with Threading Building Blocks** by Michael Voss
- **Structured Parallel Programming** by Michael McCool, James Reinders, Arch Robison
