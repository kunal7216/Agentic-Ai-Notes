# Agentic-Ai-Notes


SECTION I: CORE JAVA MECHANICS & JVM INTERNALS
CHAPTER 1: JVM Architecture & Memory Model
1.1 ClassLoader Subsystem
The JVM ClassLoader subsystem dynamically loads, links, and initializes classes.
                  +-----------------------------------+
                  |        Bootstrap ClassLoader      |  <-- C/C++, java.base
                  +-----------------------------------+
                                    ▲
                                    │ Delegation
                  +-----------------------------------+
                  |      Platform ClassLoader         |  <-- System modules
                  +-----------------------------------+
                                    ▲
                                    │ Delegation
                  +-----------------------------------+
                  |     Application ClassLoader       |  <-- -classpath / Application JARs
                  +-----------------------------------+
                                    ▲
                                    │ Delegation
                  +-----------------------------------+
                  |        Custom ClassLoader         |  <-- Plugins / Frameworks
                  +-----------------------------------+

 * Delegation Hierarchy: When a class is requested, a ClassLoader delegates up to its parent before attempting its own findClass().
 * Class Identity: Defined by \{\text{Fully Qualified Name}, \text{ClassLoader Instance}\}. Two classes with the same package and name loaded by different ClassLoaders cannot be cast to each other (ClassCastException).
 * Thread Context ClassLoader (TCCL): Used by system frameworks (like JDBC DriverManager) to break standard upward delegation and load implementation drivers from the application classpath.
 * Three Linking Phases:
   * Verification: Checks bytecode integrity (0xCAFEBABE magic header, operand stack bounds, access modifiers).
   * Preparation: Allocates memory in Metaspace for static fields and initializes them to language default zero values (0, null, false).
   * Resolution: Replaces symbolic references in the Constant Pool with direct Metaspace pointers.
 * Initialization (<clinit>): Executes static field assignments and static blocks thread-safely via JVM locks.
1.2 Memory Layout & Off-Heap
 * Heap: Shared across threads. Divided into Young Generation (Eden, S0, S1) and Old Generation (Tenured).
 * TLAB (Thread-Local Allocation Buffer): Region in Eden assigned per thread. Enables pointer-bump allocation without acquiring global heap locks.
 * Thread Stacks (-Xss): Per-thread private stack frames containing Local Variable Tables (LVT), Operand Stacks, and Constant Pool references.
 * Metaspace: Native off-heap storage replacing PermGen (Java 8+). Stores class metadata, bytecode streams, Vtables, and Constant Pools.
1.3 Object Header & Compressed OOPs
+-----------------------------------------------------------------------------------+
| Mark Word (8 Bytes) | Klass Word (4B/8B) | Instance Data | Alignment Padding (0-7B)|
+-----------------------------------------------------------------------------------+

 * Mark Word (64-bit): Stores HashCode, GC Age (4 bits, max 15), Biased Lock bit, and Lock State tags (01 Unlocked, 00 Lightweight, 10 Heavyweight).
 * Compressed OOPs (-XX:+UseCompressedOops): Shifts 32-bit pointers left by 3 bits (x \ll 3) on 8-byte aligned heaps, enabling 32-bit references to address up to 32 GB of Heap space (2^{32} \times 8 = 32\text{ GB}). Exceeding ~32 GB disables compression and inflates object memory footprints by 20–40%. *

   
CHAPTER 2: Garbage Collection & Profiling

2.1 GC Reachability & Safepoints
 * GC Roots: Stack local variables, active thread references, static variables, JNI references, and system locks.
 * Reference Types:
   * Strong: Never collected while root-reachable.
   * Soft: Collected only before OutOfMemoryError based on free heap space.
   * Weak: Reclaimed on the very next GC cycle (e.g., WeakHashMap).
   * Phantom: Enqueued to a ReferenceQueue post-mortem for off-heap native resource cleanup (Cleaner).
 * Safepoints: Points in execution where all application threads suspend at a memory trap (PROT_NONE page poll). Counted loops (int i = 0; i < N; i++) strip safepoint checks by default, which can cause Safepoint Sync Delays. Fix using -XX:+UseCountedLoopSafepoints.
2.2 Collector Architectures
 * G1GC (-XX:+UseG1GC): Divides heap into 2048 non-contiguous regions (1–32 MB). Uses Remembered Sets (RSets) and 512-byte Card Tables with post-write barriers to track cross-region references, enabling incremental collection pauses (-XX:MaxGCPauseMillis=200).
 * ZGC (-XX:+UseZGC): Ultra-low-latency collector (<1 ms pauses) handling heaps up to 16 TB. Employs Colored Pointers (upper metadata bits) and JIT-injected Load Barriers to relocate objects and self-heal references concurrently.
