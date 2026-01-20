# libcds - Concurrent Data Structures Library

## 📌 프로젝트 개요

**libcds (Concurrent Data Structures library)**는 C++로 작성된 고성능 Lock-Free 및 Wait-Free 자료구조 라이브러리입니다.

### 기본 정보
- **저장소**: https://github.com/khizmax/libcds
- **언어**: C++11/14/17
- **라이선스**: Boost Software License 1.0
- **주요 개발자**: Maxim Khizhinsky
- **활성 상태**: 활발히 유지보수 중
- **난이도**: ⭐⭐⭐⭐⭐

### 주요 특징
- Header-only 라이브러리 (일부 컴포넌트 제외)
- 다양한 Lock-Free 자료구조
- 여러 메모리 회수(Memory Reclamation) 기법 지원
- 크로스 플랫폼 (Windows, Linux, macOS)
- 광범위한 단위 테스트와 벤치마크

---

## 🎯 왜 libcds를 분석해야 하는가?

### 학습 가치

1. **Lock-Free 알고리즘의 교과서**
   - 학술 논문의 이론을 실제 구현으로 확인
   - 다양한 Lock-Free 자료구조 구현 예제

2. **메모리 회수 기법의 표준**
   - Hazard Pointers
   - Epoch-Based Reclamation (EBR)
   - Reference Counting
   - 각 기법의 장단점 비교 가능

3. **프로덕션 레벨 코드**
   - 실전에서 사용 가능한 품질
   - 엣지 케이스와 최적화 고려
   - 철저한 테스트

4. **성능 최적화 기법**
   - Cache-friendly 설계
   - False sharing 회피
   - Memory ordering 최적화

---

## 🏗️ 아키텍처

### 전체 구조

```
libcds/
├── cds/                      # 핵심 라이브러리
│   ├── algo/                 # 알고리즘 (split-ordered list 등)
│   ├── container/            # 컨테이너 (Stack, Queue, Map 등)
│   ├── gc/                   # Garbage Collection (메모리 회수)
│   ├── intrusive/            # Intrusive 자료구조
│   ├── opt/                  # 옵션 및 정책
│   └── sync/                 # 동기화 프리미티브
├── test/                     # 단위 테스트
└── bench/                    # 벤치마크
```

### 핵심 컴포넌트

#### 1. Garbage Collection (GC) 서브시스템
메모리 회수 전략을 추상화한 계층:

```cpp
namespace cds { namespace gc {
    class HP;        // Hazard Pointers
    class DHP;       // Dynamic Hazard Pointers
    class DHPGC;     // DHP with explicit GC
    class nogc;      // No GC (수동 관리)
}}
```

#### 2. Container 계층
사용하기 쉬운 STL-like 인터페이스:

```cpp
// Lock-Free Stack
cds::container::TreiberStack<int, cds::gc::HP> stack;

// Lock-Free Queue
cds::container::MoirQueue<int, cds::gc::HP> queue;

// Lock-Free Map
cds::container::MichaelHashMap<int, std::string, cds::gc::HP> map;
```

#### 3. Intrusive 계층
메모리 할당을 사용자가 제어하는 저수준 인터페이스:

```cpp
struct MyData : public cds::intrusive::treiber_stack::node<cds::gc::HP>
{
    int value;
};

cds::intrusive::TreiberStack<cds::gc::HP, MyData> stack;
```

---

## 🔬 핵심 자료구조 분석

### 1. Treiber Stack

#### 개요
가장 간단하면서도 효율적인 Lock-Free Stack 구현입니다.

#### 핵심 코드 분석

