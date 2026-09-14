# DIAGRAMS — Visual Workflow & Architecture

Visual diagrams for the qa-automation skill. See [SKILL.md](./SKILL.md) for detailed rules and [REFERENCE.md](./REFERENCE.md) for code templates.

> **Domain examples are illustrative.** Some diagrams use payment-style specifics (response codes `00`/`01`, `payment_code`, a notify step) as concrete examples. The flow *structure* is what generalizes — substitute your own codes, resource IDs, and follow-up calls.

---

## 1. Skill Activation Flow

What happens when the skill is triggered:

```mermaid
flowchart TD
    A[User Input] --> B{Input Type?}
    B -->|CSV / JSON / curl / API spec| C[Phase 1: Analyze]
    B -->|"add test for X"| D[Ask for details]
    B -->|"test is failing"| E[Debug Flow]
    B -->|"just do it"| F[Skip to Tasks]

    D --> C
    C --> G[Phase 2: Requirements]
    G --> H[Phase 3: Design]
    H --> I[Phase 4: Tasks]
    I --> J[Implementation]

    F --> J

    E --> K{Identify Symptom}
    K --> L[Check CSV alignment]
    K --> M[Check env vars]
    K --> N[Check payload/URL]
    L --> O[Fix & Re-run]
    M --> O
    N --> O
```

---

## 2. Workflow Phases

The four-phase process before writing any code:

```mermaid
flowchart LR
    subgraph Phase1[Phase 1: Analyze]
        A1[Identify endpoint]
        A2[Request fields]
        A3[Response fields]
        A4[Error cases]
    end

    subgraph Phase2[Phase 2: Requirements]
        R1[Acceptance criteria]
        R2[SHALL language]
        R3[Pipeline order]
    end

    subgraph Phase3[Phase 3: Design]
        D1[Architecture flow]
        D2[Decision logic]
        D3[Files to create/modify]
        D4[Override/injection pattern]
    end

    subgraph Phase4[Phase 4: Tasks]
        T1[Checkboxes ordered by pipeline]
        T2[Each references requirement]
        T3[Implementable steps]
    end

    Phase1 -->|User approves| Phase2
    Phase2 -->|User approves| Phase3
    Phase3 -->|User approves| Phase4
    Phase4 -->|User approves| IMPL[Start coding]
```

---

## 3. Data Pipeline Architecture

How data flows from test data through each layer to assertions:

```mermaid
flowchart TD
    subgraph Input["Test Data Layer"]
        CSV[CSV File]
        JSON[JSON File]
    end

    subgraph Transform["Transform Layer"]
        TYPE[Type Definition<br/><code>src/types/</code>]
        SENTINEL[Sentinel Resolver<br/><code>__EMPTY__ → ""</code><br/><code>__NULL__ → null</code><br/><code>__DYNAMIC__ → runtime</code><br/><code>blank → omit field</code>]
        BUILDER[Builder<br/><code>src/builders/</code>]
    end

    subgraph Execute["Execution Layer"]
        ENV[".env → credentials"]
        AUTH[Auth Token<br/><code>beforeAll + cache</code>]
        SEND["sendRequest()<br/><code>src/commons/</code>"]
    end

    subgraph Validate["Validation Layer"]
        ZOD[Zod Schema<br/><code>src/schemas/</code>]
        RESP[Response Assert<br/><code>code + message</code>]
        DB[DB Assert<br/><code>src/db/queries/</code>]
    end

    CSV --> TYPE
    JSON --> TYPE
    TYPE --> SENTINEL
    SENTINEL --> BUILDER
    BUILDER --> SEND
    ENV --> AUTH
    AUTH --> SEND
    SEND --> ZOD
    SEND --> RESP
    RESP -->|success path| DB
```

---

## 4. Multi-Step Branching Flow

How multi-step tests branch based on response:

```mermaid
flowchart TD
    START[Build Payload from Row] --> CALL1[Step 1: Call Main API]
    CALL1 --> ASSERT1[Assert response_code & message]

    ASSERT1 --> CHECK{response_code?}

    CHECK -->|"00" success| SUCCESS_PATH
    CHECK -->|"01" error| ERROR_PATH

    subgraph SUCCESS_PATH[Success Path]
        S1[Extract payment_code]
        S2[Step 2: Call Notify API]
        S3[Assert notify response]
        S4[Step 3: Query DB]
        S5[Assert DB status = expected]
        S1 --> S2 --> S3 --> S4 --> S5
    end

    subgraph ERROR_PATH[Error Path]
        E1[Step 2: Query DB]
        E2[Assert state unchanged / no record]
        E1 --> E2
    end

    style SUCCESS_PATH fill:#d4edda,stroke:#28a745
    style ERROR_PATH fill:#f8d7da,stroke:#dc3545
```

---

## 5. Decision Tree: Do I Need Code Changes?

When adding a new test case, decide what needs to change:

