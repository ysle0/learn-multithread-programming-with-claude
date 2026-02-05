# Actor Model

## 개요

Actor Model은 독립적인 "actor"들이 비동기 message passing을 통해서만 통신하는 동시성 패러다임입니다. 각 actor는 자체적인 private state와 mailbox를 가지며, message를 순차적으로 처리합니다. 이를 통해 공유 가변 상태를 제거하고 동시성 프로그래밍을 더 관리하기 쉽고 확장 가능하게 만듭니다.

## 문제 정의

전통적인 공유 메모리 동시성에는 근본적인 문제들이 있습니다:
- 공유 가변 상태는 경쟁 조건(race condition)을 유발합니다
- Lock은 deadlock을 일으키고 확장성을 저하시킵니다
- 복잡한 스레드 상호작용을 추론하기 어렵습니다
- 여러 머신에 걸쳐 분산하기 어렵습니다
- 비동기 코드에서의 콜백 지옥(callback hell)

## 솔루션 아키텍처

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Actor A   │         │   Actor B   │         │   Actor C   │
│             │         │             │         │             │
│  ┌───────┐  │         │  ┌───────┐  │         │  ┌───────┐  │
│  │ State │  │         │  │ State │  │         │  │ State │  │
│  └───────┘  │         │  └───────┘  │         │  └───────┘  │
│             │         │             │         │             │
│  ┌───────┐  │         │  ┌───────┐  │         │  ┌───────┐  │
│  │Mailbox│  │         │  │Mailbox│  │         │  │Mailbox│  │
│  │[M1 M2]│  │         │  │[M3]   │  │         │  │[M4 M5]│  │
│  └───────┘  │         │  └───────┘  │         │  └───────┘  │
└──────┬──────┘         └──────┬──────┘         └──────┬──────┘
       │                       │                       │
       │   send(msg)          │   send(msg)          │
       └──────────────────────┴───────────────────────┘

핵심 원칙:
1. Actor 간 공유 상태 없음
2. 비동기 message passing만 사용
3. 각 actor는 message를 순차적으로 처리
4. 위치 투명성 (로컬 또는 원격)
```

## 기본 구현 (C++)

### 핵심 Actor 프레임워크

```cpp
#include <memory>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <thread>
#include <unordered_map>
#include <functional>
#include <any>
#include <optional>

// 기본 message 타입
struct Message {
    virtual ~Message() = default;
};

// Actor 주소 (고유 식별자)
using ActorId = size_t;

// Message 전송을 위한 actor 참조
class ActorRef {
private:
    ActorId id_;
    class ActorSystem* system_;

public:
    ActorRef(ActorId id, ActorSystem* system) : id_(id), system_(system) {}

    template<typename MsgType>
    void send(const MsgType& msg) const;

    ActorId id() const { return id_; }
};

// 기본 Actor 클래스
class Actor {
protected:
    ActorId id_;
    ActorSystem* system_;
    std::queue<std::shared_ptr<Message>> mailbox_;
    std::mutex mailbox_mutex_;
    std::condition_variable mailbox_cv_;
    std::atomic<bool> stopped_{false};

public:
    Actor(ActorId id, ActorSystem* system)
        : id_(id), system_(system) {}

    virtual ~Actor() = default;

    // Message 처리를 위해 오버라이드
    virtual void receive(std::shared_ptr<Message> msg) = 0;

    // Actor의 message 처리 루프
    void run() {
        while (!stopped_.load()) {
            std::shared_ptr<Message> msg;

            {
                std::unique_lock<std::mutex> lock(mailbox_mutex_);
                mailbox_cv_.wait_for(lock, std::chrono::milliseconds(100),
                    [this] { return !mailbox_.empty() || stopped_.load(); });

                if (stopped_.load() && mailbox_.empty()) {
                    break;
                }

                if (mailbox_.empty()) {
                    continue;
                }

                msg = mailbox_.front();
                mailbox_.pop();
            }

            try {
                receive(msg);
            } catch (const std::exception& e) {
                // 예외 처리 (supervisor에게 전송 가능)
                std::cerr << "Actor " << id_ << " error: " << e.what() << "\n";
            }
        }
    }

