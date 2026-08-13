🤖 Agentic AI — Java, Spring Boot & Distributed Systems Deep-Dive Notes

«A production-oriented engineering handbook covering Core Java, JVM internals, concurrency, Spring Boot, JPA/Hibernate, databases, distributed systems, Kafka, caching, system design, and resilient microservices.»

""Java" (https://img.shields.io/badge/Java-17%2B-orange?logo=openjdk)" (https://www.oracle.com/java/)
""Spring Boot" (https://img.shields.io/badge/Spring%20Boot-3.x-brightgreen?logo=springboot)" (https://spring.io/projects/spring-boot)
""Kafka" (https://img.shields.io/badge/Apache%20Kafka-Distributed%20Streaming-black?logo=apachekafka)" (https://kafka.apache.org/)
""Redis" (https://img.shields.io/badge/Redis-Caching-red?logo=redis)" (https://redis.io/)
""PostgreSQL" (https://img.shields.io/badge/PostgreSQL-Database-blue?logo=postgresql)" (https://www.postgresql.org/)
""Docker" (https://img.shields.io/badge/Docker-Containers-blue?logo=docker)" (https://www.docker.com/)

---

📚 Table of Contents

- "About" (#-about)
- "Learning Philosophy" (#-learning-philosophy)
- "Knowledge Map" (#-knowledge-map)
- "Section I — Core Java & JVM" (#-section-i--core-java--jvm-internals)
  - "JVM Architecture" (#chapter-1-jvm-architecture--memory-model)
  - "Garbage Collection" (#chapter-2-garbage-collection--profiling)
  - "Collections" (#chapter-3-collections--data-structures)
  - "Concurrency" (#chapter-4-java-concurrency--virtual-threads)
- "Section II — Spring Boot" (#-section-ii--spring-boot-framework-architecture)
  - "IoC & Bean Lifecycle" (#chapter-5-ioc-container--bean-lifecycle)
  - "AOP & Proxies" (#chapter-6-spring-aop--proxies)
  - "Transactions" (#chapter-7-transaction-management--isolation)
- "Section III — Persistence" (#-section-iii--data-persistence--database-internals)
  - "JPA & Hibernate" (#chapter-8-jpa-mechanics--entity-states)
  - "Database Indexing" (#chapter-9-database-tuning--indexing)
- "Section IV — Distributed Systems" (#-section-iv--system-design--distributed-architectures)
  - "Scalability" (#chapter-10-scalability--load-balancing)
  - "Caching" (#chapter-11-distributed-caching-architecture)
  - "Kafka" (#chapter-12-asynchronous-messaging--kafka)
- "Section V — FAANG Interview Playbook" (#-section-v--faang-interview-playbook--code-blueprints)
  - "System Design" (#chapter-13-system-design-blueprints)
  - "Resilient Services" (#chapter-14-enterprise-resilient-code-blueprint)
- "Interview Questions" (#-interview-question-bank)
- "Production Checklist" (#-production-engineering-checklist)
- "Recommended Study Order" (#-recommended-study-order)

---

🧠 About

Agentic AI Notes is a deep-dive engineering knowledge base designed around one objective:

«Understand what happens underneath the abstraction.»

Instead of treating Java, Spring Boot, databases, Kafka, Redis, and distributed systems as isolated technologies, these notes connect their runtime behavior, memory model, concurrency model, failure modes, performance characteristics, and production architecture.

The notes focus particularly on questions such as:

- What actually happens when a Java class is loaded?
- Where does an object live in memory?
- How does the JVM identify an object?
- Why does "HashMap" use "(n - 1) & hash"?
- Why does "ConcurrentHashMap" not simply use one global lock?
- What exactly does "volatile" guarantee?
- Why can a "@Transactional" method fail during self-invocation?
- How does Spring resolve circular dependencies?
- What happens during Hibernate dirty checking?
- Why does an N+1 query problem occur?
- Why does a B+Tree make database lookups fast?
- Why does consistent hashing reduce cache movement?
- How does Kafka achieve high throughput?
- Why is the Transactional Outbox safer than directly publishing events?
- How can Redis implement an atomic distributed rate limiter?
- How does a Snowflake-style ID generator avoid database coordination?
- How should a production service behave when a downstream dependency fails?

---

🎯 Learning Philosophy

These notes follow a layered approach:

                 ┌─────────────────────────────┐
                 │     System Architecture     │
                 └──────────────┬──────────────┘
                                │
                 ┌──────────────▼──────────────┐
                 │   Distributed Architecture  │
                 └──────────────┬──────────────┘
                                │
                 ┌──────────────▼──────────────┐
                 │ Spring Boot / Microservices │
                 └──────────────┬──────────────┘
                                │
                 ┌──────────────▼──────────────┐
                 │       Persistence Layer     │
                 └──────────────┬──────────────┘
                                │
                 ┌──────────────▼──────────────┐
                 │    Core Java / Concurrency  │
                 └──────────────┬──────────────┘
                                │
                 ┌──────────────▼──────────────┐
                 │         JVM / OS / CPU      │
                 └─────────────────────────────┘

The goal is not merely to memorize APIs.

The goal is to understand the chain:

Application Code
      ↓
Framework
      ↓
JVM
      ↓
Operating System
      ↓
CPU / Memory

---

🗺️ Knowledge Map

Domain| Topics
Java| JVM, ClassLoaders, Collections, Generics, Exceptions
JVM| Heap, Stack, Metaspace, Object Headers, TLAB
GC| G1, ZGC, GC Roots, Safepoints, Memory Leaks
Concurrency| JMM, volatile, CAS, locks, false sharing
Loom| Virtual Threads, Carrier Threads, Pinning
Spring| IoC, DI, Bean Lifecycle, AOP, Proxies
Transactions| Isolation, Propagation, Rollback
JPA| Entity States, Persistence Context, Dirty Checking
Database| B+Trees, Indexes, Query Optimization, N+1
Distributed Systems| Consistent Hashing, Scalability, Fault Tolerance
Redis| Caching, Rate Limiting, Distributed Coordination
Kafka| Partitions, Ordering, Replication, Zero-Copy
Reliability| Retry, Circuit Breaker, Backoff
System Design| Rate Limiter, ID Generator, Outbox Pattern

---

☕ SECTION I — CORE JAVA & JVM INTERNALS

Chapter 1 — JVM Architecture & Memory Model

1.1 ClassLoader Subsystem

The JVM does not load every class into memory at startup.

Classes are loaded dynamically when required.

                Bootstrap ClassLoader
                         ▲
                         │
                Platform ClassLoader
                         ▲
                         │
              Application ClassLoader
                         ▲
                         │
                Custom ClassLoader

Delegation Model

When a class is requested:

Custom ClassLoader
       ↓
Parent
       ↓
Platform
       ↓
Bootstrap
       ↓
If not found
       ↓
Child attempts loading

This prevents application code from replacing core Java classes such as "java.lang.String".

Class Identity

A Java class is identified conceptually by:

Fully Qualified Class Name
+
ClassLoader Instance

Therefore:

com.example.User + LoaderA

and

com.example.User + LoaderB

are different runtime classes.

This is particularly important in:

- Application servers
- Plugin architectures
- Hot reloading
- OSGi-style systems
- Framework isolation

Linking

Class loading is followed by linking:

Loading
   ↓
Verification
   ↓
Preparation
   ↓
Resolution

Verification

The JVM verifies bytecode correctness and safety.

Examples include:

- Valid class-file structure
- Valid bytecode instructions
- Operand-stack consistency
- Access-control rules
- Type safety

The class-file magic number is:

0xCAFEBABE

Preparation

Memory is allocated for class-level structures and static fields receive their default values.

For example:

static int count = 10;

During preparation, the field initially has:

count = 0

The explicit assignment to "10" occurs during initialization.

Resolution

Symbolic references can be resolved to runtime references.

Initialization

Initialization executes:

static {
    // initialization logic
}

static int x = calculate();

The JVM guarantees class initialization is synchronized so that initialization occurs safely with respect to concurrent threads.

---

1.2 JVM Memory Areas

A simplified runtime model:

                JVM Process
                     │
       ┌─────────────┴─────────────┐
       │                           │
     Heap                    Native Memory
       │                           │
  ┌────┴────┐                ┌─────┴─────┐
  │ Young   │                │ Metaspace │
  │  Gen    │                │ Code Cache│
  └────┬────┘                └───────────┘
       │
 ┌─────┼─────┐
 │     │     │
Eden   S0    S1

        Old Generation

Heap

Shared among Java threads.

Typically contains Java objects.

Thread Stack

Each Java thread has its own stack.

A stack frame can contain:

- Local variables
- Operand stack
- Return information
- References needed by runtime execution

Example:

int add(int a, int b) {
    return a + b;
}

Conceptually:

Stack Frame
┌──────────────────────┐
│ Local Variable Table │
│ a                    │
│ b                    │
├──────────────────────┤
│ Operand Stack        │
│ a + b                │
└──────────────────────┘

TLAB

A Thread-Local Allocation Buffer is a small allocation area assigned to a thread.

Without TLAB:

Thread A ──┐
Thread B ──┼──► Shared Allocation
Thread C ──┘

With TLAB:

Thread A → TLAB A
Thread B → TLAB B
Thread C → TLAB C

This reduces synchronization during frequent object allocation.

---

1.3 Object Header

A simplified HotSpot object layout:

┌──────────────────────────────┐
│          Mark Word           │
├──────────────────────────────┤
│        Klass Pointer         │
├──────────────────────────────┤
│       Instance Fields        │
├──────────────────────────────┤
│       Alignment Padding      │
└──────────────────────────────┘

The object header can contain information related to:

- Identity hash code
- GC age
- Lock state
- Class metadata reference

Compressed OOPs

On supported 64-bit JVM configurations, references can be compressed to reduce object memory overhead.

Conceptually:

32-bit compressed reference
          ↓
decoded address
          ↓
64-bit heap address

With 8-byte alignment, compressed references can address a large heap range efficiently.

«Important: The exact compressed-oops heap limit depends on JVM configuration and object alignment; do not treat 32 GB as an absolute universal cutoff.»

---

♻️ Chapter 2 — Garbage Collection & Profiling

2.1 GC Roots

Garbage collection begins from objects considered reachable from GC roots.

Typical roots include:

GC Roots
 ├── Live Thread Stacks
 ├── Static References
 ├── JNI References
 ├── Active Monitors
 └── Runtime Structures

An object is collectible when no path exists from a GC root to that object.

GC Root
   │
   ▼
 Object A
   │
   ▼
 Object B

Object B → Reachable

But:

GC Root

Object C

(no path)

Object C → Eligible for collection

---

2.2 Java Reference Types

Strong Reference

User user = new User();

As long as "user" remains strongly reachable, normal GC does not reclaim it.

Soft Reference

Useful historically for memory-sensitive caches, although modern caching systems generally use explicit eviction policies.

Weak Reference

Weakly reachable objects can be reclaimed during GC.

Example:

WeakHashMap<Key, Value>

Phantom Reference

Used with reference queues for post-mortem cleanup patterns.

Modern Java resource-management mechanisms should generally prefer structured resource handling and "Cleaner" only when appropriate.

---

2.3 Safepoints

A safepoint is a JVM execution state where the JVM can safely perform certain global operations.

Examples:

- Garbage collection coordination
- Deoptimization
- Thread inspection

A long-running loop can matter when threads fail to reach safepoints promptly.

This is why JVM observability should include:

Application latency
+
GC pauses
+
Safepoint synchronization

---

🚀 Chapter 3 — Garbage Collectors

G1 GC

G1 divides the heap into regions.

┌────┬────┬────┬────┬────┬────┐
│ E  │ E  │ S  │ O  │ O  │ E  │
├────┼────┼────┼────┼────┼────┤
│ O  │ E  │ S  │ O  │ E  │ O  │
└────┴────┴────┴────┴────┴────┘

The JVM can select regions containing large amounts of reclaimable garbage.

Important concepts:

- Regions
- Remembered Sets
- Card tables
- Write barriers
- Concurrent marking
- Mixed collections
- Pause-time goals

---

ZGC

ZGC is designed for very large heaps and low pause times.

Important concepts include:

- Concurrent relocation
- Load barriers
- Colored pointers / object metadata
- Relocation sets
- Concurrent phases

The key architectural idea is:

«Move much of the expensive GC work concurrently with application execution.»

Do not interpret “sub-millisecond pauses” as a guaranteed application-level latency guarantee; actual latency depends on workload, hardware, JVM configuration, and application behavior.

---

🔍 Chapter 4 — Memory Leak Diagnostics

A Java memory leak usually means:

«Objects are no longer logically needed but remain strongly reachable.»

Common sources:

ThreadLocal
   ↓
Thread Pool
   ↓
Long-lived Worker Thread
   ↓
Value remains reachable

Other examples:

- Static collections
- Unremoved listeners
- Caches without eviction
- Incorrect lifecycle management
- ClassLoader retention

MAT

Eclipse Memory Analyzer can help investigate:

- Shallow Heap
- Retained Heap
- Dominator Tree
- GC Root paths

Shallow vs Retained Heap

Shallow Heap
= memory occupied by the object itself

Retained Heap
= memory that becomes collectible if this object
  becomes unreachable

For leak analysis, Retained Heap + GC Root path is often more informative than object size alone.

---

🧩 Chapter 5 — Collections & Data Structures

HashMap Internal Structure

Conceptually:

HashMap
   │
   ▼
Bucket Array
 ┌────┬────┬────┬────┐
 │    │    │    │    │
 ▼    ▼    ▼    ▼
Node Node Tree Node

Bucket Index

For power-of-two table capacity:

index = (n - 1) & hash

This is efficient because bitwise AND replaces a modulo operation.

Hash Spreading

Java's HashMap mixes high bits into lower bits:

h ^ (h >>> 16)

This helps distribute keys more evenly when the table index primarily uses lower bits.

---

Treeification

When a bucket becomes heavily populated, HashMap can convert a long collision chain into a Red-Black Tree.

Important thresholds include:

TREEIFY_THRESHOLD = 8
MIN_TREEIFY_CAPACITY = 64
UNTREEIFY_THRESHOLD = 6

Treeification is therefore not simply:

"If bucket size >= 8, always treeify."

The table capacity also matters.

---

🔒 Chapter 6 — ConcurrentHashMap

Java's modern "ConcurrentHashMap" avoids a single global lock.

Conceptually:

Bucket 0 → independent synchronization
Bucket 1 → independent synchronization
Bucket 2 → independent synchronization
Bucket 3 → independent synchronization

An empty bucket can often be initialized using CAS.

Conceptually:

CAS(null → new Node)

For a populated bin, synchronization is localized to that bin.

During resizing, special forwarding nodes coordinate migration.

This provides significantly greater concurrency than:

synchronized Map

---

🧵 Chapter 7 — Java Concurrency & Virtual Threads

Java Memory Model

The Java Memory Model defines how threads interact with shared memory and what guarantees exist around:

- Visibility
- Ordering
- Atomicity
- Happens-before relationships

Volatile

"volatile" provides:

Visibility
+
Ordering guarantees

It does not make arbitrary compound operations atomic.

For example:

volatile int count;

count++;

is still conceptually:

read
+
add
+
write

Therefore multiple threads can still lose updates.

Use:

AtomicInteger

or synchronization when atomic read-modify-write semantics are required.

---

False Sharing

Consider:

Cache Line
┌──────────────────────────────────────────┐
│ Thread A variable │ Thread B variable   │
└──────────────────────────────────────────┘

Even though the variables are logically independent, CPU cache coherence can cause unnecessary invalidations.

This is known as false sharing.

"@Contended" can be used in appropriate JVM configurations to separate frequently modified fields.

---

🧵 Virtual Threads — Java 21+

Virtual threads implement lightweight JVM-managed threads.

Millions of Virtual Threads
          │
          ▼
Small number of Carrier Threads
          │
          ▼
Operating System Threads

They are particularly useful for applications with large numbers of blocking I/O operations.

Example:

Request
   ↓
Virtual Thread
   ↓
HTTP call
   ↓
Virtual Thread waits
   ↓
Carrier Thread becomes available

Pinning

A virtual thread can become pinned to its carrier in situations such as certain synchronized/native execution paths.

Modern Java applications should therefore profile blocking behavior rather than assuming every blocking operation has identical cost.

---

🌱 SECTION II — SPRING BOOT FRAMEWORK ARCHITECTURE

Chapter 8 — IoC Container & Bean Lifecycle

Spring's IoC container manages:

Bean Creation
      ↓
Dependency Injection
      ↓
Initialization
      ↓
Runtime Usage
      ↓
Destruction

Simplified lifecycle:

Instantiation
      ↓
Dependency Injection
      ↓
Aware callbacks
      ↓
BeanPostProcessor Before
      ↓
@PostConstruct
      ↓
afterPropertiesSet()
      ↓
Custom init-method
      ↓
BeanPostProcessor After
      ↓
Ready
      ↓
@PreDestroy
      ↓
destroy()

---

Three-Level Singleton Cache

Spring can resolve certain circular dependencies involving singleton beans and setter/field injection.

Level 1
singletonObjects
        ↓
Fully initialized singleton

Level 2
earlySingletonObjects
        ↓
Early singleton reference

Level 3
singletonFactories
        ↓
Factory capable of exposing early references

Important Limitation

Constructor cycles cannot generally be resolved through early exposure because the object cannot be exposed before its constructor has completed.

Prefer:

A → B

rather than:

A ↔ B

If a cycle is unavoidable, "@Lazy" can sometimes break initialization timing, but architectural refactoring is usually preferable.

---

🎯 Chapter 9 — Spring AOP & Proxies

Spring AOP works primarily through proxies.

Client
  │
  ▼
Proxy
  │
  ├── Transaction
  ├── Security
  ├── Logging
  └── Target Method

Common proxy mechanisms include:

JDK Dynamic Proxy

Works through interfaces.

Interface
    ▲
    │
 Proxy
    │
 Target

CGLIB / Subclass-Based Proxying

The proxy subclasses the target class.

The exact default proxy mechanism can depend on Spring configuration/version; do not reduce the rule to “Boot 2 = CGLIB, Boot 3 = CGLIB” without checking the actual proxy configuration.

---

Self-Invocation Trap

@Service
public class PaymentService {

    public void process() {
        this.save();
    }

    @Transactional
    public void save() {
        // database work
    }
}

The call:

this.save();

does not pass through the Spring proxy.

Therefore the transactional interceptor may not execute.

Better Design

PaymentService
      │
      ▼
PaymentRepository

Or move the transactional boundary to another Spring-managed component.

---

💳 Chapter 10 — Transactions

Isolation Levels

Transaction isolation controls what one transaction can observe from concurrent transactions.

Isolation| Dirty Read| Non-Repeatable Read| Phantom Read
READ UNCOMMITTED| Possible| Possible| Possible
READ COMMITTED| Prevented| Possible| Possible
REPEATABLE READ| Prevented| Prevented| DB-dependent
SERIALIZABLE| Prevented| Prevented| Prevented

«Exact behavior is database-specific. For example, PostgreSQL's MVCC behavior differs from MySQL/InnoDB behavior for some phenomena.»

---

Propagation

REQUIRED

Existing transaction?
      │
 ┌────┴────┐
Yes       No
 │         │
Join     Create

REQUIRES_NEW

Outer Transaction
       │
     suspend
       ↓
New Transaction
       │
     commit
       ↓
Resume Outer

NESTED

Uses database savepoint semantics when supported.

Transaction
    │
    ├── Operation A
    │
    ├── SAVEPOINT
    │
    ├── Operation B
    │
    └── Rollback to SAVEPOINT

---

🗄️ SECTION III — DATA PERSISTENCE & DATABASE INTERNALS

Chapter 11 — JPA / Hibernate

Entity Lifecycle

Transient
   ↓ persist()
Persistent / Managed
   ↓
Detached
   ↓
Removed

Transient

User user = new User();

Not managed by the persistence context.

Persistent

EntityManager
      ↓
Persistence Context
      ↓
Managed Entity

Detached

The entity was previously managed but is no longer associated with the current persistence context.

Removed

The entity is scheduled for deletion.

---

🔬 Dirty Checking

Hibernate maintains state information for managed entities.

Example:

@Transactional
public void updateUser(Long id) {

    User user = repository.findById(id).orElseThrow();

    user.setName("Kunal");
}

No explicit:

repository.save(user);

is necessarily required for the update.

At flush time Hibernate detects:

Original State
name = "Old"

Current State
name = "Kunal"

        ↓

Difference detected

        ↓

UPDATE users
SET name = 'Kunal'
WHERE id = ?

This is dirty checking.

---

🚨 Chapter 12 — N+1 Query Problem

Suppose:

1 Author
+
N Books

A naive ORM mapping can result in:

SELECT * FROM author;

SELECT * FROM book WHERE author_id = 1;
SELECT * FROM book WHERE author_id = 2;
SELECT * FROM book WHERE author_id = 3;
...

Total:

1 + N queries

Solutions

JOIN FETCH

SELECT a
FROM Author a
JOIN FETCH a.books

EntityGraph

@EntityGraph(attributePaths = {"books"})

Batch Fetching

@BatchSize(size = 25)

The correct solution depends on:

- Cardinality
- Pagination requirements
- Result-set size
- Query shape
- Fetch frequency

---

🌳 Chapter 13 — Database Indexing

A B+Tree provides ordered traversal and efficient lookup.

              Root
          [50 | 100]
         /    |     \
       /      |       \
 [20|35]   [70|85]   [120|150]
    │
    ▼
 Leaf Nodes

Leaf nodes are linked for efficient range scans.

---

Clustered vs Secondary Index

Clustered

The index structure determines where table data is stored/organized in databases that support clustered storage.

Secondary

A secondary index generally stores:

Indexed Key
     +
Row Locator / Primary Key

The exact physical implementation depends on the database engine.

---

Leftmost Prefix Rule

For:

INDEX(A, B, C)

The index is naturally useful for predicates involving:

A
A + B
A + B + C

But:

B alone
C alone

do not generally provide the same direct left-prefix traversal.

Always validate actual index usage with:

EXPLAIN

or:

EXPLAIN ANALYZE

---

🌐 SECTION IV — SYSTEM DESIGN & DISTRIBUTED ARCHITECTURES

Chapter 14 — Consistent Hashing

Traditional routing:

hash(key) % N

If:

N = 4

becomes:

N = 5

many keys change destinations.

This causes large-scale cache movement.

---

Consistent Hash Ring

              Server A
                ●
          ┌────────────┐
       ●                  ●
    Key K1              Server B
       │                  │
       ●                  ●
    Server C            Key K2
          └────────────┘

Servers and keys are mapped onto the same logical hash space.

When a server is added:

Only a subset of keys
need reassignment

Virtual Nodes

One physical server can occupy many logical positions:

Server A
 ├── vnode 1
 ├── vnode 2
 ├── vnode 3
 ├── vnode 4
 └── ...

This improves distribution and reduces hotspots.

---

⚡ Chapter 15 — Distributed Caching

Cache-Aside

Application
     │
     ▼
   Cache
   /   \
Hit     Miss
 │       │
Return   DB
         │
         ▼
       Cache

Commonly used because the application controls what enters the cache.

---

Write-Through

Application
     ↓
 Cache
     ↓
 Database

Cache and database are updated synchronously according to the cache implementation.

---

Write-Behind

Application
     ↓
 Cache
     ↓
Async Queue
     ↓
 Database

Higher write throughput can come at the cost of increased durability/consistency complexity.

---

Cache Stampede

When a popular key expires:

1000 requests
      │
      ▼
Cache Miss
      │
      ▼
1000 DB Queries

Solutions include:

- Distributed locking
- Request coalescing
- Early refresh
- Jittered TTL
- Probabilistic expiration

---

Cache Penetration

Attacker repeatedly requests nonexistent IDs:

ID = -1
ID = 999999999
ID = abcxyz

If every request reaches the database:

Cache
  ↓
MISS
  ↓
Database
  ↓
NOT FOUND

Solutions:

Bloom Filter
+
Negative Cache
+
Input Validation

---

📨 Chapter 16 — Kafka

Kafka is a distributed append-only log.

Topic
 ├── Partition 0
 ├── Partition 1
 └── Partition 2

Each partition is ordered:

0 → 1 → 2 → 3 → 4 → 5

Ordering is guaranteed within a partition, not globally across an entire topic.

---

Sequential Writes

Kafka primarily appends records sequentially to partition logs.

Sequential I/O is efficient because storage devices and operating systems handle sequential access efficiently.

---

Page Cache

Kafka benefits heavily from the operating system's page cache.

Conceptually:

Kafka Log
   ↓
OS Page Cache
   ↓
Network

Kafka can also leverage zero-copy mechanisms such as "sendfile" where supported, reducing unnecessary user-space data copying.

---

Transactional Outbox

Problem:

DB Transaction
      +
Kafka Publish

These are two different systems.

If:

DB COMMIT ✓
Kafka SEND ✗

the system becomes inconsistent.

---

Outbox Solution

BEGIN TRANSACTION
       │
       ├── Update Business Data
       │
       └── Insert Outbox Event
       │
     COMMIT
       │
       ▼
 Outbox Processor / CDC
       │
       ▼
     Kafka

A CDC tool such as Debezium can capture database changes and publish events.

This avoids requiring a distributed 2PC transaction between the database and Kafka.

---

🏆 SECTION V — FAANG INTERVIEW PLAYBOOK & CODE BLUEPRINTS

Chapter 17 — Distributed Rate Limiter

A sliding-window rate limiter can use:

Redis
 +
Sorted Set
 +
Lua

Why Lua?

Because multiple Redis commands can be executed atomically inside the script.

Conceptual algorithm:

1. Remove expired requests
2. Count active requests
3. Compare against limit
4. Add new request if allowed
5. Set TTL

Request
   ↓
Redis Lua Script
   │
   ├── ZREMRANGEBYSCORE
   ├── ZCARD
   ├── ZADD
   └── EXPIRE
   ↓
ALLOW / REJECT

HTTP rejection:

429 Too Many Requests

---

🆔 Chapter 18 — Distributed ID Generator

Snowflake-style IDs avoid a central database sequence.

Typical structure:

┌───────┬───────────────────┬──────────────┬────────────┐
│Unused │ Timestamp         │ Worker ID    │ Sequence   │
│1 bit  │ 41 bits           │ 10 bits      │ 12 bits    │
└───────┴───────────────────┴──────────────┴────────────┘

The timestamp provides ordering.

The worker identifier provides node uniqueness.

The sequence handles multiple IDs generated within the same timestamp unit.

Critical Failure Mode

Clock rollback can produce duplicate or incorrectly ordered IDs.

Production implementations therefore need a clock strategy, such as:

- Rejecting backward movement
- Waiting
- Logical clock handling
- Worker fencing

---

🛡️ Chapter 19 — Resilient Microservices

A production service should assume downstream dependencies can fail.

Common failure modes:

Timeout
Connection Failure
5xx
Rate Limit
Network Partition
Slow Dependency
Partial Failure

A resilient service can combine:

Timeout
   ↓
Retry
   ↓
Circuit Breaker
   ↓
Fallback

---

Retry

Retry only failures that are safe to retry.

Bad:

POST payment
POST payment
POST payment

without idempotency protection.

Potential result:

Customer charged 3 times

Better:

Idempotency Key
+
Retry

---

Exponential Backoff

Instead of:

500ms
500ms
500ms

use:

500ms
1s
2s
4s
...

Add jitter in distributed systems to avoid synchronized retry storms.

---

Circuit Breaker

State machine:

              Failure Threshold
 CLOSED ─────────────────────────► OPEN
   ▲                                │
   │                                │ Wait
   │                                ▼
   │                           HALF_OPEN
   │                            /      \
   │                           /        \
Success                  Success       Failure
   │                        │             │
   └────────────────────────┘             │
                                          ▼
                                         OPEN

Why Circuit Breakers Matter

Without a circuit breaker:

Service A
   ↓
Service B ❌
   ↓
Timeout
   ↓
Retry
   ↓
Timeout
   ↓
Thread Pool Exhaustion
   ↓
Service A ❌

This can cause cascading failure.

---

🧱 Production Architecture

A resilient Spring Boot service might look like:

                    ┌───────────────┐
                    │   API Client  │
                    └───────┬───────┘
                            │
                            ▼
                   ┌────────────────┐
                   │ Spring Boot API│
                   └───────┬────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
           Redis        PostgreSQL     Kafka
              │            │            │
              │            │            ▼
              │            │       Event Consumers
              │            │
              │            ▼
              │       Outbox Table
              │
              ▼
       Rate Limiting /
       Distributed Cache

Reliability layer:

           Downstream API
                 │
                 ▼
        ┌─────────────────┐
        │ Timeout         │
        │ Retry           │
        │ Backoff         │
        │ Circuit Breaker │
        │ Fallback        │
        └─────────────────┘

---

🎯 Interview Question Bank

JVM

- What happens when a Java class is loaded?
- Explain ClassLoader delegation.
- What determines Java class identity?
- Heap vs Stack vs Metaspace?
- What is TLAB?
- What is an object header?
- What are compressed ordinary object pointers?
- What are GC roots?
- What is a safepoint?
- G1 vs ZGC?
- How would you investigate an "OutOfMemoryError"?

Collections

- Why does HashMap use power-of-two capacities?
- Explain "(n - 1) & hash".
- How does HashMap handle collisions?
- When does HashMap treeify?
- How does ConcurrentHashMap achieve concurrency?
- Why is LinkedList often slower than ArrayList despite O(1) insertion?

Concurrency

- Explain the Java Memory Model.
- What does volatile guarantee?
- Is "volatile int count; count++" thread-safe?
- What is CAS?
- What is false sharing?
- Explain happens-before.
- What are virtual threads?
- What is virtual-thread pinning?

Spring

- Explain Spring Bean lifecycle.
- What is IoC?
- How does dependency injection work?
- Explain the three-level singleton cache.
- Why can't constructor circular dependencies be resolved?
- JDK proxy vs subclass-based proxy?
- Why does self-invocation break "@Transactional"?
- Where should transaction boundaries exist?

Database

- Explain persistence context.
- What is dirty checking?
- What is N+1?
- JOIN FETCH vs EntityGraph?
- What is a B+Tree?
- Explain composite indexes.
- Explain the leftmost-prefix rule.
- How would you diagnose a slow SQL query?

Distributed Systems

- Why use consistent hashing?
- What are virtual nodes?
- Explain cache stampede.
- Explain cache penetration.
- Cache-aside vs write-through?
- How does Kafka achieve high throughput?
- What is partition ordering?
- What is the Transactional Outbox?
- Why avoid distributed 2PC when possible?

System Design

- Design a distributed rate limiter.
- Design a distributed ID generator.
- Design a URL shortener.
- Design a notification system.
- Design a payment system.
- Design a distributed cache.
- Design a job scheduler.
- Design an event-driven order-processing system.
- How would you make a microservice resilient?

---

🧪 Production Engineering Checklist

Before calling a service production-ready, evaluate:

JVM

- [ ] Heap sizing
- [ ] GC selection
- [ ] GC pause monitoring
- [ ] Thread count
- [ ] Heap dump capability
- [ ] CPU profiling
- [ ] Safepoint monitoring

Spring Boot

- [ ] Dependency injection boundaries
- [ ] Transaction boundaries
- [ ] Proxy behavior
- [ ] Circular dependency analysis
- [ ] Exception handling
- [ ] Configuration management

Database

- [ ] Query plans
- [ ] Index strategy
- [ ] Connection pool sizing
- [ ] N+1 detection
- [ ] Transaction isolation
- [ ] Slow query monitoring

Redis

- [ ] TTL policy
- [ ] Eviction policy
- [ ] Stampede protection
- [ ] Penetration protection
- [ ] Distributed locking strategy

Kafka

- [ ] Partition strategy
- [ ] Consumer groups
- [ ] Retry handling
- [ ] Dead-letter strategy
- [ ] Ordering requirements
- [ ] Idempotent consumers

Microservices

- [ ] Timeouts
- [ ] Retries
- [ ] Exponential backoff
- [ ] Jitter
- [ ] Circuit breakers
- [ ] Bulkheads
- [ ] Idempotency
- [ ] Observability

---

📈 Recommended Study Order

Do not study these topics randomly.

Follow the dependency chain:

                    ┌─────────────────┐
                    │ System Design   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Distributed     │
                    │ Systems         │
                    └────────┬────────┘
                             │
              ┌──────────────▼──────────────┐
              │ Kafka / Redis / DB / APIs   │
              └──────────────┬──────────────┘
                             │
                    ┌────────▼────────┐
                    │ Spring Boot     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ JPA / Hibernate │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Concurrency     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Collections     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Core Java       │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ JVM Internals   │
                    └─────────────────┘

Phase 1 — Java Foundations

Study:

Collections
Generics
Exceptions
Streams
Functional Interfaces
JVM Basics

Phase 2 — JVM Deep Dive

Study:

ClassLoader
Bytecode
Heap
Stack
Metaspace
Object Layout
GC
JIT
Safepoints

Phase 3 — Concurrency

Study:

JMM
volatile
synchronized
CAS
Atomic Classes
Locks
Executors
CompletableFuture
Virtual Threads

Phase 4 — Spring

Study:

IoC
DI
Bean Lifecycle
AOP
Proxy
Transactions
Spring MVC
Spring Security

Phase 5 — Persistence

Study:

JPA
Hibernate
Persistence Context
Dirty Checking
Lazy Loading
N+1
Indexes
Transactions
Query Optimization

Phase 6 — Distributed Systems

Study:

Caching
Consistent Hashing
Kafka
Replication
Partitioning
Idempotency
Outbox
Rate Limiting
Distributed IDs

Phase 7 — System Design

Finally combine everything:

Requirements
     ↓
Capacity Estimation
     ↓
API Design
     ↓
Data Model
     ↓
Caching
     ↓
Database
     ↓
Messaging
     ↓
Scaling
     ↓
Failure Handling
     ↓
Observability
     ↓
Security

---

🏁 Final Objective

The ultimate goal of these notes is to move from:

"I know Spring Boot."

to:

"I understand what Spring Boot is doing,
why it is doing it,
what the JVM is doing underneath it,
how the database behaves,
how the system behaves under concurrency,
and how the architecture behaves when components fail."

That is the level of understanding expected in strong backend, Java, distributed-systems, and system-design interviews.

---

⭐ Core Principle

«Don't memorize the framework. Understand the mechanism underneath the framework.»

Java
 ↓
JVM
 ↓
Concurrency
 ↓
Spring
 ↓
Persistence
 ↓
Database
 ↓
Redis / Kafka
 ↓
Distributed Systems
 ↓
System Design

This repository is intended to serve as a continuously evolving engineering handbook rather than a collection of isolated interview answers.

---

📌 Status

Deep-Dive Backend Engineering Notes

Topics currently covered:

- ✅ Core Java
- ✅ JVM Internals
- ✅ Garbage Collection
- ✅ Collections
- ✅ Java Concurrency
- ✅ Virtual Threads
- ✅ Spring IoC
- ✅ Spring AOP
- ✅ Transactions
- ✅ JPA / Hibernate
- ✅ Database Indexing
- ✅ Distributed Caching
- ✅ Consistent Hashing
- ✅ Kafka
- ✅ Transactional Outbox
- ✅ Distributed Rate Limiting
- ✅ Snowflake IDs
- ✅ Resilience Patterns
- 🚧 Advanced System Design
- 🚧 Observability
- 🚧 Security
- 🚧 Kubernetes
- 🚧 Cloud Architecture
- 🚧 Agentic AI Architecture

---

📖 License

These notes are intended for learning, interview preparation, experimentation, and engineering reference.
