# REFERENCE — Code Templates & Patterns

Implementation templates for the qa-automation skill. See [SKILL.md](./SKILL.md) for workflow and conventions.

> **Examples are illustrative, not prescriptive.** Templates use a payment-style API (response codes `00`/`01`, checksum, merchant fields), a `payment` module with a `start-payment` endpoint, SQL Server (`mssql`), and env prefixes `STAGING`/`SANDBOX` purely as concrete examples. Replace `payment/start-payment` with your `{module}/{endpoint}`, and adapt the domain fields, response-code convention, DB driver, and env prefixes to your project. The reusable part is the structure: `Data → Type → Builder → sendRequest → Schema → Assert → DB`.
>
> Blocks marked ` ```typescript ` with a `File:` line are complete files that compile together under `tsc --strict`. Blocks marked ` ```ts ` are snippets shown in context.

---

## tsconfig.json — Path Aliases

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "moduleResolution": "node",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "baseUrl": ".",
    "types": ["node"],
    "paths": {
      "@utils/*": ["./src/utils/*"],
      "@payloads/*": ["./src/types/*"],
      "@builders/*": ["./src/builders/*"],
      "@commons/*": ["./src/commons/*"],
      "@fixtures/*": ["./src/fixtures/*"],
      "@schemas/*": ["./src/schemas/*"],
      "@db/*": ["./src/db/*"],
      "@config/*": ["./src/config/*"],
      "@constants/*": ["./src/constants/*"],
      "@locators/*": ["./src/locators/*"],
      "@pages/*": ["./src/pages/*"]
    }
  },
  "include": ["src/**/*.ts", "scripts/**/*.ts", "playwright.config.ts"]
}
```

> **Don't name the types alias `@types/*`.** TypeScript reserves the `@types` scope for declaration packages and rejects `import ... from "@types/..."` (error TS6137). Use `@payloads/*` (or another name) for `src/types/`.

---

## Test Data

### CSV — `csv/start-payment.csv`

```csv
test_id,test_name,amount,currency_code,expected_response_code,expected_response_message,merchant_id_env,secret_key_env,release_tag
TC-001,success case,400.00,THB,00,success,SANDBOX_MERCHANT_ID,SANDBOX_SECRET_KEY,@v1.0
TC-002,missing amount,__EMPTY__,THB,01,invalid request,SANDBOX_MERCHANT_ID,SANDBOX_SECRET_KEY,@v1.0
TC-003,invalid checksum,400.00,THB,01,invalid checksum,SANDBOX_MERCHANT_ID,WRONG_KEY_ENV,@v1.0
```

### JSON — `data/start-payment.json`

```json
[
  {
    "test_id": "TC-001",
    "test_name": "success case",
    "amount": 400.00,
    "currency_code": "THB",
    "expected_response_code": "00",
    "expected_response_message": "success",
    "merchant_id_env": "SANDBOX_MERCHANT_ID",
    "secret_key_env": "SANDBOX_SECRET_KEY",
    "release_tag": "@v1.0"
  }
]
```

---

## Type Definition

File: `src/types/payment/start-payment.ts`

```typescript
// CSV variant (all strings). JSON variant: use proper types (number | null, boolean, ...).
export type StartPaymentTestData = {
  test_id: string;
  test_name: string;
  amount: string;
  currency_code: string;
  expected_response_code: string;
  expected_response_message: string;
  merchant_id_env: string;
  secret_key_env: string;
  release_tag: string;
};

// Multi-step variant: same row plus expected values for the follow-up steps.
export type StartPaymentFlowTestData = StartPaymentTestData & {
  expected_notify_code: string;
  expected_db_status: string;
};
```

---

## Sentinel Resolver

File: `src/utils/sentinel.ts`

```typescript
export type ResolvedValue = string | null | undefined;

/**
 * Resolves CSV sentinels. `__DYNAMIC__` must have a runtime value supplied by the builder;
 * a missing one throws instead of silently omitting the field.
 */
export function resolveRow<T extends Record<string, string>>(
  row: T,
  dynamicValues: Readonly<Record<string, string>> = {},
): Record<keyof T, ResolvedValue> {
  const resolved = {} as Record<keyof T, ResolvedValue>;
  for (const key of Object.keys(row) as (keyof T & string)[]) {
    const value = row[key];
    if (value === "__EMPTY__") resolved[key] = "";
    else if (value === "__NULL__") resolved[key] = null;
    else if (value === "__DYNAMIC__") {
      const dynamic = dynamicValues[key];
      if (dynamic === undefined) {
        throw new Error(`__DYNAMIC__ used for "${key}" but the builder supplied no runtime value`);
      }
      resolved[key] = dynamic;
    } else if (value === "") resolved[key] = undefined;
    else resolved[key] = value;
  }
  return resolved;
}

export function removeUndefined<T extends Record<string, unknown>>(obj: T): Partial<T> {
  return Object.fromEntries(Object.entries(obj).filter(([, v]) => v !== undefined)) as Partial<T>;
}
```

---

## Crypto Helpers

File: `src/utils/crypto.ts`

```typescript
import { createHmac, randomInt } from "crypto";

export function generateUniqueReference(prefix = "TEST"): string {
  return `${prefix}_${Date.now()}${randomInt(100, 999)}`;
}

/** Example only: use the exact algorithm and field order from your API spec. */
export function computeChecksum(data: string, secretKey: string): string {
  return createHmac("sha256", secretKey).update(data).digest("hex");
}
```

---

## Required Env Vars

File: `src/utils/env.ts`

```typescript
/** Fails fast with the variable NAME (never the value) instead of sending an empty credential. */
export function requireEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing env var: ${name}`);
  return value;
}
```

---

## Builder

File: `src/builders/payment/start-payment.ts`

```typescript
import type { StartPaymentTestData } from "@payloads/payment/start-payment";
import { resolveRow, removeUndefined } from "@utils/sentinel";
import { computeChecksum, generateUniqueReference } from "@utils/crypto";

export function buildPayload(row: StartPaymentTestData, merchantId: string, secretKey: string) {
  const merchantReference = generateUniqueReference();
  const resolved = resolveRow(row);
  const checksum = computeChecksum(`${merchantId}${merchantReference}${resolved.amount ?? ""}`, secretKey);

  const payload = removeUndefined({
    amount: resolved.amount,
    currency_code: resolved.currency_code,
    merchant_id: merchantId,
    merchant_reference: merchantReference,
    checksum,
  });

  return { payload, merchantReference };
}

export function buildNotifyPayload(paymentCode: string, merchantReference: string, secretKey: string) {
  return {
    payment_code: paymentCode,
    merchant_reference: merchantReference,
    checksum: computeChecksum(`${paymentCode}${merchantReference}`, secretKey),
  };
}
```

