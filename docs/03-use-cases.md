# Actors, Roles and Use Cases

> Design status: DRAFT V1
>
> Implementation status: NOT IMPLEMENTED

Product name: **DevMentor**.

## Actors and Roles

| Role | Display name | Description |
| --- | --- | --- |
| `GUEST` | Visitor | Not logged in; can only see the landing page, Register and Log in |
| `LEARNER` | Learner | Completes onboarding, follows learning paths, studies, practices, takes quizzes, learns with the AI Mentor, views the dashboard |
| `ADMIN` | Admin | Manages content, test cases, questions, users, and monitors the system |

External systems that take part in use cases:

| Secondary actor | Role |
| --- | --- |
| Code Execution Service | Runs code in a sandbox and runs test cases |
| LLM Provider | Generates AI Mentor responses, competency assessment questions, profile comments and AI-added practice |

## Role × Feature Matrix

Features are grouped by the mind map feature groups.

| Mind map group | Feature | Visitor | Learner | Admin |
| --- | --- | --- | --- | --- |
| Authentication & Authorization | Register, log in | ✓ | | |
| Authentication & Authorization | Log out | | ✓ | ✓ |
| Assessment & Evaluation | Competency assessment, quizzes, coding challenges, code evaluation | | ✓ | ✓ (test) |
| Personalized Learning Path | Onboarding survey, competency profile, learning paths, next lesson, knowledge gaps, mastery, AI-added practice | | ✓ | ✓ (own data) |
| Courses & Lessons | Path detail (chapters and lessons), lessons, practice lessons | | ✓ | ✓ (check) |
| Coding Workspace | Code editor, run, submit | | ✓ | ✓ (test) |
| Learn with AI Mentor | Explain, Hint, Debug, Review, Q&A | | ✓ | ✓ (test) |
| Dashboard | Continue learning, streak, metrics, paths, to-do, competency profile | | ✓ | ✓ (own data) |
| Content Management & Admin | Manage courses, lessons, exercises, test cases, questions | | | ✓ |
| Content Management & Admin | Manage users and roles | | | ✓ |
| Content Management & Admin | System statistics and monitoring | | | ✓ |

Menus and buttons are hidden when the user lacks permission. Opening a
URL without a session goes to Log in (`401`); opening it with the wrong
role goes to the `403 Forbidden` screen. The backend always re-checks
permissions (BR-001, BR-005). A learner who has not finished onboarding
is sent back to it (BR-032).

## Use Case Diagram

