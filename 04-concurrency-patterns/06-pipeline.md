# Pipeline 패턴

## 개요

Pipeline 패턴은 일련의 순차적인 stage를 통해 데이터를 처리하며, 각 stage는 특정 변환을 수행합니다. stage들은 동시에 실행되며, 각 stage가 서로 다른 항목을 동시에 처리합니다. 이 패턴은 데이터가 여러 처리 단계를 거치는 스트림 처리에 이상적입니다.

## 문제 정의

많은 데이터 처리 작업에는 여러 순차적 변환이 포함됩니다:
- 비디오 인코딩 (디코드 → 변환 → 인코드)
- 이미지 처리 (로드 → 필터 → 리사이즈 → 저장)
- 데이터 ETL (추출 → 변환 → 적재)
- 컴파일 (파싱 → 최적화 → 코드 생성)

순차 처리는 CPU 시간을 낭비합니다. Pipeline 패턴은 stage 간 병렬성을 가능하게 합니다.

## 솔루션 아키텍처

```
Input Stream
     │
     ▼
┌────────────┐         ┌────────────┐         ┌────────────┐
│  Stage 1   │──queue─▶│  Stage 2   │──queue─▶│  Stage 3   │
│  (Thread)  │         │  (Thread)  │         │  (Thread)  │
└────────────┘         └────────────┘         └────────────┘
     │                      │                      │
   Item A                 Item B                 Item C
                                                    │
                                                    ▼
                                              Output Stream

병렬성:
- Stage 1은 Item D를 처리
- Stage 2는 Item B를 처리 (이미 Stage 1에서 처리 완료)
- Stage 3는 Item C를 처리 (이미 Stage 1과 2에서 처리 완료)

처리량 = min(각 stage의 처리량)
지연 시간 = sum(각 stage의 지연 시간)
```

## 기본 구현

### 간단한 Pipeline

```cpp
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <optional>
#include <vector>

template<typename Input, typename Output>
class Stage {
private:
    std::function<Output(Input)> processor_;
    std::queue<Input> input_queue_;
    Stage<Output, void>* next_stage_ = nullptr;

    mutable std::mutex mutex_;
    std::condition_variable cv_;
    bool stopped_ = false;

public:
    explicit Stage(std::function<Output(Input)> processor)
        : processor_(std::move(processor)) {}

    void set_next(Stage<Output, void>* next) {
        next_stage_ = next;
    }

    void push(Input item) {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            input_queue_.push(std::move(item));
        }
        cv_.notify_one();
    }

    void run() {
        while (true) {
            Input item;

            {
                std::unique_lock<std::mutex> lock(mutex_);
                cv_.wait(lock, [this] {
                    return !input_queue_.empty() || stopped_;
                });

                if (stopped_ && input_queue_.empty()) {
                    // 다음 stage에 중지 알림
                    if (next_stage_) {
                        next_stage_->stop();
                    }
                    break;
                }

                if (input_queue_.empty()) {
                    continue;
                }

                item = std::move(input_queue_.front());
                input_queue_.pop();
            }

            // 항목 처리
            try {
                Output result = processor_(std::move(item));

                // 다음 stage로 전달
                if (next_stage_) {
                    next_stage_->push(std::move(result));
                }
            } catch (const std::exception& e) {
                std::cerr << "Stage 오류: " << e.what() << "\n";
            }
        }
    }

    void stop() {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            stopped_ = true;
        }
        cv_.notify_all();
    }

    size_t queue_size() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return input_queue_.size();
    }
};

// 터미널 stage를 위한 특수화 (출력 없음)
template<typename Input>
class Stage<Input, void> {
private:
    std::function<void(Input)> processor_;
    std::queue<Input> input_queue_;

    mutable std::mutex mutex_;
    std::condition_variable cv_;
    bool stopped_ = false;

public:
    explicit Stage(std::function<void(Input)> processor)
        : processor_(std::move(processor)) {}

    void push(Input item) {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            input_queue_.push(std::move(item));
        }
        cv_.notify_one();
    }

    void run() {
        while (true) {
            Input item;

            {
                std::unique_lock<std::mutex> lock(mutex_);
                cv_.wait(lock, [this] {
                    return !input_queue_.empty() || stopped_;
                });

                if (stopped_ && input_queue_.empty()) {
                    break;
                }

                if (input_queue_.empty()) {
                    continue;
                }

                item = std::move(input_queue_.front());
                input_queue_.pop();
            }

            try {
                processor_(std::move(item));
            } catch (const std::exception& e) {
                std::cerr << "터미널 stage 오류: " << e.what() << "\n";
            }
        }
    }

    void stop() {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            stopped_ = true;
        }
        cv_.notify_all();
    }
};
```

