# Assessment Platform Architecture — A Case Study

A technical write-up of the architecture behind a multi-portal exam/assessment SaaS platform I've worked on professionally as a full-stack engineer since January 2026. This is **not** the platform's source code — it's my own description of the system design, the trade-offs behind it, and what I'd change, written to be read by other engineers rather than end users. No proprietary code, credentials, or business logic from the original codebase is reproduced here.

**Live product:** [rankotest.com](https://rankotest.com) — built by the team at TalentXminds; this write-up covers architecture I've worked on there, not a solo project.

## Why this exists

Most of my day-to-day engineering happens inside a private, employer-owned repository, so it can't be shared directly. This repo exists to make that work legible: the actual complexity — six user roles, RBAC across five separate portals, an untrusted code-execution pipeline, Redis-backed rate limiting and caching — without exposing anything I don't have the right to publish.

## System overview

```
                     ┌───────────────────────────┐
                     │      React SPA clients     │
                     │ (Admin/Vendor/Institute/   │
                     │  Teacher/Student portals)   │
                     └──────────────┬─────────────┘
                                    │ REST (JWT bearer)
                     ┌──────────────▼─────────────┐
                     │   Node.js / Express API    │
                     │  ┌───────────────────────┐  │
                     │  │ auth + RBAC middleware│  │
                     │  ├───────────────────────┤  │
                     │  │ rate-limit middleware │──┼──► Redis (or in-memory fallback)
                     │  ├───────────────────────┤  │
                     │  │  route controllers    │  │
                     │  └───────────┬───────────┘  │
                     └──────────────┼──────────────┘
                                    │
                  ┌─────────────────┼─────────────────┐
                  │                 │                   │
           ┌──────▼──────┐   ┌──────▼──────┐    ┌───────▼────────┐
           │  MongoDB    │   │    Redis    │    │  Piston (code   │
           │ (documents, │   │ (sessions,  │    │  execution      │
           │  RBAC data) │   │ rate limits,│    │  sandbox, via   │
           │             │   │  caching)   │    │  HTTP API)      │
           └─────────────┘   └─────────────┘    └────────────────┘
```

## Authentication & multi-tenancy

Auth is JWT-based, but the harder problem isn't verifying a token — it's scoping every query correctly once you know *who* the caller is. The platform has five portal types (Admin, Vendor, Institute, Teacher, Student) plus sub-roles within some of them (e.g. multiple teacher sub-types), and a Vendor's data must never leak across to a different Vendor's institutes, teachers, or students.

In practice this means auth middleware does more than decode a token — it also has to answer questions like "is this vendor account still active, and does the resource being requested actually belong to them?" before a request reaches a controller. Getting this wrong is the single highest-blast-radius class of bug on a platform like this: a boundary check missed in one endpoint can expose another tenant's exam data.

**What I'd improve:** tenant-scoping logic like this tends to accrete as ad-hoc checks scattered across middleware and controllers. A cleaner design centralizes "does caller X have access to resource Y" behind a single authorization layer (e.g. a policy/ability object per request) rather than re-deriving it per route.

## RBAC across five portals

Role checks gate both routes (can this role hit this endpoint at all) and data (which rows can this role see, for records it can access). The tricky part is less "define the roles" and more "keep the permission matrix consistent as new features land in five portals simultaneously" — a new exam feature usually needs answering, per role: can create, can edit, can view results, can grade.

## Assessment lifecycle

An assessment moves through a lifecycle roughly like: draft → scheduled → live → submitted → graded → published. Different roles can act at different stages (a teacher can edit a draft; once it's live, edits have to be constrained so they don't retroactively change what students already saw). Time-based transitions (a test opening/closing at a scheduled time) is one of the areas where "looks fine in the UI" and "is actually consistent under concurrent access and clock skew" diverge — validating that a schedule isn't in the past, and that state transitions are atomic, is exactly the kind of validation gap that's easy to miss until it ships.

## Coding assessments — untrusted code execution

For coding questions, submitted code can't run in-process — it has to execute in an isolated sandbox with resource limits, then report back stdout/stderr/exit code/time/memory. This platform uses **Piston** (an open-source code execution engine) as that sandbox, called over HTTP from the API layer rather than shelling out to a local interpreter.

Practical failure modes worth designing for: the sandbox service being slow or down (needs a timeout + clear failure state, not a hung request), a submission that intentionally tries to exhaust CPU/memory (needs the sandbox's own limits, not just app-level ones), and language/runtime version drift between what a student expects and what the sandbox actually has installed.

## Grading

Objective questions (MCQ, coding test cases) can be graded synchronously or in a background step. Subjective grading needs a human in the loop, which means the data model has to represent "graded" as a spectrum, not a boolean — some parts of a submission may be auto-scored while others wait on a teacher.

## Redis: rate limiting and caching

Redis backs two distinct concerns that are easy to conflate but shouldn't share the same keys or eviction policy: **rate limiting** (short-lived counters keyed by IP/user/route) and **caching** (session or computed data with its own TTL). The rate-limit layer is written with a fallback: if Redis is unreachable, it degrades to an in-memory limiter (scoped to a single process) rather than failing open or crashing the request path — an important choice, since losing Redis shouldn't mean losing all abuse protection, even if the in-memory fallback is weaker in a multi-instance deployment.

**What I'd improve:** an in-memory fallback is inherently per-process, so behind a load balancer with N instances the *effective* rate limit is N× the configured value during a Redis outage. That's a reasonable trade-off for availability, but it should be an explicit, documented one — and ideally monitored, so a prolonged Redis outage is visible rather than silently changing rate-limit behavior.

## Rate limiting

Applied per-route, since a login endpoint and a read-heavy dashboard endpoint have very different abuse profiles. Keying strategy (by IP vs. by authenticated user vs. both) matters: IP-only limiting hurts shared-IP users (a whole institute's lab behind one NAT); user-only limiting doesn't protect unauthenticated endpoints at all.

## Testing

End-to-end flows (login → take exam → submit → get graded) are the ones most worth covering with Playwright, since they cross the most layers (auth, RBAC, scheduling, and — for coding questions — the sandbox integration) and are exactly where a change in one portal can silently break another.

## Failure scenarios worth designing for

- **Sandbox (Piston) unavailable** — submissions should queue or clearly fail, not hang.
- **Redis unavailable** — rate limiting degrades to in-memory per-process; sessions/caching need their own fallback story.
- **Concurrent submission at exam close time** — the boundary between "still open" and "closed" needs to be atomic, not a read-then-write race.
- **Cross-tenant data leakage** — the most damaging failure mode; needs to be a first-class thing you test for, not an incidental side effect of correct-looking code.

## Scaling considerations

- Stateless API instances behind a load balancer, with session/rate-limit state pushed to Redis rather than kept in-process (with the caveat above about degraded behavior when Redis is down).
- The code-execution sandbox is naturally the bottleneck under load (it's CPU/memory bound per submission) — it needs its own scaling story independent of the API layer, and ideally a queue in front of it rather than direct synchronous calls once volume is high enough.

## What I'd change with more time

- Centralize tenant-scoping/authorization behind one policy layer instead of ad-hoc checks per route.
- Make the Redis-down degraded mode for rate limiting observable (metric/alert), not just a silent fallback.
- Move coding-assessment execution behind a queue so grading throughput doesn't couple directly to sandbox response time.

## About me

I'm a full-stack engineer (TypeScript, React, Node.js, MongoDB, Redis) who's worked on this system in production since January 2026, and I use AI tooling (Claude Code, MCP-based code graphs) heavily as part of how I understand and navigate a codebase this size and interconnected. Reach me via [LinkedIn](https://www.linkedin.com/in/deepak-jain-ab8aa924/).
