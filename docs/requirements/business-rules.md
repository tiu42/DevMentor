# Business Rules — DevMentor V1

> Design status: DRAFT V1
>
> Implementation status: NOT IMPLEMENTED

Each rule has a stable `BR-xxx` ID so acceptance criteria, APIs, tests
and the traceability matrix can reference it. Status:

- **CONFIRMED:** stated in the product mind map, in the README (User
    Stories, Acceptance Criteria or architecture notes), or decided by
    the product owner during UI design (UI canvas "DevMentor — UI",
    2026-11-01).
- **PROPOSED:** a new detail that clarifies behavior and needs team
    confirmation. Values marked "taken from the UI prototype" come from
    the prototype's sample data, not from an explicit decision.
- **REMOVED:** no longer part of V1. The ID is kept so old references
    stay traceable and is never reused.

Rules are grouped by the mind map feature groups.

## Group A — Authentication & Authorization

### BR-001 — Roles

- **Source:** mind map (Role-based access control), README §1.4,
    [Use Cases](../03-use-cases.md)
- **Related US:** US01–US34
- **Related API:** all APIs
- **Rule:** V1 has only the roles `GUEST` (not logged in), `LEARNER`
    and `ADMIN`. A newly registered account is always `LEARNER`.
    Permissions are decided by the backend; hiding buttons or screens in
    the frontend is not a security control.
- **Status:** CONFIRMED

### BR-002 — Ownership of learning data

- **Source:** AC US03, US11, US24
- **Related US:** US03–US11, US13–US25, US27, US32–US34
- **Related API:** `/users/me`, `/onboarding/*`,
    `/competency-assessments`, `/competency-profiles/*`,
    `/assessment-attempts/*`, `/learning-paths/*`, `/lessons/*/notes`,
    `/executions`, `/submissions`, `/ai/*`, `/learning/*`, `/dashboard/*`
- **Rule:** A learner can only read and write their own data: survey
    answers, competency assessment attempts, competency profiles,
    learning paths, lesson notes, code runs, submissions, AI
    conversations, quiz attempts, progress, mastery, recommendations,
    AI-generated practice (BR-035) and streak. Admins also see only
    their own learning data in learner screens; aggregated data is
    available through BR-030.
- **Status:** CONFIRMED

### BR-003 — Unique email

- **Source:** AC US01
- **Related US:** US01
- **Related API:** `POST /auth/register`
- **Rule:** Email is the login identifier, unique in the system,
    compared case-insensitively and normalized (trim, lowercase) before
    it is stored.
- **Status:** CONFIRMED (normalization is PROPOSED)

### BR-004 — Login error message

- **Source:** AC US02
- **Related US:** US02
- **Related API:** `POST /auth/login`
- **Rule:** A wrong email, a wrong password or a locked account all
    return the same generic error, which does not reveal whether the
    email exists. Login attempts are limited by NFR-SEC-003.
- **Status:** CONFIRMED

### BR-005 — Blocking unauthorized access

- **Source:** mind map (Block access when unauthorized), AC US03
- **Related US:** US03, and every story with a protected API
- **Related API:** all protected APIs
- **Rule:**
    - No valid session → `401`; the UI redirects to Log in and returns
        to the requested page after login.
    - Valid session but insufficient role (for example a learner
        calling `/admin/*`) → `403`; the UI shows the 403 screen.
    - A resource owned by another user, or content that is not
        published (BR-007) → `404`, so the API does not reveal that the
        resource exists.
    - Logout revokes the refresh token; a revoked or expired token is
        treated as no session.
- **Status:** CONFIRMED (the `401`/`403`/`404` split is PROPOSED)

### BR-006 — Locking accounts and changing roles

- **Source:** AC US30
- **Related US:** US30
- **Related API:** `PATCH /admin/users/{userId}`
- **Rule:** A locked account (`is_active = false`) cannot log in, and
    every existing token is rejected on the next request. An Admin
    cannot remove their own Admin role or lock their own account. The
    system always keeps at least one active Admin.
