# 라이브락 (Livelock)

## 라이브락이란 무엇인가?

**라이브락(Livelock)**은 스레드가 차단되지 않지만(교착 상태와 달리) 서로에게 응답하여 상태를 지속적으로 변경하면서도 의미 있는 진전을 이루지 못하는 상황입니다. 스레드는 활성 상태를 유지하고 CPU 자원을 소비하지만 시스템 전체가 완료를 향해 전진하지 못합니다.

이는 좁은 복도에서 두 사람이 서로를 지나치려고 할 때 - 둘 다 동시에 같은 쪽으로 이동하고, 그 다음 둘 다 반대쪽으로 이동하며 영원히 반복하면서 실제로는 지나가지 못하는 것과 같습니다.

## 라이브락 vs 교착 상태

```
┌──────────────────┬───────────────────┬──────────────────┐
│   Characteristic │     Deadlock      │     Livelock     │
├──────────────────┼───────────────────┼──────────────────┤
│  Thread State    │     Blocked       │      Active      │
│  CPU Usage       │       None        │       High       │
│  Progress        │       None        │       None       │
│  Detection       │     Easier        │      Harder      │
│  Visibility      │   Threads stuck   │  Threads busy    │
│  Resource Usage  │   Held/Locked     │   Released/Retry │
└──────────────────┴───────────────────┴──────────────────┘
```

### 시각적 비교

**교착 상태:**
```
Thread 1: [BLOCKED] ━━━━━━━━━━━━━━━━━ (waiting forever)
Thread 2: [BLOCKED] ━━━━━━━━━━━━━━━━━ (waiting forever)

CPU: Idle
Progress: NONE
```

**라이브락:**
```
Thread 1: [ACTIVE] ──↺──↺──↺──↺──↺── (busy but no progress)
Thread 2: [ACTIVE] ──↺──↺──↺──↺──↺── (busy but no progress)

CPU: 100% busy
Progress: NONE
```

## 고전적 예제: 복도 문제

```
Person A ←─────────────────→ Person B
         Narrow Hallway

Step 1: A moves left, B moves left   (both still blocked)
Step 2: A moves right, B moves right (both still blocked)
Step 3: A moves left, B moves left   (both still blocked)
...repeats forever...
```

### 코드 구현

```cpp
#include <thread>
#include <iostream>
#include <chrono>

struct Person {
    bool trying_left;
    bool trying_right;
    int id;
};

Person person_a = {false, false, 1};
Person person_b = {false, false, 2};

void person_a_walk() {
    while (true) {
        if (person_b.trying_left) {
            std::cout << "Person A: B is on left, I'll go left too\n";
            person_a.trying_left = true;
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        } else if (person_b.trying_right) {
            std::cout << "Person A: B is on right, I'll go right too\n";
            person_a.trying_right = true;
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }

        // Reset and try again
        person_a.trying_left = false;
        person_a.trying_right = false;
    }
}

void person_b_walk() {
    while (true) {
        if (person_a.trying_left) {
            std::cout << "Person B: A is on left, I'll go left too\n";
            person_b.trying_left = true;
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        } else if (person_a.trying_right) {
            std::cout << "Person B: A is on right, I'll go right too\n";
            person_b.trying_right = true;
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }

        // Reset and try again
        person_b.trying_left = false;
        person_b.trying_right = false;
    }
}

// This creates LIVELOCK - both keep moving but never pass!
```

## 일반적인 라이브락 패턴

### 패턴 1: 충돌 회피

스레드가 충돌을 감지하고 백오프하지만 동기화된 방식으로 수행할 때 발생합니다.

```cpp
#include <thread>
#include <mutex>
#include <iostream>

std::mutex resource_a;
std::mutex resource_b;

void thread_with_livelock(int id) {
    while (true) {
        // Try to acquire both resources
        resource_a.lock();

        if (!resource_b.try_lock()) {
            // Failed to get B, release A and retry
            std::cout << "Thread " << id << ": Failed to get B, releasing A\n";
            resource_a.unlock();

            // PROBLEM: Both threads do this simultaneously!
            // They keep releasing and retrying forever
            continue;
        }

        // Critical section
        std::cout << "Thread " << id << ": Got both resources!\n";
        resource_b.unlock();
        resource_a.unlock();
        break;
    }
}

// LIVELOCK: If both threads retry at same time, they collide repeatedly
```

