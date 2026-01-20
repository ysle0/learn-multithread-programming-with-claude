# Concurrency Patterns

## Overview

Concurrency patterns are proven solutions to common problems encountered when designing concurrent and parallel systems. These patterns help developers write safer, more efficient, and more maintainable multithreaded code by providing well-tested approaches to coordination, communication, and resource management.

## Why Concurrency Patterns Matter

- **Reusability**: Proven solutions that work across different applications
- **Communication**: Common vocabulary for discussing concurrent designs
- **Efficiency**: Optimized approaches to common concurrency challenges
- **Safety**: Patterns that help avoid race conditions, deadlocks, and other concurrency bugs
- **Scalability**: Designs that scale effectively with increasing parallelism

## Pattern Categories

### 1. Communication Patterns
Patterns focused on how threads communicate and exchange data:
- **Producer-Consumer**: Decoupling producers of data from consumers
- **Pipeline**: Processing data through multiple sequential stages
- **Actor Model**: Message-passing between independent actors

### 2. Resource Management Patterns
Patterns for managing shared resources:
- **Reader-Writer**: Optimizing concurrent read/write access
- **Thread Pool**: Reusing threads to execute tasks efficiently

### 3. Coordination Patterns
Patterns for coordinating work across threads:
- **Future/Promise**: Representing eventual results of asynchronous operations
- **Fan-Out/Fan-In**: Distributing work and collecting results

## Patterns Covered

### 01. Producer-Consumer Pattern
```
[Producers] --> [Shared Queue] --> [Consumers]
```
**Use When**: You need to decouple data production from consumption, handle varying rates of production/consumption, or implement work queues.

**Key Concepts**: Bounded buffers, blocking operations, backpressure

### 02. Reader-Writer Pattern
```
Multiple Readers (concurrent)
     OR
Single Writer (exclusive)
```
**Use When**: Read operations vastly outnumber writes, or you need to optimize read throughput while ensuring write safety.

**Key Concepts**: Shared/exclusive locks, read bias vs write bias, fairness

### 03. Thread Pool Pattern
```
[Tasks] --> [Task Queue] --> [Worker Threads]
```
**Use When**: Creating threads is expensive, you have many short-lived tasks, or you need to limit concurrent execution.

**Key Concepts**: Work stealing, task scheduling, thread lifecycle management

### 04. Actor Model
```
[Actor A] <--messages--> [Actor B] <--messages--> [Actor C]
```
**Use When**: You want to eliminate shared state, need location transparency, or are building distributed systems.

**Key Concepts**: Message passing, actor isolation, mailboxes

### 05. Future/Promise Pattern
```
Promise (Writer) --> Shared State <-- Future (Reader)
```
**Use When**: You need to represent values computed asynchronously, chain operations, or handle async errors.

**Key Concepts**: Deferred computation, continuation passing, async/await

### 06. Pipeline Pattern
```
[Stage 1] --> [Stage 2] --> [Stage 3] --> [Output]
```
**Use When**: Processing can be divided into sequential stages, each stage can run concurrently, or you need stream processing.

**Key Concepts**: Stage parallelism, buffering between stages, backpressure

### 07. Fan-Out/Fan-In Pattern
```
           --> [Worker 1] --
[Input] --> --> [Worker 2] --> --> [Aggregator]
           --> [Worker 3] --
```
**Use When**: A task can be split into independent subtasks, you need to aggregate results from parallel operations.

**Key Concepts**: Work distribution, result aggregation, synchronization

## Pattern Selection Guide

### Choose Producer-Consumer When:
- Data production and consumption rates differ
- You need buffering between components
- Decoupling producers and consumers improves design

### Choose Reader-Writer When:
- Reads vastly outnumber writes (>90% reads)
- Read operations can safely occur concurrently
- Write operations need exclusive access

### Choose Thread Pool When:
- You have many short-duration tasks
- Thread creation overhead is significant
- You need to limit resource usage

### Choose Actor Model When:
- You can design around message passing
- Each component can be isolated
- You might need distribution/location transparency

### Choose Future/Promise When:
- Operations complete asynchronously
- You need to compose async operations
- Error handling must propagate through async calls

### Choose Pipeline When:
- Processing consists of distinct stages
- Stages can process different items simultaneously
- Data flows in one direction

### Choose Fan-Out/Fan-In When:
- Work can be partitioned into independent chunks
- Results need to be aggregated
- You want to exploit data parallelism

## Combining Patterns

Patterns can be combined for more complex scenarios:

- **Thread Pool + Producer-Consumer**: Workers in a thread pool consume tasks from a queue
- **Pipeline + Fan-Out/Fan-In**: Individual pipeline stages can fan-out for parallel processing
- **Actor Model + Future/Promise**: Actors return futures for asynchronous request-response
- **Thread Pool + Future/Promise**: Thread pool executes tasks that set promise values

## Common Anti-Patterns to Avoid

### 1. Over-Synchronization
Using more synchronization than necessary, leading to sequential execution.

### 2. Under-Synchronization
Insufficient synchronization leading to race conditions.

### 3. Lock Convoy
Threads queueing up on a lock, reducing parallelism.

### 4. Priority Inversion
High-priority threads waiting for low-priority threads.

### 5. Thread Exhaustion
Creating unbounded numbers of threads.

## Performance Considerations

### Granularity
- **Too Coarse**: Limited parallelism, underutilized cores
- **Too Fine**: Synchronization overhead dominates useful work
- **Just Right**: Balance between parallelism and overhead

### Contention
- **High Contention**: Many threads competing for same resource
- **Solutions**: Reduce critical section size, use lock-free structures, partition data

### Cache Effects
- **False Sharing**: Different threads accessing different data on same cache line
- **Solutions**: Padding, alignment, data structure design

## Testing Concurrent Patterns

### Approaches:
1. **Stress Testing**: Run with high thread counts and loads
2. **Race Detection**: Use tools like ThreadSanitizer
3. **Formal Verification**: Model checking for critical sections
4. **Performance Testing**: Measure scalability with increasing cores

### Common Issues:
- Deadlocks
- Livelocks
- Race conditions
- Memory ordering bugs
- Resource leaks

## Further Reading

### Books
- "Java Concurrency in Practice" by Brian Goetz (principles apply to C++)
- "Concurrency in C++" by Anthony Williams
- "The Art of Multiprocessor Programming" by Herlihy & Shavit

### Papers
- "Communicating Sequential Processes" by C.A.R. Hoare
- "Actors: A Model of Concurrent Computation" by Gul Agha

### Standards
- C++11/14/17/20 concurrency features
- ISO/IEC 14882 (C++ Standard)

## Pattern Implementation Notes

Each pattern in this section includes:

1. **Conceptual Overview**: What problem does it solve?
2. **Architecture Diagrams**: Visual representation using ASCII art
3. **C++ Implementation**: Complete working examples
4. **Variants**: Common variations and alternatives
5. **Performance Analysis**: When to use, scalability characteristics
6. **Common Pitfalls**: What to watch out for
7. **Real-World Examples**: Where this pattern is used in practice

## Next Steps

1. Start with **Producer-Consumer** - the most fundamental pattern
2. Progress through **Reader-Writer** and **Thread Pool** for resource management
3. Explore **Actor Model** for a different concurrency paradigm
4. Learn **Future/Promise** for async programming
5. Study **Pipeline** and **Fan-Out/Fan-In** for complex data processing

Each pattern builds on concepts from earlier sections (synchronization primitives, memory models, lock-free programming) and demonstrates how to combine these low-level tools into high-level designs.