```cpp
template <typename GC, typename T>
class TreiberStack
{
private:
    struct Node {
        T data;
        typename GC::template GuardedPtr<Node> next;

        Node(T const& val) : data(val), next(nullptr) {}
    };

    std::atomic<Node*> m_pHead;

public:
    void push(T const& val)
    {
        Node* pNode = new Node(val);
        Node* pHead;

        do {
            pHead = m_pHead.load(std::memory_order_acquire);
            pNode->next.store(pHead, std::memory_order_relaxed);
        } while (!m_pHead.compare_exchange_weak(
            pHead, pNode,
            std::memory_order_release,
            std::memory_order_acquire
        ));
    }

    bool pop(T& dest)
    {
        typename GC::Guard guard;
        Node* pHead;

        do {
            pHead = guard.protect(m_pHead);
            if (!pHead)
                return false;

            Node* pNext = pHead->next.load(std::memory_order_acquire);
        } while (!m_pHead.compare_exchange_weak(
            pHead, pNext,
            std::memory_order_release,
            std::memory_order_acquire
        ));

        dest = pHead->data;
        GC::retire(pHead);
        return true;
    }
};
```

#### 핵심 기법

**1. CAS 루프 (Compare-And-Swap Loop)**
```cpp
do {
    pHead = m_pHead.load(std::memory_order_acquire);
    pNode->next = pHead;
} while (!m_pHead.compare_exchange_weak(pHead, pNode, ...));
```
- ABA 문제를 메모리 회수 기법(GC)으로 해결
- Acquire-Release 시맨틱으로 메모리 순서 보장

**2. Hazard Pointer 통합**
```cpp
typename GC::Guard guard;
pHead = guard.protect(m_pHead);
```
- `Guard`가 노드를 보호하여 안전하게 접근
- 다른 스레드가 해당 노드를 삭제하지 못하도록 방지

---

### 2. Michael-Scott Queue

#### 개요
가장 널리 사용되는 Lock-Free Queue 알고리즘입니다.

#### 핵심 구조

```cpp
template <typename GC, typename T>
class MSQueue
{
private:
    struct Node {
        std::atomic<T*> data;
        std::atomic<Node*> next;

        Node() : data(nullptr), next(nullptr) {}
        Node(T const& val) : data(new T(val)), next(nullptr) {}
    };

    std::atomic<Node*> m_pHead;
    std::atomic<Node*> m_pTail;

public:
    MSQueue()
    {
        Node* pNode = new Node();  // Dummy node
        m_pHead.store(pNode, std::memory_order_relaxed);
        m_pTail.store(pNode, std::memory_order_relaxed);
    }

    void enqueue(T const& val)
    {
        Node* pNode = new Node(val);
        typename GC::Guard tailGuard;

        for (;;) {
            Node* pTail = tailGuard.protect(m_pTail);
            Node* pNext = pTail->next.load(std::memory_order_acquire);

            if (pTail == m_pTail.load(std::memory_order_acquire)) {
                if (pNext == nullptr) {
                    // Tail이 실제 마지막을 가리킴
                    if (pTail->next.compare_exchange_weak(
                        pNext, pNode,
                        std::memory_order_release,
                        std::memory_order_acquire))
                    {
                        // CAS 성공, Tail 업데이트 시도
                        m_pTail.compare_exchange_strong(
                            pTail, pNode,
                            std::memory_order_release,
                            std::memory_order_acquire
                        );
                        break;
                    }
                }
                else {
                    // Tail이 뒤쳐짐, 도와주기
                    m_pTail.compare_exchange_weak(
                        pTail, pNext,
                        std::memory_order_release,
                        std::memory_order_acquire
                    );
                }
            }
        }
    }

    bool dequeue(T& dest)
    {
        typename GC::Guard headGuard, nextGuard;

        for (;;) {
            Node* pHead = headGuard.protect(m_pHead);
            Node* pTail = m_pTail.load(std::memory_order_acquire);
            Node* pNext = nextGuard.protect(pHead->next);

            if (pHead == m_pHead.load(std::memory_order_acquire)) {
                if (pHead == pTail) {
                    if (pNext == nullptr)
                        return false;  // 큐가 비어있음

                    // Tail이 뒤쳐짐, 도와주기
                    m_pTail.compare_exchange_weak(
                        pTail, pNext,
                        std::memory_order_release,
                        std::memory_order_acquire
                    );
                }
                else {
                    T* pData = pNext->data.load(std::memory_order_acquire);
                    if (m_pHead.compare_exchange_weak(
                        pHead, pNext,
                        std::memory_order_release,
                        std::memory_order_acquire))
                    {
                        dest = *pData;
                        GC::retire(pHead);
                        delete pData;
                        return true;
                    }
                }
            }
        }
    }
};
```

