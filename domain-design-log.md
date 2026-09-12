# MyCompass — Domain & Product Decision Log

**Status:** Living document  
**Purpose:** Record the decisions from our product/domain Q&A and provide the roadmap for all remaining questions.  
**Project:** MyCompass — a personal-first task management and life-management platform built as a serious Spring Boot learning project.

---

## 1. Product Vision

> **MyCompass helps a person manage everything they need to DO.**

The core object is a **Task**.

The product is intentionally **task-centric**, rather than a generic calendar/event-management application.

### Long-term modules

1. **Tasks** — core execution/work-management domain
2. **Projects** — group related tasks around a larger goal
3. **Categories & Tags** — classification and flexible organization
4. **Reminders & Notifications** — make sure work is not forgotten
5. **Recurrence/Scheduling** — automatically generate recurring work
6. **Finance** — expenses, bills, budgets and financial tasks
7. **Collaboration** — shared tasks, assignment and permissions
8. **Search/Filtering/Dashboard** — make the system useful at scale
9. **Audit/Activity** — preserve meaningful history
10. **Automation/Event-driven architecture** — future evolution toward asynchronous processing

---

# 2. Architecture Direction

## Initial architecture

**Modular monolith** is the preferred starting point.

Potential Spring Boot modules/packages:

```text
mycompass
├── auth
├── user
├── task
├── project
├── category
├── tag
├── reminder
├── notification
├── recurrence
└── finance
```

Do **not** start with microservices.

The project is intentionally designed to evolve over months:

```text
3-day MVP
    ↓
Modular Spring Boot application
    ↓
Authentication + richer scheduling
    ↓
Redis + background workers
    ↓
Notifications + retries
    ↓
Kafka / event-driven architecture
    ↓
Outbox + idempotency + DLQ
    ↓
Observability
    ↓
Potential service extraction
    ↓
Distributed deployment / Kubernetes
```

The architecture should be **evolution-friendly**, but the MVP should remain simple.

---

# 3. Product Design Principles

These principles should guide future questions.

### 3.1 Explicit state changes

Editing task data should not unexpectedly change workflow state.

Example:

```text
COMPLETED
    │
    ├── edit description → COMPLETED
    ├── edit priority    → COMPLETED
    └── edit due date    → COMPLETED
```

Changing status must be an explicit operation.

### 3.2 Preserve history

Avoid destructive operations when a historical record is useful.

Prefer:

- soft delete
- independent recurring occurrences
- audit/activity history
- preserving subtask state

### 3.3 Keep derived state derived

Do not persist states that can reliably be calculated.

Example:

```text
OVERDUE
```

is derived from:

```text
dueAt < now
AND status != COMPLETED
AND status != CANCELLED
```

### 3.4 Don't over-engineer the MVP

A decision may be designed with future extensibility in mind without implementing the future system immediately.

Example:

```text
Personal MVP
    ↓
createdBy

Future collaboration
    ↓
participants / roles
```

---

# 4. Locked Product Decisions

## Q1 — Core object

**Decision:** Task

A task represents something the user needs to do.

---

## Q2 — Personal vs collaboration

**Decision:** Personal first, collaboration later.

Phase 1:

```text
Task → one user
```

Phase 2:

```text
Task → multiple participants
```

The personal model should not prevent future collaboration.

---

## Q3 — Reminders

**Decision:** A task can have **one or more Reminder entries**.

Reminder is a separate entity.

Example:

```text
Task: Pay electricity bill
Due: Sep 15, 10 AM

Reminders:
- Sep 13
- Sep 14
- Sep 15, 9 AM
```

Reminders can eventually be:

```text
SCHEDULED
TRIGGERED
DELIVERED
FAILED
CANCELLED
PAUSED
```

Not all of this needs to exist in the MVP.

When a task is completed:

> Cancel all future/pending reminders.

Already delivered notifications are not retroactively cancelled.