2.3 Memory Leak Diagnostics
 * Common Leak Sources: Uncleaned ThreadLocalMap entries in pooled worker threads, mutating hashCode() fields on active HashMap keys, and unassigned event listeners.
 * Analysis with MAT (Memory Analyzer Tool): Inspect Retained Heap (total memory freed if an object is GC'd) versus Shallow Heap (instance footprint). Follow Path to GC Roots excluding weak/soft references.

   
CHAPTER 3: Collections & Data Structures

3.1 HashMap & ConcurrentHashMap Internal Mechanics
 * Bitwise Bucket Index: (n - 1) \ \& \ \text{hash}, where array capacity n = 2^k.
 * Hash Spreading: (h = key.hashCode()) ^ (h >>> 16) mixes upper 16 bits into lower 16 bits to reduce bucket collisions.
 * Treeification: Converts a singly-linked list bin to a Red-Black Tree when bin node count \ge 8 AND total array capacity \ge 64. Reverts to a linked list when bin node count drops \le 6 during resize.
 * ConcurrentHashMap (Java 8+): Lock-free insertions on empty buckets via CAS (Unsafe.compareAndSwapObject). Non-empty bucket updates lock only the head node using synchronized(headNode). Uses ForwardingNode (hash = -1) during concurrent table expansion.
// Simplified ConcurrentHashMap CAS / Synchronized Bucket Invariant
final V putVal(K key, V value) {
    int hash = spread(key.hashCode());
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int i, n;
        if (tab == null || (n = tab.length) == 0) tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                break; // Lock-free insertion!
        }
        else if (f.hash == MOVED) tab = helpTransfer(tab, f);
        else {
            synchronized (f) { // Locks ONLY the head node of bucket i
                // Perform bucket insertion or update
            }
        }
    }
}

3.2 Cache Locality & SkipLists
 * ArrayList vs LinkedList: ArrayList uses contiguous memory blocks, making it CPU cache-line friendly (64-byte pre-fetching). LinkedList scatters nodes across memory, causing severe L1/L2 cache misses.
 * ConcurrentSkipListMap: A thread-safe, lock-free O(\log n) ordered map implementation using a probabilistic multi-level SkipList with CAS pointer updates.

   
CHAPTER 4: Java Concurrency & Virtual Threads

4.1 JMM, Volatile & Hardware Fences
 * Volatile Semantics: Guarantees variable read/write visibility by bypassing local CPU L1/L2 write buffers and reading directly from main memory. Prevents instruction reordering across boundaries via JIT-injected memory fences (StoreStore, StoreLoad, LoadLoad).
 * False Sharing & @Contended: Occurs when independent variables modified by separate threads reside on the same 64-byte CPU cache line, triggering MESI cache invalidation. Fixed via 128-byte padding (@Contended).
4.2 Lock Escalation & Virtual Threads
 * HotSpot Lock Escalation: Unlocked (01) \rightarrow Lightweight Lock (00) (CAS Spinlock) \rightarrow Heavyweight Lock (10) (OS Mutex / ObjectMonitor).
 * Java 21 Virtual Threads (Project Loom): M:N thread model mapping millions of Virtual Threads onto a small pool of native OS Carrier Threads.
   * Mount/Unmount: Virtual threads mount to Carrier Threads during computation and unmount during blocking I/O (stack frames move to heap memory), freeing Carrier Threads for other tasks.
   * Thread Pinning: Blocking inside a native synchronized block or JNI call pins the virtual thread to its Carrier Thread. Resolve by replacing synchronized with ReentrantLock.

     
SECTION II: SPRING BOOT FRAMEWORK ARCHITECTURE
CHAPTER 5: IoC Container & Bean Lifecycle


5.1 Bean Lifecycle
[ Instantiation ] ──► [ Populate Properties ] ──► [ Aware Interfaces ]
                                                         │
[ Destruction ] ◄── [ Bean Ready ] ◄── [ BeanPostProcessor (After) ] ◄── [ Init Methods ] ◄── [ BeanPostProcessor (Before) ]

 * Instantiation: Reflected constructor execution.
 * Populate Properties: Dependency Injection (@Autowired).
 * Aware Hooks: Invokes BeanNameAware, BeanFactoryAware, ApplicationContextAware.
 * BeanPostProcessor.postProcessBeforeInitialization().
 * Initialization: Executed in order: @PostConstruct \rightarrow InitializingBean.afterPropertiesSet() \rightarrow Custom init-method.
 * BeanPostProcessor.postProcessAfterInitialization(): AOP Proxies are created here.
 * Destruction: @PreDestroy \rightarrow DisposableBean.destroy() \rightarrow Custom destroy-method.
