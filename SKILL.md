---
name: qa-automation
description: Build and maintain API test automation projects using CSV/JSON-driven data, Playwright, Zod validation, and database assertions. Use when users want to create a new test project, add a new API endpoint test, debug test failures, or follow the test automation workflow.
---

# qa-automation

API test automation methodology. Architecture, workflow, conventions, debugging.

See [REFERENCE.md](./REFERENCE.md) for code templates and examples.
See [DIAGRAMS.md](./DIAGRAMS.md) for visual workflow and architecture diagrams.

## When this skill activates

- "create new test project", "set up test automation"
- "add test for this endpoint", "new API test"
- "test is failing", "debug this test", "why is this test broken"
- "how should I structure this test", "test workflow"
- "add new integration", "add new endpoint variant test"
- "analyze this CSV/JSON", "create requirement from this data"
- "design the test", "create tasks for this test"
- User provides CSV, JSON, curl command, or API spec for testing

---

## Quick Start: Add 1 Test Case to Existing Project

1. Open the CSV/JSON file for that endpoint
2. Add a new row with unique `test_id`
3. Fill request fields + expected fields
4. Done — the `for` loop in the spec picks it up automatically

**When code changes ARE needed:**

| Situation                    | Update                                              |
| ---------------------------- | --------------------------------------------------- |
| New request field            | CSV column → Type → Builder                         |
| New response field to assert | Zod Schema → Spec assertion                         |
| New endpoint entirely        | Full pipeline: CSV → Type → Builder → Schema → Spec |

---

## Workflow: Analyze → Requirement → Design → Tasks

Follow this process **before writing any code** when receiving new test data:

### Phase 1: Analyze

| Input       | Action                                                                       |
| ----------- | ---------------------------------------------------------------------------- |
| CSV         | Analyze columns, identify request vs expected fields, detect sentinel values |
| JSON        | Analyze keys/types, identify request vs expected, detect nested structures   |
| curl        | Extract method, endpoint, headers, body; identify dynamic vs static fields   |
| API spec    | Identify endpoints, request/response schemas, error codes                    |
| Description | Ask clarifying questions about fields, responses, error cases                |

Output: endpoint, request fields, response fields, success code, error cases, follow-up steps.

### Phase 2: Requirements

Write formal requirements with acceptance criteria (SHALL language). Follow pipeline order: Test Data → Type → Builder → Schema → Spec → DB.

### Phase 3: Design

1. Architecture flow — data movement through pipeline
2. Decision logic — conditional paths (e.g. validate-only vs validate-and-follow-up)
3. Files to create/modify — table of paths and purposes
4. Override/injection pattern — how to inject invalid values for negative tests

### Phase 4: Tasks

Implementable tasks with checkboxes, ordered by pipeline. Each references its requirement.

### Workflow triggers

| Trigger                     | Action                                      |
| --------------------------- | ------------------------------------------- |
| User provides CSV/JSON/curl | Analyze → Requirements → Design → Tasks     |
| User says "add test for X"  | Ask for details → full workflow             |
| User says "just do it"      | Skip to Tasks (still follow pipeline order) |

### Workflow rules

- Never skip straight to code — confirm plan first
- Workflow phases (Analyze → Requirements → Design → Tasks) apply when creating NEW code files or modifying existing code. Adding data rows to existing CSV/JSON does NOT require workflow phases — use Quick Start instead.
- Requirements must have acceptance criteria
- Design must show decision logic
- Tasks ordered by pipeline
- Wait for user approval between phases (unless "just do it all")

---

## Architecture

```
CSV/JSON → Type → Builder → sendRequest() → Zod Schema → Response Assert → DB Assert
```

| Step            | Purpose                       | Location                 |
| --------------- | ----------------------------- | ------------------------ |
| Test Data       | All inputs + expected outputs | `csv/` or `data/`        |
| Type            | TypeScript type for row shape | `src/types/{module}/`    |
| Builder         | Constructs payload from row   | `src/builders/{module}/` |
| sendRequest     | Single shared HTTP function   | `src/commons/common.ts`  |
| Zod Schema      | Validates response structure  | `src/schemas/{module}/`  |
| Response Assert | Checks code/message           | Inside spec              |
| DB Assert       | Verifies state change         | `src/db/queries/`        |