---

## Q4 — Recurring tasks

**Decision:** Each occurrence is a separate Task.

Example:

```text
September rent → Task #101 → COMPLETED
October rent   → Task #102 → TODO
November rent  → Task #103 → TODO
```

Do **not** generate years of occurrences upfront.

Use a recurrence rule/schedule that generates the next occurrence.

---

## Q5 — Missed recurring task

**Decision:**

If September's recurring occurrence is missed:

```text
September → OVERDUE
October   → new independent Task
```

Both can coexist.

Do not merge missed occurrences.

---

## Q6 — Rescheduling

**Decision:** Rescheduling changes the due date on the **same Task**.

Example:

```text
Sep 15 → Sep 18
```

No new task is created.

Future audit history should preserve the change:

```text
created
due date changed Sep 15 → Sep 18
completed
```

---

## Q7 — Overdue modeling

**Decision:** `OVERDUE` is **not** a persisted workflow status.

### Workflow status

```text
TODO
IN_PROGRESS
COMPLETED
CANCELLED
```

### Derived deadline state

```text
UPCOMING
DUE_SOON
OVERDUE
```

Example:

```text
isOverdue =
    status != COMPLETED
    && status != CANCELLED
    && dueAt != null
    && dueAt < now
```

A task can therefore be:

```text
IN_PROGRESS + OVERDUE
```

---

## Q8 — Categories and Projects

**Decision:** Both.

### Category

Answers:

> What kind of thing is this?

Examples:

```text
Finance
Learning
Health
Work
Personal
```

### Project

Answers:

> What larger goal does this belong to?

Example:

```text
Project: Java Interview Preparation

Tasks:
- Revise Collections
- Revise Multithreading
- Revise Spring Boot
- Practice System Design
- Mock Interview
```

Projects can later contain:

- progress
- deadline
- budget
- members
- project-level notifications

Projects are probably not required for the 3-day MVP.

---

## Q9 — Tags

**Decision:** System + custom tags.

Examples:

```text
System:
#urgent
#important
#personal
#work
#followup

Custom:
#interview
#sideproject
#vacation
```

Tags are multi-valued.

Category remains the primary classification mechanism.

---

## Q10 — Task scheduling

**Decision:** Optional `startAt` + optional `dueAt`.

Examples:

```text
Buy groceries
startAt = null
dueAt   = Sep 15, 7 PM
```

```text
Prepare presentation
startAt = Sep 15, 10 AM
dueAt   = Sep 17, 6 PM
```

`estimatedDuration` can be considered later.

---

## Q11 — No due date

**Decision:** Due date is optional.

Examples:

```text
Learn Kubernetes
Read Clean Code
Fix the cupboard
```

These remain open without a deadline.

```text
dueAt = null
→ never overdue
```

---

## Q12 — Dependencies

**Decision:** Start with one dependency and evolve toward multiple.

Initial:

```text
Task B depends on Task A
```

Future:

```text
Task C depends on Task A + Task B
```

Do not build an arbitrary dependency graph initially.

`BLOCKED` should probably be derived rather than persisted.

---

## Q13 — Dependency behavior

**Decision:** Block by default + manual override.

If dependency is incomplete:

```text
effective state = BLOCKED
```

But user can choose:

> Start anyway

Then the task can become:

```text
IN_PROGRESS
```

Future activity history can record the override.

---

## Q14 — Subtasks

**Decision:** One level only.

Example:

```text
Prepare for Java Interview
├── Revise Collections
├── Revise Multithreading
├── Revise Spring Boot
├── Practice System Design
└── Mock Interview
```

No recursive unlimited nesting initially.

---

## Q15 — Completing parent with incomplete subtasks

**Decision:** Allow completion with confirmation.

Example:

```text
2 subtasks are incomplete.
Are you sure you want to complete this task?
```

If confirmed:

```text
Parent → COMPLETED
Subtasks → unchanged
```

