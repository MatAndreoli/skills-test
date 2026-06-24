---
name: ruby-to-java-component-test-migration
description: Use when migrating Ruby Cucumber component test .feature files to Java integration tests (JUnit 5 + RestAssured + WireMock + Quarkus). Accepts a single file, multiple files, a directory, or "all" to migrate the full suite with a phased plan.
---

# Ruby Component Test → Java Integration Test Migration

Migrates Ruby Cucumber `.feature` files to Java integration test classes and WireMock stub files. Always creates from scratch — no existing test file is assumed.

## Input Modes

The skill accepts four input forms. Detect which applies, then follow the corresponding instructions below before running the standard migration phases.

| Input | Example | Behaviour |
|-------|---------|-----------|
| Single file | `path/to/foo.feature` | Migrate that one file. After each generated test class, run `mvn verify -P integration-tests -Dtest="<ClassName>"` to confirm it compiles and passes. |
| Multiple files | `foo.feature bar.feature` | Activate **Coordinator Mode** (see below). |
| Directory | `src/test/ruby/components/features/` | Enumerate all `.feature` files in that directory (non-recursive). Activate **Coordinator Mode** (see below). |
| All features | `all` (or no path given) | Run the **Phased Migration Plan** procedure (see below) first, then activate **Coordinator Mode** per phase. |

**When a test fails after migration** — do not block progress. Mark the failing test with `@Disabled` and a description, then continue. Do not spend time debugging during migration.

```java
@Test
@Disabled("Migration: fails with XXX — needs investigation")
void context_condition_retornaN() { ... }
```

Add `import org.junit.jupiter.api.Disabled;` only when used. The disabled test serves as a marker to revisit later.

**Migrate first, investigate only on failure.** Do not read application source code or trace business logic before attempting migration. Translate the scenario mechanically from the `.feature` + TShield stubs. Run the test. Only if it fails **and** the cause is not an obvious migration error (wrong URL, missing header, gRPC status misconfiguration, missing stub) should you investigate the real application flow.

### Phased Migration Plan (for "all" input)

When migrating the entire suite, migrating everything at once maximises risk. Instead:

**Step 1 — Discovery.** List every `.feature` file under `src/test/ruby/components/features/`. For each file record:
- Number of scenarios
- Whether it uses MongoDB (`Dado que o mongo`)
- Whether it uses RabbitMQ (`tópico`, `exchange`)
- Number of distinct session names (`Dado que o "..."`)

Present a discovery table to the user before migrating anything.

**Step 2 — Risk classification.** Assign each file to a phase:

| Phase | Criteria |
|-------|----------|
| 1 — Simple | HTTP only, no MongoDB, no RabbitMQ, no gRPC, ≤ 3 sessions |
| 2 — Moderate | HTTP + MongoDB, or > 3 sessions, no RabbitMQ, no gRPC |
| 3 — Complex | RabbitMQ, gRPC stubs, ≥ 10 scenarios, or both MongoDB and many sessions |

**Step 3 — Present the plan to the user** (the discovery table + phase assignment) and ask for approval before starting any migration.

**Step 4 — Phased execution.** Migrate one phase at a time:
1. Migrate all Phase 1 files.
2. Run `mvn verify -P integration-tests`. If failures remain, mark them `@Disabled("Migration: <reason>")` and continue.
3. Migrate all Phase 2 files.
4. Run `mvn verify -P integration-tests`. Same rule: `@Disabled` unresolved failures before continuing.
5. Migrate all Phase 3 files.
6. Run `mvn verify -P integration-tests` (full suite).

Within each phase, activate **Coordinator Mode** (see below): group the phase's files by shared sessions, dispatch agents in parallel, await all agents, then run `mvn verify -P integration-tests` once. Agents mark individual tests `@Disabled("Migration: <reason>")` for any test that fails and whose cause is not immediately obvious.

After each file is processed, mark the Ruby `.feature` file's `Funcionalidade` line:
- All tests passing → add `@migrado`
- Any test marked `@Disabled` → add `@migrado_wip`

Also add a comment on the line immediately before `Funcionalidade` indicating the corresponding Java class:

```gherkin
@migrado
# Java: com.example.<module>.integrationtests.tests.<ClassName>IntegrationTest
Funcionalidade: Invoice Barcode
```

## Coordinator Mode

**Activation rule:**
- 1 file → standard migration (phases 1–7 directly, no coordinator)
- 2+ files → coordinator mode (this section)

Applies to: multiple files, directory, and `all`.

### Coordinator Responsibilities

The main Claude Code instance acts as coordinator. Execute these steps before dispatching any agent:

**1. Read shared infrastructure once:**
- `src/test/ruby/components/features/step_definitions/api_steps.rb`
- `src/test/ruby/components/features/step_definitions/sessions_steps.rb`
- `src/test/ruby/components/features/step_definitions/database_steps.rb`
- `src/test/ruby/components/features/step_definitions/queue_steps.rb`
- Any existing test class in `integrationtests/tests/` (for package name)
- All files in `src/test/java/com/example/<module>/integrationtests/support/docs/` (existing Doc builders; `<module>` derived from step above)
- List of existing dirs in `src/test/resources/integrationtests/stub-sessions/`

**2. Scan all target `.feature` files** to extract session names AND read their full content (to embed in agent prompts later). Read `sessions_steps.rb` to derive the step patterns that call `start_session(...)` — do not hardcode. Session names may be delimited by `"` or `'`. Do not read TShield stubs yet.

**3. Identify gRPC services** referenced across all target files. Read the matching `.proto` files once before dispatching.

**4. Detect conflicts and group files** (see algorithm below).

**5. Per batch (phase for `all` mode, or the full set for other modes):**
- Dispatch agents in parallel — one per conflict group
- Await all agents
- Run `mvn verify -P integration-tests`
- Apply `@migrado`/`@migrado_wip` tags and Java class comments to `.feature` files

> **Note:** In coordinator mode, the per-file `mvn verify -Dtest="<ClassName>"` step from the Phased Migration Plan is **replaced** by this single batch `mvn verify -P integration-tests` after all agents finish. Agents never run Maven.

### Conflict Detection and Grouping

```
1. Build map: session name → list of .feature files using it
2. Files sharing ≥1 session name → same group → same agent
3. Non-overlapping groups → independent agents (dispatch in parallel)
```

Sessions already present in `stub-sessions/` do **not** create conflicts — agents only reuse them.

**Example:**
```
foo.feature → sessions: ["oauth-token", "consultar-cliente"]
bar.feature → sessions: ["oauth-token", "validar-contrato"]   ← shares "oauth-token"
baz.feature → sessions: ["buscar-fatura"]                     ← independent

Agents:
  Agent A → [foo.feature, bar.feature]   ─┐ parallel
  Agent B → [baz.feature]                ─┘
```

