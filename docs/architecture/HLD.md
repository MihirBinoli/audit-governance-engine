# High-Level Design (HLD)
## Enterprise Audit & Governance Workflow Engine

| Field        | Detail                          |
|--------------|---------------------------------|
| Version      | 1.0.0                           |
| Status       | Approved                        |
| Author       | [Your Name]                     |
| Last Updated | [Date]                          |

---

## 1. Purpose

This document defines the high-level architecture for the Enterprise Audit & Governance
Workflow Engine — a centralized platform for processing compliance documents through a
configurable multi-tier Maker-Checker approval system, with an automated SLA escalation
engine and real-time administrative notifications.

This document is intended for: engineering leads, architects, and evaluators reviewing
the system design before implementation begins.

---

## 2. Functional Requirements

### 2.1 Core Workflow
- Users submit compliance documents (audit submissions) with metadata and file attachments.
- Submissions flow through a configurable multi-tier approval chain:
  `MAKER → CHECKER_L1 → CHECKER_L2 → ADMIN`
- Each approver can: **Approve**, **Reject**, or **Request Revision**, with mandatory comments.
- Rejected submissions can be revised and resubmitted by the original Maker.
- Full, immutable audit trail is maintained for every state transition.

### 2.2 SLA & Escalation Engine
- Each approval tier has a configurable SLA duration (e.g., 24h for L1, 48h for L2), stored in the database — not hardcoded.
- A background scheduler polls for overdue approvals on a configurable interval.
- On breach: the escalation is logged, the responsible approver is flagged, and a notification is dispatched.
- Real-time push notifications are delivered to the admin dashboard via WebSockets on each escalation event.

### 2.3 User & Access Management
- Role-based access control (RBAC) with four roles: `MAKER`, `CHECKER_L1`, `CHECKER_L2`, `ADMIN`.
- Users are assigned one or more roles; roles carry a `tier_level` integer that drives workflow routing.
- Admins can configure SLA rules per document category via a dedicated API.

---

## 3. Non-Functional Requirements

| Category        | Requirement                                                                 |
|-----------------|-----------------------------------------------------------------------------|
| Performance     | API p95 response time < 200ms for standard CRUD; WS broadcast < 500ms      |
| Security        | JWT stateless auth, BCrypt hashing, HTTPS enforced, no secrets in logs      |
| Scalability     | Stateless backend; horizontal scaling ready; HikariCP connection pooling     |
| Maintainability | Clean Architecture; no business logic in controllers or repositories         |
| Observability   | Structured logging (SLF4J + Logback); Spring Actuator health/metrics exposed |
| Reliability     | All state transitions in DB transactions; idempotent scheduler on restart    |
| Auditability    | Every approval action writes an immutable `approval_actions` record          |

---

## 4. System Architecture Overview

The system is organized into five tiers, each with a distinct responsibility boundary.

```
┌─────────────────────────────────────────────────────────────────┐
│  CLIENT TIER                                                    │
│  Angular SPA  │  WebSocket Client  │  NgRx State Store          │
└───────────────────────────┬─────────────────────────────────────┘
                            │ HTTPS / WSS
┌───────────────────────────▼─────────────────────────────────────┐
│  SECURITY & API TIER                                            │
│  JWT Auth Filter  │  REST Controllers  │  WebSocket Endpoint    │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│  SERVICE / DOMAIN TIER                                          │
│  Submission Service  │  Approval Service  │  Notification Svc   │
│                      SLA Escalation Engine (@Scheduled)         │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│  DATA TIER                                                      │
│  Spring Data JPA Repositories  │  PostgreSQL  │  Flyway          │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│  INFRA / DEVOPS TIER                                            │
│  Docker  │  Kubernetes (local)  │  GitHub Actions  │  Actuator  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Component Descriptions

### 5.1 Angular SPA (Frontend)
- Built with Angular 17+ using standalone components.
- NgRx used for centralized state management of submissions and notification queue.
- Connects to the backend via REST (HTTP) for data operations and STOMP-over-SockJS for real-time events.
- Lazy-loaded feature modules: `auth`, `submissions`, `approvals`, `admin`.

### 5.2 Spring Boot Backend
- Single deployable Spring Boot application following Clean Architecture layering:
  - **Controller layer:** HTTP/WebSocket entry points, input validation, auth enforcement.
  - **Service layer:** All business logic, workflow state transitions, SLA calculations.
  - **Repository layer:** Spring Data JPA interfaces, no business logic.
  - **Domain layer:** JPA entities and enums representing the core model.
- RBAC enforced via Spring Security `@PreAuthorize` annotations on service methods.

### 5.3 SLA Escalation Engine
- Implemented as a Spring `@Scheduled` component running on a configurable cron interval (e.g., every 5 minutes).
- On each tick: queries `sla_tracking` for records where `sla_deadline < NOW()` and `is_escalated = false`.
- Updates the tracking record atomically and publishes an escalation event to the Notification Service.
- Idempotent by design: the `is_escalated` flag prevents duplicate escalations across restarts.

### 5.4 WebSocket Notification Service
- Uses Spring WebSocket with STOMP message broker.
- Admin-subscribed clients receive real-time escalation payloads on topic `/topic/escalations`.
- Notification payload contains: submission ID, title, breached tier, deadline, and escalation timestamp.

### 5.5 PostgreSQL Database
- Primary relational datastore.
- Schema managed exclusively by Flyway versioned migration scripts (never manual DDL).
- Connection pool managed by HikariCP with tuned min/max pool sizes.

---

## 6. Database Schema (ERD Summary)

Seven tables govern the full domain:

| Table               | Purpose                                                         |
|---------------------|-----------------------------------------------------------------|
| `users`             | System users with credentials and profile info                  |
| `roles`             | RBAC roles with a `tier_level` integer for workflow routing     |
| `user_roles`        | Many-to-many join between users and roles                       |
| `audit_submissions` | Core submission entity; holds workflow `status` and `current_tier` |
| `approval_actions`  | Immutable audit log of every approve/reject/revise action       |
| `sla_configs`       | Data-driven SLA rules keyed on `(document_category, tier_level)` |
| `sla_tracking`      | Per-submission, per-tier SLA state and escalation tracking      |

> Full ERD with column definitions is maintained alongside this document.

---

## 7. Workflow State Machine

The `status` column on `audit_submissions` follows this state machine:

```
         ┌──────────────────────────────────────┐
         │              [Maker]                 │
         │           DRAFT ──────► PENDING_L1   │
         │             ▲                │        │
         │             │ revise         │ approve│
         │     REVISION_REQUESTED ◄────┘        │
         │             │           reject        │
         │             │         ──────────────► REJECTED
         └─────────────┼────────────────────────┘
                       │
              PENDING_L1 ──► PENDING_L2 ──► PENDING_ADMIN ──► APPROVED
