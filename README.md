# Agentic AI Notes

> Deep-Dive Engineering Notes for Java, JVM Internals, Spring Boot, JPA, Databases, Distributed Systems, Kafka, System Design, and Production-Grade Backend Architecture


A structured collection of advanced Java and backend engineering notes focused on understanding what happens under the hood, rather than memorizing APIs.

These notes are designed for:

FAANG / product-based company interviews

Java Backend / Full-Stack interviews

Spring Boot interviews

System Design interviews

Distributed Systems

JVM and performance debugging

Production-grade application development

Agentic AI / AI backend engineering



---

# Table of Contents

Section I — Core Java Mechanics & JVM Internals

Chapter 1 — JVM Architecture & Memory Model

Chapter 2 — Garbage Collection & Profiling

Chapter 3 — Collections & Data Structures

Chapter 4 — Java Concurrency & Virtual Threads


Section II — Spring Boot Framework Architecture

Chapter 5 — IoC Container & Bean Lifecycle

Chapter 6 — Spring AOP & Proxies

Chapter 7 — Transaction Management & Isolation


Section III — Data Persistence & Database Internals

Chapter 8 — JPA Mechanics & Entity States

Chapter 9 — Database Tuning & Indexing


Section IV — System Design & Distributed Architectures

Chapter 10 — Scalability & Load Balancing

Chapter 11 — Distributed Caching

Chapter 12 — Asynchronous Messaging & Kafka


Section V — FAANG Interview Playbook & Code Blueprints

Chapter 13 — System Design Blueprints

Chapter 14 — Enterprise Resilient Code Blueprint




---

# SECTION I — CORE JAVA MECHANICS & JVM INTERNALS


---

Chapter 1 — JVM Architecture & Memory Model

The Java Virtual Machine (JVM) is the runtime environment responsible for executing Java bytecode.

A simplified execution pipeline is:

Java Source Code
       │
       ▼
     javac
       │
       ▼
   Bytecode (.class)
       │
       ▼
   Class Loader
       │
       ▼
 Verification / Linking
       │
       ▼
 Runtime Data Areas
       │
       ▼
 Interpreter + JIT Compiler
       │
       ▼
 Native Machine Code
       │
       ▼
      CPU

The JVM provides:

Platform independence

Automatic memory management

Runtime class loading

Bytecode verification

JIT compilation

Garbage collection

Thread management

Security boundaries



---

1.1 ClassLoader Subsystem

The ClassLoader subsystem dynamically loads classes into the JVM.

Class Loading Pipeline

┌─────────────────────────┐
                 │    Bootstrap Loader     │
                 │      java.base          │
                 └────────────┬────────────┘
                              │
                              ▼
                 ┌─────────────────────────┐
                 │   Platform Loader      │
                 │   Java platform modules │
                 └────────────┬────────────┘
                              │
                              ▼
                 ┌─────────────────────────┐
                 │ Application ClassLoader │
                 │ classpath / application │
                 └────────────┬────────────┘
                              │
                              ▼
                 ┌─────────────────────────┐
                 │   Custom ClassLoader    │
                 │ plugins / frameworks    │
                 └─────────────────────────┘

Delegation Model

When a class is requested:

Application ClassLoader
          │
          ▼
  Ask Parent Loader
          │
          ▼
 Platform ClassLoader
          │
          ▼
  Ask Bootstrap Loader
          │
          ├── Found → return class
          │
          └── Not Found
                  │
                  ▼
       Child tries to load class

This is called the parent delegation model.

The purpose is to prevent application code from replacing core platform classes accidentally or maliciously.

For example, an application should not be able to provide its own:

java.lang.String

and have that class replace the JDK implementation.


---

Class Identity

A JVM class is identified by:

Fully Qualified Class Name
            +
      ClassLoader

Therefore:

com.example.User

loaded by:

Loader A

is considered a different runtime type from:

com.example.User

loaded by:

Loader B

This can lead to:

ClassCastException

even when the class names are identical.

This behavior is important in:

Application servers

Plugin systems

OSGi

Application reloaders

Java agents

Modular applications



---

Thread Context ClassLoader

The Thread Context ClassLoader (TCCL) allows framework code to load classes using the application's class-loading context.

Conceptually:

Framework
   │
   │ uses
   ▼
Thread.currentThread()
   │
   ▼
Context ClassLoader
   │
   ▼
Application Classes

This is useful when a framework needs to discover application-provided implementations.

Common examples include:

JDBC

Service Provider mechanisms

Application servers

Plugin frameworks



---

Linking

After loading a class, the JVM performs linking.

Class Loading
     │
     ▼
Verification
     │
     ▼
Preparation
     │
     ▼
Resolution
     │
     ▼
Initialization

Verification

The JVM verifies bytecode correctness.

Examples include checking:

Bytecode structure

Type safety

Operand stack consistency

Access control

Valid class-file format


A Java class file begins with the well-known magic number:

0xCAFEBABE


---

Preparation

The JVM allocates and initializes class/static storage to default values.

For example:

static int count = 100;
static String name = "Java";

Conceptually, preparation initially establishes default values:

count → 0
name  → null

The explicit assignments occur during initialization.


---

Resolution

Symbolic references from the class file can be resolved into references usable by the JVM.

For example:

Constant Pool
     │
     ▼
Symbolic Reference
     │
     ▼
Runtime Resolution

Resolution can occur eagerly or lazily depending on the JVM implementation and circumstances.


---

Class Initialization

Initialization executes:

Static field initializers

Static initialization blocks


Example:

class Config {

    static int port = 8080;

    static {
        System.out.println("Initializing...");
    }
}

Conceptually:

Class Loading
     ↓
Linking
     ↓
Initialization
     ↓
<clinit>
     ↓
Class Ready

The JVM ensures class initialization is properly synchronized so that initialization occurs safely when multiple threads encounter a class for the first time.


---

1.2 JVM Runtime Memory

A simplified JVM memory model:

┌────────────────────────────────────────────────────┐
│                      JVM                           │
│                                                    │
│  ┌──────────────────────────────────────────────┐  │
│  │                    HEAP                      │  │
│  │                                              │  │
│  │  Young Generation        Old Generation      │  │
│  │  ┌───────┬─────┬─────┐                      │  │
│  │  │ Eden  │ S0  │ S1  │                      │  │
│  │  └───────┴─────┴─────┘                      │  │
│  └──────────────────────────────────────────────┘  │
│                                                    │
│  ┌──────────────────────────────────────────────┐  │
│  │                  Metaspace                   │  │
│  │       Class Metadata / Runtime Metadata      │  │
│  └──────────────────────────────────────────────┘  │
│                                                    │
│  ┌──────────────────────────────────────────────┐  │
│  │              Thread Stacks                   │  │
│  │   Thread 1 │ Thread 2 │ Thread 3 │ ...      │  │
│  └──────────────────────────────────────────────┘  │
│                                                    │
│  PC Registers / Native Method Stacks / Code Cache  │
└────────────────────────────────────────────────────┘


---

Heap

The heap is the primary memory area for Java objects.

