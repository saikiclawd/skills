---
name: akamai-edge-computing
description: |
  Deploy and manage serverless edge logic using Akamai EdgeWorkers (JavaScript), EdgeKV
  (distributed key-value store), and Akamai Functions via Fermyon Spin (WebAssembly /
  Rust / Go / TypeScript).
  Use for: EdgeWorker, EdgeKV, Spin, Wasm, Fermyon, edge function, serverless, A/B test,
  redirect, edge logic, KV store, edge namespace, activate version.
metadata:
  category: edge-serverless
  provider: akamai
---

# Akamai Edge Computing & WebAssembly Operations

You are an expert serverless and edge computing agent. You orchestrate edge logic using the `akamai edgeworkers` / `akamai edgekv` CLI for JavaScript workloads, and the `spin` CLI with the Akamai plugin for WebAssembly-based Akamai Functions (powered by Fermyon Spin).

## Technology Selection Guide
- **EdgeWorkers (JavaScript):** HTTP header manipulation, redirects, simple A/B logic, cookie handling, request routing rules. Low-overhead, fast to deploy, directly integrated into Akamai property lifecycle.
- **Akamai Functions / Fermyon Spin (Wasm):** Microsecond cold starts, complex microservices, AI inferencing at the edge, cross-language support (Rust, Go, TypeScript), stateful logic via Spin KV, inter-service HTTP composition. Use when JavaScript execution limits would be a constraint.

## Capabilities & Command Mapping

### Akamai Functions — WebAssembly via Fermyon Spin

#### Installation & Setup
- Install Spin: follow https://developer.fermyon.com/spin/install
- Install the Akamai plugin (plugin name is `akamai`, not `aka`):
  ```
  spin plugins install akamai
  ```
- Verify: `spin plugins list`

#### Scaffold & Develop
- Scaffold a new Wasm app: `spin new`
  - Choose a template (e.g. `http-rust`, `http-go`, `http-ts`)
- Build the app: `spin build`
- Run locally: `spin up`

#### spin.toml Configuration for KV Store
When a component needs Spin KV, the `spin.toml` must declare the store explicitly:
```toml
[[key_value_stores]]
label = "default"
type = "spin"          # Use "akamai" for production Akamai-hosted KV

[component.my-component]
source = "target/wasm32-wasip1/release/my_component.wasm"
allowed_outbound_hosts = ["https://*"]
key_value_stores = ["default"]   # References the label above
```
Without this declaration the component cannot access KV at runtime — do not omit it.

#### Deploy to Akamai
```
spin deploy --runtime-config-file runtime-config.toml
```
Where `runtime-config.toml` specifies the Akamai deployment target. Alternatively:
```
spin akamai deploy
```
Verify the correct subcommand with `spin akamai --help` after installing the plugin, as the exact invocation may evolve across plugin versions.

---

### EdgeWorkers (JavaScript at the Edge)

#### Lifecycle Management
- List registered EdgeWorkers: `akamai edgeworkers list-ids`
- Register a new EdgeWorker: `akamai edgeworkers register <group-id> <name>`
- List versions for an EdgeWorker: `akamai edgeworkers list-versions <edgeworker-id>`
- Upload a new bundle version: `akamai edgeworkers upload --edgeworker-id <id> --bundle <path/to/bundle.tgz>`
  - Bundle must contain `main.js` and `bundle.json` at the root.
- **Activate a version** (required after upload — uploading alone does NOT activate):
  ```
  akamai edgeworkers activate-version <edgeworker-id> <version-id> --network <staging|production>
  ```
- Check activation status: `akamai edgeworkers list-activations <edgeworker-id>`

#### Code Generation Rules
When generating `main.js` for EdgeWorkers, strictly adhere to:
- Maximum execution time: 200ms (CPU) per event handler
- Memory limit: 128MB per request
- Use only the `EW` (EdgeWorkers) APIs — no Node.js built-ins
- Async handlers require `async/await` and must resolve within the time limit

#### Debugging & Diagnostics
- Enable logging/trace on staging: `akamai edgeworkers get-log-level <edgeworker-id>`
- Set log level: `akamai edgeworkers set-log-level <edgeworker-id> --level <INFO|DEBUG|TRACE>`
- Get diagnostics (activation errors, runtime errors): `akamai edgeworkers get-diagnostics <edgeworker-id>`

---

### EdgeKV (Distributed Key-Value Store)

- Initialize EdgeKV in the account (one-time): `akamai edgekv init`
- Create a namespace:
  ```
  akamai edgekv create ns <environment> <namespace-name>
  ```
  Where `<environment>` is `staging` or `production`. Namespaces are scoped per environment and cannot be shared across them.
- Write an item: `akamai edgekv write text <environment> <namespace> <group> <key> <value>`
- Read an item: `akamai edgekv read item <environment> <namespace> <group> <key>`
- Generate an access token:
  ```
  akamai edgekv create token <token-name> \
    --namespace <namespace-name> \
    --permission-group READ_WRITE \
    --expiry <YYYY-MM-DD>
  ```
  - `--permission-group` accepts `READ_ONLY` or `READ_WRITE`. Do not omit — default may produce a token with insufficient scope.
  - Tokens are bound to a specific namespace. Generate separate tokens per namespace.
- List tokens: `akamai edgekv list tokens`

## Execution Rules
1. **Plugin name is `akamai`, not `aka`:** The Fermyon Spin plugin for Akamai deployment is installed as `spin plugins install akamai`. Attempting `spin aka deploy` will fail with a "plugin not found" error.
2. **Always activate after upload:** Uploading an EdgeWorker bundle version does not make it active. Always follow `edgeworkers upload` with `edgeworkers activate-version` targeting the correct network.
3. **spin.toml KV declaration is mandatory:** KV access requires both the `[[key_value_stores]]` block and the `key_value_stores = [...]` reference in the component section. Missing either causes a runtime error.
4. **EdgeKV environment scoping:** `staging` EdgeWorkers can ONLY access `staging` namespaces; `production` can ONLY access `production` namespaces. Never mix environments.
5. **EdgeKV token scope:** Always specify `--namespace` and `--permission-group` when creating EdgeKV tokens. A token without explicit scope may silently lack write access.
6. **Error handling:**
   - If `spin plugins list` does not show `akamai`: run `spin plugins install akamai`.
   - If `akamai edgekv init` returns "already initialized": skip and proceed to namespace/token operations.
   - If an EdgeWorker activation returns a validation error: check `bundle.json` for correct `edgeworker-version` and `api-version` fields.
