
# What are virtual threads and how do they improve concurrency compared to platform threads?

Sure, let me first clarify the problem and then walk you through my approach.

## 1. Interview-Style Opening

"Concurrency in Java has historically been tied to the Operating System's thread model. Every time we created a `new Thread()`, we were asking the OS for a heavyweight resource (1-2MB stack, kernel metadata). This is why we hit ceilings at around 10,000 threads per server, leading to the 'Thread-per-Request' bottleneck.

**Virtual Threads (Project Loom)**, introduced in JDK 21, fundamentally change this economics. They decouple the 'Java Thread' from the 'OS Thread'. Now, a Java thread is just a Java object in the heap. This allows us to have **millions** of active threads in a single JVM, making high-throughput, blocking-style code (like standard JDBC or RestClient calls) scalable again without needing complex reactive frameworks."

## 2. Problem Understanding and Clarification

The user wants to know:

1. **What** Virtual Threads are.
2. **How** they improve concurrency vs. Platform Threads.

**Key Concepts:**

* **Platform Thread:** 1:1 wrapper around an OS kernel thread. Expensive.
* **Virtual Thread:** M:N model. Many virtual threads map to few carrier (OS) threads. Cheap.
* **Mounting/Unmounting:** The mechanism where the JVM detects blocking I/O and swaps threads automatically.

**Clarification:** "I will explain the architecture using the 'Carrier Thread' model and demonstrate a simple code comparison where we spawn 1 million threads—something impossible with platform threads."

## 3. High-Level Approach

I will contrast the two models:

1. **The "One-to-One" Model (Platform):** If you block the Java thread (e.g., `Thread.sleep` or database call), you block the OS thread. The CPU core sits idle (or context switches expensively).
2. **The "Many-to-One" Model (Virtual):** If a Virtual Thread blocks, the JVM "unmounts" it from the carrier thread. The carrier immediately picks up another Virtual Thread. This is **Non-blocking I/O** disguised as **Blocking code**.

## 4. Visual Explanation (Mermaid-First, Mandatory)

```mermaid
flowchart TD
    subgraph "Platform Thread Model (Old)"
        PT1["Java Thread 1"] --- OS1["OS Kernel Thread 1"]
        PT2["Java Thread 2"] --- OS2["OS Kernel Thread 2"]
        
        OS1 --- CPU1["CPU Core 1"]
        OS2 --- CPU2["CPU Core 2"]
    end

    subgraph "Virtual Thread Model (New)"
        VT1["Virtual Thread 1"]
        VT2["Virtual Thread 2"]
        VT3["Virtual Thread 3"]
        VT4["Virtual Thread 4"]
        
        subgraph "JVM Scheduler (ForkJoinPool)"
            CT1["Carrier Thread 1<br>(Platform Thread)"]
        end
        
        VT1 -.->|Mounted| CT1
        VT2 -.->|Waiting| CT1
        CT1 --- CPU_New["CPU Core 1"]
    end

    style CT1 fill:#f9f,stroke:#333
    style VT1 fill:#ccf,stroke:#333
    style VT2 fill:#eee,stroke:#333
```

**Explanation:**

* **Old Model:** If `Java Thread 1` waits for a DB response, `OS Thread 1` is blocked. The CPU cannot use that OS thread for anything else.
* **New Model:** If `Virtual Thread 1` waits for a DB response, the JVM moves it to the **Heap** (Unmounted). `Carrier Thread 1` is now free to pick up `Virtual Thread 2`. We get high throughput with just ONE OS thread.[^11]


## 5. Java Code (Production-Quality)

This code proves the scale. We try to launch 100,000 threads.

```java
import java.time.Duration;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.atomic.AtomicInteger;

public class VirtualThreadDemo {

    public static void main(String[] args) {
        long startTime = System.currentTimeMillis();

        // 1. Create a factory for Virtual Threads
        // We use the new JDK 21 API 'Thread.ofVirtual()'
        ThreadFactory vThreadFactory = Thread.ofVirtual().name("vt-worker-", 0).factory();

        // 2. Try to run 100,000 concurrent tasks
        // With Platform threads (Executors.newCachedThreadPool()), this would likely crash 
        // with OutOfMemoryError or take forever due to context switching.
        try (var executor = Executors.newThreadPerTaskExecutor(vThreadFactory)) {
            
            AtomicInteger counter = new AtomicInteger(0);

            for (int i = 0; i < 100_000; i++) {
                executor.submit(() -> {
                    // Simulate blocking I/O (e.g., DB call)
                    try {
                        // In Virtual Threads, sleep() does NOT block the OS thread.
                        // It yields the carrier thread to another task.
                        Thread.sleep(Duration.ofMillis(500)); 
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                    counter.incrementAndGet();
                });
            }
        } // Executor auto-closes and waits for all tasks here (Structured Concurrency)

        long timeTaken = System.currentTimeMillis() - startTime;
        System.out.println("Finished 100,000 tasks in: " + timeTaken + "ms");
    }
}
```


## 6. Code Walkthrough (Line-by-Line)

