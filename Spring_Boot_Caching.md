# Spring Boot Caching

## Why Caching

Caching stores the result of expensive work (queries, remote calls, computations) so repeated requests are served cheaply. Spring provides a **cache abstraction** — a consistent annotation model over many providers — so you can swap the backing store without changing business code.

## Enabling Caching

Add `@EnableCaching` to a configuration class (Boot auto-configures a `CacheManager` from the classpath). Without a dedicated provider, Boot uses a simple `ConcurrentHashMap`-backed cache — fine for tests, not for production.

```java
@Configuration
@EnableCaching
public class CacheConfig {}
```

## Core Annotations

* `@Cacheable` — return the cached value if present; otherwise invoke the method and store the result.
* `@CachePut` — always invoke the method and update the cache (use for writes/updates).
* `@CacheEvict` — remove one entry or clear a cache (`allEntries = true`).
* `@Caching` — group multiple cache operations on one method.
* `@CacheConfig` — share cache names/config at the class level.

```java
@Service
public class ProductService {

    @Cacheable(cacheNames = "products", key = "#id")
    public Product findById(Long id) {
        return repository.findById(id).orElseThrow();
    }

    @CachePut(cacheNames = "products", key = "#product.id")
    public Product update(Product product) {
        return repository.save(product);
    }

    @CacheEvict(cacheNames = "products", key = "#id")
    public void delete(Long id) {
        repository.deleteById(id);
    }
}
```

## How It Works

`@EnableCaching` registers a proxy-based `CacheInterceptor` (Spring AOP). On a `@Cacheable` call the interceptor computes a key, asks the `CacheManager` for the named `Cache`, and returns a hit or proceeds and stores the miss. Because it is proxy-based, the **self-invocation** limitation applies — internal calls bypass caching.

## Key Generation

The default key is derived from the method arguments. Customize with a SpEL `key`, or a `KeyGenerator` bean for complex cases:

* `key = "#id"` — a single argument.
* `key = "#user.id + '-' + #region"` — composite key.
* `keyGenerator = "myKeyGen"` — delegate to a bean.

## Conditional Caching

* `condition = "#id > 0"` — only cache when the condition holds (evaluated before invocation).
* `unless = "#result == null"` — skip caching based on the result (evaluated after).
* `sync = true` — lock so only one thread computes a missing key (guards against cache stampede on a single node).

## Cache Providers

* **Simple** (`ConcurrentMapCacheManager`) — default, in-memory, no eviction/TTL.
* **Caffeine** — high-performance local cache with size/TTL eviction; configure via `spring.cache.caffeine.spec`.
* **Redis** — distributed cache shared across instances; per-cache TTLs via `RedisCacheConfiguration`.
* **JCache (JSR-107)** — Ehcache, Hazelcast, Infinispan through a standard SPI.

```yaml
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=10000,expireAfterWrite=5m
```

## TTL and Eviction

The core annotations do not express TTL; configure expiry on the provider (Caffeine `spec`, `RedisCacheConfiguration.entryTtl(...)`, Ehcache XML). Choose an eviction policy (size, time, or reference based) that matches access patterns and memory budget.

## Common Pitfalls

* **Self-invocation** — internal calls skip the cache proxy.
* **Caching `null`/errors** — guard with `unless = "#result == null"`.
* **Mutable cached objects** — callers can mutate shared instances; cache immutable copies/DTOs.
* **Key collisions** — overlapping keys across methods sharing a cache name return wrong data.
* **Stampede** — many concurrent misses recompute at once; use `sync = true` or a provider that coalesces loads.
* **Distributed staleness** — with Redis, evictions must be coordinated; set sensible TTLs.

## Interview Q&A

**Q: What is the difference between `@Cacheable` and `@CachePut`?**
A: `@Cacheable` short-circuits the method on a hit and only stores on a miss; `@CachePut` always runs the method and refreshes the cache — used to keep the cache consistent on updates.

**Q: How is the cache key computed, and how do you customize it?**
A: By default from the method arguments via a `KeyGenerator`. Customize with a SpEL `key` expression or a named `KeyGenerator` bean.

**Q: Why might `@Cacheable` be ignored?**
A: Proxy-based caching means self-invocation bypasses it; also check that `@EnableCaching` is present and the method is a public method on a Spring bean.

**Q: How do you cache in a distributed system so all instances share state?**
A: Use a distributed provider like Redis so entries live outside the JVM; set TTLs and coordinate evictions, accepting eventual consistency.

**Q: How do you prevent a cache stampede?**
A: Use `sync = true` (single-flight per key on a node) and/or a provider that coalesces concurrent loads; pre-warm hot keys and stagger TTLs.

**Q: Where do TTL and size limits come from?**
A: From the provider configuration (Caffeine `spec`, `RedisCacheConfiguration`, Ehcache), not the Spring cache annotations, which only describe *when* to cache/evict.

## Interview Notes

* Explain the cache abstraction and how `@Cacheable`/`@CachePut`/`@CacheEvict` differ.
* Know that caching is proxy-based, so self-invocation and `null`-result handling matter.
* Be able to contrast a local cache (Caffeine) with a distributed cache (Redis) and where TTL is configured.
