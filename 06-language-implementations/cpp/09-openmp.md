# OpenMP (Open Multi-Processing)

## 📌 개요

**OpenMP**는 C, C++, Fortran을 위한 병렬 프로그래밍 API로, **컴파일러 지시문(pragma)**을 사용하여 매우 간단하게 병렬화를 구현할 수 있습니다. 기존 순차 코드에 몇 줄만 추가하면 병렬 프로그램으로 변환할 수 있습니다.

**주요 특징:**
- **간단한 사용법**: #pragma 지시문으로 병렬화
- **이식성**: 모든 주요 컴파일러 지원 (GCC, Clang, MSVC, Intel)
- **공유 메모리 모델**: 스레드 간 메모리 공유
- **점진적 병렬화**: 코드 일부만 병렬화 가능
- **성능**: 과학 계산에 최적화

**컴파일:**
```bash
# GCC/Clang
g++ -fopenmp program.cpp -o program

# MSVC
cl /openmp program.cpp

# 실행 시 스레드 수 지정
export OMP_NUM_THREADS=8
./program
```

**헤더:**
```cpp
#include <omp.h>
```

## 1. 기본 사용법

### Hello World

```cpp
#include <omp.h>
#include <iostream>

int main() {
    // 병렬 영역
    #pragma omp parallel
    {
        int thread_id = omp_get_thread_num();
        int num_threads = omp_get_num_threads();

        std::cout << "Hello from thread " << thread_id
                  << " of " << num_threads << "\n";
    }

    return 0;
}

// 출력 (4 스레드):
// Hello from thread 0 of 4
// Hello from thread 2 of 4
// Hello from thread 1 of 4
// Hello from thread 3 of 4
// (순서는 비결정적)
```

### 병렬 for 루프

```cpp
#include <omp.h>
#include <vector>
#include <iostream>

int main() {
    const int N = 1000;
    std::vector<int> data(N);

    // 순차 버전
    for (int i = 0; i < N; ++i) {
        data[i] = i * i;
    }

    // 병렬 버전 - 단 한 줄 추가!
    #pragma omp parallel for
    for (int i = 0; i < N; ++i) {
        data[i] = i * i;
    }

    return 0;
}
```

## 2. 주요 지시문

### parallel for - 병렬 루프

```cpp
#include <omp.h>
#include <vector>
#include <iostream>
#include <chrono>

int main() {
    const int N = 100'000'000;
    std::vector<double> data(N);

    auto start = std::chrono::high_resolution_clock::now();

    // 병렬 for 루프
    #pragma omp parallel for
    for (int i = 0; i < N; ++i) {
        data[i] = std::sin(i * 0.001);
    }

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    std::cout << "병렬 처리 시간: " << duration.count() << "ms\n";

    return 0;
}

// 출력 (8코어):
// 병렬 처리 시간: 850ms (순차: 5200ms)
```

### reduction - 축약 연산

```cpp
#include <omp.h>
#include <vector>
#include <iostream>

int main() {
    std::vector<double> data(100'000'000);

    // 데이터 초기화
    #pragma omp parallel for
    for (size_t i = 0; i < data.size(); ++i) {
        data[i] = i * 0.5;
    }

    // Reduction: 합계 계산
    double sum = 0.0;
    #pragma omp parallel for reduction(+:sum)
    for (size_t i = 0; i < data.size(); ++i) {
        sum += data[i];
    }

    std::cout << "합계: " << sum << "\n";

    // Reduction: 최댓값
    double max_val = data[0];
    #pragma omp parallel for reduction(max:max_val)
    for (size_t i = 0; i < data.size(); ++i) {
        if (data[i] > max_val) {
            max_val = data[i];
        }
    }

    std::cout << "최댓값: " << max_val << "\n";

    // Reduction: 곱셈
    double product = 1.0;
    #pragma omp parallel for reduction(*:product)
    for (int i = 1; i <= 10; ++i) {
        product *= i;  // 10!
    }

    std::cout << "10! = " << product << "\n";

    return 0;
}

// 출력:
// 합계: 2.49999e+15
// 최댓값: 4.99999e+07
// 10! = 3628800
```

### sections - 태스크 병렬화

