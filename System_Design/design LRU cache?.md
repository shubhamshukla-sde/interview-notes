<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# design LRU cache?

Sure, let me first clarify the problem and then walk you through my approach.

## 1. Interview-Style Opening

"Designing an **LRU (Least Recently Used) Cache** is a classic problem because it tests our ability to combine multiple data structures to achieve optimal time complexity.

The goal is to store key-value pairs with a fixed capacity. When we exceed this capacity, we must evict the item that hasn't been accessed for the longest time.

The industry standard for this is achieving **O(1)** time complexity for both `get()` and `put()` operations. To do this, a simple Array or LinkedList isn't enough. We need a combination of a **HashMap** (for fast lookup) and a **Doubly Linked List** (for fast ordering and removal)."

## 2. Problem Understanding and Clarification

**Functional Requirements:**

1. **Capacity:** The cache has a fixed size (e.g., 3 items).
2. **Get(key):** Return value if exists, else -1. *Side effect:* This item becomes the "Most Recently Used" (move to front).
3. **Put(key, value):** Update if exists, insert if new. If capacity is full, remove the "Least Recently Used" (tail) item. *Side effect:* This item becomes the "Most Recently Used" (move to front).

**Constraints:**

* **Time Complexity:** O(1) for both operations.
* **Space Complexity:** O(N) where N is the capacity.


## 3. High-Level Approach (The "HashMap + DLL" Pattern)

Why do we need two data structures?

1. **HashMap (`Map<Key, Node>`):** Allows us to find the address of a node in the linked list instantly. Without this, finding a node in a list takes O(N).
2. **Doubly Linked List:** Allows us to remove a node from the middle and add it to the front in O(1) time. A Singly Linked List takes O(N) to remove because you need a pointer to the *previous* node.

**Visualizing the Structure:**

* **Head:** Represents the Most Recently Used (MRU).
* **Tail:** Represents the Least Recently Used (LRU).
* **Eviction:** We simply remove the node at `Tail.prev` and delete it from the HashMap.


## 4. Visual Explanation (Mermaid-First, Mandatory)

```mermaid
flowchart TD
    subgraph "LRU Cache Components"
        Map["HashMap<br>{Key -> Node Address}"]
        
        subgraph "Doubly Linked List"
            Head[Head<br>(Dummy)] <--> NodeA["Node A<br>(MRU)"]
            NodeA <--> NodeB["Node B"]
            NodeB <--> NodeC["Node C<br>(LRU)"]
            NodeC <--> Tail[Tail<br>(Dummy)]
        end
    end

    Map -->|Key 'A'| NodeA
    Map -->|Key 'B'| NodeB
    Map -->|Key 'C'| NodeC

    note1[Get(B) Called:<br>1. Look up B in Map<br>2. Detach B from list<br>3. Move B to after Head]
    
    style Map fill:#f9f,stroke:#333
    style Head fill:#ccc,stroke:#333
    style Tail fill:#ccc,stroke:#333
```

**Why Dummy Nodes?**
Using `head` and `tail` as dummy (sentinel) nodes prevents null pointer exceptions. We never have to check `if (head == null)`. The list is never empty; it always has at least the two dummies.

## 5. Java Code (Production-Quality)

I will implement this from scratch using a custom `Node` class, as standard `java.util.LinkedList` doesn't expose the Node objects required for O(1) removal.

```java
import java.util.HashMap;
import java.util.Map;

class LRUCache {

    // 1. Internal Node Class
    private class Node {
        int key;
        int value;
        Node prev;
        Node next;

        Node(int key, int value) {
            this.key = key;
            this.value = value;
        }
    }

    private final int capacity;
    private final Map<Integer, Node> cacheMap;
    private final Node head;
    private final Node tail;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.cacheMap = new HashMap<>();
        
        // Initialize Dummy Nodes
        this.head = new Node(-1, -1);
        this.tail = new Node(-1, -1);
        
        // Connect Dummy Nodes
        this.head.next = this.tail;
        this.tail.prev = this.head;
    }

    public int get(int key) {
        if (!cacheMap.containsKey(key)) {
            return -1;
        }

        Node node = cacheMap.get(key);
        
        // CRITICAL: Move accessed node to front (MRU)
        removeNode(node);
        addToFront(node);
        
        return node.value;
    }

    public void put(int key, int value) {
        if (cacheMap.containsKey(key)) {
            // Update existing
            Node node = cacheMap.get(key);
            node.value = value;
            removeNode(node);
            addToFront(node);
        } else {
            // Insert new
            if (cacheMap.size() >= capacity) {
                // Evict LRU (node before tail)
                Node lruNode = tail.prev;
                removeNode(lruNode);
                cacheMap.remove(lruNode.key);
            }
            
            Node newNode = new Node(key, value);
            addToFront(newNode);
            cacheMap.put(key, newNode);
        }
    }

    // --- Helper Methods (O(1)) ---

    // Remove node from its current position
    private void removeNode(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    // Add node right after head
    private void addToFront(Node node) {
        node.next = head.next;
        node.prev = head;
        
        head.next.prev = node;
        head.next = node;
    }
}

// Test Harness
public class Main {
    public static void main(String[] args) {
        LRUCache lru = new LRUCache(2);
        lru.put(1, 1); // Cache: {1=1}
        lru.put(2, 2); // Cache: {2=2, 1=1}
        System.out.println(lru.get(1)); // Returns 1. Cache: {1=1, 2=2}
        lru.put(3, 3); // Evicts 2. Cache: {3=3, 1=1}
        System.out.println(lru.get(2)); // Returns -1 (not found)
    }
}
```


