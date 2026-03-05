# Complete Java Spring Boot Microservices Interview Mastery Guide

---

## MODULE 1 — Spring Boot Core (Deep Dive)

---

### 1. Auto-Configuration — How it really works

**Concept:**
Spring Boot uses `@EnableAutoConfiguration` which triggers `spring.factories` (or `AutoConfiguration.imports` in Spring Boot 3.x) to load hundreds of configuration classes conditionally.

**How to answer in interview:**
> *"When Spring Boot starts, it scans the classpath and uses `@Conditional` annotations like `@ConditionalOnClass`, `@ConditionalOnMissingBean` to decide which beans to auto-configure. For example, if HikariCP is on the classpath and no `DataSource` bean is manually defined, Spring Boot auto-configures one. I've overridden this in production by defining my own `DataSource` bean with custom pool settings."*

**Real-world follow-up Q:** *"How did you customize auto-configuration in your project?"*
- Define your own bean → Spring backs off
- Use `application.yml` properties to tune defaults
- Use `@SpringBootTest` with `exclude` to disable specific auto-configs in tests

---

### 2. Bean Lifecycle & Scopes

**Critical Scopes:**
```
Singleton  → One instance per Spring context (default, most used in prod)
Prototype  → New instance every time requested
Request    → One per HTTP request (web apps)
Session    → One per HTTP session
```

**Lifecycle hooks (must know):**
```java
@PostConstruct   // runs after dependency injection — use for init logic
@PreDestroy      // runs before bean is destroyed — use for cleanup
InitializingBean / DisposableBean  // interface-based alternative
```

**Real-world scenario answer:**
> *"In production, I used `@PostConstruct` to warm up a cache after the service started, so the first user request wouldn't hit the database. I also used `@PreDestroy` to gracefully close external connections like Kafka producers during shutdown."*

---

### 3. Dependency Injection — Constructor vs Field vs Setter

**Interview trap:** Most devs say "I use `@Autowired`" — that's a red flag.

**Production best practice:**
```java
// BEST — Constructor injection (recommended)
@Service
public class OrderService {
    private final PaymentService paymentService; // immutable, testable

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

**How to answer:**
> *"I always use constructor injection in production because it makes dependencies explicit, supports immutability with `final`, and makes unit testing easier since you can pass mocks directly without needing a Spring context. Field injection with `@Autowired` hides dependencies and makes testing harder."*

---

## MODULE 2 — REST API Design (Production Level)

---

### 4. Exception Handling at Scale

**The wrong way (junior):** try-catch in every controller method.

**The right way (production):**
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.404, ex.getMessage(), Instant.now()
        );
        return ResponseEntity.status(404).body(error);
    }

    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(ConstraintViolationException ex) {
        // extract field-level errors and return structured response
    }
}
```

**How to answer:**
> *"In production, I implemented a `@RestControllerAdvice` as a single exception handling layer. Each exception maps to a specific HTTP status and returns a structured `ErrorResponse` with timestamp, message, and error code. This keeps controllers clean and gives clients consistent error contracts. I also added correlation IDs to error responses so we could trace errors in our ELK stack."*

---

### 5. Idempotency in REST APIs

**Most asked real-world scenario:**
> *"How do you prevent duplicate order creation if the client retries?"*

**Answer:**
```java
// Client sends Idempotency-Key header
// Server checks if key was already processed

@PostMapping("/orders")
public ResponseEntity<Order> createOrder(
    @RequestHeader("Idempotency-Key") String idempotencyKey,
    @RequestBody OrderRequest request) {

    if (idempotencyStore.exists(idempotencyKey)) {
        return ResponseEntity.ok(idempotencyStore.get(idempotencyKey));
    }

    Order order = orderService.create(request);
    idempotencyStore.save(idempotencyKey, order, Duration.ofHours(24));
    return ResponseEntity.status(201).body(order);
}
```