### 완전한 예제: 이미지 처리 Pipeline

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <chrono>

// 이미지 데이터 구조체
struct Image {
    int id;
    std::string filename;
    std::vector<uint8_t> data;
    int width, height;

    Image(int id, const std::string& name, int w, int h)
        : id(id), filename(name), width(w), height(h),
          data(w * h * 3) {}  // RGB
};

// Stage 1: 디스크에서 이미지 로드
Image load_image(const std::string& filename) {
    std::this_thread::sleep_for(std::chrono::milliseconds(50));
    std::cout << "로드 완료: " << filename << "\n";
    return Image(0, filename, 1920, 1080);
}

// Stage 2: 필터 적용
Image apply_filter(Image img) {
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    std::cout << "필터 적용 완료: " << img.filename << "\n";
    // img.data에 필터 적용
    return img;
}

// Stage 3: 리사이즈
Image resize_image(Image img) {
    std::this_thread::sleep_for(std::chrono::milliseconds(75));
    std::cout << "리사이즈 완료: " << img.filename << "\n";
    img.width /= 2;
    img.height /= 2;
    img.data.resize(img.width * img.height * 3);
    return img;
}

// Stage 4: 디스크에 저장
void save_image(Image img) {
    std::this_thread::sleep_for(std::chrono::milliseconds(50));
    std::cout << "저장 완료: " << img.filename << "\n";
}

int main() {
    // Pipeline stage 생성
    Stage<std::string, Image> loader(load_image);
    Stage<Image, Image> filter(apply_filter);
    Stage<Image, Image> resizer(resize_image);
    Stage<Image, void> saver(save_image);

    // stage 연결
    loader.set_next(&filter);
    filter.set_next(&resizer);
    resizer.set_next(&saver);

    // stage 스레드 시작
    std::thread t1([&] { loader.run(); });
    std::thread t2([&] { filter.run(); });
    std::thread t3([&] { resizer.run(); });
    std::thread t4([&] { saver.run(); });

    // 입력 투입
    std::vector<std::string> files = {
        "image1.jpg", "image2.jpg", "image3.jpg",
        "image4.jpg", "image5.jpg"
    };

    for (const auto& file : files) {
        loader.push(file);
    }

    // 처리 대기
    std::this_thread::sleep_for(std::chrono::seconds(3));

    // Pipeline 종료
    loader.stop();

    t1.join();
    t2.join();
    t3.join();
    t4.join();

    std::cout << "Pipeline 완료\n";
    return 0;
}
```

## 고급 구현: 제네릭 Pipeline 빌더

```cpp
template<typename T>
class Pipeline {
private:
    struct StageBase {
        virtual ~StageBase() = default;
        virtual void start() = 0;
        virtual void stop() = 0;
        virtual void join() = 0;
    };

    template<typename In, typename Out>
    struct StageImpl : StageBase {
        Stage<In, Out> stage;
        std::thread thread;

        StageImpl(std::function<Out(In)> func) : stage(func) {}

        void start() override {
            thread = std::thread([this] { stage.run(); });
        }

        void stop() override {
            stage.stop();
        }

        void join() override {
            if (thread.joinable()) {
                thread.join();
            }
        }
    };

    std::vector<std::unique_ptr<StageBase>> stages_;
    StageBase* first_stage_ = nullptr;

public:
    template<typename In, typename Out>
    Pipeline& add_stage(std::function<Out(In)> processor) {
        auto stage_impl = std::make_unique<StageImpl<In, Out>>(processor);

        if (!first_stage_) {
            first_stage_ = stage_impl.get();
        }

        // 이전 stage에 연결
        if (!stages_.empty()) {
            // 타입 안전한 연결을 위해서는 더 많은 템플릿 기법이 필요
            // 이것은 단순화된 버전
        }

        stages_.push_back(std::move(stage_impl));
        return *this;
    }