User user = new User();

The User object is allocated on the heap.

The heap is shared between Java threads.

Garbage collection primarily operates on heap objects.


---

Young Generation

Traditionally, generational collectors divide the young generation into:

Young Generation
       │
       ├── Eden
       ├── Survivor 0
       └── Survivor 1

Most newly allocated objects initially enter Eden.

Example:

new Object()
     ↓
   Eden
     ↓
Minor GC
     ↓
Survivor
     ↓
Repeated GC
     ↓
Old Generation

Objects that survive multiple collection cycles may eventually become candidates for promotion to the old generation.

> Exact generational behavior depends on the garbage collector. For example, modern ZGC uses a different architecture from the traditional young/old layout.




---

TLAB — Thread Local Allocation Buffer

A TLAB is a small allocation region associated with a thread.

Without a TLAB:

Thread A ─┐
Thread B ─┼──► Shared allocation region
Thread C ─┘

Threads could contend for allocation.

With TLABs:

Heap / Eden

┌────────────┬────────────┬────────────┐
│ TLAB A     │ TLAB B     │ TLAB C     │
│ Thread A   │ Thread B   │ Thread C   │
└────────────┴────────────┴────────────┘

A thread can often perform fast pointer-bump allocation inside its own TLAB.

This reduces allocation contention.


---

Thread Stack

Each Java thread has its own JVM stack.

A method invocation creates a stack frame.

Thread Stack

┌──────────────────────────┐
│ Frame: methodC()         │
│ Local Variables          │
│ Operand Stack            │
│ Return Information       │
├──────────────────────────┤
│ Frame: methodB()         │
├──────────────────────────┤
│ Frame: methodA()         │
└──────────────────────────┘

A frame can contain:

Local variables

Operand stack

Reference to runtime constant pool

Return information


Stack size can be influenced using:

-Xss


---

Metaspace

Java 8 removed the traditional PermGen memory model and introduced Metaspace.

Metaspace is native memory used for class-related metadata.

It can contain information such as:

Class metadata

Method metadata

Runtime structures

Class hierarchy information


Metaspace should not be confused with the Java heap.


---

1.3 Object Header & Compressed OOPs

A simplified HotSpot object layout can be represented as:

┌─────────────────────────────────────────────────────┐
│ Mark Word                                           │
├─────────────────────────────────────────────────────┤
│ Klass Pointer                                       │
├─────────────────────────────────────────────────────┤
│ Instance Fields                                     │
├─────────────────────────────────────────────────────┤
│ Alignment Padding                                   │
└─────────────────────────────────────────────────────┘

The exact layout depends on:

JVM implementation

Architecture

JVM flags

Object type

Compressed references



---

Mark Word

The Mark Word is used by HotSpot for object metadata such as:

Identity hash code

GC age information

Lock-related state


Lock representation is implementation-dependent and has evolved across JDK versions.

> Important: Older JVM documentation often describes biased locking. Modern JDKs no longer use biased locking in the same way, so interview answers should be version-aware.




---

Compressed OOPs

OOP means:

Ordinary Object Pointer

On 64-bit JVMs, object references can be compressed to reduce memory usage.

Conceptually:

64-bit reference
       ↓
Compressed reference
       ↓
Decoded using heap addressing scheme

With suitable alignment, compressed references can address a large heap while using fewer bits per reference.

A common simplified calculation is:

2^32 × 8 bytes
≈ 32 GB

However, the exact supported heap range depends on the JVM's compressed-reference mode and configuration.


---

Chapter 2 — Garbage Collection & Profiling

Garbage Collection automatically reclaims objects that are no longer reachable.

The fundamental question is:

> Can this object still be reached from a GC Root?




---

2.1 GC Roots

Typical GC roots include:

Local variables in active stack frames

Live thread references

Static references

JNI references

JVM-internal references


Conceptually:

GC Root
   │
   ▼
Object A
   │
   ▼
Object B
   │
   ▼
Object C

All these objects are reachable.

If:

GC Root
   X
   │
   └── Object A

Object B ── Object C

and Object B is not reachable from any root, it can become eligible for collection.


---

Java Reference Types

Strong Reference

Object obj = new Object();

As long as obj remains strongly reachable, the object is not eligible for ordinary GC reclamation.


---

Soft Reference

Soft references are designed for memory-sensitive caching.

They may be cleared under memory pressure.

They should not be treated as a guaranteed cache policy.


---

Weak Reference

Weakly reachable objects can be reclaimed when the garbage collector determines that no strong or soft reachability keeps them alive.

Example:

WeakHashMap

is a common use case.


---

Phantom Reference

Phantom references are used with ReferenceQueue for post-mortem notification/cleanup coordination.

They are useful when working with resources outside normal Java memory management.

Modern Java applications should generally prefer safer resource-management mechanisms such as:

try-with-resources

and appropriate cleaners where necessary.


---

Safepoints

A safepoint is a point at which the JVM can safely perform certain global operations while threads are brought to a state where their references and execution state can be inspected.

Examples include operations related to:

Garbage collection

Deoptimization

Some JVM maintenance operations


Conceptually:

Thread A ───────●──────────────
Thread B ─────────────●───────
Thread C ─────●────────────────
               │
               ▼
           Safepoint

A thread that takes a long time to reach a safepoint can contribute to:

Safepoint Synchronization Delay

Modern HotSpot has mechanisms for safepoint polling and counted-loop safepoints.


---

2.2 Garbage Collectors

Different collectors optimize for different goals.

Garbage Collectors
                           │
          ┌────────────────┼────────────────┐
          │                │                │
       Throughput        Balanced         Latency
          │                │                │
       Parallel           G1              ZGC
       Serial


---

G1 Garbage Collector

G1 divides the heap into many regions rather than relying exclusively on one contiguous young/old layout.

Heap

┌────┬────┬────┬────┬────┬────┬────┬────┐
│ E  │ O  │ E  │ S  │ O  │ E  │ O  │ E  │
├────┼────┼────┼────┼────┼────┼────┼────┤
│ O  │ E  │ E  │ O  │ S  │ E  │ O  │ E  │
└────┴────┴────┴────┴────┴────┴────┴────┘

E = Eden
S = Survivor
O = Old

G1 uses:

Regions

Remembered Sets

Write barriers

Concurrent marking

Mixed collections


G1 attempts to meet a pause-time goal specified through:

-XX:MaxGCPauseMillis

This is a target, not a hard guarantee.


---

Remembered Sets

Suppose:

Region A ─────► Region B

If Region B is being collected, the JVM needs to know whether objects in other regions reference objects inside B.

Remembered Sets help track these cross-region references.


---

ZGC

ZGC is designed for extremely low pause times, even with very large heaps.

Its architecture relies heavily on concurrent work.

Conceptually:

Application Threads
       │
       │ continue running
       ▼
┌───────────────────┐
│ Concurrent GC     │
│ Mark / Relocate   │
└───────────────────┘

ZGC uses mechanisms including:

Colored pointers / pointer metadata

Load barriers

Concurrent marking