---

## Q16 — Completing the last subtask

**Decision:** Suggest parent completion.

When all subtasks are complete:

> All subtasks are complete. Mark parent task as completed?

Do **not** auto-complete the parent.

---

## Q17 — Task deletion

**Decision:** Soft delete.

Preferred model:

```text
deletedAt
```

rather than:

```text
isDeleted
```

Future collaboration can add:

```text
deletedBy
```

Deleted tasks disappear from normal views but remain potentially recoverable.

---

## Q18 — Task ownership

**Decision:** Initially use `createdBy`.

```text
Task
└── createdBy → User
```

Future collaboration can evolve toward:

```text
Task
├── createdBy
└── participants[]
    ├── OWNER
    ├── EDITOR
    └── VIEWER
```

Avoid a complex participant model in the MVP.

---

## Q19 — Task assignment

**Decision:** Creator can assign a task to another user.

Conceptually:

```text
Task
├── createdBy
└── assignedTo
```

For personal tasks, creator and assignee may be the same user.

---

## Q20 — Reassignment

**Decision:** Creator can reassign anytime.

Example:

```text
Somu → Kinu
```

No need to create a new task.

Future activity history:

```text
Task assigned: Devesh → Somu
Task reassigned: Somu → Kinu
```

---

## Q21 — Assignment change notifications

**Decision:** Notify both affected users.

On reassignment:

```text
Old assignee → "Task reassigned"
New assignee → "New task assigned"
```

Future event concepts:

```text
TaskAssigned
TaskReassigned
```

The MVP does not need Kafka.

---

## Q22 — Task permissions

**Decision:** Different permissions based on responsibility.

Conceptually:

```text
Creator
→ title
→ description
→ priority
→ due date
→ reminders
→ assignment
→ status

Assignee
→ status
→ progress
→ completion
```

A future role system may include:

```text
OWNER
EDITOR
ASSIGNEE
VIEWER
```

Do not build a complicated permission engine in the MVP.

---

## Q23 — Delete assigned task

**Decision:** Soft delete for everyone, recoverable by creator.

Delete means:

> This task should no longer appear in normal active views.

Cancel means:

> This was a real task, but we are no longer doing it.

This distinction should remain.

---

## Q24 — Reminder behavior when rescheduling

**Decision:** Support both absolute and relative reminders.

Conceptually:

```text
Reminder
├── type
│   ├── ABSOLUTE
│   └── RELATIVE_TO_DUE
├── triggerAt
└── offset
```

Examples:

```text
ABSOLUTE
→ Sep 13, 10 AM
```

```text
RELATIVE_TO_DUE
→ 2 days before due date
```

If due date changes:

```text
Sep 15 → Sep 20

2-days-before reminder:
Sep 13 → Sep 18
```

MVP may initially focus on relative-to-due reminders while still preserving the domain design for both modes.

---

## Q25 — Reminder without due date

**Decision:** Only absolute reminders.

```text
dueAt = null

ABSOLUTE reminder       ✅
RELATIVE_TO_DUE         ❌
```

Example:

```text
Learn Kubernetes
Reminder → Sep 20, 8 PM
```

This avoids ambiguous reminder semantics.

---

## Q26 — Completing a task

**Decision:** Complete task + cancel future reminders.

Completed tasks remain accessible and editable.

```text
TODO
 ↓
COMPLETED
 ↓
future reminders → CANCELLED
```

Subtasks remain unchanged.

For recurring tasks, completing one occurrence does not complete the recurrence rule.

---

## Q27 — Editing completed task

**Decision:** Editing does not reopen the task.

```text
COMPLETED
├── edit description → COMPLETED
├── edit priority    → COMPLETED
└── edit due date    → COMPLETED
```

Reopening must be explicit.

---

## Q28 — Reopening completed task

**Decision:** Allow:

