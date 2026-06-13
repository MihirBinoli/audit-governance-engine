# Low-Level Design (LLD)
## Enterprise Audit & Governance Workflow Engine

| Field        | Detail                          |
|--------------|---------------------------------|
| Version      | 1.0.0                           |
| Status       | Approved                        |
| Author       | [Your Name]                     |
| Last Updated | [Date]                          |
| Depends on   | HLD.md v1.0.0                   |

---

## 1. Purpose

This document translates the High-Level Design into implementable specifications. It defines
the exact package structure, class responsibilities, service method contracts, and runtime
behaviour for the two most critical flows in the system:

1. The multi-tier approval workflow (happy path + rejection)
2. The automated SLA escalation engine

Any engineer should be able to read this document and begin writing code without ambiguity.

---

## 2. Backend Package Structure (Clean Architecture)

The Spring Boot application enforces a strict four-layer architecture. Dependencies only
point inward: `controller → service → domain`. The `repository` layer is a plugin of
the `domain` layer via Spring Data interfaces.

```
com.enterprise.auditengine
│
├── domain/                          ← Enterprise business rules (no Spring deps)
│   ├── model/
│   │   ├── AuditSubmission.java     ← JPA entity + domain logic methods
│   │   ├── ApprovalAction.java      ← Immutable audit log entity
│   │   ├── SlaTracking.java         ← SLA state entity
│   │   ├── SlaConfig.java           ← SLA configuration entity
│   │   ├── User.java                ← User entity
│   │   └── Role.java                ← Role entity (with tierLevel)
│   ├── enums/
│   │   ├── SubmissionStatus.java    ← DRAFT, PENDING_L1, PENDING_L2,
│   │   │                               PENDING_ADMIN, APPROVED, REJECTED,
│   │   │                               REVISION_REQUESTED
│   │   ├── ActionType.java          ← APPROVE, REJECT, REQUEST_REVISION, SUBMIT
│   │   └── RoleName.java            ← MAKER, CHECKER_L1, CHECKER_L2, ADMIN
│   └── exception/
│       ├── InvalidStateTransitionException.java
│       ├── SlaConfigNotFoundException.java
│       └── UnauthorizedActionException.java
│
├── repository/                      ← Data access interfaces (Spring Data JPA)
│   ├── AuditSubmissionRepository.java
│   ├── ApprovalActionRepository.java
│   ├── SlaTrackingRepository.java
│   ├── SlaConfigRepository.java
│   └── UserRepository.java
│
├── service/                         ← Application business logic
│   ├── SubmissionService.java       ← Workflow state machine
│   ├── ApprovalService.java         ← Maker-Checker action processing
│   ├── SlaEscalationService.java    ← @Scheduled escalation engine
│   ├── NotificationService.java     ← WebSocket broadcast
│   └── UserService.java             ← User management
│
├── controller/                      ← HTTP entry points (no business logic)
│   ├── AuthController.java          ← POST /api/auth/login, /register
│   ├── SubmissionController.java    ← CRUD for audit submissions
│   ├── ApprovalController.java      ← POST /api/approvals/{id}/action
│   ├── SlaConfigController.java     ← Admin SLA configuration CRUD
│   └── WebSocketController.java     ← STOMP message mapping
│
├── security/                        ← Spring Security configuration
│   ├── JwtTokenProvider.java        ← Token generation + validation
│   ├── JwtAuthenticationFilter.java ← OncePerRequestFilter
│   ├── SecurityConfig.java          ← HttpSecurity bean
│   └── UserPrincipal.java           ← UserDetails implementation
│
├── dto/                             ← Request/response transfer objects
│   ├── request/
│   │   ├── LoginRequest.java
│   │   ├── CreateSubmissionRequest.java
│   │   └── ApprovalActionRequest.java
│   └── response/
│       ├── AuthResponse.java
│       ├── SubmissionResponse.java
│       ├── ApprovalActionResponse.java
│       └── EscalationNotificationPayload.java
│
├── mapper/                          ← Entity ↔ DTO mapping (MapStruct)
│   ├── SubmissionMapper.java
│   └── ApprovalMapper.java
│
├── config/                          ← Spring configuration beans
│   ├── WebSocketConfig.java         ← STOMP broker + endpoint registration
│   ├── SchedulerConfig.java         ← @EnableScheduling + task executor
│   └── OpenApiConfig.java           ← Springdoc/OpenAPI bean
│
└── AuditEngineApplication.java      ← @SpringBootApplication entry point
```