**타임라인:**
```
Time    Thread 1                Thread 2
----    --------                --------
  1     Lock A                  Lock A (wait)
  2     Try B (fail)            -
  3     Unlock A                Lock A (acquired)
  4     -                       Try B (fail)
  5     Lock A (wait)           Unlock A
  6     Lock A (acquired)       Lock A (wait)
  7     Try B (fail)            -
  8     ...repeats...           ...repeats...
```

### 패턴 2: 정중한 스레드

스레드가 "정중하게" 다른 스레드에게 양보하려고 하지만 모두 동시에 수행합니다.

```cpp
#include <thread>
#include <atomic>

std::atomic<bool> thread1_wants{false};
std::atomic<bool> thread2_wants{false};

void polite_thread1() {
    while (true) {
        thread1_wants = true;

        // Be polite: if other thread wants it, yield
        while (thread2_wants.load()) {
            thread1_wants = false;  // Give way
            std::this_thread::yield();  // Let other thread go
            thread1_wants = true;   // Want it again
        }

        // Critical section
        critical_section();

        thread1_wants = false;
    }
}

void polite_thread2() {
    while (true) {
        thread2_wants = true;

        // Be polite: if other thread wants it, yield
        while (thread1_wants.load()) {
            thread2_wants = false;  // Give way
            std::this_thread::yield();  // Let other thread go
            thread2_wants = true;   // Want it again
        }

        // Critical section
        critical_section();

        thread2_wants = false;
    }
}

// LIVELOCK: Both keep yielding to each other!
```

### 패턴 3: 메시지 재전송

분산 시스템에서 노드가 충돌 시 재전송하지만 더 많은 충돌을 생성합니다.

```cpp
#include <iostream>
#include <random>
#include <thread>
#include <chrono>

struct Node {
    int id;
    int attempts;
};

bool try_send(Node& node) {
    // Simulate collision detection
    static std::random_device rd;
    static std::mt19937 gen(rd());
    static std::uniform_int_distribution<> dis(0, 1);
    bool collision = (dis(gen) == 0);

    if (collision) {
        std::cout << "Node " << node.id << ": Collision detected, retry attempt "
                  << node.attempts << "\n";
        node.attempts++;
        return false;
    }

    std::cout << "Node " << node.id << ": Sent successfully!\n";
    return true;
}

void node_send_with_livelock(Node& node) {
    while (!try_send(node)) {
        // Fixed retry interval - causes synchronized retries
        std::this_thread::sleep_for(std::chrono::microseconds(1000));

        // LIVELOCK: All nodes retry at same time!
    }
}
```

## 해결책 및 예방

### 해결책 1: 랜덤 백오프

무작위성을 도입하여 동기화를 깹니다.

```cpp
#include <thread>
#include <mutex>
#include <random>
#include <chrono>
#include <iostream>

std::mutex resource_a;
std::mutex resource_b;

void thread_with_random_backoff(int id) {
    std::random_device rd;
    std::mt19937 gen(rd() + id);  // Different seed per thread
    std::uniform_int_distribution<> dis(0, 10000);

    while (true) {
        resource_a.lock();

        if (!resource_b.try_lock()) {
            resource_a.unlock();

            // Random backoff: 0-10ms
            int backoff = dis(gen);
            std::cout << "Thread " << id << ": Backing off " << backoff << "μs\n";
            std::this_thread::sleep_for(std::chrono::microseconds(backoff));
            continue;
        }

        // Critical section
        std::cout << "Thread " << id << ": Success!\n";
        resource_b.unlock();
        resource_a.unlock();
        break;
    }
}
```

### 해결책 2: 지수 백오프

각 재시도마다 백오프 시간을 증가시킵니다 (이더넷 CSMA/CD처럼).