## 6. Code Walkthrough (Line-by-Line)

* `class Node`: We store the `key` inside the Node as well. Why? Because when we evict the LRU node from the linked list (at `tail.prev`), we need to know its key to remove it from the HashMap.
* `removeNode()`: This simply "unplugs" a node by connecting its neighbors to each other.
* `addToFront()`: This "plugs" a node in between `head` and `head.next`.
* `put()`:
    * If key exists: Update value, unplug, replug at front.
    * If key is new: Check capacity. If full, look at `tail.prev` (the LRU victim), unplug it, remove from map. Then plug in the new node at front.


## 7. How I Would Explain This to the Interviewer

"To implement an LRU cache efficiently, I use a composite data structure.
A **HashMap** gives me O(1) access to any cached item.
A **Doubly Linked List** maintains the order of access. I chose a *doubly* linked list because removing a node requires updating pointers of both the previous and next nodes. If I only had a singly linked list, I couldn't remove the 'tail' element in O(1) time because I wouldn't have a reference to the node *before* the tail.

In my implementation, the 'Head' of the list represents the Most Recently Used items, and the 'Tail' represents the Least Recently Used. Every time we `get` or `put` an item, we physically move its Node to the front of the list."

## 8. Edge Cases and Follow-Up Questions

**Q: Is this thread-safe?**

* *A:* "No. `HashMap` and standard pointer operations are not thread-safe. To make it thread-safe, I would use `ConcurrentHashMap` and wrap the linked list operations in a `ReentrantLock` (or use `Collections.synchronizedMap`)."

**Q: Can we use Java's built-in `LinkedHashMap`?**

* *A:* "Yes! In a real production application, I would just extend `LinkedHashMap` and override the `removeEldestEntry` method. This turns `LinkedHashMap` into an LRU cache with 5 lines of code. But for this interview, I implemented the internals to show I understand the pointers."


## 9. Optimization and Trade-offs

| Implementation | Time Complexity | Space Overhead | Pros | Cons |
| :-- | :-- | :-- | :-- | :-- |
| **Array / ArrayList** | O(N) Scan | Low | Cache locality (CPU friendly) | Slow eviction/search |
| **LinkedHashMap** | O(1) | Medium | Production ready, minimal code | "Black box" implementation |
| **Custom HashMap + DLL** | O(1) | High (2 pointers per node) | Full control, standard interview answer | Verbose code |

**Optimization:** "If memory is strictly constrained, the overhead of two reference pointers (next/prev) per entry might be too high (adding 16 bytes per entry on 64-bit JVM). In that case, we might switch to a **Clock Replacement Algorithm** (used in database buffer pools) which approximates LRU with O(1) space overhead but slightly less accuracy."

## 10. Real-World Application and Engineering Methodology

**Use Case: Redis Cache Eviction**

* **Scenario:** Redis is used as an LRU cache for session data.
* **Engineering:** Redis actually implements an *approximated* LRU. Instead of a massive linked list (which consumes too much RAM for billions of keys), Redis samples 5 random keys and evicts the oldest one. This provides "good enough" LRU behavior with zero pointer overhead.
* **My Choice:** For a local Java cache (e.g., caching parsed config files), I would use **Guava Cache** or **Caffeine**, which are high-performance implementations of the design I just showed.
<span style="display:none">[^1][^10][^11][^12][^13][^14][^15][^2][^3][^4][^5][^6][^7][^8][^9]</span>

<div align="center">⁂</div>

[^1]: https://www.youtube.com/watch?v=lZ5QuFLCVn0

[^2]: https://interviewing.io/questions/lru-cache

[^3]: https://www.geeksforgeeks.org/system-design/lru-cache-implementation/

[^4]: https://javarevisited.substack.com/p/how-to-design-lru-cache-on-system

[^5]: https://www.finalroundai.com/interview-questions/microsoft-system-design-lru

[^6]: https://www.linkedin.com/pulse/how-implement-lru-cache-using-hashmap-doubly-linked-list-singhal

[^7]: https://www.scaler.in/lru-cache-implementation/

[^8]: https://github.com/bediger4000/lru-cache

[^9]: https://www.linkedin.com/posts/dipalibohara_dsa-case-study-designed-an-lru-cache-activity-7395002108372086784-Lw4I

[^10]: https://www.scaler.com/topics/lru-cache-implementation/

[^11]: https://algomaster.io/learn/lld/design-lru-cache

[^12]: https://www.jointaro.com/interview-insights/google/how-would-you-implement-an-lru-cache-using-a-hashmap-and-a-doubly-linked-list/

[^13]: https://www.jointaro.com/interview-insights/amazon/design-and-implement-an-lru-cache-with-get-and-put-operations-in-o1-time/

[^14]: https://www.reddit.com/r/softwarearchitecture/comments/1o9xn6r/how_to_design_lru_cache_on_system_design_interview/

[^15]: https://www.enjoyalgorithms.com/blog/implement-least-recently-used-cache/