Within a single agent, files are migrated sequentially.

### Agent Prompt Template

Pass the following to each agent. Fill `<...>` with actual content read in step 1 above:

```
You are migrating Ruby Cucumber .feature files to Java integration tests.

**First action:** Read the full skill file at `~/.claude/skills/ruby-to-java-component-test-migration/SKILL.md` and follow Phases 1–7 exactly.

SHARED CONTEXT (pre-read — treat as authoritative, do not re-read these files):

=== api_steps.rb ===
<full content>

=== sessions_steps.rb ===
<full content>

=== database_steps.rb ===
<full content>

=== queue_steps.rb ===
<full content>

=== Package name ===
<e.g. com.example.manager>

=== Existing Doc builders in support/docs/ ===
<class names and their fields>

=== Proto files ===
<full content of relevant .proto files>

=== Existing stub sessions (reuse; do not recreate) ===
<list of dir names already in stub-sessions/>

ASSIGNED FILES:
<for each file: path + full content>

RULES:
- Execute Phases 1–7 for each assigned file in sequence.
- SHARED CONTEXT above is authoritative — do NOT re-read api_steps.rb, sessions_steps.rb, database_steps.rb, queue_steps.rb, proto files, or check stub-sessions/ for existing dirs. Use what was provided.
- Do NOT run Maven (`mvn`). Other shell commands (e.g. `protoc` for `.dsc` generation in Phase 3) are allowed.
- Do NOT tag the .feature file (@migrado / @migrado_wip) — coordinator handles that.
- After all files, return a report in the format below.

REPORT FORMAT:
FILE: <path/to/file.feature>
  JAVA_CLASS: <path/to/XxxIntegrationTest.java>
  STUBS: <comma-separated stub file paths>
  STATUS: ok | disabled:<reason>
  SESSIONS_CREATED: <new session dir names, or "none">
---
```

### Agent Return Format

```
FILE: src/test/ruby/components/features/foo.feature
  JAVA_CLASS: src/test/java/com/example/mymodule/integrationtests/tests/FooIntegrationTest.java
  STUBS: src/test/resources/integrationtests/stub-sessions/consultar-cliente/payment-service/stub.json, src/test/resources/integrationtests/stub-sessions/consultar-cliente/oauth/stub.json
  STATUS: ok
  SESSIONS_CREATED: consultar-cliente
---
FILE: src/test/ruby/components/features/bar.feature
  JAVA_CLASS: src/test/java/com/example/mymodule/integrationtests/tests/BarIntegrationTest.java
  STUBS: src/test/resources/integrationtests/stub-sessions/validar-contrato/service-b/stub.json
  STATUS: disabled:Migration: fails with ClassNotFoundException — needs investigation
  SESSIONS_CREATED: validar-contrato
---
```

### Post-Phase Flow (Coordinator)

After all agents in a batch return:

1. Run `mvn verify -P integration-tests`.
2. For each `.feature` file in this batch:
   - Agent reported `ok` AND `mvn verify` passes → add `@migrado` + Java class comment before `Funcionalidade`.
   - Agent reported `disabled:*` OR `mvn verify` has failures for this class → add `@migrado_wip` instead.

```gherkin
@migrado_wip
# Java: com.example.manager.integrationtests.tests.BarIntegrationTest
Funcionalidade: Bar Feature
```

3. For `all` mode: proceed to next phase only after tagging is complete.

## Prerequisites

Before generating any file, detect the project module name by reading the `package` declaration of an existing test class under `src/test/java/`. Never hardcode the module name.

## Phase 1: Analysis

> **Multi-file sessions:** Items 2–8 are shared infrastructure — read them once per invocation, not once per feature file. Items 1, 9, and 10 are file-specific — repeat for each `.feature` file being migrated.
>
> **Item 8 exception (multi-file only):** Proto files can only be read once all sessions are known. In multi-file mode, do a first pass over all target `.feature` files (item 1) to collect every session name and identify which gRPC services appear. Then read the matching `.proto` files once before starting the actual migration.

Read these files in order before writing anything:

1. **Target `.feature` file** — extract: feature-level tags, all scenarios, per-scenario tags, all steps *(repeat per file)*
   - **Parameter substitution:** Values matching `$xxx` in feature steps are placeholders resolved at runtime from `src/test/ruby/components/config/parameters.yml`. Read that file once when you first encounter a `$xxx` value. Replace every `$xxx` occurrence with its YAML value before using it in Java test code or WireMock stubs.
   - **`$boolean_true` / `$boolean_false`** are built-in special cases — they resolve to the Java booleans `true` / `false` respectively. They are **not** in `parameters.yml`.
2. **`src/test/ruby/components/features/step_definitions/api_steps.rb`** — resolve each `Quando` step to URL, HTTP method, and param/header structure *(read once)*
3. **`src/test/ruby/components/features/step_definitions/sessions_steps.rb`** — verify that `Dado que o "<text>"` calls `start_session(text.gsub(' ', '-'))`, meaning it maps to `loadStubs("<kebab-text>")` *(read once)*
4. **`src/test/ruby/components/features/step_definitions/database_steps.rb`** — identify MongoDB setup steps and their collection names *(read once)*
5. **`src/test/ruby/components/features/step_definitions/queue_steps.rb`** — identify RabbitMQ publish/consume steps and exchange names *(read once)*
6. **Any existing test class in `integrationtests/tests/`** — confirm package name and naming conventions *(read once)*
7. **`src/test/java/com/example/<module>/integrationtests/support/docs/`** — check for existing Doc builder classes before creating new ones *(read once)*
8. **Proto files in `src/main/proto/`** — for every gRPC service encountered across all sessions being migrated, read the matching `.proto` once to get exact package, service name, and rpc method name *(read once)*
9. **`src/test/ruby/components/requests/<session-name>/`** — for each session used, read the TShield stub files. **HTTP stubs:** service subdirectory contains `N.json` (URL, method, status, headers) + `N.content` (body), N starting at 0. If `1.json` exists → sequential calls. **gRPC stubs:** service directory uses Ruby class notation (`Module::ClassName/method_name/`) and contains hash subdirectories with `0.original_request` (request JSON), `0.response` (success body JSON) or `0.error` (error JSON), and `0.response_class` (informational). Map each N (0-based) to WireMock file number N+1 (1-based). *(repeat per file)*
10. **`src/test/resources/integrationtests/stub-sessions/`** — check if any session dir with the exact same name already exists (reuse it; do not recreate) *(repeat per file)*