Concurrent relocation


The exact implementation has evolved across JDK releases, so specific heap-size and feature claims should always be tied to a particular JDK version.


---

2.3 Memory Leak Diagnostics

Java has automatic memory management, but memory leaks can still occur.

A Java memory leak occurs when objects are unintentionally kept reachable.

Common examples:

ThreadLocal Leaks

Thread Pool
    │
    ▼
Worker Thread
    │
    ▼
ThreadLocalMap
    │
    ▼
Large Object

If a pooled thread lives for a long time, retained ThreadLocal values can keep objects alive longer than intended.


---

Collection Growth

static List<Object> cache = new ArrayList<>();

If objects are continuously added without eviction:

List
 ├── Object
 ├── Object
 ├── Object
 ├── Object
 └── ...

the application may eventually run out of heap.


---

Listener Leaks

Objects registered as event listeners but never removed can remain reachable through the event source.


---

Shallow Heap vs Retained Heap

Shallow Heap

Memory directly occupied by an object.

Retained Heap

Memory that would become unreachable if that object were removed.

Object A
   │
   ├── Object B
   ├── Object C
   └── Object D

If A is the only thing keeping B, C and D alive:

Retained Heap(A)
≈ A + B + C + D

Tools such as Eclipse MAT can help analyze:

Dominator trees

Retained heap

GC roots

Heap dumps

Object references



---

Chapter 3 — Collections & Data Structures


---

3.1 HashMap Internals

A simplified HashMap:

HashMap
   │
   ▼
Bucket Array

0 ──► Node
1 ──► Node ──► Node
2
3 ──► Node
4
...

For a table of size n, bucket selection is conceptually:

index = (n - 1) & hash

This works efficiently when the table length is a power of two.


---

Hash Spreading

Java's HashMap spreads hash bits approximately using:

h ^ (h >>> 16)

Why?

Because the bucket calculation uses low-order bits.

If the original hash has useful entropy primarily in the upper bits, spreading helps distribute keys more evenly.


---

Collision Handling

Multiple keys can map to the same bucket.

Historically:

Bucket
  │
  ▼
Node → Node → Node → Node

Java 8 introduced treeification of sufficiently large collision chains.

When appropriate conditions are satisfied, a bucket can become a Red-Black Tree.

Typical thresholds include:

TREEIFY_THRESHOLD = 8
MIN_TREEIFY_CAPACITY = 64
UNTREEIFY_THRESHOLD = 6

These are implementation details rather than guarantees of the Map interface.


---

HashMap Complexity

Average case:

get() → O(1)
put() → O(1)
remove() → O(1)

Under severe collisions:

Linked structure → O(n)
Treeified structure → O(log n)

assuming normal tree behavior.


---

3.2 ConcurrentHashMap

ConcurrentHashMap provides concurrent access without using one global lock for the entire map.

A simplified model:

Thread A ──► Bucket 1
Thread B ──► Bucket 7
Thread C ──► Bucket 15

Different buckets
      ↓
Higher concurrency

For empty bins, insertion can use CAS-based operations.

Conceptually:

if bucket == null
       │
       ▼
   CAS insert
       │
       ▼
     success

For non-empty bins, synchronization may occur around the relevant bin.

During resizing, special forwarding nodes help coordinate table migration.

Old Table
   │
   ▼
ForwardingNode
   │
   ▼
New Table

The important interview point is:

> ConcurrentHashMap avoids a single global lock and allows multiple threads to operate concurrently on independent portions of the table.




---

ArrayList vs LinkedList

ArrayList

┌────┬────┬────┬────┬────┬────┐
│ A  │ B  │ C  │ D  │ E  │ F  │
└────┴────┴────┴────┴────┴────┘

Advantages:

Contiguous backing array

Excellent locality

Fast indexed access

Lower per-element overhead


get(index) → O(1)


---

LinkedList

A ──► B ──► C ──► D

Each node contains references to neighboring nodes.

Node
┌──────────────┐
│ prev         │
│ data         │
│ next         │
└──────────────┘

Disadvantages:

Pointer chasing

Poor cache locality

Higher memory overhead


Therefore, despite both supporting list operations, ArrayList is often substantially faster in real workloads.


---

Skip Lists

A Skip List maintains multiple levels of linked nodes.

Level 3:  A ─────────────────────► G
Level 2:  A ─────► D ────────────► G
Level 1:  A ─► B ─► C ─► D ─► E ─► F ─► G

Average search complexity:

O(log n)

ConcurrentSkipListMap provides a concurrent sorted-map implementation based on this concept.


---

Chapter 4 — Java Concurrency & Virtual Threads


---

4.1 Java Memory Model

The Java Memory Model (JMM) defines how threads interact through memory.

The three major concerns are:

Visibility
Ordering
Atomicity


---

Volatile

Consider:

volatile boolean running = true;

A write to a volatile variable establishes visibility guarantees to subsequent volatile reads.

It also provides ordering guarantees around the volatile access.

Conceptually:

Thread A
running = false
      │
      ▼
Visibility / Ordering
      │
      ▼
Thread B
reads running == false

> volatile does not make arbitrary compound operations atomic.



For example:

count++;

is not made atomic simply because count is volatile.

It is effectively:

read
+
increment
+
write

For atomic counters, use tools such as:

AtomicInteger
AtomicLong
LongAdder

when appropriate.


---

CPU Cache and Memory Ordering

Modern CPUs contain multiple cache levels.

CPU Core 1                  CPU Core 2
   │                           │
 L1 / L2                     L1 / L2
   │                           │
   └──────── L3 ───────────────┘
                │
              RAM

The JVM and CPU memory model cooperate to maintain the guarantees defined by the JMM.

The simplistic statement:

> "volatile always reads directly from RAM"



is not technically accurate.

The important guarantee is visibility and ordering, not bypassing CPU caches literally.


---

False Sharing

False sharing occurs when independent variables used by different threads occupy the same cache line.

Cache Line
┌──────────────────────────────────────────────┐
│ Thread A variable │ Thread B variable        │
└──────────────────────────────────────────────┘

Thread A modifies its variable.

The cache line becomes invalid/reconciled across cores, even though Thread B is modifying a logically unrelated variable.

This can cause severe contention.

Possible mitigation:

Padding
Alignment
@Contended
Data layout changes

@Contended is primarily intended for JDK/internal performance-sensitive use and may require JVM options depending on usage.


---

4.2 Locking

Conceptually, synchronization can involve:

Unlocked
   │
   ▼
Contended locking
   │
   ▼
ObjectMonitor / heavier synchronization

Modern HotSpot locking implementations have evolved significantly, so avoid presenting old "biased → lightweight → heavyweight" diagrams as universal behavior across all JDK versions.


---

Java 21 Virtual Threads

Virtual threads were finalized in Java 21.

Traditional platform threads:

Java Thread
     │
     ▼
OS Thread
     │
     ▼
CPU

Virtual threads:

Millions of Virtual Threads
          │
          ▼
Small Number of Carrier Threads
          │
          ▼
       OS Threads

