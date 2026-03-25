# 🏗️ System Design - Complete Learning Guide

A comprehensive repository for learning system design concepts, patterns, and real-world case studies. This guide covers everything from fundamentals to advanced distributed systems and AI/ML system design.

> ⚠️ **Note**: This repository is a work in progress. Currently, only the **[Caching](./04-caching/)** section is complete with detailed content. Other sections are planned and will be added over time.

---

## ✅ Available Content

| Section | Status | Description |
|---------|--------|-------------|
| [**04-caching/**](./04-caching/) | ✅ Complete | 12 detailed files covering concepts + 7 Real-World Scenarios |

---

## 📚 Table of Contents

1. [Fundamentals](#1-fundamentals)
2. [Scalability](#2-scalability)
3. [Databases](#3-databases)
4. [Caching](#4-caching) ✅
5. [Message Queues & Event-Driven Architecture](#5-message-queues--event-driven-architecture)
6. [Microservices](#6-microservices)
7. [API Design](#7-api-design)
8. [Security](#8-security)
9. [Monitoring & Observability](#9-monitoring--observability)
10. [AI/ML Systems](#10-aiml-systems)
11. [Case Studies](#11-case-studies)

---

## 1. Fundamentals

### 📁 [01-fundamentals/](./01-fundamentals/) *(Coming Soon)*

| Topic | Description |
|-------|-------------|
| Client-Server Architecture | Basic request-response model |
| Network Protocols | TCP/IP, HTTP, HTTPS, WebSockets |
| DNS & Domain Resolution | How domain names resolve to IPs |
| Latency vs Throughput | Performance metrics |
| CAP Theorem | Consistency, Availability, Partition Tolerance |
| ACID vs BASE | Transaction properties |
| Consistent Hashing | Distributed data partitioning |

---

## 2. Scalability

### 📁 [02-scalability/](./02-scalability/) *(Coming Soon)*

| Topic | Description |
|-------|-------------|
| Horizontal vs Vertical Scaling | Scale out vs scale up |
| Load Balancing | Distributing traffic across servers |
| Load Balancer Algorithms | Round Robin, Least Connections, IP Hash |
| Reverse Proxy | Nginx, HAProxy patterns |
| CDN (Content Delivery Network) | Edge caching and content distribution |
| Database Replication | Master-Slave, Master-Master |
| Database Sharding | Horizontal partitioning strategies |
| Auto Scaling | Dynamic resource allocation |

---

## 3. Databases

### 📁 [03-databases/](./03-databases/) *(Coming Soon)*

| Topic | Description |
|-------|-------------|
| SQL vs NoSQL | When to use which |
| Relational Databases | MySQL, PostgreSQL, Oracle |
| Document Databases | MongoDB, CouchDB |
| Key-Value Stores | Redis, DynamoDB, Memcached |
| Wide-Column Stores | Cassandra, HBase |
| Graph Databases | Neo4j, Amazon Neptune |
| Time-Series Databases | InfluxDB, TimescaleDB |
| Vector Databases | Pinecone, Weaviate, Milvus (for AI/ML) |
| Database Indexing | B-Tree, Hash, Bitmap indexes |
| Query Optimization | Execution plans, query tuning |

---

## 4. Caching ✅

### 📁 [04-caching/](./04-caching/) - **AVAILABLE NOW**

**Complete guide with 12 detailed files + 7 Real-World Scenarios**

| Topic | Description |
|-------|-------------|
| What is Caching? | Introduction, terminology, principles of locality |
| Why Caching Matters | Performance benefits, cost savings |
| Types of Caches | Client-side, CDN, Application, Database |
| Caching Strategies | Cache-Aside, Read-Through, Write-Through, Write-Behind |
| Cache Eviction Policies | LRU, LFU, FIFO, TTL |
| Caching Technologies | Redis vs Memcached comparison |
| Distributed Caching | Consistent Hashing, Replication, Sharding |
| Cache Invalidation | TTL-based, Event-based, Version-based |
| Cache Stampede Prevention | Locking, Coalescing, Probabilistic Expiration |
| Real-World Examples | Twitter, Netflix, Amazon case studies |
| Best Practices | Do's, Don'ts, Common mistakes |
| **Real-World Scenarios** | 7 detailed scenarios with solutions |

**Real-World Scenarios Include:**
1. Cache Stampede (50k req/s hot key expires)
2. Why Not Cache Everything?
3. Cache-Database Consistency
4. Local vs Distributed Cache
5. Cold Start Problem
6. Cache Cost Optimization
7. Cache Penetration (non-existent IDs)

---

## 5. Message Queues & Event-Driven Architecture

### 📁 [05-messaging/](./05-messaging/) *(Coming Soon)*

| Topic | Description |
|-------|-------------|
| Message Queue Fundamentals | Async communication patterns |
| Apache Kafka | Distributed streaming platform |
| RabbitMQ | Traditional message broker |
| Amazon SQS/SNS | AWS messaging services |
| Event Sourcing | Storing state as events |
| CQRS | Command Query Responsibility Segregation |
| Pub/Sub Pattern | Publisher-Subscriber model |
| Dead Letter Queues | Handling failed messages |
| Idempotency | Handling duplicate messages |

---

## 6. Microservices

### 📁 [06-microservices/](./06-microservices/) *(Coming Soon)*

| Topic | Description |
|-------|-------------|
| Monolith vs Microservices | Architecture comparison |
| Service Discovery | Consul, Eureka, Kubernetes DNS |
| API Gateway | Kong, AWS API Gateway, Zuul |
| Circuit Breaker Pattern | Handling service failures |
| Saga Pattern | Distributed transactions |
| Service Mesh | Istio, Linkerd |
| Sidecar Pattern | Auxiliary containers |
| Strangler Fig Pattern | Migrating from monolith |
| Domain-Driven Design | Bounded contexts, aggregates |
| 12-Factor App | Cloud-native principles |

---

## 7. API Design

### 📁 [07-api-design/](./07-api-design/) *(Coming Soon)*

| Topic | Description |
|-------|-------------|
| REST API Design | RESTful principles and best practices |
| GraphQL | Query language for APIs |
| gRPC | High-performance RPC framework |
| API Versioning | Managing API changes |
| Rate Limiting | Token bucket, leaky bucket |
| Pagination | Offset, cursor-based pagination |
| API Authentication | OAuth, JWT, API Keys |
| Idempotent APIs | Safe retry mechanisms |
| Long Polling vs WebSockets | Real-time communication |

---

## 8. Security

### 📁 [08-security/](./08-security/) *(Coming Soon)*

| Topic | Description |
|-------|-------------|
| Authentication vs Authorization | Identity and access control |
| OAuth 2.0 & OpenID Connect | Modern auth protocols |
| JWT (JSON Web Tokens) | Token-based authentication |
| SSL/TLS | Transport layer security |
| Encryption at Rest | Data encryption strategies |
| SQL Injection Prevention | Input validation |
| XSS & CSRF Protection | Web security |
| Secrets Management | Vault, AWS Secrets Manager |
| Zero Trust Architecture | Never trust, always verify |

---

## 9. Monitoring & Observability

### 📁 [09-observability/](./09-observability/) *(Coming Soon)*

| Topic | Description |
|-------|-------------|
| Three Pillars of Observability | Logs, Metrics, Traces |
| Logging Best Practices | Structured logging, log aggregation |
| Metrics & Monitoring | Prometheus, Grafana, CloudWatch |
| Distributed Tracing | Jaeger, Zipkin, X-Ray |
| Alerting Strategies | PagerDuty, alert fatigue |
| SLI, SLO, SLA | Service level objectives |
| Health Checks | Liveness, readiness probes |
| Chaos Engineering | Testing system resilience |

---

## 10. AI/ML Systems

### 📁 [10-ai-ml-systems/](./10-ai-ml-systems/) *(Coming Soon)*

| Topic | Description |
|-------|-------------|
| **ML Fundamentals** | |
| ML Pipeline Architecture | Data → Training → Serving |
| Model Serving | Inference at scale, latency vs throughput |
| Feature Stores | Centralized feature management |
| **LLM Systems** | |
| LLM Architecture | Transformers, context windows, tokenization |
| LLM Serving | vLLM, batching, quantization, GPU optimization |
| Fine-tuning vs Prompting | When to use each approach |
| **RAG (Retrieval Augmented Generation)** | |
| RAG Architecture | Embedding → Retrieval → Generation |
| Vector Databases | Pinecone, Weaviate, pgvector |
| **AI Agents** | |
| Agent Architecture | Planning, tool use, memory |
| Tool Use & Function Calling | Structured outputs, tool schemas |
| **MLOps** | |
| ML Monitoring | Model drift, data drift |
| Cost Optimization | GPU utilization, inference costs |

---

## 11. Case Studies

### 📁 [11-case-studies/](./11-case-studies/) *(Coming Soon)*

Real-world system design examples:

| System | Key Concepts |
|--------|--------------|
| **Classic Systems** | |
| Design URL Shortener | Hashing, Base62, Read-heavy |
| Design Twitter/X | Fan-out, Timeline, Caching |
| Design Instagram | Image storage, CDN, Feed |
| Design WhatsApp | Real-time messaging, E2E encryption |
| Design YouTube | Video streaming, Transcoding |
| Design Netflix | Streaming, Recommendations |
| Design Uber | Location services, Matching |
| Design Dropbox | File sync, Chunking |
| Design Rate Limiter | Token bucket, Sliding window |
| Design Notification System | Push, Email, SMS |
| **AI/ML Case Studies** | |
| Design ChatGPT | LLM serving, conversation memory |
| Design Recommendation System | Collaborative filtering, embeddings |
| Design AI Code Assistant | RAG, code embeddings |

---

## 🗂️ Directory Structure

```
system-design/
├── README.md
├── 01-fundamentals/        (Coming Soon)
├── 02-scalability/         (Coming Soon)
├── 03-databases/           (Coming Soon)
├── 04-caching/             ✅ AVAILABLE
├── 05-messaging/           (Coming Soon)
├── 06-microservices/       (Coming Soon)
├── 07-api-design/          (Coming Soon)
├── 08-security/            (Coming Soon)
├── 09-observability/       (Coming Soon)
├── 10-ai-ml-systems/       (Coming Soon)
├── 11-case-studies/        (Coming Soon)
└── assets/
    └── diagrams/
```

---

## 🚀 How to Use This Repository

1. **Start with Caching**: The [04-caching/](./04-caching/) section is complete and ready to use
2. **Read Concepts First**: Go through files 01-11 to understand the fundamentals
3. **Practice with Scenarios**: Use the 7 real-world scenarios to test your understanding
4. **Check Back for Updates**: More sections will be added over time

---

## 📖 Recommended Learning Path

```
Week 1-2: Caching (Available Now!)
         - Read all 12 concept files
         - Work through 7 real-world scenarios
         
Week 3+:  Check back for new content!
```

---

## 🔗 Additional Resources

### Books
- "Designing Data-Intensive Applications" by Martin Kleppmann
- "System Design Interview" by Alex Xu
- "Building Microservices" by Sam Newman
- "Designing Machine Learning Systems" by Chip Huyen

### Online Resources
- [System Design Primer](https://github.com/donnemartin/system-design-primer)
- [ML System Design](https://github.com/chiphuyen/machine-learning-systems-design)
- [AWS Architecture Center](https://aws.amazon.com/architecture/)

---

## 🤝 Contributing

Feel free to add new topics, improve existing content, or add more case studies!

---

*Happy Learning! 🎯*