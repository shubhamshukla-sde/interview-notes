<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# Design URL Shortener (like Bitly)?

Sure, let me first clarify the problem and then walk you through my approach.

## 1. Interview-Style Opening

"Designing a URL shortener like Bitly is a classic system design question. It’s deceptively simple: map a long string to a short one. However, the real challenge lies in designing it to scale for billions of clicks, handling collisions, and ensuring low latency redirection.

I'll start by clarifying the functional and non-functional requirements, then move to the high-level architecture, database schema, and finally dive deep into the 'secret sauce'—the ID generation algorithm."

## 2. Problem Understanding and Clarification

We are building a service like **Bit.ly** or **TinyURL**.

**Functional Requirements:**

1. **Shorten:** Given a long URL, generate a unique, short alias (e.g., `bit.ly/5gWq`).
2. **Redirect:** Given a short alias, redirect the user to the original long URL.
3. **Custom Alias (Optional):** Allow users to pick `bit.ly/my-blog`.
4. **Expiry (Optional):** Links should expire after a default time (e.g., 5 years).

**Non-Functional Requirements:**

1. **Read-Heavy:** The ratio of reads (redirects) to writes (creation) is likely 100:1.
2. **Low Latency:** Redirection should be fast (< 10ms), as it's in the critical path of user experience.
3. **High Availability:** The service cannot go down; otherwise, links across the internet break.

**Back-of-the-Envelope Math (Capacity Planning):**

* **Writes:** 100M new URLs per month = ~40 writes/sec.
* **Reads:** 100M * 100 = 10B reads per month = ~4,000 reads/sec.
* **Storage:** If each URL mapping is 500 bytes and we store 5 years of data:
    * 100M * 12 * 5 = 6 Billion URLs.
    * 6 Billion * 500 Bytes = 3 TB of storage. (A single database instance can likely handle this, but sharding might be needed for throughput).


## 3. High-Level Approach

The core logic is the **ID Generation**. We need a unique, short string.

1. **Hashing (MD5/SHA256):** Hash the Long URL.
    * *Problem:* Collisions. MD5 produces a 128-bit string (too long). Truncating it (e.g., first 7 chars) causes collisions.
2. **Base62 Counter (Recommended):** Use a distributed unique ID generator (like Snowflake or a DB auto-increment) to get an integer ID (e.g., `1000000`) and convert it to Base62 (`aB3`). This guarantees uniqueness and shortness.

## 4. Visual Explanation (Mermaid-First, Mandatory)

```mermaid
flowchart TD
    subgraph "Write Path (Shorten)"
        User1["User"] -->|POST /shorten| API["API Service"]
        API -->|Get Unique ID| KGS["Key Generation Service<br>(Token Service)"]
        KGS -->|Return ID: 1001| API
        API -->|Convert 1001 -> 'g7'| Logic["Base62 Encoder"]
        Logic --> DB_W[("Database (Primary)")]
        DB_W -->|Save: {id: g7, url: google.com}| DB_W
    end

    subgraph "Read Path (Redirect)"
        User2["User"] -->|GET /g7| LB["Load Balancer"]
        LB -->|Check Cache| Cache["Redis Cache"]
        Cache -->|Hit: google.com| User2
        Cache -.->|Miss| ReadAPI["API Service"]
        ReadAPI -->|Select * where short='g7'| DB_R[("Database (Replica)")]
        DB_R -->|Return Long URL| ReadAPI
        ReadAPI -->|302 Redirect| User2
    end
    
    style KGS fill:#f9f,stroke:#333
    style Cache fill:#bbf,stroke:#333
```

**Explanation:**

* **KGS (Key Generation Service):** To avoid collision checks on every write, we pre-generate IDs or use a specialized counter service.
* **Caching:** Since reads dominate writes (100:1), a Redis cache is mandatory. It will serve 99% of traffic, protecting the DB.


## 5. Java Code (Production-Quality)

This code demonstrates the core **Base62 Encoding** logic, which is the heart of the shortener.

```java
import org.springframework.stereotype.Service;
import org.springframework.data.redis.core.StringRedisTemplate;
import java.util.concurrent.TimeUnit;

@Service
public class UrlShortenerService {

    private static final String BASE62 = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    private static final long BASE = 62;
    
    // Counter simulation (In production, use Zookeeper/Database Sequence)
    private long globalCounter = 1000000000L; 

    private final StringRedisTemplate redisTemplate;
    
    public UrlShortenerService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    // 1. Encode: Long ID -> Base62 String
    public String encode(long id) {
        StringBuilder sb = new StringBuilder();
        while (id > 0) {
            int remainder = (int) (id % BASE);
            sb.append(BASE62.charAt(remainder));
            id /= BASE;
        }
        return sb.reverse().toString();
    }

    // 2. Shorten Logic
    public String shortenUrl(String originalUrl) {
        long uniqueId = getNextId(); // Get unique ID from distributed counter
        String shortCode = encode(uniqueId);
        
        // Save to DB (omitted) and Cache
        redisTemplate.opsForValue().set(shortCode, originalUrl, 24, TimeUnit.HOURS);
        
        return "http://bit.ly/" + shortCode;
    }

    // 3. Resolve Logic
    public String getOriginalUrl(String shortCode) {
        // Look in Cache first
        String cachedUrl = redisTemplate.opsForValue().get(shortCode);
        if (cachedUrl != null) {
            return cachedUrl;
        }
        
        // Fallback to DB (Simulated)
        // String dbUrl = repository.findByShortCode(shortCode);
        // return dbUrl;
        throw new RuntimeException("URL not found");
    }

    private synchronized long getNextId() {
        return globalCounter++;
    }
}
```