``` mermaid
flowchart LR
    subgraph left["Actors"]
        direction TB
        guest(["Visitor"])
        learner(["Learner"])
    end

    subgraph system["DevMentor — System Boundary"]
        direction TB
        subgraph auth_g["Authentication & Authorization"]
            direction LR
            uc01["UC01 Register an account"]
            uc02["UC02 Log in and log out"]
        end
        subgraph assess_g["Assessment & Evaluation"]
            direction LR
            uc03["UC03 Complete onboarding and the competency assessment"]
            uc04["UC04 Take a quiz"]
            uc05["UC05 Solve a coding challenge"]
        end
        subgraph path_g["Personalized Learning Path"]
            direction LR
            uc06["UC06 Follow a learning path"]
        end
        subgraph course_g["Courses & Lessons"]
            direction LR
            uc08["UC08 Study a lesson"]
            uc09["UC09 Practice a coding exercise"]
        end
        subgraph ws_g["Coding Workspace"]
            direction LR
            uc10["UC10 Write and run code"]
        end
        subgraph ai_g["Learn with AI Mentor"]
            direction LR
            uc11["UC11 Get a hint / debugging help"]
            uc12["UC12 Ask the AI Mentor"]
            uc13["UC13 Get an AI code review"]
        end
        subgraph dash_g["Dashboard"]
            direction LR
            uc14["UC14 View the dashboard"]
        end
        subgraph admin_g["Content Management & Admin"]
            direction LR
            uc15["UC15 Manage learning content"]
            uc16["UC16 Manage users and roles"]
            uc17["UC17 Monitor the system"]
        end
    end

    subgraph right["Actors"]
        direction TB
        admin(["Admin"])
        runner["Code Execution Service"]
        llm["LLM Provider"]
    end

    guest --> uc01 & uc02
    learner --> uc02 & uc03 & uc04 & uc05 & uc06 & uc08 & uc09 & uc10 & uc11 & uc12 & uc13 & uc14
    admin --> uc02 & uc15 & uc16 & uc17
    uc01 -.->|include| uc03
    uc03 -.->|extend| uc06
    uc04 -.->|extend| uc06
    uc09 -.->|include| uc10
    uc05 -.->|include| uc10
    uc10 --- runner
    uc03 --- llm
    uc11 --- llm
    uc12 --- llm
    uc13 --- llm

    classDef actor fill:#666,color:#fff,stroke:#333
    classDef usecase fill:#1168bd,color:#fff,stroke:#0b4f9e
    classDef ext fill:#999,color:#fff,stroke:#555
    class guest,learner,admin actor
    class runner,llm ext
    class uc01,uc02,uc03,uc04,uc05,uc06,uc08,uc09,uc10,uc11,uc12,uc13,uc14,uc15,uc16,uc17 usecase
    style left fill:#fff,stroke:#bbb,stroke-dasharray:5 5
    style right fill:#fff,stroke:#bbb,stroke-dasharray:5 5
    style system fill:#f8fbff,stroke:#1168bd,stroke-dasharray:5 5
```

Visitors only see the landing page, Register and Log in (BR-009). UC07
(Explore and enroll in courses) was removed: the system picks the
course for each learning path (BR-025).

### Shared rules

``` mermaid
flowchart LR
    auth["Authenticate session"]:::shared
    authorize["Check role"]:::shared
    onboard["Check onboarding is complete"]:::rule
    own["Check ownership"]:::rule
    published["Check content is published"]:::rule
    validate["Validate input"]:::shared
    quota["Check rate limit / quota"]:::rule
    sandbox["Run in sandbox"]:::shared
    guard["AI response guardrail"]:::rule

    onboarding["UC03 Onboarding"] --> auth & own & validate
    learn["UC06/UC08 Paths and lessons"] --> auth & onboard & own & published
    assessUc["UC04 Quizzes"] --> auth & onboard & published & own & validate
    run["UC05/UC09/UC10 Run, submit code"] --> auth & onboard & published & validate & quota & sandbox
    ai["UC11/UC12/UC13 AI Mentor"] --> auth & onboard & published & quota & guard
    data["UC14 Dashboard"] --> auth & onboard & own
    adminUc["UC15/UC16/UC17 Admin"] --> auth & authorize & validate

    classDef shared fill:#1168bd,color:#fff,stroke:#0b4884
    classDef rule fill:#e8f1fc,color:#123,stroke:#3a7bd5
```

## User Story ↔ Use Case Traceability