> *"We stored idempotency keys in Redis with a 24-hour TTL. If the same key came in again, we returned the cached response instead of re-processing. This was critical for our payment service where double charges had to be prevented."*

---

## MODULE 3 — Microservices Patterns (Most Critical Section)

---

### 6. Circuit Breaker Pattern (Resilience4j)

**Why it exists:** Without it, a failing downstream service causes thread exhaustion in your service → cascading failure.

**States to know:**
```
CLOSED    → Normal operation, requests pass through
OPEN      → Failure threshold crossed, requests fail fast
HALF-OPEN → Test requests allowed to check if service recovered
```

**Production config:**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        slidingWindowSize: 10
        failureRateThreshold: 50        # open if 50% of 10 requests fail
        waitDurationInOpenState: 10000  # wait 10s before half-open
        permittedNumberOfCallsInHalfOpenState: 3
```

**How to answer:**
> *"In our order service, calls to the payment service were wrapped with a circuit breaker. When the payment service went down during a deployment, the circuit opened after 5 failures, and we returned a fallback response — 'Payment service temporarily unavailable, retry in a few seconds.' This prevented our order service threads from being blocked, keeping the rest of the system healthy."*

---

### 7. Saga Pattern — Distributed Transactions

**The core problem:** In microservices, you can't use database ACID transactions across services.

**Two types:**

**Choreography-based Saga (event-driven):**
```
Order Service → emits OrderCreated event
  → Payment Service listens → processes payment → emits PaymentProcessed
    → Inventory Service listens → reserves stock → emits StockReserved
      → Order Service listens → marks order CONFIRMED

On failure: each service emits compensating events to undo previous steps
```

**Orchestration-based Saga:**
```
Saga Orchestrator controls the flow:
  → calls Payment Service
  → calls Inventory Service
  → if any step fails, orchestrator triggers compensating transactions
```

**How to answer:**
> *"We used the Choreography Saga for our order flow via Kafka events. Each service listened for domain events and published its own. On payment failure, the Payment Service emitted a `PaymentFailed` event, which the Order Service consumed to revert the order to `CANCELLED`. For complex flows with many services, I'd prefer Orchestration using a tool like Temporal or Axon to have a clear view of the flow state."*

---

### 8. API Gateway — What it really does

**Production responsibilities of API Gateway:**
```
✅ Request routing
✅ Authentication/Authorization (JWT validation)
✅ Rate limiting
✅ Request/Response transformation
✅ SSL termination
✅ Load balancing
✅ Logging & tracing (inject correlation IDs)
```

**Common interview question:** *"What's the difference between API Gateway and Load Balancer?"*

| API Gateway | Load Balancer |
|---|---|
| Layer 7 (application) | Layer 4 (network) or Layer 7 |
| Understands HTTP, headers, paths | Distributes TCP/HTTP connections |
| Business logic (auth, rate limit) | Pure traffic distribution |
| One entry point for all services | Works at infrastructure level |

**How to answer:**
> *"In our system, Spring Cloud Gateway acted as the single entry point. It validated JWTs and enriched requests with user context before routing. We added rate limiting using Redis to prevent abuse — 100 requests per minute per user. The load balancer (AWS ALB) sat in front of the gateway to distribute traffic across gateway instances."*

---

### 9. Service Discovery (Eureka)

**The problem it solves:** In containers/cloud, service IPs change constantly — you can't hardcode URLs.

**Flow:**
```
1. Each service registers itself with Eureka on startup
   (sends: serviceId, host, port, health check URL)

2. Client service asks Eureka: "Where is payment-service?"
   Eureka returns list of healthy instances

