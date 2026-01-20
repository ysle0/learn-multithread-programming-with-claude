# Concurrency Problems

## Overview

Concurrent programming introduces a class of problems that don't exist in sequential programming. These problems arise from the non-deterministic interleaving of thread execution and shared resource access. Understanding these problems is crucial for writing correct and efficient multi-threaded programs.

## The Five Major Concurrency Problems

### 1. Race Condition
When multiple threads access shared data concurrently, and at least one modifies it, without proper synchronization.

**Key Characteristics:**
- Non-deterministic behavior
- Results depend on timing
- Difficult to reproduce

**Learn More:** [01-race-condition.md](./01-race-condition.md)

### 2. Deadlock
When two or more threads are permanently blocked, each waiting for resources held by others.

**Key Characteristics:**
- Complete standstill
- Circular dependency
- Requires intervention to resolve

**Learn More:** [02-deadlock.md](./02-deadlock.md)

### 3. Livelock
When threads continuously change state in response to each other without making progress.

**Key Characteristics:**
- Threads remain active but unproductive
- No forward progress
- Resource intensive

**Learn More:** [03-livelock.md](./03-livelock.md)

### 4. Starvation
When a thread is perpetually denied access to resources it needs.

**Key Characteristics:**
- Unfair scheduling
- Some threads make progress, others don't
- Can lead to performance degradation

**Learn More:** [04-starvation.md](./04-starvation.md)

### 5. Priority Inversion
When a high-priority thread is blocked waiting for a low-priority thread to release a resource.

**Key Characteristics:**
- Violates priority semantics
- Can cause critical failures
- Requires priority inheritance to fix

**Learn More:** [05-priority-inversion.md](./05-priority-inversion.md)

## Problem Comparison Matrix

```
┌─────────────────────┬──────────────┬────────────┬─────────────┬──────────────┐
│      Problem        │   Progress   │  Resource  │   Detection │  Severity    │
│                     │              │   Usage    │  Difficulty │              │
├─────────────────────┼──────────────┼────────────┼─────────────┼──────────────┤
│ Race Condition      │ Unpredictable│    Normal  │     High    │   Critical   │
│ Deadlock            │     None     │    Locked  │    Medium   │   Critical   │
│ Livelock            │     None     │     High   │    Medium   │     High     │
│ Starvation          │   Partial    │   Unfair   │      Low    │    Medium    │
│ Priority Inversion  │   Delayed    │    Locked  │    Medium   │     High     │
└─────────────────────┴──────────────┴────────────┴─────────────┴──────────────┘
```

## General Prevention Strategies

### 1. Minimize Shared State
- Prefer message passing over shared memory
- Use thread-local storage
- Design for immutability

### 2. Use Proper Synchronization
- Locks, mutexes, semaphores
- Atomic operations
- Memory barriers

### 3. Follow Best Practices
- Lock ordering conventions
- Timeout mechanisms
- Fair scheduling policies

### 4. Testing and Validation
- Stress testing
- Race detection tools (ThreadSanitizer, Helgrind)
- Formal verification methods

## Common Patterns That Lead to Problems

### Pattern 1: Check-Then-Act
```c
// WRONG: Race condition
if (resource->available) {
    // Another thread might grab it here!
    resource->use();
}

// CORRECT: Atomic check-and-act
lock(mutex);
if (resource->available) {
    resource->use();
}
unlock(mutex);
```

### Pattern 2: Multiple Lock Acquisition
```c
// WRONG: Potential deadlock
lock(mutex_a);
lock(mutex_b);  // Another thread might lock in reverse order

// CORRECT: Consistent lock ordering
lock(min(mutex_a, mutex_b));
lock(max(mutex_a, mutex_b));
```

### Pattern 3: Lock + Wait
```c
// WRONG: Can cause deadlock
lock(mutex);
wait_for_event();  // Holding lock while waiting

// CORRECT: Release lock before waiting
lock(mutex);
unlock(mutex);
wait_for_event();
```

## Tools for Detection

### Static Analysis
- **Clang Thread Safety Analysis**: Compile-time detection
- **Coverity**: Commercial static analyzer
- **Infer**: Facebook's static analyzer

### Dynamic Analysis
- **ThreadSanitizer (TSan)**: Race condition detector
- **Helgrind**: Valgrind's thread error detector
- **DRD**: Data race detector

### Profiling Tools
- **perf**: Linux performance profiler
- **VTune**: Intel's performance profiler
- **gprof**: GNU profiler

## Real-World Impact

### Famous Incidents

1. **Therac-25 (1985-1987)**
   - Race condition in radiation therapy machine
   - Result: Lethal radiation overdoses
   - Cause: Concurrent access to shared state

2. **Northeast Blackout (2003)**
   - Race condition in alarm system
   - Result: 50 million people without power
   - Cause: Unprotected shared data structure

3. **Mars Pathfinder (1997)**
   - Priority inversion problem
   - Result: System resets on Mars
   - Cause: Low-priority thread blocking high-priority thread

4. **Knight Capital (2012)**
   - Race condition in trading software
   - Result: $440 million loss in 45 minutes
   - Cause: Concurrent modification of order state

## Testing Strategies

### 1. Stress Testing
```bash
# Run with maximum threads
./program --threads=1000 --iterations=10000000

# Use CPU stress
stress --cpu 8 --timeout 60s & ./program
```

### 2. Race Detection
```bash
# Compile with ThreadSanitizer
gcc -fsanitize=thread -g program.c -o program

# Run with Helgrind
valgrind --tool=helgrind ./program
```

### 3. Deterministic Testing
```c
// Use barriers to force specific interleavings
pthread_barrier_t barrier;
pthread_barrier_init(&barrier, NULL, 2);

// Thread 1
critical_section_1();
pthread_barrier_wait(&barrier);  // Sync point

// Thread 2
pthread_barrier_wait(&barrier);  // Sync point
critical_section_2();
```

## Learning Path

1. **Start Here**: Understand race conditions thoroughly
2. **Build Foundation**: Learn about deadlocks and prevention
3. **Advanced Topics**: Study livelock and starvation
4. **Real-Time Systems**: Master priority inversion

## Exercises

Each section contains practical exercises. Work through them in order:

1. **Race Condition**: Implement a thread-safe counter
2. **Deadlock**: Fix the dining philosophers problem
3. **Livelock**: Resolve the hallway problem
4. **Starvation**: Implement fair reader-writer locks
5. **Priority Inversion**: Simulate priority inheritance

## Additional Resources

### Books
- "The Art of Multiprocessor Programming" by Herlihy & Shavit
- "Java Concurrency in Practice" by Goetz et al.
- "Programming with POSIX Threads" by Butenhof

### Papers
- "Dining Philosophers Problem" - Dijkstra (1965)
- "Monitors: An Operating System Structuring Concept" - Hoare (1974)
- "Priority Inheritance Protocols" - Sha, Rajkumar, Lehoczky (1990)

### Online Resources
- POSIX Threads Programming (LLNL Tutorial)
- MIT 6.826: Principles of Computer Systems
- CMU 15-410: Operating System Design

## Next Steps

After completing this section, you should:
1. Understand all five major concurrency problems
2. Be able to identify them in code
3. Know prevention and detection strategies
4. Have practical experience fixing these issues

Continue to: **04-synchronization-primitives/** to learn the tools for solving these problems.