5.2 Three-Level Cache for Circular Dependencies
Spring resolves circular dependencies for singleton field/setter injection using three internal caches:
 * 1st Level (singletonObjects): Fully initialized, ready-to-use beans.
 * 2nd Level (earlySingletonObjects): Raw, early bean instances (unpopulated properties).
 * 3rd Level (singletonFactories): ObjectFactory wrappers capable of producing early AOP proxies.
Note: Circular dependencies in constructors cannot be auto-resolved because early instances cannot be created prior to constructor completion. Fix using @Lazy.


CHAPTER 6: Spring AOP & Proxies


6.1 Proxy Selection
 * JDK Dynamic Proxy: Interface-based dynamic proxying.
 * CGLIB Proxy: Subclass-based dynamic proxying (Spring Boot 2.x/3.x default).
6.2 The Self-Invocation Trap
Direct internal method calls within the same class bypass the Spring AOP proxy wrapper:
@Service
public class PaymentService {
    public void process() {
        // SELF-INVOCATION BUG: Calling internal method directly!
        // @Transactional annotation on save() is IGNORED because it bypasses the proxy.
        this.save(); 
    }

    @Transactional
    public void save() { /* DB Operation */ }
}

 * Solution: Refactor save() into a separate Spring Bean or self-inject the service.

   
CHAPTER 7: Transaction Management & Isolation


7.1 Isolation Levels & Anomalies
 * Dirty Read: Reading uncommitted changes. (Prevented at READ_COMMITTED).
 * Non-Repeatable Read: Re-reading a row yields modified column values. (Prevented at REPEATABLE_READ).
 * Phantom Read: Re-reading a range yields new inserted rows. (Prevented at SERIALIZABLE).
7.2 Propagation Types
 * REQUIRED (Default): Join current transaction; create a new one if none exists.
 * REQUIRES_NEW: Suspend current transaction; open a completely independent outer transaction.
 * NESTED: Create a database Savepoint. Failure rolls back to the savepoint without failing the outer transaction.


   
SECTION III: DATA PERSISTENCE & DATABASE INTERNALS
CHAPTER 8: JPA Mechanics & Entity States


8.1 Entity Lifecycle
 * Transient: Newly created instance (new User()), not associated with an EntityManager.
 * Persistent: Managed by the active Persistence Context.
 * Detached: Previously persistent instance whose EntityManager session was closed or cleared (.clear()).
 * Removed: Marked for physical deletion upon session flush.
8.2 Dirty Checking
Hibernate maintains a Hydrated State Snapshot of persistent entities upon loading. On session flush/commit, it performs property diffing against this snapshot and automatically issues necessary SQL UPDATE statements without requiring explicit .save() calls.


CHAPTER 9: Database Tuning & Indexing


9.1 N+1 Query Problem Solutions
Occurs when fetching 1 parent collection triggers N separate child SQL queries.
 * JOIN FETCH (JPQL): SELECT a FROM Author a JOIN FETCH a.books
 * Entity Graphs: @EntityGraph(attributePaths = {"books"})
 * Batch Fetching: @BatchSize(size = 25) on child collections.
9.2 B+Tree Indexing Rules
                [ Root Node: 50 | 100 ]
               /           |           \
      [ 20 | 35 ]      [ 70 | 85 ]     [ 120 | 150 ]
      /         \
  [10|15] <---> [20|30] <---> [35|45]  (Leaf Nodes - Linked List)

 * Clustered Index: Primary key index where leaf nodes hold actual physical row data.
 * Secondary Index: Leaf nodes hold the indexed column value and a pointer to the Primary Key (requiring a secondary lookup unless covered by a Covering Index).
 * Leftmost Prefix Rule: Composite indexes on (A, B, C) serve queries filtering (A), (A,B), or (A,B,C), but cannot serve queries filtering on (B) or (C) alone.


SECTION IV: SYSTEM DESIGN & DISTRIBUTED ARCHITECTURES
CHAPTER 10: Scalability & Load Balancing


10.1 Consistent Hashing
Standard modulo routing (hash(key) % N) causes massive cache invalidation when servers scale (N \to N+1). Consistent Hashing maps both servers and keys to an abstract hash ring (0 to 2^{32}-1).
                      Node A (Hash: 1000)
                   /                       \
     Key K1 (3500)                            Node B (Hash: 3000)
             \                               /
                      Node C (Hash: 6000)

 * Adding/Removing Nodes: Re-maps only \frac{1}{N} of keys.
 * Virtual Nodes: Assigns multiple ring locations per physical machine to prevent uneven load distribution (hotspots).