    void start() {
        for (auto& stage : stages_) {
            stage->start();
        }
    }

    void stop() {
        if (!stages_.empty()) {
            stages_[0]->stop();
        }
    }

    void join() {
        for (auto& stage : stages_) {
            stage->join();
        }
    }

    template<typename In>
    void push(In item) {
        // 첫 번째 stage에 투입
        // 캐스팅이 필요하며, 여기서는 단순화함
    }
};
```

## 병렬 Stage (Stage 내 Fan-Out)

```cpp
template<typename Input, typename Output>
class ParallelStage {
private:
    std::function<Output(Input)> processor_;
    std::queue<Input> input_queue_;
    size_t num_workers_;

    mutable std::mutex mutex_;
    std::condition_variable cv_;
    bool stopped_ = false;

    Stage<Output, void>* next_stage_ = nullptr;

public:
    ParallelStage(std::function<Output(Input)> processor, size_t num_workers)
        : processor_(std::move(processor)), num_workers_(num_workers) {}

    void set_next(Stage<Output, void>* next) {
        next_stage_ = next;
    }

    void push(Input item) {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            input_queue_.push(std::move(item));
        }
        cv_.notify_one();
    }

    void run_worker() {
        while (true) {
            Input item;

            {
                std::unique_lock<std::mutex> lock(mutex_);
                cv_.wait(lock, [this] {
                    return !input_queue_.empty() || stopped_;
                });

                if (stopped_ && input_queue_.empty()) {
                    break;
                }

                if (input_queue_.empty()) {
                    continue;
                }

                item = std::move(input_queue_.front());
                input_queue_.pop();
            }

            try {
                Output result = processor_(std::move(item));

                if (next_stage_) {
                    next_stage_->push(std::move(result));
                }
            } catch (const std::exception& e) {
                std::cerr << "워커 오류: " << e.what() << "\n";
            }
        }
    }

    std::vector<std::thread> start() {
        std::vector<std::thread> workers;
        for (size_t i = 0; i < num_workers_; ++i) {
            workers.emplace_back([this] { run_worker(); });
        }
        return workers;
    }

    void stop() {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            stopped_ = true;
        }
        cv_.notify_all();
    }
};
```

## Backpressure 처리

```cpp
template<typename Input, typename Output>
class BoundedStage {
private:
    std::function<Output(Input)> processor_;
    std::queue<Input> input_queue_;
    size_t max_queue_size_;

    mutable std::mutex mutex_;
    std::condition_variable input_cv_;
    std::condition_variable capacity_cv_;
    bool stopped_ = false;

    Stage<Output, void>* next_stage_ = nullptr;

public:
    BoundedStage(std::function<Output(Input)> processor, size_t max_queue_size)
        : processor_(std::move(processor)),
          max_queue_size_(max_queue_size) {}

    // backpressure가 적용되는 블로킹 push
    bool push(Input item) {
        std::unique_lock<std::mutex> lock(mutex_);

        // 용량이 확보될 때까지 대기
        capacity_cv_.wait(lock, [this] {
            return input_queue_.size() < max_queue_size_ || stopped_;
        });

        if (stopped_) {
            return false;
        }

        input_queue_.push(std::move(item));
        input_cv_.notify_one();
        return true;
    }

    void run() {
        while (true) {
            Input item;

            {
                std::unique_lock<std::mutex> lock(mutex_);
                input_cv_.wait(lock, [this] {
                    return !input_queue_.empty() || stopped_;
                });

                if (stopped_ && input_queue_.empty()) {
                    if (next_stage_) {
                        next_stage_->stop();
                    }
                    break;
                }

                if (input_queue_.empty()) {
                    continue;
                }

                item = std::move(input_queue_.front());
                input_queue_.pop();
            }

            // 용량이 확보되었음을 알림
            capacity_cv_.notify_one();

            try {
                Output result = processor_(std::move(item));

                if (next_stage_) {
                    next_stage_->push(std::move(result));
                }
            } catch (const std::exception& e) {
                std::cerr << "Stage 오류: " << e.what() << "\n";
            }
        }
    }