---

## Zod Schema

File: `src/schemas/payment/start-payment.ts`

```typescript
import { z } from "zod";

/** Envelope present on every response, success or error. Used to read the code/message safely. */
export const responseEnvelopeSchema = z.object({
  response_code: z.string(),
  response_message: z.string(),
});

/** Full shape of a success response. */
export const startPaymentResponseSchema = responseEnvelopeSchema.extend({
  payment_code: z.string(),
  amount: z.number(),
  currency_code: z.string(),
  optional_field: z.string().nullable(),
});

export const notifyResponseSchema = responseEnvelopeSchema;

export type StartPaymentResponse = z.infer<typeof startPaymentResponseSchema>;
```

File: `src/utils/schema.ts`

```typescript
import type { z } from "zod";

/** Parses with a [test_id] message on failure, and returns typed data (no `any`, no `!`). */
export function parseOrFail<T>(schema: z.ZodType<T>, body: unknown, testId: string, label: string): T {
  const result = schema.safeParse(body);
  if (!result.success) {
    throw new Error(`[${testId}] ${label} schema mismatch: ${result.error.message}`);
  }
  return result.data;
}
```

---

## Endpoint Constants

File: `src/constants/endpoints.ts`

```typescript
export const ENDPOINTS = {
  START_PAYMENT: "start-payment",
  REFUND: "refund",
  GET_STATUS: "get-payment-status",
  NOTIFY: "notify-payment",
} as const;
```

---

## Masking Helper

File: `src/utils/mask.ts`

Used by `sendRequest` logging and any `testInfo.attach()`. The key list is a floor: add project-specific secret field names when the source data shows them.

```typescript
const SECRET_KEYS: readonly string[] = [
  "password", "passwd", "pwd", "secret", "key", "apikey", "token", "authorization", "cookie",
  "checksum", "signature", "hash", "otp", "cvv", "cvc", "pin",
];

const CARD_KEYS: readonly string[] = ["card_number", "cardnumber", "pan", "card_no"];

const MASKED = "***MASKED***";
const SECRET_WORD = "(?:password|passwd|secret|key|token|checksum|signature|hash|otp|cvv|cvc|pin)";

function words(key: string): string[] {
  // "SecretKey" -> ["secret", "key"]; "x-api-key" -> ["x", "api", "key"]; "shipping" stays one word.
  return key.replace(/([a-z0-9])([A-Z])/g, "$1_$2").toLowerCase().split(/[_\-\s]+/).filter(Boolean);
}

function isSecretKey(key: string): boolean {
  const k = key.toLowerCase();
  return SECRET_KEYS.includes(k) || words(key).some((w) => SECRET_KEYS.includes(w));
}

function isCardKey(key: string): boolean {
  return CARD_KEYS.includes(key.toLowerCase());
}

function luhnValid(digits: string): boolean {
  let sum = 0;
  for (let i = 0; i < digits.length; i++) {
    let d = Number(digits[digits.length - 1 - i]);
    if (i % 2 === 1) {
      d *= 2;
      if (d > 9) d -= 9;
    }
    sum += d;
  }
  return sum % 10 === 0;
}

function maskCard(value: string): string {
  return value.length > 4 ? `****${value.slice(-4)}` : "****";
}

/** Masks secrets inside raw text: XML (with attributes), form/query strings, JSON text, JWTs, card numbers. */
export function maskText(text: string): string {
  return text
    .replace(
      new RegExp(`<((?:\\w+:)?[\\w-]*${SECRET_WORD}[\\w-]*)(\\s[^>]*)?>[^<]*</\\1>`, "gi"),
      (_m, tag: string, attrs: string | undefined) => `<${tag}${attrs ?? ""}>${MASKED}</${tag}>`,
    )
    .replace(new RegExp(`((?:^|[&?])[\\w-]*${SECRET_WORD}[\\w-]*=)[^&#\\s]*`, "gi"), `$1${MASKED}`)
    .replace(
      new RegExp(`("[\\w-]*${SECRET_WORD}[\\w-]*"\\s*:\\s*)("[^"]*"|-?\\d+(?:\\.\\d+)?)`, "gi"),
      `$1"${MASKED}"`,
    )
    .replace(/eyJ[\w-]+\.[\w-]+\.[\w-]+/g, MASKED) // JWT
    .replace(/\b\d{13,19}\b/g, (m) => (luhnValid(m) ? maskCard(m) : m)); // card numbers only (Luhn-valid)
}

export function mask(value: unknown, key = ""): unknown {
  if (value === null || value === undefined) return value;
  const primitive = typeof value === "string" || typeof value === "number" || typeof value === "bigint" || typeof value === "boolean";
  if (primitive && isCardKey(key)) return maskCard(String(value));
  if (primitive && isSecretKey(key)) return MASKED; // strings AND numbers (pin: 1234, otp: 123456)
  if (typeof value === "string") return maskText(value);
  if (Array.isArray(value)) return value.map((v) => mask(v, key));
  if (typeof value === "object") {
    return Object.fromEntries(Object.entries(value as Record<string, unknown>).map(([k, v]) => [k, mask(v, k)]));
  }
  return value;
}

export function maskUrl(url: string): string {
  return maskText(url);
}
```

---

## sendRequest (with request/response logging)

File: `src/commons/common.ts`

One function for ALL calls. When `testInfo` is passed, it attaches the masked request **before** the call and the masked response (or the error) **after** it, on pass and fail, so timeouts and connection errors are logged too. `common.ts` imports only types from Playwright, so it never imports a fixture (no circular imports).

```typescript
import type { APIRequestContext, APIResponse, TestInfo } from "@playwright/test";
import { mask, maskText, maskUrl } from "@utils/mask";

export type RequestOptions = {
  endpoint: string;
  data?: object | string;
  form?: Record<string, string>;
  headers?: Record<string, string>;
  method?: "post" | "get" | "put" | "delete";
  baseUrl?: string;
  label?: string; // e.g. "step1-start"; defaults to the endpoint
  testInfo?: TestInfo; // pass from the test to attach req/res to the report
};

export type SendResult = {
  response: APIResponse;
  responseBody: unknown;
  durationMs: number;
};

function parseBody(text: string): unknown {
  const trimmed = text.trim();
  if (trimmed.startsWith("{") || trimmed.startsWith("[")) {
    try {
      return JSON.parse(trimmed);
    } catch {
      return { raw: text }; // malformed JSON: keep it raw so the test can assert on it
    }
  }
  return { raw: text };
}