This enables applications to create very large numbers of concurrent tasks efficiently, particularly for workloads involving blocking I/O.


---

Mounting and Unmounting

Conceptually:

Virtual Thread
      │
      ▼
Mounted on Carrier
      │
      ▼
Executing
      │
      ▼
Blocking I/O
      │
      ▼
Unmount
      │
      ▼
Carrier available

The virtual thread's continuation can be suspended while the carrier thread executes other work.


---

Virtual Thread Pinning

Certain operations can prevent a virtual thread from efficiently unmounting.

One important example historically has been blocking while holding a monitor associated with:

synchronized

JNI/native interactions can also cause pinning.

Therefore, virtual-thread applications should carefully examine synchronization and blocking behavior.

ReentrantLock can be preferable in some cases, but it should not be treated as a universal replacement for synchronized.


---

SECTION II — SPRING BOOT FRAMEWORK ARCHITECTURE


---

Chapter 5 — IoC Container & Bean Lifecycle

Spring's Inversion of Control (IoC) container manages objects called beans.

Instead of manually constructing every dependency:

PaymentService service =
    new PaymentService(new PaymentRepository());

Spring manages the dependency graph.

ApplicationContext
       │
       ├── Controller
       │      │
       │      ▼
       │   Service
       │      │
       │      ▼
       │   Repository
       │
       └── Other Beans


---

5.1 Bean Lifecycle

A simplified lifecycle:

Bean Definition
      │
      ▼
Instantiation
      │
      ▼
Dependency Injection
      │
      ▼
Aware Callbacks
      │
      ▼
BeanPostProcessor
Before Initialization
      │
      ▼
@PostConstruct
      │
      ▼
InitializingBean
      │
      ▼
Custom init-method
      │
      ▼
BeanPostProcessor
After Initialization
      │
      ▼
READY
      │
      ▼
Destruction


---

Instantiation

Spring creates the object, often through reflection and constructor resolution.


---

Dependency Injection

Example:

@Service
public class PaymentService {

    private final PaymentRepository repository;

    public PaymentService(PaymentRepository repository) {
        this.repository = repository;
    }
}

Spring resolves:

PaymentService
      │
      ▼
PaymentRepository

Constructor injection is generally preferred because dependencies are explicit and can support immutable fields.


---

Aware Interfaces

Spring can provide framework-related information through interfaces such as:

BeanNameAware
BeanFactoryAware
ApplicationContextAware

These should generally be used only when necessary because they couple application code to Spring infrastructure.


---

Initialization Order

A simplified sequence is:

BeanPostProcessor.beforeInitialization()
          ↓
@PostConstruct
          ↓
InitializingBean.afterPropertiesSet()
          ↓
custom init-method
          ↓
BeanPostProcessor.afterInitialization()

The exact lifecycle can contain additional callbacks and infrastructure-specific processing.


---

Destruction

For singleton beans, destruction can involve:

@PreDestroy
     ↓
DisposableBean.destroy()
     ↓
custom destroy-method


---

5.2 Circular Dependencies

Consider:

Service A
   │
   ▼
Service B
   │
   ▼
Service A

For certain singleton field/setter-injection scenarios, Spring can expose an early reference through its singleton creation infrastructure.

Conceptually:

┌───────────────────────────────┐
│ singletonObjects              │
│ Fully initialized beans       │
└───────────────────────────────┘

┌───────────────────────────────┐
│ earlySingletonObjects         │
│ Early singleton references    │
└───────────────────────────────┘

┌───────────────────────────────┐
│ singletonFactories            │
│ Factories for early references│
└───────────────────────────────┘

This is commonly described as Spring's three-level singleton cache.

Constructor circular dependencies cannot be solved this way because the object cannot be exposed before its constructor has completed.

Possible solution:

@Lazy

or, preferably, redesign the dependency graph.


---

Chapter 6 — Spring AOP & Proxies

Spring AOP is commonly implemented using proxies.

Client
  │
  ▼
Proxy
  │
  ├── Transaction
  ├── Security
  ├── Logging
  ├── Metrics
  │
  ▼
Target Object


---

6.1 JDK Dynamic Proxy

If proxying an interface:

interface PaymentService {
    void pay();
}

Spring can use a JDK dynamic proxy.

Client
  │
  ▼
JDK Proxy
  │
  ▼
Interface
  │
  ▼
Target


---

CGLIB / Class-Based Proxying

Spring can also create subclass-based proxies.

Target Class
     │
     ▼
Generated Subclass
     │
     ▼
Intercept method

Modern Spring commonly uses class-based proxies by default in many Spring Boot configurations, but proxy selection can depend on configuration and framework version.


---

6.2 Self-Invocation Trap

Consider:

@Service
public class PaymentService {

    public void process() {
        this.save();
    }

    @Transactional
    public void save() {
        // DB operation
    }
}

External call:

Client
  │
  ▼
Spring Proxy
  │
  ▼
process()

But inside:

this.save();

the call goes directly to the target object.

It does not pass through the Spring proxy.

Therefore the transactional interceptor is bypassed.


---

Solution 1 — Refactor

Move the transactional method into another bean:

PaymentService
      │
      ▼
PaymentPersistenceService
      │
      ▼
@Transactional

This is usually the cleanest solution.


---

Solution 2 — Proxy-Aware Invocation

Another approach is obtaining/injecting the proxied bean, but this introduces additional coupling and should be used carefully.


---

Chapter 7 — Transaction Management & Isolation

A database transaction provides a logical unit of work.

BEGIN
  │
  ├── Operation A
  ├── Operation B
  ├── Operation C
  │
  ▼
COMMIT

If something fails:

ROLLBACK


---

7.1 Isolation Levels

The common SQL isolation levels are:

READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE

Different databases implement these semantics differently.


---

Dirty Read

Transaction A:

UPDATE balance = 500

before commit.

Transaction B reads:

balance = 500

If A rolls back:

Original = 1000
Final    = 1000

B read data that never became committed.

This is a dirty read.


---

Non-Repeatable Read

Transaction A:

SELECT balance
→ 1000

Transaction B:

UPDATE balance = 500
COMMIT

Transaction A reads again:

SELECT balance
→ 500

The same row returned different committed values.


---

Phantom Read

Transaction A:

SELECT * FROM orders
WHERE amount > 1000;

Transaction B inserts another matching row and commits.

Transaction A executes the range query again and sees an additional row.

That is a phantom read.


---

Isolation Overview

Isolation Level       Dirty    Non-repeatable    Phantom
---------------------------------------------------------
READ UNCOMMITTED       ✓             ✓              ✓
READ COMMITTED         ✗             ✓              ✓
REPEATABLE READ        ✗             ✗              DB-dependent
SERIALIZABLE           ✗             ✗              ✗

The exact behavior depends on the database's concurrency-control implementation.


---

7.2 Transaction Propagation

Spring transaction propagation determines how a method participates in an existing transaction.


---

REQUIRED

Default propagation.

Existing Transaction?
       │
   ┌───┴───┐
   │       │
  Yes      No
   │       │
 Join    Create


