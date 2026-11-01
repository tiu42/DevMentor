# Non-Functional Requirements — DevMentor V1

> Design status: DRAFT V1 — every value is PROPOSED and needs team
> confirmation
>
> Verification status: PLANNED

These are realistic targets for a small-to-medium learning platform, not
a commercial SLA. An NFR counts as met only when its verification method
runs successfully under the [reference workload](#reference-workload).

## API Performance

| ID | Requirement | Rationale | Measurement | Verification | Status |
| --- | --- | --- | --- | --- | --- |
| NFR-PERF-001 | Regular read APIs (learning path detail, lessons, quizzes) have P95 ≤ 500 ms. | Navigating the learning content must feel smooth. | Server-side latency under the reference workload. | `PERF-READ-001` | PROPOSED / PLANNED |
| NFR-PERF-002 | Regular write APIs (survey answers, lesson completion, lesson notes, saving and submitting assessment and quiz answers, profile changes, content administration) have P95 ≤ 800 ms. | Data-entry actions respond quickly. | Server-side latency including DB commit. | `PERF-WRITE-001` | PROPOSED / PLANNED |
| NFR-PERF-003 | Dashboard, My learning paths, recommendations, knowledge gaps, mastery and admin statistics have P95 ≤ 1.5 s. | They aggregate data but must stay interactive. | Server-side latency. | `PERF-LEARN-001` | PROPOSED / PLANNED |
| NFR-PERF-004 | Mastery, learning path progress, Dashboard metrics and streak reflect a graded activity within 10 s of the result. AI-added practice (BR-035) may arrive later; see Open Policies. | Learners expect the dashboard and path to update right after they finish. | Time from verdict/attempt submission to updated read model. | `PERF-ASYNC-001` | PROPOSED / PLANNED |

## Code Execution

| ID | Requirement | Rationale | Measurement | Verification | Status |
| --- | --- | --- | --- | --- | --- |
| NFR-EXEC-001 | Each run (each test case) has a 2 s CPU time limit and a 5 s wall time limit. | Stops infinite loops and protects resources. | Time measured inside the sandbox. | `SEC-EXEC-001` | PROPOSED / PLANNED |
| NFR-EXEC-002 | Each sandbox is limited to 256 MB RAM, 64 processes, 64 KB of output; the learner's source file is at most 64 KB. | Stops fork bombs, memory exhaustion and huge output. | Sandbox configuration and over-limit tests. | `SEC-EXEC-002` | PROPOSED / PLANNED |
| NFR-EXEC-003 | The sandbox has no network access, cannot read host system files, runs as a non-root user, and has a read-only filesystem except for a temp directory. | Learner code is untrusted. | Sandbox escape test suite (network, file, privilege). | `SEC-EXEC-003` | PROPOSED / PLANNED |
| NFR-EXEC-004 | The sandbox is destroyed after each run; no state is shared between runs or between learners. | Prevents data leaks between users. | Test that writes a file in one run and reads it in the next. | `SEC-EXEC-004` | PROPOSED / PLANNED |
| NFR-EXEC-005 | A run returns its result in P95 ≤ 5 s; a submission (≤ 20 test cases) returns a verdict and evaluation in P95 ≤ 15 s, including queue time. | Keeps the practice loop continuous. | Time from request to result under the reference workload. | `PERF-EXEC-001` | PROPOSED / PLANNED |
| NFR-EXEC-006 | Each learner can make at most 10 runs/submissions per minute; beyond that, `429`. | Prevents abuse of sandbox resources. | Rate limit counter. | `INT-EXEC-RATE-001` | PROPOSED / PLANNED |
| NFR-EXEC-007 | Runtime and memory reported in the code evaluation vary by at most ±20% when the same code is submitted repeatedly under the reference workload. | Evaluation metrics must be trustworthy enough to compare solutions. | Repeated submissions of reference solutions. | `PERF-EXEC-002` | PROPOSED / PLANNED |

## AI Mentor

| ID | Requirement | Rationale | Measurement | Verification | Status |
| --- | --- | --- | --- | --- | --- |
| NFR-AI-001 | AI Mentor responses show the first token within P95 ≤ 3 s (streaming) and complete or fail within at most 30 s (60 s for code review). | Learners should not wait long when stuck. | Time to first token and server-side timeout. | `PERF-AI-001` | PROPOSED / PLANNED |
| NFR-AI-002 | Each request is limited to 8,000 input context tokens and 1,000 output tokens (1,500 for code review). | Controls cost and latency. | Token counts recorded in `AI_MESSAGES`. | `UNIT-AI-CTX-001` | PROPOSED / PLANNED |
| NFR-AI-003 | Each learner has at most 50 AI Mentor requests per day across all modes (BR-021). Whether generation calls (BR-034, BR-035) count is an open question. | Limits cost per user. | Daily counter. | `INT-AI-QUOTA-001` | PROPOSED / PLANNED |
| NFR-AI-004 | Average LLM cost ≤ USD 0.02 per active learner per day; Admins get an alert when daily cost exceeds the configured budget (BR-030). | Keeps operating cost sustainable. | Total `cost_usd` per day / active learners. | `OPS-AI-COST-001` | PROPOSED / PLANNED |
| NFR-AI-005 | On a standard evaluation set (≥ 50 cases per mode), ≥ 95% of Hint/Debug/Review responses contain no complete solution, and ≥ 85% of Explain/Ask/Hint/Debug/Review responses are rated correct and helpful. Generated assessment questions and practice are evaluated on their own sets (correct answer key, valid tests). | Makes sure the AI teaches rather than does the work (BR-019, BR-020). | Eval set scored manually or by LLM-as-judge with sample checks. | `EVAL-AI-001` | PROPOSED / PLANNED |
| NFR-AI-006 | The system defends against prompt injection from learner code and questions: separate system prompt, no secrets or hidden tests in prompts, response checks before returning. | Learner input is untrusted. | Prompt injection test suite. | `SEC-AI-001` | PROPOSED / PLANNED |
| NFR-AI-007 | The LLM provider is accessed through an adapter; changing provider or model only requires configuration and adapter changes, not business modules. | Avoids lock-in to one provider. | Architecture review, adapter tests. | `ARCH-AI-001` | PROPOSED / PLANNED |

## Capacity and Scalability

| ID | Requirement | Rationale | Measurement | Verification | Status |
| --- | --- | --- | --- | --- | --- |
| NFR-CAP-001 | Supports at least 1,000 accounts, 50 courses, 2,000 lessons/exercises/challenges, 5,000 assessment questions and 500,000 submission/execution records. | Covers V1 growth. | Seeded DB size. | `PERF-CAP-001` | PROPOSED / PLANNED |
| NFR-CAP-002 | Supports 200 concurrent users and 20 parallel code executions while still meeting NFR-PERF-001–004 and NFR-EXEC-005. | V1 concurrency target. | Load test with 200 virtual users. | `PERF-CAP-002` | PROPOSED / PLANNED |

## Security and Privacy

| ID | Requirement | Rationale | Measurement | Verification | Status |
| --- | --- | --- | --- | --- | --- |
| NFR-SEC-001 | Access tokens expire after 15 minutes; refresh tokens expire after 7 days, rotate on every use and are revoked on logout. | Reduces the risk of token leaks. | Check claims and refresh/logout behavior. | `SEC-AUTH-001` | PROPOSED / PLANNED |
| NFR-SEC-002 | Passwords have at least 8 characters with both letters and digits, and are stored with a slow hash (Argon2id or bcrypt). | Minimum protection. | Validation boundary tests, stored data check. | `UNIT-AUTH-PWD-001` | PROPOSED / PLANNED |
| NFR-SEC-003 | At most 5 failed logins per 15 minutes per account, with a similar limit per IP. | Prevents brute force. | Rate limit counter. | `SEC-AUTH-002` | PROPOSED / PLANNED |
| NFR-SEC-004 | Every protected API checks authentication, role and ownership in the backend and returns `401`/`403`/`404` per BR-005 (BR-001, BR-002). | The browser is not a security boundary. | Authorization tests for every endpoint and role. | `SEC-AUTHZ-001` | PROPOSED / PLANNED |
| NFR-SEC-005 | All traffic uses HTTPS (TLS 1.2+); LLM API keys and secrets are stored outside source code. | Protects data in transit and secrets. | Configuration check, secret scanning in the repo. | `SEC-CONF-001` | PROPOSED / PLANNED |
| NFR-SEC-006 | Request bodies are at most 1 MB (except Admin learning material uploads); all input is validated against the OpenAPI schema. | Blocks oversized payloads and malformed data. | Boundary tests. | `INT-VALID-001` | PROPOSED / PLANNED |
| NFR-PRIV-001 | Only email, display name, onboarding survey answers and necessary learning data are collected; a learner can request deletion of their account and personal data. | Data minimization, compliance with personal data protection rules. | Schema review, deletion procedure. | `PRIV-001` | PROPOSED / PLANNED |
| NFR-PRIV-002 | Learners are told that questions and code sent to the AI Mentor are passed to an external LLM provider. | Transparency about data flows. | UI and terms check. | `UX-PRIV-001` | PROPOSED / PLANNED |

## Reliability, Operations and Observability

| ID | Requirement | Rationale | Measurement | Verification | Status |
| --- | --- | --- | --- | --- | --- |
| NFR-REL-001 | Target availability 99.5% per month, excluding planned maintenance. | V1 operating target. | Successfully served minutes / eligible minutes. | `OPS-AVAIL-001` | PROPOSED / PLANNED |
| NFR-REL-002 | When the LLM or sandbox fails, lessons, quizzes, learning paths and the dashboard keep working; competency assessment generation, AI-added practice, the AI Mentor and code runs show a specific error with Retry for the affected feature only. | A failing dependency must not bring down the whole system. | Tests with dependencies turned off. | `INT-DEGRADE-001` | PROPOSED / PLANNED |
| NFR-BCP-001 | PostgreSQL is backed up daily and kept 30 days; RPO ≤ 24 hours, RTO ≤ 4 hours. | Limits data loss and recovery time. | Backup timestamps and restore drills. | `OPS-RESTORE-001` | PROPOSED / PLANNED |
| NFR-OBS-001 | The API writes structured logs with a correlation ID, exposes a health check endpoint, and emits metrics for code runs, AI latency, provider errors and AI cost that feed Statistics & Monitoring (US31); logs follow BR-029. | Safe incident diagnosis and cost tracking. | Log schema check, metrics dashboard. | `INT-OBS-001` | PROPOSED / PLANNED |

## Maintainability

| ID | Requirement | Rationale | Measurement | Verification | Status |
| --- | --- | --- | --- | --- | --- |
| NFR-MAINT-001 | Unit test coverage of critical business logic (grading, code evaluation, authorization, mastery, knowledge gaps, learning path building, recommendations, streak, AI guardrails, validation of AI-generated questions and practice) ≥ 80%; authorization ≥ 90%. | Focus testing on high-risk logic. | Line and branch coverage. | `CI-COV-001` | PROPOSED / PLANNED |
| NFR-MAINT-002 | Build, lint, module architecture checks, OpenAPI validation and static analysis all pass in CI. | Catch drift early. | CI status. | `CI-BUILD-001` | PROPOSED / PLANNED |
| NFR-MAINT-003 | Each language (C, C++, Python, Java, JavaScript, SQL in V1) is a runtime/image configuration for the sandbox plus an editor language mode; adding one does not change the grading module. | Six languages in V1 and more later (AS-02). | Architecture review. | `ARCH-EXEC-001` | PROPOSED / PLANNED |
| NFR-MAINT-004 | Thresholds for mastery, knowledge gaps, topic handling, assessment question counts, quiz pass score and the level → course mapping are configuration, not hard-coded. | Personalization rules are PROPOSED and will be tuned. | Architecture review, config tests. | `ARCH-LEARN-001` | PROPOSED / PLANNED |

## Usability

| ID | Requirement | Rationale | Measurement | Verification | Status |
| --- | --- | --- | --- | --- | --- |
| NFR-UX-001 | The UI is in Vietnamese; common programming terms stay in English. AI Mentor responses are in Vietnamese by default. | Fits the target learners. | UI and prompt review. | `UX-LANG-001` | PROPOSED / PLANNED |
| NFR-UX-002 | The web app is responsive from 360 px; the Lesson and Coding Workspace screens are optimized for screens ≥ 1024 px. | Learning works on many devices. | Tests at several screen sizes. | `UX-RESP-001` | PROPOSED / PLANNED |
| NFR-UX-003 | The in-browser code editor becomes interactive within 3 s on a typical connection and keeps typing latency under 50 ms for files up to 64 KB. | Writing code must feel native. | Browser performance measurements. | `UX-EDITOR-001` | PROPOSED / PLANNED |
| NFR-ACC-001 | Inputs have visible labels; status is not conveyed by color alone; keyboard navigation and focus meet the applicable WCAG 2.1 AA criteria; animations respect `prefers-reduced-motion`. | Accessibility. | Automated and manual checks. | `UX-A11Y-001` | PROPOSED / PLANNED |

## Reference Workload

- PostgreSQL holds data per NFR-CAP-001, with a realistic distribution
    across courses, exercises, submissions, assessment attempts and AI
    conversations.
- The load test uses 200 virtual users with this reference mix: 50%
    watching and reading lessons, 25% running/submitting code, 10% AI
    Mentor, 10% onboarding, quizzes, learning paths and dashboard, 5%
    admin and statistics.
- AI performance tests use the real LLM or a mock with simulated
    latency; results state which was used.
- Each test run records warm-up time, duration, environment and raw
    results.

## Open Policies

- Retention periods for logs, AI conversation history and submission
    source code.
- The monthly AI budget and system behavior when the global budget is
    exceeded (temporarily disable AI or switch to a cheaper model).
- The specific sandbox technology (container + seccomp, gVisor,
    Firecracker or an external service) — to be decided in an ADR.
- Target times for generating the competency assessment and the
    analysis step, and for inserting AI-added practice after a quiz.
- Video storage and delivery: formats, maximum size, streaming or
    CDN.