```cpp
#include <omp.h>
#include <iostream>
#include <chrono>

void task1() {
    std::cout << "Task 1 시작 (Thread " << omp_get_thread_num() << ")\n";
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Task 1 완료\n";
}

void task2() {
    std::cout << "Task 2 시작 (Thread " << omp_get_thread_num() << ")\n";
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Task 2 완료\n";
}

void task3() {
    std::cout << "Task 3 시작 (Thread " << omp_get_thread_num() << ")\n";
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Task 3 완료\n";
}

int main() {
    auto start = std::chrono::high_resolution_clock::now();

    #pragma omp parallel sections
    {
        #pragma omp section
        task1();

        #pragma omp section
        task2();

        #pragma omp section
        task3();
    }

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::seconds>(end - start);

    std::cout << "총 실행 시간: " << duration.count() << "초\n";

    return 0;
}

// 출력:
// Task 1 시작 (Thread 0)
// Task 2 시작 (Thread 1)
// Task 3 시작 (Thread 2)
// Task 1 완료
// Task 2 완료
// Task 3 완료
// 총 실행 시간: 1초 (순차: 3초)
```

### critical - 임계 영역

```cpp
#include <omp.h>
#include <iostream>

int main() {
    int counter = 0;

    // ❌ 잘못된 예: 데이터 레이스
    // #pragma omp parallel for
    // for (int i = 0; i < 10000; ++i) {
    //     counter++;  // Race condition!
    // }

    // ✅ 올바른 예 1: critical
    #pragma omp parallel for
    for (int i = 0; i < 10000; ++i) {
        #pragma omp critical
        {
            counter++;  // 한 번에 하나의 스레드만 실행
        }
    }

    std::cout << "Counter (critical): " << counter << "\n";

    // ✅ 올바른 예 2: atomic (더 빠름)
    counter = 0;
    #pragma omp parallel for
    for (int i = 0; i < 10000; ++i) {
        #pragma omp atomic
        counter++;  // 원자적 증가
    }

    std::cout << "Counter (atomic): " << counter << "\n";

    // ✅ 올바른 예 3: reduction (가장 빠름)
    counter = 0;
    #pragma omp parallel for reduction(+:counter)
    for (int i = 0; i < 10000; ++i) {
        counter += 1;
    }

    std::cout << "Counter (reduction): " << counter << "\n";

    return 0;
}

// 출력:
// Counter (critical): 10000
// Counter (atomic): 10000
// Counter (reduction): 10000
```

## 3. 실전 예제

### 예제 1: 행렬 곱셈

```cpp
#include <omp.h>
#include <vector>
#include <iostream>
#include <chrono>
#include <random>

class Matrix {
    std::vector<std::vector<double>> data;

public:
    size_t rows, cols;

    Matrix(size_t r, size_t c) : rows(r), cols(c), data(r, std::vector<double>(c, 0.0)) {}

    double& at(size_t i, size_t j) { return data[i][j]; }
    const double& at(size_t i, size_t j) const { return data[i][j]; }

    void randomize() {
        std::mt19937 gen(42);
        std::uniform_real_distribution<> dist(0.0, 1.0);

        #pragma omp parallel for collapse(2)
        for (size_t i = 0; i < rows; ++i) {
            for (size_t j = 0; j < cols; ++j) {
                data[i][j] = dist(gen);
            }
        }
    }
};

// 병렬 행렬 곱셈
Matrix multiply(const Matrix& A, const Matrix& B) {
    if (A.cols != B.rows) {
        throw std::invalid_argument("Matrix dimensions mismatch");
    }

    Matrix C(A.rows, B.cols);

    #pragma omp parallel for collapse(2)
    for (size_t i = 0; i < A.rows; ++i) {
        for (size_t j = 0; j < B.cols; ++j) {
            double sum = 0.0;
            for (size_t k = 0; k < A.cols; ++k) {
                sum += A.at(i, k) * B.at(k, j);
            }
            C.at(i, j) = sum;
        }
    }

    return C;
}

int main() {
    const size_t N = 1000;

    Matrix A(N, N), B(N, N);
    A.randomize();
    B.randomize();

    std::cout << "행렬 곱셈 시작 (" << N << "x" << N << ")...\n";

    auto start = std::chrono::high_resolution_clock::now();

    Matrix C = multiply(A, B);

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    std::cout << "완료! 시간: " << duration.count() << "ms\n";

    return 0;
}

// 출력 (8코어):
// 행렬 곱셈 시작 (1000x1000)...
// 완료! 시간: 450ms (순차: 2800ms)
```