- **Status:** CONFIRMED

## Group B — Courses & Lessons

### BR-007 — Visibility by publication status

- **Source:** AC US13, US14, US28
- **Related US:** US13–US15, US28, US29
- **Related API:** `/learning-paths/*`, `/lessons/*`, `/exercises/*`
- **Rule:** Courses, modules, lessons, exercises and quizzes have status
    `DRAFT`, `PUBLISHED` or `HIDDEN`. Learners only see `PUBLISHED`
    content whose parents are all `PUBLISHED`. Accessing content that
    does not qualify returns `404`. AI-generated practice (BR-035) is
    visible only to the learner it was generated for.
- **Status:** CONFIRMED (status names and the parent rule are PROPOSED)

### BR-008 — Hide instead of delete

- **Source:** AC US28, US29
- **Related US:** US28, US29
- **Related API:** `PATCH /admin/*` (`status` field)
- **Rule:** Admins hide content by setting it to `HIDDEN`; it is not
    physically deleted. Submissions, progress, mastery and AI
    conversations linked to hidden content are kept.
- **Status:** CONFIRMED

### BR-009 — Visitor scope

- **Source:** README §1.4, §2.1; UI decision: no course catalog
- **Related US:** US01, US02
- **Related API:** `POST /auth/register`, `POST /auth/login`
- **Rule:** Visitors can only see the landing page, Register and Log in.
    There is no public course catalog or public course page. Every
    learning feature (onboarding, competency assessment, learning paths,
    lessons, exercises, the Coding Workspace, quizzes and the AI Mentor)
    requires login.
- **Status:** CONFIRMED

### BR-010 — Enrollment

- **Status:** REMOVED. Learners no longer browse or enroll in courses;
    the system picks the course when it builds a learning path (BR-025,
    BR-033).

### BR-036 — Lesson types and video lessons

- **Source:** UI prototype (Learning path detail, Lesson screens); UI
    decision: self-hosted video
- **Related US:** US13, US14, US15, US05, US06
- **Related API:** `GET /lessons/{lessonId}`,
    `POST /lessons/{lessonId}/checkpoints/{checkpointId}/answer`,
    `GET|PUT /lessons/{lessonId}/notes`
- **Rule:**
    - Each lesson has exactly one type: `VIDEO`, `READING`, `PRACTICE`,
        `QUIZ` or `CHALLENGE`. Modules are shown as "chapters".
    - Video files are uploaded by Admins and stored in Object Storage.
        A video lesson shows the video with Summary, Transcript and
        Notes tabs.
    - A video lesson may have checkpoint questions that pop up at a set
        time in the video. They are **not scored**; the learner gets
        right/wrong feedback and continues watching.
    - Notes are private to the learner (BR-002).
- **Status:** CONFIRMED (checkpoint effect on mastery is an open
    question)

## Group C — Coding Workspace, Grading and Code Evaluation

### BR-011 — Isolated code execution

- **Source:** AC US17, README §3 architecture notes
- **Related US:** US06, US15, US17, US18, US34
- **Related API:** `/executions`, `/submissions`
- **Rule:** Learner code never runs in the Backend API process. Each run
    takes place in its own sandbox with no network, and with limits on
    time, memory, output size and number of processes per
    NFR-EXEC-001–004. A run that exceeds a limit is stopped with status
    `TIME_LIMIT_EXCEEDED` or `MEMORY_LIMIT_EXCEEDED`. The sandbox has a
    runtime for each V1 language: C, C++, Python, Java, JavaScript and
    SQL (AS-02).
- **Status:** CONFIRMED (limit values are PROPOSED)

### BR-012 — Run vs. submit