**Step not found:** if a `Quando` step in the `.feature` file has no matching definition in `api_steps.rb` → pause and ask the user before continuing.

## Phase 2: Feature Flag Mapping

**Tag handling — only act on toggle and wip/broken tags. Ignore all other tags.**

| Location | Ruby tag | Java annotation |
|----------|----------|-----------------|
| `Funcionalidade` (feature-level) | `@toggletrue_<flag>` | `@WithFlags(enable = "<flag>")` on class |
| `Funcionalidade` (feature-level) | `@togglefalse_<flag>` | `@WithFlags(disable = "<flag>")` on class |
| `Funcionalidade` (feature-level) | both true and false | `@WithFlags(enable = "rt-x", disable = "rt-y")` on class |
| `Cenário` (scenario-level) | `@toggletrue_<flag>` | `@WithFlags(enable = "<flag>")` on method |
| `Cenário` (scenario-level) | `@togglefalse_<flag>` | `@WithFlags(disable = "<flag>")` on method |
| `Cenário` (scenario-level) | both true and false | `@WithFlags(enable = "rt-x", disable = "rt-y")` on method |
| Either | `@wip` or `@broken` | migrate but add `// @wip: <scenario title>` on the line immediately before `@Test` — leave test enabled |
| Either | any other tag | **ignore** |

`@WithFlags` at class level is used **only** when the `Funcionalidade` line itself has a toggle. Scenario-level toggles always go on the method. When both class and method have `@WithFlags`, both annotations are present — the library applies them independently. No merging needed.

## Phase 3: WireMock Stub Sessions

### Directory naming

Ruby session name → remove accents (ç→c, ã→a, ê→e, etc.) → kebab-case:

- `"payment service returns authenticated url successfully"` → `payment-service-returns-authenticated-url`
- `"payment service returns service error"` → `payment-service-returns-error`

### Structure

```
src/test/resources/integrationtests/
  stub-sessions/
    <session-name>/
      <service>/      ← one dir per downstream service (e.g. payment-service, oauth, service-b, service-c)
        stub.json     ← or 1-desc.json / 2-desc.json if service called multiple times
  __files/
    <session-name>/
      <service>/
        response.json
```

`loadStubs("<session-name>")` loads the entire session directory tree.

**Service directory name** is derived from the service identifier in the Ruby request path (e.g. `payment-service`, `service-b`, `service-c`, `service-d`, `oauth`). If the service cannot be identified → **pause and ask the user**.

### URL field in request

| Field | When to use |
|-------|-------------|
| `"urlPath"` | Exact path match; use with `"queryParameters"` to match query string separately |
| `"urlPathPattern"` | Regex path match (e.g. `".*/Service/Method"`) |
| `"url"` | Exact full URL match (path with no separate query string needed) |

### Response body placement

- Small body (JSON fits on one line under 120 chars) → inline `"body": "{...}"` in `stub.json`
- Large body → save to `__files/<session-name>/<service>/response.<ext>` (extension matches content type: `.json`, `.xml`, etc.), reference via `"bodyFileName": "<session-name>/<service>/response.<ext>"`
- `bodyFileName` is resolved by WireMock relative to the `__files/` directory — do not include `__files/` in the path
- If the exact same file path already exists in `__files/` → reuse it; otherwise create a new file

### Stub examples

**Simple GET with query parameters:**

```json
{
  "request": {
    "method": "GET",
    "urlPath": "/billingAccountConfigurationManagement/v1/customers/bill/1234567890/barcode",
    "queryParameters": {
      "imageType": {
        "equalTo": "1"
      }
    }
  },
  "response": {
    "status": 200,
    "headers": {
      "Content-Type": "application/json"
    },
    "body": "{\"barCode\":\"846300000029793900800012112345678908122202301034\"}"
  }
}
```

**POST with regex URL and external body file:**

```json
{
  "request": {
    "method": "POST",
    "urlPathPattern": ".*/Cliente/ConsultarClienteB2B"
  },
  "response": {
    "status": 200,
    "headers": {
      "Content-Type": "text/xml; charset=utf-8"
    },
    "bodyFileName": "consultar-cliente-b2b-sucesso/service-c/response.xml"
  }
}
```

### Multiple calls to the same service

**Detection:** a service directory in TShield contains `0.json` + `0.content`. If `1.json` is also present, the service is called multiple times.

**TShield → WireMock mapping:**

| TShield file | WireMock file |
|--------------|---------------|
| `0.json` / `0.content` | `1-<description>.json` |
| `1.json` / `1.content` | `2-<description>.json` |
| `N.json` / `N.content` | `(N+1)-<description>.json` |

Read request/status/headers from `N.json`. Use `N.content` as the response body (apply the small/large body placement rule).

Number the WireMock stub files starting at 1:

```
<service>/
  1-<description>.json
  2-<description>.json
```

#### Choosing the matching strategy

**Inspect the TShield directory names** for each call. TShield encodes the request URL (including query string) in the subdirectory name. If the calls have **different query params or path params**, they are distinguishable by their parameters — use exact parameter matching. If the calls are **identical** (same URL, same params), they cannot be told apart — use WireMock scenarios.

| Calls differ by | Strategy |
|-----------------|----------|
| Query params (`?q=...`, `?imageType=...`) | `"queryParameters"` with `"equalTo"` on each param — **no scenarios** |
| Path params (`/customers/123` vs `/customers/456`) | `"urlPath"` with `"equalTo"` — **no scenarios** |
| Nothing (identical URL + params) | WireMock scenario state machine |

**Prefer exact matching over scenarios whenever possible.** Scenarios are stateful and order-dependent; exact matching is stateless and more robust.

#### Exact query param matching (preferred)

Each stub gets its own file with `"queryParameters"` → `"equalTo"` for every distinguishing param. No `scenarioName`, `requiredScenarioState`, or `newScenarioState` fields.

Real example — Salesforce Asset queries (same endpoint `/salesforce/v2/data/v44.0/query`, four calls each with a different `q` SQL):

`1-account-contact-relation.json`:
```json
{
  "request": {
    "method": "GET",
    "urlPathPattern": ".*/salesforce/v2/data/v44.0/query",
    "queryParameters": {
      "q": {
        "equalTo": "SELECT account.BI_No_Identificador_fiscal__c,AccountId,Isprincipal__c,Papel_Contato__c,Contact.Name,Account.Name,Account.SegmentacaoCliente__c FROM AccountContactRelation WHERE Contact.BI_Numero_de_documento__c='documentId' AND Papel_Contato__c!=null AND account.recordtype.name IN ('Prospect','Cliente') LIMIT 2000 OFFSET 0"
      }
    }
  },
  "response": {
    "status": 200,
    "headers": { "Content-Type": "application/json" },
    "bodyFileName": "session-name/service-b/account-contact-relation.json"
  }
}
```

