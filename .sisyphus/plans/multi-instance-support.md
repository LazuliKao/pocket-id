# Multi-Instance Deployment Support for Pocket-ID

## TL;DR

> **Quick Summary**: Enable Pocket-ID to run multiple replicas with CockroachDB cluster by implementing leader election for scheduled jobs, refactoring bootstrap initialization, and enforcing database-backed file storage.
> 
> **Deliverables**:
> - Leader election service using CockroachDB distributed lock
> - Refactored Bootstrap flow with initialization lock protection
> - Multi-instance mode detection and validation
> - Updated scheduled jobs with leader checks
> - App images service startup-time rebuild
> - Configuration and documentation for multi-instance deployment
> 
> **Estimated Effort**: Large
> **Parallel Execution**: YES - 4 waves
> **Critical Path**: Task 1 → Task 5 → Task 8 → Task 13 → Task 18 → F1-F4

---

### Wave 2: Bootstrap Refactoring

- [ ] 4. Extract Initialization Lock Logic

  **What to do**:
  - Modify `backend/internal/service/app_lock_service.go`
  - Add new method: `AcquireInitLock(ctx context.Context) error` - acquires lock only for initialization phase
  - Add new method: `ReleaseInitLock(ctx context.Context) error` - releases initialization lock
  - Use different lock key: `initialization_lock` (separate from `application_lock`)
  - No renewal needed (held only during Bootstrap, released after initServices() completes)
  - Timeout: 5 minutes (if initialization takes longer, something is wrong)

  **Must NOT do**:
  - Do not remove the existing `Acquire()` method (still needed for leader election integration)
  - Do not make initialization lock renewable (short-lived lock only)

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Modifying critical bootstrap lock service
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential (Task 5 depends on this)
  - **Blocks**: Task 5 (Bootstrap refactoring)
  - **Blocked By**: Tasks 1, 2 (needs leader election service and multi-instance utils)

  **References**:
  - `backend/internal/service/app_lock_service.go:73-169` - Acquire() method pattern
  - `backend/internal/service/app_lock_service.go:191-240` - Release() method pattern

  **Acceptance Criteria**:
  - [ ] Methods added: `AcquireInitLock(ctx)` and `ReleaseInitLock(ctx)`
  - [ ] `go build ./backend` → SUCCESS
  - [ ] `go test ./backend/internal/service -run TestAppLockService` → PASS

  **QA Scenarios**:
  ```
  Scenario: Acquire and release initialization lock
    Tool: Bash (go test)
    Preconditions: Empty database
    Steps:
      1. Call AcquireInitLock(ctx)
      2. Query database: SELECT value FROM kv WHERE key='initialization_lock'
      3. Assert record exists with expires_at > now
      4. Call ReleaseInitLock(ctx)
      5. Query database again
      6. Assert record deleted or expires_at in past
    Expected Result: Lock acquired and released cleanly
    Evidence: .sisyphus/evidence/task-4-init-lock.log

  Scenario: Initialization lock conflict (two instances)
    Tool: Bash (go test)
    Preconditions: One instance holds initialization lock
    Steps:
      1. Insert initialization_lock record with expires_at=now+300s
      2. Call AcquireInitLock(ctx) from test
      3. Assert error returned
    Expected Result: Second instance cannot acquire init lock
    Evidence: .sisyphus/evidence/task-4-init-lock-conflict.log
  ```

  **Commit**: NO (groups with Wave 2)

---

- [ ] 5. Refactor Bootstrap to Acquire Init Lock Early

  **What to do**:
  - Edit `backend/internal/bootstrap/bootstrap.go`
  - Move initialization lock acquisition to BEFORE `initServices()` call (currently at line 64, move to ~line 43)
  - New flow:
    ```
    1. NewDatabase() → migrations run (already safe with golang-migrate locks)
    2. InitStorage()
    3. [NEW] Create appLockService (without initServices)
    4. [NEW] appLockService.AcquireInitLock(ctx) ← EARLY LOCK
    5. initApplicationImages()
    6. initServices() ← JWT key gen, instance ID gen now protected
    7. [NEW] appLockService.ReleaseInitLock(ctx)
    8. [CONDITIONAL] If MultiInstanceMode: leaderElectionService.AcquireLeadership(ctx) (don't fail if not leader)
    9. registerScheduledJobs()
    10. initRouter()
    11. Run services (including leader renewal)
    ```
  - Handle errors gracefully: if AcquireInitLock fails, retry with exponential backoff (max 5 retries)
  - Add shutdown cleanup: ensure ReleaseInitLock called in defer block

  **Must NOT do**:
  - Do not break single-instance deployments (if MULTI_INSTANCE_MODE=false, skip leader election)
  - Do not remove the existing app lock logic entirely (still useful for runtime protection)

  **Recommended Agent Profile**:
  - **Category**: `deep`
    - Reason: Critical refactoring of bootstrap flow requiring careful orchestration
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential
  - **Blocks**: Tasks 6, 11, 13 (downstream bootstrap logic)
  - **Blocked By**: Task 4 (needs AcquireInitLock method)

  **References**:
  - `backend/internal/bootstrap/bootstrap.go:20-112` - Full Bootstrap function
  - `backend/internal/bootstrap/services_bootstrap.go` - initServices() implementation
  - `backend/internal/service/app_lock_service.go:73-84` - Lock acquisition pattern with retry

  **Acceptance Criteria**:
  - [ ] `go build ./backend` → SUCCESS
  - [ ] `go test ./backend/internal/bootstrap` → PASS
  - [ ] Single-instance mode still works: `MULTI_INSTANCE_MODE=false go run ./backend/cmd/pocket-id` → starts successfully

  **QA Scenarios**:
  ```
  Scenario: Bootstrap with initialization lock (single instance)
    Tool: Bash
    Preconditions: Clean database, MULTI_INSTANCE_MODE=false
    Steps:
      1. Start Pocket-ID: go run ./backend/cmd/pocket-id
      2. Check logs for "Acquired initialization lock"
      3. Check logs for "Released initialization lock"
      4. Check HTTP health endpoint: curl http://localhost:8080/health
      5. Assert HTTP 200 OK
    Expected Result: App starts successfully with init lock lifecycle logged
    Evidence: .sisyphus/evidence/task-5-bootstrap-single.log

  Scenario: Bootstrap with multi-instance mode (leader election)
    Tool: Bash
    Preconditions: Clean database, MULTI_INSTANCE_MODE=true
    Steps:
      1. Start Pocket-ID: MULTI_INSTANCE_MODE=true go run ./backend/cmd/pocket-id
      2. Check logs for "Acquired initialization lock"
      3. Check logs for "Released initialization lock"
      4. Check logs for "Attempting to acquire leadership" or "Running as follower"
      5. curl http://localhost:8080/health → 200 OK
    Expected Result: App starts with init lock + leader election logic
    Evidence: .sisyphus/evidence/task-5-bootstrap-multi.log

  Scenario: Bootstrap initialization lock conflict (two instances starting simultaneously)
    Tool: Bash (docker-compose with 2 replicas)
    Preconditions: docker-compose.yml with 2 Pocket-ID services
    Steps:
      1. docker-compose up -d (starts 2 instances simultaneously)
      2. Check logs: docker-compose logs pocket-id-1 pocket-id-2
      3. One instance logs "Acquired initialization lock" immediately
      4. Other instance logs "Retrying lock acquisition" (exponential backoff)
      5. Both instances eventually reach "Running" state
    Expected Result: Sequential initialization, no race condition errors
    Evidence: .sisyphus/evidence/task-5-bootstrap-conflict.log
  ```

  **Commit**: NO (groups with Wave 2)