---

REQUIRES_NEW

Suspends the existing transaction and creates a separate transaction.

Outer Transaction
       │
       ▼
Suspend
       │
       ▼
New Transaction
       │
       ▼
Commit/Rollback
       │
       ▼
Resume Outer

Useful for independent operations such as certain audit/logging workflows.


---

NESTED

Uses a savepoint when supported by the underlying transaction manager/database.

Outer Transaction
      │
      ▼
 Savepoint
      │
      ├── Operation A
      ├── Operation B
      │
      ▼
Failure
      │
      ▼
Rollback to Savepoint
      │
      ▼
Continue Outer Transaction

NESTED is not equivalent to REQUIRES_NEW.


---

SECTION III — DATA PERSISTENCE & DATABASE INTERNALS


---

Chapter 8 — JPA Mechanics & Entity States

JPA entities move through different lifecycle states.

new Entity()
                  │
                  ▼
              Transient
                  │
             persist()
                  │
                  ▼
              Persistent
              /         \
             /           \
         detach()       remove()
           │               │
           ▼               ▼
       Detached         Removed


---

8.1 Transient

User user = new User();

The object exists only in application memory.

It is not managed by the persistence context.


---

Persistent / Managed

Once associated with an EntityManager:

Entity
  │
  ▼
Persistence Context

Hibernate tracks changes to the entity.


---

Detached

An entity becomes detached when, for example:

entityManager.detach(user);

or:

entityManager.clear();

or when the persistence context/session ends.

The Java object still exists, but Hibernate is no longer tracking it as managed.


---

Removed

An entity can be marked for deletion:

entityManager.remove(user);

The SQL DELETE typically occurs during flush/transaction synchronization.


---

8.2 Dirty Checking

Hibernate performs dirty checking for managed entities.

Suppose:

User user = repository.findById(1L);

user.setName("Kunal");

No explicit:

repository.save(user);

is necessarily required for the update when the entity is managed inside an active transaction.

Conceptually:

Database
   │
   ▼
Load Entity
   │
   ▼
Persistence Context
   │
   ├── Current State
   └── Snapshot
   │
   ▼
Entity Modified
   │
   ▼
Flush
   │
   ▼
Compare State
   │
   ▼
Generate UPDATE

Hibernate's exact implementation and optimizations can vary by version and configuration, but the key concept is:

> Managed entity changes can be detected automatically during flush.




---

Chapter 9 — Database Tuning & Indexing


---

9.1 N+1 Query Problem

Suppose we have:

Author
 ├── Book
 ├── Book
 └── Book

Application loads:

SELECT * FROM author;

Then for every author:

SELECT * FROM book WHERE author_id = ?;

If there are N authors:

1 query
+
N queries

= N + 1 queries


---

Solution 1 — JOIN FETCH

JPQL:

SELECT a
FROM Author a
JOIN FETCH a.books

Conceptually:

Author ───────── Book
       JOIN

This can reduce the number of round trips.

However, fetch joins need careful handling when fetching multiple collections because they can create row multiplication.


---

Solution 2 — Entity Graph

@EntityGraph(attributePaths = {"books"})

This allows fetch plans to be specified declaratively.


---

Solution 3 — Batch Fetching

Hibernate can batch lazy collection/entity loading.

Example:

@BatchSize(size = 25)

Instead of:

SELECT book WHERE author_id = 1
SELECT book WHERE author_id = 2
SELECT book WHERE author_id = 3
...

Hibernate may group identifiers:

SELECT book
WHERE author_id IN (1,2,3,...)


---

9.2 B+Tree Index

A simplified B+Tree:

[50 | 100]
                /     |      \
               /      |       \
       [20 | 35]   [70 | 85]   [120 | 150]
          /   \         \           \
        ...   ...       ...         ...

Leaf Level:

[10|15] ⇄ [20|30] ⇄ [35|45] ⇄ [50|60] ⇄ ...

B+Trees are efficient for:

Equality lookup
Range lookup
Ordering
Prefix scans


---

Clustered Index

In systems such as InnoDB, the primary key determines the clustered storage organization.

Conceptually:

Primary Key Index
       │
       ▼
Actual Row Data

The exact storage model depends on the database engine.


---

Secondary Index

A secondary index generally stores:

Indexed Key
     +
Row Locator / Primary Key

Depending on the database engine, the lookup may require another traversal to retrieve the complete row.


---

Covering Index

If the index contains everything needed for the query:

SELECT name, age
FROM users
WHERE email = ?;

and an appropriate index contains:

email + name + age

the database may satisfy the query directly from the index.

This is called a covering index.


---

Leftmost Prefix Rule

For:

INDEX(A, B, C)

the index is naturally useful for:

(A)
(A, B)
(A, B, C)

But generally not for:

(B)
(C)
(B, C)

because the B+Tree is ordered first by A.


---

SECTION IV — SYSTEM DESIGN & DISTRIBUTED ARCHITECTURES


---

Chapter 10 — Scalability & Load Balancing


---

10.1 Consistent Hashing

Traditional routing:

server = hash(key) % N

Suppose:

N = 3

and then:

N = 4

A large portion of keys may map to different servers.

This creates massive cache movement.


---

Consistent Hashing Ring

Instead, map servers and keys onto a logical ring:

Server A
                      ●
                  .       .
               .             .
             .                 .
        Key K1 ●               ● Server B
             .                 .
               .             .
                  .       .
                      ●
                   Server C

Keys are assigned to the next server encountered while moving around the ring.


---

Adding a Server

Before:

A ───────── B ───────── C

After:

A ───── D ───── B ───── C

Only keys in the affected region need to move.

This significantly reduces remapping compared with:

hash(key) % N


---

Virtual Nodes

A physical server can own multiple positions:

Server A
 ├── vnode 1
 ├── vnode 2
 ├── vnode 3
 └── vnode 4

Ring:

A1 ─ B1 ─ A2 ─ C1 ─ B2 ─ A3 ─ C2 ─ ...

This improves distribution and reduces hotspots caused by uneven server positions.


---

Chapter 11 — Distributed Caching

Caching reduces expensive database access.

Client
  │
  ▼
Application
  │
  ├────────► Cache
  │            │
  │          HIT
  │            │
  │            ▼
  │         Response
  │
  └────────► Database


---

11.1 Cache-Aside

Most common application caching strategy.

Application
    │
    ▼
 Check Cache
    │
 ┌──┴──┐
 │     │
Hit   Miss
 │     │
 ▼     ▼
Return DB
       │
       ▼
 Populate Cache
       │
       ▼
    Return

Example:

GET /users/101
       │
       ▼
Redis GET user:101
       │
       ├── Hit → Return
       │
       └── Miss
             │
             ▼
         PostgreSQL
             │
             ▼
          Redis SET
             │
             ▼
          Response


---

Write-Through

Application
     │
     ▼
   Cache
     │
     ▼
 Database

The cache write path synchronously updates the database.


---

Write-Behind / Write-Back

Application
     │
     ▼
   Cache
     │
     ▼