---

## 3. Frontend Module Structure (Angular)

```
src/
├── app/
│   ├── core/                        ← Singleton services, guards, interceptors
│   │   ├── auth/
│   │   │   ├── auth.service.ts      ← Login, logout, token storage
│   │   │   ├── auth.guard.ts        ← Route protection
│   │   │   └── jwt.interceptor.ts   ← Attaches Bearer token to all requests
│   │   ├── websocket/
│   │   │   └── websocket.service.ts ← STOMP client lifecycle management
│   │   └── core.module.ts
│   │
│   ├── shared/                      ← Reusable components, pipes, directives
│   │   ├── components/
│   │   │   ├── status-badge/        ← Submission status chip
│   │   │   ├── confirm-dialog/      ← Reusable approve/reject modal
│   │   │   └── page-header/
│   │   └── shared.module.ts
│   │
│   ├── features/
│   │   ├── auth/                    ← Login page (lazy loaded)
│   │   │   ├── login/
│   │   │   └── auth.routes.ts
│   │   │
│   │   ├── submissions/             ← Maker: create & track submissions
│   │   │   ├── submission-list/
│   │   │   ├── submission-detail/
│   │   │   ├── create-submission/
│   │   │   ├── store/               ← NgRx: actions, reducers, effects, selectors
│   │   │   │   ├── submission.actions.ts
│   │   │   │   ├── submission.reducer.ts
│   │   │   │   ├── submission.effects.ts
│   │   │   │   └── submission.selectors.ts
│   │   │   └── submissions.routes.ts
│   │   │
│   │   ├── approvals/               ← Checker: pending approvals queue
│   │   │   ├── approval-queue/
│   │   │   ├── approval-detail/
│   │   │   └── approvals.routes.ts
│   │   │
│   │   └── admin/                   ← Admin: SLA config + escalation dashboard
│   │       ├── dashboard/           ← Real-time WebSocket feed
│   │       ├── sla-config/
│   │       └── admin.routes.ts
│   │
│   ├── store/                       ← Root NgRx store
│   │   ├── app.state.ts
│   │   └── notification/            ← Escalation alert state
│   │       ├── notification.actions.ts
│   │       ├── notification.reducer.ts
│   │       └── notification.selectors.ts
│   │
│   └── app.config.ts                ← provideRouter, provideHttpClient, provideStore
│
└── environments/
    ├── environment.ts               ← { apiUrl, wsUrl, production: false }
    └── environment.prod.ts
```

---

## 4. Sequence Diagram — Submission Approval Flow

This diagram traces a submission from creation through full L1 → L2 → Admin approval.
The sad path (rejection + revision) is shown as an alternate sequence at the end.