```text
COMPLETED → TODO
COMPLETED → IN_PROGRESS
```

This preserves continuity rather than forcing creation of a new task.

Future audit:

```text
Sep 15 → COMPLETED
Sep 18 → REOPENED
Sep 18 → IN_PROGRESS
Sep 20 → COMPLETED
```

---

## Q29 — Reopening cancelled task

**Decision:** Allow:

```text
CANCELLED → TODO
```

Do not directly jump to `IN_PROGRESS`.

Cancellation means the task is intentionally not being done right now.

---

## Q30 — Reminders on cancellation

**Decision:** Pause future reminders and restore eligible ones when reopened.

```text
Task CANCELLED
    ↓
future reminders → PAUSED
    ↓
Task reopened
    ↓
future reminders → SCHEDULED
```

Past reminder times should not suddenly fire after reopening.

---

## Q31 — Subtasks on parent cancellation

**Decision:** Pause subtasks and restore their previous states on reopening.

Example before cancellation:

```text
Collections      TODO
Multithreading   IN_PROGRESS
Mock Interview   TODO
```

After reopening:

```text
Collections      TODO
Multithreading   IN_PROGRESS
Mock Interview   TODO
```

The system must preserve the previous subtask state.

`PAUSED` may be modeled as an operational concept without becoming a general workflow status unless future requirements justify it.

---

## Q32 — Completing parent with incomplete subtasks

**Decision:** Leave subtasks unchanged after confirmed parent completion.

```text
Parent → COMPLETED

Subtasks:
TODO          → TODO
IN_PROGRESS   → IN_PROGRESS
COMPLETED     → COMPLETED
```

Do not assume that completing the parent means every child was completed.

---

## Q33 — Adding a subtask to completed parent

**Decision:** Parent remains completed; new subtask starts as TODO.

```text
Parent → COMPLETED

Add subtask
    ↓
Parent → COMPLETED
Subtask → TODO
```

Adding/editing data never implicitly changes workflow state.

---

## Q34 — Subtask reminders

**Decision:** Subtasks can have independent reminders.

Example:

```text
Parent: Prepare for Java Interview

├── Revise Collections
│   └── Reminder: Sep 20
│
├── Revise Multithreading
│   └── Reminder: Sep 24
│
└── Mock Interview
    └── Reminder: Sep 28
```

Subtasks are treated as genuine units of work.

If the parent is cancelled, child reminders should also be paused consistently with child task state.

---

# 5. Current Conceptual Task Model

The current conceptual shape is approximately:

```text
Task
├── id
├── title
├── description
├── status
├── priority
├── startAt              optional
├── dueAt                optional
├── createdBy
├── assignedTo           optional / future collaboration
├── category
├── tags[]
├── project              optional
├── parentTask           optional
├── dependency           optional initially
├── recurrence           optional
├── reminders[]
└── deletedAt            optional
```

### Priority

Locked:

```text
LOW
MEDIUM
HIGH
URGENT
```

---

# 6. Important Domain Distinctions

These should not be collapsed into one field.

### Workflow status

```text
TODO
IN_PROGRESS
COMPLETED
CANCELLED
```

### Deadline state

```text
UPCOMING
DUE_SOON
OVERDUE
```

### Dependency state

Potentially:

```text
BLOCKED
UNBLOCKED
```

derived from dependencies.

### Deletion

```text
deletedAt
```

### Reminder execution state

Potentially:

```text
SCHEDULED
PAUSED
TRIGGERED
DELIVERED
FAILED
CANCELLED
```

---

# 7. Future Question Sets

The remaining Q&A should be handled **one question at a time**.

For every question:

1. Present clear options.
2. Give **⭐ My recommendation**.
3. Explain why.
4. Let the user choose.
5. Lock the decision into this document.
6. Continue to the next question.

The user has explicitly chosen to follow the recommended approach, so the recommendation should be strong and opinionated while still allowing the user to override it.