## 6. Code Walkthrough (Line-by-Line)

* **`BASE62` String:** This defines our alphabet. `0-9` + `A-Z` + `a-z` = 62 characters. This allows us to represent 3.5 trillion URLs with just 7 characters (`62^7`).
* **`encode` Method:** This is a standard base conversion algorithm (Base 10 to Base 62). It repeatedly takes the modulo to find the character index.
* **`getNextId`:** In a real distributed system, `synchronized` won't cut it. We would use a Database Sequence (Postgres `SEQUENCE`), a Redis `INCR` command, or a specialized service like Twitter Snowflake to generate this ID.
* **Redis Caching:** We cache the `shortCode -> originalUrl` mapping immediately. This ensures the subsequent user click is served in milliseconds.


## 7. How I Would Explain This to the Interviewer

"The core engineering decision here is choosing between **Online Hashing** (MD5) and **Offline ID Generation** (Base62).

I chose **Base62 ID Generation** because it guarantees uniqueness without collision checks.
If we used MD5, we'd have to handle collisions (when two URLs hash to the same substring), which requires 'check-then-write' database logic and slows down writes.

With Base62, we simply treat every URL as a number.
ID 1 = `a`
ID 1000 = `qi`
ID 1,000,000 = `4c92`

The system then becomes a simple wrapper around a **Distributed ID Generator**. For the database, I’d choose a NoSQL store like **DynamoDB** or **Cassandra** because this is a simple Key-Value lookup problem, and those databases scale write throughput linearly."

## 8. Edge Cases and Follow-Up Questions

**Q: How do you handle 301 vs 302 Redirects?**

* *A:* "I would default to **302 (Temporary Redirect)**.
    * **301 (Permanent):** The browser caches the redirection locally. This saves server load but **breaks analytics** because the user never hits our server again for that link.
    * **302 (Temporary):** The browser always pings our server. This creates higher load but allows us to track click stats accurately."

**Q: What if the ID counter runs out?**

* *A:* "With 7 characters in Base62, we have `62^7` = ~3.5 Trillion combinations. Even at 1,000 requests per second, it would take ~100 years to exhaust. We are safe."

**Q: How do you prevent users from guessing sequential URLs?**

* *A:* "Sequential IDs (abc, abd, abe) are a security risk (Enumeration Attack). To fix this, we can introduce a random 'salt' or use a non-sequential ID generator (like Hashids) that still maps to an integer but looks random."


## 9. Optimization and Trade-offs

| Strategy | Pros | Cons |
| :-- | :-- | :-- |
| **Random ID + DB Check** | Secure, unguessable URLs | High latency (Collision checks on write) |
| **Base62 Counter** | Extremely fast, collision-proof | Predictable IDs (Security risk) |
| **Pre-generated Keys (KGS)** | Fastest write speed | Complex architecture (Need a Key-DB) |

**Optimization:** "To optimize writes, I would use **Pre-generated Keys**. We can have an offline worker generate 10 million unique 7-char keys and store them in a 'Unused Keys' table. When a write comes in, we simply pop one key. This moves all computation offline."

## 10. Real-World Application and Engineering Methodology

**Use Case: Marketing Campaigns (SMS)**

* **Scenario:** A bank sends 10M SMS messages: `bank.com/offer123`.
* **Constraint:** SMS has a 160 char limit. Long URLs consume the budget.
* **Implementation:** We built a private shortener.
* **Challenge:** The "Thundering Herd". 10M users click the link within 5 minutes of the SMS blast.
* **Solution:** We pre-warmed the **Redis Cache** with the specific campaign links *before* sending the SMS. This ensured 100% cache hit ratio, and the Database saw zero load during the spike.
<span style="display:none">[^1][^10][^11][^12][^13][^14][^15][^2][^3][^4][^5][^6][^7][^8][^9]</span>

<div align="center">⁂</div>

[^1]: https://blog.algomaster.io/p/design-a-url-shortener

[^2]: https://www.hellointerview.com/learn/system-design/problem-breakdowns/bitly

[^3]: https://www.youtube.com/watch?v=qSJAvd5Mgio

[^4]: https://www.designgurus.io/blog/url-shortening

[^5]: https://www.geeksforgeeks.org/system-design/system-design-url-shortening-service/

[^6]: https://procodebase.com/article/efficient-database-design-for-url-shorteners

[^7]: https://shortenworld.com/blog/the-role-of-base62-encoding-in-url-shortening-algorithms

[^8]: https://systemdesignschool.io/problems/url-shortener/solution

[^9]: https://www.linkedin.com/posts/jayant-aggarwal-418910248_remember-those-short-tinyurl-urls-ever-activity-7417197682689003520-Xj6j

[^10]: https://www.linkedin.com/pulse/building-url-shortener-using-hash-functions-base62-conversion-singh-y01oc

[^11]: https://www.systemdesignhandbook.com/guides/design-bitly/

[^12]: https://bytebytego.com/courses/system-design-interview/design-a-url-shortener

[^13]: https://www.kerstner.at/2012/07/shortening-strings-using-base-62-encoding/

[^14]: https://www.youtube.com/watch?v=xFeWVugaouk

[^15]: https://leetcode.com/discuss/general-discussion/6153958/HLD-Design-URL-Shortener/