```
  Maker          SubmissionCtrl    SubmissionSvc      DB              SlaTrackingSvc
    │                  │                 │             │                     │
    │ POST /submissions│                 │             │                     │
    │─────────────────►│                 │             │                     │
    │                  │ createSubmission│             │                     │
    │                  │────────────────►│             │                     │
    │                  │                 │ INSERT      │                     │
    │                  │                 │ status=DRAFT│                     │
    │                  │                 │────────────►│                     │
    │                  │                 │             │                     │
    │                  │                 │ startSlaTimer(submissionId, L1)   │
    │                  │                 │────────────────────────────────►  │
    │                  │                 │             │  INSERT sla_tracking│
    │                  │                 │             │◄────────────────────│
    │                  │                 │             │                     │
    │                  │◄────────────────│             │                     │
    │◄─────────────────│  201 Created    │             │                     │
    │                  │                 │             │                     │


  CheckerL1      ApprovalCtrl     ApprovalSvc       DB           NotificationSvc
    │                  │                 │             │                  │
    │ POST /approvals  │                 │             │                  │
    │ /{id}/action     │                 │             │                  │
    │ {action:APPROVE} │                 │             │                  │
    │─────────────────►│                 │             │                  │
    │                  │ processAction() │             │                  │
    │                  │────────────────►│             │                  │
    │                  │                 │ validate    │                  │
    │                  │                 │ actor role  │                  │
    │                  │                 │ vs tier     │                  │
    │                  │                 │             │                  │
    │                  │                 │ UPDATE status=PENDING_L2        │
    │                  │                 │────────────►│                  │
    │                  │                 │             │                  │
    │                  │                 │ INSERT approval_action (APPROVE)│
    │                  │                 │────────────►│                  │
    │                  │                 │             │                  │
    │                  │                 │ closeSlaTimer(id, L1)           │
    │                  │                 │ startSlaTimer(id, L2)           │
    │                  │                 │────────────►│                  │
    │                  │◄────────────────│             │                  │
    │◄─────────────────│  200 OK         │             │                  │


    [Repeat with CheckerL2 → PENDING_ADMIN, then Admin → APPROVED]


    ── ALTERNATE: Rejection + Revision ──────────────────────────────

  CheckerL1      ApprovalCtrl     ApprovalSvc       DB
    │                  │                 │             │
    │ {action:REJECT}  │                 │             │
    │─────────────────►│                 │             │
    │                  │────────────────►│             │
    │                  │                 │ UPDATE status=REJECTED
    │                  │                 │────────────►│
    │                  │                 │ INSERT approval_action (REJECT)
    │                  │                 │────────────►│
    │                  │◄────────────────│             │
    │◄─────────────────│  200 OK         │             │
    │                  │                 │             │

  Maker          SubmissionCtrl    SubmissionSvc      DB
    │ PUT /submissions │                 │             │
    │ /{id}/revise     │                 │             │
    │─────────────────►│                 │             │
    │                  │────────────────►│             │
    │                  │                 │ validate    │
    │                  │                 │ actor=owner │
    │                  │                 │             │
    │                  │                 │ UPDATE status=PENDING_L1 (restart)
    │                  │                 │────────────►│
    │                  │                 │ startSlaTimer(id, L1) [fresh]
    │                  │◄────────────────│             │
    │◄─────────────────│  200 OK         │             │
```

---

## 5. Sequence Diagram — SLA Escalation Engine

This diagram shows the scheduler tick, breach detection, database update, and real-time
WebSocket delivery to the admin dashboard.

```
  Spring Scheduler   SlaEscalationSvc   SlaTrackingRepo      NotificationSvc   WS Broker
         │                  │                  │                    │               │
  [every 5 min]             │                  │                    │               │
  @Scheduled tick           │                  │                    │               │
         │──────────────────►                  │                    │               │
         │                  │ findBreached()   │                    │               │
         │                  │─────────────────►│                    │               │
         │                  │                  │ SELECT * FROM      │               │
         │                  │                  │ sla_tracking WHERE │               │
         │                  │                  │ sla_deadline<NOW() │               │
         │                  │                  │ AND is_escalated=false             │
         │                  │◄─────────────────│                    │               │
         │                  │  [list of        │                    │               │
         │                  │   breached rows] │                    │               │
         │                  │                  │                    │               │
         │         [for each breached row]     │                    │               │
         │                  │                  │                    │               │
         │                  │ markEscalated(id)│                    │               │
         │                  │─────────────────►│                    │               │
         │                  │                  │ UPDATE             │               │
         │                  │                  │ is_escalated=true  │               │
         │                  │                  │ escalated_at=NOW() │               │
         │                  │                  │ escalation_count++ │               │
         │                  │◄─────────────────│                    │               │
         │                  │                  │                    │               │
         │                  │ broadcastEscalation(payload)          │               │
         │                  │───────────────────────────────────────►               │
         │                  │                  │                    │ convertAndSend │
         │                  │                  │                    │ /topic/escalat.│
         │                  │                  │                    │───────────────►│
         │                  │                  │                    │               │
         │                  │                  │           [Admin dashboard client  │
         │                  │                  │            receives STOMP frame]   │
         │◄──────────────────                  │                    │               │
         │  [next tick scheduled]              │                    │               │
```

**Idempotency guarantee:** The `WHERE is_escalated = false` predicate ensures that even if
the scheduler restarts mid-tick or runs overlapping executions (misconfiguration), a breach
is never escalated twice. The `escalation_count` column supports future multi-tier
escalation (e.g., escalate again if still idle after 2× SLA).