```cpp
#include <thread>
#include <mutex>
#include <chrono>
#include <iostream>

constexpr int MAX_BACKOFF = 1000000;  // 1 second

void thread_with_exponential_backoff(int id) {
    int backoff = 1000;  // Start with 1ms

    while (true) {
        resource_a.lock();

        if (!resource_b.try_lock()) {
            resource_a.unlock();

            std::cout << "Thread " << id << ": Backing off " << backoff << "μs\n";
            std::this_thread::sleep_for(std::chrono::microseconds(backoff));

            // Exponential backoff
            backoff = (backoff * 2 < MAX_BACKOFF) ? backoff * 2 : MAX_BACKOFF;
            continue;
        }

        // Critical section
        std::cout << "Thread " << id << ": Success!\n";
        resource_b.unlock();
        resource_a.unlock();
        break;
    }
}
```

### 해결책 3: 우선순위 기반 해결

하나의 스레드에 더 높은 우선순위를 부여합니다.

```cpp
#include <thread>
#include <atomic>

struct ThreadInfo {
    int id;
    int priority;
};

std::atomic<bool> low_priority_wants{false};
std::atomic<bool> high_priority_wants{false};

void low_priority_thread() {
    while (true) {
        low_priority_wants = true;

        // Yield to high priority thread
        while (high_priority_wants.load()) {
            low_priority_wants = false;
            std::this_thread::yield();
            low_priority_wants = true;
        }

        // Critical section
        critical_section();
        low_priority_wants = false;
    }
}

void high_priority_thread() {
    while (true) {
        high_priority_wants = true;

        // Don't yield - take priority!
        // Critical section
        critical_section();
        high_priority_wants = false;
    }
}

// NO LIVELOCK: High priority always proceeds
```

### 해결책 4: 락 순서 지정

재시도를 피하기 위해 일관된 락 순서를 사용합니다.

```cpp
#include <mutex>

std::mutex resource_a;
std::mutex resource_b;

void thread_with_ordering() {
    // Always acquire in order: A then B
    resource_a.lock();
    resource_b.lock();

    // Critical section
    critical_section();

    resource_b.unlock();
    resource_a.unlock();
}

// NO LIVELOCK: No trylock, no retries needed
```

### 해결책 5: 무작위화를 사용한 타임아웃

타임아웃과 랜덤 재시도를 결합합니다.

```cpp
#include <mutex>
#include <chrono>
#include <random>
#include <thread>
#include <iostream>

// Note: use std::timed_mutex for timeout functionality
std::timed_mutex resource_a;
std::timed_mutex resource_b;

void thread_with_timeout(int id) {
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<> dis(0, 100000);

    while (true) {
        resource_a.lock();

        auto timeout = std::chrono::seconds(1);
        if (!resource_b.try_lock_for(timeout)) {
            resource_a.unlock();

            // Random backoff before retry
            std::this_thread::sleep_for(std::chrono::microseconds(dis(gen)));
            continue;
        }

        // Critical section
        std::cout << "Thread " << id << ": Success!\n";
        resource_b.unlock();
        resource_a.unlock();
        break;
    }
}
```

## 실제 사례

### 예제 1: 네트워크 충돌 (이더넷)

```c
// Simplified CSMA/CD (Carrier Sense Multiple Access with Collision Detection)
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define MAX_ATTEMPTS 16

typedef struct {
    int id;
    int collisions;
} NetworkNode;

void transmit_with_csma_cd(NetworkNode* node) {
    int attempt = 0;

    while (attempt < MAX_ATTEMPTS) {
        // Listen for carrier
        if (channel_busy()) {
            wait_until_idle();
        }

        // Transmit
        if (send_frame()) {
            printf("Node %d: Transmission successful\n", node->id);
            return;
        }

        // Collision detected
        node->collisions++;
        printf("Node %d: Collision #%d\n", node->id, node->collisions);

        // Binary exponential backoff
        int k = (attempt < 10) ? attempt : 10;
        int backoff_slots = rand() % (1 << k);  // 0 to 2^k - 1
        usleep(backoff_slots * 512);  // 512μs per slot

        attempt++;
    }

    printf("Node %d: Failed after %d attempts\n", node->id, MAX_ATTEMPTS);
}
```

