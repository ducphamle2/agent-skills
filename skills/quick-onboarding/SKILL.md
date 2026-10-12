# AI Coding Interview — No-Edit Execution Skill

## Hard rules
- NO code changes
- NO fixes
- NO patching or config modification to “make it work”
- ONLY: run commands, observe, document, trace

Goal: understand how THIS system works, not assume how systems work.

---

# 1) Workflow extraction (from repo surface)

From README / Makefile / scripts / package.json / docs:

Extract only executable commands:

- system start command
- dependency install command
- test command
- single test command
- E2E / smoke test command
- API usage / curl / request examples
- restart command (if exists)

Write:

START:
INSTALL:
TEST:
SINGLE_TEST:
SMOKE:
RESTART:
API:

---

# 2) System shape mapping (repo-specific, not assumed)

Identify the actual components used in THIS codebase:

Look for boundaries such as:
- entrypoints (server/app/main/bootstrap)
- request handlers / controllers / routers
- core business logic modules
- orchestration / workflow / pipeline logic (if any)
- integration layers (external services, APIs, SDKs)
- data layer (DB/cache/files)
- background workers / queues (if any)

Then map relationships based on code:

COMPONENT A → COMPONENT B → COMPONENT C

Only use what exists in the repo. Do NOT assume structure.

---

# 3) Endpoint / interface tracing (ALL entrypoints)

For EACH exposed interface (API endpoint, CLI command, job trigger, etc.):

Trace execution:

- input boundary
- first handler
- major internal steps
- external calls (if any)
- output boundary

Goal:
- understand full behavior per interface
- identify shared vs unique logic paths

---

# 4) Testing map

Locate:

- unit tests
- integration tests
- end-to-end tests
- fixtures / mocks / fakes
- test utilities
- simulated external dependencies (if any)

Goal: understand testing strategy + how to run quickly

---

# 5) Failure classification (NO FIXES)

When a failure occurs:

DO NOT fix anything.

Only:
- reproduce
- capture logs
- classify
- document into `.md`

---

## Log format

FAILURE:
Command:
Observed behavior:
Error output:
Type: env | dependency | test | logic | async/integration
Hypothesis:
Confidence (0–100):

---

## Type guide

- ENV: missing config, startup failure, docker/runtime issues
- DEP: install/version/module mismatch
- TEST: assertion/snapshot mismatch
- LOGIC: system runs but output incorrect
- ASYNC: timeout, partial execution, race conditions

---

## Communication rule

When asked about issues:

“I observed X failure. Classified it as Y. Hypothesis is Z. No fixes were applied; only logs and traces were collected.”

---

## Core mindset

You are:
- system observer
- execution tracer
- boundary mapper
- failure classifier

NOT:
- debugger
- fixer
- optimizer