export async function sendRequest(apiContext: APIRequestContext, options: RequestOptions): Promise<SendResult> {
  const { endpoint, data, form, headers, method = "post", baseUrl, label = endpoint, testInfo } = options;
  const url = baseUrl ? `${baseUrl}/${endpoint}` : endpoint;

  if (testInfo) {
    await testInfo.attach(`${label}-request`, {
      body: JSON.stringify(
        { method: method.toUpperCase(), url: maskUrl(url), headers: mask(headers ?? {}), body: mask(data ?? form ?? null) },
        null,
        2,
      ),
      contentType: "application/json",
    });
  }

  const started = Date.now();
  let response: APIResponse | undefined;
  let responseBody: unknown = null;
  let failure: string | undefined;

  try {
    response = await apiContext[method](url, {
      ...(data !== undefined && { data }),
      ...(form !== undefined && { form }),
      ...(headers !== undefined && { headers }),
      timeout: 30000,
    });
    responseBody = parseBody(await response.text());
    return { response, responseBody, durationMs: Date.now() - started };
  } catch (error) {
    failure = error instanceof Error ? error.message.split("\n")[0] : String(error);
    throw error; // let the test fail with the real error
  } finally {
    if (testInfo) {
      const durationMs = Date.now() - started;
      const logged = response
        ? { status: response.status(), durationMs, url: maskUrl(response.url()), headers: mask(response.headers()), body: mask(responseBody) }
        : { error: `no response: ${maskText(failure ?? "unknown error")}`, durationMs };
      await testInfo.attach(`${label}-response`, { body: JSON.stringify(logged, null, 2), contentType: "application/json" });
    }
  }
}
```

---

## Authentication

File: `src/commons/auth.ts`

```typescript
import type { APIRequestContext } from "@playwright/test";
import { z } from "zod";

export type TokenConfig = {
  oauthUrl: string;
  clientId: string;
  clientSecret: string;
  contentType?: "form" | "json";
};

const tokenResponseSchema = z.object({
  access_token: z.string(),
  expires_in: z.number(),
});

const tokenCache = new Map<string, { token: string; expiresAt: number }>();

export async function getAccessToken(request: APIRequestContext, config: TokenConfig): Promise<string> {
  const cacheKey = `${config.oauthUrl}_${config.clientId}`;
  const cached = tokenCache.get(cacheKey);
  if (cached && Date.now() < cached.expiresAt - 60000) return cached.token;

  const body = { grant_type: "client_credentials", client_id: config.clientId, client_secret: config.clientSecret };
  const response = await request.post(config.oauthUrl, config.contentType === "json" ? { data: body } : { form: body });
  if (!response.ok()) throw new Error(`Token request failed: HTTP ${response.status()}`); // never log the body or secret

  const parsed = tokenResponseSchema.safeParse(await response.json());
  if (!parsed.success) throw new Error(`Token response schema mismatch: ${parsed.error.message}`);

  tokenCache.set(cacheKey, { token: parsed.data.access_token, expiresAt: Date.now() + parsed.data.expires_in * 1000 });
  return parsed.data.access_token;
}
```

**Usage.** `beforeAll` can't use test-scoped fixtures such as `apiContext`, so create a short-lived request context from the worker-scoped `playwright` fixture:

```ts
let token = "";

test.beforeAll(async ({ playwright }, workerInfo) => {
  const request = await playwright.request.newContext();
  try {
    token = await getAccessToken(request, {
      oauthUrl: getConfigVar("OAUTH_URL", workerInfo.project.name),
      clientId: getConfigVar("CLIENT_ID", workerInfo.project.name),
      clientSecret: getConfigVar("CLIENT_SECRET", workerInfo.project.name),
    });
  } finally {
    await request.dispose();
  }
});

// in the test: sendRequest(apiContext, { ..., headers: { Authorization: `Bearer ${token}` }, testInfo })
```

---

## Environment Config

File: `src/config/project.config.ts`

```typescript
// Project name -> env var prefix. Production is deliberately absent: add it only if your team runs
// production smoke tests on purpose. Auto Mode never runs a production project.
const prefixMap: Readonly<Record<string, string>> = { staging: "STAGING", sandbox: "SANDBOX" };

export function getConfigVar(key: string, environment: string): string {
  const prefix = prefixMap[environment];
  if (!prefix) throw new Error(`Unknown environment: ${environment}`);
  const envVarName = `${prefix}_${key}`;
  const value = process.env[envVarName];
  if (!value) throw new Error(`Missing: ${envVarName}`); // name only, never the value
  return value;
}
```

### .env.example

Commit `.env.example` with names only. The real `.env` stays git-ignored.

```bash
SANDBOX_BASE_URL=
SANDBOX_NOTIFY_BASE_URL=
SANDBOX_MERCHANT_ID=
SANDBOX_SECRET_KEY=
SANDBOX_DB_HOST=
SANDBOX_DB_NAME=
SANDBOX_DB_USER=
SANDBOX_DB_PASSWORD=
```

---

## Fixtures

File: `src/fixtures/apicontext.fixture.ts`

```typescript
import { test as base, expect, type APIRequestContext } from "@playwright/test";
import { getConfigVar } from "@config/project.config";

type TestFixtures = { apiContext: APIRequestContext; environment: string };

export const test = base.extend<TestFixtures>({
  environment: [
    async ({}, use, testInfo) => {
      await use(testInfo.project.name);
    },
    { scope: "test" },
  ],

  apiContext: [
    async ({ playwright, environment }, use) => {
      const context = await playwright.request.newContext({
        baseURL: getConfigVar("BASE_URL", environment),
        extraHTTPHeaders: { "Content-Type": "application/json" },
      });
      await use(context);
      await context.dispose();
    },
    { scope: "test" },
  ],
});

export { expect };
```

File: `src/fixtures/ui.fixture.ts`

```typescript
import { test as base, expect } from "@playwright/test";

export const test = base; // extend here if page-level setup (auth storage state, etc.) is needed
export { expect };
```

---

## Spec File

File: `src/tests/api/payment/start-payment.spec.ts`

```typescript
import { test, expect } from "@fixtures/apicontext.fixture";
import { sendRequest } from "@commons/common";
import { readCsv } from "@utils/csv";
import { requireEnv } from "@utils/env";
import { parseOrFail } from "@utils/schema";
import { ENDPOINTS } from "@constants/endpoints";
import type { StartPaymentTestData } from "@payloads/payment/start-payment";
import { buildPayload } from "@builders/payment/start-payment";
import { responseEnvelopeSchema, startPaymentResponseSchema } from "@schemas/payment/start-payment";