---

## 6. Service Method Contracts

### 6.1 SubmissionService

```java
/**
 * Creates a new submission in DRAFT status and starts the L1 SLA timer.
 * @throws IllegalArgumentException if the actor does not hold the MAKER role
 */
SubmissionResponse createSubmission(CreateSubmissionRequest request, UUID actorId);

/**
 * Advances a DRAFT submission to PENDING_L1. Only the original submitter may call this.
 * @throws InvalidStateTransitionException if submission is not in DRAFT status
 * @throws UnauthorizedActionException if actorId != submission.submittedBy
 */
SubmissionResponse submitForApproval(UUID submissionId, UUID actorId);

/**
 * Applies a revised document to a REJECTED submission, resetting it to PENDING_L1.
 * Restarts the L1 SLA timer from zero.
 * @throws InvalidStateTransitionException if submission is not in REJECTED status
 */
SubmissionResponse reviseAndResubmit(UUID submissionId, CreateSubmissionRequest request, UUID actorId);
```

### 6.2 ApprovalService

```java
/**
 * Processes an approval action (APPROVE | REJECT | REQUEST_REVISION) on a submission.
 *
 * Business rules enforced here:
 *  - Actor's role tierLevel must match submission's currentTier.
 *  - APPROVE on final tier (ADMIN) transitions status to APPROVED.
 *  - APPROVE on intermediate tier advances to next PENDING_* status.
 *  - REJECT at any tier transitions status to REJECTED.
 *  - REQUEST_REVISION transitions status to REVISION_REQUESTED.
 *  - Every call writes an immutable ApprovalAction record.
 *  - On APPROVE: closes current SLA timer, opens next tier timer.
 *  - On REJECT/REVISION: closes current SLA timer, no new timer started.
 *
 * @throws UnauthorizedActionException if actor's tier does not match currentTier
 * @throws InvalidStateTransitionException if submission is not in an actionable state
 */
ApprovalActionResponse processAction(UUID submissionId, ApprovalActionRequest request, UUID actorId);
```

### 6.3 SlaEscalationService

```java
/**
 * Scheduled method — runs every 5 minutes (configurable via application.properties).
 * Queries for all SLA tracking records where:
 *   sla_deadline < NOW() AND is_escalated = false
 * For each result: marks the record as escalated and publishes a notification.
 * The full operation is @Transactional to prevent partial updates on failure.
 */
@Scheduled(cron = "${sla.escalation.cron:0 */5 * * * *}")
@Transactional
void processEscalations();

/**
 * Creates an sla_tracking record for a given submission + tier.
 * Reads SLA duration from sla_configs(document_category, tier_level).
 * Sets sla_deadline = NOW() + sla_hours.
 * @throws SlaConfigNotFoundException if no config exists for the category+tier combination
 */
void startSlaTimer(UUID submissionId, int tierLevel);

/**
 * Marks the current open SLA tracking record for a submission+tier as resolved
 * by setting a resolved_at timestamp. Does not delete the record (audit trail).
 */
void closeSlaTimer(UUID submissionId, int tierLevel);
```

### 6.4 NotificationService

```java
/**
 * Broadcasts an escalation payload to all admin clients subscribed to
 * the STOMP topic /topic/escalations.
 *
 * Payload contains: submissionId, submissionTitle, documentCategory,
 * breachedTierLevel, slaDeadline, escalatedAt.
 */
void broadcastEscalation(EscalationNotificationPayload payload);
```

---

## 7. State Transition Table

This table is the single source of truth for valid state transitions. The `ApprovalService`
validates every incoming action against it before executing.