    void stop() {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            stopped_ = true;
        }
        input_cv_.notify_all();
        capacity_cv_.notify_all();
    }
};
```

## 성능 모니터링

```cpp
template<typename Input, typename Output>
class MonitoredStage : public Stage<Input, Output> {
private:
    std::atomic<size_t> items_processed_{0};
    std::atomic<size_t> total_processing_time_ms_{0};
    std::chrono::steady_clock::time_point start_time_;

public:
    MonitoredStage(std::function<Output(Input)> processor)
        : Stage<Input, Output>(
            [this, processor](Input item) {
                auto start = std::chrono::steady_clock::now();

                Output result = processor(std::move(item));

                auto end = std::chrono::steady_clock::now();
                auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
                    end - start).count();

                items_processed_.fetch_add(1);
                total_processing_time_ms_.fetch_add(ms);

                return result;
            }
        ),
        start_time_(std::chrono::steady_clock::now()) {}

    void print_stats() const {
        auto now = std::chrono::steady_clock::now();
        auto elapsed_sec = std::chrono::duration_cast<std::chrono::seconds>(
            now - start_time_).count();

        size_t processed = items_processed_.load();
        size_t total_time = total_processing_time_ms_.load();

        std::cout << "Stage 통계:\n"
                  << "  처리된 항목 수: " << processed << "\n"
                  << "  처리량: " << (processed / std::max(1L, elapsed_sec))
                  << " items/sec\n"
                  << "  평균 처리 시간: "
                  << (processed > 0 ? total_time / processed : 0) << "ms\n"
                  << "  큐 크기: " << this->queue_size() << "\n";
    }
};
```

## 실제 응용 사례

### 1. 비디오 처리

```
┌──────┐    ┌────────┐    ┌──────────┐    ┌────────┐    ┌──────┐
│Decode│───▶│Denoise │───▶│Color Adj.│───▶│ Encode │───▶│ Save │
└──────┘    └────────┘    └──────────┘    └────────┘    └──────┘
  (GPU)      (CPU 4x)         (GPU)         (CPU 8x)     (Disk)
```

### 2. ETL (추출, 변환, 적재)

```
┌────────┐    ┌─────────┐    ┌──────────┐    ┌──────┐
│Extract │───▶│Transform│───▶│ Validate │───▶│ Load │
│  (DB)  │    │ (CPU)   │    │  (CPU)   │    │ (DB) │
└────────┘    └─────────┘    └──────────┘    └──────┘
```

### 3. 로그 처리

```
┌──────┐    ┌──────┐    ┌──────────┐    ┌───────────┐
│ Read │───▶│Parse │───▶│ Analyze  │───▶│Aggregate  │
│(Disk)│    │(CPU) │    │  (CPU)   │    │(Memory/DB)│
└──────┘    └──────┘    └──────────┘    └───────────┘
```

### 4. 웹 스크래핑

```
┌──────┐    ┌───────┐    ┌─────────┐    ┌───────┐
│Fetch │───▶│ Parse │───▶│ Extract │───▶│ Store │
│(HTTP)│    │(HTML) │    │  (Data) │    │  (DB) │
└──────┘    └───────┘    └─────────┘    └───────┘
```

## Pipeline 변형

### 1. 분기 Pipeline

```
         ┌──────────┐
    ┌───▶│  Stage B │
    │    └──────────┘
┌───┴──┐
│Stage A│
└───┬──┘
    │    ┌──────────┐
    └───▶│  Stage C │
         └──────────┘
```

### 2. 병합 Pipeline

```
┌──────────┐
│  Stage A │───┐
└──────────┘   │
               ├───▶┌──────────┐
┌──────────┐   │    │  Stage C │
│  Stage B │───┘    └──────────┘
└──────────┘
```

### 3. 순환 Pipeline (피드백)

```
┌──────────┐    ┌──────────┐
│  Stage A │───▶│  Stage B │
└──────────┘    └──────────┘
      ▲               │
      └───────────────┘
       (재시도/정제)
```

## 일반적인 함정

### 1. 불균형한 Stage

```cpp
// 나쁨: 느린 stage가 병목이 됨
Stage 1: 10ms  ───┐
Stage 2: 100ms ───┼── 처리량이 Stage 2에 의해 제한됨
Stage 3: 10ms  ───┘