### 예제 2: 이미지 처리

```cpp
#include <omp.h>
#include <vector>
#include <cmath>
#include <iostream>
#include <chrono>

struct Pixel {
    unsigned char r, g, b;
};

class Image {
    std::vector<Pixel> pixels;
    size_t width, height;

public:
    Image(size_t w, size_t h) : width(w), height(h), pixels(w * h) {}

    Pixel& at(size_t x, size_t y) {
        return pixels[y * width + x];
    }

    // 그레이스케일 변환
    void toGrayscale() {
        #pragma omp parallel for
        for (size_t i = 0; i < pixels.size(); ++i) {
            auto& p = pixels[i];
            unsigned char gray = static_cast<unsigned char>(
                0.299 * p.r + 0.587 * p.g + 0.114 * p.b
            );
            p.r = p.g = p.b = gray;
        }
    }

    // 밝기 조정
    void adjustBrightness(float factor) {
        #pragma omp parallel for
        for (size_t i = 0; i < pixels.size(); ++i) {
            auto& p = pixels[i];
            p.r = std::min(255, static_cast<int>(p.r * factor));
            p.g = std::min(255, static_cast<int>(p.g * factor));
            p.b = std::min(255, static_cast<int>(p.b * factor));
        }
    }

    // 가우시안 블러
    void gaussianBlur(int radius) {
        Image temp(width, height);

        #pragma omp parallel for collapse(2)
        for (size_t y = 0; y < height; ++y) {
            for (size_t x = 0; x < width; ++x) {
                int r_sum = 0, g_sum = 0, b_sum = 0, count = 0;

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

                temp.at(x, y) = Pixel{
                    static_cast<unsigned char>(r_sum / count),
                    static_cast<unsigned char>(g_sum / count),
                    static_cast<unsigned char>(b_sum / count)
                };
            }
        }

        pixels = std::move(temp.pixels);
    }

    size_t getPixelCount() const { return pixels.size(); }
};

int main() {
    // 4K 이미지
    Image img(3840, 2160);

    std::cout << "이미지 처리 시작...\n";

    auto start = std::chrono::high_resolution_clock::now();

    img.adjustBrightness(1.2f);
    img.toGrayscale();
    img.gaussianBlur(3);

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    std::cout << "처리 완료: " << img.getPixelCount() << " pixels\n";
    std::cout << "시간: " << duration.count() << "ms\n";

    return 0;
}

// 출력:
// 이미지 처리 시작...
// 처리 완료: 8294400 pixels
// 시간: 120ms (순차: 850ms)
```

### 예제 3: 몬테카를로 시뮬레이션

```cpp
#include <omp.h>
#include <random>
#include <iostream>
#include <chrono>

double estimate_pi(long long num_samples) {
    long long inside_circle = 0;

    #pragma omp parallel
    {
        // 각 스레드마다 독립적인 난수 생성기
        std::mt19937 gen(omp_get_thread_num());
        std::uniform_real_distribution<> dist(0.0, 1.0);

        long long local_inside = 0;

        #pragma omp for
        for (long long i = 0; i < num_samples; ++i) {
            double x = dist(gen);
            double y = dist(gen);

            if (x * x + y * y <= 1.0) {
                ++local_inside;
            }
        }

        #pragma omp atomic
        inside_circle += local_inside;
    }

    return 4.0 * inside_circle / num_samples;
}

int main() {
    const long long N = 1'000'000'000;  // 10억 샘플

    std::cout << "Pi 추정 중 (" << N << " 샘플)...\n";

    auto start = std::chrono::high_resolution_clock::now();

    double pi = estimate_pi(N);

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    std::cout << "추정 Pi: " << pi << "\n";
    std::cout << "실제 Pi: " << M_PI << "\n";
    std::cout << "오차: " << std::abs(pi - M_PI) << "\n";
    std::cout << "시간: " << duration.count() << "ms\n";

    return 0;
}

// 출력 (8코어):
// Pi 추정 중 (1000000000 샘플)...
// 추정 Pi: 3.14159
// 실제 Pi: 3.14159
// 오차: 0.0000265
// 시간: 850ms (순차: 6500ms)
```

## 4. 고급 기능

### collapse - 중첩 루프 병렬화