const testData = readCsv<StartPaymentTestData>("./csv/start-payment.csv");
// For JSON: const testData = readJson<StartPaymentTestData>("./data/start-payment.json");

for (const row of testData) {
  test(
    `[${row.test_id}][PAYMENT][MERCHANT][start-payment] - ${row.test_name} - response ${row.expected_response_code} : ${row.expected_response_message}`,
    { tag: row.release_tag ? ["@start-payment", row.release_tag] : ["@start-payment"] },
    async ({ apiContext }, testInfo) => {
      const { payload, merchantReference } = buildPayload(row, requireEnv(row.merchant_id_env), requireEnv(row.secret_key_env));

      const { response, responseBody } = await sendRequest(apiContext, {
        endpoint: ENDPOINTS.START_PAYMENT,
        data: payload,
        label: "start-payment",
        testInfo,
      });

      // Record the reference right after the call, before any assertion can fail.
      await testInfo.attach("transaction-ref", {
        body: JSON.stringify({ merchant_reference: merchantReference }),
        contentType: "application/json",
      });

      expect(response.status(), `[${row.test_id}] HTTP status: expected 200, got ${response.status()}`).toBe(200);

      const envelope = parseOrFail(responseEnvelopeSchema, responseBody, row.test_id, "response envelope");
      expect(envelope.response_code, `[${row.test_id}] response_code: expected "${row.expected_response_code}", got "${envelope.response_code}"`).toBe(row.expected_response_code);
      expect(envelope.response_message, `[${row.test_id}] response_message: expected "${row.expected_response_message}", got "${envelope.response_message}"`).toBe(row.expected_response_message);

      if (row.expected_response_code === "00") {
        parseOrFail(startPaymentResponseSchema, responseBody, row.test_id, "start-payment success");
      }
    },
  );
}
```

---

## Assertion Pattern — Descriptive Error Messages

Always include a custom message in `expect()` so failures are immediately understandable:

```ts
// ✅ Good: failure shows  [TC-001] response_code: expected "00", got "01"
expect(envelope.response_code, `[${row.test_id}] response_code: expected "${row.expected_response_code}", got "${envelope.response_code}"`).toBe(row.expected_response_code);

// ❌ Bad: failure shows  Expected "00", received "01"  (no context which test or field)
expect(envelope.response_code).toBe(row.expected_response_code);
```

**Format:** `[{test_id}] {field}: expected "{expected}", got "{actual}"`

| Assertion type | Message format |
|----------------|---------------|
| HTTP status | `[${row.test_id}] HTTP status: expected {n}, got ${response.status()}` |
| Response field | `[${row.test_id}] {field_name}: expected "${expected}", got "${actual}"` |
| DB field | `[${row.test_id}] DB {column}: expected "${expected}", got "${actual}"` |
| Null check | `[${row.test_id}] {field_name}: expected record to exist but got null` (use `.not.toBeNull()`, never `.toBeTruthy()`) |
| Schema | `parseOrFail(schema, body, row.test_id, "{label}")` throws `[{test_id}] {label} schema mismatch: ...` |

---

## Multi-Step Flow

One test per row; each step is a `test.step()`. This keeps the `for` loop at the top level, keeps state local to the test (no shared mutable variables), and a retry re-runs the whole flow for that row only. Use separate serial tests (`test.describe.configure({ mode: "serial" })`) only when each step must appear as its own test in the report; note that a retry then re-runs the whole serial group.

File: `src/tests/api/payment/start-payment-flow.spec.ts`

```typescript
import { test, expect } from "@fixtures/apicontext.fixture";
import { sendRequest } from "@commons/common";
import { readCsv } from "@utils/csv";
import { requireEnv } from "@utils/env";
import { parseOrFail } from "@utils/schema";
import { ENDPOINTS } from "@constants/endpoints";
import { getConfigVar } from "@config/project.config";
import type { StartPaymentFlowTestData } from "@payloads/payment/start-payment";
import { buildPayload, buildNotifyPayload } from "@builders/payment/start-payment";
import { notifyResponseSchema, responseEnvelopeSchema, startPaymentResponseSchema } from "@schemas/payment/start-payment";
import { getTransaction, closeDbPool } from "@db/queries/payment";

const testData = readCsv<StartPaymentFlowTestData>("./csv/start-payment-flow.csv");

test.afterAll(async () => {
  await closeDbPool();
});

for (const row of testData) {
  test(
    `[${row.test_id}][PAYMENT][MERCHANT][start-payment-flow] - ${row.test_name} - response ${row.expected_response_code} : ${row.expected_response_message}`,
    { tag: ["@start-payment-flow"] },
    async ({ apiContext, environment }, testInfo) => {
      const secretKey = requireEnv(row.secret_key_env);
      const { payload, merchantReference } = buildPayload(row, requireEnv(row.merchant_id_env), secretKey);

      const paymentCode = await test.step("step 1: start payment", async () => {
        const { responseBody } = await sendRequest(apiContext, { endpoint: ENDPOINTS.START_PAYMENT, data: payload, label: "step1-start", testInfo });
        const envelope = parseOrFail(responseEnvelopeSchema, responseBody, row.test_id, "step 1 envelope");
        expect(envelope.response_code, `[${row.test_id}] step 1 response_code: expected "${row.expected_response_code}", got "${envelope.response_code}"`).toBe(row.expected_response_code);
        if (envelope.response_code !== "00") return null;

        const success = parseOrFail(startPaymentResponseSchema, responseBody, row.test_id, "step 1 success");
        await testInfo.attach("transaction-ref", {
          body: JSON.stringify({ merchant_reference: merchantReference, payment_code: success.payment_code }),
          contentType: "application/json",
        });
        return success.payment_code;
      });

      if (paymentCode === null) {
        await test.step("step 2: DB unchanged", async () => {
          const record = await getTransaction(merchantReference, environment);
          expect(record, `[${row.test_id}] DB: expected no record for a rejected request, got one`).toBeNull();
        });
        return;
      }

      await test.step("step 2: notify", async () => {
        const { responseBody } = await sendRequest(apiContext, {
          endpoint: ENDPOINTS.NOTIFY,
          baseUrl: getConfigVar("NOTIFY_BASE_URL", environment),
          data: buildNotifyPayload(paymentCode, merchantReference, secretKey),
          label: "step2-notify",
          testInfo,
        });
        const notify = parseOrFail(notifyResponseSchema, responseBody, row.test_id, "step 2 notify");
        expect(notify.response_code, `[${row.test_id}] step 2 response_code: expected "${row.expected_notify_code}", got "${notify.response_code}"`).toBe(row.expected_notify_code);
      });

      await test.step("step 3: DB verify", async () => {
        const record = await getTransaction(merchantReference, environment);
        expect(record, `[${row.test_id}] step 3 DB: expected record to exist but got null`).not.toBeNull();
        expect(record?.STATUS, `[${row.test_id}] step 3 DB STATUS: expected "${row.expected_db_status}", got "${record?.STATUS}"`).toBe(row.expected_db_status);
      });
    },
  );
}
```

---

## Database

File: `src/db/connection.ts`

```typescript
import sql, { type ConnectionPool } from "mssql";
import { getConfigVar } from "@config/project.config";