| Use Case | User Stories | Main business rules |
| --- | --- | --- |
| UC01 Register an account | US01 | BR-001, BR-003, BR-032 |
| UC02 Log in and log out | US02, US03 | BR-004, BR-005, BR-006, BR-032 |
| UC03 Complete onboarding and the competency assessment | US32, US04, US33, US08 | BR-016, BR-025, BR-031, BR-032, BR-033, BR-034 |
| UC04 Take a quiz | US05, US34 | BR-016, BR-023, BR-035 |
| UC05 Solve a coding challenge | US06, US07 | BR-013, BR-014, BR-015, BR-017 |
| UC06 Follow a learning path | US08, US09, US10, US11, US13, US25, US34 | BR-022, BR-023, BR-024, BR-025, BR-026, BR-033, BR-035 |
| UC07 Explore and enroll in courses | US12 (removed) | — (removed) |
| UC08 Study a lesson | US14 | BR-007, BR-022, BR-036 |
| UC09 Practice a coding exercise | US15, US18, US07 | BR-012, BR-013, BR-014, BR-015, BR-022 |
| UC10 Write and run code | US16, US17 | BR-011, BR-012 |
| UC11 Get a hint or debugging help | US20, US21 | BR-014, BR-018, BR-019, BR-021 |
| UC12 Ask the AI Mentor | US19, US23 | BR-018, BR-021 |
| UC13 Get an AI code review | US22 | BR-018, BR-020, BR-021 |
| UC14 View the dashboard | US24, US25, US27 | BR-002, BR-022, BR-026, BR-028, BR-037 |
| UC15 Manage learning content | US28, US29 | BR-007, BR-008, BR-014 |
| UC16 Manage users and roles | US30, US03 | BR-005, BR-006 |
| UC17 Monitor the system | US31 | BR-021, BR-030 |

US03 (role-based access control and blocking unauthorized access) also
applies to every use case through the shared rules above.

## Use Case Specifications

### UC01 Register an account

- **Actor:** Visitor
- **Precondition:** not logged in
- **Main flow:** open Register (from a landing page CTA or the Register
    tab of the Register / Log in screen) → enter display name, email
    and password, and accept the terms → `POST /auth/register` → the
    backend normalizes the email, checks for duplicates and password
    strength → creates an account with role `LEARNER` and starts a
    session → the learner lands on the onboarding survey (UC03)
- **Alternative flows:**
    - A1. Email already exists → error on the email field.
    - A2. Invalid data (bad email format, weak password, terms not
        accepted) → error on each field; entered data is kept except
        the password.
    - A3. The learner presses Google or GitHub → not available in V1
        (the buttons are a future direction).
- **Permission:** public
- **Result:** an active `LEARNER` account is created and onboarding
    starts
- **Rules:** BR-001, BR-003, BR-032, NFR-SEC-002

### UC02 Log in and log out

- **Actor:** Visitor (a user who has an account), Learner, Admin
- **Precondition:** the account exists
- **Main flow (log in):** open the Log in tab → enter email and
    password → `POST /auth/login` → the backend verifies credentials and
    account status → returns an access token, a refresh token and a
    short profile → a learner lands on the Dashboard (or back on the
    onboarding step they left, if no competency profile is confirmed
    yet, BR-032); an admin lands on Statistics & Monitoring
- **Main flow (log out):** choose Log out → `POST /auth/logout` revokes
    the refresh token → the client clears its state and returns to the
    landing page
- **Alternative flows:**
    - A1. Wrong credentials or locked account → generic error (BR-004);
        stay on Log in.
    - A2. Too many attempts → `429`, try again later.
    - A3. The access token expires → the client calls
        `POST /auth/refresh`; if refresh fails, the user is sent to Log
        in (BR-005).
    - A4. The user opens a page their role does not allow → `403`
        screen (BR-005).
- **Permission:** public (log in); authenticated (log out)
- **Result:** a valid session is created or ended
- **Rules:** BR-004, BR-005, BR-006, BR-032, NFR-SEC-001, NFR-SEC-003

### UC03 Complete onboarding and the competency assessment

- **Actor:** Learner; secondary actor: LLM Provider
- **Precondition:** logged in; either the learner has no confirmed
    competency profile yet (after UC01), or they chose "Create new path"
    in My learning paths (BR-033)