- **Source:** AC US17, US18
- **Related US:** US17, US18
- **Related API:** `POST /executions`, `POST /submissions`
- **Rule:**
    - **Run (execution):** runs with custom input or the sample tests;
        creates no official verdict and does not affect progress or
        mastery.
    - **Submit (submission):** runs all test cases (sample and hidden);
        creates a verdict and evaluation, is stored in history and
        cannot be changed afterwards.
    - No limit on the number of submissions in V1 other than rate
        limits (NFR-EXEC-006); progress uses the best result.
    - An exercise has one file the learner edits and may have extra
        read-only files supplied by the exercise (for example `main.c`).
        Only the editable file is stored as the learner's source code.
- **Status:** CONFIRMED (unlimited submissions and best-result rule are
    PROPOSED)

### BR-013 — Verdict

- **Source:** AC US18, README §3 architecture notes
- **Related US:** US06, US18
- **Related API:** `POST /submissions`, `GET /submissions/{id}`
- **Rule:** The verdict is decided only by execution results against
    the test cases, **never** by an LLM. Verdict values: `ACCEPTED`,
    `WRONG_ANSWER`, `RUNTIME_ERROR`, `TIME_LIMIT_EXCEEDED`,
    `MEMORY_LIMIT_EXCEEDED`, `COMPILE_ERROR`, `SYSTEM_ERROR`. A solution
    is `ACCEPTED` when it passes every test case. Output comparison
    ignores trailing whitespace and trailing blank lines. A
    `SYSTEM_ERROR` does not count as a failed attempt.
- **Status:** CONFIRMED (verdict list and comparison rule are PROPOSED)

### BR-014 — Hidden test protection

- **Source:** AC US18, US29
- **Related US:** US06, US18, US20, US21, US22, US29, US34
- **Related API:** all learner APIs, `/ai/*`
- **Rule:** The input and expected output of hidden tests are never
    returned by a learner API. Learners only see how many hidden tests
    passed and failed. Hidden test content is never placed in a prompt
    sent to the LLM for the AI Mentor. This also applies to the hidden
    tests of AI-generated practice (BR-035).
- **Status:** CONFIRMED

### BR-015 — Code evaluation score

- **Source:** mind map (Code evaluation), AC US07
- **Related US:** US06, US07
- **Related API:** `GET /submissions/{id}`
- **Rule:** Every submission gets an evaluation:
    - `score = passed_tests / total_tests × 100`, rounded to a whole
        number;
    - `max_runtime_ms` and `max_memory_kb` across all tests, plus
        per-test runtime and memory for sample tests;
    - the best score per learner and exercise is kept.

    The evaluation is computed only from execution results. AI code
    review (BR-020) is separate feedback and never changes the score.
- **Status:** CONFIRMED (formula and metrics are PROPOSED)

## Group D — Assessment & Evaluation

### BR-016 — Competency assessment and quizzes

- **Source:** AC US04, US05; UI decisions: "competency assessment"
    instead of "placement test", AI-generated questions, no pass/fail
- **Related US:** US04, US05, US08, US32
- **Related API:** `POST /competency-assessments`,
    `POST /quizzes/{quizId}/attempts`,
    `PATCH /assessment-attempts/{attemptId}`,
    `POST /assessment-attempts/{attemptId}/submit`