let pool: ConnectionPool | null = null;

export async function getDbPool(environment: string): Promise<ConnectionPool> {
  if (pool?.connected) return pool;
  pool = await sql.connect({
    server: getConfigVar("DB_HOST", environment),
    database: getConfigVar("DB_NAME", environment),
    user: getConfigVar("DB_USER", environment),
    password: getConfigVar("DB_PASSWORD", environment),
    // trustServerCertificate: true is for test environments with self-signed certs only.
    // Never use it against a production database.
    options: { encrypt: true, trustServerCertificate: true },
    pool: { max: 5, min: 1, idleTimeoutMillis: 30000 },
  });
  return pool;
}

export async function closeDbPool(): Promise<void> {
  if (pool?.connected) {
    await pool.close();
    pool = null;
  }
}
```

File: `src/db/queries/payment.ts`

```typescript
import { getDbPool } from "@db/connection";

export { closeDbPool } from "@db/connection";

export type TransactionRow = {
  MERCHANT_REFERENCE: string;
  PAYMENT_CODE: string;
  STATUS: string;
};

export async function getTransaction(merchantReference: string, environment: string): Promise<TransactionRow | null> {
  const pool = await getDbPool(environment);
  const result = await pool
    .request()
    .input("merchantReference", merchantReference)
    .query<TransactionRow>("SELECT TOP 1 MERCHANT_REFERENCE, PAYMENT_CODE, STATUS FROM TRANSACTIONS WHERE MERCHANT_REFERENCE = @merchantReference");
  return result.recordset[0] ?? null;
}
```

---

## Data Readers

File: `src/utils/csv.ts`

```typescript
import fs from "fs";
import { parse } from "csv-parse/sync";

export function readCsv<T>(filePath: string): T[] {
  return parse(fs.readFileSync(filePath, "utf-8"), { columns: true, skip_empty_lines: true }) as T[];
}
```

File: `src/utils/json.ts`

```typescript
import fs from "fs";

export function readJson<T>(filePath: string): T[] {
  return JSON.parse(fs.readFileSync(filePath, "utf-8")) as T[];
}
```

---

## UI Login — Federated Auth (SSO / OAuth) & OTP

Generic template. Replace provider selectors, login path, and class name with project-specific values.

File: `src/utils/otp.ts`

```typescript
import { authenticator } from "otplib";

authenticator.options = { window: 1 }; // tolerate ±1 step clock drift

export function generateOTP(secret: string): string {
  return authenticator.generate(secret);
}

export async function waitForFreshOTP(): Promise<void> {
  const timeRemaining = 30 - (Math.floor(Date.now() / 1000) % 30);
  if (timeRemaining < 5) {
    await new Promise((resolve) => setTimeout(resolve, (timeRemaining + 1) * 1000));
  }
}

export async function waitForNextOTPWindow(): Promise<void> {
  const timeRemaining = 30 - (Math.floor(Date.now() / 1000) % 30);
  await new Promise((resolve) => setTimeout(resolve, (timeRemaining + 1) * 1000));
}
```

File: `src/locators/login.ts`

```typescript
export const loginLocators = {
  // Combine every configured provider's sign-in trigger into one selector.
  signInButton: "[data-testid='sso-sign-in'], [data-testid='google-sign-in']", // replace with real selectors
  identifierInput: "[data-testid='identifier-input']",
  identifierSubmitButton: "[data-testid='identifier-submit']",
  passwordInput: "[data-testid='password-input']",
  passwordSubmitButton: "[data-testid='password-submit']",
  otpInput: "[data-testid='otp-input']",
  otpSubmitButton: "[data-testid='otp-submit']",
  otpErrorMessage: "[data-testid='otp-error']",
  errorMessage: "[data-testid='login-error']",
  signedInMarker: "[data-testid='logout-button']", // any element only visible when signed in
} as const;
```

File: `src/pages/login.page.ts`

```typescript
import type { Locator, Page } from "@playwright/test";
import { expect } from "@fixtures/ui.fixture";
import { generateOTP, waitForFreshOTP, waitForNextOTPWindow } from "@utils/otp";
import { loginLocators } from "@locators/login";

const LOGIN_PATH = "/login"; // replace with the project's login route

export class LoginPage {
  readonly signInButton: Locator;
  readonly identifierInput: Locator;
  readonly identifierSubmitButton: Locator;
  readonly passwordInput: Locator;
  readonly passwordSubmitButton: Locator;
  readonly otpInput: Locator;
  readonly otpSubmitButton: Locator;
  readonly otpErrorMessage: Locator;
  readonly errorMessage: Locator;
  readonly signedInMarker: Locator;

  constructor(private readonly page: Page) {
    this.signInButton = page.locator(loginLocators.signInButton);
    this.identifierInput = page.locator(loginLocators.identifierInput);
    this.identifierSubmitButton = page.locator(loginLocators.identifierSubmitButton);
    this.passwordInput = page.locator(loginLocators.passwordInput);
    this.passwordSubmitButton = page.locator(loginLocators.passwordSubmitButton);
    this.otpInput = page.locator(loginLocators.otpInput);
    this.otpSubmitButton = page.locator(loginLocators.otpSubmitButton);
    this.otpErrorMessage = page.locator(loginLocators.otpErrorMessage);
    this.errorMessage = page.locator(loginLocators.errorMessage);
    this.signedInMarker = page.locator(loginLocators.signedInMarker);
  }

  async goTo(): Promise<void> {
    await this.page.goto(LOGIN_PATH);
  }

  /**
   * Submits credentials (and OTP if provided). Settles once the attempt resolves (signed in
   * or rejected) and asserts neither. The calling spec asserts the expected outcome.
   */
  async signIn(identifier: string, password: string, otpSecret?: string): Promise<void> {
    await this.signInButton.click();
    await this.identifierInput.fill(identifier);
    await this.identifierSubmitButton.click();
    await this.passwordInput.fill(password);
    await this.passwordSubmitButton.click();

    if (otpSecret) {
      await this.submitOtp(otpSecret);
    }

    // Gate on page elements, not URL: federated flows redirect through another host mid-flow.
    await expect(this.signedInMarker.or(this.errorMessage).first(), "login did not settle: neither signed-in marker nor error visible").toBeVisible();
  }