- **Main flow:**
    1. Survey: answer 4 questions, one per step — language, goals,
        experience, self-assessment per topic (BR-031) →
        `POST /onboarding/surveys`.
    2. Intro: see the number of questions and estimated time → press
        "Start".
    3. Assessment: `POST /competency-assessments` → the LLM generates
        the questions from the survey (BR-034) → answer each question
        (A–D) or press "I don't know"; jump between questions with the
        numbered boxes; answers are saved with
        `PATCH /assessment-attempts/{id}` → submit
        (`POST /assessment-attempts/{id}/submit`).
    4. Analysis: a waiting screen shows four stages in order — scoring,
        building the competency profile, creating the learning path,
        choosing the first lesson.
    5. Competency profile: overall level, score, the AI Mentor's comment
        comparing results with the self-assessment, counts of topics to
        learn from scratch / review / skip, and each topic with its
        score and suggested handling → the learner changes any handling
        if they want (`PATCH /competency-profiles/{id}/topics`) →
        "Confirm and start learning"
        (`POST /competency-profiles/{id}/confirm`) → the learning path
        is created (BR-025) → Dashboard.
- **Alternative flows:**
    - A1. The learner leaves midway → the next login or learning screen
        sends them back to the same step; an unsubmitted attempt is
        resumed (BR-032).
    - A2. "Retake the assessment" on the profile → back to step 3 with
        new questions; the latest attempt replaces the profile. Not
        available after confirming (BR-016).
    - A3. The language chosen already has an `IN_PROGRESS` path → it
        cannot be chosen (BR-033).
    - A4. The LLM fails to generate questions or the comment → error
        with Retry; the survey answers are kept.
    - A5. Answers sent for a submitted attempt → `409`.
- **Permission:** `LEARNER`
- **Result:** a survey, a scored attempt, a confirmed competency profile
    and a new learning path for the chosen language
- **Rules:** BR-016, BR-023, BR-025, BR-031, BR-032, BR-033, BR-034

### UC04 Take a quiz

- **Actor:** Learner; secondary actor: LLM Provider
- **Precondition:** logged in; onboarding complete; the quiz lesson is
    `PUBLISHED`
- **Main flow:** open a quiz lesson from the learning path, the
    lesson sidebar or the Dashboard to-do list → start an attempt
    (`POST /quizzes/{id}/attempts`) → answer; some questions have
    several correct answers; jump between questions with the numbered
    boxes → submit → the system scores it and shows the result: pass or
    not, score per skill, the wrong answers with the AI Mentor's
    explanation → mastery is updated per concept → the learning path is
    updated if needed (BR-035) and the learner sees "Learning path
    updated" with the added practice
- **Alternative flows:**
    - A1. The learner leaves midway → the attempt can be resumed.
    - A2. Answers sent for a submitted attempt → `409`.
    - A3. Retake → a new, separate attempt.
    - A4. Not passed → the learner continues; practice may be added
        (BR-016, BR-035).
- **Permission:** `LEARNER`
- **Result:** a scored, immutable attempt; updated mastery; possibly an
    updated learning path
- **Rules:** BR-016, BR-022, BR-023, BR-035

### UC05 Solve a coding challenge

- **Actor:** Learner; secondary actor: Code Execution Service
- **Precondition:** logged in; the challenge lesson is `PUBLISHED` and
    in the learner's learning path
- **Main flow:** open the challenge from the learning path → the Coding
    Workspace shows the statement, sample tests and starter code →
    write and run code (include UC10) → submit → the system grades all
    tests and shows the verdict and code evaluation (score, runtime,
    memory) → mastery and path progress are updated
- **Alternative flows:**
    - A1. The learner asks for Hint, Debug or Review before the challenge
        is `ACCEPTED` → the action is unavailable, with an explanation
        (BR-017).
    - A2. Not accepted → the learner can see sample test output and
        submit again.
    - A3. Grading takes longer than the synchronous threshold → `202`
        with `submissionId`; the client polls `GET /submissions/{id}`.
- **Permission:** `LEARNER`
- **Result:** an immutable submission with verdict and evaluation
- **Rules:** BR-011, BR-013, BR-014, BR-015, BR-017

### UC06 Follow a learning path

