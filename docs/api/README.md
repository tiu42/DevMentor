# API Documentation — DevMentor V1

> Design status: DRAFT V1
>
> Implementation status: NOT IMPLEMENTED

The contract between the Web Application and the Backend API is defined
in [openapi.yaml](openapi.yaml) (OpenAPI 3.0.3). This page explains the
conventions, the main flows and gives request/response examples. When
this page and the specification disagree, the specification wins.

To browse the specification, open it in any OpenAPI viewer (for
example Swagger Editor, or `npx @redocly/cli preview-docs
docs/api/openapi.yaml`). Validate it with
`npx @redocly/cli lint docs/api/openapi.yaml`.

Related documents: [Use Cases](../03-use-cases.md),
[Business Rules](../requirements/business-rules.md),
[C4 Architecture](../architecture/c4/README.md).

## Conventions

| Topic | Convention |
| --- | --- |
| Base URL | `/api/v1` |
| Format | JSON (`application/json`), UTF-8. Field names in `camelCase`, enum values in `UPPER_SNAKE_CASE`. |
| IDs and time | IDs are UUIDs. Timestamps are ISO 8601 in UTC; dates (`YYYY-MM-DD`) for streak days use `Asia/Ho_Chi_Minh` (AS-09). |
| Authentication | `Authorization: Bearer <accessToken>` on every endpoint except register, login and refresh. |
| Languages | `C`, `CPP`, `PYTHON`, `JAVA`, `JAVASCRIPT`, `SQL` (AS-02). |
| Pagination | `page` (from 1) and `pageSize` (default 20, max 100); responses carry `page`, `pageSize`, `total`, `items`. |
| Partial update | `PATCH` sends only the fields to change. Content is hidden or published through its `status` field (BR-008). |
| Request size | Body ≤ 1 MB; source code ≤ 64 KB (NFR-SEC-006, NFR-EXEC-002). |
| UI text | Messages meant for the learner (`title`, `errors[].message`, AI answers) are in Vietnamese (NFR-UX-001). |

## Authentication and Sessions

1. `POST /auth/register` or `POST /auth/login` returns an access token
    (15 minutes), a refresh token (7 days) and the current user with
    its onboarding state (NFR-SEC-001).
2. When a call returns `401`, the client calls `POST /auth/refresh`
    once. Refresh tokens rotate: each call returns a new one and
    invalidates the old one. If refresh fails, the client goes to Log
    in and returns to the requested page afterwards (BR-005).
3. `POST /auth/logout` revokes the refresh token.

How the client stores the refresh token (memory, storage or an
`HttpOnly` cookie) is an open question; the specification sends it in
the request body.

### Access checks

Every protected endpoint checks, in this order (BR-001, BR-002, BR-005,
BR-032):

| Check | Failure |
| --- | --- |
| Valid session | `401 UNAUTHENTICATED` |
| Role (`/admin/*` needs `ADMIN`) | `403 FORBIDDEN` |
| Onboarding finished (learning endpoints only) | `403 ONBOARDING_REQUIRED` |
| The resource belongs to the caller and is published | `404 NOT_FOUND` |

Endpoints open during onboarding: `/users/me`, `/onboarding/*`,
`/competency-assessments`, `/competency-profiles/*` and
`/assessment-attempts/*` for the competency attempt.

## Errors