3. Ribbon/Spring Cloud LoadBalancer picks one instance (round-robin by default)
```

**Production concern answer:**
> *"We ran Eureka in a multi-zone setup with peer replication between zones for high availability. One issue we faced was stale registry entries when a service crashed without deregistering. We tuned the eviction timer and also implemented health checks so unhealthy instances were removed faster. In Kubernetes environments, we moved to Kubernetes-native discovery, so Eureka became less relevant there."*

---

### 10. Inter-Service Communication — Sync vs Async

**Synchronous (Feign Client):**
```java
@FeignClient(name = "payment-service", fallback = PaymentFallback.class)
public interface PaymentClient {
    @PostMapping("/payments")
    PaymentResponse processPayment(@RequestBody PaymentRequest request);
}
```

**Asynchronous (Kafka):**
```java
// Producer
kafkaTemplate.send("order-events", new OrderCreatedEvent(orderId, userId, amount));

// Consumer
@KafkaListener(topics = "order-events", groupId = "payment-group")
public void handleOrderCreated(OrderCreatedEvent event) {
    paymentService.processPayment(event);
}
```

**How to answer the "when to use which" question:**
> *"I use synchronous communication (Feign) when the caller needs an immediate response — like checking inventory before confirming an order. I use async (Kafka) for operations that can happen in the background — like sending notifications or updating analytics. Async improves resilience since the producer doesn't care if the consumer is temporarily down. In production, we moved most non-critical flows to Kafka to decouple services and improve throughput."*

---

## MODULE 4 — Data & Database Patterns

---

### 11. Database Per Service Pattern

**Core principle:** Each microservice owns its data. No shared databases.

**Interview follow-up:** *"How do you query data that spans multiple services?"*

**Answer options:**
```
1. API Composition   → call multiple services, aggregate in memory
2. CQRS + Read Model → maintain a denormalized read DB synced via events
3. GraphQL Federation → federated queries across services
```

**How to answer:**
> *"We followed database-per-service strictly. For cross-service queries — like fetching an order with user details and product info — we used the API Composition pattern in our BFF (Backend for Frontend). For reporting use cases that needed joins across services, we maintained a separate read-optimized database that was kept in sync via Kafka events using the CQRS pattern."*

---

### 12. N+1 Problem & JPA Optimization

**The trap:** Fetching a list of orders and lazily loading each order's items → 1 query for orders + N queries for items.

**Solutions:**
```java
// Option 1: JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.userId = :userId")
List<Order> findOrdersWithItems(@Param("userId") Long userId);

// Option 2: @EntityGraph
@EntityGraph(attributePaths = {"items", "items.product"})
List<Order> findByUserId(Long userId);

// Option 3: Batch fetching
@BatchSize(size = 20)
```

**How to answer:**
> *"We hit this in production — our order list API was firing 50+ queries for a single request. I identified it using Hibernate's statistics logging and P6Spy. Fixed it using `JOIN FETCH` for mandatory associations and `@BatchSize` for optional lazy ones. Response time dropped from 800ms to 60ms."*

---

### 13. Optimistic vs Pessimistic Locking

**Optimistic (best for low contention):**
```java
@Entity
public class Product {
    @Version
    private Long version; // auto-incremented, throws OptimisticLockException on conflict
}
```

**Pessimistic (for high contention — e.g., seat booking, flash sales):**
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Product findByIdForUpdate(@Param("id") Long id);
// Translates to SELECT ... FOR UPDATE — database-level row lock
```

**How to answer:**
> *"For our flash sale feature where thousands of users hit the same product simultaneously, we used pessimistic locking with `SELECT FOR UPDATE` to prevent overselling. For regular cart updates, we used optimistic locking with `@Version` and retried on `OptimisticLockException`. The key is matching the locking strategy to your contention level."*

---

## MODULE 5 — Security (Production Grade)

---

### 14. JWT Authentication Flow

**Complete flow to explain:**
```
1. User logs in → Auth Service validates credentials
2. Auth Service issues: Access Token (15 min) + Refresh Token (7 days)
3. Client sends Access Token in Authorization: Bearer <token> header
4. API Gateway validates JWT signature using public key (RS256)
5. Gateway injects user context (userId, roles) into request headers
6. Downstream services read user context — they NEVER re-validate the token
7. When Access Token expires, client uses Refresh Token to get new Access Token
8. On logout: Refresh Token is invalidated in Redis blacklist
```

