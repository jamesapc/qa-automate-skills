# qa-automate-skills

An agent skill for building and maintaining **API test automation** projects using a data-driven methodology. It codifies the architecture, workflow, conventions, and debugging practices for writing reliable, maintainable tests with **Playwright**, **Zod** validation, CSV/JSON-driven test data, and database assertions.

## What this skill does

The skill guides an AI agent (and the team) through a consistent process for test automation:

- **Create new test projects** with a standard structure and conventions.
- **Add API endpoint tests** driven by CSV/JSON data rather than hardcoded values.
- **Debug failing tests** using a systematic symptom → cause → fix approach.
- **Follow a repeatable workflow** — Analyze → Requirements → Design → Tasks — before writing any code.

## Core methodology

### Data-driven pipeline

```
CSV/JSON → Type → Builder → sendRequest() → Zod Schema → Response Assert → DB Assert
```

| Step            | Purpose                        |
| --------------- | ------------------------------ |
| Test Data       | All inputs + expected outputs  |
| Type            | TypeScript type for row shape  |
| Builder         | Constructs payload from a row  |
| sendRequest     | Single shared HTTP function    |
| Zod Schema      | Validates response structure   |
| Response Assert | Checks status / business code  |
| DB Assert       | Verifies state change          |

Add a new test case by adding a row to a CSV/JSON file — the spec's `for` loop picks it up automatically. Code changes are only needed for new request fields, new asserted response fields, or entirely new endpoints.

### Four-phase workflow

Before writing code for new test data, the skill enforces:

1. **Analyze** — inspect CSV/JSON/curl/API spec, identify request vs expected fields, detect sentinel values.
2. **Requirements** — formal acceptance criteria in SHALL language, ordered by the pipeline.
3. **Design** — architecture flow, decision logic, files to create/modify, injection pattern for negative tests.
4. **Tasks** — implementable, checkbox-tracked tasks ordered by the pipeline.

## Key conventions

- **Path aliases** everywhere (`@utils/*`, `@builders/*`, `@schemas/*`, …) — never relative imports.
- **`type`, not `interface`** for row and payload shapes.
- **No hardcoded data or secrets** — all data from CSV/JSON, all credentials from `.env` (by var name).
- **Endpoint paths** live in `@constants/endpoints`.
- **Zod validation** on every success response — no `z.any()`.
- **One endpoint per spec**, `for` loop at top level (no `test.describe()` wrapping data loops).
- **Single shared `sendRequest`** for all HTTP calls.
- **Sentinel values** (`__EMPTY__`, `__NULL__`, `__DYNAMIC__`, blank) for precise CSV-driven inputs.

## Additional coverage

- **Multi-step flows** — dependent calls + DB checks using serial mode.
- **Authentication** — token cached in `beforeAll`, auto-refresh before expiry.
- **UI testing** — Page Object Model (Locators → Pages → Spec).
- **Federated login (SSO/OAuth) + OTP (TOTP)** — provider-agnostic login pattern with dynamic OTP timing.
- **CI/CD** — parameterized pipelines, secret handling, artifact archiving.
- **Failure reporting** — `failures.md`, HTML reports, and traces as a triage-to-forensics ladder.
- **Mandatory self-review** — a checklist the agent runs against generated code before presenting it.

## Files

| File           | Contents                                             |
| -------------- | ---------------------------------------------------- |
| `SKILL.md`     | Methodology: architecture, workflow, conventions, debugging, self-review |
| `REFERENCE.md` | Code templates and examples                          |
| `DIAGRAMS.md`  | Visual workflow and architecture diagrams            |

## When the skill activates

- "create new test project", "set up test automation"
- "add test for this endpoint", "new API test"
- "test is failing", "debug this test"
- "add new integration", "add new endpoint variant test"
- When a user provides a CSV, JSON, curl command, or API spec to test.

## Tech stack

**Core:** `@playwright/test`, `zod`, `csv-parse`, `dotenv`, `typescript`
**DB assertions (optional):** `mssql`, `pg`, `mysql2`, `mongodb`, …
**UI OTP (optional):** `otplib`