- **Actor:** Learner
- **Precondition:** logged in; onboarding complete
- **Main flow:** open My learning paths → `GET /learning-paths` lists
    one card per language with status, level, date created, progress,
    number of skipped lessons, number of AI-added lessons and the next
    lesson; filter by All / In progress / Completed → open a path →
    `GET /learning-paths/{id}` shows the course card (level, chapters,
    lessons, estimated weeks, progress, "Continue") and the chapters as
    an accordion; each lesson shows its type (Video / Reading /
    Practice / Quiz / Challenge), duration and labels ("Quick review",
    "AI added", in progress, skipped) → the learner opens a lesson
    (UC08) or presses "Continue" for the next lesson (BR-026)
- **Alternative flows:**
    - A1. "Create new path" → UC03 for another language (BR-033).
    - A2. The learner opens a skipped lesson → it opens normally but
        does not count toward progress (BR-022).
    - A3. A completed path → "Review" instead of "Continue".
- **Permission:** `LEARNER` (own data only)
- **Result:** the learner knows where they stand and what to study next
- **Rules:** BR-002, BR-022, BR-023, BR-024, BR-025, BR-026, BR-033,
    BR-035

### UC07 Explore and enroll in courses

Removed. The system picks the course for each path (BR-025); there is
no catalog, course page or enrollment.

### UC08 Study a lesson

- **Actor:** Learner
- **Precondition:** logged in; the lesson is `PUBLISHED` and in one of
    the learner's paths
- **Main flow:** open a lesson from the learning path, the Dashboard or
    the course sidebar (which can be collapsed) → for a video lesson,
    watch the video with the Summary, Transcript and Notes tabs; at a
    checkpoint the video pauses for a question that is not scored, the
    learner gets right/wrong feedback and continues watching → for a
    reading lesson, read the content and code examples → optionally
    learn with the AI Mentor (UC12) in the side panel → mark the lesson
    complete (`POST /lessons/{id}/complete`) → path progress and streak
    are updated
- **Alternative flows:**
    - A1. The lesson does not exist or is not published → `404`.
    - A2. The learner writes notes → saved privately for that lesson
        (`PUT /lessons/{id}/notes`).
    - A3. The learner returns to a video → it resumes at the last
        position.
- **Permission:** `LEARNER`
- **Result:** the learner studies the content; progress is recorded
- **Rules:** BR-007, BR-022, BR-028, BR-036

### UC09 Practice a coding exercise

- **Actor:** Learner; secondary actor: Code Execution Service
- **Precondition:** logged in; the practice lesson is `PUBLISHED` (or
    was AI-generated for this learner, BR-035)
- **Main flow:** open a practice lesson from the learning path → the
    Coding Workspace shows the statement (requirements as a checklist,
    examples, an "AI added" label if generated), the editor with the
    learner's file and any read-only files, and the AI Mentor panel;
    the course sidebar is hidden by default → write code → run it
    (include UC10) → submit (`POST /submissions`) → the Code Execution
    Service runs every test → show the verdict, sample test detail,
    hidden test pass/fail counts, the console and the code evaluation
    (score, runtime, memory) → save to the submission history → update
    mastery and path progress
- **Alternative flows:**
    - A1. Not accepted → the learner can view sample test output, ask
        for Debug (UC11) and submit again.
    - A2. Grading takes longer than the synchronous threshold → `202`
        with `submissionId`; the client polls `GET /submissions/{id}`.
    - A3. Sandbox failure → verdict `SYSTEM_ERROR`, not counted as a
        failed attempt.
    - A4. Rate limit exceeded → `429`.
- **Permission:** `LEARNER`
- **Result:** an immutable submission with verdict and evaluation
- **Rules:** BR-011, BR-012, BR-013, BR-014, BR-015, BR-022, BR-023,
    NFR-EXEC-005

### UC10 Write and run code

- **Actor:** Learner; secondary actor: Code Execution Service
- **Precondition:** logged in; the Coding Workspace is open on an
    exercise or challenge