- **Rule:**
    - **Common:** a submitted attempt cannot be changed. Scores are
        computed automatically from the stored correct answers. An
        unsubmitted attempt keeps its answers and can be resumed. Every
        answered question tagged with a concept updates mastery
        (BR-023).
    - **Competency assessment:** the questions are generated by the LLM
        for each learner from their survey (BR-031, BR-034). Number of
        questions per topic: 3 for "Unsure", 1 for "Confident", 0 for
        "Not learned". Each question has 4 options (A–D) and one
        correct answer. The learner may answer "I don't know" instead
        of guessing; it scores 0. There is no pass or fail: the result
        is used only to build the learning path. The learner can retake
        the assessment (new questions) while the competency profile is
        not yet confirmed; the latest submitted attempt replaces the
        profile. After the profile is confirmed (BR-032), it cannot be
        retaken for that path.
    - **Quiz:** a lesson of type `QUIZ` at the end of a lesson group.
        A question may have one or several correct answers ("select all
        correct answers"). Correct answers and AI Mentor explanations
        of wrong answers are shown only after submission. Quizzes can be
        retaken without limit; each try is a separate attempt. The
        result is "passed" when the score reaches a configurable
        threshold. Failing does not block the learner; the system adds
        practice instead (BR-035).
- **Status:** CONFIRMED (questions-per-topic counts are taken from the
    UI prototype and are PROPOSED; the quiz pass threshold is an open
    question)

### BR-017 — Coding challenges

- **Source:** mind map (Coding challenges), AC US06; UI decision:
    challenges are part of the learning path
- **Related US:** US06
- **Related API:** `GET /lessons/{lessonId}`, `POST /submissions`
- **Rule:** A coding challenge is a lesson of type `CHALLENGE` in a
    course, with one exercise of `type = CHALLENGE`. It appears in the
    learning path like any other lesson; there is no separate challenge
    list. It is run and graded with the same rules as practice
    exercises (BR-011 to BR-015). AI Mentor Hint, Debug and Review are
    unavailable for a challenge until the learner has an `ACCEPTED`
    submission for it. V1 has no leaderboard, ranking or time limit per
    challenge. Challenge results count toward mastery.
- **Status:** CONFIRMED (no-ranking is PROPOSED)

## Group E — Learn with AI Mentor

### BR-018 — AI context

- **Source:** AC US19–US23; UI prototype (Lesson, Practice screens)
- **Related US:** US19–US23
- **Related API:** `/ai/explain`, `/ai/ask`, `/ai/hint`, `/ai/debug`,
    `/ai/review`
- **Rule:** Every AI Mentor request is tied to a lesson or exercise the
    learner can access. The context sent to the LLM contains only:
    published lesson content (including the video transcript) or
    exercise statement, the current position in the video, the
    learner's current code, the latest run result with sample test data
    (if any), and the conversation history within the token limit. The
    AI Mentor answers in a guiding style by default: it asks leading
    questions rather than giving the answer, and the learner can ask
    for a simpler or more detailed explanation. Answers may link to a
    point in the lesson video. The AI Mentor answers only within the
    scope of learning programming; off-topic questions are declined
    politely.
- **Status:** CONFIRMED (topic restriction is PROPOSED)

### BR-019 — Step-by-step guidance, no solutions

- **Source:** mind map (Step-by-step hints, Debug), AC US20, US21
- **Related US:** US20, US21
- **Related API:** `/ai/hint`, `/ai/debug`
- **Rule:**
    - Hints have at most 3 levels: (1) idea direction, (2) algorithm or
        structure hint, (3) concrete hint at pseudo-code level. Each
        request returns only the next level.
    - Debug points to the location and cause of the error and suggests
        a direction to fix it; it does not rewrite the whole solution.
    - AI responses pass through a guardrail step; if a complete
        solution is detected, the response is blocked and regenerated
        once, or replaced with a generic message.
- **Status:** CONFIRMED (3 hint levels and the guardrail mechanism are
    PROPOSED)

### BR-020 — AI code review

- **Source:** mind map (Review code), AC US22
- **Related US:** US22
- **Related API:** `POST /ai/review`
- **Rule:** Review is available only for code the learner has
    submitted. Feedback covers correctness risks, readability, naming,
    structure and time/space complexity, and may include short
    illustrative snippets, but never a complete rewritten solution. A
    review does not change the verdict or score (BR-013, BR-015). For
    coding challenges, BR-017 applies.
- **Status:** PROPOSED

### BR-021 — AI quota and failures

- **Source:** AC US19, README §3 architecture notes
- **Related US:** US19–US23, US31
- **Related API:** `/ai/*`
- **Rule:** Each learner has a daily AI request quota (NFR-AI-003)
    shared by all AI Mentor modes. Exceeding it returns `429` with the
    time the quota resets. Requests that fail because of the LLM
    provider or a timeout do not count toward the quota; the learner may
    retry. Every LLM call, including content generation (BR-034,
    BR-035) and profile comments, records the model, input/output
    tokens and estimated cost.
- **Status:** CONFIRMED (quota values and no-charge-on-failure are
    PROPOSED; whether generation calls count toward the quota is an
    open question)

## Group F — Personalized Learning Path and Progress

### BR-031 — Onboarding survey

- **Source:** UI prototype (Onboarding survey screen)
- **Related US:** US32
- **Related API:** `POST /onboarding/surveys`
- **Rule:** The survey has 4 questions, one per step, with Back and
    Continue:
    1. Language to learn: one of C, C++, Python, Java, JavaScript, SQL,
        or "Other" (free text).
    2. Goals (several allowed): do well in school courses, data
        structures and algorithms, job interviews, personal projects,
        embedded/systems, hobby, or "Other" (free text).
    3. Experience with that language: 5 levels, from "Never
        programmed" to "Use it regularly".
    4. Self-assessment of each topic of that language: "Not learned",
        "Unsure" or "Confident".

    After the survey, an intro step shows the number of assessment
    questions (BR-016) and the estimated time.
- **Status:** CONFIRMED (the handling of "Other" language is an open
    question)

### BR-032 — Mandatory onboarding

- **Source:** UI decision
- **Related US:** US01, US02, US04, US32, US33
- **Related API:** `GET /onboarding/status`,
    `POST /competency-profiles/{profileId}/confirm`
- **Rule:** Onboarding has no skip: survey → competency assessment →
    competency profile → learning path. A learner without a confirmed
    profile is sent back to the step they left whenever they log in or
    open a learning screen; the Dashboard is available only after the
    first profile is confirmed. On the competency profile, the system
    suggests a handling for each topic (BR-025) and the learner can
    change it before confirming. Confirming creates the learning path.
- **Status:** CONFIRMED

### BR-022 — Lesson completion and path progress

- **Source:** AC US14, US24
- **Related US:** US13, US14, US15, US24, US25
- **Related API:** `POST /lessons/{lessonId}/complete`,
    `GET /learning-paths/{pathId}`, `GET /dashboard/overview`
- **Rule:** A lesson is "in progress" once opened and not complete. It
    is complete when:
    - `VIDEO`, `READING`: the learner marks it complete;
    - `PRACTICE`, `CHALLENGE`: the exercise has at least one `ACCEPTED`
        submission;
    - `QUIZ`: the learner has submitted at least one attempt.

    Path progress = completed items / items that are not skipped.
    Skipped items can still be opened but do not count toward progress.
    A path is `COMPLETED` when every item that is not skipped is
    complete.
- **Status:** PROPOSED

### BR-023 — Concept mastery (rule-based)

- **Source:** mind map (Track mastery level), README §4.1
    (`CONCEPT_MASTERY`), assumption AS-07
- **Related US:** US04, US05, US06, US11, US18
- **Related API:** `GET /learning/mastery`
- **Rule:** `mastery_score` is in the range 0–1 and is updated after
    each submission and each submitted assessment or quiz attempt
    tagged with that concept. V1 uses a weighted average of the 10 most
    recent results, with more recent results weighted higher (`ACCEPTED`
    submission = 1, other verdicts = 0 except `SYSTEM_ERROR`, which is
    ignored; correct answer = 1, wrong or "I don't know" = 0). A concept
    with no evidence shows the learner's survey self-assessment, labeled
    "not tested, from survey".
- **Status:** PROPOSED

### BR-024 — Knowledge gaps

- **Source:** mind map (Detect knowledge gaps), AC US10
- **Related US:** US10, US09, US34
- **Related API:** `GET /learning/knowledge-gaps`
- **Rule:** A concept is a knowledge gap when `mastery_score < 0.6` and
    it has at least 3 pieces of evidence (`evidence_count ≥ 3`). Gaps are
    sorted by lowest mastery first and trigger AI-generated practice
    (BR-035).
- **Status:** PROPOSED

### BR-025 — Learning path from the competency profile

- **Source:** mind map (Recommend a learning path based on the placement
    test), AC US08; UI decisions: one existing course per path, no
    generated courses, rule-based with LLM explanations
- **Related US:** US04, US08, US13, US33
- **Related API:** `POST /competency-profiles/{profileId}/confirm`,
    `GET /learning-paths/{pathId}`
- **Rule:**
    - A learning path = **one** existing `PUBLISHED` course of the
        chosen language + an order of its lessons. A path never spans
        several courses, and the system never generates a new course.
    - The course is chosen by rules from the language and the overall
        level in the competency profile.
    - Each topic gets a handling: **Learn from scratch**, **Review** or
        **Skip**. Suggested handling from the assessment score: 100% →
        Skip; 50% or more → Review; below 50% → Learn from scratch. A
        topic with no questions uses the survey: "Not learned" → Learn
        from scratch. The learner can change any handling before
        confirming (BR-032).
    - Lessons of Skip topics are marked `SKIPPED`; lessons of Review
        topics are labeled "Quick review"; weak topics are moved
        earlier where prerequisites allow; the rest follow course
        order.
    - Item statuses: `NOT_STARTED`, `IN_PROGRESS`, `COMPLETED`,
        `SKIPPED`. Items added by BR-035 are labeled "AI added".
    - The LLM writes the profile comment (comparing results with the
        self-assessment) and the reason shown for each item. It does not
        decide the course or the order.
    - There is no default path without an assessment (BR-032).
- **Status:** CONFIRMED (handling thresholds are taken from the UI
    prototype and are PROPOSED; the level → course mapping is an open
    question)

### BR-026 — Next lesson recommendation

- **Source:** mind map (Recommend the next lesson based on learning and
    assessment history), AC US09
- **Related US:** US09
- **Related API:** `GET /learning/recommendations`
- **Rule:** The next lesson of a path is its first item, in path order,
    that is not completed and not skipped (AI-added items included). The
    Dashboard's "Continue learning" uses the path with the most recent
    activity and resumes a video at the last position watched.
- **Status:** PROPOSED

### BR-033 — One learning path per language

- **Source:** UI decision; landing page FAQ
- **Related US:** US25, US32
- **Related API:** `GET /learning-paths`, `POST /onboarding/surveys`
- **Rule:** Each language has its own learning path, and a learner can
    follow several paths at the same time (for example C and SQL).
    "Create new path" runs onboarding again (survey → assessment →
    profile). A language that already has an `IN_PROGRESS` path cannot
    be chosen; once that path is `COMPLETED`, a new path for the same
    language can be created. My learning paths can be filtered by All,
    In progress and Completed.
- **Status:** CONFIRMED

### BR-034 — AI-generated assessment questions

- **Source:** UI decision (assessment generated by AI from the survey)
- **Related US:** US04
- **Related API:** `POST /competency-assessments`
- **Rule:** The LLM generates the competency assessment questions for
    each learner, tagged with the topics from the survey and following
    BR-016. Each question is checked against a schema (4 options,
    exactly one correct answer, a concept tag) before it is shown; a
    question that fails is regenerated. Correct answers are stored on
    the server and never returned before submission.
- **Status:** PROPOSED

### BR-035 — Adaptive, AI-generated practice

- **Source:** UI decisions: content adapts to learning history; "AI
    added" practice is generated by the LLM
- **Related US:** US34, US05, US10
- **Related API:** `GET /learning-paths/{pathId}`
- **Rule:**
    - After a quiz, and when a knowledge gap is found (BR-024), the
        system may generate a practice lesson for the weak concept and
        insert it into the learning path before the next chapter. The
        learner sees "Learning path updated" and the item is labeled
        "AI added".
    - The LLM generates the statement, requirements, examples, starter
        code, sample and hidden test cases and a reference solution.
        The item is inserted only after the reference solution passes
        every test case in the sandbox; otherwise it is regenerated or
        dropped.
    - Generated practice is graded like any exercise (BR-013); the LLM
        never decides the verdict.
    - Difficulty is tuned from the learner's recent results.
- **Status:** CONFIRMED (validation step and placement are PROPOSED;
    the maximum number of inserted items is an open question)

## Group G — Dashboard

### BR-027 — Exercise statistics

- **Status:** REMOVED. The Dashboard no longer shows exercise
    statistics; see BR-037.

### BR-028 — Learning streak

- **Source:** mind map (Achievements & Streak), AC US27
- **Related US:** US27
- **Related API:** `GET /dashboard/overview`
- **Rule:**
    - A day counts toward the streak when the learner completes at
        least one qualifying activity: completing a lesson, an
        `ACCEPTED` submission, or a submitted quiz or competency
        assessment attempt. Days follow the time zone in AS-09.
    - The current streak is the number of consecutive qualifying days
        ending today or yesterday; it resets to 0 after a full day with
        no qualifying activity. The Dashboard shows it with the last 7
        days.
    - V1 has no achievements or badges.
- **Status:** CONFIRMED (qualifying activities are PROPOSED)

### BR-037 — Dashboard content

- **Source:** UI prototype (Dashboard screen)
- **Related US:** US24, US25
- **Related API:** `GET /dashboard/overview`, `GET /dashboard/todo`
- **Rule:** The Dashboard shows:
    - a "Continue learning" card (BR-026);
    - three metrics: the streak (BR-028), lessons completed (total and
        this week) and the average quiz score with the number of
        quizzes taken;
    - the paths being followed, each with progress and its next lesson;
    - a to-do list: AI-added practice not yet done and quizzes not yet
        taken in active paths;
    - the competency profile, one tab per language, with a bar per
        skill.
- **Status:** CONFIRMED (which quiz attempt the average uses is an open
    question)

## Group H — Admin and Sensitive Data

### BR-029 — Data that must not be logged

- **Source:** README §4.2 (API conventions)
- **Related US:** all
- **Related API:** all
- **Rule:** Never write passwords, password hashes, tokens, LLM provider
    API keys or unnecessary personal data to logs. Learner source code
    is stored only in business tables, not in application logs.
- **Status:** CONFIRMED

### BR-030 — Admin statistics and monitoring

- **Source:** mind map (System statistics & monitoring), AC US31
- **Related US:** US31
- **Related API:** `GET /admin/stats`, `GET /admin/monitoring`
- **Rule:** Statistics show aggregated data only: active users,
    enrollments, code runs, submissions, AI requests, tokens and cost
    per day, error rates for the sandbox and LLM provider, and service
    health. They do not show individual learners' source code or AI
    conversations. An alert is shown when the daily AI cost exceeds the
    configured budget (NFR-AI-004).
- **Status:** CONFIRMED (aggregation-only rule is PROPOSED)

## Open Questions

- Should submissions be limited over time to prevent sandbox abuse, or
    is rate limiting (NFR-EXEC-006) enough?
- Can learners delete their AI conversation history? How does that
    affect cost statistics?
- Should streak days follow each learner's own time zone instead of a
    single one (AS-09)?
- Quiz pass threshold (BR-016): what score counts as "passed"?
- Average quiz score (BR-037): best attempt, latest attempt or every
    attempt?
- Do in-video checkpoint answers (BR-036) count toward mastery?
- Which course is chosen for each overall level and language (BR-025)?
- What happens when a learner picks "Other" language in the survey
    (BR-031), since no course exists for it?
- Do system-initiated LLM calls (assessment generation, practice
    generation, profile comment) count toward the learner's daily AI
    quota (BR-021)?
- How many AI-added items can be inserted per chapter or per week
    (BR-035)?
- How are SQL exercises graded (result-set comparison rules, BR-013)?