  private async submitOtp(secret: string, maxRetries = 3): Promise<void> {
    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      if (attempt === 0) await waitForFreshOTP();
      else await waitForNextOTPWindow();

      await this.otpInput.fill(generateOTP(secret));
      await this.otpSubmitButton.click();

      const errorVisible = await this.otpErrorMessage
        .waitFor({ timeout: 3000 })
        .then(() => true)
        .catch(() => false); // timeout here means "no OTP error", not a hidden failure

      if (!errorVisible) return;
      await this.otpInput.clear();
    }
  }
}
```

File: `src/tests/ui/login.spec.ts`

```typescript
import { test, expect } from "@fixtures/ui.fixture";
import { LoginPage } from "@pages/login.page";
import { requireEnv } from "@utils/env";

test("[TC-001][UI][ADMIN][login] - valid credentials - signs in successfully", { tag: ["@login"] }, async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goTo();
  await loginPage.signIn(requireEnv("LOGIN_USERNAME"), requireEnv("LOGIN_PASSWORD"), process.env.LOGIN_OTP_SECRET);

  await expect(loginPage.signedInMarker, "[TC-001] expected signed-in marker to be visible").toBeVisible();
});
```

---

## UI Page Object Model

File: `src/locators/payment.ts`

```typescript
export const paymentPageLocators = {
  submitButton: "[data-testid='submit']",
  amountField: "#amount",
  statusLabel: ".payment-status",
  errorMessage: "[data-testid='error-msg']",
} as const;
```

File: `src/pages/payment.page.ts`

```typescript
import type { Locator, Page } from "@playwright/test";
import { expect } from "@fixtures/ui.fixture";
import { paymentPageLocators } from "@locators/payment";

export class PaymentPage {
  readonly submitButton: Locator;
  readonly amountField: Locator;
  readonly statusLabel: Locator;

  constructor(private readonly page: Page) {
    this.submitButton = page.locator(paymentPageLocators.submitButton);
    this.amountField = page.locator(paymentPageLocators.amountField);
    this.statusLabel = page.locator(paymentPageLocators.statusLabel);
  }

  async fillAmount(amount: string): Promise<void> {
    await this.amountField.fill(amount);
  }

  async submit(): Promise<void> {
    await this.submitButton.click();
  }

  async expectStatus(text: string): Promise<void> {
    await expect(this.statusLabel, `payment status: expected to contain "${text}"`).toContainText(text, { timeout: 15000 });
  }
}
```

---

## Playwright Config

File: `playwright.config.ts`

```typescript
import { defineConfig } from "@playwright/test";
import dotenv from "dotenv";

dotenv.config();

export default defineConfig({
  testDir: "./src/tests",
  timeout: 60000,
  // retries: 1 captures a trace on the first retry, but a retry RE-SENDS every request.
  // If your tests create real transactions, use retries: 0 + trace: "retain-on-failure" instead.
  retries: 1,
  workers: 4,
  use: {
    trace: "on-first-retry",
    screenshot: "only-on-failure",
  },
  reporter: [
    ["html", { open: "never", outputFolder: "playwright-report" }],
    ["list"],
    ["./src/reporters/markdown-failure.reporter.ts"], // writes failures.md
  ],
  // Non-production projects only. Add a production project only for deliberate smoke tests;
  // Auto Mode never runs one.
  projects: [
    { name: "staging", testMatch: /.*\.spec\.ts/ },
    { name: "sandbox", testMatch: /.*\.spec\.ts/ },
  ],
});
```

---

## package.json

```json
{
  "devDependencies": {
    "@playwright/test": "^1.45.0",
    "zod": "^3.23.0",
    "csv-parse": "^5.5.0",
    "mssql": "^11.0.0",
    "@types/mssql": "^9.1.0",
    "dotenv": "^16.4.0",
    "typescript": "^5.5.0",
    "tsx": "^4.19.0"
  },
  "scripts": {
    "test:staging": "npx playwright test --project=staging",
    "test:sandbox": "npx playwright test --project=sandbox",
    "test:start-payment": "npx playwright test --project=sandbox --grep @start-payment",
    "typecheck": "tsc --noEmit",
    "report": "npx playwright show-report"
  }
}
```

---

## CI/CD — Jenkinsfile

```groovy
pipeline {
  agent { kubernetes { yaml readTrusted('cicd/pod-template.yaml') } }
  parameters {
    choice(name: 'ENVIRONMENT', choices: ['staging', 'sandbox'])
    string(name: 'TAG_FILTER', defaultValue: '')
  }
  environment {
    SANDBOX_BASE_URL    = credentials('sandbox-base-url')
    SANDBOX_MERCHANT_ID = credentials('sandbox-merchant-id')
    SANDBOX_SECRET_KEY  = credentials('sandbox-secret-key')
  }
  stages {
    stage('Install') { steps { sh 'npm ci'; sh 'npx playwright install chromium --with-deps' } }
    stage('Test') { steps { script {
      def grep = params.TAG_FILTER ? "--grep '${params.TAG_FILTER}'" : ""
      sh "npx playwright test --project=${params.ENVIRONMENT} ${grep}"
    }}}
  }
  post {
    always {
      archiveArtifacts artifacts: 'failures.md, playwright-report/**, test-results/**', allowEmptyArchive: true
      publishHTML(target: [reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Report'])
    }
  }
}
```

### Pod template — `cicd/pod-template.yaml`

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: playwright
      image: mcr.microsoft.com/playwright:v1.45.0-jammy
      command: ['sleep', 'infinity']
      resources:
        requests: { memory: "2Gi", cpu: "1" }
        limits: { memory: "4Gi", cpu: "2" }
```

---

## Failure Reporting & Artifacts

See [SKILL.md](./SKILL.md) → "Failure Reporting & Artifacts" for the rules. Three artifacts form a ladder: `failures.md` (quick triage) → HTML report (browse) → trace (forensic). Config wiring lives in the Playwright Config and Jenkinsfile above.

### Markdown failure reporter

File: `src/reporters/markdown-failure.reporter.ts`

```typescript
import type { Reporter, TestCase, TestResult, FullResult } from "@playwright/test/reporter";
import fs from "fs";

class MarkdownFailureReporter implements Reporter {
  private failures: string[] = [];

  onTestEnd(test: TestCase, result: TestResult): void {
    if (result.status === "passed" || result.status === "skipped") return;

    const attachments = result.attachments
      .filter((a) => a.name !== "screenshot") // screenshots are already in the HTML report
      .map((a) => `  - ${a.name}: \`${a.path ?? "inline"}\``)
      .join("\n");

    this.failures.push(
      [
        `### ❌ ${test.title}`,
        `- **File:** \`${test.location.file}:${test.location.line}\``,
        `- **Status:** ${result.status} (retry ${result.retry})`,
        "- **Error:**",
        "  ```",
        `  ${(result.error?.message ?? "no message").trim()}`,
        "  ```",
        attachments ? `- **Attachments:**\n${attachments}` : "",
      ]
        .filter(Boolean)
        .join("\n"),
    );
  }

  onEnd(result: FullResult): void {
    const header =
      `# Test Failures\n\n` +
      `- **Run status:** ${result.status}\n` +
      `- **Failed:** ${this.failures.length}\n` +
      `- **Generated:** ${new Date().toISOString()}\n\n---\n\n`;
    const body = this.failures.length ? this.failures.join("\n\n---\n\n") : "✅ No failures.";
    fs.writeFileSync("failures.md", header + body);
  }
}