`2-atlys.json`:
```json
{
  "request": {
    "method": "GET",
    "urlPathPattern": ".*/salesforce/v2/data/v44.0/query",
    "queryParameters": {
      "q": {
        "equalTo": "SELECT AccountId,SystemOrigin__c,productFamily__c,ProductCode FROM Asset WHERE AccountId = '0013600000EXAMPLE1' AND SystemOrigin__c = 'ATLYS'  LIMIT 1"
      }
    }
  },
  "response": {
    "status": 200,
    "headers": { "Content-Type": "application/json" },
    "body": "{\"totalSize\":1,\"done\":true,\"records\":[{\"SystemOrigin__c\":\"ATLYS\",\"ProductFamily__c\":\"M2M\",\"ProductCode\":null}]}"
  }
}
```

`3-legacy-system.json`:
```json
{
  "request": {
    "method": "GET",
    "urlPathPattern": ".*/salesforce/v2/data/v44.0/query",
    "queryParameters": {
      "q": {
        "equalTo": "SELECT AccountId,SystemOrigin__c,productFamily__c,ProductCode FROM Asset WHERE AccountId = '0013600000EXAMPLE1' AND SystemOrigin__c = 'LEGACY_SYSTEM'  LIMIT 1"
      }
    }
  },
  "response": {
    "status": 200,
    "headers": { "Content-Type": "application/json" },
    "body": "{\"totalSize\":1,\"done\":true,\"records\":[{\"SystemOrigin__c\":\"LEGACY_SYSTEM\",\"ProductFamily__c\":\"Internet\",\"ProductCode\":null}]}"
  }
}
```

`4-pabx.json` and `5-internetx.json` follow the same pattern, each with their own `equalTo` on `q`.

> **Deriving the exact SQL from TShield:** TShield encodes the query string in the subdirectory name. Some use URL-encoded strings (e.g. `salesforce-v2-data-v44.0-query?q=SELECT+AccountId...`), others use a hash (e.g. `salesforce-v2-data-v44.0-query?b3ae03ff...`). For URL-encoded names, URL-decode to get the exact value. For hashed names, derive the query from the application source code (look for `String.format(QUERY_TEMPLATE, ...)` calls with the relevant parameters).

#### WireMock scenario state machine (identical calls only)

Use only when the same URL + same params are called multiple times and need to return different responses in sequence.

Rules:
- `requiredScenarioState` of the first file is always `"Started"`
- Each file sets `"newScenarioState"` to the state required by the next file
- The last file has no `"newScenarioState"`

**Complete two-call HTTP example (truly identical requests):**

`1-first-call.json`:
```json
{
  "scenarioName": "barcode",
  "requiredScenarioState": "Started",
  "newScenarioState": "SECOND_CALL",
  "request": {
    "method": "GET",
    "urlPath": "/billingAccountConfigurationManagement/v1/customers/bill/1234567890/barcode",
    "queryParameters": {
      "imageType": {
        "equalTo": "1"
      }
    }
  },
  "response": {
    "status": 200,
    "headers": { "Content-Type": "application/json" },
    "body": "{\"barCode\":\"846300000029793900800012112345678908122202301034\"}"
  }
}
```

`2-second-call.json`:
```json
{
  "scenarioName": "barcode",
  "requiredScenarioState": "SECOND_CALL",
  "request": {
    "method": "GET",
    "urlPath": "/billingAccountConfigurationManagement/v1/customers/bill/1234567890/barcode",
    "queryParameters": {
      "imageType": {
        "equalTo": "1"
      }
    }
  },
  "response": {
    "status": 200,
    "headers": { "Content-Type": "application/json" },
    "body": "{\"barCode\":\"846300000029793900800012112345678908122202301035\"}"
  }
}
```

### gRPC stubs

#### Pre-requisites (do before creating any gRPC stub)

**Generate `.dsc` descriptor file for each proto being mocked.**
Without the `.dsc`, WireMock cannot convert JSON ↔ protobuf — file-based gRPC stubs will not work.

Check if `src/test/resources/integrationtests/wiremock-grpc/grpc/<proto-name>.dsc` already exists. If it does → skip. If not → generate it:

```bash
# Find the protoc binary (Maven downloads it during build)
# Check pom.xml for <protoc.version>, then:
~/.m2/repository/com/google/protobuf/protoc/<version>/protoc-<version>-linux-x86_64.exe \
  --descriptor_set_out=src/test/resources/integrationtests/wiremock-grpc/grpc/<proto-name>.dsc \
  --include_imports \
  --proto_path=src/main/proto \
  src/main/proto/<path/to/service.proto>
```

- `--include_imports`: required — without it WireMock cannot resolve imported proto types and fails to decode messages
- `--descriptor_set_out`: path relative to project root; the lib reads all `.dsc` files present in `src/test/resources/integrationtests/wiremock-grpc/grpc/`
- The generated `.dsc` must be committed alongside the code — it does not change unless the `.proto` changes
- If `protoc` binary not found (build never run) → run `mvn generate-sources` first

#### Detection

A service subdirectory in TShield is a **gRPC stub** (not HTTP) when its name uses Ruby class notation: `Module::ClassName/method_name/`. HTTP stubs use URL-like path segments (e.g. `billing-service-v1/get/`).

#### URL and directory derivation

The TShield Ruby class name does **not** reliably match the proto service name — always read the proto file:

1. From `Module::ClassName/method_name` in TShield, identify the proto file in `src/main/proto/` whose `package` and `service` match (case-insensitive, ignoring version suffixes like V2).
2. Read the proto to get the exact: `package`, `service`, and `rpc` method name.
3. WireMock URL: `POST /<package>.<ServiceName>/<MethodName>`
4. Stub directory: `grpc/<package>/<service-kebab>-<method-kebab>/stub.json`
   - Service and method names converted to kebab-case: `AccountV2` → `accountv2`, `UserAccountsV2` → `user-accounts-v2`

**Example:**

TShield: `Accounts::Account/user_accounts/` → find `src/main/proto/v2/accounts.proto` (package `accounts`, service `AccountV2`, rpc `UserAccountsV2`):
- URL: `POST /accounts.AccountV2/UserAccountsV2`
- Dir: `grpc/accounts/accountv2-user-accounts-v2/stub.json`