Async Queue / Flush
     │
     ▼
 Database

Advantages:

Lower write latency

Batch writes


Risks:

Data loss if durability isn't handled correctly

More complex consistency semantics



---

11.2 Cache Stampede

Suppose a popular key expires:

Cache Miss
                 │
      ┌──────────┼──────────┐
      ▼          ▼          ▼
 Request A   Request B   Request C
      │          │          │
      └──────────┼──────────┘
                 ▼
             Database

Thousands of requests can hit the database simultaneously.


---

Solutions

Mutex / Distributed Lock

Request A
   │
   ▼
Acquire Lock
   │
   ▼
Load DB
   │
   ▼
Populate Cache
   │
   ▼
Release Lock

Other requests wait and then read the cache.


---

Probabilistic Early Expiration

Refresh popular keys before their exact expiration time to reduce synchronized cache misses.


---

Cache Penetration

Attackers may repeatedly request nonexistent keys:

user:999999999
user:999999998
user:999999997
...

If every request misses the cache:

Request
  ↓
Cache MISS
  ↓
Database
  ↓
NOT FOUND

the database can be overloaded.


---

Solutions

Bloom Filter

Request
   │
   ▼
Bloom Filter
   │
   ├── Definitely absent → Reject
   │
   └── Possibly present → Cache / DB

A Bloom filter provides probabilistic membership testing with possible false positives but no false negatives under normal operation.


---

Null Sentinel

Cache the fact that an object does not exist:

user:999
    ↓
NULL

with a short TTL.


---

Chapter 12 — Asynchronous Messaging & Kafka

Apache Kafka is a distributed event-streaming platform.

A simplified architecture:

Producer
   │
   ▼
Kafka Cluster
   │
   ├── Topic A
   │    ├── Partition 0
   │    ├── Partition 1
   │    └── Partition 2
   │
   ▼
Consumer Group
   ├── Consumer 1
   ├── Consumer 2
   └── Consumer 3


---

12.1 Kafka Partitions

A topic can have multiple partitions:

Orders Topic

Partition 0:
[0][1][2][3][4]

Partition 1:
[0][1][2][3]

Partition 2:
[0][1][2][3][4][5]

Ordering is guaranteed within a partition, not globally across all partitions.


---

Sequential Writes

Kafka uses append-oriented logs.

Conceptually:

Partition Log

┌────┬────┬────┬────┬────┬────┐
│ E1 │ E2 │ E3 │ E4 │ E5 │ E6 │ →
└────┴────┴────┴────┴────┴────┘

Sequential I/O is efficient for high-throughput workloads.


---

Page Cache

Kafka benefits heavily from the operating system's page cache.

Conceptually:

Kafka Log
   │
   ▼
OS Page Cache
   │
   ▼
Network
   │
   ▼
Consumer


---

Zero-Copy

Kafka can use OS mechanisms such as sendfile() in relevant data-transfer paths.

Traditional conceptual path:

Disk
 ↓
Kernel
 ↓
User Space
 ↓
Kernel
 ↓
Socket

Zero-copy style path:

Disk / Page Cache
        │
        ▼
Kernel transfer
        │
        ▼
Network Socket

This reduces unnecessary copying between kernel and user-space buffers.

The exact path depends on the operating system and Kafka/network configuration.


---

Consumer Groups

Suppose:

Topic = Orders
Partitions = 4

Consumer group:

Consumer Group

C1 → P0
C2 → P1
C3 → P2
C4 → P3

Within one consumer group, a partition is assigned to one consumer at a time.

Adding consumers beyond the number of partitions does not increase parallelism for that topic/group.


---

12.2 Transactional Outbox Pattern

A major distributed-systems problem:

Database Update
       +
Kafka Publish

What happens if:

DB COMMIT ✓
Kafka PUBLISH ✗

Now the database says the operation happened, but downstream services never receive the event.


---

Outbox Solution

Local DB Transaction
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
     Domain Table            OUTBOX Table
          │                       │
          └───────────┬───────────┘
                      ▼
                   COMMIT
                      │
                      ▼
               CDC / Outbox Worker
                      │
                      ▼
                    Kafka

The application atomically performs:

UPDATE business data
+
INSERT outbox event

inside the same database transaction.

A separate process publishes the outbox event to Kafka.

Tools such as Debezium can implement the CDC side.


---

SECTION V — FAANG INTERVIEW PLAYBOOK & CODE BLUEPRINTS


---

Chapter 13 — System Design Blueprints


---

13.1 Distributed Rate Limiter

A distributed rate limiter protects services from excessive traffic.

Example:

Client
   │
   ▼
API Gateway
   │
   ▼
Redis Rate Limiter
   │
   ├── Allowed → Backend
   │
   └── Rejected → HTTP 429


---

Sliding Window Log

Redis Sorted Set:

Key: rate:user:123

Score             Request
--------------------------------
1710000000010     R1
1710000000030     R2
1710000000050     R3

The score can represent a timestamp.

At request time:

Remove entries older than window
        │
        ▼
Count remaining requests
        │
   ┌────┴────┐
   │         │
count < L   count >= L
   │         │
Allow       Reject


---

Atomic Redis Lua Script

local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])

local clearBefore = now - window

redis.call(
    'ZREMRANGEBYSCORE',
    key,
    0,
    clearBefore
)

local currentRequests = redis.call(
    'ZCARD',
    key
)

if currentRequests < limit then

    redis.call(
        'ZADD',
        key,
        now,
        now
    )

    redis.call(
        'EXPIRE',
        key,
        math.ceil(window / 1000)
    )

    return 1

else

    return 0

end

Lua is important here because the entire sequence executes atomically inside Redis.


---

Rate Limiter Complexity

For a simple conceptual model:

ZADD              → O(log N)
ZREMRANGEBYSCORE  → depends on removed entries
ZCARD             → O(1)

The exact performance depends on traffic patterns and the number of expired entries removed.


---

13.2 Distributed ID Generator — Snowflake

A Snowflake-style ID avoids centralized database coordination.

Typical 64-bit structure:

┌──────┬───────────────────────┬──────────────┬──────────────┐
│  1   │        41 bits        │   10 bits    │   12 bits    │
│Sign  │      Timestamp        │ Worker / Node │   Sequence   │
└──────┴───────────────────────┴──────────────┴──────────────┘

The exact bit allocation is a design choice.


---

Timestamp

Provides roughly time-ordered IDs.

Timestamp
    ↓
Monotonically increasing component


---

Worker ID

Identifies the node generating the ID.

Example:

Worker 1
Worker 2
Worker 3
...


---

Sequence

Multiple IDs generated within the same timestamp tick require a sequence number.

Timestamp T

T + sequence 0
T + sequence 1
T + sequence 2
T + sequence 3


---

Snowflake Advantages

Distributed

No central database required

High throughput

Roughly time ordered

Compact compared with UUID strings



---

Snowflake Challenges

Clock Rollback

If system time moves backwards:

Current timestamp
       ↓
Clock rollback
       ↓
Duplicate / ordering risk

Production implementations must handle clock drift carefully.