#### 핵심 개념

**1. Dummy Node**
- 큐가 비어있을 때도 항상 하나의 노드 유지
- Head와 Tail 처리를 단순화

**2. Helping Mechanism (도와주기)**
```cpp
// 다른 스레드가 Tail을 업데이트하지 못했다면 대신 해줌
m_pTail.compare_exchange_weak(pTail, pNext, ...);
```
- Lock-Free 진전 보장의 핵심
- 한 스레드가 멈춰도 다른 스레드가 계속 진행

**3. 이중 Hazard Pointer**
```cpp
typename GC::Guard headGuard, nextGuard;
pHead = headGuard.protect(m_pHead);
pNext = nextGuard.protect(pHead->next);
```
- Head와 Next 모두 보호
- 순차적 보호로 안전성 보장

---

### 3. Split-Ordered List 기반 Hash Map

#### 개요
동적으로 크기가 조정되는 Lock-Free Hash Map입니다.

#### 핵심 아이디어

**1. Split-Ordered Hashing**
```
일반적인 순서: 000, 001, 010, 011, 100, 101, 110, 111
Split-Order:   000, 100, 010, 110, 001, 101, 011, 111 (역순 비트)
```

역순 비트를 사용하면:
- 해시 테이블 크기를 2배로 늘릴 때 기존 노드의 절반만 이동
- Lock 없이 동적 크기 조정 가능

**2. Sentinel 노드**
```cpp
struct Node {
    size_t hash;         // 역순 비트 해시
    T key;
    V value;
    std::atomic<Node*> next;
    bool is_sentinel;    // 버킷의 시작 표시
};
```

#### 간소화된 구조

```cpp
template <typename K, typename V, typename GC>
class SplitListMap
{
private:
    struct Node {
        size_t reversed_hash;
        K key;
        V value;
        std::atomic<Node*> next;
    };

    std::atomic<Node*> m_List;          // 정렬된 링크드 리스트
    std::vector<std::atomic<Node*>> m_Buckets;  // 버킷 배열
    std::atomic<size_t> m_ItemCount;

    size_t reverse_bits(size_t hash) const
    {
        // 비트를 역순으로 뒤집기
        size_t result = 0;
        for (size_t i = 0; i < sizeof(size_t) * 8; ++i) {
            result = (result << 1) | (hash & 1);
            hash >>= 1;
        }
        return result;
    }

    Node* get_bucket(size_t bucket_idx)
    {
        // 버킷이 초기화되지 않았다면 초기화
        Node* pBucket = m_Buckets[bucket_idx].load(std::memory_order_acquire);
        if (!pBucket) {
            // Lazy initialization
            pBucket = init_bucket(bucket_idx);
        }
        return pBucket;
    }

public:
    bool insert(K const& key, V const& value)
    {
        size_t hash = std::hash<K>{}(key);
        size_t reversed = reverse_bits(hash);
        size_t bucket_idx = hash % m_Buckets.size();

        Node* pBucket = get_bucket(bucket_idx);

        // 정렬된 리스트에 삽입
        Node* pNode = new Node{reversed, key, value, nullptr};
        // ... Lock-Free 삽입 로직 ...

        // 필요시 리사이징
        if (++m_ItemCount > threshold) {
            resize();
        }

        return true;
    }
};
```

#### 장점
- Lock-Free 동적 크기 조정
- O(1) 평균 시간 복잡도
- 높은 동시성 확장성