TShield: `Customers::CustomerAccess/has_access/` → find `src/main/proto/customer_access.proto` (package `customers`, service `CustomerAccess`, rpc `HasAccess`):
- URL: `POST /customers.CustomerAccess/HasAccess`
- Dir: `grpc/customers/customeraccess-has-access/stub.json`

If the matching proto file cannot be found → **pause and ask the user**.

#### Stub file format

gRPC stubs always use `"url"` (exact match), `"jsonBody"` for the response, and mandatory gRPC headers.

**`jsonBody` is required in every gRPC stub, including error stubs.** If it is absent, WireMock's `JsonMessageConverter.toMessage()` crashes with `InvalidProtocolBufferException: Expect message object but got: null`. For error stubs, use `"jsonBody": {}` (empty object).

**Success stub** (`0.response` exists):

```json
{
  "request": {
    "method": "POST",
    "url": "/accounts.AccountV2/UserAccountsV2"
  },
  "response": {
    "status": 200,
    "jsonBody": { "accounts": ["1111111111"] },
    "headers": {
      "content-type": "application/grpc",
      "grpc-status-name": "OK"
    }
  }
}
```

**Error stub** (`0.error` exists, no `0.response`):

```json
{
  "request": {
    "method": "POST",
    "url": "/customers.CustomerAccess/HasAccess"
  },
  "response": {
    "status": 200,
    "jsonBody": {},
    "headers": {
      "content-type": "application/grpc",
      "grpc-status-name": "INTERNAL"
    }
  }
}
```

**Critical:** use `"status": 200` (not 500) with `"grpc-status-name": "INTERNAL"` in headers. Setting `"status": 500` does **not** propagate as a gRPC error — WireMock serialises the `jsonBody` and returns it as a successful response regardless, so the gRPC client receives `authorized: false` (proto default) instead of a `StatusRuntimeException`. The `grpc-status-name` header is the correct mechanism to signal a gRPC error. `0.response_class` is informational — do not use in the stub.

#### Directory placement

gRPC stubs go inside a `grpc/` subdirectory of the session. `loadStubs("<session-name>")` routes files under `grpc/` to the WireMock gRPC server automatically.

```
stub-sessions/
  <session-name>/
    grpc/
      <package>/
        <service-kebab>-<method-kebab>/
          stub.json
    <http-service>/        ← HTTP stubs as normal
      stub.json
```

#### Sequential gRPC calls

If `1.response` or `1.error` exists in the same TShield hash directory, use WireMock scenarios — same rules as HTTP sequential calls (numbered files, `scenarioName`, `requiredScenarioState`, `newScenarioState`). All scenario stub files stay inside the same `grpc/<package>/<service>-<method>/` directory.

**Complete three-call gRPC example:**

`1-first-call.json`:
```json
{
  "scenarioName": "accounts-v2-sequence",
  "requiredScenarioState": "Started",
  "newScenarioState": "SECOND_CALL",
  "request": {
    "method": "POST",
    "url": "/accounts.AccountV2/UserAccountsV2"
  },
  "response": {
    "status": 200,
    "jsonBody": {
      "accounts": ["2222222222"]
    },
    "headers": {
      "content-type": "application/grpc",
      "grpc-status-name": "OK"
    }
  }
}
```

`2-second-call.json`:
```json
{
  "scenarioName": "accounts-v2-sequence",
  "requiredScenarioState": "SECOND_CALL",
  "newScenarioState": "THIRD_CALL",
  "request": {
    "method": "POST",
    "url": "/accounts.AccountV2/UserAccountsV2"
  },
  "response": {
    "status": 200,
    "jsonBody": {
      "accounts": ["3333333333"]
    },
    "headers": {
      "content-type": "application/grpc",
      "grpc-status-name": "OK"
    }
  }
}
```

`3-third-call.json`:
```json
{
  "scenarioName": "accounts-v2-sequence",
  "requiredScenarioState": "THIRD_CALL",
  "request": {
    "method": "POST",
    "url": "/accounts.AccountV2/UserAccountsV2"
  },
  "response": {
    "status": 200,
    "jsonBody": {
      "accounts": ["4444444444"]
    },
    "headers": {
      "content-type": "application/grpc",
      "grpc-status-name": "OK"
    }
  }
}
```

### Multiple sessions per scenario

When a Ruby scenario has more than one `Dado que o "..."` step, it loads multiple stub sessions. In Java, **do not call `loadStubs` multiple times**. Instead, merge all service subdirectories into a single combined session directory and call `loadStubs` once.

**Naming the combined session:** kebab-case derived from the primary (most descriptive) session name. If all sessions are equally specific, concatenate the key part of each with `-e-` (Portuguese "and"):
- Sessions `"pedido com sucesso"` + `"oauth token success"` → `pedido-com-sucesso` (oauth is infrastructure, not the business scenario — use the business name)
- Sessions `"consultar cliente"` + `"consultar contrato"` → `consultar-cliente-e-contrato`

If the right combined name is unclear → **pause and ask the user**.

**Structure for merged session:**
```
stub-sessions/
  <combined-name>/
    oauth/          ← from session A
      stub.json
    payment-service/   ← from session A
      stub.json
    service-b/            ← from session B
      stub.json
```

**Reuse rule still applies:** if the combined session dir already exists with the exact same name → reuse it; do not recreate.

### Existing stubs

If a stub session with the **exact same dir name** already exists → reuse it via `loadStubs(...)`. Do not recreate or modify it.

## Phase 4: Java Test Class

**Location:** `src/test/java/com/example/<module>/integrationtests/tests/<FeatureName>IntegrationTest.java`

**Class name:** CamelCase derived from the `.feature` filename.
- `payment_service_authenticated_url.feature` → `PaymentServiceAuthenticatedUrlIntegrationTest`
- `validate_customer_access.feature` → `ValidateCustomerAccessIntegrationTest`

**Method name:** `<context>_<condition>_retorna<StatusName>()`

Use the semantic HTTP status name, never the numeric code:

| HTTP code | Method suffix |
|-----------|--------------|
| 200 | `retornaOK` |
| 201 | `retornaCreated` |
| 204 | `retornaNoContent` |
| 400 | `retornaBadRequest` |
| 401 | `retornaUnauthorized` |
| 403 | `retornaForbidden` |
| 404 | `retornaNotFound` |
| 500 | `retornaInternalServerError` |
| 502 | `retornaBadGateway` |

Examples:
- `urlAutenticada_comCnpjNumerico_retornaOK`
- `urlAutenticada_comErroDeServico_retornaBadGateway`
- `gestorComAcesso_retornaOK`
- `documentIdEmBranco_retornaBadRequest`

