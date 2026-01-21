# Actor Model

## Overview

The Actor Model is a concurrency paradigm where independent "actors" communicate exclusively through asynchronous message passing. Each actor has its own private state and mailbox, processing messages sequentially. This eliminates shared mutable state and makes concurrent programming more manageable and scalable.

## Problem Statement

Traditional shared-memory concurrency has fundamental challenges:
- Shared mutable state leads to race conditions
- Locks cause deadlocks and reduce scalability
- Difficult to reason about complex thread interactions
- Hard to distribute across machines
- Callback hell in asynchronous code

## Solution Architecture

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

Key Principles:
1. No shared state between actors
2. Asynchronous message passing only
3. Each actor processes messages sequentially
4. Location transparency (local or remote)
```

## Basic Implementation (C++)

### Core Actor Framework

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

// Base message type
struct Message {
    virtual ~Message() = default;
};

// Actor address (unique identifier)
using ActorId = size_t;

// Actor reference for sending messages
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

// Base Actor class
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

    // Override to handle messages
    virtual void receive(std::shared_ptr<Message> msg) = 0;

    // Actor's message processing loop
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
                // Handle exception (could send to supervisor)
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

// Actor System (manages actor lifecycle)
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

        // Stop all actors
        for (auto& [id, actor] : actors_) {
            actor->stop();
        }

        // Wait for all threads
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

// ActorRef implementation
template<typename MsgType>
void ActorRef::send(const MsgType& msg) const {
    system_->send(id_, std::make_shared<MsgType>(msg));
}
```

### Example: Bank Account Actor

```cpp
// Messages
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

// Bank Account Actor
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

// Response handler actor
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

    // Create actors
    auto account = system.spawn<BankAccountActor>(1000.0);
    auto handler = system.spawn<ResponseHandlerActor>();

    // Send messages
    account.send(Deposit(500.0));
    account.send(Withdraw(200.0, handler));
    account.send(Withdraw(2000.0, handler));  // Should fail
    account.send(GetBalance(handler));

    // Let actors process messages
    std::this_thread::sleep_for(std::chrono::seconds(1));

    return 0;
}
```

## Advanced Patterns

### Supervision and Fault Tolerance

```cpp
// Supervision strategies
enum class SupervisionStrategy {
    RESTART,      // Restart failed actor
    STOP,         // Stop failed actor
    ESCALATE,     // Escalate to parent supervisor
    RESUME        // Resume actor (ignore error)
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
                // Restart logic
                break;

            case SupervisionStrategy::STOP:
                std::cout << "Stopping actor " << failed_id << "\n";
                system_->stop(failed_id);
                break;

            case SupervisionStrategy::ESCALATE:
                // Report to parent supervisor
                break;

            case SupervisionStrategy::RESUME:
                // Do nothing
                break;
        }
    }
};
```

### Request-Response Pattern

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
            stop();  // One-shot actor
        }
    }
};

// Usage: Request-response with future
auto future_actor = system.spawn<FutureActor<Balance>>();
auto future = future_actor.get_future();

account.send(GetBalance(future_actor));

auto balance = future.get();  // Blocks until response
std::cout << "Balance: $" << balance.amount << "\n";
```

### Router Pattern (Load Balancing)

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
            // Round-robin routing
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
            // Process work...
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    }
};

// Create router with workers
std::vector<ActorRef> workers;
for (int i = 0; i < 4; ++i) {
    workers.push_back(system.spawn<WorkerActor>(i));
}
auto router = system.spawn<RouterActor>(workers);

// Send work to router
for (int i = 0; i < 20; ++i) {
    router.send(WorkItem(i, "data_" + std::to_string(i)));
}
```

## Actor Coordination Patterns

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
        // Handle messages based on current state
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
        // Transition to PROCESSING
        state_ = State::PROCESSING;
    }

    void handle_processing(std::shared_ptr<Message> msg) {
        // Transition to SHIPPED
        state_ = State::SHIPPED;
    }

    void handle_shipped(std::shared_ptr<Message> msg) {
        // Transition to DELIVERED
        state_ = State::DELIVERED;
    }

    void handle_delivered(std::shared_ptr<Message> msg) {
        // Final state
    }
};
```

## Performance Considerations

### Message Batching

```cpp
struct BatchMessage : Message {
    std::vector<std::shared_ptr<Message>> messages;
};

// Process multiple messages at once to reduce overhead
void receive(std::shared_ptr<Message> msg) override {
    if (auto batch = std::dynamic_pointer_cast<BatchMessage>(msg)) {
        for (auto& m : batch->messages) {
            process_single(m);
        }
    }
}
```

### Mailbox Size Limits

```cpp
class BoundedMailboxActor : public Actor {
private:
    static constexpr size_t MAX_MAILBOX_SIZE = 1000;

public:
    bool enqueue(std::shared_ptr<Message> msg) {
        std::lock_guard<std::mutex> lock(mailbox_mutex_);

        if (mailbox_.size() >= MAX_MAILBOX_SIZE) {
            return false;  // Mailbox full, apply backpressure
        }

        mailbox_.push(msg);
        mailbox_cv_.notify_one();
        return true;
    }
};
```

## Real-World Applications

### 1. Erlang/OTP
The original and most successful actor model implementation.

### 2. Akka (JVM)
Popular actor framework for Java/Scala applications.

### 3. Orleans (.NET)
Virtual actor framework from Microsoft.

### 4. Distributed Systems
```
Actor 1 (Server A) --network--> Actor 2 (Server B)
```

### 5. Game Servers
```
Player Actor <--> NPC Actor <--> World Actor
```

### 6. Chat Systems
```
User Actor <--> Room Actor <--> Message Actor
```

## Pros and Cons

### Pros
- No shared mutable state (eliminates many bugs)
- Natural fit for distributed systems
- Location transparency
- Easy to reason about (single-threaded semantics per actor)
- Built-in fault tolerance with supervision
- Scales well (message passing)

### Cons
- Learning curve (different programming model)
- Message passing overhead
- Debugging can be challenging
- Not suitable for all problems
- Memory overhead per actor
- Potential for message queue buildup

## Best Practices

1. **Keep actors small and focused**:
   - Single responsibility
   - Minimal state

2. **Make messages immutable**:
   ```cpp
   struct ImmutableMessage : Message {
       const std::string data;
       ImmutableMessage(std::string d) : data(std::move(d)) {}
   };
   ```

3. **Use supervision trees**:
   ```
   Supervisor
   ├── Worker 1
   ├── Worker 2
   └── Worker 3
   ```

4. **Design for failure**:
   - Actors should be restartable
   - Separate persistent state from actor state

5. **Avoid blocking operations**:
   - Don't block waiting for responses
   - Use async I/O

6. **Monitor mailbox sizes**:
   - Apply backpressure when needed
   - Prevent memory exhaustion

## Testing Strategies

### Unit Testing
```cpp
// Test actor in isolation
BankAccountActor actor(1, &system, 100.0);
actor.receive(std::make_shared<Deposit>(50.0));
// Verify state
```

### Integration Testing
```cpp
// Test actor interactions
auto sender = system.spawn<SenderActor>();
auto receiver = system.spawn<ReceiverActor>();
sender.send(SendTo(receiver));
// Verify message received
```

### Chaos Testing
```cpp
// Random actor failures
// Message loss
// Out-of-order delivery
```

## Summary

The Actor Model is ideal for:
- Distributed systems
- High-concurrency applications
- Systems requiring fault tolerance
- Applications with complex state machines

It provides a fundamentally different approach to concurrency that eliminates shared state and makes concurrent programming more manageable at scale.