- **Main flow:** write code in the in-browser editor (in the exercise's
    language: C, C++, Python, Java, JavaScript or SQL) → enter optional
    custom input or choose the sample tests → press Run →
    `POST /executions` → the backend validates size and queues the run →
    the sandbox executes it with the exercise's read-only files →
    stdout, stderr, run time and status are returned → saved to the run
    history
- **Alternative flows:**
    - A1. Time/memory limit exceeded → stopped with status
        `TIME_LIMIT_EXCEEDED`/`MEMORY_LIMIT_EXCEEDED` and a clear
        message.
    - A2. Compile, runtime or syntax error → show stderr; suggest Debug
        (UC11).
    - A3. Code larger than 64 KB → `413`/`422`.
    - A4. Rate limit exceeded → `429`.
    - A5. Page reloaded → the unsaved draft is restored from the browser.
- **Permission:** `LEARNER`
- **Result:** an execution result; no verdict, no effect on progress
- **Rules:** BR-011, BR-012, NFR-EXEC-001–006

### UC11 Get a hint or debugging help

- **Actor:** Learner; secondary actor: LLM Provider
- **Precondition:** working on a practice exercise in the Coding
    Workspace (or on a challenge already `ACCEPTED`)
- **Main flow (Hint):** press "Next hint" in the AI Mentor panel →
    `POST /ai/hint` with the exercise, current code and the hint level
    already received → the AI Mentor Service builds the context (no
    hidden tests) → calls the LLM → the guardrail checks there is no
    complete solution → shows the next hint level, as a question that
    points to the relevant line where possible ("Go to line 12")
- **Main flow (Debug):** after a failed run or submission, press
    "Explain the failing test" → `POST /ai/debug` with the code and
    stderr / sample test results → the AI Mentor explains the cause,
    points to the related lines and asks a leading question instead of
    fixing the code
- **Alternative flows:**
    - A1. The highest hint level was already given → message that no
        new hints are available; suggest reviewing the lesson.
    - A2. The guardrail blocks the response → regenerate once; if still
        blocked, return a generic message.
    - A3. LLM error or more than 30 s → error message, Retry allowed,
        quota not charged.
    - A4. Daily quota used up → `429`.
    - A5. Coding challenge not yet `ACCEPTED` → unavailable (BR-017).
- **Permission:** `LEARNER`
- **Result:** the guidance is saved in the AI conversation with tokens
    and cost
- **Rules:** BR-014, BR-017, BR-018, BR-019, BR-021, NFR-AI-001–006

### UC12 Ask the AI Mentor