**Status code assertions** — always use `HttpResponseStatus` constants, never raw integers:

| HTTP code | Assertion |
|-----------|-----------|
| 200 | `HttpResponseStatus.OK.code()` |
| 201 | `HttpResponseStatus.CREATED.code()` |
| 204 | `HttpResponseStatus.NO_CONTENT.code()` |
| 400 | `HttpResponseStatus.BAD_REQUEST.code()` |
| 401 | `HttpResponseStatus.UNAUTHORIZED.code()` |
| 403 | `HttpResponseStatus.FORBIDDEN.code()` |
| 404 | `HttpResponseStatus.NOT_FOUND.code()` |
| 500 | `HttpResponseStatus.INTERNAL_SERVER_ERROR.code()` |
| 502 | `HttpResponseStatus.BAD_GATEWAY.code()` |

**Content-Type header** — never use `.header("Content-Type", "application/json")`. Use `.contentType(ContentType.JSON)` instead (requires `import io.restassured.http.ContentType`).

> RabbitMQ test methods omit the `retorna<N>` suffix: `<context>_<condition>()` only.

**No `@DisplayName`** — not used in this project.

**Any scenario can mix concerns** — there is no "HTTP-only" or "RabbitMQ-only" test. A single scenario may combine any of:
- MongoDB data setup (insert before the request)
- HTTP mocks (`loadStubs`)
- gRPC mocks (also via `loadStubs`)
- RabbitMQ publish/consume
- MongoDB assertions (verify what was persisted after the request)

Apply every applicable section (phases 4–7) per scenario, not per class.

**`loadStubs` only when needed** — `loadStubs(String dirPath)` is inherited from `IntegrationTestBase`. Call it only if the scenario actually reaches a downstream service. If the test expects a 4xx produced by local validation (before any HTTP call is made), omit `loadStubs` entirely.

**Constants for repeated values** — if a literal value (header value, query param value, stub session name, etc.) appears in more than one test method, extract it as a `private static final` constant at the top of the class. Use `UPPER_SNAKE_CASE`.

```java
private static final String ACCESS_TOKEN = "accessToken";
private static final String PROMOTION_ID = "667";
private static final String NUMERIC_CNPJ = "11222333000181";
```

**Response body assertions** — some scenarios validate the response body, not just the status code. The Ruby step `Então a resposta contém os seguintes dados` with a data table maps to assertions based on field count:

| Fields validated | Strategy |
|-----------------|----------|
| ≤ 3 fields | `.body(field, equalTo(value))` chains |
| > 3 fields | Extract body as string + `jsonMapper.readTree` equality |

**≤ 3 fields — inline Hamcrest:**

Add `import static org.hamcrest.Matchers.equalTo;` only when used. For nested JSON paths use dot notation: `"user.name"`. For arrays use `[0]` indexing: `"items[0].id"`.

```java
given()
        .header("accessToken", ACCESS_TOKEN)
        .when()
        .get("/path")
        .then()
        .statusCode(HttpResponseStatus.OK.code())
        .body("field", equalTo("expectedValue"))
        .body("nested.field", equalTo("otherValue"));
```

**> 3 fields — full JSON comparison:**

Add `import static org.assertj.core.api.Assertions.assertThat;` when using this pattern. `jsonMapper` is inherited from `IntegrationTestBase` — no declaration needed.

```java
var expectedJson = """
        {
          "id": null,
          "document": "99888777000166",
          "accountId": "8-00000000001",
          "code": "899981490040",
          "dueDate": "2",
          "paymentMethod": "Fatura",
          "activationDate": [2020,11,20,7,50],
          "billingAddress": {
            "neighborhood": "JD CELIA",
            "streetCode": "52",
            "streetName": "DIOGO DA COSTA TAVARES"
          }
        }
        """;

var responseBody = given()
        .header("headerName", "value")
        .when()
        .get("/path")
        .then()
        .statusCode(200)
        .extract().body().asString();

assertThat(jsonMapper.readTree(responseBody)).isEqualTo(jsonMapper.readTree(expectedJson));
```

**Class template:**

```java
package com.example.<module>.integrationtests.tests;

import com.example.integrationtest.support.WithFlags; // include only if class or any method uses @WithFlags
import com.example.<module>.integrationtests.base.ManagerIntegrationTestBase;
import com.rabbitmq.client.Connection; // include only if scenario uses RabbitMQ
import io.netty.handler.codec.http.HttpResponseStatus;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType; // include only if scenario sends a request body
import org.junit.jupiter.api.Disabled;
import org.junit.jupiter.api.Test;

import java.util.ArrayList; // include only if scenario uses RabbitMQ
import java.util.List;      // include only if scenario uses RabbitMQ
import java.util.Map;       // include only if scenario uses RabbitMQ

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat; // include only if test asserts mongo docs
import static org.hamcrest.Matchers.equalTo; // include only if test asserts body fields

@QuarkusTest
class <FeatureName>IntegrationTest extends ManagerIntegrationTestBase {

    // private static final String CONSTANT = "value";  ← repeated values go here

    // @wip: <scenario title>   ← only when scenario has @wip or @broken; placed immediately before @Test
    @Test
    @WithFlags(enable = "rt-flag")   // only when scenario has @toggletrue_*
    void <context>_<condition>_retorna<N>() throws Exception {
        // 1. MongoDB setup (omit if scenario has no pre-existing data requirement)
        mongoHelper.insertInto("collection", """
                {"field": "value"}
                """);

        // 2. Load stubs (omit if scenario never calls a downstream service)
        loadStubs("session-dir-name");

        // 3. RabbitMQ consume setup (omit entire try block if scenario has no RabbitMQ)
        List<Map<String, Object>> receivedMessages = new ArrayList<>();
        try (Connection ignored = rabbitMQHelper.startConsuming(OUT_EXCHANGE, "", receivedMessages)) {

            // 4. HTTP request / RabbitMQ publish
            given()
                    .header("headerName", "value")
                    .contentType(ContentType.JSON)      // omit if scenario sends no request body
                    .queryParam("paramName", "value")
                    .when()
                    .get("/path")
                    .then()
                    .statusCode(HttpResponseStatus.OK.code())
                    .body("field", equalTo("value"));   // omit if scenario only asserts status code

            // 5. All assertions go inside waitUntil when using RabbitMQ
            waitUntil(() -> {
                assertThat(receivedMessages).isNotEmpty();   // omit if scenario does not assert received messages

                // MongoDB assertions inside waitUntil (omit if scenario does not validate persisted data — see Phase 6)
                var doc = mongoHelper.findOne("collection", Map.of("filter_field", "filter_value"));
                assertThat(doc).isNotNull();
                var expected = jsonMapper.readTree("""
                        {"field": "expectedValue"}
                        """);
                assertThat(mongoHelper.toJson(doc, "_id")).isEqualTo(expected);
            });
        }

        // MongoDB assertions without RabbitMQ (omit if scenario has RabbitMQ or does not validate persisted data — see Phase 6)
        var doc = mongoHelper.findOne("collection", Map.of("filter_field", "filter_value"));
        assertThat(doc).isNotNull();
        var expected = jsonMapper.readTree("""
                {"field": "expectedValue"}
                """);
        assertThat(mongoHelper.toJson(doc, "_id")).isEqualTo(expected);
    }
}
```