---

Chapter 14 — Enterprise Resilient Code Blueprint

Distributed services fail.

A production service should assume that:

Network can fail
Dependency can timeout
Database can become slow
Service can become unavailable
Packets can be lost

Two common resilience mechanisms are:

Retry
Circuit Breaker


---

Retry

If a transient failure occurs:

Request
  │
  ▼
Dependency
  │
  ✗ Failure
  │
  ▼
Retry
  │
  ▼
Dependency

But retries must be controlled.

Unbounded retries can create a retry storm.


---

Exponential Backoff

Instead of:

Retry immediately
Retry immediately
Retry immediately

use:

Attempt 1 → 500 ms
Attempt 2 → 1000 ms
Attempt 3 → 2000 ms

Conceptually:

delay = baseDelay × multiplier^(attempt-1)

A production implementation often adds jitter to avoid many clients retrying simultaneously.


---

Circuit Breaker

Circuit breaker states:

failures
CLOSED ─────────────────► OPEN
  ▲                         │
  │                         │ wait
  │                         ▼
  └──────── success ── HALF_OPEN


---

CLOSED

Requests flow normally.

Client
  │
  ▼
Service
  │
  ▼
Dependency

Failures are measured.


---

OPEN

After the failure threshold is reached:

Client
  │
  ▼
Circuit Breaker
  │
  X
  │
Fallback

The dependency is not called.

This protects the system from repeatedly hitting an unhealthy service.


---

HALF_OPEN

After the configured wait period:

OPEN
 │
 ▼
HALF_OPEN
 │
 ├── Success → CLOSED
 │
 └── Failure → OPEN

A limited number of trial calls determine whether the dependency has recovered.


---

Production-Style Spring Boot Example

package com.handbook.architecture;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;