```cpp
#include <omp.h>
#include <vector>
#include <iostream>

int main() {
    const int N = 1000;
    std::vector<std::vector<int>> matrix(N, std::vector<int>(N));

    // collapse(2): 두 개의 중첩 루프를 하나로 병합
    #pragma omp parallel for collapse(2)
    for (int i = 0; i < N; ++i) {
        for (int j = 0; j < N; ++j) {
            matrix[i][j] = i * j;
        }
    }

    return 0;
}
```

### schedule - 작업 분배 방식

```cpp
#include <omp.h>
#include <iostream>
#include <chrono>

void work(int i) {
    // 불균형한 작업 (i가 클수록 느림)
    for (int j = 0; j < i * 1000; ++j) {
        volatile int x = 0;
        ++x;
    }
}

int main() {
    const int N = 100;

    // static: 균등 분배 (기본)
    auto start = std::chrono::high_resolution_clock::now();
    #pragma omp parallel for schedule(static)
    for (int i = 0; i < N; ++i) {
        work(i);
    }
    auto end = std::chrono::high_resolution_clock::now();
    std::cout << "static: "
              << std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count()
              << "ms\n";

    // dynamic: 동적 분배 (불균형 작업에 적합)
    start = std::chrono::high_resolution_clock::now();
    #pragma omp parallel for schedule(dynamic, 10)  // 청크 크기 10
    for (int i = 0; i < N; ++i) {
        work(i);
    }
    end = std::chrono::high_resolution_clock::now();
    std::cout << "dynamic: "
              << std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count()
              << "ms\n";

    // guided: 적응형 분배
    start = std::chrono::high_resolution_clock::now();
    #pragma omp parallel for schedule(guided)
    for (int i = 0; i < N; ++i) {
        work(i);
    }
    end = std::chrono::high_resolution_clock::now();
    std::cout << "guided: "
              << std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count()
              << "ms\n";

    return 0;
}

// 출력 (불균형 작업):
// static: 850ms  // 불균형으로 인해 느림
// dynamic: 420ms  // 동적 분배로 빠름
// guided: 450ms
```

### task - 동적 태스크

```cpp
#include <omp.h>
#include <iostream>

int fib(int n) {
    if (n < 2) return n;

    int x, y;

    #pragma omp task shared(x)
    x = fib(n - 1);

    #pragma omp task shared(y)
    y = fib(n - 2);

    #pragma omp taskwait  // 두 태스크 완료 대기

    return x + y;
}

int main() {
    int result;

    #pragma omp parallel
    {
        #pragma omp single
        {
            result = fib(40);
        }
    }

    std::cout << "fib(40) = " << result << "\n";

    return 0;
}
```

## 5. 데이터 환경 절

### private, shared, firstprivate

```cpp
#include <omp.h>
#include <iostream>

int main() {
    int shared_var = 100;
    int private_var = 200;

    #pragma omp parallel num_threads(4) \
                         shared(shared_var) \
                         private(private_var)
    {
        int tid = omp_get_thread_num();

        // private_var는 각 스레드마다 독립적
        private_var = tid * 10;

        // shared_var는 모든 스레드가 공유
        #pragma omp critical
        {
            std::cout << "Thread " << tid
                      << ": private=" << private_var
                      << ", shared=" << shared_var << "\n";
        }
    }

    std::cout << "After parallel: private_var=" << private_var << "\n";  // 200 (변경 안됨)

    return 0;
}
```

## 6. 성능 최적화

### 벤치마크

