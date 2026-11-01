# V1 Scope

> Design status: DRAFT V1
>
> Implementation status: NO SOURCE CODE YET — requirements,
> architecture documents and a UI prototype for the learner flow exist

Product display name: **DevMentor**. Tagline: "DevMentor understands
you first, then starts teaching."

## Problem

People who are new to programming usually struggle with the following:

- they start at the wrong point: courses are too easy or too hard for
    what they already know;
- they get stuck and no one is there to guide them in time;
- they watch a lot but write little code;
- general-purpose AI chatbots tend to hand over a complete solution, so
    learners do not build their own reasoning;
- they do not know which concepts they are weak at, so they cannot
    target their study;
- they lose motivation without visible progress.

DevMentor puts the whole learning loop in one place: **survey and
competency assessment → competency profile → personalized learning path
(one course, reordered) → study the lesson → practice code → get
context-aware help from the AI Mentor → get evaluated automatically →
the path adapts (AI-added practice) → next lesson**.

DevMentor is **not** a full IDE, not a competitive programming platform
and does not replace instructors in formal courses.

## Target Users

| Group | Description | Main needs |
| --- | --- | --- |
| Visitor | Has no account | See the landing page, register |
| Learner | Students and self-learners in Vietnam, from people who have never coded to those who know the basics | Get assessed, follow a path that skips what they know, practice, get guiding help, see progress and stay motivated |
| Admin | Operations / content team | Manage courses, lessons, exercises, test cases, questions and users; monitor the system and AI cost |

## V1 Goals