---

- [ ] 6. Add Filesystem Backend Validation

  **What to do**:
  - Edit `backend/internal/bootstrap/bootstrap.go`
  - After `InitStorage()` call (line ~43), add validation:
    ```go
    if common.EnvConfig.MultiInstanceMode && common.EnvConfig.FileBackend == storage.TypeFileSystem {
        return fmt.Errorf("multi-instance mode requires 'database' or 's3' file storage backend; 'filesystem' is not supported for concurrent access")
    }
    ```
  - Add integration test: start with `MULTI_INSTANCE_MODE=true FILE_BACKEND=filesystem` → expect startup failure with clear error

  **Must NOT do**:
  - Do not silently switch storage backend (fail loudly with clear error)
  - Do not validate this in multiple places (single point of validation in Bootstrap)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Simple validation check with error return
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential (depends on Task 5)
  - **Blocks**: Task 13 (integration test)
  - **Blocked By**: Task 5 (Bootstrap refactoring), Task 3 (EnvConfig.MultiInstanceMode)

  **References**:
  - `backend/internal/bootstrap/bootstrap.go:43-46` - InitStorage() call location
  - `backend/internal/storage/storage.go:8-12` - Storage backend type constants

  **Acceptance Criteria**:
  - [ ] `go build ./backend` → SUCCESS
  - [ ] Startup with invalid config fails: `MULTI_INSTANCE_MODE=true FILE_BACKEND=filesystem go run ./backend/cmd/pocket-id` → exits with error message

  **QA Scenarios**:
  ```
  Scenario: Reject filesystem backend in multi-instance mode
    Tool: Bash
    Preconditions: None
    Steps:
      1. Set env: MULTI_INSTANCE_MODE=true, FILE_BACKEND=filesystem
      2. Run: go run ./backend/cmd/pocket-id
      3. Assert exit code != 0
      4. Check stderr contains "multi-instance mode requires 'database' or 's3' file storage backend"
    Expected Result: Startup fails with clear error message
    Evidence: .sisyphus/evidence/task-6-filesystem-reject.log

  Scenario: Accept database backend in multi-instance mode
    Tool: Bash
    Preconditions: CockroachDB running
    Steps:
      1. Set env: MULTI_INSTANCE_MODE=true, FILE_BACKEND=database
      2. Run: go run ./backend/cmd/pocket-id
      3. Assert app starts successfully (HTTP 200 on /health)
    Expected Result: App starts without errors
    Evidence: .sisyphus/evidence/task-6-database-accept.log
  ```

  **Commit**: YES
  - Message: `feat(bootstrap): refactor initialization with early lock and multi-instance validation`
  - Files: `backend/internal/bootstrap/bootstrap.go`, `backend/internal/service/app_lock_service.go`
  - Pre-commit: `go test ./backend/internal/...`

---

### Wave 3: Scheduled Jobs Leader Checks

- [ ] 7. Update LDAP Sync Job with Leader Check

  **What to do**:
  - Edit `backend/internal/job/ldap_job.go`
  - Inject `leaderElectionService *service.LeaderElectionService` into job function
  - At start of `SyncLdap()` function (line ~23), add:
    ```go
    if common.EnvConfig.MultiInstanceMode && !leaderElectionService.IsLeader() {
        slog.Debug("Skipping LDAP sync: not leader")
        return nil
    }
    ```
  - Update `registerScheduledJobs()` in `scheduler_bootstrap.go` to pass leaderElectionService

  **Must NOT do**:
  - Do not skip job registration (register on all instances, but only leader executes)
  - Do not remove existing jitter/retry logic

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Simple if-check addition
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3 (with Tasks 8-10)
  - **Blocks**: Task 14 (integration test for leader failover)
  - **Blocked By**: Task 1 (leader election service)

  **References**:
  - `backend/internal/job/ldap_job.go:22-43` - SyncLdap function
  - `backend/internal/bootstrap/scheduler_bootstrap.go:20-110` - Job registration

  **Acceptance Criteria**:
  - [ ] `go build ./backend` → SUCCESS
  - [ ] Leader instance logs "Running LDAP sync"
  - [ ] Follower instances log "Skipping LDAP sync: not leader"

  **QA Scenarios**:
  ```
  Scenario: Leader executes LDAP sync job
    Tool: Bash (manual trigger + log check)
    Preconditions: Multi-instance mode, current instance is leader, LDAP configured
    Steps:
      1. Force job execution: trigger LDAP sync via API or wait for schedule
      2. Check logs for "Running LDAP sync" or similar
      3. Verify LDAP service method was called (check audit logs or database changes)
    Expected Result: Job executes on leader
    Evidence: .sisyphus/evidence/task-7-ldap-leader.log

  Scenario: Follower skips LDAP sync job
    Tool: Bash (manual trigger + log check)
    Preconditions: Multi-instance mode, current instance is NOT leader
    Steps:
      1. Trigger job (or wait for schedule)
      2. Check logs for "Skipping LDAP sync: not leader"
      3. Verify no LDAP API calls made (check network logs or mocks)
    Expected Result: Job skipped on follower
    Evidence: .sisyphus/evidence/task-7-ldap-follower.log
  ```

  **Commit**: NO (groups with Wave 3)

---

- [ ] 8. Update SCIM Sync Job with Leader Check

  **What to do**:
  - Edit `backend/internal/job/scim_job.go`
  - Add leader check at start of `SyncScim()` function (same pattern as Task 7)

  **Recommended Agent Profile**: `quick`
  **Parallelization**: YES (Wave 3)
  **Blocks**: Task 14
  **Blocked By**: Task 1
  **References**: Same as Task 7, replace ldap_job.go with scim_job.go
  **Acceptance Criteria**: Same as Task 7 (leader runs, follower skips)
  **QA Scenarios**: Same as Task 7 (replace LDAP with SCIM)
  **Commit**: NO

---

- [ ] 9. Update GeoLite Update Job with Leader Check

  **What to do**:
  - Edit `backend/internal/job/geoloite_update_job.go`
  - Add leader check at start of `UpdateGeoLiteDB()` function

  **Recommended Agent Profile**: `quick`
  **Parallelization**: YES (Wave 3)
  **Blocks**: Task 14
  **Blocked By**: Task 1
  **References**: Same as Task 7, replace ldap_job.go with geoloite_update_job.go
  **Acceptance Criteria**: Same as Task 7
  **QA Scenarios**: Same as Task 7 (replace LDAP with GeoLite update)
  **Commit**: NO

---