> **Response assertion is API-shaped.** The examples assert a business `response_code` + `response_message` envelope, but that's one convention. Assert on whatever your API defines — HTTP status alone, a typed body, or a status/error field. Zod still validates the response structure regardless.

---

## Project Structure

```
project-root/
├── csv/                    # One CSV per endpoint/suite
├── data/                   # One JSON per endpoint/suite
├── src/
│   ├── types/{module}/     # Row type definitions
│   ├── builders/{module}/  # Payload constructors
│   ├── schemas/{module}/   # Zod response schemas
│   ├── commons/            # sendRequest + auth
│   ├── constants/          # Endpoint paths
│   ├── db/queries/         # DB connection + queries
│   ├── config/             # Environment resolution
│   ├── utils/              # CSV/JSON parser, crypto, helpers
│   ├── fixtures/           # Playwright fixtures
│   ├── locators/           # UI selectors (POM)
│   ├── pages/              # Page objects (POM)
│   └── tests/api/{module}/ # Spec files
├── cicd/                   # Pipeline configs
├── .env / .env.example
├── playwright.config.ts
└── tsconfig.json
```

**Path aliases** (never use relative imports):
`@utils/*`, `@types/*`, `@builders/*`, `@commons/*`, `@fixtures/*`, `@schemas/*`, `@db/*`, `@config/*`, `@constants/*`, `@locators/*`, `@pages/*`

---

## Step Rules

### 1. Test Data

| Format | Use when                                         |
| ------ | ------------------------------------------------ |
| CSV    | Many cases, flat data, editable in Excel         |
| JSON   | Nested payloads, mixed types, complex structures |

**CSV column order:** `test_id, test_name, [request_fields...], [expected_fields...], [credential_env_vars...], release_tag`

**Rules (both formats):**

- `test_id` unique per row
- Expected fields prefixed with `expected_`
- Credentials: store .env var NAME, not value
- Minimum cases: success, missing field, invalid value

**Sentinel values (CSV only):**

| Value         | Resolves to   | API receives    |
| ------------- | ------------- | --------------- |
| `__EMPTY__`   | `""`          | empty string    |
| `__NULL__`    | `null`        | null            |
| `__DYNAMIC__` | runtime value | generated value |
| _(blank)_     | `undefined`   | field omitted   |

### 2. Type

- Use `type` not `interface`
- CSV: all `string`. JSON: proper types
- Field names match CSV headers / JSON keys exactly

### 3. Builder

- Maps row fields → API payload
- Generates dynamic values (IDs, timestamps, checksums)
- Resolves sentinel values
- Never hardcodes secrets — receives as parameters

### 4. Zod Schema

- One schema per response shape
- `.nullable()` for null fields, never `.optional()` unless truly absent
- Never `z.any()`

### 5. Spec File

- Import `test`/`expect` from fixture — never from `@playwright/test`
- One endpoint per spec file
- `for` loop at top level (no `test.describe()` wrapping)
- Schema validation on success responses only

### 6. npm Script

`"test:{endpoint}": "npx playwright test --project={env} --grep @{endpoint-tag}"`

---

## Multi-Step Flow

When an endpoint requires a follow-up (a dependent call + DB check):

1. Call main API → assert response → capture the resource ID/reference from the response
2. IF success: call the dependent endpoint → assert → query DB → assert final state
3. IF error: query DB → assert state unchanged

Use `test.describe.configure({ mode: "serial" })` for dependent steps.

---

## Infrastructure Rules

**Authentication:**

- Token fetched once in `test.beforeAll()`, cached per client
- Auto-refresh at 60s before expiry
- Credentials from `.env`, never hardcoded

**Fixture:**

- `environment` from `project.name` in playwright config
- `apiContext` resolves `baseURL` from `.env` via prefix pattern
- Always `dispose()` after use

**sendRequest:**

- One function for ALL calls (JSON, form, XML, custom headers)
- Base URL from fixture by default; override via `baseUrl` option

**Environment:**

- Pattern: `{ENV_PREFIX}_{KEY}` (e.g., `STAGING_BASE_URL`)
- CSV references env var names so same data works across environments

**Endpoint constants:**

- All paths in `src/constants/endpoints.ts`, UPPER_SNAKE_CASE
- Never hardcode strings in specs

**Database:**

