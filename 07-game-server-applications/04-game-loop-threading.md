# Game Loop and Multithreading

## Overview

The game loop is the heartbeat of any game server. It processes inputs, updates game state, and generates outputs at a fixed or variable rate. Designing an efficient, thread-safe game loop is crucial for performance and fairness. This document covers game loop patterns, multithreading strategies, frame pacing, and deterministic simulation.

## Table of Contents

1. [Game Loop Fundamentals](#game-loop-fundamentals)
2. [Single-threaded vs Multi-threaded](#single-threaded-vs-multi-threaded)
3. [Fixed Timestep vs Variable Timestep](#fixed-timestep-vs-variable-timestep)
4. [Threading Patterns](#threading-patterns)
5. [Deterministic Simulation](#deterministic-simulation)
6. [Performance Optimization](#performance-optimization)

## Game Loop Fundamentals

### Basic Game Loop Structure

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

### Basic Implementation

```cpp
// Simple single-threaded game loop
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

            // 1. Process input
            ProcessInput();

            // 2. Update game state
            Update(delta_time);

            // 3. Generate output
            Render();

            // 4. Measure frame time
            auto frame_end = std::chrono::high_resolution_clock::now();
            auto frame_time = std::chrono::duration<float>(
                frame_end - current_time
            ).count();

            // Optionally sleep to maintain target frame rate
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
        // Read network packets
        // Parse player commands
        // Queue inputs for processing
    }

    void Update(float delta_time) {
        // Update physics
        // Update AI
        // Process combat
        // Update game logic
    }

    void Render() {
        // Prepare network updates
        // Send state to clients
    }

private:
    bool is_running_;
};
```

## Single-threaded vs Multi-threaded

### Single-threaded Game Loop

**Advantages:**
- No synchronization overhead
- Deterministic execution order
- Simple to debug
- No race conditions

**Disadvantages:**
- Limited CPU utilization (1 core)
- Cannot scale to many players
- I/O blocks game logic

```cpp
// Single-threaded server
class SingleThreadedServer {
public:
    void Run() {
        while (running_) {
            // Everything happens sequentially on one thread

            // 1. Network I/O (blocking or non-blocking)
            PollNetworkEvents();

            // 2. Process inputs from players
            for (auto& input : input_queue_) {
                ProcessInput(input);
            }
            input_queue_.clear();

            // 3. Update game simulation
            UpdatePhysics(delta_time_);
            UpdateAI(delta_time_);
            UpdateGameLogic(delta_time_);

            // 4. Prepare and send updates
            for (auto& player : players_) {
                SendUpdate(player);
            }

            // 5. Wait for next frame
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

### Multi-threaded Game Loop

**Advantages:**
- Utilize multiple CPU cores
- Scale to more players
- Separate I/O from game logic
- Better performance

**Disadvantages:**
- Synchronization overhead
- More complex
- Potential race conditions
- Harder to debug

```cpp
// Multi-threaded server
class MultiThreadedServer {
public:
    void Start() {
        running_ = true;

        // I/O thread (network)
        io_thread_ = std::thread(&MultiThreadedServer::IOThread, this);

        // Game logic thread
        game_thread_ = std::thread(&MultiThreadedServer::GameThread, this);

        // Worker thread pool
        for (int i = 0; i < num_worker_threads_; ++i) {
            worker_threads_.emplace_back(
                &MultiThreadedServer::WorkerThread, this
            );
        }
    }

    void Stop() {
        running_ = false;
        // Join threads...
    }

private:
    void IOThread() {
        // Handles all network I/O
        while (running_) {
            // Non-blocking I/O
            PollNetworkEvents();

            // Push received data to input queue
            for (auto& packet : received_packets_) {
                input_queue_.push(packet);
            }
            received_packets_.clear();

            // Send outgoing packets
            SendPendingPackets();

            // Small sleep to prevent busy-waiting
            std::this_thread::sleep_for(std::chrono::microseconds(100));
        }
    }

    void GameThread() {
        // Main game simulation thread
        auto tick_interval = std::chrono::milliseconds(16); // ~60 FPS
        auto next_tick = std::chrono::steady_clock::now();

        while (running_) {
            auto tick_start = std::chrono::steady_clock::now();

            // 1. Process inputs from queue
            ProcessInputs();

            // 2. Update game state (deterministic)
            UpdateGameState();

            // 3. Prepare outputs
            PrepareNetworkUpdates();

            // 4. Wait for next tick
            next_tick += tick_interval;
            std::this_thread::sleep_until(next_tick);

            // Measure tick time
            auto tick_time = std::chrono::steady_clock::now() - tick_start;
            RecordTickTime(tick_time);
        }
    }

    void WorkerThread() {
        // Worker threads process parallel tasks
        while (running_) {
            // Wait for work
            std::unique_lock<std::mutex> lock(work_mutex_);
            work_cv_.wait(lock, [this]() {
                return !work_queue_.empty() || !running_;
            });

            if (!running_) break;

            if (!work_queue_.empty()) {
                auto task = std::move(work_queue_.front());
                work_queue_.pop();
                lock.unlock();

                // Execute task
                task();
            }
        }
    }

    void ProcessInputs() {
        // Pop all inputs from lock-free queue
        std::vector<Input> inputs;
        Input input;
        while (input_queue_.try_pop(input)) {
            inputs.push_back(input);
        }

        // Process inputs
        for (auto& input : inputs) {
            ProcessInput(input);
        }
    }

    void UpdateGameState() {
        // Update can be parallelized
        SubmitParallelTasks();

        // Update physics
        UpdatePhysics(delta_time_);

        // Update AI (can be parallel)
        UpdateAI(delta_time_);

        // Update game logic
        UpdateGameLogic(delta_time_);

        // Wait for all parallel tasks to complete
        WaitForTasks();
    }

private:
    std::atomic<bool> running_;

    // Threading
    std::thread io_thread_;
    std::thread game_thread_;
    std::vector<std::thread> worker_threads_;
    int num_worker_threads_ = 4;

    // Communication
    LockFreeQueue<Input> input_queue_;

    // Worker pool
    std::queue<std::function<void()>> work_queue_;
    std::mutex work_mutex_;
    std::condition_variable work_cv_;

    float delta_time_ = 1.0f / 60.0f;
};
```

## Fixed Timestep vs Variable Timestep

### Fixed Timestep

**Concept:** Update game at fixed intervals (e.g., 60 times per second).

**Advantages:**
- Deterministic simulation
- Consistent physics
- Easier to debug
- Network synchronization easier

**Disadvantages:**
- Can lag if update takes too long
- May need interpolation for rendering

```cpp
// Fixed timestep game loop
class FixedTimestepLoop {
public:
    void Run() {
        constexpr double dt = 1.0 / 60.0; // Fixed 60 Hz
        constexpr auto tick_duration = std::chrono::duration<double>(dt);

        auto current_time = std::chrono::high_resolution_clock::now();
        double accumulator = 0.0;

        while (running_) {
            auto new_time = std::chrono::high_resolution_clock::now();
            auto frame_time = std::chrono::duration<double>(new_time - current_time).count();
            current_time = new_time;

            // Prevent spiral of death (cap maximum frame time)
            if (frame_time > 0.25) {
                frame_time = 0.25; // Max 4 frames to catch up
            }

            accumulator += frame_time;

            // Process input
            ProcessInput();

            // Update simulation with fixed timestep
            while (accumulator >= dt) {
                Update(dt);
                tick_number_++;
                accumulator -= dt;
            }

            // Render with interpolation
            double alpha = accumulator / dt;
            Render(alpha);
        }
    }

private:
    void Update(double dt) {
        // Fixed timestep update
        UpdatePhysics(dt);
        UpdateGameLogic(dt);
    }

    void Render(double alpha) {
        // Interpolate between previous and current state
        // for smooth rendering
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

### Variable Timestep

**Concept:** Update based on actual elapsed time.

**Advantages:**
- Adapts to system load
- Smoother frame rate
- Simple implementation

**Disadvantages:**
- Non-deterministic
- Physics instability with large dt
- Harder to reproduce bugs

```cpp
// Variable timestep game loop
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

            // Cap delta time to prevent huge jumps
            if (delta_time > 0.1) {
                delta_time = 0.1; // Cap at 100ms
            }

            // Process input
            ProcessInput();

            // Update with variable timestep
            Update(delta_time);

            // Render
            Render();
        }
    }

private:
    void Update(double dt) {
        // All updates use actual elapsed time
        UpdatePhysics(dt);
        UpdateGameLogic(dt);
    }

private:
    bool running_ = true;
};
```

### Hybrid Approach (Semi-Fixed)

**Best of both worlds:** Fixed timestep for physics, variable for other systems.

```cpp
// Hybrid timestep game loop
class HybridTimestepLoop {
public:
    void Run() {
        constexpr double physics_dt = 1.0 / 60.0; // 60 Hz physics
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

            // Process input
            ProcessInput();

            // Update physics with fixed timestep
            while (physics_accumulator >= physics_dt) {
                UpdatePhysics(physics_dt);
                physics_accumulator -= physics_dt;
            }

            // Update game logic with variable timestep
            UpdateGameLogic(frame_time);
            UpdateAI(frame_time);

            // Render
            double alpha = physics_accumulator / physics_dt;
            Render(alpha);
        }
    }
};
```

## Threading Patterns

### Pattern 1: IO Thread + Game Thread

```
┌──────────────┐         ┌──────────────────┐
│  I/O Thread  │────────▶│   Game Thread    │
│              │  Queue  │                  │
│ • Recv       │         │ • Process Input  │
│ • Send       │◀────────│ • Update State   │
│ • Poll       │  Queue  │ • Prepare Output │
└──────────────┘         └──────────────────┘
```

**Implementation:**

```cpp
class IOAndGameThreadServer {
public:
    void Start() {
        io_thread_ = std::thread([this]() {
            while (running_) {
                // Receive from network
                auto packets = network_.Receive();
                for (auto& packet : packets) {
                    input_queue_.push(packet);
                }

                // Send outgoing
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
                // Process all pending inputs
                Packet input;
                while (input_queue_.try_pop(input)) {
                    ProcessInput(input);
                }

                // Update game
                Update(0.016f);

                // Prepare outputs
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

### Pattern 2: Thread Pool for Parallel Updates

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

**Implementation:**

```cpp
class ThreadPoolGameLoop {
public:
    ThreadPoolGameLoop(int num_workers = 4)
        : thread_pool_(num_workers) {}

    void Run() {
        auto tick_interval = std::chrono::milliseconds(16);
        auto next_tick = std::chrono::steady_clock::now();

        while (running_) {
            // Sequential: Process input
            ProcessInput();

            // Parallel: Update systems
            std::vector<std::future<void>> futures;

            // AI updates (parallel)
            for (auto& zone : zones_) {
                futures.push_back(
                    thread_pool_.Submit([&zone]() {
                        zone.UpdateAI(0.016f);
                    })
                );
            }

            // Physics updates (parallel)
            for (auto& physics_island : physics_islands_) {
                futures.push_back(
                    thread_pool_.Submit([&physics_island]() {
                        physics_island.Update(0.016f);
                    })
                );
            }

            // Wait for all parallel work
            for (auto& future : futures) {
                future.wait();
            }

            // Sequential: Finalize state
            FinalizeGameState();

            // Sequential: Prepare output
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

### Pattern 3: Job System

```cpp
// Job-based parallelism
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
        // Spin-wait (can help with work stealing)
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

// Usage
class GameLoopWithJobs {
public:
    GameLoopWithJobs() : job_system_(4) {}

    void Run() {
        while (running_) {
            ProcessInput();

            // Submit parallel jobs
            std::atomic<int> job_counter{0};
            std::vector<JobSystem::Job> jobs;

            // AI jobs
            for (auto& npc : npcs_) {
                jobs.push_back({
                    [&npc]() { npc.UpdateAI(0.016f); },
                    &job_counter
                });
            }

            // Physics jobs
            for (auto& entity : entities_) {
                jobs.push_back({
                    [&entity]() { entity.UpdatePhysics(0.016f); },
                    &job_counter
                });
            }

            job_system_.SubmitJobs(jobs, job_counter);

            // Wait for completion
            job_system_.WaitForCounter(job_counter);

            // Finalize and output
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

## Deterministic Simulation

### Why Determinism Matters

- **Reproducible bugs:** Same inputs = same outputs
- **Replay systems:** Store inputs, replay simulation
- **Rollback netcode:** Rewind and re-simulate
- **Lockstep multiplayer:** All clients run same simulation

### Achieving Determinism

```cpp
// Deterministic game loop
class DeterministicGameLoop {
public:
    void Run() {
        // Fixed timestep is crucial
        constexpr double dt = 1.0 / 30.0; // 30 Hz
        uint64_t tick = 0;

        while (running_) {
            // 1. Collect all inputs for this tick
            auto inputs = CollectInputs(tick);

            // 2. Sort inputs deterministically
            std::sort(inputs.begin(), inputs.end(),
                [](const Input& a, const Input& b) {
                    // Sort by player ID first, then timestamp
                    if (a.player_id != b.player_id) {
                        return a.player_id < b.player_id;
                    }
                    return a.timestamp < b.timestamp;
                });

            // 3. Process inputs in order
            for (const auto& input : inputs) {
                ProcessInput(input);
            }

            // 4. Update simulation (deterministic)
            UpdateDeterministic(dt);

            // 5. Save state snapshot (for rollback/replay)
            SaveSnapshot(tick);

            tick++;

            // 6. Wait for next tick
            WaitForNextTick();
        }
    }

private:
    void UpdateDeterministic(double dt) {
        // Use deterministic math (no floating point inconsistencies)
        // - Fixed-point arithmetic, or
        // - Carefully controlled floating point

        // Update in consistent order
        for (auto& entity : entities_) {
            entity.Update(dt);
        }

        // Resolve collisions deterministically
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

        // Store in circular buffer
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

### Rollback and Replay

```cpp
// Rollback netcode (for fighting games, fast-paced games)
class RollbackGameLoop {
public:
    void Run() {
        constexpr double dt = 1.0 / 60.0;
        uint64_t current_tick = 0;

        while (running_) {
            // Get local input for current tick
            auto local_input = GetLocalInput();
            input_buffer_[current_tick % input_buffer_size_] = local_input;

            // Send to other players
            SendInput(local_input, current_tick);

            // Receive remote inputs
            auto remote_inputs = ReceiveRemoteInputs();

            // Check if we need to rollback
            uint64_t oldest_new_input = FindOldestNewInput(remote_inputs);

            if (oldest_new_input < current_tick) {
                // Rollback required
                Rollback(oldest_new_input);

                // Re-simulate from rollback point to current
                for (uint64_t tick = oldest_new_input; tick <= current_tick; ++tick) {
                    auto all_inputs = GetAllInputs(tick);
                    UpdateDeterministic(all_inputs, dt);
                    SaveState(tick);
                }
            } else {
                // No rollback needed, just simulate current tick
                auto all_inputs = GetAllInputs(current_tick);
                UpdateDeterministic(all_inputs, dt);
                SaveState(current_tick);
            }

            // Render (with prediction)
            Render();

            current_tick++;
            WaitForNextTick();
        }
    }

private:
    void Rollback(uint64_t target_tick) {
        // Restore state from snapshot
        game_state_ = snapshots_[target_tick % snapshot_buffer_size_];
    }

    void UpdateDeterministic(const std::vector<Input>& inputs, double dt) {
        // Sort inputs
        auto sorted_inputs = inputs;
        std::sort(sorted_inputs.begin(), sorted_inputs.end(),
            [](const Input& a, const Input& b) {
                return a.player_id < b.player_id;
            });

        // Process inputs
        for (const auto& input : sorted_inputs) {
            ApplyInput(input);
        }

        // Update physics
        physics_.Update(dt);

        // Update game logic
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

## Performance Optimization

### 1. Tick Budget Management

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

        // Record metrics
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

// Usage
void GameLoop() {
    TickBudgetManager budget(16.0); // 16ms for 60 Hz

    while (running_) {
        budget.BeginTick();

        ProcessInput();
        UpdatePhysics(0.016);

        // Only update AI if we have budget
        if (budget.HasBudget()) {
            UpdateAI(0.016);
        }

        UpdateGameLogic(0.016);

        budget.EndTick();

        WaitForNextTick();
    }
}
```

### 2. Adaptive Tick Rate

```cpp
// Dynamically adjust tick rate based on load
class AdaptiveTickRateLoop {
public:
    void Run() {
        double current_tick_rate = 30.0; // Start at 30 Hz
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

            // Adjust tick rate based on actual performance
            if (tick_time > target_tick_time * 1.2) {
                // Server is overloaded - reduce tick rate
                current_tick_rate = std::max(min_tick_rate, current_tick_rate * 0.9);
            } else if (tick_time < target_tick_time * 0.8) {
                // Server has headroom - increase tick rate
                current_tick_rate = std::min(max_tick_rate, current_tick_rate * 1.1);
            }

            // Wait for next tick
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

## Conclusion

Game loop design is fundamental to server performance and correctness. Fixed timestep loops provide determinism crucial for fairness and reproducibility. Multi-threading can dramatically improve performance but requires careful synchronization. The choice of threading pattern depends on game requirements: real-time competitive games benefit from deterministic single-threaded or carefully synchronized loops, while MMOs can leverage parallelism more aggressively.

## Further Reading

- [Fix Your Timestep!](https://gafferongames.com/post/fix_your_timestep/)
- [Game Programming Patterns - Game Loop](https://gameprogrammingpatterns.com/game-loop.html)
- [Doom 3 Source Code Review](https://fabiensanglard.net/doom3/)
- [Rollback Networking in GGPO](https://www.ggpo.net/)
- [Job System Design](https://blog.molecular-matters.com/2015/08/24/job-system-2-0-lock-free-work-stealing-part-1-basics/)
