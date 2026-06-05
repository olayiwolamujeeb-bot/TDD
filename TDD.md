Document: Technical Design Document
Feature: Course Registration Portal
Name: Mujeeb 

1. Feature Overview And Objective
- **Feature**: Add Enrollment allow a student to enroll in a specific course section for a term.
- **Objective**: Ensure correct, consistent, and auditable enrollment handling with prerequisite checks, time-conflict detection, capacity enforcement, waitlisting, idempotency, and notifications.
- **Acceptance Criteria**: 
  - Student can enroll when prerequisites met, no schedule conflicts, and seats available.
  - If seats unavailable, student is placed on the waitlist with a deterministic position.
  - Concurrent enrollment attempts never oversubscribe capacity.
  - Actions are idempotent for retries and audited.
  - Waitlist promotions re-check constraints before enrollment.

2. Component Architecture
- **Frontend**: SPA component (`EnrollButton`, `ScheduleViewer`) calling Enrollment API; shows real-time seat/waitlist state.
- **API Layer**: `POST /enrollments` endpoint implementing auth, validation, and transactional enrollment logic.
- **Service / Business Logic**: Enrollment service with modules: `PrereqChecker`, `ConflictChecker`, `CapacityManager`, `WaitlistManager`, `AuditLogger`.
- **Persistence**: Primary RDBMS (Postgres) for ACID operations.
- **Cache/Locking**: Redis for ephemeral counters, optimistic locks, and distributed mutexes (as fallback).
- **Async Worker / Queue**: Message broker (RabbitMQ/Kafka) for `enrollment.created`, `seat.available`, `waitlist.promotion`, and notifications.
- **Notification Service**: Sends emails/SMS/web notifications asynchronously.
- **Monitoring/Audit**: Structured logs, audit table, metrics (Prometheus), tracing (OpenTelemetry).

3. Step-by-Step Data Flow (with TDD test cases)**
1. Client initiates enrollment.
   - Flow: Frontend sends POST /enrollments {student_id, section_id, idempotency_key}.
   - Tests:
     - Unit: API accepts well-formed requests and rejects unauthorized ones.
     - Integration: Endpoint returns 200 + enrollment id on success.
     - E2E: Authenticated student can call endpoint.
2. Auth & RBAC check.
   - Flow: Verify token, role == student or allowed actor.
   - Tests:
     - Unit: Unauthorized token → 401.
     - Integration: Different roles (student, admin, instructor) behave as expected (admin can force-enroll).
3. Validate student existence & status.
   - Flow: Confirm student record active.
   - Tests:
     - Unit: Non-existent student → 404.
     - Unit: Inactive student → 403 with descriptive error.
4. Prerequisite check.
   - Flow: `PrereqChecker` queries completed courses/grades.
   - Tests:
     - Unit: Missing prereqs → 400 with list of unmet prerequisites.
     - Integration: Completed prereqs pass.
5. Time-conflict detection.
   - Flow: Fetch student's enrollments for term, compare meeting slots with `ConflictChecker`.
   - Tests:
     - Unit: Overlap detection returns true for intersecting slots.
     - Integration: Attempt to enroll in conflicting section → 409 Conflict.
6. Begin transaction & concurrency control.
   - Flow: Begin DB transaction; lock Section row (SELECT FOR UPDATE) or use Redis atomic counter.
   - Tests:
     - Integration/Load: Simulate N concurrent enroll attempts where seats = M < N; assert final seats_taken == min(M,capacity) and no oversubscription.
     - Unit: Failure to acquire lock triggers retry/backoff.
7. Capacity check & enroll or waitlist.
   - Flow A (seats available): create Enrollment(status=enrolled), increment seats_taken, commit, emit `enrollment.created`.
   - Flow B (full): insert Waitlist entry (position = max+1), create Enrollment(status=waitlisted) OR only keep Waitlist depending on model, commit, emit `waitlist.joined`.
   - Tests:
     - Unit: Enroll success updates seats and audit record.
     - Integration: Full section → student placed on waitlist with correct position.
     - Edge: Capacity exactly 1, two concurrent requests → one success, one waitlist.
