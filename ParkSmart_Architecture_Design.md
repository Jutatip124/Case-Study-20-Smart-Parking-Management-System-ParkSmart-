# ParkSmart — Smart Parking Management System
## Software Architecture Design Document

---

## 1. Overview

**ParkSmart** is a city-wide smart parking management system deployed across **120 car parks** and **4,000 on-street parking bays**. IoT sensors detect space occupancy in real time. Drivers use a mobile app to find available spaces, navigate to them, and pay digitally. Parking wardens use a dedicated app for enforcement. Car park operators and the city transport authority access dashboards for management and policy decisions.

The primary city goal is to **reduce driver search time from 15–20 minutes to under 5 minutes** and **eliminate revenue loss** from unpaid parking.

---

## 2. Organisation & Stakeholders

| Stakeholder | Role | Interaction with System |
|---|---|---|
| **Drivers** | End users searching and paying for parking | Mobile App (find, navigate, pay) |
| **Parking Wardens** | Enforcement officers | Warden App (licence plate lookup, session verification) |
| **Car Park Operators** | Private and city-owned site managers | Operator Dashboard (occupancy, revenue) |
| **City Transport Authority** | Policy and pricing governance | Admin Portal (city-wide usage, dynamic pricing) |
| **Maintenance Team** | Sensor and infrastructure maintenance | Alert System (sensor fault notifications) |
| **IoT Sensors** | Hardware embedded in each space | Sensor Gateway (report occupancy status) |

---

## 3. Current Situation & Problems

| # | Problem | Impact |
|---|---|---|
| P1 | Drivers cannot know space availability before arriving | 15–20 min wasted searching |
| P2 | Payment machines break frequently | Revenue loss; drivers leave without paying |
| P3 | Wardens rely on physical windscreen tickets | Inefficient enforcement; revenue leakage |
| P4 | Sensor data arrives in different formats and frequencies | No unified data processing |
| P5 | No historical occupancy data | Cannot inform pricing or infrastructure planning |
| P6 | No enforcement technology for on-street parking | Unpaid parking is widespread |

---

## 4. Functional Requirements

| ID | Requirement |
|---|---|
| FR1 | Sensors **report space occupancy** (free / occupied) in real time |
| FR2 | Drivers can **view available spaces** on a map and navigate to them |
| FR3 | Drivers can **start, manage, and stop a parking session** and pay through the app |
| FR4 | Wardens can **look up a licence plate** to verify a valid parking session |
| FR5 | Car park operators can view **live occupancy and revenue dashboards** |
| FR6 | The city transport authority can **apply dynamic pricing** based on occupancy thresholds |
| FR7 | The maintenance team receives **automated alerts** when sensors malfunction or go offline |
| FR8 | The system stores **historical occupancy and revenue data** for analytics and planning |
| FR9 | The system handles **sensor failure gracefully**, showing appropriate space status |
| FR10 | The system supports **multiple payment methods** (card, wallet, in-app) |

---

## 5. Quality Attributes

Five key quality attributes are selected based on the system's context:

### QA1 — Availability
> The system must be available at all times, including during peak hours and when individual sensors or services fail.

- **Scenario:** A sensor gateway goes offline. Drivers must still be able to view other available spaces and pay for parking without interruption.
- **Target:** 99.9% uptime (≤ 8.7 hours downtime/year)

### QA2 — Scalability
> The system must handle thousands of concurrent sensor updates and user requests without degraded performance.

- **Scenario:** On a busy Saturday, 4,000 sensors emit updates simultaneously while 3,000 drivers query the app.
- **Target:** Support 10,000+ sensor events/min and 5,000 concurrent users

### QA3 — Security
> Driver location, payment data, and session records must be protected from unauthorized access and tampering.

- **Scenario:** A warden app is compromised; the attacker should not be able to access driver personal or payment data.
- **Target:** Role-based access control; PCI-DSS compliance for payments; GDPR compliance for personal data

### QA4 — Reliability / Fault Tolerance
> The system must behave predictably when sensors fail, connectivity is lost (underground car parks), or a microservice crashes.

- **Scenario:** A sensor in an underground car park goes offline due to connectivity loss. The system should fall back to the last known state and mark the space as "status unknown" rather than incorrectly showing it as available.
- **Target:** No silent failures; all faults are detected, logged, and acted upon

### QA5 — Performance
> Real-time occupancy updates must reach the driver app within a defined time limit to be actionable.

- **Scenario:** A driver is navigating to a space; occupancy changes must be reflected in the app within 5 seconds.
- **Target:** End-to-end latency from sensor event to driver app update ≤ 5 seconds (p95)

### QA6 — Modifiability
> The system must accommodate new sensor types, new pricing models, and new payment providers without major rework.

- **Scenario:** The city adds a new brand of sensor that reports in a different data format. Only the Sensor Adapter needs to change.
- **Target:** Adding a new sensor type or payment provider requires changes in ≤ 2 components

---

## 6. Architecture Model

