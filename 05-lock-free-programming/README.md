# Lock-Free & Wait-Free 프로그래밍

## 📌 개요

Lock-Free와 Wait-Free 프로그래밍은 전통적인 락(lock) 기반 동기화를 사용하지 않고 동시성을 관리하는 고급 기법입니다. 이러한 기법은 높은 성능과 확장성을 요구하는 시스템에서 사용됩니다.

이 섹션에서는 다음을 다룹니다:
- Lock-Free와 Wait-Free 개념 및 차이점
- 대표적인 Lock-Free 자료구조 구현
- Memory Ordering과 Atomic 연산
- ABA 문제와 해결 방법
- 실전 적용 사례

---

## 🔄 동기화 접근법 비교

### Critical Section (Lock-Based)

**정의**: 한 번에 하나의 스레드만 실행할 수 있도록 락(Mutex, Semaphore 등)으로 보호된 코드 영역

**특징**:
- ✅ 구현이 비교적 간단하고 직관적
- ✅ 데이터 일관성 보장이 명확
- ⚠️ **Blocking**: 락을 기다리는 동안 스레드가 블로킹됨
- ⚠️ **Deadlock 위험**: 여러 락을 사용할 때 교착 상태 가능
- ⚠️ **Priority Inversion**: 우선순위 역전 문제 발생 가능
- ⚠️ **Context Switching 오버헤드**: 락 대기로 인한 성능 저하

**예시**:
```cpp
mutex.lock();
// Critical Section
shared_counter++;
mutex.unlock();
```

### Lock-Free

**정의**: 시스템 전체가 항상 진전(progress)을 보장하는 동기화 방식. 일부 스레드가 지연되더라도 **최소 하나의 스레드는 항상 진행**

**특징**:
- ✅ **Non-blocking**: 스레드가 블로킹되지 않음
- ✅ **No Deadlock**: 교착 상태가 발생하지 않음
- ✅ **높은 성능**: 락 경합이 없어 확장성 우수
- ✅ **Fault Tolerance**: 한 스레드가 멈춰도 다른 스레드는 진행
- ⚠️ **구현 복잡도 높음**: CAS, Memory Ordering 등 고급 개념 필요
- ⚠️ **ABA 문제**: 포인터 재사용으로 인한 문제 발생 가능
- ⚠️ **Starvation 가능**: 특정 스레드가 계속 실패할 수 있음

**예시**:
```cpp
// CAS (Compare-And-Swap) 사용
do {
    old_value = shared_counter.load();
    new_value = old_value + 1;
} while (!shared_counter.compare_exchange_weak(old_value, new_value));
```

### Wait-Free

**정의**: **모든 스레드**가 유한한 시간 내에 작업을 완료하는 것을 보장하는 가장 강력한 동기화 방식

**특징**:
- ✅ **최강의 진전 보장**: 모든 스레드가 O(n) 시간 내 완료
- ✅ **No Starvation**: 모든 스레드가 공정하게 진행
- ✅ **예측 가능한 레이턴시**: 실시간 시스템에 적합
- ✅ **완벽한 확장성**: 스레드 수에 관계없이 일정한 성능
- ⚠️ **구현 매우 어려움**: Lock-Free보다 훨씬 복잡
- ⚠️ **오버헤드 증가 가능**: 모든 스레드 보장을 위한 추가 비용
- ⚠️ **실용적인 구현이 적음**: 대부분 이론적으로만 존재

**예시**:
```cpp
// Fetch-and-Add는 Wait-Free 연산
shared_counter.fetch_add(1, memory_order_relaxed);
```

---

## 📊 세 가지 접근법 비교표

| 특성 | Critical Section | Lock-Free | Wait-Free |
|------|------------------|-----------|-----------|
| **진전 보장** | ❌ 없음 (블로킹) | ✅ 시스템 수준 | ✅ 모든 스레드 |
| **블로킹** | 블로킹 발생 | 비블로킹 | 비블로킹 |
| **Deadlock** | 발생 가능 | 발생 불가 | 발생 불가 |
| **Starvation** | 발생 가능 | 발생 가능 | 발생 불가 |
| **구현 난이도** | ⭐ 쉬움 | ⭐⭐⭐ 어려움 | ⭐⭐⭐⭐⭐ 매우 어려움 |
| **성능 (저경합)** | 우수 | 매우 우수 | 우수 |
| **성능 (고경합)** | 저하 | 우수 | 매우 우수 |
| **확장성** | 제한적 | 높음 | 매우 높음 |
| **예측 가능성** | 낮음 | 중간 | 매우 높음 |
| **실시간성** | ❌ 부적합 | △ 제한적 | ✅ 적합 |
| **메모리 순서** | 암묵적 | 명시적 필요 | 명시적 필요 |
| **사용 사례** | 일반적인 동기화 | 고성능 자료구조 | 실시간 시스템 |