    void enqueue(std::shared_ptr<Message> msg) {
        {
            std::lock_guard<std::mutex> lock(mailbox_mutex_);
            mailbox_.push(msg);
        }
        mailbox_cv_.notify_one();
    }

    void stop() {
        stopped_.store(true);
        mailbox_cv_.notify_one();
    }

    ActorId id() const { return id_; }
};

// Actor System (actor 생명주기 관리)
class ActorSystem {
private:
    std::unordered_map<ActorId, std::unique_ptr<Actor>> actors_;
    std::unordered_map<ActorId, std::thread> threads_;
    std::mutex system_mutex_;
    ActorId next_id_ = 1;

public:
    ~ActorSystem() {
        shutdown();
    }

    template<typename ActorType, typename... Args>
    ActorRef spawn(Args&&... args) {
        std::lock_guard<std::mutex> lock(system_mutex_);

        ActorId id = next_id_++;
        auto actor = std::make_unique<ActorType>(id, this, std::forward<Args>(args)...);
        auto thread = std::thread([actor_ptr = actor.get()] {
            actor_ptr->run();
        });

        ActorRef ref(id, this);
        actors_[id] = std::move(actor);
        threads_[id] = std::move(thread);

        return ref;
    }

    void send(ActorId id, std::shared_ptr<Message> msg) {
        std::lock_guard<std::mutex> lock(system_mutex_);

        auto it = actors_.find(id);
        if (it != actors_.end()) {
            it->second->enqueue(msg);
        }
    }

    void stop(ActorId id) {
        std::lock_guard<std::mutex> lock(system_mutex_);

        auto it = actors_.find(id);
        if (it != actors_.end()) {
            it->second->stop();
        }
    }

    void shutdown() {
        std::unique_lock<std::mutex> lock(system_mutex_);

        // 모든 actor 중지
        for (auto& [id, actor] : actors_) {
            actor->stop();
        }

        // 모든 스레드 대기
        auto threads = std::move(threads_);
        lock.unlock();

        for (auto& [id, thread] : threads) {
            if (thread.joinable()) {
                thread.join();
            }
        }

        lock.lock();
        actors_.clear();
    }
};

// ActorRef 구현
template<typename MsgType>
void ActorRef::send(const MsgType& msg) const {
    system_->send(id_, std::make_shared<MsgType>(msg));
}
```

### 예제: 은행 계좌 Actor

```cpp
// Message 정의
struct Deposit : Message {
    double amount;
    Deposit(double amt) : amount(amt) {}
};

struct Withdraw : Message {
    double amount;
    ActorRef reply_to;
    Withdraw(double amt, ActorRef ref) : amount(amt), reply_to(ref) {}
};

struct WithdrawResult : Message {
    bool success;
    double balance;
    WithdrawResult(bool s, double b) : success(s), balance(b) {}
};

struct GetBalance : Message {
    ActorRef reply_to;
    GetBalance(ActorRef ref) : reply_to(ref) {}
};

struct Balance : Message {
    double amount;
    Balance(double amt) : amount(amt) {}
};

// 은행 계좌 Actor
class BankAccountActor : public Actor {
private:
    double balance_;

public:
    BankAccountActor(ActorId id, ActorSystem* system, double initial_balance)
        : Actor(id, system), balance_(initial_balance) {
        std::cout << "Account " << id << " created with balance: $"
                  << balance_ << "\n";
    }

    void receive(std::shared_ptr<Message> msg) override {
        if (auto deposit = std::dynamic_pointer_cast<Deposit>(msg)) {
            balance_ += deposit->amount;
            std::cout << "Account " << id_ << ": Deposited $" << deposit->amount
                      << ", new balance: $" << balance_ << "\n";
        }
        else if (auto withdraw = std::dynamic_pointer_cast<Withdraw>(msg)) {
            bool success = false;
            if (balance_ >= withdraw->amount) {
                balance_ -= withdraw->amount;
                success = true;
                std::cout << "Account " << id_ << ": Withdrew $" << withdraw->amount
                          << ", new balance: $" << balance_ << "\n";
            } else {
                std::cout << "Account " << id_ << ": Insufficient funds for $"
                          << withdraw->amount << "\n";
            }

            withdraw->reply_to.send(WithdrawResult(success, balance_));
        }
        else if (auto get_bal = std::dynamic_pointer_cast<GetBalance>(msg)) {
            get_bal->reply_to.send(Balance(balance_));
        }
    }
};