- [ ] 10. Update API Key Expiry Job with Leader Check

  **What to do**:
  - Edit `backend/internal/job/api_key_expiry_job.go`
  - Add leader check at start of job function (line ~28)

  **Recommended Agent Profile**: `quick`
  **Parallelization**: YES (Wave 3)
  **Blocks**: Task 14
  **Blocked By**: Task 1
  **References**: Same as Task 7, replace ldap_job.go with api_key_expiry_job.go
  **Acceptance Criteria**: Same as Task 7
  **QA Scenarios**: Same as Task 7 (replace LDAP with API key notification)
  **Commit**: YES
  - Message: `feat(jobs): add leader election checks to unsafe scheduled jobs`
  - Files: `backend/internal/job/*.go`, `backend/internal/bootstrap/scheduler_bootstrap.go`
  - Pre-commit: `go test ./backend/internal/job`

### Wave 5: Integration & Testing

- [ ] 13. Integration Test: Multi-Instance Bootstrap

  **What to do**:
  - Create `backend/tests/integration/multi_instance_bootstrap_test.go`
  - Test scenario: Start 3 Pocket-ID instances simultaneously
  - Verify:
    1. All 3 instances acquire initialization lock sequentially (check logs)
    2. All 3 instances complete Bootstrap successfully
    3. Exactly one instance becomes leader (check database leader_election record)
    4. All 3 instances respond to HTTP health check
  - Use docker-compose or Go test with multiple goroutines simulating instances
  - Clean up: stop all instances, clean database

  **Must NOT do**:
  - Do not use real external services (LDAP, SCIM) - use mocks or skip those tests

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Integration test requiring orchestration of multiple instances
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES (with Tasks 14-16)
  - **Parallel Group**: Wave 5
  - **Blocks**: Task 18 (docker-compose example)
  - **Blocked By**: Tasks 5, 6, 7-10, 11 (full Bootstrap flow and job checks)

  **References**:
  - `backend/internal/bootstrap/bootstrap_test.go` - Existing bootstrap test patterns (if any)
  - `docker-compose.yml` - Docker compose configuration

  **Acceptance Criteria**:
  - [ ] `go test ./backend/tests/integration -run TestMultiInstanceBootstrap` → PASS

  **QA Scenarios**:
  ```
  Scenario: 3 instances bootstrap successfully
    Tool: Bash (docker-compose or go test)
    Preconditions: Clean database, MULTI_INSTANCE_MODE=true
    Steps:
      1. docker-compose up -d --scale pocket-id=3
      2. Wait 30 seconds for all instances to start
      3. Check logs: docker-compose logs pocket-id
      4. Assert all 3 instances log "Bootstrap completed"
      5. Query database: SELECT COUNT(*) FROM kv WHERE key='leader_election'
      6. Assert exactly 1 leader record exists
      7. Test HTTP: curl http://localhost:8080/health (load balancer distributes)
      8. Assert 200 OK from all instances
    Expected Result: All instances healthy, one leader elected
    Evidence: .sisyphus/evidence/task-13-multi-bootstrap.log
  ```

  **Commit**: NO (groups with Wave 5)

---

- [ ] 14. Integration Test: Leader Election Failover

  **What to do**:
  - Create `backend/tests/integration/leader_failover_test.go`
  - Test scenario:
    1. Start 3 instances, verify one becomes leader
    2. Identify leader instance (query database or check logs)
    3. Kill leader instance (docker-compose stop pocket-id-1)
    4. Wait for lease expiration (30 seconds + buffer)
    5. Verify new leader elected (query database, check different leader_id)
    6. Verify unsafe jobs resume on new leader (trigger LDAP sync, check logs)
  - Use docker-compose for realistic testing

  **Recommended Agent Profile**: `unspecified-high`
  **Parallelization**: YES (Wave 5)
  **Blocks**: Task 18
  **Blocked By**: Tasks 7-10 (job leader checks)
  **References**: `backend/internal/service/leader_election_service.go`, `backend/internal/job/*.go`
  **Acceptance Criteria**: `go test ./backend/tests/integration -run TestLeaderFailover` → PASS

  **QA Scenarios**:
  ```
  Scenario: Leader crash triggers re-election
    Tool: Bash (docker-compose)
    Preconditions: 3 instances running, one is leader
    Steps:
      1. Query database: SELECT value FROM kv WHERE key='leader_election' → note leader_id (e.g., "abc123")
      2. Identify leader container: docker-compose logs | grep "abc123"
      3. Stop leader: docker-compose stop pocket-id-1
      4. Wait 35 seconds (TTL=30s + 5s buffer)
      5. Query database again: leader_id should be different
      6. Check logs of remaining instances: one should log "Acquired leadership"
      7. Trigger unsafe job (or wait for schedule): verify new leader executes it
    Expected Result: New leader elected within 35 seconds, jobs resume
    Evidence: .sisyphus/evidence/task-14-leader-failover.log
  ```

  **Commit**: NO (groups with Wave 5)

---

- [ ] 15. Update AppLockService Tests

  **What to do**:
  - Edit `backend/internal/service/app_lock_service_test.go`
  - Add tests for new methods:
    - `TestAcquireInitLock` - verify initialization lock acquisition
    - `TestReleaseInitLock` - verify initialization lock release
    - `TestInitLockConflict` - two instances trying to acquire init lock
    - `TestInitLockTimeout` - init lock expires after 5 minutes
  - Update existing tests if necessary (ensure backward compatibility)

  **Recommended Agent Profile**: `quick`
  **Parallelization**: YES (Wave 5)
  **Blocks**: Task 18
  **Blocked By**: Task 4 (AcquireInitLock/ReleaseInitLock methods)
  **References**: `backend/internal/service/app_lock_service_test.go` - Existing test patterns
  **Acceptance Criteria**: `go test ./backend/internal/service -run TestAppLockService` → PASS
  **QA Scenarios**: Covered by automated tests
  **Commit**: NO (groups with Wave 5)

---

- [ ] 16. Add LeaderElectionService Unit Tests

  **What to do**:
  - Create `backend/internal/service/leader_election_service_test.go`
  - Test coverage:
    - `TestAcquireLeadership` - first instance acquires leadership
    - `TestIsLeader` - verify IsLeader() returns correct status
    - `TestRenewLeadership` - renewal loop updates lease
    - `TestLeadershipConflict` - second instance cannot acquire while leader exists
    - `TestLeadershipExpiration` - expired lease can be acquired by another instance
    - `TestReleaseLeadership` - clean release of leadership
  - Use in-memory SQLite database for testing (fast, no external dependencies)

  **Recommended Agent Profile**: `unspecified-high`
  **Parallelization**: YES (Wave 5)
  **Blocks**: Task 18
  **Blocked By**: Task 1 (LeaderElectionService implementation)
  **References**: `backend/internal/service/app_lock_service_test.go` - Similar test patterns
  **Acceptance Criteria**: `go test ./backend/internal/service -run TestLeaderElection` → PASS (at least 6 tests, all pass)

  **QA Scenarios**:
  ```
  Scenario: Verify test coverage
    Tool: Bash
    Preconditions: None
    Steps:
      1. Run: go test -cover ./backend/internal/service -run TestLeaderElection
      2. Assert coverage >= 80% for leader_election_service.go
      3. Assert all tests pass
    Expected Result: High test coverage, all tests pass
    Evidence: .sisyphus/evidence/task-16-test-coverage.log
  ```

  **Commit**: YES
  - Message: `test(multi-instance): add integration and unit tests for multi-instance support`
  - Files: `backend/tests/integration/*.go`, `backend/internal/service/*_test.go`
  - Pre-commit: `go test ./backend/...`