**How to answer:**
> *"We used RS256 (asymmetric) signing — Auth Service holds the private key, all other services have the public key. This means services can validate tokens without calling the Auth Service, reducing latency. We stored refresh tokens in Redis with TTL. On logout or suspicious activity, we added the token's `jti` claim to a Redis blacklist. The API Gateway checked this blacklist on every request."*

---

### 15. OAuth2 + OIDC (Real Production Setup)

**Key roles:**
```
Resource Owner  → the user
Client          → your application
Auth Server     → Keycloak / Okta / AWS Cognito
Resource Server → your Spring Boot APIs
```

**How to answer:**
> *"In our B2B product, we integrated Keycloak as our Auth Server. Spring Boot services were configured as Resource Servers using `spring-security-oauth2-resource-server`. The API Gateway validated the JWT using Keycloak's JWKS endpoint (public keys). We used scopes and roles for authorization — `SCOPE_read:orders` allowed reading, `ROLE_ADMIN` allowed management operations."*

---

## MODULE 6 — Observability (Most Underrated Topic)

---

### 16. Distributed Tracing (The Big One)

**The problem:** A request flows through 5 services. Something is slow. Which service is the culprit?

**Solution — The Three Pillars:**
```
Logs   → What happened (ELK Stack: Elasticsearch, Logstash, Kibana)
Metrics → How the system is performing (Prometheus + Grafana)
Traces  → How a request flowed through services (Zipkin / Jaeger)
```

**Trace propagation:**
```
Request comes in → API Gateway generates Trace ID + Span ID
→ Passes them as headers (X-B3-TraceId, X-B3-SpanId) to downstream services
→ Each service creates child spans and logs them
→ Zipkin collects all spans and visualizes the full trace
```

**Spring Boot setup:**
```yaml
# Spring Boot 3.x (Micrometer Tracing)
management:
  tracing:
    sampling:
      probability: 0.1   # trace 10% of requests in production (cost management)
```

**How to answer:**
> *"When a production issue occurred — a checkout taking 5 seconds — I used Jaeger to pull the trace. I could see the entire flow: Gateway → Order Service → Inventory Service → Payment Service. The trace showed the Payment Service was taking 4 seconds because it was doing a synchronous call to a fraud detection API that was under load. We added a timeout and circuit breaker, which resolved it."*

---

### 17. Health Checks & Graceful Shutdown

**Production health check setup:**
```yaml
management:
  endpoint:
    health:
      show-details: always
  health:
    db:
      enabled: true
    redis:
      enabled: true
    kafka:
      enabled: true
```

**Graceful shutdown (critical for zero-downtime deployments):**
```yaml
server:
  shutdown: graceful         # finish in-flight requests before stopping
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # give 30s for requests to complete
```

**How to answer:**
> *"In Kubernetes, we configured liveness and readiness probes separately. The readiness probe checked DB and Kafka connectivity — if they were down, the pod was removed from the load balancer but not restarted. The liveness probe only checked if the app itself was running. We enabled graceful shutdown to ensure in-flight requests weren't dropped during rolling deployments."*

---

## MODULE 7 — Performance & Scalability

---

### 18. Caching Strategy (Multi-Level)

**Levels:**
```
L1: In-process cache   → Caffeine (within JVM, ultra-fast, limited size)
L2: Distributed cache  → Redis (shared across service instances)
L3: CDN cache          → CloudFront (for static/semi-static API responses)
```

**Cache-Aside Pattern (most used):**
```java
public Product getProduct(Long id) {
    // 1. Check Redis
    Product cached = redisTemplate.opsForValue().get("product:" + id);
    if (cached != null) return cached;

    // 2. Cache miss — hit DB
    Product product = productRepository.findById(id).orElseThrow();

    // 3. Populate cache with TTL
    redisTemplate.opsForValue().set("product:" + id, product, Duration.ofMinutes(30));
    return product;
}
```