// 좋음: 느린 stage를 병렬화
Stage 1: 10ms     ───┐
Stage 2: 100ms (4x) ─┼── 균형 잡힌 처리량
Stage 3: 10ms     ───┘
```

### 2. 크기 제한 없는 큐

```cpp
// 나쁨: 빠른 생산자, 느린 소비자
// 큐가 제한 없이 증가 → OOM

// 좋음: backpressure가 적용된 제한 큐
BoundedStage stage(processor, 100);  // 최대 100개 항목
```

### 3. 오류 처리 누락

```cpp
// 나쁨: 예외가 stage 스레드를 종료시킴
Output result = processor(item);  // 예외가 발생할 수 있음!

// 좋음: 오류를 포착하고 처리
try {
    Output result = processor(item);
} catch (const std::exception& e) {
    // 오류 로그 기록, 항목 건너뛰기, 또는 재시도
}
```

## 성능 고려사항

### 지연 시간 vs 처리량

```
지연 시간: 하나의 항목이 전체 pipeline을 통과하는 데 걸리는 시간
  = Stage1_time + Stage2_time + Stage3_time

처리량: 초당 처리되는 항목 수
  = 1 / max(Stage1_time, Stage2_time, Stage3_time)

예시:
  Stage 1: 100ms
  Stage 2: 200ms  (병목)
  Stage 3: 100ms

  지연 시간: 항목당 400ms
  처리량: 5 items/sec (Stage 2에 의해 제한됨)
```

### 버퍼 크기 조정

```cpp
// 너무 작음: 빈번한 블로킹 발생
const size_t BUFFER_SIZE = 1;  // stage가 자주 대기함

// 너무 큼: 메모리 낭비, 높은 지연 시간
const size_t BUFFER_SIZE = 10000;  // 많은 항목이 큐에 대기

// 최적: 메모리와 블로킹 간 균형
const size_t BUFFER_SIZE = 10-100;  // 대부분의 경우에 적합한 범위
```

## 테스트 전략

### 정확성 테스트

```cpp
// 모든 항목이 정확히 한 번 처리되었는지 확인
std::atomic<int> counter{0};
auto terminal = Stage<int, void>([&](int x) {
    counter.fetch_add(1);
});

// N개의 항목을 투입하고 counter == N인지 확인
```

### 스트레스 테스트

```cpp
// 대량 테스트
for (int i = 0; i < 1000000; ++i) {
    pipeline.push(i);
}
```

### 병목 식별

```cpp
// 큐 크기 모니터링
std::cout << "Stage 1 큐: " << stage1.queue_size() << "\n";
std::cout << "Stage 2 큐: " << stage2.queue_size() << "\n";
// 큐가 증가하면 하류에 병목이 있음을 나타냄
```

## 장단점

### 장점
- stage 간 자연스러운 병렬성
- 우수한 CPU 활용률
- 확장 가능 (stage 추가 또는 stage 병렬화)
- 명확한 관심사 분리
- 데이터 흐름에 대한 추론이 용이

### 단점
- stage 수가 증가할수록 지연 시간 증가
- 가장 느린 stage에 의해 처리량이 제한됨
- 큐 오버헤드 및 메모리 사용량
- stage 간 복잡한 오류 처리
- 디버깅이 어려울 수 있음

## 모범 사례

1. **stage 처리 시간의 균형을 맞추기**
2. **메모리 문제를 방지하기 위해 제한 큐 사용**
3. **각 stage의 성능 모니터링**
4. **오류를 우아하게 처리 (pipeline을 중단시키지 않기)**
5. **처리량 균형을 위해 느린 stage 병렬화**
6. **stage를 독립적으로 유지 (공유 상태 없음)**
7. **느린 소비자를 위한 backpressure 구현**

## 요약

Pipeline 패턴은 다음에 적합합니다:
- 스트림 처리
- 순차적 데이터 변환
- ETL 워크로드
- 미디어 처리

이 패턴은 모든 stage를 동시에 활성 상태로 유지하여 우수한 처리량을 제공하면서, 처리 단계 간 명확한 분리를 유지합니다.