- Parameterized queries only
- Pool shared per worker, closed in `afterAll`
- Return `null` when not found (let test assert)

---

## Debugging

| Symptom                    | Cause                           | Fix                                 |
| -------------------------- | ------------------------------- | ----------------------------------- |
| Business error code (e.g. "01")        | Wrong signature/credentials     | Print env var name + resolved value |
| Auth error code (e.g. "02")            | Expired token                   | Check .env values for target env    |
| Zod error: undefined field             | Response changed or CSV shifted | Compare actual response vs schema   |
| Expected success code, got error code  | Wrong/missing payload field     | Log full payload, compare with curl |
| DB assertion null          | Prior step didn't succeed       | Assert after each step              |
| Timeout                    | Wrong base URL                  | Print full URL being called         |
| Cannot read undefined      | CSV column misalignment         | Count commas in row vs header       |

**Debug order:** CSV row → env vars → built payload → full URL → compare with curl → raw response → DB query

**CSV misalignment** (most common silent bug): unquoted comma in value, missing field, extra comma at end.

---

## Conventions

**Naming:**

- Files: kebab-case (`create-resource.ts`)
- Test: `[{test_id}][{MODULE}][{ROLE}][{endpoint}] - {name} - response {code} : {message}`
- Tags: `@{endpoint-name}`, `@{release-tag}`

**Code rules:**

- All data from CSV/JSON — never hardcode in specs
- Secrets from `.env` only
- Endpoints from `@constants/endpoints`
- `type` not `interface`
- Path aliases always
- One endpoint per spec
- Single shared `sendRequest`
- Zod on every success response

---

## Anti-Patterns

| Don't                                | Do                                                   | Why                                        |
| ------------------------------------ | ---------------------------------------------------- | ------------------------------------------ |
| `test.describe()` wrapping data loop | `for` loop at top level                              | No value for data-driven                   |
| Share mutable state between tests    | Each test builds own payload                         | Parallel corruption                        |
| `waitForTimeout(3000)`               | `expect(locator).toBeVisible()`                      | Flaky vs reliable                          |
| Catch errors and return null         | Let test fail with real error                        | Hides bugs                                 |
| `z.any()`                            | Real types                                           | Defeats validation                         |
| Fetch token per test                 | `beforeAll` + cache                                  | N requests waste                           |
| `expect(x).toBeTruthy()`             | `expect(x).toBe(expected)`                           | Hides wrong values                         |
| `expect(x).toBe(y)` without message  | `expect(x, "[test_id] field: expected/got").toBe(y)` | Silent failures — no context on what broke |
| `test.setTimeout(120000)`            | Fix root cause                                       | Masks problems                             |

---

## Test Isolation

| Scenario                       | Mode                               |
| ------------------------------ | ---------------------------------- |
| Independent rows               | Parallel (default, workers: 4)     |
| Steps depend on prior output   | Serial (`test.describe.configure`) |
| Shared token, independent data | Parallel + `beforeAll`             |

---

## UI Testing (Page Object Model)

```
Locators (selectors only) → Pages (actions + assertions) → Spec (orchestration)
```

| Layer    | Location                 | Responsibility        |
| -------- | ------------------------ | --------------------- |
| Locators | `src/locators/{page}.ts` | Selector strings only |
| Pages    | `src/pages/{page}.ts`    | Actions + assertions  |
| Spec     | `src/tests/`             | Orchestration         |

**Locator rules:** one `const` object per page, descriptive names, prefer `data-testid` > `id` > `class` > XPath

**Page rules:** constructor receives `Page`, creates Locators from locator file, methods are actions or assertions, `readonly` properties

---

## UI Login: Federated Auth (SSO / OAuth) & OTP

Generic pattern for portals that sign in via a federated provider (AWS SSO, Google, Okta, etc.), optionally followed by OTP. **This section describes the pattern only — the implementing project supplies its own selectors, login path, provider name(s), and page object class.** See REFERENCE.md for a placeholder template.

**Multi-provider entry point:**

- One login page object handles all configured providers — combine provider buttons into a single locator (e.g. comma-separated selector list) rather than branching per-provider logic
- Don't assert *which* provider rendered — click whichever resolves, let the flow proceed

**Sequential credential flow:**