**Cache invalidation strategies (always asked):**
```
TTL-based       → auto-expire after X seconds (simple, eventual consistency)
Event-driven    → on product update, publish event → service invalidates cache
Write-through   → update cache AND DB together on every write
```

**How to answer:**
> *"We used a two-level cache — Caffeine as L1 for hot product data (1000 items, 5-min TTL) and Redis as L2. On a cache miss in Caffeine, we checked Redis before hitting the DB. For invalidation, we published a `ProductUpdated` Kafka event, and each service instance consumed it to evict from both L1 and L2. This handled the cache consistency problem across multiple instances."*

---

### 19. Connection Pooling (HikariCP)

**Why it matters:** DB connections are expensive. Without pooling, every request creates a new connection → system collapses under load.

**Production tuning:**
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20        # don't set too high — DB has limits
      minimum-idle: 5
      connection-timeout: 3000     # fail fast — don't queue requests too long
      idle-timeout: 600000
      max-lifetime: 1800000        # recycle connections to prevent stale connections
      leak-detection-threshold: 60000  # warn if connection held > 60s (bug detection)
```

**How to answer:**
> *"We had a production incident where our pool size was set to the default of 10, but under peak load, all 10 connections were taken and new requests timed out. We analyzed our DB's `max_connections`, subtracted connections used by other services, and allocated a per-service pool size accordingly. We also enabled leak detection, which caught a bug where a connection wasn't being released inside a conditional code path."*

---

### 20. Async Processing with CompletableFuture

**Real-world use case — parallel service calls:**
```java
public OrderSummary getOrderSummary(Long orderId) {
    // Call 3 services IN PARALLEL instead of sequentially
    CompletableFuture<Order> orderFuture =
        CompletableFuture.supplyAsync(() -> orderService.getOrder(orderId), executor);

    CompletableFuture<User> userFuture =
        CompletableFuture.supplyAsync(() -> userService.getUser(orderId), executor);

    CompletableFuture<List<Product>> productFuture =
        CompletableFuture.supplyAsync(() -> productService.getProducts(orderId), executor);

    // Wait for all and combine
    return CompletableFuture.allOf(orderFuture, userFuture, productFuture)
        .thenApply(v -> new OrderSummary(
            orderFuture.join(),
            userFuture.join(),
            productFuture.join()
        )).get(5, TimeUnit.SECONDS); // timeout to prevent hanging
}
```

**How to answer:**
> *"Our order summary API was calling 3 services sequentially — 300ms + 200ms + 250ms = 750ms total. I refactored it to use `CompletableFuture.allOf()` with a dedicated thread pool executor. All 3 calls fired in parallel and the total time became the max of the three — ~300ms. I always use a custom executor with bounded queue size rather than the common pool to prevent thread starvation."*

---

## MODULE 8 — Kafka (Deep Production Knowledge)

---

### 21. Kafka Core Concepts + Interview Questions

**Must-know concepts:**
```
Topic      → category of messages (e.g., "order-events")
Partition  → ordered, immutable log within a topic (enables parallelism)
Consumer Group → group of consumers sharing partition load
Offset     → position of a message in a partition (consumer tracks this)
```

**Most asked question:** *"How do you ensure exactly-once processing?"*
```
At-most-once   → fire and forget, may lose messages
At-least-once  → ack after processing, may get duplicates (default)
Exactly-once   → idempotent producers + transactional consumers (Kafka Streams)
```

**How to answer:**
> *"For our payment events, losing or duplicating a message was unacceptable. We used idempotent Kafka producers (`enable.idempotence=true`) and made our consumer idempotent by checking if a payment with the given `eventId` already existed in the DB before processing. This gave us effectively-once semantics without the overhead of full Kafka transactions. For reporting pipelines where duplicates were less critical, we used at-least-once."*

---

### 22. Kafka Consumer Lag & Rebalancing

**Consumer lag = messages produced - messages consumed** — a key production health metric.

**Rebalancing problem:**
> When a consumer joins or leaves a group, Kafka reassigns partitions. During rebalancing, consumption pauses — this causes latency spikes.

**How to answer:**
> *"We monitored consumer lag via Prometheus and set alerts when lag exceeded 10,000 messages. During high load, we scaled out consumer instances — since we had 6 partitions, we could have up to 6 consumers in parallel. To reduce rebalancing impact, we used `CooperativeStickyAssignor` (incremental rebalancing) instead of the default eager protocol, which stopped all consumers during reassignment."*

---

## MODULE 9 — Deployment & Production Patterns

---

### 23. Deployment Strategies

**Blue-Green:**
```
Blue  = current production (v1)
Green = new version (v2)
→ Switch load balancer from Blue to Green instantly
→ Rollback = switch back to Blue
→ Problem: requires double infrastructure
```

**Canary:**
```
→ Route 5% of traffic to new version
→ Monitor error rates, latency
→ Gradually increase to 100% or rollback
→ Used by Netflix, Uber for safe rollouts
```

**Rolling:**
```
→ Replace instances one by one
→ Always some instances running old + new version simultaneously
→ Requires backward-compatible APIs and DB migrations
```

**How to answer:**
> *"We used Canary deployments via Kubernetes Argo Rollouts. We'd push to 5% of traffic, monitor Prometheus metrics (error rate, P99 latency) for 10 minutes, and auto-promote or auto-rollback based on thresholds. For database schema changes, we always followed expand-contract migrations — add the new column, deploy code that handles both, then remove the old column — to ensure zero-downtime compatibility."*

---

### 24. The 12-Factor App (Always Asked)

**The most critical factors for interviews:**
```
Config         → Store config in environment, NEVER in code
Logs           → Treat as event streams, never write to files
Stateless      → No session state in the app; use Redis for shared state
Disposability  → Fast startup, graceful shutdown
Dev/Prod Parity → Keep environments as similar as possible
```

**How to answer:**
> *"We followed 12-factor principles strictly. All config — DB URLs, API keys, feature flags — came from environment variables or Spring Cloud Config, never hardcoded. Our services were stateless — any instance could handle any request because session data lived in Redis. This made horizontal scaling trivial; we just added more pods in Kubernetes."*

---

## BONUS — How to Structure ANY Interview Answer

Use the **STAR-T format** for scenario questions:

```
Situation  → "We had a production system processing 10,000 orders/day..."
Task       → "The payment service was causing cascading failures..."
Action     → "I implemented a Circuit Breaker with Resilience4j..."
Result     → "Failures were isolated, uptime went from 99.5% to 99.95%..."
Trade-off  → "The trade-off was slightly stale fallback responses,
              which was acceptable given the business requirement..."
```

**The "Trade-off" part is what separates senior engineers from juniors.** Always mention what you gave up to gain something.

---

## Quick Reference — Cheat Sheet

| Topic | Key Answer Point |
|---|---|
| Auto-config | `@ConditionalOnClass` + override by defining your own bean |
| Circuit Breaker | 3 states, prevents cascading failures, use fallback |
| Saga | Choreography (events) vs Orchestration (central coordinator) |
| JWT | RS256, Gateway validates, services trust, Redis blacklist |
| Caching | Cache-aside + event-driven invalidation |
| Kafka | Idempotent consumer = at-least-once + deduplication |
| Tracing | TraceId propagated across services, sample in prod |
| Locking | Optimistic for low contention, Pessimistic for flash sales |
| Deployment | Canary + expand-contract DB migrations |
| Performance | Parallel calls with CompletableFuture, tune HikariCP |

Master these with your own project experience woven in, and you'll consistently outperform other candidates.