---

## 💾 메모리 회수 기법

libcds의 핵심 강점 중 하나는 여러 메모리 회수 기법을 지원한다는 것입니다.

### 1. Hazard Pointers (HP)

#### 원리
```cpp
class HazardPointerGC
{
private:
    static constexpr size_t MAX_HAZARDS = 100;

    thread_local static HazardPointer hazards[MAX_HAZARDS];
    static std::vector<Node*> retired_list;

public:
    class Guard {
        HazardPointer* hp;

    public:
        template <typename T>
        T* protect(std::atomic<T*>& src)
        {
            T* p;
            do {
                p = src.load(std::memory_order_acquire);
                hp->store(p, std::memory_order_release);
            } while (p != src.load(std::memory_order_acquire));
            return p;
        }

        ~Guard() {
            hp->store(nullptr, std::memory_order_release);
        }
    };

    static void retire(Node* p)
    {
        retired_list.push_back(p);

        if (retired_list.size() > THRESHOLD) {
            scan();
        }
    }

private:
    static void scan()
    {
        // 모든 Hazard Pointer 수집
        std::set<Node*> protected_nodes;
        for (auto& hp : all_hazards) {
            Node* p = hp.load(std::memory_order_acquire);
            if (p) protected_nodes.insert(p);
        }

        // 보호되지 않은 노드 삭제
        auto it = retired_list.begin();
        while (it != retired_list.end()) {
            if (protected_nodes.find(*it) == protected_nodes.end()) {
                delete *it;
                it = retired_list.erase(it);
            } else {
                ++it;
            }
        }
    }
};
```

#### 장단점
- ✅ **간단한 개념**: 이해하기 쉬움
- ✅ **예측 가능**: 메모리 사용량 제어 가능
- ⚠️ **오버헤드**: 스캔 비용
- ⚠️ **제한된 슬롯**: Hazard Pointer 개수 제한

### 2. Epoch-Based Reclamation (EBR)

#### 원리
```cpp
class EpochGC
{
private:
    static std::atomic<uint64_t> global_epoch;
    thread_local static uint64_t local_epoch;
    thread_local static std::vector<Node*> retired_lists[3];

public:
    class Guard {
    public:
        Guard() {
            local_epoch = global_epoch.load(std::memory_order_acquire);
        }

        ~Guard() {
            local_epoch = 0;
        }
    };

    static void retire(Node* p)
    {
        uint64_t epoch = global_epoch.load(std::memory_order_acquire);
        retired_lists[epoch % 3].push_back(p);
    }

    static void try_advance_epoch()
    {
        uint64_t current_epoch = global_epoch.load(std::memory_order_acquire);

        // 모든 스레드가 현재 에포크에 있는지 확인
        bool can_advance = true;
        for (auto& thread_epoch : all_thread_epochs) {
            if (thread_epoch != 0 && thread_epoch < current_epoch) {
                can_advance = false;
                break;
            }
        }

        if (can_advance) {
            global_epoch.fetch_add(1, std::memory_order_release);

            // 2 에포크 전 리스트의 노드들 삭제
            uint64_t old_epoch = (current_epoch + 1) % 3;
            for (Node* p : retired_lists[old_epoch]) {
                delete p;
            }
            retired_lists[old_epoch].clear();
        }
    }
};
```

#### 장단점
- ✅ **낮은 오버헤드**: Hazard Pointer보다 빠름
- ✅ **무제한 보호**: 포인터 개수 제한 없음
- ⚠️ **지연된 회수**: 에포크 진전 대기
- ⚠️ **메모리 증가**: 일시적으로 많은 메모리 사용 가능

---

## 🚀 사용 예제

### 기본 사용법