---

# Set A — Task Domain Completion

We are currently around Question 34. Continue the task-domain questions before jumping into architecture.

Potential topics:

### A1. Priority behavior
- Can priority change after creation?
- Does urgent affect reminders?
- Can priority be inherited by subtasks?

### A2. Start date semantics
- What happens when `startAt` arrives?
- Can a task be started before `startAt`?
- Is startAt informational or restrictive?

### A3. Due date semantics
- Exact timestamp vs date-only due dates
- Time zone handling
- Due-soon calculation
- End-of-day behavior

### A4. Dependency rules
- Multiple dependencies
- Circular dependency prevention
- What counts as dependency completion
- Manual override history

### A5. Subtask edge cases
- Can a completed parent accept new subtasks? (already decided: yes)
- Can subtasks be moved between parents?
- Can a subtask become a standalone task?
- What happens when parent is deleted?

### A6. Task duplication
- Should users duplicate tasks?
- What should be copied?
- Should reminders/recurrence be copied?

### A7. Task templates
- Should recurring workflows support reusable templates?
- Example: monthly expense process

### A8. Task notes/attachments
- Plain notes
- File attachments
- Links
- Comments

### A9. Task ordering
- Manual ordering
- Priority ordering
- Due-date ordering
- User-defined sort

### A10. Archive/history
- Whether completed tasks remain in normal lists
- Archive behavior
- Historical search

---

# Set B — Projects

Topics:

1. Project lifecycle
2. Project status
3. Project owner
4. Project members
5. Project tasks
6. Project deadlines
7. Project progress calculation
8. Project budget
9. Project-level reminders
10. Project archive
11. Project completion semantics
12. Nested projects — likely avoid initially

⭐ General architectural recommendation:

Keep Projects above Tasks conceptually:

```text
Project
   ├── Task
   ├── Task
   └── Task
```

Avoid making Project simply another Task type.

---

# Set C — Categories & Tags

Topics:

1. Built-in categories
2. User-created categories
3. Category hierarchy
4. One vs multiple categories
5. System tags
6. Custom tags
7. Tag ownership
8. Tag deletion
9. Tag renaming
10. Search/filter semantics

Likely direction:

```text
Task
├── one primary Category
└── many Tags
```

---

# Set D — Recurrence Engine

This deserves a dedicated design session.

Topics:

1. Daily recurrence
2. Weekly recurrence
3. Monthly recurrence
4. Yearly recurrence
5. Weekday rules
6. Month-end behavior
7. Start/end dates
8. Time zones
9. Missed occurrences
10. Generation timing
11. Duplicate occurrence prevention
12. Editing future recurrence
13. Editing one occurrence
14. Cancelling recurrence
15. Exceptions
16. Recurrence history

Potential model:

```text
RecurrenceRule
├── frequency
├── interval
├── startAt
├── endAt
├── timezone
└── rule configuration
```

Each generated occurrence remains a normal Task.

---

# Set E — Reminder & Notification System

Topics:

1. Notification channels
2. In-app notifications
3. Email
4. Push notifications
5. SMS — likely later
6. User preferences
7. Quiet hours
8. Retry strategy
9. Delivery status
10. Failed notification handling
11. Idempotency
12. Notification templates
13. Notification batching
14. Notification priority
15. Cancellation races
16. Time-zone behavior

Long-term architecture:

```text
Task
 ↓
Reminder
 ↓
Notification Event
 ↓
Queue / Kafka
 ↓
Notification Worker
 ↓
Provider
 ↓
Delivery Result
```

---

# Set F — Finance Module

The finance module should support the broader MyCompass goal without turning the entire application into an accounting system.

Topics:

1. Expense
2. Income
3. Accounts
4. Categories
5. Transactions
6. Recurring bills
7. Payment reminders
8. Budgets
9. Budget periods
10. Financial goals
11. Linking a Task to a financial event
12. Currency
13. Split expenses
14. Attachments/receipts
15. Reports
16. Import/export