## Phase 5: MongoDB Setup

The base class calls `clearDatabase()` in `@BeforeEach`. **Never call `mongoHelper.clearCollections(...)` directly in a test method.** Instead, ensure the module's base test class overrides `clearDatabase()` to include all collections the tests use.

**Before writing any test that touches MongoDB:** read the module's base class (e.g. `InvoiceComponentTestBase`) and check which collections are already listed in `clearDatabase()`. If the scenario's collection is missing, add it:

```java
@Override
protected void clearDatabase() {
    mongoHelper.clearCollections("invoice", "agreements");
}
```

Data is inserted at the start of each test method. Exception: if **all** test methods in the class require the same base setup (e.g. a manager + customer record that every scenario assumes), extract it to `@BeforeEach` to avoid repetition — but only when the data is truly shared across every test in the class. Partial sharing (some tests need it, others don't) is not a valid reason; in that case, keep inserts in each method.

**`insertInto` has exactly two forms — pick one:**

| Condition | Use |
|-----------|-----|
| Single-use (1 test method) | Raw JSON string `"""..."""` |
| Reused across > 1 test method | Doc builder class |

**Never use `jsonToMap` with `insertInto`.** The raw JSON string form works for all document types including plain types and BSON extended types (`$date`, `$numberDecimal`).

`insertInto` accepts varargs — pass multiple documents in one call:

```java
mongoHelper.insertInto("customers",
    """{"user_document": "111", "company_document": "AAA"}""",
    """{"user_document": "222", "company_document": "BBB"}"""
);
```

Single document example (plain types):

```java
mongoHelper.insertInto("customers", """
        {
          "user_document": "11122233344",
          "company_document": "11222333000144",
          "status": "active",
          "plan": "empresarial",
          "region": "SP"
        }
        """);
```

Single document example (BSON extended types):

```java
mongoHelper.insertInto("accounts", """
        {
          "document": "99888777000166",
          "account_id": "8-00000000001",
          "code": "899981490040",
          "activation_date": {"$date": "2020-11-20T04:50:00-03:00"},
          "billing_address": {
            "street_name": "DIOGO DA COSTA TAVARES",
            "street_code": "52",
            "neighborhood": "JD CELIA"
          }
        }
        """);
```

`jsonToMap(String json)` is inherited from the base class — no import needed. Returns `Map<String, Object>`. Use it for `publishToExchange` and `findOne` filters, not for `insertInto`.

```java
// RabbitMQ publish — jsonToMap is correct here
rabbitMQHelper.publishToExchange(EXCHANGE, jsonToMap("""
        {
          "documentNumber": "12345678922",
          "agreementItem": {
            "typeBilletCurrentPayment": "Boleto/Boleto",
            "paymentBill": {
              "bankBilletCode": "846100000047855700820894996215388801018716194990",
              "billAmountCurrent": " 485.57",
              "billUnits": "1"
            }
          },
          "customerBills": [
            {"billNo": "0000000000-0"}
          ]
        }
        """));
```

**Doc builder class** — create in `src/test/java/com/example/<module>/integrationtests/support/docs/<Entity>Doc.java`:

```java
package com.example.<module>.integrationtests.support.docs;

import java.util.HashMap;
import java.util.Map;

public class <Entity>Doc {

    private String field = "defaultValue";   // one field per relevant MongoDB field
                                             // add one method per field following the same pattern

    public static <Entity>Doc defaults() { return new <Entity>Doc(); }

    public <Entity>Doc field(String v) { this.field = v; return this; }

    public Map<String, Object> build() {
        Map<String, Object> doc = new HashMap<>();
        doc.put("field_key", field);   // use the exact MongoDB field name (snake_case)
        return doc;
    }
}
```

Usage:

```java
mongoHelper.insertInto("customers",
    CustomerDoc.defaults()
        .userDocument("11122233344")
        .companyDocument("11222333000144")
        .build()
);
```

**Before creating a new Doc class:** check `support/docs/` for an existing one. Reuse and extend rather than duplicate.

## Phase 6: MongoDB Assertions

Some scenarios validate data that was **persisted** by the system under test (e.g. "deve existir na collection 'orders' uma fatura para o cnpj '11122233344556' com os valores"). These assertions go **after** the HTTP request or RabbitMQ await block.

Available helpers:

```java
// Find single document matching filter
Document doc = mongoHelper.findOne("orders", Map.of("cnpj", "11122233344556"));

// Count documents matching filter
long count = mongoHelper.count("orders", Map.of("status", "ACTIVE"));
```

**Primary assertion pattern — `toJson` comparison:**

`jsonMapper` is a `protected static final ObjectMapper` inherited from `IntegrationTestBase` (via `JsonMapper.INSTANCE`) — no import or declaration needed.

Use `mongoHelper.toJson(doc, excludedFields...)` to convert the document to a `JsonNode`, then compare against an expected JSON text block. This mirrors the Ruby table assertion style and handles nested objects naturally.

**Which fields to exclude:** only exclude fields that are **not tested in the original Ruby scenario**. Do not exclude fields just because they are inconvenient — if the scenario asserts a field, include it in the expected JSON. Common fields to exclude are MongoDB metadata (`_id`, `expirationAt`) and any dynamic/generated fields the scenario does not check (e.g. `createdAt`, `updatedAt`).

`excludedFields` accepts dot-notation JsonPath to exclude nested fields too:

```java
// nested object field
mongoHelper.toJson(doc, "_id", "expirationAt", "agreementItem.paymentBill.createdAt")

// field inside array items — use index (0-based)
mongoHelper.toJson(doc, "_id", "items.0.createdAt", "items.1.createdAt")
```

```java
var doc = mongoHelper.findOne("orders", Map.of("cnpj", "11122233344556"));
assertThat(doc).isNotNull();

var expected = jsonMapper.readTree("""
        {
          "cnpj": "11122233344556",
          "status": "PAID",
          "amount": {"$numberDecimal": "199.90"},
          "dueDate": {"$date": "2020-02-01T00:00:00Z"},
          "items": [
            {"description": "linha 1"},
            {"description": "linha 2"}
          ]
        }
        """);
assertThat(mongoHelper.toJson(doc, "_id", "expirationAt")).isEqualTo(expected);
```

**BSON extended JSON types** — the `toJson` output uses BSON relaxed JSON format. Special types appear as:
- `Decimal128` (monetary values, parsed decimals) → `{"$numberDecimal": "485.57"}`
- `Date` (stored dates) → `{"$date": "2019-12-06T00:00:00Z"}`
- `Integer`, `Long`, `String` → plain JSON values

**Assert count:**

```java
assertThat(mongoHelper.count("orders", Map.of("cnpj", "11122233344556"))).isEqualTo(1);
```

Import `import static org.assertj.core.api.Assertions.assertThat;` when using these assertions.

### AssertJ — use the full API

Use AssertJ's `assertThat(...)` for all assertions. The library is rich — always pick the most specific method available for the type being asserted. More specific assertions produce clearer failure messages and communicate intent better than generic `.isEqualTo`.

**Core principle:** never reach for `assertTrue(x.someCheck())` or `.isEqualTo(true/false)` when AssertJ has a dedicated method for the check.

Examples of the breadth available — use whatever fits:

```java
// nullability
assertThat(doc).isNotNull();
assertThat(doc).isNull();

// booleans — never .isEqualTo(true/false)
assertThat(resp.getAuthorized()).isTrue();
assertThat(customer.getHasMobile()).isFalse();

// collections / iterables
assertThat(list).isEmpty();
assertThat(list).isNotEmpty();
assertThat(list).hasSize(3);
assertThat(list).contains("item");
assertThat(list).containsExactly("a", "b", "c");
assertThat(list).containsExactlyInAnyOrder("b", "a", "c");
assertThat(list).doesNotContain("x");
assertThat(list).allMatch(item -> item.startsWith("acme"));

// strings
assertThat(str).isEqualTo("expected");
assertThat(str).contains("substring");
assertThat(str).startsWith("prefix");
assertThat(str).endsWith("suffix");
assertThat(str).matches("regex.*");
assertThat(str).isBlank();
assertThat(str).isNotBlank();

// numbers
assertThat(count).isEqualTo(1);
assertThat(count).isGreaterThan(0);
assertThat(count).isGreaterThanOrEqualTo(1);
assertThat(count).isBetween(1, 10);

// objects with multiple fields — .satisfies groups assertions and reports all failures
assertThat(resp.getCustomers(0)).satisfies(c -> {
    assertThat(c.getUserDocument()).isEqualTo("11122233300");
    assertThat(c.getHasMobile()).isTrue();
    assertThat(c.getIsPrincipal()).isFalse();
});

// extracting field values from a list
assertThat(list).extracting("fieldName").containsExactly("val1", "val2");

// JSON / mongo comparisons (project helpers)
assertThat(mongoHelper.toJson(doc, "_id")).isEqualTo(expectedJsonNode);
assertThat(jsonMapper.readTree(responseBody)).isEqualTo(jsonMapper.readTree(expectedJson));
```

This is not exhaustive — AssertJ has assertions for `Optional`, `Map`, `Path`, `Throwable`, `Date`, and more. Use whatever the library provides that best expresses the intent of the assertion.

**RabbitMQ + MongoDB:** when a scenario publishes to a queue and then asserts mongo state, place the mongo assertions **inside** the `waitUntil` block — the document may not exist yet when the assertion runs:

```java
waitUntil(() -> {
    var doc = mongoHelper.findOne("orders", Map.of("cnpj", CNPJ));
    assertThat(doc).isNotNull();

    var expected = jsonMapper.readTree("""
            {
              "cnpj": "11122233344556",
              "status": "PAID"
            }
            """);
    assertThat(mongoHelper.toJson(doc, "_id", "expirationAt")).isEqualTo(expected);
});
```

## Phase 7: RabbitMQ Tests

Exchange names go in `private static final String` constants at the top of the test class. Derive names from the exchange strings in the Ruby feature steps.

**`waitUntil`** — use the inherited `waitUntil(assertion)` helper instead of writing `Awaitility.await()` directly. Default timeout is 5 seconds, poll interval 500 ms. Only pass explicit values when the scenario genuinely needs more time:

```java
waitUntil(() -> { ... });                          // default: 5s / 500ms
waitUntil(() -> { ... }, 10, 500);                 // custom: 10s / 500ms
```

Do not import `Awaitility` or `Duration` unless `waitUntil` is insufficient for the scenario.

**Template:**

```java
package com.example.<module>.integrationtests.tests;

import com.example.<module>.integrationtests.base.ManagerIntegrationTestBase;
import com.rabbitmq.client.Connection;
import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class <Feature>IntegrationTest extends ManagerIntegrationTestBase {

    private static final String IN_EXCHANGE  = "com.example.in.<topic>";
    private static final String OUT_EXCHANGE = "com.example.out.<topic>";

    @Test
    void <context>_<condition>() throws Exception {
        List<Map<String, Object>> receivedMessages = new ArrayList<>();

        try (Connection ignored = rabbitMQHelper.startConsuming(OUT_EXCHANGE, "", receivedMessages)) {
            rabbitMQHelper.publishToExchange(IN_EXCHANGE, jsonToMap("""
                    {"key": "value"}
                    """));

            waitUntil(() -> {
                assertThat(receivedMessages).isNotEmpty();

                var expected = jsonMapper.readTree("""
                        {
                          "status": "EXPECTED_VALUE",
                          "nested": {
                            "field": "value"
                          }
                        }
                        """);
                assertThat(rabbitMQHelper.toJson(receivedMessages.get(0))).isEqualTo(expected);
            });
        }
    }
}
```

**`toJson` for message assertions** — use `rabbitMQHelper.toJson(message, excludedFields...)` to convert the received message map to a `JsonNode`, then compare against an expected JSON text block. The comparison is order-independent.

Only exclude fields not tested in the original Ruby scenario. Accepts dot-notation JsonPath for nested exclusions (e.g. `"header.sentAt"`). To exclude fields inside list items, use index notation (e.g. `"businessContactList.0.expiration_at"`, `"businessContactList.1.expiration_at"`). Pass broker metadata fields that the scenario never asserts as exclusions.

For message payloads published to the queue, apply the same rules as Phase 5: `jsonToMap` for payloads used in only 1 test method, Doc builder for payloads reused in > 1 test method. The Doc's `build()` returns `Map<String, Object>` passed directly to `publishToExchange`.