V1 covers every leaf of the [product mind map](../README.md#11-product-mind-map)
(see the mind map note in README §1.1 for leaves pending an update):

| Mind map group | V1 capability | User Stories |
| --- | --- | --- |
| Authentication & Authorization | Register, log in, log out; role-based access control enforced in the backend; unauthorized access blocked with `401`/`403`/`404`. | US01–US03 |
| Assessment & Evaluation | AI-generated competency assessment from the onboarding survey; end-of-lesson quizzes scored automatically; coding challenges inside the learning path graded by test cases; code evaluation with score and runtime/memory metrics. | US04–US07 |
| Personalized Learning Path | Mandatory onboarding survey; competency profile the learner can adjust; one path per language = one existing course reordered by rules with LLM explanations; next lesson; knowledge gaps; mastery per concept; AI-generated practice inserted into the path. | US08–US11, US32–US34 |
| Courses & Lessons | View the chapters and lessons of a path; video and reading lessons with in-video checkpoint questions and notes; coding practice lessons. | US13–US15 |
| Coding Workspace | In-browser code editor for C, C++, Python, Java, JavaScript and SQL; run code in an isolated sandbox with stdout/stderr/run time; submit and view results with sample and hidden tests. | US16–US18 |
| Learn with AI Mentor | Explain concepts, step-by-step hints, error analysis (Debug), code review and Q&A, aware of the current lesson, video position and code, guiding rather than solving. | US19–US23 |
| Dashboard | Continue learning, streak, lessons completed, average quiz score, paths being followed, to-do list, competency profile per language. | US24, US25, US27 |
| Content Management & Admin | Manage courses and modules; lessons, exercises, test cases and assessment questions; users and roles; system statistics and monitoring. | US28–US31 |

Removed from V1 by UI decisions (IDs kept for traceability): US12
(explore and enroll in courses) and US26 (exercise statistics).

Cross-cutting V1 requirements:

- Authentication and authorization enforced in the backend for every
    API.
- Model, token counts and cost recorded for every LLM call.
- The whole UI is in Vietnamese.

## Out of Scope for V1

V1 does **not** include:

- A course catalog, course search, public course pages or self
    enrollment: the system picks the course for each path (BR-025).
- Paths that span several courses, or AI-generated courses.
- Profile editing, password change and password reset (not in the mind
    map; see [Backlog gaps](#backlog-gaps)).
- A separate instructor role, classes, manual grading.
- Payments, paid plans, certificates.
- Native mobile apps (V1 is a responsive web app).
- Forums, learner-to-learner chat, livestreams.
- Machine-learning Knowledge Tracing (V1 is rule-based; see BR-023).
- Plagiarism detection / code comparison between learners.
- Third-party login (Google, GitHub) and multi-factor authentication.
    The prototype shows Google and GitHub buttons as a future direction;
    they do not work in V1.
- Multi-file projects where the learner edits several files,
    installing arbitrary packages, or network access from learner code.
    An exercise may include read-only supporting files (BR-012).
- Leaderboards, rankings or timed contests for coding challenges.
- Achievements and badges, and exercise statistics on the Dashboard.
- AI-assigned grades: scores and verdicts come only from test results.

## V1 Assumptions (to confirm)

The assumptions below are default choices so design can continue. Each
must be confirmed or changed before the matching ADR is finalized.

| ID | Open question (README) | Proposed V1 assumption | Status |
| --- | --- | --- | --- |
| AS-01 | Audience and roles | Students and self-learners in Vietnam, from never coded to basic; only 3 roles: Visitor / Learner / Admin | PROPOSED |
| AS-02 | Programming languages | C, C++, Python, Java, JavaScript and SQL, each with its own courses and sandbox runtime | CONFIRMED |
| AS-03 | Content source | Courses, lessons, videos and quizzes are written by the project team and entered by Admins. Competency assessment questions and "AI added" practice are generated by the LLM per learner (BR-034, BR-035) | CONFIRMED |
| AS-04 | Code execution | A dedicated sandbox container with no network and CPU/RAM/time limits (see NFR-EXEC-*) | PROPOSED |
| AS-05 | AI provider | A single LLM provider behind a replaceable adapter; per-user quota (see NFR-AI-*) | PROPOSED |
| AS-06 | RAG | Only published lesson content inside the system, including video transcripts, is used as the knowledge base | PROPOSED |
| AS-07 | Personalization | Rule-based course choice, ordering, mastery and gaps (BR-023 to BR-026); the LLM writes explanations and generates practice but does not decide the order | CONFIRMED |
| AS-08 | Deployment | Reference scale of 1,000 registered users and 200 concurrent users (see NFR-CAP-*) | PROPOSED |
| AS-09 | Streak day boundary | A streak day follows the `Asia/Ho_Chi_Minh` time zone for all users in V1 (BR-028) | PROPOSED |
| AS-10 | Account self-service | Profile editing, password change and password reset are out of V1 because they are not in the mind map | PROPOSED |
| AS-11 | Lesson video | Videos are uploaded by Admins and stored in Object Storage (BR-036) | CONFIRMED |

## Backlog Gaps

The items below are not in the mind map but are commonly expected or
implied by other parts of the design. Each needs a decision: **add to
the mind map and backlog** or **keep out of V1**.

| Item | Why it matters | Proposal |
| --- | --- | --- |
| Profile editing and password change | Users cannot fix a display name or rotate a password. | Keep out of V1 (AS-10); add to the mind map under Authentication & Authorization if needed. |
| Forgot / reset password | Users who forget their password cannot recover the account without Admin help; needs an email service. | Keep out of V1; Admin can reset access manually in the meantime. |
| Managing assessment questions | Quizzes need questions, but the mind map's admin group only names "lessons & exercises". | Included in US29 as part of "Manage lessons & exercises"; confirm. |
| Mind map leaves for the new learner flow | The onboarding survey, competency profile and AI-added practice (US32–US34) have no leaf yet, and some leaves describe removed features. | Update the mind map (see README §1.1). |
| Empty, error and loading states | The prototype does not yet show them for the Dashboard, learning paths and practice. | Design before implementation. |

## Current State

| Component | Reality |
| --- | --- |
| Documentation | README overview, mind map, scope, use cases, business rules, NFR documents, [C4 architecture](architecture/c4/README.md) and the [OpenAPI contract](api/README.md) |
| UI | Interactive prototype of the learner flow, 12 screens (README §2.3); no admin screens yet |
| Frontend | None yet |
| Backend | None yet |
| Database | Only the ER diagram in the README |
| Code execution | Sandbox technology not chosen |
| AI | LLM provider not chosen |

## Target V1 Architecture

``` text
Web Application (incl. in-browser code editor and video player)
    → REST/HTTPS/JSON
Backend API (modular monolith)
    → PostgreSQL
    → Redis (cache, queues, rate limits)
    → Code Execution Service (isolated sandbox, one runtime per language)
    → AI Mentor Service → Vector Store, LLM Provider
    → Object Storage (lesson videos and learning material files)
Background Worker (mastery, learning path updates, AI-generated
    practice, recommendations, streaks, statistics)
```

See section 3 (C4) of the [README](../README.md#3-c4-architecture) for
details.

## Related Documents

- [Actors, Roles and Use Cases](03-use-cases.md)
- [Business Rules](requirements/business-rules.md)
- [Non-Functional Requirements](requirements/non-functional-requirements.md)