---

### Wave 6: Documentation & Deployment

- [ ] 17. Write Multi-Instance Deployment Guide

  **What to do**:
  - Create `docs/multi-instance-deployment.md`
  - Sections:
    1. **Overview** - Why multi-instance? Benefits (high availability, load distribution)
    2. **Requirements** - CockroachDB/PostgreSQL cluster, database or S3 file storage
    3. **Configuration** - Set `MULTI_INSTANCE_MODE=true`, `FILE_BACKEND=database`
    4. **Leader Election** - Explain automatic leader election, job execution on leader only
    5. **Rate Limiting Behavior** - Link to `multi-instance-rate-limiting.md`
    6. **Deployment Examples** - Docker Compose, Kubernetes (3 replicas)
    7. **Monitoring** - How to check which instance is leader, job execution logs
    8. **Troubleshooting** - Common issues (filesystem backend error, leader not elected)
  - Add diagrams (optional): leader election flow, multi-instance architecture
  - Update main `README.md` to link to this guide

  **Recommended Agent Profile**:
  - **Category**: `writing`
    - Reason: Documentation task
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO (sequential after Task 12)
  - **Parallel Group**: Wave 6 (with Task 18)
  - **Blocks**: None (final deliverable)
  - **Blocked By**: Tasks 12, 13, 14 (needs integration test results and rate limit docs)

  **References**:
  - `docs/configuration.md` - Existing configuration documentation
  - `README.md` - Main readme structure
  - `.sisyphus/drafts/multi-instance-lock-analysis.md` - Draft with research findings

  **Acceptance Criteria**:
  - [ ] File created: `docs/multi-instance-deployment.md`
  - [ ] `README.md` updated with link to multi-instance guide
  - [ ] Documentation reviewed for clarity and completeness

  **QA Scenarios**:
  ```
  Scenario: Follow deployment guide manually
    Tool: Manual (human QA in Task F3)
    Preconditions: Clean CockroachDB cluster
    Steps:
      1. Follow guide step-by-step to deploy 3 Pocket-ID instances
      2. Verify all steps work as documented
      3. Check monitoring section instructions (query database for leader)
    Expected Result: Successful deployment following guide
    Evidence: .sisyphus/evidence/task-17-deployment-guide-qa.md (QA notes)
  ```

  **Commit**: NO (groups with Wave 6)

---

- [ ] 18. Update Docker Compose Example with 3 Replicas

  **What to do**:
  - Edit `docker-compose.yml` (or create `docker-compose-multi-instance.yml`)
  - Add 3 Pocket-ID service replicas:
    ```yaml
    services:
      pocket-id-1:
        image: pocket-id:latest
        environment:
          MULTI_INSTANCE_MODE: "true"
          FILE_BACKEND: "database"
          DB_CONNECTION_STRING: "postgresql://cockroachdb:26257/pocketid"
      pocket-id-2:
        image: pocket-id:latest
        environment:
          MULTI_INSTANCE_MODE: "true"
          FILE_BACKEND: "database"
          DB_CONNECTION_STRING: "postgresql://cockroachdb:26257/pocketid"
      pocket-id-3:
        image: pocket-id:latest
        environment:
          MULTI_INSTANCE_MODE: "true"
          FILE_BACKEND: "database"
          DB_CONNECTION_STRING: "postgresql://cockroachdb:26257/pocketid"
      cockroachdb:
        image: cockroachdb/cockroach:latest
        command: start-single-node --insecure
    ```
  - Add load balancer (nginx or traefik) to distribute traffic
  - Test: `docker-compose up -d` → verify all 3 instances healthy

  **Recommended Agent Profile**: `quick`
  **Parallelization**: NO (depends on Task 17)
  **Blocks**: None (final deliverable)
  **Blocked By**: Tasks 13, 14, 17 (integration tests and documentation)
  **References**: `docker-compose.yml` - Existing docker-compose file
  **Acceptance Criteria**: `docker-compose -f docker-compose-multi-instance.yml up -d` → all services healthy

  **QA Scenarios**:
  ```
  Scenario: Start 3-replica deployment with docker-compose
    Tool: Bash
    Preconditions: Docker and docker-compose installed
    Steps:
      1. docker-compose -f docker-compose-multi-instance.yml up -d
      2. Wait 30 seconds
      3. docker-compose ps → verify all services "Up"
      4. curl http://localhost:8080/health → 200 OK (load balanced)
      5. docker-compose logs pocket-id-1 pocket-id-2 pocket-id-3
      6. Verify one instance logs "Acquired leadership"
    Expected Result: All instances healthy, one leader
    Evidence: .sisyphus/evidence/task-18-docker-compose-multi.log
  ```

  **Commit**: YES
  - Message: `docs(multi-instance): add deployment guide and docker-compose example`
  - Files: `docs/multi-instance-deployment.md`, `docker-compose-multi-instance.yml`, `README.md`
  - Pre-commit: `docker-compose -f docker-compose-multi-instance.yml config` (validate syntax)

---

## Final Verification Wave

- [ ] F1. **Plan Compliance Audit** — `oracle`

  Read the plan end-to-end. For each "Must Have": verify implementation exists (read file, check database schema, run startup). For each "Must NOT Have": search codebase for forbidden patterns (Redis imports, breaking changes to single-instance mode). Check evidence files exist in `.sisyphus/evidence/`. Compare deliverables against plan.

  Output: `Must Have [N/N] | Must NOT Have [N/N] | Tasks [18/18] | VERDICT: APPROVE/REJECT`

---

- [ ] F2. **Code Quality Review** — `unspecified-high`

  Run `go build ./backend` + `go test ./backend/...` + `golangci-lint run`. Review all changed files for: unsafe concurrent access (missing mutexes), error handling gaps, commented-out code, `TODO` markers. Check for proper logging (leader election events, job skips). Verify no hardcoded values (timeouts should be configurable).

  Output: `Build [PASS/FAIL] | Tests [N pass/N fail] | Lint [N issues] | Code Review [N issues] | VERDICT`

---

- [ ] F3. **Real Manual QA** — `unspecified-high`

  Execute EVERY QA scenario from EVERY task — follow exact steps, capture evidence. Test cross-task integration: leader failover triggers job execution on new leader. Test edge cases: all 3 instances crash simultaneously (no leader), then restart (one becomes leader). Save evidence to `.sisyphus/evidence/final-qa/`.

  Output: `Scenarios [N/N pass] | Integration [N/N] | Edge Cases [N tested] | VERDICT`

---

- [ ] F4. **Scope Fidelity Check** — `deep`

  For each task: read "What to do", read actual diff (`git log -p`). Verify 1:1 — everything in spec was built (no missing), nothing beyond spec was built (no creep). Check "Must NOT do" compliance. Detect cross-task contamination: Task N touching unrelated files. Flag unaccounted changes.

  Output: `Tasks [18/18 compliant] | Contamination [CLEAN/N issues] | Unaccounted [CLEAN/N files] | VERDICT`

---

---

## Commit Strategy

