# C4 Architecture — DevMentor V1

> Design status: DRAFT V1 — logical architecture, technologies not yet
> chosen unless stated
>
> Implementation status: NOT IMPLEMENTED

This document describes DevMentor with the [C4 model](https://c4model.com):
System Context (C1), Containers (C2), Components (C3) and dynamic
diagrams for the main flows. The README shows a summary of C1–C3; this
document is the detailed version and the source of truth.

Related documents: [V1 Scope](../../01-scope.md),
[Use Cases](../../03-use-cases.md),
[Business Rules](../../requirements/business-rules.md),
[Non-Functional Requirements](../../requirements/non-functional-requirements.md),
[API documentation](../../api/README.md).

## Notation

| Shape | Meaning |
| --- | --- |
| `(["Name"])` rounded | Person |
| Rectangle | Software system, container or component |
| Cylinder | Data store |
| Dashed box | System or container boundary |
| Arrow label | What flows and how (protocol where it matters) |

Every element shows its type in italics. Technologies marked **TBD** are
not chosen yet; see [Open decisions](#open-decisions).

## C1 — System Context

``` mermaid
flowchart LR
    visitor(["Visitor<br/><i>Person</i><br/>Not logged in"])
    learner(["Learner<br/><i>Person</i><br/>Student or self-learner"])
    admin(["Admin<br/><i>Person</i><br/>Content and operations team"])

    system["DevMentor<br/><i>Software System</i><br/>Competency assessment, personalized learning paths,<br/>lessons, code practice, evaluation and AI mentoring"]

    llm["LLM Provider<br/><i>External Software System</i><br/>Text generation API"]

    visitor -->|"Reads the landing page, registers, logs in"| system
    learner -->|"Completes onboarding, follows learning paths,<br/>watches lessons, practices code, asks the AI Mentor"| system
    admin -->|"Manages content and users, monitors usage and AI cost"| system
    system -->|"Sends prompts with learning context,<br/>receives responses (HTTPS)"| llm
```

| Element | Type | Description |
| --- | --- | --- |
| Visitor | Person | Sees only the landing page, Register and Log in (BR-009). |
| Learner | Person | Students and self-learners in Vietnam; uses every learning feature after mandatory onboarding (BR-032). |
| Admin | Person | Manages courses, lessons, exercises, test cases, questions and users; monitors the system. |
| DevMentor | Software system | The system described here. |
| LLM Provider | External system | Generates AI Mentor answers, competency assessment questions, profile comments and "AI added" practice. One provider behind an adapter (AS-05, NFR-AI-007). |

The Code Execution Service is a container inside DevMentor. If an
external sandbox service is chosen instead (NFR open policy), it moves
to this diagram as an external system.

V1 has no email service, payment provider or third-party login (V1
Scope, Out of Scope).

## C2 — Containers

``` mermaid
flowchart TB
    learner(["Learner / Admin<br/><i>Person</i>"])

    subgraph boundary["DevMentor — System Boundary"]
        web["Web Application<br/><i>Container: SPA, TBD framework</i><br/>Landing, onboarding, learning UI,<br/>video player, code editor, admin UI"]
        api["Backend API<br/><i>Container: modular monolith, TBD language</i><br/>REST/JSON, business rules,<br/>authentication, authorization"]
        ai["AI Mentor Service<br/><i>Container or module</i><br/>Context building, retrieval, guardrails,<br/>content generation"]
        worker["Background Worker<br/><i>Container</i><br/>Grading pipeline, mastery, path updates,<br/>AI practice, streaks, statistics"]
        runner["Code Execution Service<br/><i>Isolated container</i><br/>Runs code and test cases in a sandbox<br/>C, C++, Python, Java, JavaScript, SQL"]
        db[("PostgreSQL<br/><i>Database</i><br/>Accounts, content, attempts,<br/>submissions, paths, progress, AI logs")]
        redis[("Redis<br/><i>Cache / queue</i><br/>Job queues, rate limits, quotas, cache")]
        vector[("Vector Store<br/><i>Data store, TBD</i><br/>Lesson and transcript embeddings")]
        object[("Object Storage<br/><i>Storage, TBD</i><br/>Lesson videos, learning material")]
    end

    llm["LLM Provider<br/><i>External system</i>"]

    learner -->|"HTTPS"| web
    learner -->|"Streams video (HTTPS, signed URL)"| object
    web -->|"REST/JSON, SSE for AI answers (HTTPS)"| api
    api -->|"SQL"| db
    api -->|"Enqueue jobs, counters"| redis
    api -->|"In-process call or internal HTTP"| ai
    api -->|"Signed URLs, uploads"| object
    worker -->|"Consume jobs"| redis
    worker -->|"SQL"| db
    worker -->|"Run / grade request"| runner
    worker -->|"Generate practice, comments"| ai
    ai -->|"Similarity search"| vector
    ai -->|"HTTPS"| llm
    ai -->|"Read lesson context"| db
```

| Container | Responsibility | Technology | Key NFRs |
| --- | --- | --- | --- |
| Web Application | All screens of the UI prototype (README §2.3) and the admin UI; in-browser editor; video player with checkpoints; keeps unsaved code drafts in the browser. | SPA, framework TBD; shadcn/ui style; code editor component TBD | NFR-UX-001–003, NFR-ACC-001 |
| Backend API | The only entry point for clients. Authentication, role and ownership checks, onboarding, learning paths, lessons, assessments, submissions, dashboard, admin. | Modular monolith, language TBD; OpenAPI 3.0 contract ([openapi.yaml](../../api/openapi.yaml)) | NFR-PERF-001–003, NFR-SEC-001–006 |
| AI Mentor Service | Builds the learning context, retrieves lesson material, calls the LLM, applies guardrails, generates assessment questions, profile comments and practice. Records tokens and cost of every call. | Module of the API process at first; may become its own service (NFR-AI-007) | NFR-AI-001–007 |
| Background Worker | Runs everything that must not block a request: grading jobs, mastery and path recalculation, AI practice generation and validation, streaks, statistics aggregation. | Same codebase as the API, separate process | NFR-PERF-004, NFR-EXEC-005 |
| Code Execution Service | Runs untrusted code: one sandbox per run, no network, CPU/memory/output/process limits, one runtime image per language. Never shares state between runs. | Sandbox technology TBD (container + seccomp, gVisor, Firecracker or external) | NFR-EXEC-001–007 |
| PostgreSQL | System of record (README §4.1). | PostgreSQL | NFR-BCP-001, NFR-CAP-001 |
| Redis | Job queues (grading, generation), rate limits (NFR-EXEC-006, NFR-SEC-003), AI quota counters (NFR-AI-003), short-lived cache. | Redis | NFR-CAP-002 |
| Vector Store | Embeddings of published lesson content and video transcripts for RAG (AS-06). | TBD (e.g. pgvector or a dedicated store) | NFR-AI-001 |
| Object Storage | Lesson videos uploaded by Admins (AS-11) and other material files; served to browsers with signed URLs. | TBD | Open policy: video delivery |

### Communication rules

- The browser talks only to the Backend API and, for video, to Object
    Storage through short-lived signed URLs issued by the API.
- Learner code reaches the Code Execution Service only through a job
    created by the API and processed by the Worker; it never runs in the
    API process (BR-011).
- Only the AI Mentor Service talks to the LLM Provider. Hidden tests,
    secrets and other learners' data are never part of a prompt
    (BR-014, NFR-AI-006).
- Long work (grading over the synchronous threshold, practice
    generation) is asynchronous: the API returns `202` and the client
    polls (see [API documentation](../../api/README.md#asynchronous-operations)).

## C3 — Components of the Backend API

``` mermaid
flowchart TB
    client["Web Application"]

    subgraph backend["Backend API — Container Boundary"]
        router["API Router<br/><i>Component</i><br/>Routing, validation against OpenAPI,<br/>error format, rate limits"]
        auth["Authentication & Access Control<br/><i>Component</i><br/>Tokens, roles, ownership,<br/>onboarding gate"]
        onboarding["Onboarding & Competency Profile<br/><i>Component</i><br/>Survey, assessment start,<br/>profile, confirmation"]
        assessment["Assessment<br/><i>Component</i><br/>Competency attempts, quizzes,<br/>scoring"]
        course["Course & Lesson<br/><i>Component</i><br/>Lessons, video, checkpoints,<br/>notes, completion"]
        exercise["Exercise, Submission & Evaluation<br/><i>Component</i><br/>Runs, submissions, verdicts,<br/>scores"]
        path["Learning Path & Mastery<br/><i>Component</i><br/>Path building, progress, mastery,<br/>gaps, recommendations"]
        dashboard["Dashboard & Streak<br/><i>Component</i><br/>Overview, to-do, streak"]
        mentorApi["AI Mentor Gateway<br/><i>Component</i><br/>Quota check, request to<br/>AI Mentor Service, SSE"]
        admin["Admin Management<br/><i>Component</i><br/>Content, test cases,<br/>questions, users"]
        monitoring["Statistics & Monitoring<br/><i>Component</i><br/>Aggregates, health"]
        jobs["Job Publisher<br/><i>Component</i>"]
        storage["Media Adapter<br/><i>Component</i><br/>Signed URLs"]
        repository["Repository / Persistence<br/><i>Component</i>"]
    end

    db[("PostgreSQL")]
    redis[("Redis")]
    object[("Object Storage")]
    aiService["AI Mentor Service"]

    client --> router
    router --> auth
    router --> onboarding & assessment & course & exercise & path & dashboard & mentorApi & admin & monitoring

    onboarding -->|"Start competency attempt"| assessment
    onboarding -->|"Confirmed profile → build path"| path
    onboarding -->|"Generate questions, comment"| aiService
    assessment -->|"Attempt scored"| path
    exercise -->|"Grading job"| jobs
    path -->|"Practice generation job"| jobs
    course -->|"Lesson completed"| path
    course --> storage
    dashboard -->|"Reads progress, paths, mastery"| path
    mentorApi --> aiService

    jobs --> redis
    storage --> object
    auth & onboarding & assessment & course & exercise & path & dashboard & admin & monitoring --> repository
    repository --> db
```

| Component | Responsibilities | API groups | Main rules |
| --- | --- | --- | --- |
| API Router | Routing, schema validation, request size limit, standard error format, rate limits. | all | NFR-SEC-006, NFR-EXEC-006 |
| Authentication & Access Control | Register, login, refresh, logout; role check; ownership check; `404` for foreign or unpublished resources; blocks learning APIs until onboarding is done. | Authentication | BR-001–006, BR-032 |
| Onboarding & Competency Profile | Survey; onboarding status; starting (and restarting) the competency assessment; profile with suggested handling per topic; confirmation that creates the path; one path in progress per language. | Onboarding | BR-016, BR-025, BR-031–034 |
| Assessment | Attempts for competency assessments and quizzes; answer saving; scoring; "I don't know"; quiz pass threshold; correct answers only after submit. | Assessments | BR-016, BR-023 |
| Course & Lesson | Lesson content by type; signed video URLs; checkpoint answers (not scored); notes; video position; manual completion of video and reading lessons. | Lessons | BR-007, BR-022, BR-036 |
| Exercise, Submission & Evaluation | Exercise detail with read-only files; runs and submissions as jobs; verdict and score from test results only; hidden test protection. | Exercises, Executions, Submissions | BR-011–015, BR-017 |
| Learning Path & Mastery | Builds a path from a confirmed profile (course choice and order by rules); item statuses; progress; mastery; knowledge gaps; next lesson; inserts validated AI practice. | Learning Paths, Learning | BR-022–026, BR-033, BR-035 |
| Dashboard & Streak | Continue learning, streak, lessons completed, average quiz score, to-do, competency per language. | Dashboard | BR-028, BR-037 |
| AI Mentor Gateway | Checks quota and the challenge lock, forwards to the AI Mentor Service, streams answers, stores feedback. | AI Mentor | BR-017, BR-018–021 |
| Admin Management | Content, test cases, questions, users and roles. | Admin | BR-005–008, BR-014 |
| Statistics & Monitoring | Aggregated usage, AI cost, error rates, health; budget alert. | Admin | BR-030, NFR-OBS-001 |
| Job Publisher | Puts grading and generation jobs on Redis queues. | — | NFR-EXEC-005 |
| Media Adapter | Signed URLs for video playback; uploads for Admins. | — | AS-11 |
| Repository / Persistence | Data access to PostgreSQL; transactions. | — | README §4.1 |

## C3 — Components of the AI Mentor Service

``` mermaid
flowchart TB
    gateway["AI Mentor Gateway (Backend API)"]
    workerIn["Background Worker"]

    subgraph ai_service["AI Mentor Service — Container Boundary"]
        orchestrator["Mentor Orchestrator<br/><i>Component</i><br/>Explain, Ask, Hint, Debug, Review"]
        generator["Content Generator<br/><i>Component</i><br/>Assessment questions, profile comment,<br/>path item reasons, practice"]
        context["Learning Context Builder<br/><i>Component</i><br/>Lesson, video position, code,<br/>sample results, history"]
        retrieval["Knowledge Retrieval<br/><i>Component</i><br/>RAG over lessons and transcripts"]
        guard["Response Guardrails<br/><i>Component</i><br/>No full solution, topic scope,<br/>prompt-injection checks"]
        validator["Generated Content Validator<br/><i>Component</i><br/>Question schema, practice tests"]
        llmAdapter["LLM Adapter<br/><i>Component</i><br/>Provider API, timeouts,<br/>token and cost logging"]
    end

    vector[("Vector Store")]
    db[("PostgreSQL")]
    llm["LLM Provider"]

    gateway --> orchestrator
    gateway --> generator
    workerIn --> generator
    orchestrator --> context
    orchestrator --> retrieval
    orchestrator --> guard
    orchestrator --> llmAdapter
    generator --> context
    generator --> validator
    generator --> llmAdapter
    retrieval --> vector
    context --> db
    llmAdapter --> llm
    llmAdapter -->|"model, tokens, cost"| db
```

| Component | Responsibilities | Main rules |
| --- | --- | --- |
| Mentor Orchestrator | Runs the five Mentor modes; guiding style by default; hint levels 1–3; refuses off-topic questions. | BR-018, BR-019, BR-020 |
| Content Generator | Generates competency assessment questions from the survey, the profile comment, item reasons, quiz explanations and "AI added" practice. Never decides course choice, order or verdicts. | BR-025, BR-034, BR-035 |
| Learning Context Builder | Assembles only allowed context: published lesson or statement, transcript around the video position, learner code, latest sample results, conversation history within the token limit. | BR-014, BR-018, NFR-AI-002 |
| Knowledge Retrieval | Retrieves lesson and transcript chunks; returns sources for citation. | AS-06 |
| Response Guardrails | Blocks complete solutions (regenerate once, then a generic message); checks scope; strips injected instructions. | BR-019, NFR-AI-005, NFR-AI-006 |
| Generated Content Validator | Checks generated questions (4 options, one correct answer, a concept tag); sends generated practice to the sandbox and accepts it only if the reference solution passes every test. | BR-034, BR-035 |
| LLM Adapter | The only code that knows the provider; timeouts; records model, input/output tokens and cost for every call. | BR-021, NFR-AI-001, NFR-AI-007 |

## C3 — Components of the Background Worker and Code Execution Service

``` mermaid
flowchart LR
    redis[("Redis queues")]

    subgraph worker["Background Worker — Container Boundary"]
        grading["Grading Job Handler<br/><i>Component</i>"]
        recalc["Learning Recalculator<br/><i>Component</i><br/>Mastery, gaps, path progress"]
        practice["Practice Generation Job<br/><i>Component</i>"]
        streak["Streak & Activity Job<br/><i>Component</i>"]
        stats["Statistics Aggregator<br/><i>Component</i>"]
    end

    subgraph runner["Code Execution Service — Container Boundary"]
        dispatcher["Run Dispatcher<br/><i>Component</i><br/>Limits, language selection"]
        sandbox["Sandbox Pool<br/><i>Component</i><br/>One runtime image per language,<br/>destroyed after each run"]
        comparator["Output Comparator<br/><i>Component</i>"]
    end

    db[("PostgreSQL")]
    ai["AI Mentor Service"]

    redis --> grading & practice & streak & stats
    grading --> dispatcher
    dispatcher --> sandbox
    sandbox --> comparator
    grading -->|"verdict, score"| db
    grading --> recalc
    recalc --> db
    recalc -->|"weak concept"| practice
    practice --> ai
    streak --> db
    stats --> db
```

| Component | Responsibilities | Main rules |
| --- | --- | --- |
| Grading Job Handler | Takes run and submission jobs, calls the Code Execution Service, stores results, verdict and evaluation. | BR-012, BR-013, BR-015 |
| Learning Recalculator | After every graded activity: mastery, knowledge gaps, path item statuses and progress, path completion. | BR-022–024, NFR-PERF-004 |
| Practice Generation Job | Asks the Content Generator for practice on a weak concept, waits for validation, inserts the item before the next chapter and flags "Learning path updated". | BR-035 |
| Streak & Activity Job | Records qualifying activities and updates the streak by day in `Asia/Ho_Chi_Minh`. | BR-028, AS-09 |
| Statistics Aggregator | Daily aggregates for Admin statistics. | BR-030 |
| Run Dispatcher | Applies limits and chooses the runtime image for the exercise language; mounts read-only exercise files. | BR-011, BR-012, NFR-EXEC-001–004 |
| Sandbox Pool | Isolated, network-less sandboxes destroyed after each run. | NFR-EXEC-003, NFR-EXEC-004 |
| Output Comparator | Compares output with expected output, ignoring trailing whitespace and blank lines. SQL comparison rules are an open question. | BR-013 |

## Dynamic Diagrams

### D1 — Onboarding to first learning path (UC03)

``` mermaid
sequenceDiagram
    actor L as Learner
    participant W as Web Application
    participant A as Backend API
    participant G as AI Mentor Service
    participant D as PostgreSQL

    L->>W: Answer survey (4 steps)
    W->>A: POST /onboarding/surveys
    A->>D: Save survey
    A-->>W: 201 survey (question count, minutes)
    L->>W: Start assessment
    W->>A: POST /competency-assessments
    A->>G: Generate questions for survey topics
    G-->>A: Validated questions (BR-034)
    A->>D: Save assessment and attempt
    A-->>W: 201 attempt (questions, no answers)
    loop Each question
        W->>A: PATCH /assessment-attempts/{id}
    end
    W->>A: POST /assessment-attempts/{id}/submit
    A->>D: Score attempt, update mastery, draft profile
    A->>G: Profile comment
    A-->>W: 200 result with profileId
    W->>A: GET /competency-profiles/{id}
    L->>W: Change topic handling (optional)
    W->>A: PATCH /competency-profiles/{id}/topics
    L->>W: Confirm and start learning
    W->>A: POST /competency-profiles/{id}/confirm
    A->>D: Choose course by rules, build path items
    A->>G: Item reasons
    A-->>W: 201 learning path
```

### D2 — Submitting code (UC09, UC05)

``` mermaid
sequenceDiagram
    actor L as Learner
    participant W as Web Application
    participant A as Backend API
    participant R as Redis
    participant K as Background Worker
    participant X as Code Execution Service
    participant D as PostgreSQL

    L->>W: Submit
    W->>A: POST /submissions
    A->>A: Check onboarding, rate limit, size
    A->>D: Create submission (PENDING)
    A->>R: Enqueue grading job
    alt Graded within the synchronous threshold
        A-->>W: 201 submission with verdict
    else Still running
        A-->>W: 202 submission (PENDING)
        loop Until finished
            W->>A: GET /submissions/{id}
        end
    end
    K->>R: Take job
    K->>X: Run all test cases
    X-->>K: Results per test
    K->>D: Verdict, score, runtime, memory
    K->>D: Mastery, path progress, streak (BR-022, BR-023, BR-028)
```

### D3 — Asking for a hint (UC11)

``` mermaid
sequenceDiagram
    actor L as Learner
    participant W as Web Application
    participant A as Backend API
    participant G as AI Mentor Service
    participant P as LLM Provider

    L->>W: Next hint
    W->>A: POST /ai/hint
    A->>A: Quota and challenge lock (BR-017, BR-021)
    A->>G: Hint request (exercise, code, level)
    G->>G: Build context without hidden tests
    G->>P: Prompt
    P-->>G: Response
    G->>G: Guardrail: no complete solution
    alt Blocked
        G->>P: Regenerate once
        P-->>G: Response
    end
    G-->>A: Hint level n
    A-->>W: 200 hint
```

### D4 — Quiz result adapts the path (UC04, BR-035)

``` mermaid
sequenceDiagram
    actor L as Learner
    participant W as Web Application
    participant A as Backend API
    participant K as Background Worker
    participant G as AI Mentor Service
    participant X as Code Execution Service
    participant D as PostgreSQL

    L->>W: Submit quiz
    W->>A: POST /assessment-attempts/{id}/submit
    A->>D: Score, update mastery
    A->>K: Practice generation job (weak concept)
    A-->>W: 200 result (pathUpdate PENDING)
    K->>G: Generate practice
    G-->>K: Statement, tests, reference solution
    K->>X: Run reference solution on all tests
    alt All tests pass
        K->>D: Insert "AI added" item before next chapter
    else Fails
        K->>G: Regenerate or drop
    end
    W->>A: GET /learning-paths/{id}
    A-->>W: Path with the new item
```

## Deployment

The deployment is not decided yet. The logical units to deploy are the
Web Application (static files), the Backend API, the Background Worker,
the Code Execution Service (on hosts that allow strong isolation),
PostgreSQL, Redis, the Vector Store and Object Storage. Targets for
capacity and availability are in NFR-CAP-* and NFR-REL-001. A
deployment diagram will be added once the cloud environment and sandbox
technology are chosen.

## Architecture Principles

- Start as a modular monolith; split out a service only when scaling,
    security isolation or independent deployment requires it.
- Learner code never runs in the API process; each run has its own
    sandbox.
- Verdicts and scores come only from test results, never from an LLM.
- Course choice, lesson order, mastery, gaps and recommendations are
    rule-based in V1. The LLM explains and generates content but never
    decides order or verdicts (AS-07).
- Generated practice enters a path only after its reference solution
    passes all its tests.
- Every LLM call has a timeout and a token limit and is logged with
    model, tokens and cost.
- The backend enforces authentication, role, ownership and the
    onboarding gate on every API; the frontend only hides what the user
    cannot use.

## Open Decisions

| Decision | Options | Related |
| --- | --- | --- |
| Frontend framework and code editor component | TBD | NFR-UX-003 |
| Backend language and framework | TBD | NFR-MAINT-002 |
| Sandbox technology | container + seccomp, gVisor, Firecracker, external service | NFR-EXEC-003, NFR open policies |
| SQL runtime and result comparison | TBD | BR-013 open question |
| LLM provider and model | TBD | AS-05 |
| Vector store | pgvector or a dedicated store | AS-06 |
| Object storage and video delivery | TBD; streaming or CDN | AS-11, NFR open policies |
| AI Mentor Service as a module or a separate service | module first | NFR-AI-007 |