Potential relationship:

```text
Task: Pay electricity bill
       │
       └── Financial Event
              ↓
           Expense
```

Keep Finance independently understandable.

---

# Set G — Authentication & Users

Topics:

1. Registration
2. Login
3. Password hashing
4. JWT/session approach
5. Refresh tokens
6. Email verification
7. Password reset
8. Account deletion
9. User time zone
10. User preferences
11. Roles
12. Security boundaries

Security should be treated as a first-class domain concern.

---

# Set H — Search, Filtering & Views

Topics:

1. Search by title
2. Search by description
3. Filter by status
4. Filter by priority
5. Filter by category
6. Filter by tags
7. Filter by project
8. Filter by due date
9. Filter by assignee
10. Overdue view
11. Today view
12. Upcoming view
13. My tasks
14. Assigned tasks
15. Completed history
16. Sorting
17. Pagination

Likely core views:

```text
Inbox
Today
Upcoming
Overdue
My Tasks
Assigned to Me
Projects
Completed
```

---

# Set I — Dashboard / MyCompass Home

Possible dashboard widgets:

```text
Today
Overdue
Upcoming
High Priority
My Tasks
Project Progress
Financial Snapshot
Recent Activity
```

Need to avoid turning the dashboard into an information dump.

---

# Set J — Audit & Activity History

Topics:

1. What changes are audited?
2. Who changed them?
3. Old value/new value
4. Assignment changes
5. Status changes
6. Due-date changes
7. Reminder changes
8. Dependency overrides
9. Collaboration activity
10. Retention policy

Potential model:

```text
Activity
├── taskId
├── actorId
├── type
├── oldValue
├── newValue
└── createdAt
```

This is especially important for future collaboration.

---

# Set K — API & Backend Design

Topics:

1. REST resource design
2. Endpoint naming
3. DTOs
4. Entity/DTO separation
5. Validation
6. Pagination
7. Sorting
8. Filtering
9. Error response format
10. Idempotency
11. API versioning
12. Optimistic locking
13. Concurrency
14. Transactions
15. Security boundaries

Likely philosophy:

> Domain model first, API second, database implementation third.

---

# Set L — Database Design

Likely PostgreSQL.

Topics:

1. Entity relationships
2. Foreign keys
3. Indexes
4. Unique constraints
5. Soft delete indexes
6. Audit tables
7. Many-to-many tags
8. Subtask relationship
9. Recurrence storage
10. Reminder storage
11. Notification storage
12. Optimistic locking
13. Migration strategy
14. Flyway/Liquibase
15. Query performance

---

# Set M — Scheduler / Background Processing

Topics:

1. How reminders are discovered
2. Polling interval
3. Job locking
4. Duplicate execution
5. Retry
6. Failure handling
7. Time zones
8. Recurrence generation
9. Distributed scheduling
10. Scheduler observability

Evolution:

```text
MVP:
Spring Scheduler

Later:
Persistent job table
     ↓
Distributed workers
     ↓
Queue/Kafka
```

---

# Set N — Event-Driven Architecture

Later, introduce events such as:

```text
TaskCreated
TaskUpdated
TaskCompleted
TaskCancelled
TaskAssigned
TaskReassigned
TaskRescheduled
ReminderTriggered
NotificationRequested
NotificationDelivered
NotificationFailed
```

Topics:

1. Domain events
2. Application events
3. Kafka
4. Outbox pattern
5. Idempotency
6. Ordering
7. Retry
8. Dead-letter queues
9. Consumer groups
10. Event schema evolution

---

# Set O — Redis / Caching

Topics:

1. What deserves caching?
2. User preferences
3. Dashboard
4. Frequently accessed task lists
5. Rate limiting
6. Distributed locks
7. Scheduler coordination
8. Cache invalidation

Do not add Redis merely because it is available.