1. Click provider sign-in button
2. Fill identifier (username/email) → submit
3. Fill password → submit
4. If OTP is required: generate + submit OTP, with retry
5. Wait for a terminal state locator (signed-in marker OR error marker) to become visible

**OTP (TOTP) rules — when the flow requires it:**

- Use a standard TOTP library (e.g. `otplib`) with `window: 1` to tolerate ±1 step clock drift
- Before first submit: if less than ~5s remain in the current 30s window, wait for the next window — avoids submitting a code that expires mid-request
- Before each retry: always wait a full window, not just on the first attempt
- Retry loop: generate → fill → submit → check OTP-error locator with a short `waitFor` timeout → clear and retry on error, return on success
- Never use a fixed `waitForTimeout` for OTP timing — always compute remaining seconds from `Date.now()`
- OTP secret (TOTP seed) comes from `.env`, same secret-handling rule as API credentials — never hardcoded

**Settle-not-assert pattern:**

- The login action (`signIn()`/equivalent) returns once the attempt *resolves* — signed in or rejected — and asserts neither outcome
- The calling spec asserts the expected outcome (success or specific error)
- This keeps the login page object reusable for both positive and negative login tests

**Don't gate on URL:**

- Federated flows redirect through another host mid-flow, so a "left the login URL" check can be true before credentials are even submitted
- Gate on page elements instead — e.g. `signedInMarker.or(errorMessage).first()`

**Login path constant:**

- The `@constants/endpoints` rule applies to API paths only. For a UI login route, declare a single `const LOGIN_PATH` at the top of the login page object file instead — never inline the path string elsewhere (spec files, other pages)

**Anti-patterns specific to login:**

| Don't                                     | Do                                          | Why                                                       |
| ------------------------------------------ | -------------------------------------------- | ----------------------------------------------------------- |
| Assert success/failure inside the login action | Return after settling, assert in the spec  | Reusable for both positive and negative login tests          |
| Fixed `waitForTimeout` for OTP freshness  | Compute remaining seconds, wait dynamically  | TOTP window is exactly 30s — drift causes flaky retries       |
| Hardcode OTP secret or password in test/page | Read from `.env`                          | Same secret-handling rule as API tests                        |
| Check URL to confirm sign-in               | Check for a signed-in marker / error marker | Federated flows redirect to another host mid-flow             |
| Branch per-provider logic in the spec      | One locator/action, provider-agnostic        | Avoids duplicated flows per provider                          |

---

## CI/CD

**Pipeline:** install deps → install browser → run tests → publish HTML report → archive artifacts

**Parameterized:** environment + tag filter + browser at runtime

**Secrets:** Jenkins credentials / Vault / K8s secrets → env vars matching `{ENV}_SECRET_KEY` pattern

**Always archive (even on failure):** `failures.md`, `playwright-report/**`, `test-results/**` — so a failed run is investigable by downloading artifacts.

---

## Failure Reporting & Artifacts

When a run fails, produce artifacts that make it investigable without re-running. Think of three layers — a ladder from fastest triage to deepest forensics:

| Artifact           | Produced by                          | Use for                                  | Investigation depth |
| ------------------ | ------------------------------------ | ---------------------------------------- | ------------------- |
| `failures.md`      | Custom Markdown reporter             | Quick scan — which tests, which mismatch | Seconds             |
| HTML report        | `html` reporter (`playwright-report/`) | Browse results, filter, see attachments  | Minutes             |
| Trace (+ screenshot) | `trace: "on-first-retry"` (`test-results/`) | Forensic: every request/response, DOM snapshots | Deep                |

**Rules:**