```mermaid
flowchart TD
    START[New test case needed] --> Q1{Same endpoint<br/>as existing test?}

    Q1 -->|Yes| Q2{New request field<br/>in CSV/JSON?}
    Q1 -->|No| FULL[Full Pipeline:<br/>CSV → Type → Builder<br/>→ Schema → Spec]

    Q2 -->|No| Q3{New response field<br/>to assert?}
    Q2 -->|Yes| PARTIAL_REQ[Update:<br/>CSV column + Type + Builder]

    Q3 -->|No| EASY[Just add CSV/JSON row<br/>No code changes!]
    Q3 -->|Yes| PARTIAL_RESP[Update:<br/>Zod Schema + Spec assertion]

    PARTIAL_REQ --> Q3

    style EASY fill:#d4edda,stroke:#28a745
    style PARTIAL_REQ fill:#fff3cd,stroke:#ffc107
    style PARTIAL_RESP fill:#fff3cd,stroke:#ffc107
    style FULL fill:#f8d7da,stroke:#dc3545
```

---

## 6. Project Structure Overview

```mermaid
flowchart TD
    subgraph Data["Data Sources"]
        CSV_DIR["csv/<br/>One CSV per endpoint"]
        DATA_DIR["data/<br/>One JSON per endpoint"]
        ENV[".env<br/>Credentials per environment"]
    end

    subgraph Core["Core Pipeline (src/)"]
        TYPES["types/{module}/<br/>Row type definitions"]
        BUILDERS["builders/{module}/<br/>Payload constructors"]
        SCHEMAS["schemas/{module}/<br/>Zod response schemas"]
        COMMONS["commons/<br/>sendRequest + auth"]
        CONSTANTS["constants/<br/>Endpoint paths"]
    end

    subgraph Support["Support (src/)"]
        CONFIG["config/<br/>Environment resolution"]
        UTILS["utils/<br/>CSV/JSON parser, crypto"]
        DB["db/queries/<br/>DB connection + queries"]
        FIXTURES["fixtures/<br/>Playwright fixtures"]
    end

    subgraph Tests["Test Files"]
        SPECS["tests/api/{module}/<br/>Spec files"]
    end

    subgraph UI["UI Testing (optional)"]
        LOCATORS["locators/<br/>Selectors only"]
        PAGES["pages/<br/>Actions + assertions"]
    end

    CSV_DIR --> TYPES
    DATA_DIR --> TYPES
    TYPES --> BUILDERS
    BUILDERS --> SPECS
    SCHEMAS --> SPECS
    COMMONS --> SPECS
    CONSTANTS --> SPECS
    ENV --> CONFIG
    CONFIG --> FIXTURES
    FIXTURES --> SPECS
    DB --> SPECS
    LOCATORS --> PAGES
    PAGES --> SPECS
```

---

## 7. Authentication & Token Lifecycle

```mermaid
sequenceDiagram
    participant Spec as Spec File
    participant Auth as auth.ts
    participant Cache as Token Cache
    participant OAuth as OAuth Server

    Spec->>Auth: getAccessToken(config)
    Auth->>Cache: Check cache for clientId

    alt Token valid (>60s remaining)
        Cache-->>Auth: Return cached token
        Auth-->>Spec: token
    else Token expired or missing
        Auth->>OAuth: POST /token (client_credentials)
        OAuth-->>Auth: { access_token, expires_in }
        Auth->>Cache: Store token + expiry
        Auth-->>Spec: token
    end

    Note over Spec: Use token in sendRequest headers
```

---

## 8. CI/CD Pipeline

```mermaid
flowchart LR
    subgraph Trigger
        PARAM[Parameters:<br/>ENVIRONMENT<br/>TAG_FILTER<br/>BROWSER]
    end

    subgraph Pipeline
        INSTALL[Install deps<br/><code>npm ci</code>]
        BROWSER[Install browser<br/><code>playwright install</code>]
        TEST[Run tests<br/><code>playwright test</code>]
        REPORT[Publish HTML report]
        ARCHIVE[Archive artifacts<br/><code>post always</code>]
    end

    subgraph Infra
        K8S[Kubernetes Pod]
        VAULT[Secrets → env vars]
    end

    subgraph Artifacts["Artifacts (downloadable, even on failure)"]
        FAIL["failures.md<br/>quick triage"]
        HTML["playwright-report/**<br/>browse"]
        TRACE["test-results/**<br/>trace + screenshots"]
    end

    PARAM --> INSTALL
    INSTALL --> BROWSER
    BROWSER --> TEST
    TEST --> REPORT
    REPORT --> ARCHIVE
    K8S --> INSTALL
    VAULT --> TEST
    ARCHIVE --> FAIL
    ARCHIVE --> HTML
    ARCHIVE --> TRACE
```

---

## Diagram Legend

| Symbol | Meaning |
|--------|---------|
| 🟢 Green box | Easy / no code changes |
| 🟡 Yellow box | Partial code changes |
| 🔴 Red box | Full pipeline changes needed |
| Diamond | Decision point |
| Rectangle | Action or process |
| Subgraph | Logical grouping |