CHAPTER 11: Distributed Caching Architecture


11.1 Caching Strategies
 * Cache-Aside: Application reads cache \rightarrow On miss, reads DB \rightarrow Populates cache.
 * Write-Through: Application writes to cache \rightarrow Cache synchronously updates DB.
 * Write-Behind (Write-Back): Application writes to cache \rightarrow Cache asynchronously updates DB in batches.
11.2 Mitigation Strategies
 * Cache Stampede: Concurrent requests hit an expired key simultaneously. Fix using Mutex Locks or Probabilistic Early Expiration.
 * Cache Penetration: Malicious queries for non-existent keys bypass the cache entirely to hit the DB. Fix using Bloom Filters or Null Sentinels.

   
CHAPTER 12: Asynchronous Messaging & Kafka


12.1 Kafka High-Throughput Mechanics
 * Sequential Disk I/O: Appends log events sequentially to partition files.
 * Page Cache & Zero-Copy Reads: Bypasses JVM user-space heap allocation by leveraging sendfile() system calls to stream data directly from OS Page Cache to the Network Socket.
+-------------------------------------------------------------------------------+
|                               ZERO-COPY DATA PATH                             |
|  OS Page Cache ────── (Direct Kernel Memory Transfer via sendfile()) ─────► Network Socket |
+-------------------------------------------------------------------------------+

12.2 Transactional Outbox Pattern
Ensures atomic execution across database updates and event publishing without expensive 2-Phase Commit (2PC) protocols:
 * Begin local DB transaction.
 * Update domain entity table.
 * Insert event message into a local OUTBOX table.
 * Commit local DB transaction.
 * An asynchronous process (e.g., Change Data Capture via Debezium) reads the OUTBOX table and publishes events to Kafka.



SECTION V: FAANG INTERVIEW PLAYBOOK & CODE BLUEPRINTS
CHAPTER 13: System Design Blueprints



13.1 Distributed Rate Limiter (Redis + Lua)
Uses Redis Sorted Sets (ZSET) executed atomically inside a Lua Script to enforce a Sliding Window Log:
-- Atomically enforced Redis Sliding Window Rate Limiter
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local clearBefore = now - window

redis.call('ZREMRANGEBYSCORE', key, 0, clearBefore)
local currentRequests = redis.call('ZCARD', key)

if currentRequests < limit then
    redis.call('ZADD', key, now, now)
    redis.call('EXPIRE', key, math.ceil(window / 1000))
    return 1 -- Allowed
else
    return 0 -- Rejected (HTTP 429)
end

13.2 Distributed ID Generator (Snowflake)
Generates 64-bit globally unique, roughly time-ordered IDs without database coordination:
+-------------------------------------------------------------------------------+
| 1 Bit (Unused) | 41 Bits: Epoch Timestamp | 10 Bits: Machine ID | 12 Bits: Sequence |
+-------------------------------------------------------------------------------+




CHAPTER 14: Enterprise Resilient Code Blueprint



Production-grade Spring Boot service incorporating Resilience4j Circuit Breaker and Exponential Backoff Retry:
package com.handbook.architecture;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class ResilientPaymentClient {

    private final RestTemplate restTemplate;

    public ResilientPaymentClient(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @Retry(name = "paymentRetry", fallbackMethod = "paymentFallback")
    @CircuitBreaker(name = "paymentCircuitBreaker", fallbackMethod = "paymentFallback")
    public String processPayment(String orderId, double amount) {
        return restTemplate.postForObject("https://api.payments.com/v1/charge", 
            new PaymentRequest(orderId, amount), String.class);
    }

    // Fallback executes when Circuit is OPEN or Retries are exhausted
    public String paymentFallback(String orderId, double amount, Throwable t) {
        return "DEGRADED_STATE: Payment queued asynchronously. Cause: " + t.getMessage();
    }

    private record PaymentRequest(String orderId, double amount) {}
}

# application.yml Configuration
resilience4j.circuitbreaker:
  instances:
    paymentCircuitBreaker:
      slidingWindowSize: 100
      failureRateThreshold: 50
      waitDurationInOpenState: 10000ms

resilience4j.retry:
  instances:
    paymentRetry:
      maxAttempts: 3
      waitDuration: 500ms
      enableExponentialBackoff: true
      exponentialBackoffMultiplier: 2