Every infrastructure component should have a concrete reason to exist.

---

# Set P — Observability

Topics:

1. Structured logging
2. Correlation IDs
3. Metrics
4. Tracing
5. Health checks
6. Actuator
7. Error tracking
8. Scheduler metrics
9. Notification metrics
10. Kafka metrics

Potential stack later:

```text
Spring Boot Actuator
Prometheus
Grafana
OpenTelemetry
```

---

# Set Q — Testing Strategy

Topics:

1. Unit tests
2. Repository tests
3. Service tests
4. Controller tests
5. Integration tests
6. Testcontainers
7. Database migration tests
8. Scheduler tests
9. Notification tests
10. Contract tests
11. Concurrency tests
12. Property-based testing — later

Target:

```text
Business rules → heavily tested
Infrastructure → integration tested
Controllers → focused tests
```

---

# Set R — Deployment & DevOps

Topics:

1. Docker
2. Docker Compose
3. PostgreSQL container
4. CI/CD
5. Environment configuration
6. Secrets
7. Database migrations
8. Deployment strategy
9. Health checks
10. Rollbacks
11. Kubernetes
12. Horizontal scaling
13. Load testing

---

# Set S — 3-Day MVP Scope

After the domain Q&A is sufficiently complete, explicitly freeze the MVP.

### Strong candidate MVP

```text
Authentication
    ↓
Task CRUD
    ↓
Categories
    ↓
Priority
    ↓
startAt / dueAt
    ↓
Status lifecycle
    ↓
Soft delete
    ↓
Basic reminders
    ↓
Basic dashboard/list views
```

Potentially defer:

```text
Collaboration
Projects
Finance
Kafka
Redis
Push notifications
Advanced recurrence
AI
Kubernetes
```

The MVP should demonstrate strong Spring Boot fundamentals rather than maximum feature count.

---

# Set T — Growth Roadmap

## Stage 1 — 3 days

Goal:

> Working personal task manager.

Learn:

- Spring Boot
- REST
- JPA
- PostgreSQL
- Validation
- Exception handling
- Basic testing

## Stage 2 — 2–4 weeks

Add:

- Authentication
- Recurrence
- Reminder engine
- Email notification
- Docker
- Redis
- Testcontainers

## Stage 3 — 1–3 months

Add:

- Kafka
- Event-driven architecture
- Outbox
- Retry/DLQ
- Idempotency
- Observability
- Advanced notification system
- Collaboration

## Stage 4 — Advanced

Add:

- Service extraction
- Kubernetes
- Distributed scheduler
- Horizontal scaling
- Load testing
- Advanced analytics
- AI assistant

---

# 8. Decision Log Format

Every future decision should be added using:

```text
## Q<number> — <question>

**Decision:** <selected option>

### Options

A. ...
B. ...
C. ...
D. ...

### ⭐ Recommendation

<recommendation>

### Rationale

<why this is preferred>

### Domain impact

<what changes in the model>

### Future considerations

<what we intentionally defer>
```

---

# 9. Current Next Question

The next question should continue from **Question 35**.

Do not restart numbering.

The next session should continue the domain-design Q&A and keep asking **one question at a time**.

The user has asked to follow the recommended approach, so each question should continue to contain:

> **⭐ My recommendation**

and the reasoning behind it.

---

# 10. Final Design Philosophy

MyCompass should become more than a CRUD task application.

The long-term learning objective is to use the project to understand:

```text
Domain modeling
      ↓
Clean Spring Boot architecture
      ↓
Database design
      ↓
Transactions
      ↓
Concurrency
      ↓
Scheduling
      ↓
Notifications
      ↓
Distributed systems
      ↓
Event-driven architecture
      ↓
Observability
      ↓
Scalability
      ↓
Production engineering
```

But complexity should be introduced **only when the domain gives us a reason for it**.

That is the central engineering principle for this project.