- **Commit 1** (After Task 3): Foundation - leader election service, multi-instance config, utils
  - `feat(config): add MULTI_INSTANCE_MODE environment variable`
  - Files: `backend/internal/common/env_config.go`, `backend/internal/utils/multi_instance.go`, `backend/internal/service/leader_election_service.go` + tests
  - Pre-commit: `go test ./backend/internal/...`

- **Commit 2** (After Task 6): Bootstrap refactoring - early init lock, validation
  - `feat(bootstrap): refactor initialization with early lock and multi-instance validation`
  - Files: `backend/internal/bootstrap/bootstrap.go`, `backend/internal/service/app_lock_service.go`
  - Pre-commit: `go test ./backend/internal/bootstrap ./backend/internal/service`

- **Commit 3** (After Task 10): Scheduled jobs - leader checks
  - `feat(jobs): add leader election checks to unsafe scheduled jobs`
  - Files: `backend/internal/job/*.go`, `backend/internal/bootstrap/scheduler_bootstrap.go`
  - Pre-commit: `go test ./backend/internal/job`

- **Commit 4** (After Task 12): App images + rate limit docs
  - `feat(app-images): rebuild extensions map on startup for multi-instance support`
  - Files: `backend/internal/service/app_images_service.go`, `docs/multi-instance-rate-limiting.md`, `docs/configuration.md`
  - Pre-commit: `go test ./backend/internal/service`

- **Commit 5** (After Task 16): Integration tests
  - `test(multi-instance): add integration and unit tests for multi-instance support`
  - Files: `backend/tests/integration/*.go`, `backend/internal/service/*_test.go`
  - Pre-commit: `go test ./backend/...`

- **Commit 6** (After Task 18): Documentation and deployment examples
  - `docs(multi-instance): add deployment guide and docker-compose example`
  - Files: `docs/multi-instance-deployment.md`, `docker-compose-multi-instance.yml`, `README.md`
  - Pre-commit: `docker-compose -f docker-compose-multi-instance.yml config`

---

## Success Criteria

### Verification Commands

**Build and test**:
```bash
go build ./backend                          # Expected: SUCCESS
go test ./backend/...                       # Expected: All tests PASS
golangci-lint run ./backend/...             # Expected: 0 errors
```

**Single-instance mode (backward compatibility)**:
```bash
MULTI_INSTANCE_MODE=false FILE_BACKEND=filesystem go run ./backend/cmd/pocket-id
# Expected: App starts successfully, HTTP 200 on /health
```

**Multi-instance mode with database backend**:
```bash
MULTI_INSTANCE_MODE=true FILE_BACKEND=database DB_CONNECTION_STRING="..." go run ./backend/cmd/pocket-id
# Expected: App starts, logs "Acquired leadership" or "Running as follower"
```

**Multi-instance mode rejects filesystem backend**:
```bash
MULTI_INSTANCE_MODE=true FILE_BACKEND=filesystem go run ./backend/cmd/pocket-id
# Expected: Exit code 1, error message "multi-instance mode requires 'database' or 's3' file storage backend"
```

**Docker Compose multi-instance deployment**:
```bash
docker-compose -f docker-compose-multi-instance.yml up -d
docker-compose ps                           # Expected: All services "Up"
curl http://localhost:8080/health           # Expected: HTTP 200
docker-compose logs | grep -i "acquired leadership"  # Expected: Exactly 1 instance
```

**Leader failover test**:
```bash
# Start 3 instances, identify leader
docker-compose -f docker-compose-multi-instance.yml up -d
LEADER=$(docker-compose logs | grep "Acquired leadership" | head -1 | cut -d'|' -f1)
docker-compose stop $LEADER                 # Kill leader
sleep 35                                    # Wait for re-election (TTL=30s)
docker-compose logs | grep "Acquired leadership" | tail -1  # Expected: New leader elected
```

### Final Checklist

- [ ] **Must Have - All Present**:
  - [ ] Leader election service implemented (`leader_election_service.go`)
  - [ ] Automatic leader failover works (tested in Task 14)
  - [ ] Unsafe scheduled jobs protected (LDAP, SCIM, GeoLite, API key notifications)
  - [ ] Initialization lock protects JWT key and instance ID generation
  - [ ] Filesystem backend rejected in multi-instance mode with clear error
  - [ ] Multiple instances can start simultaneously without error
  - [ ] Configuration option `MULTI_INSTANCE_MODE` added

- [ ] **Must NOT Have - All Absent**:
  - [ ] No Redis dependency (verify: `grep -r "redis" backend/` → no imports)
  - [ ] No changes to safe scheduled jobs' execution (DB cleanup jobs still run on all instances)
  - [ ] No new database migration files (verify: `git diff backend/resources/migrations/` → no changes)
  - [ ] No breaking changes to single-instance deployments (verify: backward compatibility test passes)
  - [ ] No over-engineering (verify: leader election <300 lines of code)
  - [ ] No rate limiting migration to database (verify: `middleware/rate_limit.go` unchanged except docs)

- [ ] **All Tests Pass**:
  - [ ] Unit tests: `go test ./backend/internal/service -run TestLeaderElection` → PASS
  - [ ] Unit tests: `go test ./backend/internal/service -run TestAppLockService` → PASS
  - [ ] Integration test: `go test ./backend/tests/integration -run TestMultiInstanceBootstrap` → PASS
  - [ ] Integration test: `go test ./backend/tests/integration -run TestLeaderFailover` → PASS

- [ ] **Documentation Complete**:
  - [ ] Multi-instance deployment guide exists (`docs/multi-instance-deployment.md`)
  - [ ] Rate limiting behavior documented (`docs/multi-instance-rate-limiting.md`)
  - [ ] README.md links to multi-instance guide
  - [ ] Docker Compose example works (`docker-compose-multi-instance.yml`)

- [ ] **Manual QA Passed**:
  - [ ] 3 instances start successfully
  - [ ] One leader elected automatically
  - [ ] Leader crash triggers re-election within 30 seconds
  - [ ] Unsafe jobs execute only on leader (checked via logs)
  - [ ] Safe jobs execute on all instances (DB cleanup)
  - [ ] HTTP requests load-balanced across all instances

- [ ] **Code Quality**:
  - [ ] No commented-out code
  - [ ] No `TODO` markers in committed code
  - [ ] Proper error handling (no ignored errors)
  - [ ] Logging at appropriate levels (leader election events at INFO level)
  - [ ] No race conditions (run `go test -race ./backend/...` → PASS)

- [ ] **Evidence Files**:
  - [ ] All task evidence files exist in `.sisyphus/evidence/task-N-*.{log,md}`
  - [ ] Final QA evidence in `.sisyphus/evidence/final-qa/`
  - [ ] Integration test outputs captured

---

**END OF PLAN**
 App Images & Rate Limiting