```

State transitions are enforced in `SubmissionService` and validated against the actor's role tier. Invalid transitions throw a domain exception — they are never silently ignored.

---

## 8. Security Design

- **Authentication:** JWT issued on login, validated on every request via a `OncePerRequestFilter`.
- **Authorization:** Method-level RBAC using `@PreAuthorize("hasRole('CHECKER_L1')")` on service methods.
- **Password storage:** BCrypt with strength factor 12.
- **Token strategy:** Short-lived access tokens (15 min) + refresh token pattern (Phase 2).
- **WebSocket security:** STOMP `CONNECT` frame validated for a valid JWT before subscription is accepted.
- **No sensitive data** (passwords, tokens) written to application logs at any level.

---

## 9. Technology Stack

| Layer          | Technology                        | Version (target) |
|----------------|-----------------------------------|------------------|
| Frontend       | Angular                           | 17+              |
| State mgmt     | NgRx                              | 17+              |
| Backend        | Spring Boot                       | 3.2+             |
| Security       | Spring Security + JJWT            | 3.2+ / 0.12+     |
| Persistence    | Spring Data JPA + Hibernate       | 3.2+             |
| Database       | PostgreSQL                        | 15+              |
| Migrations     | Flyway                            | 9+               |
| Real-time      | Spring WebSocket + STOMP/SockJS   | 3.2+             |
| Scheduler      | Spring `@Scheduled`               | built-in         |
| Containerization | Docker + Docker Compose         | latest           |
| Orchestration  | Kubernetes (minikube, local)      | 1.28+            |
| CI/CD          | GitHub Actions                    | —                |
| Build tools    | Maven (backend), Angular CLI (fe) | —                |

---

## 10. Git & Branching Strategy

**Branch model:** GitFlow

| Branch          | Purpose                                      | Protected |
|-----------------|----------------------------------------------|-----------|
| `main`          | Production-ready, tagged releases only        | Yes       |
| `develop`       | Integration branch; all features merge here  | Yes       |
| `feature/AGE-*` | One branch per feature ticket                | No        |
| `hotfix/*`      | Emergency production fixes only              | No        |

**Commit convention:** Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, `test:`)

**Merge policy:** All PRs require passing CI build + at least 1 approval before merging to `develop`.

---

## 11. Deployment Architecture (Local)

```
minikube cluster
├── Namespace: audit-engine
│   ├── Deployment: backend   (Spring Boot container, 2 replicas)
│   ├── Deployment: frontend  (Nginx + Angular build, 1 replica)
│   ├── Service: backend-svc  (ClusterIP)
│   ├── Service: frontend-svc (NodePort for local access)
│   └── ConfigMap: app-config (env vars, SLA defaults)
└── Namespace: data
    ├── StatefulSet: postgres
    └── PersistentVolumeClaim: postgres-pvc
```

---

## 12. Out of Scope (v1.0)

The following are intentionally excluded from v1.0 to maintain scope integrity:

- Email/SMS notification delivery (WebSocket only in v1)
- File virus scanning on upload
- Multi-tenancy
- OAuth2 / SSO integration
- Event sourcing or CQRS patterns
- Distributed tracing (Zipkin/Jaeger)

These are documented here so reviewers understand the decisions were deliberate, not overlooked.

---

*Next document: `LLD.md` — Low-Level Design covering sequence diagrams, package structure, and component contracts.*