### 예제 2: 데이터베이스 재시도 로직

```c
#include <pthread.h>
#include <stdbool.h>
#include <stdlib.h>
#include <time.h>

typedef struct {
    pthread_mutex_t mutex;
    int value;
} DBRecord;

DBRecord records[1000];

bool update_records_with_livelock(int id1, int id2, int delta) {
    int attempts = 0;

    while (attempts < 100) {
        pthread_mutex_lock(&records[id1].mutex);

        if (pthread_mutex_trylock(&records[id2].mutex) != 0) {
            // Deadlock avoidance causes livelock!
            pthread_mutex_unlock(&records[id1].mutex);
            attempts++;
            continue;  // Fixed retry = LIVELOCK
        }

        // Update both records
        records[id1].value -= delta;
        records[id2].value += delta;

        pthread_mutex_unlock(&records[id2].mutex);
        pthread_mutex_unlock(&records[id1].mutex);
        return true;
    }

    return false;  // Failed
}

bool update_records_fixed(int id1, int id2, int delta) {
    int attempts = 0;

    while (attempts < 100) {
        pthread_mutex_lock(&records[id1].mutex);

        if (pthread_mutex_trylock(&records[id2].mutex) != 0) {
            pthread_mutex_unlock(&records[id1].mutex);

            // Random backoff prevents livelock
            usleep(rand() % 10000);
            attempts++;
            continue;
        }

        // Update both records
        records[id1].value -= delta;
        records[id2].value += delta;

        pthread_mutex_unlock(&records[id2].mutex);
        pthread_mutex_unlock(&records[id1].mutex);
        return true;
    }

    return false;
}
```

### 예제 3: 분산 합의

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>

typedef struct {
    int id;
    int proposed_value;
    int seen_proposals;
} Node;

// Simplified consensus with livelock potential
void reach_consensus_bad(Node* nodes, int num_nodes) {
    bool consensus_reached = false;

    while (!consensus_reached) {
        // Each node proposes its value
        for (int i = 0; i < num_nodes; i++) {
            nodes[i].seen_proposals = 0;

            // Check what others proposed
            for (int j = 0; j < num_nodes; j++) {
                if (nodes[j].proposed_value == nodes[i].proposed_value) {
                    nodes[i].seen_proposals++;
                }
            }

            // If not majority, change proposal
            if (nodes[i].seen_proposals < num_nodes / 2) {
                // Pick random new value
                nodes[i].proposed_value = rand() % 100;
                printf("Node %d: Changing proposal\n", i);
            }
        }

        // Check for consensus
        int first_value = nodes[0].proposed_value;
        consensus_reached = true;
        for (int i = 1; i < num_nodes; i++) {
            if (nodes[i].proposed_value != first_value) {
                consensus_reached = false;
                break;
            }
        }
    }

    // LIVELOCK: Nodes keep changing proposals!
}

// Fixed version with leader election
void reach_consensus_good(Node* nodes, int num_nodes) {
    // Elect leader (e.g., lowest ID)
    int leader_id = 0;
    for (int i = 1; i < num_nodes; i++) {
        if (nodes[i].id < nodes[leader_id].id) {
            leader_id = i;
        }
    }

    // Everyone adopts leader's proposal
    int consensus_value = nodes[leader_id].proposed_value;
    for (int i = 0; i < num_nodes; i++) {
        nodes[i].proposed_value = consensus_value;
    }

    printf("Consensus reached: %d\n", consensus_value);
    // NO LIVELOCK: Single decision maker
}
```

## 탐지 전략

### 1. 진행 상황 모니터링

```cpp
#include <chrono>
#include <iostream>

struct ProgressMonitor {
    int work_completed;
    std::chrono::time_point<std::chrono::steady_clock> last_progress;
};