```cpp
#include <cds/init.h>
#include <cds/gc/hp.h>
#include <cds/container/treiber_stack.h>
#include <cds/container/msqueue.h>
#include <thread>

int main()
{
    // 1. libcds 초기화
    cds::Initialize();

    // 2. GC 초기화
    {
        cds::gc::HP hpGC;  // Hazard Pointer GC

        // 3. 자료구조 생성
        cds::container::TreiberStack<cds::gc::HP, int> stack;
        cds::container::MSQueue<cds::gc::HP, int> queue;

        // 4. 다중 스레드 사용
        std::vector<std::thread> threads;

        // Producer 스레드
        for (int i = 0; i < 4; ++i) {
            threads.emplace_back([&stack, &queue, i]() {
                // 각 스레드는 GC에 등록해야 함
                cds::gc::HP::thread_gc myGC;

                for (int j = 0; j < 1000; ++j) {
                    stack.push(i * 1000 + j);
                    queue.enqueue(i * 1000 + j);
                }
            });
        }

        // Consumer 스레드
        for (int i = 0; i < 4; ++i) {
            threads.emplace_back([&stack, &queue]() {
                cds::gc::HP::thread_gc myGC;

                int value;
                for (int j = 0; j < 1000; ++j) {
                    while (!stack.pop(value)) {
                        std::this_thread::yield();
                    }

                    while (!queue.dequeue(value)) {
                        std::this_thread::yield();
                    }
                }
            });
        }

        for (auto& t : threads) {
            t.join();
        }

    }  // GC 정리

    // 5. libcds 종료
    cds::Terminate();

    return 0;
}
```

### 성능 벤치마크

```cpp
#include <benchmark/benchmark.h>
#include <cds/container/treiber_stack.h>
#include <mutex>
#include <stack>

// Lock-Free Stack
static void BM_TreiberStack(benchmark::State& state)
{
    cds::Initialize();
    cds::gc::HP hpGC;
    cds::gc::HP::thread_gc myGC;

    cds::container::TreiberStack<cds::gc::HP, int> stack;

    for (auto _ : state) {
        stack.push(42);
        int value;
        stack.pop(value);
    }

    cds::Terminate();
}
BENCHMARK(BM_TreiberStack)->Threads(1)->Threads(2)->Threads(4)->Threads(8);

// 전통적인 Mutex 기반 Stack
static void BM_MutexStack(benchmark::State& state)
{
    std::stack<int> stack;
    std::mutex mtx;

    for (auto _ : state) {
        {
            std::lock_guard<std::mutex> lock(mtx);
            stack.push(42);
        }
        {
            std::lock_guard<std::mutex> lock(mtx);
            if (!stack.empty()) {
                int value = stack.top();
                stack.pop();
            }
        }
    }
}
BENCHMARK(BM_MutexStack)->Threads(1)->Threads(2)->Threads(4)->Threads(8);

BENCHMARK_MAIN();
```

**예상 결과:**
```
Benchmark                   Time        CPU     Iterations
---------------------------------------------------------
BM_TreiberStack/threads:1   45 ns      45 ns   15552341
BM_TreiberStack/threads:2   78 ns      156 ns   8976543
BM_TreiberStack/threads:4   95 ns      380 ns   7345678
BM_TreiberStack/threads:8   112 ns     896 ns   6234567

BM_MutexStack/threads:1     32 ns      32 ns   21876543
BM_MutexStack/threads:2     156 ns     312 ns   4456789
BM_MutexStack/threads:4     489 ns     1956 ns  1123456
BM_MutexStack/threads:8     1234 ns    9872 ns  567890
```

**분석:**
- 낮은 경합 (1 스레드): Mutex가 약간 빠름 (오버헤드 적음)
- 높은 경합 (8 스레드): Lock-Free가 10배 이상 빠름

---

## 📊 성능 특성

### Treiber Stack
- **Push**: O(1) amortized
- **Pop**: O(1) amortized
- **경합 시 성능**: 선형 확장
- **메모리**: Hazard Pointer 오버헤드

### Michael-Scott Queue
- **Enqueue**: O(1) amortized
- **Dequeue**: O(1) amortized
- **경합 시 성능**: 매우 우수
- **특징**: Helping으로 진전 보장