### 6.1 Use Case Diagram

```
<img width="14380" height="5388" alt="image" src="https://github.com/user-attachments/assets/51c00088-3577-45f8-9e0d-ce5e09572486" />

```

**Extended Use Cases (include relationships):**

- `Pay for Parking` **includes** `Start Parking Session`
- `Stop Parking Session` **includes** `Pay for Parking` (finalise amount)
- `Verify Parking Session` **includes** `Lookup Licence Plate`
- `Set Dynamic Pricing` **extends** `View City-Wide Usage Report`

---

### 6.2 Component Diagram

The system is built on a **microservices architecture** with an **event-driven backbone** (message broker).

```
<img width="6288" height="4504" alt="image" src="https://github.com/user-attachments/assets/3781881c-a857-4e5e-ae1f-5cd98a1d02b1" />

```

**Key Data Stores:**

| Store | Technology | Used By |
|---|---|---|
| Space State DB | Redis (in-memory) | Space Service — fast read/write for live occupancy |
| Session DB | PostgreSQL | Session Service — ACID transactions for payments |
| Payment Records | PostgreSQL (encrypted) | Payment Service — PCI-DSS compliant |
| Event Store | Kafka (retained) | Message Broker — replay and auditing |
| Analytics DB | TimescaleDB (time-series) | Analytics Service — historical trends |
| Sensor Metadata | PostgreSQL | Sensor Ingestion — sensor registry and health |

---

## 7. Architecture Style Decision

**Selected Style: Microservices + Event-Driven Architecture**

| Option | Pros | Cons | Decision |
|---|---|---|---|
| Monolith | Simple deployment | Cannot scale individual components; one failure can crash all | ❌ Rejected |
| Layered / N-tier | Clear separation | Tight coupling; hard to scale sensor ingestion independently | ❌ Rejected |
| Microservices + Event-Driven | Independent scaling, fault isolation, loose coupling | Higher operational complexity | ✅ **Selected** |
| Serverless | Auto-scaling | Cold starts unacceptable for real-time; vendor lock-in | ❌ Rejected |

**Justification:**
- Sensor ingestion must scale independently from payment processing.
- An event-driven approach via Kafka allows the system to buffer sensor events during traffic spikes and ensures no data is lost even if a downstream service is temporarily unavailable.
- Microservices allow the city to deploy updates to the Pricing Service without touching the Driver App or Sensor pipeline.

---

## 8. Selected Tactics for Quality Attributes

### QA1 — Availability Tactics

| Tactic | Implementation |
|---|---|
| **Active Redundancy** | All critical services run in multiple pods (minimum 2) behind a load balancer; Kafka with 3 brokers |
| **Heartbeat / Health Checks** | Kubernetes liveness and readiness probes for all services; dead pod is automatically restarted |
| **Sensor State Fallback** | If a sensor has not reported within a configurable threshold (e.g., 5 min), its space is marked `UNKNOWN` rather than `AVAILABLE` — prevents drivers navigating to a potentially occupied space |
| **Local Buffering at Gateway** | Sensor gateways (installed per car park) store events locally and retry on reconnect — handles underground connectivity gaps |

### QA2 — Scalability Tactics

| Tactic | Implementation |
|---|---|
| **Horizontal Scaling** | Kubernetes auto-scales Sensor Ingestion pods based on Kafka consumer lag |
| **Message Broker (Kafka)** | Decouples producers (sensors) from consumers (Space Service, Analytics) — acts as a shock absorber during traffic spikes |
| **Read Replicas** | Redis read replicas for Space State DB — distributes high read load from driver app map queries |
| **CDN for Map Tiles** | Static map tile assets served via CDN to reduce latency for driver app |

### QA3 — Security Tactics

| Tactic | Implementation |
|---|---|
| **Role-Based Access Control (RBAC)** | 5 roles: Driver, Warden, Operator, City, Maintenance — enforced at API Gateway via JWT claims |
| **Encrypt Data in Transit & at Rest** | TLS 1.3 for all API calls; AES-256 encryption for payment data and driver PII at rest |
| **Payment Tokenisation** | Driver's card details never stored in ParkSmart — tokenised via PCI-DSS compliant payment gateway (e.g., Stripe) |
| **Minimal Data Exposure** | Warden app only returns "VALID / INVALID session" for a plate — never exposes driver name, address, or payment method |
| **API Gateway Rate Limiting** | Protects against DDoS and credential stuffing on licence plate lookup endpoint |

### QA4 — Reliability / Fault Tolerance Tactics

| Tactic | Implementation |
|---|---|
| **Circuit Breaker** | If Payment Gateway fails, Session Service opens a circuit breaker — queues payment for retry rather than failing the user session |
| **Sensor Fault Detection** | Sensor Ingestion Service tracks last-seen timestamp per sensor; emits a `sensor.fault` event to Alert Service when threshold exceeded |
| **Idempotent Messages** | Each sensor event carries a unique event ID — Kafka consumers deduplicate to avoid double-processing occupancy changes |
| **Graceful Degradation** | If Analytics Service is down, occupancy display still works — analytics is read-path only, not on the critical path |