Errors use [RFC 7807](https://www.rfc-editor.org/rfc/rfc7807) problem
details (`application/problem+json`) with a stable `code`:

``` json
{
  "type": "https://devmentor.example/problems/validation-failed",
  "title": "Dữ liệu không hợp lệ",
  "status": 422,
  "code": "VALIDATION_FAILED",
  "errors": [
    { "field": "password", "message": "Mật khẩu cần ít nhất 8 ký tự, gồm chữ và số" }
  ],
  "correlationId": "c1f0b2d4"
}
```

| Status | Code | When | Rule |
| --- | --- | --- | --- |
| 401 | `UNAUTHENTICATED` | No or expired session | BR-005 |
| 401 | `INVALID_CREDENTIALS` | Wrong email/password or locked account (same message) | BR-004 |
| 403 | `FORBIDDEN` | Wrong role | BR-005 |
| 403 | `ONBOARDING_REQUIRED` | Learning endpoint before the first profile is confirmed | BR-032 |
| 404 | `NOT_FOUND` | Missing, owned by someone else, or unpublished | BR-005, BR-007 |
| 409 | `EMAIL_TAKEN` | Register with an existing email | BR-003 |
| 409 | `PATH_IN_PROGRESS` | Survey for a language that has a path in progress | BR-033 |
| 409 | `PROFILE_CONFIRMED` | Retake, topic change or confirm after confirmation | BR-016, BR-032 |
| 409 | `NO_MATCHING_COURSE` | No published course for the language and level | BR-025 |
| 409 | `ATTEMPT_SUBMITTED` | Saving or submitting a submitted attempt | BR-016 |
| 409 | `NOT_MANUALLY_COMPLETABLE` | Marking a practice, quiz or challenge lesson complete | BR-022 |
| 409 | `CHALLENGE_NOT_SOLVED` | Hint, Debug or Review on a challenge not yet `ACCEPTED` | BR-017 |
| 409 | `SUBMISSION_REQUIRED` | Review without a submission | BR-020 |
| 409 | `NO_TEST_CASES`, `REFERENCE_SOLUTION_FAILS` | Publishing an invalid exercise (admin) | US29 |
| 409 | `LAST_ADMIN`, `SELF_CHANGE` | Invalid user change (admin) | BR-006 |
| 413 | `PAYLOAD_TOO_LARGE` | Body or source code over the limit | NFR-SEC-006 |
| 422 | `VALIDATION_FAILED` | Invalid fields | — |
| 429 | `RATE_LIMITED` | Too many logins or runs/submissions; see `Retry-After` | NFR-SEC-003, NFR-EXEC-006 |
| 429 | `AI_QUOTA_EXCEEDED` | Daily AI Mentor quota used; see `resetAt` | BR-021, NFR-AI-003 |
| 503 | `AI_UNAVAILABLE` | LLM error or timeout; not charged, retry allowed | BR-021 |

Error responses never include stack traces, hidden test data, tokens or
other sensitive data (BR-029).

## Main Flows

### Onboarding (UC03)

| Step | Call | Screen |
| --- | --- | --- |
| 1 | `GET /onboarding/topics?language=C` | Survey question 4 |
| 2 | `POST /onboarding/surveys` | Survey → intro |
| 3 | `POST /competency-assessments` | Competency assessment |
| 4 | `PATCH /assessment-attempts/{id}` (per answer) | Competency assessment |
| 5 | `POST /assessment-attempts/{id}/submit` | AI analysis |
| 6 | `GET /competency-profiles/{id}` | Competency profile |
| 7 | `PATCH /competency-profiles/{id}/topics` (optional) | Competency profile |
| 7b | `POST /competency-assessments` with `retake: true` (optional) | Back to step 4 |
| 8 | `POST /competency-profiles/{id}/confirm` | → Dashboard |

After login, the client reads `user.onboarding` (or
`GET /onboarding/status`) and resumes at `step` when `completed` is
false. "Create new path" in My learning paths starts again at step 1.

### Learning

| Screen | Calls |
| --- | --- |
| Dashboard | `GET /dashboard/overview`, `GET /dashboard/todo` |
| My learning paths | `GET /learning-paths?status=…` |
| Learning path detail | `GET /learning-paths/{pathId}` |
| Lesson + AI Mentor | `GET /lessons/{id}`, `PUT /lessons/{id}/position`, `POST /lessons/{id}/checkpoints/{cid}/answer`, `GET`/`PUT /lessons/{id}/notes`, `POST /lessons/{id}/complete`, `/ai/explain`, `/ai/ask` |
| Coding practice | `GET /exercises/{id}`, `POST /executions`, `POST /submissions`, `GET /exercises/{id}/submissions`, `/ai/hint`, `/ai/debug`, `/ai/review` |
| End-of-lesson quiz | `POST /quizzes/{id}/attempts`, `PATCH /assessment-attempts/{id}`, `POST /assessment-attempts/{id}/submit` |

## Asynchronous Operations

Runs and submissions return the result directly when they finish within
the synchronous threshold (`200` for runs, `201` for submissions).
Otherwise they return `202` with `status: QUEUED` or `PENDING`, and the
client polls `GET /executions/{id}` or `GET /submissions/{id}` until the
status is final (suggested interval: 1 s, NFR-EXEC-005). A
`SYSTEM_ERROR` is not counted as a failed attempt (BR-013).

After a quiz, `pathUpdate.status` is `PENDING` while AI practice is
generated and validated (BR-035). The client shows the result
immediately and refreshes `GET /learning-paths/{pathId}` to show
"Learning path updated" once the item is `APPLIED`.

## Streaming AI Answers

`POST /ai/ask` answers with Server-Sent Events (`text/event-stream`), so
the first words appear within NFR-AI-001:

``` text
event: delta
data: {"text":"Câu hỏi hay! "}

event: delta
data: {"text":"Thử nghĩ xem: khi p trỏ tới a + n thì *p là phần tử nào?"}

event: done
data: {"id":"7a1e…","conversationId":"3f9c…","mode":"ASK","content":"…","sources":[{"lessonId":"…","section":"Số học con trỏ","videoSecond":312}],"suggestedActions":["SIMPLER","ANOTHER_EXAMPLE"],"createdAt":"2026-11-02T08:15:03Z"}
```

An `error` event carries a problem object; errors found before the
stream starts (quota, permissions) are normal HTTP errors. The other
AI endpoints return a single JSON `AiMessage`.

## Examples

The examples follow the prototype's sample learner (C path, pointers).

### Register

``` http
POST /api/v1/auth/register
Content-Type: application/json

{ "displayName": "Hiếu", "email": "hieu@example.com", "password": "matkhau123", "acceptTerms": true }
```

``` json
201 Created
{
  "accessToken": "eyJ…",
  "refreshToken": "rt_…",
  "expiresIn": 900,
  "user": {
    "id": "6f1d…",
    "email": "hieu@example.com",
    "displayName": "Hiếu",
    "role": "LEARNER",
    "onboarding": { "completed": false, "step": "SURVEY" }
  }
}
```

### Submit the survey

``` http
POST /api/v1/onboarding/surveys
Authorization: Bearer eyJ…
Content-Type: application/json

{
  "language": "C",
  "goals": ["SCHOOL_COURSES", "DSA"],
  "experienceLevel": "BASICS",
  "selfAssessment": [
    { "conceptId": "c-variables", "rating": "CONFIDENT" },
    { "conceptId": "c-pointers", "rating": "UNSURE" },
    { "conceptId": "c-structs", "rating": "NOT_LEARNED" }
  ]
}
```

``` json
201 Created
{ "id": "s-01…", "language": "C", "questionCount": 12, "estimatedMinutes": 10, "createdAt": "2026-11-02T08:00:00Z", "…": "…" }
```

(Concept IDs are shortened for readability; real IDs are UUIDs.)

### Answer and submit the competency assessment

``` http
PATCH /api/v1/assessment-attempts/a-01…
Content-Type: application/json

{ "answers": [
  { "questionId": "q-07…", "selected": ["C"] },
  { "questionId": "q-08…", "dontKnow": true }
] }
```

``` http
POST /api/v1/assessment-attempts/a-01…/submit
```

``` json
200 OK
{
  "attemptId": "a-01…",
  "type": "COMPETENCY",
  "score": 67,
  "correctCount": 8,
  "questionCount": 12,
  "passed": null,
  "skills": [ { "conceptId": "c-pointers", "name": "Con trỏ", "correct": 1, "total": 3 } ],
  "profileId": "p-01…"
}
```

### Confirm the profile

``` http
PATCH /api/v1/competency-profiles/p-01…/topics
Content-Type: application/json

{ "topics": [ { "conceptId": "c-arrays", "handling": "LEARN" } ] }
```

``` http
POST /api/v1/competency-profiles/p-01…/confirm
```

``` json
201 Created
{
  "profile": { "id": "p-01…", "status": "CONFIRMED", "canRetake": false, "…": "…" },
  "path": {
    "id": "lp-01…",
    "language": "C",
    "courseTitle": "Lập trình C chuyên sâu",
    "level": "Trung cấp",
    "status": "IN_PROGRESS",
    "progress": { "completed": 0, "total": 22, "percent": 0 },
    "skippedCount": 3,
    "aiAddedCount": 0,
    "nextLesson": { "lessonId": "l-01…", "pathId": "lp-01…", "title": "Hàm nâng cao", "type": "VIDEO" },
    "createdAt": "2026-11-02T08:20:00Z"
  }
}
```

### Submit code that fails a test

``` http
POST /api/v1/submissions
Content-Type: application/json

{ "exerciseId": "e-01…", "sourceCode": "int sum(int *a, int n) { … }" }
```

``` json
201 Created
{
  "id": "sub-01…",
  "exerciseId": "e-01…",
  "status": "GRADED",
  "verdict": "WRONG_ANSWER",
  "score": 50,
  "passedTests": 2,
  "totalTests": 4,
  "maxRuntimeMs": 4,
  "maxMemoryKb": 1180,
  "sampleTests": [
    { "testId": "t-1…", "passed": true },
    { "testId": "t-2…", "passed": false, "input": "3\n1 2 3", "expectedOutput": "3", "actualOutput": "32767" }
  ],
  "hiddenTests": { "passed": 1, "failed": 1 },
  "submittedAt": "2026-11-02T09:01:00Z"
}
```

### Ask for a hint

``` http
POST /api/v1/ai/hint
Content-Type: application/json

{ "exerciseId": "e-01…", "currentCode": "…for (p = a; p <= a + n; p++)…" }
```

``` json
200 OK
{
  "id": "m-01…",
  "conversationId": "cv-01…",
  "mode": "HINT",
  "content": "Mảng có n phần tử thì địa chỉ cuối hợp lệ là gì? Điều kiện dừng ở dòng 12 có vượt qua nó không?",
  "lineRefs": [12],
  "hintLevel": 1,
  "maxHintLevel": 3,
  "suggestedActions": ["GO_TO_LINE", "NEXT_HINT"],
  "createdAt": "2026-11-02T09:02:00Z"
}
```

### Quiz result that adds practice

``` json
200 OK
{
  "attemptId": "qa-01…",
  "type": "QUIZ",
  "score": 80,
  "correctCount": 4,
  "questionCount": 5,
  "passed": true,
  "skills": [ { "conceptId": "c-pointers", "name": "Con trỏ", "correct": 4, "total": 5 } ],
  "review": [
    { "questionId": "q-3…", "selected": ["A"], "correctAnswers": ["A", "C"], "correct": false, "explanation": "Con trỏ chưa khởi tạo cũng là lỗi…" }
  ],
  "pathUpdate": { "status": "PENDING", "pathId": "lp-01…" }
}
```

## Endpoint Index

| Endpoint | Use case | Stories | Rules |
| --- | --- | --- | --- |
| `POST /auth/register` | UC01 | US01 | BR-001, BR-003, BR-032 |
| `POST /auth/login`, `/auth/refresh`, `/auth/logout`, `GET /users/me` | UC02 | US02, US03 | BR-004–006 |
| `GET /onboarding/status`, `GET /onboarding/topics`, `POST /onboarding/surveys` | UC03 | US32 | BR-031–033 |
| `POST /competency-assessments` | UC03 | US04 | BR-016, BR-034 |
| `GET`/`PATCH /competency-profiles/*`, `POST …/confirm` | UC03 | US33, US08 | BR-025, BR-032 |
| `GET /learning-paths`, `GET /learning-paths/{id}` | UC06 | US08, US13, US25 | BR-022, BR-025, BR-033 |
| `GET /lessons/{id}`, `PUT …/position`, `POST …/checkpoints/…/answer`, `…/notes`, `POST …/complete` | UC08 | US14 | BR-022, BR-036 |
| `GET /exercises/{id}` | UC05, UC09 | US06, US15 | BR-012, BR-014, BR-017 |
| `POST /executions`, `GET /executions/{id}` | UC10 | US16, US17 | BR-011, BR-012 |
| `POST /submissions`, `GET /submissions/{id}`, `GET /exercises/{id}/submissions` | UC05, UC09 | US07, US18 | BR-012–015 |
| `POST /quizzes/{id}/attempts`, `/assessment-attempts/*` | UC03, UC04 | US04, US05 | BR-016, BR-023, BR-035 |
| `POST /ai/explain`, `/ai/ask` | UC12 | US19, US23 | BR-018, BR-021 |
| `POST /ai/hint`, `/ai/debug` | UC11 | US20, US21 | BR-017, BR-019, BR-021 |
| `POST /ai/review`, `/ai/messages/{id}/feedback` | UC13, UC12 | US22, US23 | BR-020 |
| `GET /learning/recommendations`, `/learning/mastery`, `/learning/knowledge-gaps` | UC06 | US09–US11 | BR-023, BR-024, BR-026 |
| `GET /dashboard/overview`, `/dashboard/todo` | UC14 | US24, US25, US27 | BR-028, BR-037 |
| `/admin/courses*`, `/admin/modules*`, `/admin/lessons*`, `/admin/exercises*`, `/admin/test-cases*`, `/admin/assessments*`, `/admin/assessment-questions*` | UC15 | US28, US29 | BR-007, BR-008, BR-014 |
| `GET /admin/users`, `PATCH /admin/users/{id}` | UC16 | US30 | BR-006 |
| `GET /admin/stats`, `/admin/monitoring` | UC17 | US31 | BR-030 |

## Open Questions

- Where the client keeps the refresh token (body, as specified, or an
    `HttpOnly` cookie).
- How Admins upload lesson videos (direct upload with a signed URL is
    likely); admin screens are not designed yet.
- Middle values of `experienceLevel` (only three of the five labels
    are known from the prototype).
- `AdminStats.newLearningPaths` replaces "enrollments" in BR-030;
    confirm when the admin flow is designed.
- Open business questions that affect fields (quiz pass threshold,
    which attempt counts for the quiz average, SQL grading) are listed
    in the [business rules](../requirements/business-rules.md#open-questions).