ProgressMonitor monitor = {0, std::chrono::steady_clock::now()};

void check_for_livelock() {
    auto now = std::chrono::steady_clock::now();
    static int last_work = 0;

    auto duration = std::chrono::duration_cast<std::chrono::seconds>(now - monitor.last_progress);

    if (monitor.work_completed == last_work && duration.count() > 5) {
        std::cout << "LIVELOCK suspected: No progress in 5 seconds\n";
        std::cout << "Threads active but not advancing\n";
    }

    last_work = monitor.work_completed;
    monitor.last_progress = now;
}
```

### 2. 재시도 카운터

```c
#define MAX_RETRIES 1000

int retry_count = 0;

void detect_excessive_retries() {
    retry_count++;

    if (retry_count > MAX_RETRIES) {
        printf("LIVELOCK suspected: %d retries!\n", retry_count);
        // Take corrective action
        abort();
    }
}
```

### 3. CPU 사용량 분석

```bash
# Monitor CPU usage
top -H -p <pid>

# If threads show high CPU but no progress → livelock

# Use perf to see what threads are doing
perf record -p <pid> -g
perf report
```

## 예방 모범 사례

### 체크리스트

- [ ] 고정 지연 대신 랜덤 백오프 사용
- [ ] 재시도를 위한 지수 백오프 구현
- [ ] 최대 재시도 제한 설정
- [ ] 가능한 경우 trylock 대신 락 순서 지정 사용
- [ ] 타임아웃 메커니즘 추가
- [ ] 진행 상황 메트릭 모니터링
- [ ] 부하 상태에서 여러 스레드로 테스트
- [ ] 대칭적 재시도 로직 피하기

### 디자인 패턴

**패턴 1: 비대칭 동작**
```c
void* thread_function(void* arg) {
    int id = *(int*)arg;

    // Even threads use one strategy
    if (id % 2 == 0) {
        strategy_a();
    }
    // Odd threads use another
    else {
        strategy_b();
    }
}
```

**패턴 2: 중앙 집중식 조정**
```cpp
std::mutex coordinator;

void coordinated_access() {
    // Single point of coordination prevents livelock
    std::lock_guard<std::mutex> lock(coordinator);
    access_resources();
}
```

## 비교 요약

```
Deadlock vs Livelock:

Deadlock:
  State: Blocked
  CPU: Idle
  Solution: Break circular wait
  Detection: Thread dumps show waiting

Livelock:
  State: Active
  CPU: Busy
  Solution: Add randomness/priority
  Detection: High CPU, no progress
```

## 연습 문제

### 연습 1: 라이브락 식별
이 코드에서 라이브락을 찾으십시오:
```cpp
void worker() {
    while (!try_acquire_resources()) {
        yield_to_others();
    }
    do_work();
}
```

### 연습 2: 네트워크 충돌 수정
네트워크 전송 시뮬레이션을 위한 적절한 지수 백오프를 구현하십시오.

### 연습 3: 진행 상황 모니터 구축
라이브락 조건을 탐지하는 모니터링 시스템을 만드십시오.

## 요약

**라이브락**은 다음과 같은 이유로 스레드가 활성 상태이지만 진전을 이루지 못하는 것입니다:
- 동기화된 재시도 패턴
- 과도한 정중함
- 무작위화 부족
- 적절한 백오프 없는 충돌

**교착 상태와의 주요 차이점:**
- 스레드가 활성 상태 (차단되지 않음)
- 높은 CPU 사용량
- 탐지가 더 어려움
- 다른 해결책 필요

**예방:**
- 랜덤/지수 백오프
- 우선순위 체계
- 락 순서 지정
- 진행 상황 모니터링

## 추가 자료

- "Operating Systems: Three Easy Pieces" - Remzi Arpaci-Dusseau
- "The Art of Multiprocessor Programming" - Herlihy & Shavit
- Ethernet CSMA/CD specification (IEEE 802.3)

## 다음 주제

[04-starvation.md](./04-starvation.md)로 계속하여 기아 상태에 대해 배우십시오.