- [ ] 11. Modify AppImagesService to Rebuild Extensions on Startup

  **What to do**:
  - Edit `backend/internal/service/app_images_service.go`
  - Add new method: `RebuildExtensionsMap(ctx context.Context) error`
  - Implementation:
    1. Call `fileStorage.List(ctx, "application-images/")` to get all stored images
    2. Parse filenames to extract extensions (e.g., "app-123.png" → "123" maps to ".png")
    3. Rebuild `extensions` map: `map[string]string{"123": ".png", ...}`
    4. Acquire write lock: `s.mu.Lock(); defer s.mu.Unlock()`
    5. Replace `s.extensions` with rebuilt map
  - Call `RebuildExtensionsMap()` in `NewAppImagesService()` before returning
  - Add logging: "Rebuilt application images extensions map: %d entries"

  **Must NOT do**:
  - Do not remove RWMutex (still needed for concurrent access within instance)
  - Do not change SaveImage/GetImage logic (only initialization changes)

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Modifying service initialization with file storage interaction
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO (sequential after Task 5)
  - **Parallel Group**: Wave 4 (with Task 12)
  - **Blocks**: Task 13 (integration test)
  - **Blocked By**: Task 5 (Bootstrap refactoring)

  **References**:
  - `backend/internal/service/app_images_service.go:17-51` - AppImagesService struct and constructor
  - `backend/internal/service/app_images_service.go:63-100` - SaveImage method (understand how extensions are stored)
  - `backend/internal/storage/storage.go:6-15` - FileStorage.List() method signature

  **Acceptance Criteria**:
  - [ ] `go build ./backend` → SUCCESS
  - [ ] Startup logs include "Rebuilt application images extensions map: N entries"
  - [ ] `go test ./backend/internal/service -run TestAppImagesService` → PASS

  **QA Scenarios**:
  ```
  Scenario: Rebuild extensions map from existing images
    Tool: Bash (integration test)
    Preconditions: Database file storage with 3 existing images
    Steps:
      1. Pre-populate database: INSERT INTO files (path, data) VALUES ('application-images/app-1.png', '...'), ('application-images/app-2.jpg', '...'), ('application-images/app-3.gif', '...')
      2. Start AppImagesService: service := NewAppImagesService(...)
      3. Assert extensions map contains: {"1": ".png", "2": ".jpg", "3": ".gif"}
      4. Check logs: "Rebuilt application images extensions map: 3 entries"
    Expected Result: Extensions map populated from storage
    Evidence: .sisyphus/evidence/task-11-rebuild-extensions.log

  Scenario: Empty storage (no images)
    Tool: Bash (integration test)
    Preconditions: Empty file storage
    Steps:
      1. Start AppImagesService with empty storage
      2. Assert extensions map is empty: len(service.extensions) == 0
      3. Check logs: "Rebuilt application images extensions map: 0 entries"
    Expected Result: No errors, empty extensions map
    Evidence: .sisyphus/evidence/task-11-rebuild-empty.log
  ```

  **Commit**: NO (groups with Wave 4)

---

- [ ] 12. Document Rate Limiting Per-Instance Behavior

  **What to do**:
  - Create `docs/multi-instance-rate-limiting.md`
  - Document that rate limiting is per-instance, not global
  - Explain: N instances = N× effective rate limit
  - Example: If rate limit is 100 req/min per IP, and you have 3 instances, effective limit is ~300 req/min (load balancer dependent)
  - Alternatives: Use sticky sessions on load balancer, or accept relaxed limits
  - Update main `docs/configuration.md` to link to this document

  **Recommended Agent Profile**:
  - **Category**: `writing`
    - Reason: Documentation task
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES (Wave 4)
  - **Parallel Group**: Wave 4 (with Task 11)
  - **Blocks**: Task 17 (full deployment guide)
  - **Blocked By**: None

  **References**:
  - `backend/internal/middleware/rate_limit.go:1-120` - Current rate limiting implementation

  **Acceptance Criteria**:
  - [ ] File created: `docs/multi-instance-rate-limiting.md`
  - [ ] `docs/configuration.md` updated with link to rate limiting doc

  **QA Scenarios**:
  ```
  Scenario: Verify documentation completeness
    Tool: Bash (markdown linter + manual review)
    Preconditions: None
    Steps:
      1. Run: npx markdownlint docs/multi-instance-rate-limiting.md
      2. Manual review: read doc and verify it explains per-instance behavior clearly
      3. Check link in docs/configuration.md exists and works
    Expected Result: Documentation is clear, no linting errors
    Evidence: .sisyphus/evidence/task-12-rate-limit-docs.md (copy of the doc)
  ```

  **Commit**: YES
  - Message: `feat(app-images): rebuild extensions map on startup for multi-instance support`
  - Files: `backend/internal/service/app_images_service.go`, `docs/multi-instance-rate-limiting.md`, `docs/configuration.md`
  - Pre-commit: `go test ./backend/internal/service`

---


### Original Request
User encounters error when starting Pocket-ID: "it appears that there's already one instance of Pocket ID running; running multiple replicas of Pocket ID is currently not supported". They are running a CockroachDB cluster and need to deploy multiple Pocket-ID instances for high availability.

### Interview Summary
**Key Discussions**:
- Application lock (`AppLockService`) prevents multi-instance deployment - located at `bootstrap.go:64-66`
- Lock protects 4 categories: scheduled jobs, file storage, in-memory state, initialization logic
- CockroachDB cluster is already highly available (distributed SQL database)
- User does not want to introduce Redis dependency

**User's Decisions**:
1. **Scheduler strategy**: Leader election - automatic failover when leader crashes
2. **Rate limiting**: Relaxed per-instance (acceptable trade-off)
3. **App images extensions**: Rebuild from file storage scan on startup
4. **File storage**: Prohibit filesystem backend in multi-instance mode
5. **JWT/Instance ID race conditions**: Move lock acquisition timing for full fix

**Research Findings** (from 4 parallel explore agents):
- **Scheduled Jobs**: 14 jobs total
  - 4 unsafe (LDAP sync, SCIM sync, GeoLite update, API key notifications) - need leader election
  - 10 safe (database cleanup jobs) - idempotent, no protection needed
- **File Storage**:
  - Database backend: fully safe for multi-instance
  - S3 backend: mostly safe (DeleteAll has race window but acceptable)
  - Filesystem backend: UNSAFE - concurrent modifications cause corruption
- **In-Memory State**:
  - 2 unsafe components: rate limiter (solution: relax limits), app images extensions (solution: rebuild on startup)
  - 9 safe components: already database-backed or stateless
- **Initialization**:
  - golang-migrate has built-in advisory locks (safe)
  - JWT key generation and instance ID generation run BEFORE lock acquisition (race conditions)
  - Lock acquired too late in Bootstrap flow (line 64, after initServices())

---

## Work Objectives

### Core Objective
Enable Pocket-ID to run multiple replicas simultaneously with CockroachDB cluster for high availability, replacing the single-instance application lock with leader election for scheduled job execution.

### Concrete Deliverables
- `backend/internal/service/leader_election_service.go` - leader election implementation
- Refactored `backend/internal/bootstrap/bootstrap.go` - initialization lock before service init
- Updated `backend/internal/job/*.go` files - leader checks before job execution
- Modified `backend/internal/service/app_images_service.go` - startup rebuild of extensions map
- Multi-instance mode detection in bootstrap - fail if filesystem + multi-instance
- Environment variable `MULTI_INSTANCE_MODE=true/false`
- Updated documentation: `docs/multi-instance-deployment.md`

### Definition of Done
- [ ] Multiple Pocket-ID instances can start simultaneously without error
- [ ] Only leader instance executes scheduled jobs (verified via logs)
- [ ] Leader election failover works within 30 seconds when leader crashes
- [ ] JWT keys and instance IDs are consistent across instances
- [ ] Filesystem backend is rejected in multi-instance mode with clear error message
- [ ] All automated tests pass
- [ ] Manual QA: 3 instances running, kill leader, verify new leader elected and jobs continue

