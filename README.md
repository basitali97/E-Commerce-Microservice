# 🛒 E-Commerce Microservices Platform

A production-ready, distributed e-commerce backend built with **Java Spring Boot** following microservices architecture principles. Designed for scalability, fault tolerance, and independent deployability.

---

## 📌 Service Repository Index

| Service | Description | Repository |
|---|---|---|
| API Gateway | Single entry point, routing & load balancing | [https://github.com/basitali97/API-Gateway.git](#) |
| Service Discovery | Eureka-based service registry | [Link](#) |
| Product Service | Product catalogue & inventory management | [Link](#) |
| User Auth Service | Authentication & authorization (JWT + OAuth2) | [Link](#) |
| Payment Service | Stripe payment processing & webhooks | [Link](#) |
| Email Service | Event-driven email notifications via Kafka | [Link](#) |

---

## 🏗️ Architecture Overview



Client
│
▼
API Gateway  ◄──── Service Discovery (Eureka)
│                      ▲
│            (All services register here)
├──► Product Service
│         │
├──► Auth Service
│         │
├──► Payment Service
│         │ (Kafka Event: payment.success)
│         ▼
└──► Email Service  ◄── Kafka Consumer


### Request Flow

Client sends request
│
▼
API Gateway (validates JWT, routes request, load balances)
│
▼
Target Microservice processes request
│
├── Reads from Redis Cache (if available) ──► Returns instantly
│
└── Falls back to Database ──► Updates Redis Cache ──► Returns response
│
▼ (on payment success)
Payment Service publishes event to Kafka Topic
│
▼
Email Service consumes event ──► Sends confirmation email




---

## 🔧 Services — Deep Dive

### 1. API Gateway
> Single entry point for all client requests.

- Routes incoming requests to appropriate microservices
- Enforces **JWT validation** before forwarding requests — no unauthenticated request reaches any service
- **Client-side load balancing** distributes traffic across multiple instances of a service
- Integrates with Eureka for dynamic service discovery (no hardcoded URLs)

---

### 2. Service Discovery (Eureka)
> The phonebook of the system.

- All microservices register themselves on startup with their host and port
- API Gateway queries Eureka to resolve service locations dynamically
- If a service instance goes down, Eureka deregisters it — traffic automatically stops routing to it
- Enables horizontal scaling: spin up a new instance and it auto-registers

---

### 3. Product Service
> Manages the product catalogue and inventory.

- CRUD operations for products and categories
- **Redis caching** on product reads:
  - Cache hit → response in **10–20ms**
  - Cache miss → DB query → cache updated → response in ~2–4s
  - Result: **100x latency improvement** on repeated reads
- Cache invalidation on product update/delete ensures data consistency

---

### 4. User Auth Service
> Handles identity — who you are and what you can do.

- **User registration & login** with encrypted password storage (BCrypt)
- Issues **JWT tokens** on successful login — stateless, no server-side sessions
- **OAuth2 integration** for third-party login (Google / GitHub)
- Role-based access control (RBAC): `ROLE_USER`, `ROLE_ADMIN`
- All tokens validated at the Gateway — services themselves are never hit by unauthenticated requests

---

### 5. Payment Service
> Handles all money movement securely.

- Integrates with **Stripe** for payment intent creation and processing
- **Webhook handling**: Stripe asynchronously notifies the service on payment success/failure
- On successful payment:
  - Updates order status in DB
  - Publishes a `payment.success` event to **Kafka**
- On failure:
  - Publishes `payment.failed` event
  - Order rolled back / marked failed
- Payment logic is fully isolated — failures here don't crash other services

---

### 6. Email Service
> Fully decoupled notification service.

- **Kafka consumer** listening on `payment.success` and `payment.failed` topics
- Sends transactional emails (order confirmation, payment failure alerts)
- Completely async — Payment Service doesn't wait for email to be sent
- If Email Service is down, Kafka retains the message — email sends once service recovers (no lost notifications)
- Zero coupling: Email Service can be replaced or scaled without touching Payment Service

---

## ⚙️ Tech Stack

| Category | Technology |
|---|---|
| Language | Java |
| Framework | Spring Boot, Spring Cloud |
| Service Discovery | Netflix Eureka |
| API Gateway | Spring Cloud Gateway |
| Messaging | Apache Kafka |
| Caching | Redis |
| Auth | JWT, OAuth2 |
| Payment | Stripe |
| Database | MySQL / PostgreSQL |
| Logging | SLF4J + structured logging |
| Build Tool | Maven |

---

## 🚀 Running Locally

### Prerequisites
- Java 17+
- Docker & Docker Compose
- Maven

### Infrastructure Setup (Kafka, Redis, MySQL)

```bash
docker-compose up -d
```

### Start Services (in this order)

```bash
# 1. Service Discovery first
cd service-discovery && mvn spring-boot:run

# 2. API Gateway
cd api-gateway && mvn spring-boot:run

# 3. All other services (order doesn't matter after above)
cd auth-service && mvn spring-boot:run
cd product-service && mvn spring-boot:run
cd payment-service && mvn spring-boot:run
cd email-service && mvn spring-boot:run
```

> ⚠️ Always start Eureka (Service Discovery) first — other services register on startup and will fail to register if Eureka isn't running.

---

## 🔐 Environment Variables

Each service requires a `.env` or `application.yml` config. Key variables:

```env
# Auth Service
JWT_SECRET=your_secret_key
OAUTH2_CLIENT_ID=your_google_client_id
OAUTH2_CLIENT_SECRET=your_google_client_secret

# Payment Service
STRIPE_SECRET_KEY=sk_test_xxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxx

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
```

---

## 📊 Key Design Decisions

| Decision | Reason |
|---|---|
| Kafka over REST for notifications | Email Service failure shouldn't block payment flow |
| Redis caching on Product Service | Product reads are high-frequency; DB hits were a bottleneck |
| JWT validated at Gateway | Services stay stateless; no repeated auth logic per service |
| Stripe webhooks over polling | Async confirmation is more reliable than polling for payment status |
| Eureka for service discovery | No hardcoded URLs; new instances auto-register for horizontal scaling |

---

## 📈 Performance Highlights

- **Product read latency**: Reduced from 2–4s → 10–20ms with Redis (100x improvement)
- **Decoupled services**: Email Service downtime has zero impact on order/payment flow
- **Load balanced**: Multiple instances of any service can run simultaneously behind the Gateway

---

## 🙋 Author

**Basit Ali Siddiqui**
[LinkedIn](https://linkedin.com/in/basitali97) • [GitHub](https://github.com/basitali97) • basitali971999@gmail.com