---

## 🎯 언제 어떤 방법을 사용할까?

### Critical Section 사용 권장
- 일반적인 애플리케이션
- 구현의 명확성과 유지보수성이 중요한 경우
- 락 경합이 낮은 경우
- 복잡한 데이터 구조 보호

### Lock-Free 사용 권장
- 높은 동시성이 필요한 경우
- 확장성이 중요한 시스템 (예: 스레드 풀, 작업 큐)
- Deadlock을 피해야 하는 경우
- 락 경합이 높은 환경

### Wait-Free 사용 권장
- 실시간 시스템 (RTOS)
- 하드 레이턴시 제약이 있는 경우
- 모든 스레드의 공정성이 필수인 경우
- 극도로 높은 성능과 예측 가능성이 필요한 경우

---

## 📂 이 섹션의 문서들

| 문서 | 내용 |
|------|------|
| [01-cas-operation.md](./01-cas-operation.md) | Compare-And-Swap 원리와 사용법 |
| [02-memory-ordering.md](./02-memory-ordering.md) | Memory Ordering과 메모리 모델 |
| [03-lock-free-stack.md](./03-lock-free-stack.md) | Treiber Stack 구현 |
| [04-lock-free-queue.md](./04-lock-free-queue.md) | Michael-Scott Queue 구현 |
| [05-lock-free-counter.md](./05-lock-free-counter.md) | Lock-Free Counter 구현 |
| [06-wait-free-programming.md](./06-wait-free-programming.md) | Wait-Free 프로그래밍 개념과 구현 |
| [07-aba-problem.md](./07-aba-problem.md) | ABA 문제와 해결책 |
| [08-hazard-pointers.md](./08-hazard-pointers.md) | 안전한 메모리 회수 기법 |

---

## 🔬 핵심 개념

### 1. Atomic 연산
모든 Lock-Free/Wait-Free 알고리즘의 기초가 되는 하드웨어 지원 연산:
- **CAS (Compare-And-Swap)**: 값 비교 후 교체
- **Fetch-And-Add**: 원자적 증가
- **Exchange**: 원자적 교환

### 2. Memory Ordering
CPU와 컴파일러의 명령어 재배치를 제어:
- **Sequential Consistency**: 가장 강한 보장
- **Acquire-Release**: 효율적인 동기화
- **Relaxed**: 최소한의 보장

### 3. Progress Guarantees
동시성 알고리즘의 진전 보장 수준:
1. **Blocking**: 진전 보장 없음
2. **Obstruction-Free**: 홀로 실행되면 진전
3. **Lock-Free**: 시스템 전체가 진전
4. **Wait-Free**: 모든 스레드가 진전

---

## 💡 주의사항

1. **잘못 사용하면 더 느림**: Lock-Free가 항상 빠른 것은 아닙니다. 경합이 낮으면 오히려 락이 더 빠를 수 있습니다.

2. **메모리 모델 이해 필수**: 잘못된 Memory Ordering은 미묘한 버그를 만듭니다.

3. **테스트 어려움**: Race Condition이 간헐적으로 발생하여 디버깅이 어렵습니다.

4. **플랫폼 의존성**: CPU 아키텍처마다 동작이 다를 수 있습니다.

---

## 📚 다음 단계

Lock-Free와 Wait-Free 프로그래밍을 학습하는 권장 순서:

1. [CAS 연산](./01-cas-operation.md) - 기초 원자 연산 이해
2. [Memory Ordering](./02-memory-ordering.md) - 메모리 모델 학습
3. [Lock-Free Stack](./03-lock-free-stack.md) - 가장 간단한 자료구조
4. [Lock-Free Queue](./04-lock-free-queue.md) - 실용적인 자료구조
5. [Wait-Free Programming](./06-wait-free-programming.md) - 최강의 보장
6. [ABA Problem](./07-aba-problem.md) - 실전 문제 해결

---

*이 문서는 학습 목적으로 작성되었습니다. 실제 프로덕션 환경에서는 검증된 라이브러리(예: Folly, libcds)를 사용하는 것을 권장합니다.*
