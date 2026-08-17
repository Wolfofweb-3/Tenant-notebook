# Tenant-notebook 

Agent ID, test credits, and a deployed Rust contract on Terminal3's T3N testnet.

## What I built

`tenant-notebook` is a three-function TEE contract — `save-note`, `get-note`,
and `list-notes` — that reads and writes a private per-tenant KV map. It
never imports `http` at all — no outbound network capability, nothing
leaves the enclave. The point was to test the other half of the ADK's
capability model: state instead of networking, and the map-creation/ACL
flow that gates it, rather than the HTTP path most examples default to.

Source, WIT interface, and the registration script are in this repo.

## Use case

Any agent that persists something across calls — a running conversation
summary, a user's stated preferences, a partial task/workflow state — needs
somewhere to put it that isn't a plaintext file the host process (or anyone
with host access) can read. A tenant-scoped, ACL-restricted KV map inside
the enclave gives an agent private working memory: only the contract's own
code, running inside the TEE, can read the map back, and each tenant's
memory is walled off from every other tenant's by construction.

`list-notes` is what turns that from "a lookup table" into actual memory: an
agent doesn't always know the key it's looking for in advance — it needs to
enumerate what it already knows. This is the minimal version of durable,
private, *browsable* agent state — the same shape scales to session memory,
cached credentials, or draft outputs an agent isn't ready to expose yet.

## Identity + credits

Claimed a dev key and test credits from the [claim page](https://www.terminal3.io/claim-page),
authenticated, confirmed tenant status:

```
Connected as: did:t3n:f924769a5dc542faae536b178dd6f8afe9bb5e73
TenantClient ready: {
  tenant: 'did:t3n:f924769a5dc542faae536b178dd6f8afe9bb5e73',
  label: 'testnet-dev',
  status: 'active',
  quotas: { max_contracts: 10, max_maps: 50, ... },
  created_at: 1786538683
}
```

Registered a public Agent ID (ERC-8004 card, hosted directly on the T3N node):

```
$ t3n agent host-card --file agent-card.json --env testnet
agent card hosted on T3N: https://cn-api.sg.testnet.t3n.terminal3.io/api/agent-card/did:t3n:f924769a5dc542faae536b178dd6f8afe9bb5e73
```

## Deploying, provisioning, and invoking the contract

```
$ cargo build --target wasm32-wasip2 --release
$ cargo test          # 7 unit tests, native target
```

Registration, ACL update, and all three functions in one session. This is
the second deploy — the first, `save-note`/`get-note`-only version
registered as contract id 631; adding `list-notes` and redeploying bumped
it to 634:

```
registered z:f924769a5dc542faae536b178dd6f8afe9bb5e73:tenant-notebook as contract id 634
notebook map already existed — updated ACL to current contract id 634
self-grant authorized: save-note/get-note
save-note result: { key: 'hello', stored: true, saved_at_secs: 1786542621 }
get-note result: { key: 'hello', value: 'first T3N tenant-notebook write', found: true }
save-note (second key) result: { key: 'second-key', stored: true, saved_at_secs: 1786547840 }
list-notes result: {
  notes: [
    { key: 'hello', value: 'first T3N tenant-notebook write' },
    { key: 'second-key', value: 'list-notes proves this is real memory, not a lookup table' }
  ]
}
```

Two things worth pointing at directly. First, `hello` — written by the
*first* deploy, contract id 631 — is still there and still readable after
the contract was redeployed as a new id: the KV map is tenant-owned, not
contract-owned, so re-registering the code doesn't wipe the data. Second,
`list-notes` genuinely enumerates: it returns both notes, not just the one
that was looked up by key, and the second one was only written moments
before the list call, in the same run.

## Findings

**1. `tenant.tenant.claim()` (testnet self-admit) reliably 500s** for a
tenant already provisioned through the web claim page:
```
RpcError: RPC Error: Internal error [8bf34d5f-5fce-4675-925d-bb32af749974]
code: 'RPC_ERROR', rpcMethod: 'action.execute', httpStatus: -32603
```
Calling `.tenant.me()` directly instead works fine and reports
`status: 'active'` with credits already present, so the two provisioning
paths (web claim, SDK self-admit) look like they're stepping on each other
rather than composing.

**2. Redeploying silently orphans your map ACL.** Every re-registration of
a tail mints a brand-new `contract_id` (631 → 634 here, across just two
versions). A KV map's `readers`/`writers` are scoped to a specific
`contract_id`, so after a redeploy the *new* contract is not authorized
against the *old* ACL — the very next `get-note`/`list-notes` would fail
`AccessDenied` unless the map is explicitly re-pointed. Worse: `docs`
describe `"map already exists"` from `maps.create()` as idempotent and
"safe to ignore," but the SDK throws an `RpcError` rather than no-op'ing —
so the naive fix (catch and ignore) leaves the ACL stale. The actual fix is
to catch that specific error and call `maps.update()` with the current
`contract_id`:
```
RpcError: RPC Error: map already exists [ca631273-a3b0-452c-b14c-b2d8edb6d839]
code: 'RPC_ERROR', rpcMethod: 'action.execute', httpStatus: -32602
```

**3. The published SDK doesn't match its own docs at the constructor
level.** `@terminal3/t3n-sdk@4.36.0`'s `T3nClient` constructor throws
`T3nConfigError` if you follow the Quickstart page literally — `trustAnchor`
(from `fetchTrustedManifest(env)`) is now required and missing from the
docs' code sample. The error message itself is well-designed (names the
field, explains both valid shapes, warns against the unsafe bypass), so
this reads as documentation drift, not a rough API. Likewise `TenantClient`
has no `.me()` — it's `tenant.tenant.me()`, only discoverable from the
package's own `dist/index.d.ts`.

**4. `.cargo/config.toml` in the reference contract repo forces
`wasm32-wasip2` as the default build target project-wide**, breaking plain
`cargo test` (tries to execute the compiled `.wasm` as a native binary —
`os error 193` on Windows). Fix: leave the target unset in config.toml,
pass `--target wasm32-wasip2` explicitly only for the release build.

**5. The npm package ships obfuscated, and its listed source repo is
dead.** Both `dist/index.esm.js` and `dist/index.js` are ~1.2MB of
hex-identifier, string-array-encoded output — not ordinary minification.
`package.json` claims MIT and points at `github.com/Terminal-3/trinity`
(`client/t3n-sdk`), which doesn't resolve on GitHub. Likely not
adversarial — the dependency list (`@noble/curves`, `ethers`, Bytecode
Alliance's own `jco`/`preview2-shim`) and the TDX-attestation/manifest-
signature verification code in the type definitions go well past what a
throwaway credential-harvesting package would invest in — but it does mean
nobody outside the team can currently audit the exact code that touches a
raw private key.

## Suggestions

- Reconcile `.claim()` with tenants that already have credits from the web
  flow — either fix the server error or document when to skip it.
- Auto-migrate (or at minimum warn about) stale map ACLs on redeploy, and
  make `maps.create()` actually idempotent per its own docstring.
- Sync the Quickstart / Set Up Dev Env pages with the current
  `T3nClientConfig` shape and real `TenantClient` method names.
- Publish a source map or unminified build target, and fix the dead
  repository link — the "Create Tenant KV Maps" doc was accurate and saved
  real time; the rest of the SDK-facing docs would benefit from the same
  level of care.