export default MarkdownFailureReporter;
```

### Request/response context on failure

`sendRequest` attaches the masked request and response for every call when the test passes `testInfo` (see [sendRequest](#sendrequest-with-requestresponse-logging)), so failed tests already carry their context in the report and `failures.md`. For a DB step, attach the queried record the same way, through `mask()`:

```ts
await testInfo.attach("db-record", { body: JSON.stringify(mask(record), null, 2), contentType: "application/json" });
```

### Sample rendered `failures.md`

What the artifact looks like when opened on Jenkins / GitHub / VS Code preview (2 of 15 tests failed):

```markdown
# Test Failures

- **Run status:** failed
- **Failed:** 2
- **Generated:** 2026-09-13T05:04:00.000Z

---

### ❌ [TC-002][PAYMENT][MERCHANT][start-payment] - missing amount - response 01 : invalid request
- **File:** `src/tests/api/payment/start-payment.spec.ts:24`
- **Status:** failed (retry 0)
- **Error:**
  ```
  Error: [TC-002] response_code: expected "01", got "00"
  expect(received).toBe(expected)
  Expected: "01"
  Received: "00"
  ```
- **Attachments:**
  - start-payment-request: `test-results/start-payment-TC-002/start-payment-request.json`
  - start-payment-response: `test-results/start-payment-TC-002/start-payment-response.json`

---

### ❌ [TC-007][PAYMENT][MERCHANT][start-payment] - invalid currency - response 01 : invalid currency
- **File:** `src/tests/api/payment/start-payment.spec.ts:24`
- **Status:** failed (retry 0)
- **Error:**
  ```
  Error: [TC-007] HTTP status: expected 200, got 500
  ```
- **Attachments:**
  - start-payment-response: `test-results/start-payment-TC-007/start-payment-response.json`
```

### Investigation ladder

| Step | Command / action                          | Answers                                  |
| ---- | ----------------------------------------- | ---------------------------------------- |
| 1    | Open `failures.md`                        | Which tests failed + the exact mismatch  |
| 2    | Open "Playwright Report" (published HTML) | Browse/filter, view attachments per test |
| 3    | `npx playwright show-trace trace.zip`     | Every request/response + DOM snapshots   |

---

## Auto Mode — Setup, Pre-run Check, Run & Summary

Templates for [SKILL.md → Auto Mode](./SKILL.md#auto-mode-intake--review--generate--run--report).

### One-time setup (per repo)

Auto Mode needs these pieces. If any are missing, the agent proposes them once as a single setup change, waits for approval, and from then on Auto Mode runs without that stop.

| Piece                     | File                                         | If the repo already has one                                                       |
| ------------------------- | -------------------------------------------- | --------------------------------------------------------------------------------- |
| Mask helper               | `src/utils/mask.ts`                          | Reuse it; add missing keys                                                        |
| Logging in `sendRequest`  | `src/commons/common.ts`                      | Add the `testInfo` option and attach calls to the existing function. Never create a second `sendRequest` |
| `parseOrFail`, `requireEnv` | `src/utils/schema.ts`, `src/utils/env.ts`  | Reuse equivalents                                                                 |
| Markdown reporter         | `src/reporters/markdown-failure.reporter.ts` | Reuse it                                                                          |
| Pre-run check script      | `scripts/auto-run-precheck.ts`               | Reuse it                                                                          |
| Summary script            | `scripts/auto-run-summary.ts`                | Reuse it                                                                          |
| `tsx` dev dependency      | `package.json`                               | Skip                                                                              |
| `.gitignore` entries      | `.gitignore`                                 | Add only the missing lines                                                        |

### Pre-run check

File: `scripts/auto-run-precheck.ts`

Checks credentials (names only), the environment, and `.gitignore`. The environment check is an **allowlist**: the run goes ahead only when BOTH the project name and the base URL host contain a non-production marker, and neither contains a production marker. A host such as `api.example.com` or a bare IP is blocked, and the agent asks the user.

```typescript
import fs from "fs";
import dotenv from "dotenv";

const NON_PROD_MARKERS: readonly string[] = ["sandbox", "uat", "dev", "develop", "test", "testing", "staging", "stg", "qa", "sit", "local", "localhost"];
const PROD_MARKERS: readonly string[] = ["prod", "production", "prd", "live"];
const GITIGNORE_ENTRIES: readonly string[] = ["auto-run/", "playwright-report/", "test-results/", "failures.md", ".env"];

function tokens(value: string): string[] {
  return value.toLowerCase().split(/[^a-z0-9]+/).filter(Boolean);
}

export function classifyEnvironment(projectName: string, baseUrl: string | undefined): { ok: boolean; reason: string } {
  let host: string;
  try {
    host = new URL(baseUrl ?? "").hostname;
  } catch {
    return { ok: false, reason: "base URL is missing or invalid" };
  }
  const projectTokens = tokens(projectName);
  const hostTokens = tokens(host);
  if ([...projectTokens, ...hostTokens].some((t) => PROD_MARKERS.includes(t))) {
    return { ok: false, reason: `production marker in project "${projectName}" or host "${host}"` };
  }
  const missing: string[] = [];
  if (!projectTokens.some((t) => NON_PROD_MARKERS.includes(t))) missing.push(`project "${projectName}"`);
  if (!hostTokens.some((t) => NON_PROD_MARKERS.includes(t))) missing.push(`host "${host}"`);
  if (missing.length) return { ok: false, reason: `no non-production marker in ${missing.join(" and ")}` };
  return { ok: true, reason: `project "${projectName}", host "${host}"` };
}