### Split-Ordered Hash Map
- **Insert/Find**: O(1) 평균
- **Resize**: Lock-Free, 점진적
- **확장성**: 거의 선형

---

## 🎓 학습 포인트

### 1. 단계적 학습
1. **기초**: CAS 연산 이해 ([05-lock-free-programming](../05-lock-free-programming))
2. **자료구조**: Treiber Stack부터 시작
3. **메모리 회수**: Hazard Pointer 이해
4. **고급**: Michael-Scott Queue, Hash Map

### 2. 소스 코드 읽기
```bash
# 클론
git clone https://github.com/khizmax/libcds.git
cd libcds

# 빌드
mkdir build && cd build
cmake ..
make -j

# 테스트 실행
make test
```

### 3. 디버깅
```bash
# ThreadSanitizer로 실행
g++ -fsanitize=thread -g my_test.cpp -lcds
./a.out

# Valgrind Helgrind
valgrind --tool=helgrind ./a.out
```

### 4. 벤치마크
```bash
# 내장 벤치마크 실행
cd build/test/stress
./stress-queue --help
./stress-queue --seconds=60 --producers=4 --consumers=4
```

---

## ⚠️ 주의사항

### 1. 올바른 사용
```cpp
// ❌ 잘못된 사용
cds::container::TreiberStack<cds::gc::HP, int> stack;
stack.push(42);  // Error: GC 초기화 안됨

// ✅ 올바른 사용
cds::Initialize();
{
    cds::gc::HP hpGC;
    cds::gc::HP::thread_gc myThreadGC;

    cds::container::TreiberStack<cds::gc::HP, int> stack;
    stack.push(42);  // OK
}
cds::Terminate();
```

### 2. 스레드별 GC 등록
```cpp
void worker_thread()
{
    // 각 스레드는 반드시 GC에 등록
    cds::gc::HP::thread_gc myThreadGC;

    // 이제 libcds 자료구조 사용 가능
}
```

### 3. 메모리 관리
```cpp
// libcds는 자동으로 노드 메모리 관리
// 사용자는 값(value)만 신경쓰면 됨

cds::container::TreiberStack<cds::gc::HP, std::unique_ptr<int>> stack;
stack.push(std::make_unique<int>(42));  // OK, 자동 관리
```

---

## 🔗 관련 리소스

### 공식 문서
- GitHub: https://github.com/khizmax/libcds
- Documentation: http://libcds.sourceforge.net/doc/cds-api/index.html

### 학술 논문
- **Treiber Stack**: "Systems Programming: Coping with Parallelism" (1986)
- **Michael-Scott Queue**: "Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms" (1996)
- **Hazard Pointers**: "Hazard Pointers: Safe Memory Reclamation for Lock-Free Objects" (2004)
- **Split-Ordered Lists**: "Split-Ordered Lists: Lock-Free Extensible Hash Tables" (2003)

### 관련 라이브러리
- **Folly** (Facebook): 유사한 Lock-Free 자료구조
- **Boost.Lockfree**: Boost 라이브러리의 Lock-Free 컴포넌트
- **ConcurrencyKit**: C 언어 Lock-Free 라이브러리

---

## 📚 다음 단계

1. **libcds 빌드 및 실행**
   - 로컬에서 빌드하고 예제 실행
   - 테스트 코드 분석

2. **Treiber Stack 구현**
   - libcds 없이 직접 구현해보기
   - Hazard Pointer와 통합

3. **벤치마크 실험**
   - 다양한 시나리오에서 성능 측정
   - Mutex 기반과 비교

4. **다음 프로젝트**
   - [Facebook Folly](./02-folly.md) 분석
   - 실전 프로젝트에 적용

---

*이 문서는 학습 목적으로 작성되었습니다. libcds 라이선스와 사용 조건을 확인하시기 바랍니다.*