### Must Have
- Leader election based on CockroachDB (no external dependencies like Redis/etcd)
- Automatic leader failover (no manual intervention)
- Protection for unsafe scheduled jobs (LDAP, SCIM, GeoLite, API key notifications)
- Initialization lock for JWT key and instance ID generation
- Clear error messages for unsupported configurations (filesystem + multi-instance)

### Must NOT Have (Guardrails)
- **No Redis dependency** - all distributed coordination via CockroachDB
- **No removal of safe scheduled jobs' redundancy** - DB cleanup jobs can run on all instances (idempotent)
- **No changes to CockroachDB schema beyond leader election table** - minimize database impact
- **No breaking changes to existing single-instance deployments** - multi-instance is opt-in via config
- **No over-engineering** - keep leader election simple (database-based lease with TTL)
- **No rate limiting migration to database** - accept per-instance rate limiting as trade-off
- **No new database migration files** - use AutoMigrate for leader election table

---

## Verification Strategy

> **ZERO HUMAN INTERVENTION** — ALL verification is agent-executed. No exceptions.

### Test Decision
- **Infrastructure exists**: YES (Go testing framework, existing test files)
- **Automated tests**: Tests-after (implement features first, then add tests)
- **Framework**: Go standard `testing` package + `testify/assert`
- **Test Coverage**: Unit tests for leader election service, integration tests for multi-instance bootstrap

### QA Policy
Every task includes agent-executed QA scenarios using:
- **Backend/Service**: Bash (go test, curl, database queries)
- **Integration**: Bash (docker-compose with 3 replicas, check leader logs)
- **Configuration**: Bash (test startup with various FILE_BACKEND values)

Evidence saved to `.sisyphus/evidence/task-{N}-{scenario-slug}.{ext}`.

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Foundation - START IMMEDIATELY):
├── Task 1: Leader election service (core implementation) [unspecified-high]
├── Task 2: Multi-instance mode detection utility [quick]
└── Task 3: Add MULTI_INSTANCE_MODE environment variable to EnvConfig [quick]

Wave 2 (Bootstrap Refactoring - AFTER Wave 1):
├── Task 4: Extract initialization lock logic from AppLockService [unspecified-high]
├── Task 5: Refactor Bootstrap to acquire init lock before initServices() [deep]
└── Task 6: Add filesystem backend validation in multi-instance mode [quick]

Wave 3 (Scheduled Jobs - AFTER Wave 1):
├── Task 7: Update LDAP sync job with leader check [quick]
├── Task 8: Update SCIM sync job with leader check [quick]
├── Task 9: Update GeoLite update job with leader check [quick]
└── Task 10: Update API key expiry job with leader check [quick]

Wave 4 (App Images & Rate Limiting - AFTER Wave 2):
├── Task 11: Modify AppImagesService to rebuild extensions on startup [unspecified-high]
└── Task 12: Document rate limiting per-instance behavior [writing]

Wave 5 (Integration & Testing - AFTER Waves 3 & 4):
├── Task 13: Integration test: multi-instance bootstrap [unspecified-high]
├── Task 14: Integration test: leader election failover [unspecified-high]
├── Task 15: Update existing app_lock_service tests [quick]
└── Task 16: Add leader_election_service unit tests [unspecified-high]

Wave 6 (Documentation & Deployment - AFTER Wave 5):
├── Task 17: Write multi-instance deployment guide [writing]
└── Task 18: Update docker-compose example with 3 replicas [quick]