export function missingEnvVars(names: readonly string[]): string[] {
  return names.filter((n) => !process.env[n]);
}

export function missingGitignoreEntries(gitignoreText: string): string[] {
  const lines = gitignoreText.split(/\r?\n/).map((l) => l.trim());
  return GITIGNORE_ENTRIES.filter((e) => !lines.includes(e) && !lines.includes(e.replace(/\/$/, "")));
}

function arg(name: string): string | undefined {
  const i = process.argv.indexOf(`--${name}`);
  return i >= 0 ? process.argv[i + 1] : undefined;
}

if (require.main === module) {
  dotenv.config();
  const project = arg("project") ?? "";
  const baseUrlVar = arg("base-url-var") ?? `${project.toUpperCase()}_BASE_URL`;
  const envVars = (arg("env-vars") ?? "").split(",").filter(Boolean);

  const env = classifyEnvironment(project, process.env[baseUrlVar]);
  const missingVars = missingEnvVars([baseUrlVar, ...envVars]);
  const missingIgnore = missingGitignoreEntries(fs.existsSync(".gitignore") ? fs.readFileSync(".gitignore", "utf-8") : "");

  console.log(`${env.ok ? "OK     " : "BLOCKED"} environment: ${env.reason}`);
  console.log(missingVars.length ? `BLOCKED env vars missing: ${missingVars.join(", ")}` : "OK      env vars present (names checked, values not printed)");
  console.log(missingIgnore.length ? `BLOCKED not git-ignored: ${missingIgnore.join(", ")}` : "OK      outputs git-ignored");
  process.exit(env.ok && !missingVars.length && !missingIgnore.length ? 0 : 1);
}
```

Run: `npx tsx scripts/auto-run-precheck.ts --project sandbox --env-vars SANDBOX_MERCHANT_ID,SANDBOX_SECRET_KEY`

Exit code 0 means go. Any `BLOCKED` line is a stop condition: report it and ask the user. Never edit the marker lists to make a run pass.

### Run command

```bash
RUN_ID=$(date +%Y%m%d-%H%M%S)-start-payment
PLAYWRIGHT_HTML_OUTPUT_DIR="auto-run/$RUN_ID/html" \
PLAYWRIGHT_JSON_OUTPUT_NAME="auto-run/$RUN_ID/results.json" \
npx playwright test --project=sandbox --grep @start-payment \
  --retries=0 --trace=retain-on-failure \
  --reporter=list,html,json,./src/reporters/markdown-failure.reporter.ts

npx tsx scripts/auto-run-summary.ts "$RUN_ID" sandbox
npx playwright show-report "auto-run/$RUN_ID/html"
```

- `--reporter` **replaces** the config's reporters, so list every reporter Auto Mode needs (list, html, json, markdown). Leaving one out silently drops its output.
- `--retries=0` overrides the config's `retries`, so a failing test does not send its requests twice.
- `--trace=retain-on-failure` keeps a trace for failed tests even without a retry (`on-first-retry` never fires with `--retries=0`).
- `PLAYWRIGHT_HTML_OUTPUT_DIR` puts each run's HTML report in its own folder. Playwright versions before 1.46 use `PLAYWRIGHT_HTML_REPORT` instead.

### Run summary writer

File: `scripts/auto-run-summary.ts`

Reads `auto-run/{run-id}/results.json`, copies `failures.md` into the run folder, and writes `auto-run/{run-id}/summary.md`, including every `transaction-ref` attachment.

```typescript
import fs from "fs";
import path from "path";

type JsonAttachment = { name: string; body?: string; path?: string; contentType: string };
type JsonResult = { status: string; error?: { message?: string }; attachments?: JsonAttachment[] };
type JsonTest = { results: JsonResult[] };
type JsonSpec = { title: string; tests: JsonTest[] };
type JsonSuite = { specs?: JsonSpec[]; suites?: JsonSuite[] };
type JsonReport = { suites: JsonSuite[] };
type Row = { title: string; status: string; error: string; refs: string[] };

function readAttachment(a: JsonAttachment): string {
  if (a.body) return Buffer.from(a.body, "base64").toString("utf-8");
  if (a.path && fs.existsSync(a.path)) return fs.readFileSync(a.path, "utf-8");
  return "";
}

function collect(suite: JsonSuite, out: Row[]): void {
  for (const spec of suite.specs ?? []) {
    for (const t of spec.tests) {
      const last = t.results.at(-1);
      out.push({
        title: spec.title,
        status: last?.status ?? "unknown",
        error: (last?.error?.message ?? "").split("\n")[0],
        refs: (last?.attachments ?? []).filter((a) => a.name === "transaction-ref").map(readAttachment),
      });
    }
  }
  for (const child of suite.suites ?? []) collect(child, out);
}

const [, , runId, env] = process.argv;
if (!runId || !env) throw new Error("Usage: tsx scripts/auto-run-summary.ts <run-id> <env>");

const outDir = path.join("auto-run", runId);
const report = JSON.parse(fs.readFileSync(path.join(outDir, "results.json"), "utf-8")) as JsonReport;
const rows: Row[] = [];
report.suites.forEach((s) => collect(s, rows));

if (fs.existsSync("failures.md")) fs.copyFileSync("failures.md", path.join(outDir, "failures.md"));

const count = (...statuses: string[]) => rows.filter((r) => statuses.includes(r.status)).length;
const cell = (s: string) => s.replace(/\|/g, "\\|").replace(/\n/g, " ");
const refRows = rows.flatMap((r) => r.refs.map((ref) => `| ${cell(r.title)} | ${cell(ref)} |`));

const md = [
  `# Auto Mode run ${runId} (${env})`,
  "",
  "| Passed | Failed | Skipped |",
  "|---|---|---|",
  `| ${count("passed")} | ${count("failed", "timedOut", "interrupted")} | ${count("skipped")} |`,
  "",
  "| Test | Status | First error line |",
  "|---|---|---|",
  ...rows.map((r) => `| ${cell(r.title)} | ${r.status} | ${cell(r.error)} |`),
  "",
  "## Transactions created",
  "",
  ...(refRows.length ? ["| Test | References |", "|---|---|", ...refRows] : ["None recorded."]),
  "",
  `Reports: \`${outDir}/html/index.html\`, \`${outDir}/failures.md\`, decisions in \`${outDir}/decisions.md\``,
].join("\n");

fs.writeFileSync(path.join(outDir, "summary.md"), md);
console.log(`Wrote ${path.join(outDir, "summary.md")}`);
```

### `.gitignore` additions

```
.env
auto-run/
playwright-report/
test-results/
failures.md
```