### QA5 — Performance Tactics

| Tactic | Implementation |
|---|---|
| **In-Memory Cache** | Redis stores live occupancy state — O(1) lookup for driver app map queries, no DB round-trip |
| **WebSocket Push** | Space Service pushes occupancy changes to connected driver app clients via WebSocket — avoids polling |
| **Event Stream Processing** | Kafka Streams processes sensor events and updates Redis in near-real-time (target < 2 sec pipeline) |
| **Async Payments** | Payment confirmation is processed asynchronously — driver is shown "processing" immediately; session is active within 1–2 sec |

### QA6 — Modifiability Tactics

| Tactic | Implementation |
|---|---|
| **Adapter Pattern** | Sensor Adapter Layer is a plug-in module per sensor format/vendor — adding a new sensor type requires only a new adapter class |
| **Strategy Pattern** | Pricing Service uses configurable pricing strategies — city can add a new pricing rule without changing other services |
| **Published Interface** | All inter-service communication uses versioned REST or Kafka event schemas — backward-compatible changes do not break consumers |

---

## 9. Technical Decisions

| Decision | Choice | Rationale |
|---|---|---|
| **Sensor Protocol** | MQTT (primary) + HTTP (fallback) | MQTT is lightweight and suitable for low-bandwidth IoT sensors; HTTP fallback for sensors in areas with stable connectivity |
| **Message Broker** | Apache Kafka | Durable, high-throughput, supports replay for analytics backfill; proven for IoT event streams |
| **Live State Store** | Redis | Sub-millisecond reads for occupancy queries; supports pub/sub for WebSocket push |
| **Relational DB** | PostgreSQL | ACID compliance required for session and payment records |
| **Time-Series DB** | TimescaleDB (PostgreSQL extension) | Occupancy and revenue time-series data with SQL interface for reporting |
| **Container Orchestration** | Kubernetes | Auto-scaling, self-healing, rolling deployments for zero-downtime updates |
| **API Style** | REST (external APIs) + Kafka events (internal) | REST for external mobile/web clients; events for internal async processing |
| **Authentication** | OAuth 2.0 + JWT | Industry standard; short-lived tokens; supports multiple client types (app, web, warden device) |
| **Payment Integration** | Third-party gateway (Stripe/similar) | Offloads PCI-DSS scope; supports multiple payment methods |
| **Maps** | OpenStreetMap + custom tile server or Google Maps API | Show real-time space availability overlaid on a city map |
| **Sensor Failure Policy** | Mark as `UNKNOWN` after 5 min no heartbeat | Conservative approach — prevents routing drivers to potentially occupied spaces; operators are alerted |
| **Underground Connectivity** | Local sensor gateway with store-and-forward | Buffers events locally in car park gateway; replays when connection restored; Kafka deduplicates replayed events |

---

## 10. Risk & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Sensor data flood during peak hours | High | Medium | Kafka absorbs spikes; Sensor Ingestion auto-scales |
| Payment gateway outage | Medium | High | Circuit breaker; session remains active; retry queue |
| Underground connectivity loss | High | Medium | Local gateway buffering + store-and-forward |
| Sensor battery/hardware failure | High | Low–Medium | Heartbeat monitoring; `UNKNOWN` state; maintenance alert |
| Driver data breach | Low | High | Encryption at rest/transit; tokenised payments; RBAC |
| Dynamic pricing abuse (price too high) | Low | Medium | City admin sets min/max pricing caps enforced by Pricing Service |

---

## 11. Summary

ParkSmart uses a **microservices + event-driven architecture** centred on an **Apache Kafka event backbone**. Key design decisions:

- **Sensor Layer:** MQTT-based sensors report to per-car-park gateways that handle offline buffering, ensuring underground car parks retain data during connectivity loss.
- **Ingestion Layer:** A pluggable Sensor Adapter normalises heterogeneous sensor formats before publishing to Kafka.
- **Processing Layer:** Kafka Streams updates Redis (live occupancy) and TimescaleDB (history) asynchronously and with low latency.
- **API Layer:** An API Gateway enforces RBAC and routes requests to the appropriate microservice — Driver App, Warden App, Operator Dashboard, and City Admin Portal are all served through this gateway.
- **Security:** Driver personal and payment data is minimised, encrypted, and tokenised. Role-based access ensures wardens see only what they need for enforcement.
- **Fault Tolerance:** Sensor failures are detected via heartbeat; spaces are marked `UNKNOWN` (not `AVAILABLE`) on failure. Service failures are isolated using circuit breakers and Kubernetes self-healing.

This architecture meets the six quality attributes identified: **Availability, Scalability, Security, Reliability, Performance, and Modifiability**, while providing a clear path for the city to extend ParkSmart with new sensor vendors, pricing models, and payment methods in the future.

---

*End of Document*