// 응답 처리 actor
class ResponseHandlerActor : public Actor {
public:
    ResponseHandlerActor(ActorId id, ActorSystem* system)
        : Actor(id, system) {}

    void receive(std::shared_ptr<Message> msg) override {
        if (auto result = std::dynamic_pointer_cast<WithdrawResult>(msg)) {
            std::cout << "Withdraw " << (result->success ? "succeeded" : "failed")
                      << ", balance: $" << result->balance << "\n";
        }
        else if (auto balance = std::dynamic_pointer_cast<Balance>(msg)) {
            std::cout << "Current balance: $" << balance->amount << "\n";
        }
    }
};

int main() {
    ActorSystem system;

    // Actor 생성
    auto account = system.spawn<BankAccountActor>(1000.0);
    auto handler = system.spawn<ResponseHandlerActor>();

    // Message 전송
    account.send(Deposit(500.0));
    account.send(Withdraw(200.0, handler));
    account.send(Withdraw(2000.0, handler));  // 실패해야 함
    account.send(GetBalance(handler));

    // Actor들이 message를 처리할 시간을 줌
    std::this_thread::sleep_for(std::chrono::seconds(1));

    return 0;
}
```

## 고급 패턴

### Supervision과 장애 허용(Fault Tolerance)

```cpp
// Supervision 전략
enum class SupervisionStrategy {
    RESTART,      // 실패한 actor 재시작
    STOP,         // 실패한 actor 중지
    ESCALATE,     // 부모 supervisor에게 에스컬레이션
    RESUME        // Actor 재개 (오류 무시)
};

struct ActorFailed : Message {
    ActorId failed_actor;
    std::string reason;
    ActorFailed(ActorId id, const std::string& r)
        : failed_actor(id), reason(r) {}
};

class SupervisorActor : public Actor {
private:
    std::vector<ActorRef> children_;
    SupervisionStrategy strategy_;

public:
    SupervisorActor(ActorId id, ActorSystem* system, SupervisionStrategy strategy)
        : Actor(id, system), strategy_(strategy) {}

    void spawn_child(ActorRef child) {
        children_.push_back(child);
    }

    void receive(std::shared_ptr<Message> msg) override {
        if (auto failed = std::dynamic_pointer_cast<ActorFailed>(msg)) {
            handle_failure(failed->failed_actor, failed->reason);
        }
    }

private:
    void handle_failure(ActorId failed_id, const std::string& reason) {
        switch (strategy_) {
            case SupervisionStrategy::RESTART:
                std::cout << "Restarting actor " << failed_id << "\n";
                // 재시작 로직
                break;

            case SupervisionStrategy::STOP:
                std::cout << "Stopping actor " << failed_id << "\n";
                system_->stop(failed_id);
                break;

            case SupervisionStrategy::ESCALATE:
                // 부모 supervisor에게 보고
                break;

            case SupervisionStrategy::RESUME:
                // 아무것도 하지 않음
                break;
        }
    }
};
```

### Request-Response 패턴

```cpp
#include <future>

template<typename ResponseType>
class FutureActor : public Actor {
private:
    std::promise<ResponseType> promise_;

public:
    FutureActor(ActorId id, ActorSystem* system)
        : Actor(id, system) {}

    std::future<ResponseType> get_future() {
        return promise_.get_future();
    }

    void receive(std::shared_ptr<Message> msg) override {
        if (auto response = std::dynamic_pointer_cast<ResponseType>(msg)) {
            promise_.set_value(*response);
            stop();  // 일회성 actor
        }
    }
};

// 사용법: future를 이용한 request-response
auto future_actor = system.spawn<FutureActor<Balance>>();
auto future = future_actor.get_future();

account.send(GetBalance(future_actor));