Wave FINAL (Verification - AFTER ALL TASKS):
├── Task F1: Plan compliance audit (oracle)
├── Task F2: Code quality review (unspecified-high)
├── Task F3: Real manual QA (unspecified-high)
└── Task F4: Scope fidelity check (deep)
-> Present results -> Get explicit user okay
```

### Dependency Matrix

- **1-3**: — — 4-10, 1
- **4**: 1, 2 — 5, 2
- **5**: 4 — 6, 13, 3
- **6**: 5, 3 — 13, 3
- **7-10**: 1 — 13, 14, 4
- **11**: 5 — 13, 5
- **12**: — — 17, 6
- **13**: 5, 6, 7-10, 11 — 18, 7
- **14**: 7-10 — 18, 7
- **15**: 4, 5 — 18, 7
- **16**: 1 — 18, 7
- **17**: 12, 13, 14 — 18, 8
- **18**: 17 — F1-F4, 9

### Agent Dispatch Summary

- **Wave 1**: 3 tasks — T1 → `unspecified-high`, T2-T3 → `quick`
- **Wave 2**: 3 tasks — T4 → `unspecified-high`, T5 → `deep`, T6 → `quick`
- **Wave 3**: 4 tasks — T7-T10 → `quick`
- **Wave 4**: 2 tasks — T11 → `unspecified-high`, T12 → `writing`
- **Wave 5**: 4 tasks — T13-T14, T16 → `unspecified-high`, T15 → `quick`
- **Wave 6**: 2 tasks — T17 → `writing`, T18 → `quick`
- **FINAL**: 4 tasks — F1 → `oracle`, F2-F3 → `unspecified-high`, F4 → `deep`

---

## TODOs

### Wave 1: Foundation

- [ ] 1. Implement Leader Election Service

  **What to do**:
  - Create `backend/internal/service/leader_election_service.go`
  - Implement leader election using CockroachDB-backed distributed lock (similar to AppLockService pattern)
  - Lease-based approach: leader holds lease with TTL (30s), renews every 20s
  - Methods: `AcquireLeadership(ctx)`, `IsLeader()`, `ReleaseLeadership(ctx)`, `RunRenewal(ctx)`
  - Store leader state in `kv` table with key `leader_election`
  - Leader value: `{"leader_id": "uuid", "host_id": "hostname", "acquired_at": unix_timestamp, "expires_at": unix_timestamp}`
  - Automatic re-election: when leader's lease expires, any instance can acquire leadership
  - Thread-safe: use atomic flag to track local leadership status

  **Must NOT do**:
  - Do not use advisory locks (not needed, lease-based is sufficient)
  - Do not create new database table (use existing `kv` table)
  - Do not add Redis/etcd dependencies
  - Do not make it overly complex (keep it simple like AppLockService)

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Core distributed system component requiring careful concurrency handling
  - **Skills**: []
    - No specific skills needed, pure backend logic

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 2, 3)
  - **Blocks**: Tasks 4, 7-10 (all components that need to check leadership)
  - **Blocked By**: None (can start immediately)

  **References**:
  - `backend/internal/service/app_lock_service.go` - Existing lock pattern (lease with TTL, renewal loop, database KV storage)
  - `backend/internal/model/kv.go` - KV model for database storage
  - `backend/internal/bootstrap/bootstrap.go:64-84` - How AppLockService is initialized and used

  **Why each reference matters**:
  - `app_lock_service.go`: Copy the lease acquisition pattern, renewal loop, and error handling strategy
  - `kv.go`: Understand the database schema for storing leader state
  - `bootstrap.go`: See how to integrate the new service into the bootstrap flow

  **Acceptance Criteria**:
  - [ ] File created: `backend/internal/service/leader_election_service.go`
  - [ ] `go build ./backend` → SUCCESS (no compilation errors)
  - [ ] `go test ./backend/internal/service -run TestLeaderElection` → PASS (after unit tests added in Task 16)

  **QA Scenarios**:
  ```
  Scenario: Leader acquisition on first instance
    Tool: Bash (go test)
    Preconditions: Empty database (no existing leader)
    Steps:
      1. Run test that calls AcquireLeadership(ctx)
      2. Check IsLeader() returns true
      3. Query database: SELECT value FROM kv WHERE key='leader_election'
      4. Assert leader_id, host_id, expires_at fields are populated
    Expected Result: Leader acquired successfully, IsLeader()=true, database record exists
    Failure Indicators: IsLeader()=false, database record missing, error returned
    Evidence: .sisyphus/evidence/task-1-leader-acquisition.log

  Scenario: Leader lease renewal (background renewal loop)
    Tool: Bash (go test with time.Sleep)
    Preconditions: Leader acquired in previous scenario
    Steps:
      1. Start RunRenewal(ctx) in goroutine
      2. Sleep 25 seconds (renewal interval is 20s)
      3. Query database: SELECT value FROM kv WHERE key='leader_election'
      4. Assert expires_at timestamp is newer than initial value
    Expected Result: Lease renewed, expires_at updated
    Failure Indicators: expires_at not updated, renewal goroutine crashed
    Evidence: .sisyphus/evidence/task-1-leader-renewal.log

  Scenario: Non-leader instance cannot acquire leadership (lease conflict)
    Tool: Bash (go test)
    Preconditions: Leader already exists (acquired by another instance)
    Steps:
      1. Simulate existing leader by inserting record into kv table with expires_at > now
      2. Call AcquireLeadership(ctx) from test
      3. Assert error returned (similar to ErrLockUnavailable)
      4. Assert IsLeader() returns false
    Expected Result: Leadership acquisition fails gracefully, IsLeader()=false
    Failure Indicators: Acquired leadership despite existing leader, no error returned
    Evidence: .sisyphus/evidence/task-1-leader-conflict.log
  ```

  **Evidence to Capture**:
  - [ ] task-1-leader-acquisition.log - go test output + database query result
  - [ ] task-1-leader-renewal.log - renewal loop execution logs
  - [ ] task-1-leader-conflict.log - conflict scenario test output

  **Commit**: NO (groups with Wave 1)

---

- [ ] 2. Multi-Instance Mode Detection Utility

  **What to do**:
  - Create `backend/internal/utils/multi_instance.go`
  - Add function `IsMultiInstanceMode() bool` - checks `MULTI_INSTANCE_MODE` environment variable
  - Add function `ValidateMultiInstanceConfig(envConfig *common.EnvConfigSchema) error` - validates configuration compatibility
  - Validation logic:
    - If `MULTI_INSTANCE_MODE=true` AND `FILE_BACKEND=filesystem` → return error "Filesystem storage is not supported in multi-instance mode. Use 'database' or 's3' backend."
    - Otherwise → return nil
  - Add unit tests in `multi_instance_test.go`

  **Must NOT do**:
  - Do not add complex configuration logic (keep it simple boolean check)
  - Do not modify EnvConfig yet (Task 3 does that)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Simple utility functions with straightforward validation logic
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 3)
  - **Blocks**: Tasks 4, 6 (Bootstrap validation depends on this)
  - **Blocked By**: None

  **References**:
  - `backend/internal/common/env_config.go:83-229` - EnvConfig structure and validation patterns
  - `backend/internal/common/env_config.go:132` - ValidateEnvConfig function pattern

  **Acceptance Criteria**:
  - [ ] File created: `backend/internal/utils/multi_instance.go`
  - [ ] File created: `backend/internal/utils/multi_instance_test.go`
  - [ ] `go test ./backend/internal/utils -run TestMultiInstance` → PASS

  **QA Scenarios**:
  ```
  Scenario: Multi-instance mode enabled with database backend (valid)
    Tool: Bash (go test)
    Preconditions: None
    Steps:
      1. Set env MULTI_INSTANCE_MODE=true
      2. Create EnvConfig with FILE_BACKEND=database
      3. Call ValidateMultiInstanceConfig(envConfig)
      4. Assert error == nil
    Expected Result: Validation passes
    Evidence: .sisyphus/evidence/task-2-valid-config.log

  Scenario: Multi-instance mode enabled with filesystem backend (invalid)
    Tool: Bash (go test)
    Preconditions: None
    Steps:
      1. Set env MULTI_INSTANCE_MODE=true
      2. Create EnvConfig with FILE_BACKEND=filesystem
      3. Call ValidateMultiInstanceConfig(envConfig)
      4. Assert error contains "Filesystem storage is not supported in multi-instance mode"
    Expected Result: Validation fails with clear error message
    Evidence: .sisyphus/evidence/task-2-invalid-config.log
  ```

  **Commit**: NO (groups with Wave 1)

---

- [ ] 3. Add MULTI_INSTANCE_MODE Environment Variable

  **What to do**:
  - Edit `backend/internal/common/env_config.go`
  - Add field to `EnvConfigSchema`: `MultiInstanceMode bool env:"MULTI_INSTANCE_MODE" envDefault:"false"`
  - Add validation in `ValidateEnvConfig()`: call `utils.ValidateMultiInstanceConfig(&EnvConfig)` if `MultiInstanceMode == true`
  - Update tests in `env_config_test.go` to cover multi-instance mode scenarios

  **Must NOT do**:
  - Do not change default behavior (default should be false for backward compatibility)
  - Do not add multiple configuration options (single boolean is sufficient)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Simple configuration field addition
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 2)
  - **Blocks**: Tasks 5, 6 (Bootstrap uses EnvConfig)
  - **Blocked By**: None

  **References**:
  - `backend/internal/common/env_config.go:40-82` - EnvConfigSchema definition
  - `backend/internal/common/env_config.go:132-228` - ValidateEnvConfig function
  - `backend/internal/common/env_config_test.go:122-131` - Boolean env var test pattern

  **Acceptance Criteria**:
  - [ ] `go build ./backend` → SUCCESS
  - [ ] `go test ./backend/internal/common -run TestParseEnvConfig` → PASS

  **QA Scenarios**:
  ```
  Scenario: Parse MULTI_INSTANCE_MODE=true from environment
    Tool: Bash
    Preconditions: None
    Steps:
      1. Set env MULTI_INSTANCE_MODE=true
      2. Run: go test ./backend/internal/common -run TestParseEnvConfig
      3. Assert EnvConfig.MultiInstanceMode == true
    Expected Result: Environment variable parsed correctly
    Evidence: .sisyphus/evidence/task-3-parse-env.log
  ```

  **Commit**: YES
  - Message: `feat(config): add MULTI_INSTANCE_MODE environment variable`
  - Files: `backend/internal/common/env_config.go`, `backend/internal/common/env_config_test.go`, `backend/internal/utils/multi_instance.go`, `backend/internal/utils/multi_instance_test.go`, `backend/internal/service/leader_election_service.go`
  - Pre-commit: `go test ./backend/internal/...`

---