- Enable `retries: 1` + `trace: "on-first-retry"` so traces are captured only when a test flaps/fails — cheap on green runs.
- Add the Markdown reporter to the `reporter` array; it writes `failures.md` listing only failed tests with their error message and attachments.
- The descriptive assertion message (`[test_id] field: expected/got`) IS the failure summary — the reporter just surfaces it. Keep messages descriptive (Self-Review #12) so `failures.md` reads well.
- On failure, attach request payload, response body, and DB result via `testInfo.attach()` so the trace/report carry exact context.
- **Never attach secrets.** Same rule as logging — redact credentials/tokens before attaching. Attachments are downloadable artifacts.
- In CI, archive `failures.md`, `playwright-report/**`, `test-results/**` in a `post { always { ... } }` block so artifacts survive a failed build.

See [REFERENCE.md](./REFERENCE.md) for the reporter, config wiring, `testInfo.attach` usage, Jenkins `archiveArtifacts`, and a sample rendered `failures.md`.

---

## Self-Review (Mandatory)

After generating ANY code, automatically review it against the checklist below before presenting to the user. If any check fails — fix it first, then present.

### Auto-Check List

| #   | Check              | Rule                                                                                                                                                                             |
| --- | ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Path aliases       | All imports use `@` aliases — no relative imports (`../`)                                                                                                                        |
| 2   | Type keyword       | `type` used, never `interface`                                                                                                                                                   |
| 3   | No hardcoded data  | All test data from CSV/JSON/env — nothing inline in specs                                                                                                                        |
| 4   | Secrets from env   | Credentials reference `.env` var names, never values                                                                                                                             |
| 5   | Endpoint constants | All paths from `@constants/endpoints` — no string literals                                                                                                                       |
| 6   | Zod validation     | Schema `.parse()` on every success response                                                                                                                                      |
| 7   | Loop structure     | `for` loop at top level — no `test.describe()` wrapping data loop                                                                                                                |
| 8   | Test naming        | Follows `[{test_id}][{MODULE}][{ROLE}][{endpoint}] - {name} - response {code} : {message}`                                                                                          |
| 9   | Sentinel handling  | Builder resolves `__EMPTY__`, `__NULL__`, `__DYNAMIC__`, blank correctly                                                                                                         |
| 10  | Anti-patterns      | None of the Anti-Patterns table violations present                                                                                                                               |
| 11  | Fixture imports    | In specs, builders, pages: `test`/`expect` from `@fixtures/*` — never from `@playwright/test`. Exception: the fixture file itself imports and re-exports from `@playwright/test` |
| 12  | Assertion messages | Every `expect()` includes descriptive error message with `[test_id]`, field name, expected, and actual                                                                           |

**TypeScript checks:**

| #   | Check                  | Rule                                                                      |
| --- | ---------------------- | ------------------------------------------------------------------------- |
| 13  | No `any` type          | Use proper types — `unknown` if truly unknown, then narrow                |
| 14  | Strict null handling   | Check for `null`/`undefined` before accessing — no `!` non-null assertion |
| 15  | Readonly for constants | Shared data objects use `as const` or `readonly`                          |

**Security checks:**

| #   | Check                    | Rule                                                          |
| --- | ------------------------ | ------------------------------------------------------------- |
| 16  | No secrets in code       | No API keys, passwords, tokens hardcoded anywhere             |
| 17  | No secrets in logs       | `console.log` never prints secret values — only env var names |
| 18  | No secrets in test names | Test title doesn't contain account IDs, credentials, or keys  |
| 19  | `.env` in `.gitignore`   | Never committed to repo                                       |

**UI-specific checks (apply when generating Page Object Model code):**

| #   | Check                     | Rule                                                                 |
| --- | ------------------------- | -------------------------------------------------------------------- |
| 20  | Selectors in locator file | No selectors in page or spec files — all in `src/locators/`          |
| 21  | Page uses `readonly`      | All locator properties are `readonly`                                |
| 22  | No actions in spec        | Specs orchestrate only — actions live in page objects                |
| 23  | Locator strategy          | Prefer `data-testid` > `id` > `class` > XPath — no fragile selectors |

### Output Format

After code generation, append:

```
✅ Self-review passed (all applicable checks)
```

Or when deviation exists:

```
✅ Self-review: all applicable checks passed except 1
⚠️ Deviation: [what] — [why it's intentional, referencing which rule allows it]
```

### Rules

- Never present code that fails any check — fix first
- Deviations are allowed ONLY when another rule in this skill explicitly permits it (e.g., `test.describe()` is allowed for serial multi-step flows)
- If a deviation exists, explain which rule justifies it

---

## Dependencies

**Core:** `@playwright/test`, `zod`, `csv-parse`, `dotenv`, `typescript`

**DB assertions (optional — pick the driver for your engine):** `mssql` + `@types/mssql` (SQL Server), `pg` (PostgreSQL), `mysql2` (MySQL), `mongodb` (MongoDB), etc.

**UI OTP (optional):** `otplib`