auto balance = future.get();  // 응답이 올 때까지 블로킹
std::cout << "Balance: $" << balance.amount << "\n";
```

### Router 패턴 (부하 분산)

```cpp
struct WorkItem : Message {
    int id;
    std::string data;
    WorkItem(int i, const std::string& d) : id(i), data(d) {}
};

class RouterActor : public Actor {
private:
    std::vector<ActorRef> workers_;
    size_t next_worker_ = 0;

public:
    RouterActor(ActorId id, ActorSystem* system, std::vector<ActorRef> workers)
        : Actor(id, system), workers_(std::move(workers)) {}

    void receive(std::shared_ptr<Message> msg) override {
        if (auto work = std::dynamic_pointer_cast<WorkItem>(msg)) {
            // 라운드 로빈 라우팅
            workers_[next_worker_].send(*work);
            next_worker_ = (next_worker_ + 1) % workers_.size();
        }
    }
};

class WorkerActor : public Actor {
private:
    int id_;

public:
    WorkerActor(ActorId actor_id, ActorSystem* system, int worker_id)
        : Actor(actor_id, system), id_(worker_id) {}

    void receive(std::shared_ptr<Message> msg) override {
        if (auto work = std::dynamic_pointer_cast<WorkItem>(msg)) {
            std::cout << "Worker " << id_ << " processing item "
                      << work->id << "\n";
            // 작업 처리...
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    }
};

// Worker들로 router 생성
std::vector<ActorRef> workers;
for (int i = 0; i < 4; ++i) {
    workers.push_back(system.spawn<WorkerActor>(i));
}
auto router = system.spawn<RouterActor>(workers);

// Router에 작업 전송
for (int i = 0; i < 20; ++i) {
    router.send(WorkItem(i, "data_" + std::to_string(i)));
}
```

## Actor 조정 패턴

### Scatter-Gather

```cpp
struct ScatterRequest : Message {
    std::vector<std::string> tasks;
    ActorRef aggregator;
    ScatterRequest(std::vector<std::string> t, ActorRef agg)
        : tasks(std::move(t)), aggregator(agg) {}
};

struct GatherResult : Message {
    std::string result;
    GatherResult(const std::string& r) : result(r) {}
};

class AggregatorActor : public Actor {
private:
    size_t expected_results_;
    std::vector<std::string> results_;

public:
    AggregatorActor(ActorId id, ActorSystem* system, size_t expected)
        : Actor(id, system), expected_results_(expected) {}

    void receive(std::shared_ptr<Message> msg) override {
        if (auto result = std::dynamic_pointer_cast<GatherResult>(msg)) {
            results_.push_back(result->result);

            if (results_.size() == expected_results_) {
                std::cout << "All results gathered:\n";
                for (const auto& r : results_) {
                    std::cout << "  - " << r << "\n";
                }
                stop();
            }
        }
    }
};
```

### State Machine Actor

```cpp
class OrderActor : public Actor {
private:
    enum class State { PENDING, PROCESSING, SHIPPED, DELIVERED };
    State state_ = State::PENDING;
    std::string order_id_;

public:
    OrderActor(ActorId id, ActorSystem* system, const std::string& order_id)
        : Actor(id, system), order_id_(order_id) {}

    void receive(std::shared_ptr<Message> msg) override {
        // 현재 상태에 따라 message 처리
        switch (state_) {
            case State::PENDING:
                handle_pending(msg);
                break;
            case State::PROCESSING:
                handle_processing(msg);
                break;
            case State::SHIPPED:
                handle_shipped(msg);
                break;
            case State::DELIVERED:
                handle_delivered(msg);
                break;
        }
    }

private:
    void handle_pending(std::shared_ptr<Message> msg) {
        // PROCESSING으로 전환
        state_ = State::PROCESSING;
    }

    void handle_processing(std::shared_ptr<Message> msg) {
        // SHIPPED로 전환
        state_ = State::SHIPPED;
    }

    void handle_shipped(std::shared_ptr<Message> msg) {
        // DELIVERED로 전환
        state_ = State::DELIVERED;
    }