import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class ResilientPaymentClient {

    private final RestTemplate restTemplate;

    public ResilientPaymentClient(
            RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @Retry(
        name = "paymentRetry",
        fallbackMethod = "paymentFallback"
    )
    @CircuitBreaker(
        name = "paymentCircuitBreaker",
        fallbackMethod = "paymentFallback"
    )
    public String processPayment(
            String orderId,
            double amount) {

        return restTemplate.postForObject(
            "https://api.payments.com/v1/charge",
            new PaymentRequest(orderId, amount),
            String.class
        );
    }

    public String paymentFallback(
            String orderId,
            double amount,
            Throwable t) {

        return "DEGRADED_STATE: " +
               "Payment queued asynchronously. " +
               "Cause: " +
               t.getMessage();
    }

    private record PaymentRequest(
            String orderId,
            double amount) {
    }
}


---

Resilience4j Configuration

resilience4j:
  circuitbreaker:
    instances:
      paymentCircuitBreaker:
        slidingWindowSize: 100
        failureRateThreshold: 50
        waitDurationInOpenState: 10s

  retry:
    instances:
      paymentRetry:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2


---

Important Production Considerations

A retry should generally be used only when the failure is likely to be transient.

Good candidates can include:

Timeout
Temporary network failure
Transient dependency failure

Be careful retrying:

Validation errors
Authentication failures
Permanent 4xx errors
Non-idempotent operations

For payment APIs especially, retries must consider idempotency.


---

Idempotency

Suppose:

POST /payment

succeeds on the server but the response is lost.

Client:

Request → Payment Server
             │
             ▼
          Payment ✓
             │
             X
          Response lost

Client retries.

Without idempotency:

Payment #1 ✓
Payment #2 ✓

The customer could be charged twice.

With an idempotency key:

Idempotency-Key: abc123

the server can recognize that the operation was already processed.

Request 1
   │
   ▼
abc123 → Process
   │
   ▼
Success

Request 2
   │
   ▼
abc123 → Already processed
   │
   ▼
Return previous result


---

Backend Architecture — Putting Everything Together

The concepts in these notes combine into a production architecture like:

┌──────────────────┐
                         │      Client      │
                         └────────┬─────────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │   API Gateway    │
                         │                  │
                         │ Auth / RateLimit │
                         └────────┬─────────┘
                                  │
                     ┌────────────┴────────────┐
                     │                         │
                     ▼                         ▼
             ┌───────────────┐        ┌───────────────┐
             │ Spring Boot   │        │ Spring Boot   │
             │ Service A     │        │ Service B     │
             └───────┬───────┘        └───────┬───────┘
                     │                         │
             ┌───────┴────────┐        ┌───────┴───────┐
             │                │        │               │
             ▼                ▼        ▼               ▼
          Redis           PostgreSQL  Kafka         External
          Cache           Database                  Service
             │                │        │               │
             │                │        ▼               │
             │                │   Consumer Groups      │
             │                │                        │
             └────────────────┴────────────────────────┘
                              │
                              ▼
                     Observability Layer
                              │
                  ┌───────────┼───────────┐
                  ▼           ▼           ▼
              Metrics       Logs       Traces


---

End-to-End Request Flow

A production request can involve many of the concepts discussed above:

Client
  │
  ▼
Load Balancer
  │
  ▼
API Gateway
  │
  ├── Authentication
  │
  ├── Rate Limiting ─────► Redis
  │
  ▼
Spring Boot Service
  │
  ├── AOP
  │    ├── Security
  │    ├── Transaction
  │    └── Metrics
  │
  ▼
Service Layer
  │
  ▼
JPA / Hibernate
  │
  ├── Persistence Context
  ├── Dirty Checking
  └── Transaction
  │
  ▼
PostgreSQL
  │
  ├── B+Tree Index
  └── Transaction
  │
  ▼
Outbox Table
  │
  ▼
CDC / Outbox Worker
  │
  ▼
Kafka
  │
  ├── Consumer A
  ├── Consumer B
  └── Consumer C
  │
  ▼
Downstream Services


---

JVM → Spring → Database → Distributed System

The complete engineering picture can be viewed as layers:

┌───────────────────────────────────────────────────┐
│              System Design                        │
│ Scalability / Caching / Kafka / Consistency       │
├───────────────────────────────────────────────────┤
│              Distributed Systems                   │
│ Messaging / Rate Limiting / Outbox / IDs          │
├───────────────────────────────────────────────────┤
│              Database Layer                        │
│ Transactions / Indexes / JPA / Hibernate          │
├───────────────────────────────────────────────────┤
│              Spring Boot                           │
│ IoC / AOP / Transactions / Proxies                │
├───────────────────────────────────────────────────┤
│              Java                                  │
│ Concurrency / Collections / Threads / JMM         │
├───────────────────────────────────────────────────┤
│              JVM                                   │
│ ClassLoader / Heap / Stack / JIT / GC             │
├───────────────────────────────────────────────────┤
│              Operating System                      │
│ CPU / Cache / Memory / Network / Disk             │
├───────────────────────────────────────────────────┤
│              Hardware                              │
│ CPU Cores / RAM / Storage / Network               │
└───────────────────────────────────────────────────┘


---

Interview Quick Revision

JVM

Topic	Key Point

ClassLoader	Loads and links classes
Delegation	Parent-first loading model
Class Identity	FQN + ClassLoader
Heap	Java object allocation
Stack	Per-thread execution frames
Metaspace	Native class metadata
TLAB	Fast thread-local allocation
GC Root	Starting point for reachability
Safepoint	JVM-safe execution point
G1	Region-based balanced collector
ZGC	Low-latency concurrent collector



---

Collections

Topic	Key Point

HashMap	Hash table
Bucket index	(n - 1) & hash
Hash spreading	Mixes upper hash bits
Treeification	Large collision bins can become trees
ConcurrentHashMap	Concurrent hash map with fine-grained coordination
ArrayList	Contiguous backing array
LinkedList	Node-based linked structure
SkipList	Probabilistic ordered structure



---

Concurrency

Topic	Key Point

JMM	Defines inter-thread memory semantics
volatile	Visibility + ordering, not general atomicity
AtomicInteger	Atomic numeric operations
False Sharing	Cache-line contention
Virtual Thread	Lightweight JVM-managed thread
Carrier Thread	Platform thread executing virtual thread
Pinning	Virtual thread unable to unmount efficiently



---

Spring

Topic	Key Point

IoC	Container manages dependencies
DI	Dependencies supplied by container
BeanPostProcessor	Bean lifecycle interception
AOP	Cross-cutting concerns through proxies
JDK Proxy	Interface-based
CGLIB/Class Proxy	Subclass-based
Self Invocation	Bypasses proxy
@Transactional	Proxy/interceptor-based transaction management
Circular Dependency	Certain singleton injection cycles can be resolved



---

Database / JPA

Topic	Key Point

Transient	Not managed
Persistent	Managed by persistence context
Detached	No longer managed
Removed	Marked for deletion
Dirty Checking	Detects changes during flush
N+1	One parent query + N child queries
JOIN FETCH	Fetch association with query
Entity Graph	Declarative fetch plan
B+Tree	Efficient lookup/range structure
Covering Index	Query satisfied from index



---

Distributed Systems

Topic	Key Point

Consistent Hashing	Reduces remapping during scaling
Virtual Nodes	Improve hash distribution
Cache-Aside	Read cache, DB on miss
Write-Through	Cache synchronously writes DB
Write-Behind	Async persistence
Stampede	Many requests regenerate same key
Penetration	Requests for nonexistent data
Bloom Filter	Probabilistic membership filter
Kafka	Distributed event streaming
Partition	Unit of parallelism and ordering
Outbox	Atomic DB state + event intent
Snowflake	Distributed time-based ID



---

Production Engineering Principles

The most important lesson across all these topics is that production systems are built around trade-offs.

There is rarely a universally best technology.

Performance
     ▲
     │
     │       Consistency
     │          ▲
     │         /
     │        /
     │       /
     │      /
     └──────────────────► Complexity

Examples:

Cache

Higher performance:

Cache
  ↓
Lower DB load

but introduces:

Consistency problems
+
Eviction
+
Invalidation


---

Retry

Improves resilience against transient failures:

Temporary Failure
       ↓
Retry
       ↓
Success

but excessive retries can produce:

Retry Storm
       ↓
More Load
       ↓
Dependency Failure
       ↓
More Retries
       ↓
Cascading Failure


---

Kafka

Provides:

High Throughput
+
Durability
+
Asynchronous Processing

but introduces:

Eventual Consistency
+
Operational Complexity
+
Duplicate Processing Considerations


---

Distributed Systems

The central questions should always be:

What can fail?
       ↓
What happens when it fails?
       ↓
Can the operation be retried?
       ↓
Is it idempotent?
       ↓
What consistency is required?
       ↓
How does the system recover?
       ↓
How does the system scale?
       ↓
How do we observe it?


---

Final Mental Model

When debugging or designing a Java backend system, think from the bottom upward:

┌─────────────────┐
                     │  User Request   │
                     └────────┬────────┘
                              ▼
                     ┌─────────────────┐
                     │ API / Controller│
                     └────────┬────────┘
                              ▼
                     ┌─────────────────┐
                     │ Service + AOP   │
                     └────────┬────────┘
                              ▼
                     ┌─────────────────┐
                     │ Transaction     │
                     └────────┬────────┘
                              ▼
                     ┌─────────────────┐
                     │ JPA / Hibernate │
                     └────────┬────────┘
                              ▼
                     ┌─────────────────┐
                     │ Database        │
                     │ Index / Lock    │
                     └────────┬────────┘
                              ▼
                     ┌─────────────────┐
                     │ Redis / Kafka   │
                     └────────┬────────┘
                              ▼
                     ┌─────────────────┐
                     │ JVM             │
                     │ GC / Threads    │
                     └────────┬────────┘
                              ▼
                     ┌─────────────────┐
                     │ OS / CPU / RAM  │
                     └─────────────────┘

The goal of these notes is not merely to remember what a technology does, but to understand:

> How it works internally, why it exists, what problem it solves, what trade-offs it introduces, and how it behaves under production load.




---

Core Interview Questions to Master

Before considering these topics interview-ready, you should be able to explain, without memorized definitions:

1. How does a Java class get loaded into the JVM?

2. What happens between .class loading and class initialization?

3. What is actually stored in the Java heap?

4. Why are TLABs useful?

5. How does the JVM determine that an object is garbage?

6. How does G1 differ conceptually from low-latency collectors?

7. Why can HashMap achieve O(1) average lookup?

8. Why does HashMap use powers of two for capacity?

9. Why does ConcurrentHashMap scale better than synchronized HashMap?

10. Why can ArrayList outperform LinkedList even for insert-heavy workloads?

11. What does volatile guarantee?

12. Why is volatile not enough for count++?

13. What causes false sharing?

14. How do virtual threads differ from platform threads?

15. What is a Spring Bean lifecycle?

16. How does Spring resolve some circular dependencies?

17. Why does self-invocation break @Transactional?

18. How does Spring AOP actually intercept a method?

19. What is the difference between REQUIRED and REQUIRES_NEW?

20. What exactly is dirty checking?

21. How does the N+1 query problem occur?

22. How does a B+Tree index accelerate range queries?

23. What is the leftmost-prefix rule?

24. Why does modulo hashing cause cache movement during scaling?

25. How does consistent hashing reduce remapping?

26. What is a cache stampede?

27. How does a Bloom filter prevent cache penetration?

28. Why does Kafka use partitions?

29. Where does Kafka ordering exist?

30. Why is the transactional outbox pattern required?

31. How does a distributed rate limiter remain atomic?

32. How does a Snowflake-style ID generator avoid centralized coordination?

33. Why are retries dangerous?

34. What problem does a circuit breaker solve?

35. Why is idempotency critical for distributed payment operations?


---

Repository Philosophy

Understand Internals
        ↓
Understand Trade-offs
        ↓
Design Correctly
        ↓
Implement Efficiently
        ↓
Debug Production Failures
        ↓
Build Scalable Systems

Java → JVM → Spring Boot → JPA → Database → Redis → Kafka → Distributed Systems → System Design

