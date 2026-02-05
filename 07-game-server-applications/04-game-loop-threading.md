# Game Loop와 멀티스레딩

## 개요

Game loop는 모든 게임 서버의 심장박동입니다. 고정 또는 가변 속도로 입력을 처리하고, 게임 상태를 업데이트하며, 출력을 생성합니다. 효율적이고 스레드 안전한 game loop를 설계하는 것은 성능과 공정성에 필수적입니다. 이 문서에서는 game loop 패턴, 멀티스레딩 전략, 프레임 페이싱, 결정론적 시뮬레이션을 다룹니다.

## 목차

1. [Game Loop 기초](#game-loop-기초)
2. [단일 스레드 vs 멀티 스레드](#단일-스레드-vs-멀티-스레드)
3. [고정 타임스텝 vs 가변 타임스텝](#고정-타임스텝-vs-가변-타임스텝)
4. [스레딩 패턴](#스레딩-패턴)
5. [결정론적 시뮬레이션](#결정론적-시뮬레이션)
6. [성능 최적화](#성능-최적화)

## Game Loop 기초

### 기본 Game Loop 구조

```
Classic Game Loop
┌─────────────────────────────────────┐
│         Infinite Loop               │
│  ┌───────────────────────────────┐  │
│  │                               │  │
│  │  1. Process Input             │  │
│  │     • Network packets         │  │
│  │     • Player commands         │  │
│  │                               │  │
│  │  2. Update Game State         │  │
│  │     • Physics                 │  │
│  │     • AI                      │  │
│  │     • Game logic              │  │
│  │                               │  │
│  │  3. Generate Output           │  │
│  │     • Network updates         │  │
│  │     • State snapshots         │  │
│  │                               │  │
│  │  4. Frame Timing              │  │
│  │     • Sleep/wait              │  │
│  │     • Measure delta time      │  │
│  │                               │  │
│  └───────────────────────────────┘  │
│             │                        │
│             └────┐                   │
│                  │                   │
└──────────────────┴───────────────────┘
```

### 기본 구현

```cpp
// 간단한 단일 스레드 game loop
class SimpleGameLoop {
public:
    void Run() {
        is_running_ = true;

        auto last_time = std::chrono::high_resolution_clock::now();

        while (is_running_) {
            auto current_time = std::chrono::high_resolution_clock::now();
            auto delta_time = std::chrono::duration<float>(
                current_time - last_time
            ).count();
            last_time = current_time;

            // 1. 입력 처리
            ProcessInput();

            // 2. 게임 상태 업데이트
            Update(delta_time);

            // 3. 출력 생성
            Render();

            // 4. 프레임 시간 측정
            auto frame_end = std::chrono::high_resolution_clock::now();
            auto frame_time = std::chrono::duration<float>(
                frame_end - current_time
            ).count();

            // 목표 프레임 레이트 유지를 위해 선택적으로 sleep
            float target_frame_time = 1.0f / 60.0f; // 60 FPS
            if (frame_time < target_frame_time) {
                std::this_thread::sleep_for(
                    std::chrono::duration<float>(target_frame_time - frame_time)
                );
            }
        }
    }

private:
    void ProcessInput() {
        // 네트워크 패킷 읽기
        // 플레이어 명령 파싱
        // 처리를 위해 입력 큐에 넣기
    }

    void Update(float delta_time) {
        // 물리 업데이트
        // AI 업데이트
        // 전투 처리
        // 게임 로직 업데이트
    }

    void Render() {
        // 네트워크 업데이트 준비
        // 클라이언트에 상태 전송
    }

private:
    bool is_running_;
};
```

## 단일 스레드 vs 멀티 스레드

### 단일 스레드 Game Loop

**장점:**
- 동기화 오버헤드 없음
- 결정론적 실행 순서
- 디버깅 용이
- 경쟁 조건 없음

**단점:**
- 제한된 CPU 활용 (1 코어)
- 많은 플레이어로 확장 불가
- I/O가 게임 로직을 차단

```cpp
// 단일 스레드 서버
class SingleThreadedServer {
public:
    void Run() {
        while (running_) {
            // 모든 것이 하나의 스레드에서 순차적으로 실행

            // 1. 네트워크 I/O (블로킹 또는 논블로킹)
            PollNetworkEvents();

            // 2. 플레이어 입력 처리
            for (auto& input : input_queue_) {
                ProcessInput(input);
            }
            input_queue_.clear();

            // 3. 게임 시뮬레이션 업데이트
            UpdatePhysics(delta_time_);
            UpdateAI(delta_time_);
            UpdateGameLogic(delta_time_);

            // 4. 업데이트 준비 및 전송
            for (auto& player : players_) {
                SendUpdate(player);
            }

            // 5. 다음 프레임 대기
            WaitForNextFrame();
        }
    }

private:
    std::vector<Input> input_queue_;
    std::vector<Player> players_;
    float delta_time_ = 1.0f / 60.0f;
    bool running_ = true;
};
```

### 멀티 스레드 Game Loop

**장점:**
- 여러 CPU 코어 활용
- 더 많은 플레이어로 확장
- I/O와 게임 로직 분리
- 더 나은 성능

**단점:**
- 동기화 오버헤드
- 더 복잡함
- 잠재적 경쟁 조건
- 디버깅이 어려움

```cpp
// 멀티 스레드 서버
class MultiThreadedServer {
public:
    void Start() {
        running_ = true;

        // I/O 스레드 (네트워크)
        io_thread_ = std::thread(&MultiThreadedServer::IOThread, this);

        // 게임 로직 스레드
        game_thread_ = std::thread(&MultiThreadedServer::GameThread, this);

        // 워커 thread pool
        for (int i = 0; i < num_worker_threads_; ++i) {
            worker_threads_.emplace_back(
                &MultiThreadedServer::WorkerThread, this
            );
        }
    }

    void Stop() {
        running_ = false;
        // 스레드 join...
    }

private:
    void IOThread() {
        // 모든 네트워크 I/O 처리
        while (running_) {
            // 논블로킹 I/O
            PollNetworkEvents();

            // 수신 데이터를 입력 큐에 넣기
            for (auto& packet : received_packets_) {
                input_queue_.push(packet);
            }
            received_packets_.clear();

            // 송신 패킷 전송
            SendPendingPackets();

            // 바쁜 대기 방지를 위한 짧은 sleep
            std::this_thread::sleep_for(std::chrono::microseconds(100));
        }
    }

    void GameThread() {
        // 메인 게임 시뮬레이션 스레드
        auto tick_interval = std::chrono::milliseconds(16); // ~60 FPS
        auto next_tick = std::chrono::steady_clock::now();

        while (running_) {
            auto tick_start = std::chrono::steady_clock::now();

            // 1. 큐에서 입력 처리
            ProcessInputs();

            // 2. 게임 상태 업데이트 (결정론적)
            UpdateGameState();

            // 3. 출력 준비
            PrepareNetworkUpdates();

            // 4. 다음 tick 대기
            next_tick += tick_interval;
            std::this_thread::sleep_until(next_tick);

            // tick 시간 측정
            auto tick_time = std::chrono::steady_clock::now() - tick_start;
            RecordTickTime(tick_time);
        }
    }

    void WorkerThread() {
        // 워커 스레드는 병렬 작업 처리
        while (running_) {
            // 작업 대기
            std::unique_lock<std::mutex> lock(work_mutex_);
            work_cv_.wait(lock, [this]() {
                return !work_queue_.empty() || !running_;
            });

            if (!running_) break;

            if (!work_queue_.empty()) {
                auto task = std::move(work_queue_.front());
                work_queue_.pop();
                lock.unlock();

                // 작업 실행
                task();
            }
        }
    }

    void ProcessInputs() {
        // lock-free 큐에서 모든 입력 꺼내기
        std::vector<Input> inputs;
        Input input;
        while (input_queue_.try_pop(input)) {
            inputs.push_back(input);
        }

        // 입력 처리
        for (auto& input : inputs) {
            ProcessInput(input);
        }
    }

    void UpdateGameState() {
        // 업데이트는 병렬화 가능
        SubmitParallelTasks();

        // 물리 업데이트
        UpdatePhysics(delta_time_);

        // AI 업데이트 (병렬 가능)
        UpdateAI(delta_time_);

        // 게임 로직 업데이트
        UpdateGameLogic(delta_time_);

        // 모든 병렬 작업 완료 대기
        WaitForTasks();
    }

private:
    std::atomic<bool> running_;

    // 스레딩
    std::thread io_thread_;
    std::thread game_thread_;
    std::vector<std::thread> worker_threads_;
    int num_worker_threads_ = 4;

    // 통신
    LockFreeQueue<Input> input_queue_;

    // 워커 풀
    std::queue<std::function<void()>> work_queue_;
    std::mutex work_mutex_;
    std::condition_variable work_cv_;

    float delta_time_ = 1.0f / 60.0f;
};
```

## 고정 타임스텝 vs 가변 타임스텝

### 고정 타임스텝

**개념:** 고정 간격으로 게임을 업데이트합니다 (예: 초당 60회).

**장점:**
- 결정론적 시뮬레이션
- 일관된 물리
- 디버깅 용이
- 네트워크 동기화 용이

**단점:**
- 업데이트가 오래 걸리면 지연 발생 가능
- 렌더링을 위한 보간이 필요할 수 있음

```cpp
// 고정 타임스텝 game loop
class FixedTimestepLoop {
public:
    void Run() {
        constexpr double dt = 1.0 / 60.0; // 고정 60 Hz
        constexpr auto tick_duration = std::chrono::duration<double>(dt);

        auto current_time = std::chrono::high_resolution_clock::now();
        double accumulator = 0.0;

        while (running_) {
            auto new_time = std::chrono::high_resolution_clock::now();
            auto frame_time = std::chrono::duration<double>(new_time - current_time).count();
            current_time = new_time;

            // 죽음의 나선 방지 (최대 프레임 시간 제한)
            if (frame_time > 0.25) {
                frame_time = 0.25; // 최대 4 프레임 따라잡기
            }

            accumulator += frame_time;

            // 입력 처리
            ProcessInput();

            // 고정 타임스텝으로 시뮬레이션 업데이트
            while (accumulator >= dt) {
                Update(dt);
                tick_number_++;
                accumulator -= dt;
            }

            // 보간을 적용한 렌더링
            double alpha = accumulator / dt;
            Render(alpha);
        }
    }

private:
    void Update(double dt) {
        // 고정 타임스텝 업데이트
        UpdatePhysics(dt);
        UpdateGameLogic(dt);
    }

    void Render(double alpha) {
        // 부드러운 렌더링을 위해
        // 이전 상태와 현재 상태를 보간
        for (auto& entity : entities_) {
            entity.interpolated_position =
                entity.previous_position * (1.0 - alpha) +
                entity.current_position * alpha;
        }

        SendStateUpdates();
    }

private:
    bool running_ = true;
    uint64_t tick_number_ = 0;
    std::vector<Entity> entities_;
};
```

### 가변 타임스텝

**개념:** 실제 경과 시간을 기반으로 업데이트합니다.

**장점:**
- 시스템 부하에 적응
- 더 부드러운 프레임 레이트
- 간단한 구현

**단점:**
- 비결정론적
- 큰 dt에서 물리 불안정
- 버그 재현이 어려움

```cpp
// 가변 타임스텝 game loop
class VariableTimestepLoop {
public:
    void Run() {
        auto last_time = std::chrono::high_resolution_clock::now();

        while (running_) {
            auto current_time = std::chrono::high_resolution_clock::now();
            double delta_time = std::chrono::duration<double>(
                current_time - last_time
            ).count();
            last_time = current_time;

            // 큰 점프 방지를 위해 delta time 제한
            if (delta_time > 0.1) {
                delta_time = 0.1; // 100ms로 제한
            }

            // 입력 처리
            ProcessInput();

            // 가변 타임스텝으로 업데이트
            Update(delta_time);

            // 렌더링
            Render();
        }
    }

private:
    void Update(double dt) {
        // 모든 업데이트가 실제 경과 시간 사용
        UpdatePhysics(dt);
        UpdateGameLogic(dt);
    }

private:
    bool running_ = true;
};
```

### 하이브리드 접근 방식 (반고정)

**양쪽의 장점:** 물리는 고정 타임스텝, 다른 시스템은 가변 타임스텝을 사용합니다.

```cpp
// 하이브리드 타임스텝 game loop
class HybridTimestepLoop {
public:
    void Run() {
        constexpr double physics_dt = 1.0 / 60.0; // 60 Hz 물리
        double physics_accumulator = 0.0;

        auto last_time = std::chrono::high_resolution_clock::now();

        while (running_) {
            auto current_time = std::chrono::high_resolution_clock::now();
            double frame_time = std::chrono::duration<double>(
                current_time - last_time
            ).count();
            last_time = current_time;

            if (frame_time > 0.25) frame_time = 0.25;

            physics_accumulator += frame_time;

            // 입력 처리
            ProcessInput();

            // 고정 타임스텝으로 물리 업데이트
            while (physics_accumulator >= physics_dt) {
                UpdatePhysics(physics_dt);
                physics_accumulator -= physics_dt;
            }

            // 가변 타임스텝으로 게임 로직 업데이트
            UpdateGameLogic(frame_time);
            UpdateAI(frame_time);

            // 렌더링
            double alpha = physics_accumulator / physics_dt;
            Render(alpha);
        }
    }
};
```

## 스레딩 패턴

### 패턴 1: I/O 스레드 + 게임 스레드

```
┌──────────────┐         ┌──────────────────┐
│  I/O Thread  │────────▶│   Game Thread    │
│              │  Queue  │                  │
│ • Recv       │         │ • Process Input  │
│ • Send       │◀────────│ • Update State   │
│ • Poll       │  Queue  │ • Prepare Output │
└──────────────┘         └──────────────────┘
```

**구현:**

```cpp
class IOAndGameThreadServer {
public:
    void Start() {
        io_thread_ = std::thread([this]() {
            while (running_) {
                // 네트워크에서 수신
                auto packets = network_.Receive();
                for (auto& packet : packets) {
                    input_queue_.push(packet);
                }

                // 송신
                Packet output;
                while (output_queue_.try_pop(output)) {
                    network_.Send(output);
                }

                std::this_thread::sleep_for(std::chrono::microseconds(100));
            }
        });

        game_thread_ = std::thread([this]() {
            auto tick_interval = std::chrono::milliseconds(16);
            auto next_tick = std::chrono::steady_clock::now();

            while (running_) {
                // 대기 중인 모든 입력 처리
                Packet input;
                while (input_queue_.try_pop(input)) {
                    ProcessInput(input);
                }

                // 게임 업데이트
                Update(0.016f);

                // 출력 준비
                for (auto& update : PrepareUpdates()) {
                    output_queue_.push(update);
                }

                next_tick += tick_interval;
                std::this_thread::sleep_until(next_tick);
            }
        });
    }

private:
    std::thread io_thread_;
    std::thread game_thread_;
    LockFreeQueue<Packet> input_queue_;
    LockFreeQueue<Packet> output_queue_;
    std::atomic<bool> running_{true};
};
```

### 패턴 2: 병렬 업데이트를 위한 Thread Pool

```
                  ┌──────────────────┐
                  │   Game Thread    │
                  │   (Main Loop)    │
                  └────────┬─────────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
    ┌──────────┐     ┌──────────┐    ┌──────────┐
    │ Worker 1 │     │ Worker 2 │    │ Worker N │
    │  (AI)    │     │ (Physics)│    │ (Logic)  │
    └──────────┘     └──────────┘    └──────────┘
          │                │                │
          └────────────────┼────────────────┘
                           │
                           ▼
                    Wait for completion
```

**구현:**

```cpp
class ThreadPoolGameLoop {
public:
    ThreadPoolGameLoop(int num_workers = 4)
        : thread_pool_(num_workers) {}

    void Run() {
        auto tick_interval = std::chrono::milliseconds(16);
        auto next_tick = std::chrono::steady_clock::now();

        while (running_) {
            // 순차: 입력 처리
            ProcessInput();

            // 병렬: 시스템 업데이트
            std::vector<std::future<void>> futures;

            // AI 업데이트 (병렬)
            for (auto& zone : zones_) {
                futures.push_back(
                    thread_pool_.Submit([&zone]() {
                        zone.UpdateAI(0.016f);
                    })
                );
            }

            // 물리 업데이트 (병렬)
            for (auto& physics_island : physics_islands_) {
                futures.push_back(
                    thread_pool_.Submit([&physics_island]() {
                        physics_island.Update(0.016f);
                    })
                );
            }

            // 모든 병렬 작업 대기
            for (auto& future : futures) {
                future.wait();
            }

            // 순차: 상태 확정
            FinalizeGameState();

            // 순차: 출력 준비
            PrepareNetworkUpdates();

            next_tick += tick_interval;
            std::this_thread::sleep_until(next_tick);
        }
    }

private:
    ThreadPool thread_pool_;
    std::atomic<bool> running_{true};
    std::vector<Zone> zones_;
    std::vector<PhysicsIsland> physics_islands_;
};
```

### 패턴 3: Job 시스템

```cpp
// Job 기반 병렬 처리
class JobSystem {
public:
    struct Job {
        std::function<void()> work;
        std::atomic<int>* counter;
    };

    JobSystem(int num_threads) {
        for (int i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this]() {
                WorkerThread();
            });
        }
    }

    ~JobSystem() {
        running_ = false;
        cv_.notify_all();
        for (auto& worker : workers_) {
            if (worker.joinable()) worker.join();
        }
    }

    void SubmitJob(const Job& job) {
        {
            std::lock_guard<std::mutex> lock(queue_mutex_);
            job_queue_.push(job);
        }
        cv_.notify_one();
    }

    void SubmitJobs(const std::vector<Job>& jobs, std::atomic<int>& counter) {
        counter = jobs.size();

        {
            std::lock_guard<std::mutex> lock(queue_mutex_);
            for (auto& job : jobs) {
                job_queue_.push(job);
            }
        }

        cv_.notify_all();
    }

    void WaitForCounter(std::atomic<int>& counter) {
        // 스핀 대기 (작업 스틸링에 도움이 될 수 있음)
        while (counter.load() > 0) {
            std::this_thread::yield();
        }
    }

private:
    void WorkerThread() {
        while (running_) {
            Job job;

            {
                std::unique_lock<std::mutex> lock(queue_mutex_);
                cv_.wait(lock, [this]() {
                    return !job_queue_.empty() || !running_;
                });

                if (!running_) return;

                if (!job_queue_.empty()) {
                    job = job_queue_.front();
                    job_queue_.pop();
                }
            }

            if (job.work) {
                job.work();

                if (job.counter) {
                    job.counter->fetch_sub(1);
                }
            }
        }
    }

private:
    std::vector<std::thread> workers_;
    std::queue<Job> job_queue_;
    std::mutex queue_mutex_;
    std::condition_variable cv_;
    std::atomic<bool> running_{true};
};

// 사용 예시
class GameLoopWithJobs {
public:
    GameLoopWithJobs() : job_system_(4) {}

    void Run() {
        while (running_) {
            ProcessInput();

            // 병렬 job 제출
            std::atomic<int> job_counter{0};
            std::vector<JobSystem::Job> jobs;

            // AI job
            for (auto& npc : npcs_) {
                jobs.push_back({
                    [&npc]() { npc.UpdateAI(0.016f); },
                    &job_counter
                });
            }

            // 물리 job
            for (auto& entity : entities_) {
                jobs.push_back({
                    [&entity]() { entity.UpdatePhysics(0.016f); },
                    &job_counter
                });
            }

            job_system_.SubmitJobs(jobs, job_counter);

            // 완료 대기
            job_system_.WaitForCounter(job_counter);

            // 확정 및 출력
            FinalizeState();
            SendUpdates();

            WaitForNextFrame();
        }
    }

private:
    JobSystem job_system_;
    std::atomic<bool> running_{true};
    std::vector<NPC> npcs_;
    std::vector<Entity> entities_;
};
```

## 결정론적 시뮬레이션

### 결정론이 중요한 이유

- **재현 가능한 버그:** 같은 입력 = 같은 출력
- **리플레이 시스템:** 입력을 저장하고 시뮬레이션을 재생
- **롤백 넷코드:** 되감기 및 재시뮬레이션
- **Lockstep 멀티플레이어:** 모든 클라이언트가 같은 시뮬레이션 실행

### 결정론 달성

```cpp
// 결정론적 game loop
class DeterministicGameLoop {
public:
    void Run() {
        // 고정 타임스텝이 필수적
        constexpr double dt = 1.0 / 30.0; // 30 Hz
        uint64_t tick = 0;

        while (running_) {
            // 1. 이 tick의 모든 입력 수집
            auto inputs = CollectInputs(tick);

            // 2. 결정론적으로 입력 정렬
            std::sort(inputs.begin(), inputs.end(),
                [](const Input& a, const Input& b) {
                    // 플레이어 ID로 먼저 정렬, 그 다음 타임스탬프
                    if (a.player_id != b.player_id) {
                        return a.player_id < b.player_id;
                    }
                    return a.timestamp < b.timestamp;
                });

            // 3. 순서대로 입력 처리
            for (const auto& input : inputs) {
                ProcessInput(input);
            }

            // 4. 시뮬레이션 업데이트 (결정론적)
            UpdateDeterministic(dt);

            // 5. 상태 스냅샷 저장 (롤백/리플레이용)
            SaveSnapshot(tick);

            tick++;

            // 6. 다음 tick 대기
            WaitForNextTick();
        }
    }

private:
    void UpdateDeterministic(double dt) {
        // 결정론적 수학 사용 (부동소수점 불일치 방지)
        // - 고정소수점 연산, 또는
        // - 주의 깊게 제어된 부동소수점

        // 일관된 순서로 업데이트
        for (auto& entity : entities_) {
            entity.Update(dt);
        }

        // 결정론적으로 충돌 해결
        auto collisions = physics_.DetectCollisions();
        std::sort(collisions.begin(), collisions.end(),
            [](const Collision& a, const Collision& b) {
                return a.entity1_id < b.entity1_id;
            });

        for (auto& collision : collisions) {
            physics_.ResolveCollision(collision);
        }
    }

    void SaveSnapshot(uint64_t tick) {
        Snapshot snapshot;
        snapshot.tick = tick;

        for (const auto& entity : entities_) {
            snapshot.entity_states.push_back(entity.GetState());
        }

        // 순환 버퍼에 저장
        snapshots_[tick % snapshot_buffer_size_] = snapshot;
    }

private:
    std::vector<Entity> entities_;
    Physics physics_;
    std::array<Snapshot, 256> snapshots_;
    static constexpr size_t snapshot_buffer_size_ = 256;
    bool running_ = true;
};
```

### 롤백과 리플레이

```cpp
// 롤백 넷코드 (격투 게임, 빠른 페이스 게임용)
class RollbackGameLoop {
public:
    void Run() {
        constexpr double dt = 1.0 / 60.0;
        uint64_t current_tick = 0;

        while (running_) {
            // 현재 tick의 로컬 입력 가져오기
            auto local_input = GetLocalInput();
            input_buffer_[current_tick % input_buffer_size_] = local_input;

            // 다른 플레이어에게 전송
            SendInput(local_input, current_tick);

            // 원격 입력 수신
            auto remote_inputs = ReceiveRemoteInputs();

            // 롤백이 필요한지 확인
            uint64_t oldest_new_input = FindOldestNewInput(remote_inputs);

            if (oldest_new_input < current_tick) {
                // 롤백 필요
                Rollback(oldest_new_input);

                // 롤백 지점부터 현재까지 재시뮬레이션
                for (uint64_t tick = oldest_new_input; tick <= current_tick; ++tick) {
                    auto all_inputs = GetAllInputs(tick);
                    UpdateDeterministic(all_inputs, dt);
                    SaveState(tick);
                }
            } else {
                // 롤백 불필요, 현재 tick만 시뮬레이션
                auto all_inputs = GetAllInputs(current_tick);
                UpdateDeterministic(all_inputs, dt);
                SaveState(current_tick);
            }

            // 렌더링 (예측 포함)
            Render();

            current_tick++;
            WaitForNextTick();
        }
    }

private:
    void Rollback(uint64_t target_tick) {
        // 스냅샷에서 상태 복원
        game_state_ = snapshots_[target_tick % snapshot_buffer_size_];
    }

    void UpdateDeterministic(const std::vector<Input>& inputs, double dt) {
        // 입력 정렬
        auto sorted_inputs = inputs;
        std::sort(sorted_inputs.begin(), sorted_inputs.end(),
            [](const Input& a, const Input& b) {
                return a.player_id < b.player_id;
            });

        // 입력 처리
        for (const auto& input : sorted_inputs) {
            ApplyInput(input);
        }

        // 물리 업데이트
        physics_.Update(dt);

        // 게임 로직 업데이트
        UpdateGameLogic(dt);
    }

    void SaveState(uint64_t tick) {
        snapshots_[tick % snapshot_buffer_size_] = game_state_;
    }

private:
    GameState game_state_;
    Physics physics_;

    std::array<GameState, 256> snapshots_;
    std::array<Input, 256> input_buffer_;
    static constexpr size_t snapshot_buffer_size_ = 256;
    static constexpr size_t input_buffer_size_ = 256;

    bool running_ = true;
};
```

## 성능 최적화

### 1. Tick 예산 관리

```cpp
class TickBudgetManager {
public:
    TickBudgetManager(double tick_duration_ms)
        : tick_budget_(std::chrono::duration<double, std::milli>(tick_duration_ms)) {}

    void BeginTick() {
        tick_start_ = std::chrono::high_resolution_clock::now();
    }

    bool HasBudget() const {
        auto elapsed = std::chrono::high_resolution_clock::now() - tick_start_;
        return elapsed < tick_budget_;
    }

    double GetRemainingBudget() const {
        auto elapsed = std::chrono::high_resolution_clock::now() - tick_start_;
        auto remaining = tick_budget_ - elapsed;
        return std::chrono::duration<double, std::milli>(remaining).count();
    }

    void EndTick() {
        auto tick_time = std::chrono::high_resolution_clock::now() - tick_start_;

        // 지표 기록
        RecordTickTime(tick_time);

        if (tick_time > tick_budget_) {
            tick_overruns_++;
            LogWarning("Tick overrun: " +
                std::to_string(std::chrono::duration<double, std::milli>(tick_time).count()) +
                "ms");
        }
    }

private:
    std::chrono::high_resolution_clock::time_point tick_start_;
    std::chrono::duration<double> tick_budget_;
    size_t tick_overruns_ = 0;
};

// 사용 예시
void GameLoop() {
    TickBudgetManager budget(16.0); // 60 Hz를 위한 16ms

    while (running_) {
        budget.BeginTick();

        ProcessInput();
        UpdatePhysics(0.016);

        // 예산이 남은 경우에만 AI 업데이트
        if (budget.HasBudget()) {
            UpdateAI(0.016);
        }

        UpdateGameLogic(0.016);

        budget.EndTick();

        WaitForNextTick();
    }
}
```

### 2. 적응형 Tick Rate

```cpp
// 부하에 따라 tick rate를 동적으로 조정
class AdaptiveTickRateLoop {
public:
    void Run() {
        double current_tick_rate = 30.0; // 30 Hz로 시작
        double min_tick_rate = 10.0;
        double max_tick_rate = 60.0;

        while (running_) {
            auto tick_start = std::chrono::high_resolution_clock::now();

            double dt = 1.0 / current_tick_rate;

            ProcessInput();
            Update(dt);
            SendUpdates();

            auto tick_end = std::chrono::high_resolution_clock::now();
            auto tick_time = std::chrono::duration<double>(tick_end - tick_start).count();

            double target_tick_time = 1.0 / current_tick_rate;

            // 실제 성능에 따라 tick rate 조정
            if (tick_time > target_tick_time * 1.2) {
                // 서버 과부하 - tick rate 감소
                current_tick_rate = std::max(min_tick_rate, current_tick_rate * 0.9);
            } else if (tick_time < target_tick_time * 0.8) {
                // 서버 여유 있음 - tick rate 증가
                current_tick_rate = std::min(max_tick_rate, current_tick_rate * 1.1);
            }

            // 다음 tick 대기
            double sleep_time = target_tick_time - tick_time;
            if (sleep_time > 0) {
                std::this_thread::sleep_for(
                    std::chrono::duration<double>(sleep_time)
                );
            }
        }
    }

private:
    bool running_ = true;
};
```

## 결론

Game loop 설계는 서버 성능과 정확성의 근본입니다. 고정 타임스텝 loop는 공정성과 재현성에 필수적인 결정론을 제공합니다. 멀티스레딩은 성능을 극적으로 향상시킬 수 있지만 신중한 동기화가 필요합니다. 스레딩 패턴의 선택은 게임 요구사항에 따라 달라집니다: 실시간 경쟁 게임은 결정론적 단일 스레드 또는 신중하게 동기화된 loop가 유리하고, MMO는 병렬성을 더 적극적으로 활용할 수 있습니다.

## 추가 참고 자료

- [Fix Your Timestep!](https://gafferongames.com/post/fix_your_timestep/)
- [Game Programming Patterns - Game Loop](https://gameprogrammingpatterns.com/game-loop.html)
- [Doom 3 Source Code Review](https://fabiensanglard.net/doom3/)
- [Rollback Networking in GGPO](https://www.ggpo.net/)
- [Job System Design](https://blog.molecular-matters.com/2015/08/24/job-system-2-0-lock-free-work-stealing-part-1-basics/)