| Current Status        | Action             | Actor Role   | Next Status           |
|-----------------------|--------------------|--------------|-----------------------|
| `DRAFT`               | `SUBMIT`           | MAKER        | `PENDING_L1`          |
| `PENDING_L1`          | `APPROVE`          | CHECKER_L1   | `PENDING_L2`          |
| `PENDING_L1`          | `REJECT`           | CHECKER_L1   | `REJECTED`            |
| `PENDING_L1`          | `REQUEST_REVISION` | CHECKER_L1   | `REVISION_REQUESTED`  |
| `PENDING_L2`          | `APPROVE`          | CHECKER_L2   | `PENDING_ADMIN`       |
| `PENDING_L2`          | `REJECT`           | CHECKER_L2   | `REJECTED`            |
| `PENDING_L2`          | `REQUEST_REVISION` | CHECKER_L2   | `REVISION_REQUESTED`  |
| `PENDING_ADMIN`       | `APPROVE`          | ADMIN        | `APPROVED`            |
| `PENDING_ADMIN`       | `REJECT`           | ADMIN        | `REJECTED`            |
| `REJECTED`            | `REVISE`           | MAKER        | `PENDING_L1`          |
| `REVISION_REQUESTED`  | `REVISE`           | MAKER        | `PENDING_L1`          |

Any combination not listed in this table must throw `InvalidStateTransitionException`.

---

## 8. Security Layer Design

### JWT Filter Chain

```
HTTP Request
     │
     ▼
JwtAuthenticationFilter (OncePerRequestFilter)
     │
     ├── Extract Bearer token from Authorization header
     ├── Validate signature + expiry via JwtTokenProvider
     ├── Load UserDetails from UserRepository (by subject claim)
     ├── Set UsernamePasswordAuthenticationToken in SecurityContextHolder
     │
     ▼
Spring Security FilterChain
     │
     ▼
@PreAuthorize on Service methods
     │
     ├── hasRole('MAKER')        → SubmissionService.createSubmission()
     ├── hasRole('CHECKER_L1')   → ApprovalService (tier 1 actions)
     ├── hasRole('CHECKER_L2')   → ApprovalService (tier 2 actions)
     └── hasRole('ADMIN')        → SlaConfigController, ApprovalService (tier 3)
```

### WebSocket Authentication

STOMP `CONNECT` frames are intercepted by a `ChannelInterceptor`. The JWT token
is passed as a STOMP header and validated before the subscription is accepted.
Unauthenticated CONNECT frames are rejected with a STOMP `ERROR` frame.

---

## 9. Database Indexes (Performance)

These indexes are defined in Flyway migration `V3__add_indexes.sql`:

```sql
-- Most frequent query: fetch pending submissions for a role tier
CREATE INDEX idx_submissions_status ON audit_submissions(status);
CREATE INDEX idx_submissions_current_tier ON audit_submissions(current_tier);

-- SLA escalation engine query (runs every 5 minutes)
CREATE INDEX idx_sla_tracking_escalation
    ON sla_tracking(sla_deadline, is_escalated)
    WHERE is_escalated = false;

-- Audit log lookup by submission
CREATE INDEX idx_approval_actions_submission ON approval_actions(submission_id);

-- SLA config lookup (hot path in timer start)
CREATE UNIQUE INDEX idx_sla_config_category_tier
    ON sla_configs(document_category, tier_level)
    WHERE is_active = true;
```

---

## 10. Flyway Migration Strategy

All schema changes are managed exclusively through Flyway. No manual DDL is ever applied.

```
src/main/resources/db/migration/
├── V1__create_users_roles.sql          ← users, roles, user_roles tables
├── V2__create_submission_workflow.sql  ← audit_submissions, approval_actions
├── V3__create_sla_tables.sql           ← sla_configs, sla_tracking
├── V4__add_indexes.sql                 ← All performance indexes
└── V5__seed_roles_and_sla_defaults.sql ← Insert default roles + sample SLA configs
```

Migration naming convention: `V{version}__{snake_case_description}.sql`

Migration scripts are: append-only (never edited after merge), reviewed in PRs, and
applied automatically on application startup in all environments.

---

## 11. Configuration Properties

Key values exposed via `application.properties` / environment variables:

```properties
# JWT
jwt.secret=${JWT_SECRET}
jwt.expiration-ms=900000

# SLA Scheduler
sla.escalation.cron=0 */5 * * * *

# HikariCP
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.maximum-pool-size=20

# WebSocket
websocket.allowed-origins=${ALLOWED_ORIGINS:http://localhost:4200}
```

Secrets (`JWT_SECRET`, `DB_PASSWORD`) are never committed to source control.
They are injected via Docker environment variables in local development and via
Kubernetes Secrets in the cluster.

---

*Next document: `docs/api/openapi.yaml` — full REST API contract.*