- **Actor:** Learner; secondary actor: LLM Provider
- **Precondition:** on a lesson or exercise
- **Main flow (Explain):** select a passage or concept in the lesson,
    or press a quick action ("Explain more simply", "Give another
    example") → `POST /ai/explain` → the AI Mentor explains it with an
    example, using the lesson and the current video position as
    context, and names the lesson section used
- **Main flow (Ask):** in the AI Mentor panel, which shows what it is
    following ("Following the lesson · Lesson 4 · @ 05:12") → type a
    question → `POST /ai/ask` → the AI Mentor Service takes the lesson
    context and retrieves related material (RAG) → calls the LLM →
    guardrail → the answer is streamed with code examples, links to
    points in the video and a question back to the learner → the
    learner can rate it helpful / not helpful
    (`POST /ai/messages/{id}/feedback`)
- **Alternative flows:**
    - A1. Off-topic question → politely declined, with a suggestion to
        return to the topic.
    - A2. LLM error or timeout → error message and a Retry button.
    - A3. Quota used up → `429`.
- **Permission:** `LEARNER`
- **Result:** the question and answer are saved in `AI_CONVERSATIONS`
- **Rules:** BR-018, BR-021, NFR-AI-*

### UC13 Get an AI code review

- **Actor:** Learner; secondary actor: LLM Provider
- **Precondition:** the learner has at least one submission for the
    exercise (for a challenge: an `ACCEPTED` submission)
- **Main flow:** press "Review code" in the AI Mentor panel, or open a
    submission from the history and choose Review → `POST /ai/review`
    with the submission → the AI Mentor returns feedback on correctness
    risks, readability, naming, structure and complexity, with short
    illustrative snippets → guardrail → the review is shown next to the
    code
- **Alternative flows:**
    - A1. No submission yet → Review is unavailable; the learner is asked
        to submit first.
    - A2. LLM error or timeout → error message, Retry allowed, quota not
        charged.
    - A3. Quota used up → `429`.
- **Permission:** `LEARNER` (own submissions only)
- **Result:** a saved review; verdict and score unchanged
- **Rules:** BR-017, BR-018, BR-020, BR-021

### UC14 View the dashboard

- **Actor:** Learner
- **Precondition:** logged in; onboarding complete
- **Main flow:** open the Dashboard → `GET /dashboard/overview` shows a
    greeting, the "Continue learning" card (current lesson, minutes of
    video left), the streak with the last 7 days, lessons completed
    (total and this week), the average quiz score, the paths being
    followed with progress and next lesson, and the competency profile
    with one tab per language → `GET /dashboard/todo` lists AI-added
    practice and quizzes not yet done → the learner opens a lesson,
    practice or quiz directly
- **Alternative flows:**
    - A1. No activity yet → empty state, not yet designed (see
        [Backlog gaps](01-scope.md#backlog-gaps)).
    - A2. Continue → opens the next lesson of the most recently active
        path (BR-026).
- **Permission:** `LEARNER` (own data only)
- **Result:** the learner sees their progress and stays motivated
- **Rules:** BR-002, BR-022, BR-026, BR-028, BR-037

### UC15 Manage learning content

- **Actor:** Admin
- **Precondition:** logged in with role `ADMIN`
- **Main flow:** open Manage courses → create/edit a course → add and
    reorder modules and lessons → write lesson content and code examples
    → create practice exercises (statement, starter code, concepts,
    required flag) or coding challenges → create test cases marked
    sample/hidden → run the reference solution against all tests →
    create quiz and placement questions tagged with concepts → set the
    status to `PUBLISHED`
- **Alternative flows:**
    - A1. Invalid data → error on each field.
    - A2. Publishing an exercise with no test case, or whose reference
        solution fails a test → rejected.
    - A3. Hiding content → status `HIDDEN`; related learning data is
        kept (BR-008).
    - A4. A non-admin calls the API → `403`.
- **Permission:** `ADMIN`
- **Result:** content is created/updated; only `PUBLISHED` content is
    visible to learners
- **Rules:** BR-005, BR-007, BR-008, BR-014

### UC16 Manage users and roles

- **Actor:** Admin
- **Precondition:** logged in with role `ADMIN`
- **Main flow:** open Manage users → search/filter → select a user →
    lock/unlock or change role → `PATCH /admin/users/{id}` → the list is
    updated
- **Alternative flows:**
    - A1. Removing one's own Admin role or locking oneself → rejected
        (`409`).
    - A2. Removing the last Admin → rejected.
- **Permission:** `ADMIN`
- **Result:** status/role updated; a locked account loses access
    immediately
- **Rules:** BR-005, BR-006

### UC17 Monitor the system

- **Actor:** Admin
- **Precondition:** logged in with role `ADMIN`
- **Main flow:** open Statistics & Monitoring → choose a date range →
    `GET /admin/stats` shows active users, enrollments, code runs,
    submissions, AI requests, tokens and cost per day →
    `GET /admin/monitoring` shows sandbox and LLM error rates and
    service health
- **Alternative flows:**
    - A1. Daily AI cost above the configured budget → an alert is shown
        (NFR-AI-004).
    - A2. A dependency is down → its health shows as unavailable while
        the rest of the page still loads.
- **Permission:** `ADMIN`
- **Result:** the admin understands usage, cost and system health
- **Rules:** BR-021, BR-030, NFR-OBS-001