8. Post-commit actions.
   - Flow: Worker consumes `enrollment.created` for notifications and analytics (idempotent handler).
   - Tests:
     - Integration: Event enqueued and worker processes notification job (mocked transport).
     - Unit: Worker retry is idempotent.
9. Idempotency & retries.
   - Flow: Use idempotency_key persisted to prevent duplicate enrollments on retries.
   - Tests:
     - Unit: Repeat POST with same idempotency_key returns same enrollment id and does not double-enroll.
10. Dropping & waitlist promotion.
    - Flow: On drop, decrement seats and emit `seat.available`; worker processes waitlist to promote first eligible student, re-check prereqs/conflicts.
    - Tests:
      - Integration: When seat freed, next waitlist student is promoted and notified.
      - Unit: If promoted student now conflicts, skip and next is tried.
11. Audit & logging.
    - Flow: Log actor, action, target, timestamp, and relevant payload.
    - Tests:
      - Unit: Each enrollment path writes an AuditLog row.
      - Integration: Audit row contains idempotency_key when present.

**Database Table (core DDL-like summary & test checks)**
- **Users**
  - Columns: id (PK), name, email (unique), role, status
  - Tests: Unique email constraint; role enforcement.
- **Students**
  - Columns: id (PK), user_id (FK Users), student_number (unique), status
  - Tests: FK integrity.
- **Courses**
  - Columns: id, code, title, credits
  - Tests: Unique (code, term?) as needed.
- **Sections**
  - Columns: id (PK), course_id (FK), term_id (FK), section_code, instructor_id (FK), capacity (int), seats_taken (int), meeting_pattern (JSON), location
  - Indexes: (term_id), (course_id), partial index on seats_taken < capacity for quick availability queries
  - Tests:
    - Constraint: capacity >= 0.
    - Trigger/Constraint: seats_taken <= capacity enforced at DB level (CHECK or trigger).
    - Concurrency: SELECT FOR UPDATE semantics validated in integration tests.
- **ScheduleSlots** (optional normalized)
  - Columns: id, section_id (FK), day_of_week, start_time, end_time
  - Tests: Overlap detection unit tests.
- **Enrollments**
  - Columns: id (PK), student_id (FK), section_id (FK), status ENUM (enrolled, waitlisted, dropped), created_at, idempotency_key (nullable), metadata (JSON)
  - Constraints: UNIQUE(student_id, section_id)
  - Indexes: (student_id), (section_id)
  - Tests:
    - Unique constraint enforced.
    - Insertion with existing (student, section) returns 409 or updates status per business rule.
- **Waitlist**
  - Columns: id (PK), section_id (FK), student_id (FK), position (int), created_at
  - Tests:
    - Position monotonicity and gapless behavior on inserts/deletes.
- **Prerequisites**
  - Columns: id, course_id, prereq_course_id, type
  - Tests: Query correctness for complex logical prereqs (AND/OR).
- **AuditLog**
  - Columns: id, actor_id, action, target_type, target_id, payload (JSON), timestamp
  - Tests: Audit created on every enrollment-related mutation.

5. Risks And Limitations
- **Race Conditions / Oversubscription**: Risk mitigated by DB transaction locks or Redis atomic counters; must test with high-concurrency integration tests.
- **Stale Cache / Read Replicas**: Reads from replicas may show outdated seat counts; always perform final capacity checks against master in write path.
- **Complex Prereq Logic**: AND/OR prerequisites and cross-listed courses increase validation complexity—build unit tests for logic trees.
- **Waitlist Fairness & Thundering Herd**: Promotions can cause notification storms; implement rate-limiting and batched promotions.
- **Partial Failures**: If async notification fails after commit, student still enrolled; notification must be retried idempotently and not affect enrollment state.
- **Compliance (FERPA/PII)**: Ensure encrypted storage of PII, access controls, and audit trails.
- **Operational Costs**: Scaling for peak registration windows (batch jobs, DB replicas, autoscaling queues) increases cost; load-test realistic peaks.
- **Non-deterministic Time Conflicts**: Complex recurring schedules may need expansion algorithm; edge cases must be captured with deterministic unit tests.
- **Idempotency Edge-Cases**: If idempotency keys are not globally unique or expire too soon, duplicate enrollments or stale responses may occur; persist keys for a safe retention window.