    void handle_delivered(std::shared_ptr<Message> msg) {
        // 최종 상태
    }
};
```

## 성능 고려사항

### Message 배치 처리

```cpp
struct BatchMessage : Message {
    std::vector<std::shared_ptr<Message>> messages;
};

// 오버헤드를 줄이기 위해 여러 message를 한 번에 처리
void receive(std::shared_ptr<Message> msg) override {
    if (auto batch = std::dynamic_pointer_cast<BatchMessage>(msg)) {
        for (auto& m : batch->messages) {
            process_single(m);
        }
    }
}
```

### Mailbox 크기 제한

```cpp
class BoundedMailboxActor : public Actor {
private:
    static constexpr size_t MAX_MAILBOX_SIZE = 1000;

public:
    bool enqueue(std::shared_ptr<Message> msg) {
        std::lock_guard<std::mutex> lock(mailbox_mutex_);

        if (mailbox_.size() >= MAX_MAILBOX_SIZE) {
            return false;  // Mailbox가 가득 참, 배압(backpressure) 적용
        }

        mailbox_.push(msg);
        mailbox_cv_.notify_one();
        return true;
    }
};
```

## 실제 활용 사례

### 1. Erlang/OTP
최초이자 가장 성공적인 actor model 구현체입니다.

### 2. Akka (JVM)
Java/Scala 애플리케이션을 위한 인기 있는 actor 프레임워크입니다.

### 3. Orleans (.NET)
Microsoft의 가상 actor 프레임워크입니다.

### 4. 분산 시스템
```
Actor 1 (Server A) --network--> Actor 2 (Server B)
```

### 5. 게임 서버
```
Player Actor <--> NPC Actor <--> World Actor
```

### 6. 채팅 시스템
```
User Actor <--> Room Actor <--> Message Actor
```

## 장점과 단점

### 장점
- 공유 가변 상태가 없음 (많은 버그를 제거)
- 분산 시스템에 자연스러운 적합성
- 위치 투명성
- 추론이 쉬움 (actor당 단일 스레드 의미론)
- Supervision을 통한 내장 장애 허용
- 우수한 확장성 (message passing)

### 단점
- 학습 곡선 (다른 프로그래밍 모델)
- Message passing 오버헤드
- 디버깅이 어려울 수 있음
- 모든 문제에 적합하지 않음
- Actor당 메모리 오버헤드
- Message 큐 누적 가능성

## 모범 사례

1. **Actor를 작고 집중적으로 유지하기**:
   - 단일 책임
   - 최소한의 상태

2. **Message를 불변으로 만들기**:
   ```cpp
   struct ImmutableMessage : Message {
       const std::string data;
       ImmutableMessage(std::string d) : data(std::move(d)) {}
   };
   ```

3. **Supervision 트리 사용하기**:
   ```
   Supervisor
   ├── Worker 1
   ├── Worker 2
   └── Worker 3
   ```

4. **장애를 고려한 설계**:
   - Actor는 재시작 가능해야 함
   - 영속적 상태와 actor 상태를 분리

5. **블로킹 작업 피하기**:
   - 응답을 기다리며 블로킹하지 않기
   - 비동기 I/O 사용

6. **Mailbox 크기 모니터링**:
   - 필요 시 배압(backpressure) 적용
   - 메모리 고갈 방지

## 테스트 전략

### 단위 테스트
```cpp
// Actor를 격리하여 테스트
BankAccountActor actor(1, &system, 100.0);
actor.receive(std::make_shared<Deposit>(50.0));
// 상태 검증
```

### 통합 테스트
```cpp
// Actor 간 상호작용 테스트
auto sender = system.spawn<SenderActor>();
auto receiver = system.spawn<ReceiverActor>();
sender.send(SendTo(receiver));
// Message 수신 검증
```

### 카오스 테스트
```cpp
// 무작위 actor 장애
// Message 손실
// 순서가 바뀐 전달
```

## 요약

Actor Model은 다음과 같은 경우에 이상적입니다:
- 분산 시스템
- 높은 동시성 애플리케이션
- 장애 허용이 필요한 시스템
- 복잡한 상태 머신이 있는 애플리케이션

Actor Model은 공유 상태를 제거하고 대규모 동시성 프로그래밍을 더 관리하기 쉽게 만드는 근본적으로 다른 동시성 접근 방식을 제공합니다.