```cpp
#include <omp.h>
#include <vector>
#include <algorithm>
#include <chrono>
#include <iostream>

template<typename Func>
void benchmark(const std::string& name, Func func) {
    auto start = std::chrono::high_resolution_clock::now();
    func();
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    std::cout << name << ": " << duration.count() << "ms\n";
}

int main() {
    const size_t N = 100'000'000;
    std::vector<double> data(N);

    std::cout << "=== OpenMP 성능 비교 ===\n";
    std::cout << "스레드 수: " << omp_get_max_threads() << "\n\n";

    // 순차
    benchmark("순차 for", [&]() {
        for (size_t i = 0; i < N; ++i) {
            data[i] = std::sin(i * 0.001);
        }
    });

    // OpenMP (기본)
    benchmark("OpenMP parallel for", [&]() {
        #pragma omp parallel for
        for (size_t i = 0; i < N; ++i) {
            data[i] = std::sin(i * 0.001);
        }
    });

    // OpenMP (schedule dynamic)
    benchmark("OpenMP (dynamic)", [&]() {
        #pragma omp parallel for schedule(dynamic, 1000)
        for (size_t i = 0; i < N; ++i) {
            data[i] = std::sin(i * 0.001);
        }
    });

    // OpenMP (schedule guided)
    benchmark("OpenMP (guided)", [&]() {
        #pragma omp parallel for schedule(guided)
        for (size_t i = 0; i < N; ++i) {
            data[i] = std::sin(i * 0.001);
        }
    });

    return 0;
}

// 출력 (8코어):
// === OpenMP 성능 비교 ===
// 스레드 수: 8
//
// 순차 for: 5200ms
// OpenMP parallel for: 820ms
// OpenMP (dynamic): 850ms
// OpenMP (guided): 840ms
```

## 7. 모범 사례

### ✅ 권장 사항

```cpp
// 1. 큰 루프에만 병렬화
#pragma omp parallel for
for (size_t i = 0; i < 1000000; ++i) {  // OK: 충분히 큼
    // ...
}

// 2. reduction 사용
double sum = 0.0;
#pragma omp parallel for reduction(+:sum)
for (size_t i = 0; i < N; ++i) {
    sum += data[i];
}

// 3. collapse로 중첩 루프 병렬화
#pragma omp parallel for collapse(2)
for (int i = 0; i < N; ++i) {
    for (int j = 0; j < M; ++j) {
        // ...
    }
}

// 4. 불균형 작업에는 dynamic schedule
#pragma omp parallel for schedule(dynamic)
for (int i = 0; i < N; ++i) {
    irregular_work(i);
}
```

### ❌ 피해야 할 패턴

```cpp
// ❌ 1. 작은 루프 병렬화
#pragma omp parallel for
for (int i = 0; i < 10; ++i) {  // 오버헤드 > 이득
    data[i] = i;
}

// ❌ 2. 의존성 있는 루프
#pragma omp parallel for
for (int i = 1; i < N; ++i) {
    data[i] = data[i-1] + 1;  // ⚠️ 데이터 의존성!
}

// ❌ 3. critical 남용
#pragma omp parallel for
for (int i = 0; i < N; ++i) {
    #pragma omp critical
    {
        counter++;  // reduction 사용하는 것이 훨씬 빠름
    }
}

// ❌ 4. 공유 변수 경쟁
int counter = 0;
#pragma omp parallel for
for (int i = 0; i < N; ++i) {
    counter++;  // ⚠️ 데이터 레이스!
}
```

## 8. 요약

### 핵심 포인트

1. **간단함**: pragma 하나로 병렬화
2. **이식성**: 모든 주요 컴파일러 지원
3. **점진적**: 기존 코드에 점진적 적용 가능
4. **성능**: 과학 계산에 최적화
5. **제어**: schedule, reduction 등으로 세밀한 제어

### 빠른 참조

```cpp
// 병렬 영역
#pragma omp parallel
{
    // 병렬 실행
}

// 병렬 for
#pragma omp parallel for
for (int i = 0; i < N; ++i) { /* ... */ }

// Reduction
double sum = 0;
#pragma omp parallel for reduction(+:sum)
for (int i = 0; i < N; ++i) {
    sum += data[i];
}

// Critical 영역
#pragma omp critical
{
    // 한 번에 하나의 스레드만
}

// Atomic 연산
#pragma omp atomic
counter++;

// Sections
#pragma omp parallel sections
{
    #pragma omp section
    task1();

    #pragma omp section
    task2();
}
```

### 컴파일러 지원

| 컴파일러 | OpenMP 버전 | 플래그 |
|---------|------------|--------|
| GCC 9+ | 5.0 | `-fopenmp` |
| Clang 11+ | 5.0 | `-fopenmp` |
| MSVC 2019+ | 2.0 (제한적) | `/openmp` |
| Intel ICC | 5.1 | `-qopenmp` |

## 참고 자료

- [OpenMP Official Site](https://www.openmp.org/)
- [OpenMP API Specification](https://www.openmp.org/specifications/)
- **Using OpenMP** by Barbara Chapman, Gabriele Jost, and Ruud van der Pas
- **Parallel Programming in OpenMP** by Rohit Chandra et al.