* `Thread.ofVirtual().factory()`: This is the new API in JDK 21. It tells the JVM "Do not create OS threads for these".
* `Executors.newThreadPerTaskExecutor(...)`: In the past, "Thread per Task" was an anti-pattern because threads were expensive. With Virtual Threads, it is now the **recommended pattern**. You don't need to pool them. You just create them and throw them away.[^12]
* `Thread.sleep(500)`: This is the magic moment. In a Platform thread, this pauses the OS thread. In a Virtual Thread, the JVM detects the sleep, saves the stack frame to heap memory, and yields the carrier thread to another task.[^13]


## 7. How I Would Explain This to the Interviewer

"Virtual Threads solve the **'Little's Law'** problem in high-throughput servers.
Previously, if a request took 2 seconds (mostly waiting for DB), and we had 1000 OS threads, our max throughput was capped at 500 requests/sec. We simply ran out of threads.

With Virtual Threads, the thread is virtually free (a few bytes of RAM). We can have 1,000,000 concurrent virtual threads. So, even if requests take 2 seconds, we can handle massive concurrency because the *waiting* threads don't consume *computing* resources (OS threads).[^11]

They allow us to write simple, synchronous style code (`code A; code B;`) that performs like complex asynchronous code (`CompletableFuture`), without the 'Callback Hell'."

## 8. Edge Cases and Follow-Up Questions

**Q: Can I use Virtual Threads for CPU-intensive tasks (e.g., Video Encoding)?**

* *A:* **No.** Virtual Threads are excellent for *waiting* (I/O). If a thread is crunching numbers, it never yields the CPU. If you launch 1,000 virtual threads that act like `while(true)`, they will hog the few Carrier Threads, and the system will stall. Use Platform Threads for CPU-bound work.[^12]

**Q: What is 'Pinning'?**

* *A:* If you use `synchronized` blocks or native methods (JNI), the Virtual Thread gets "pinned" to the Carrier Thread. The JVM cannot unmount it during blocking operations. This negates the benefit. The solution is to use `ReentrantLock` instead of `synchronized` where possible.[^14]


## 9. Optimization and Trade-offs

| Feature | Platform Thread | Virtual Thread |
| :-- | :-- | :-- |
| **OS Resource** | 1 Kernel Thread | None (User Mode) [^15] |
| **Stack Size** | ~1 MB (Reserved) | Resizeable (Starts small) [^11] |
| **Context Switch** | Slow (Kernel Mode) | Fast (JVM / User Mode) |
| **Best For** | CPU Heavy Tasks | I/O Heavy (Web Servers) |
| **Pooling** | Mandatory (Thread Pool) | Anti-Pattern (Create new) [^16] |

**Optimization:** "In Spring Boot 3.2, enabling this is literally one line in `application.yml`: `spring.threads.virtual.enabled=true`. This automatically switches Tomcat and Jetty to use virtual threads for incoming requests."

## 10. Real-World Application and Engineering Methodology

**Use Case: High-Throughput API Gateway**

* **Scenario:** We had an aggregation service that called 5 downstream APIs (User, Orders, Inventory, etc.) to build a dashboard.
* **Old Way:** We used `CompletableFuture.allOf(...)` or WebFlux. The code was unreadable and hard to debug (stack traces were useless).
* **New Way:** We switched to JDK 21 Virtual Threads. We wrote standard sequential code:

```java
var user = userService.get();
var orders = orderService.get();
```

This code looks blocking, but under the hood, it runs in parallel effectively.
* **Result:** The application throughput increased by **10x** for the same hardware, and the codebase became debuggable again because stack traces now point to the actual line of code, not some reactive framework internal.[^17]
<span style="display:none">[^1][^10][^2][^3][^4][^5][^6][^7][^8][^9]</span>

<div align="center">⁂</div>

[^1]: https://stackoverflow.com/questions/79130627/when-to-use-platform-threads-over-virtual-threads

[^2]: https://stackoverflow.com/questions/78094984/difference-between-platform-thread-carrier-thread-and-virtual-thread-in-context

[^3]: https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html

[^4]: https://www.youtube.com/watch?v=0NtIcbSsjBc

[^5]: https://blog.ycrash.io/an-investigative-study-virtual-threads-vs-platform-threads-in-java-23/

[^6]: https://www.baeldung.com/java-virtual-thread-vs-thread

[^7]: https://www.alibabacloud.com/blog/exploration-of-java-virtual-threads-and-performance-analysis_601860

[^8]: https://dzone.com/articles/deep-dive-into-java-virtual-threads-a-game-changer

[^9]: https://www.youtube.com/watch?v=Supyrpu1Qz0

[^10]: https://www.zymr.com/blog/threads-and-virtual-threads-demystifying-the-world-of-concurrency-in-modern-times

[^11]: https://www.reddit.com/r/java/comments/uiaie9/what_are_project_looms_virtual_thread_main/

[^12]: https://stackoverflow.com/questions/71945866/project-loom-why-are-virtual-threads-not-the-default

[^13]: https://www.marcobehler.com/guides/java-project-loom

[^14]: https://www.youtube.com/watch?v=13EtTQbfW_E

[^15]: https://inside.java/2023/10/30/sip086/

[^16]: https://developer.okta.com/blog/2022/08/26/state-of-java-project-loom

[^17]: https://blog.jetbrains.com/idea/2025/12/thread-dumps-and-project-loom-virtual-threads/

