# tenant-notebook — run book

Entry B for the LOL Ventures / Terminal3 "Create Agent ID, claim free tokens,
deploy first RUST contract" bounty. Capability surface: `kv-store` only
(tenant-scoped save/get) — no HTTP at all, disjoint from Entry A on purpose.

## 0. Claim identity + credits (manual, browser)

Use a SEPARATE, genuinely distinct identity from Entry A.

1. Go to https://www.terminal3.io/claim-page and sign in with this entry's
   own work email.
2. Copy the developer key immediately — it is shown exactly once.
3. Set it locally (do not paste it into chat / commit it):
   ```
   export T3N_API_KEY="0x<key>"
   ```

## 1. Build the contract (no credentials needed)

```
cargo build --target wasm32-wasip2 --release
cargo test
wasm-tools component wit target/wasm32-wasip2/release/tenant_notebook.wasm
```

## 2. Register a public Agent ID (CLI, needs T3N_API_KEY)

```
npx @terminal3/t3n-sdk --help
t3n whoami --env testnet
export AGENT_DID=$(t3n whoami --env testnet)
t3n agent create-card --did "$AGENT_DID" \
  --name "tenant-notebook agent" \
  --description "Reads/writes a private per-tenant KV note via a T3N TEE contract" \
  --force
t3n agent host-card --file agent-card.json --env testnet
curl https://<node>/api/agent-card/"$AGENT_DID"
```

## 3. Register + invoke the contract (needs T3N_API_KEY)

```
cd scripts
npm install
npx tsx quickstart.ts
```

Expect, in order: `Connected as: did:t3n:...`, `TenantClient ready.`,
`registered z:<tid>:tenant-notebook as contract id <n>`, `notebook map
ready`, the self-grant confirmation, then `save-note result` followed by
`get-note result` (should echo the saved value back).

Ran into a few SDK/docs mismatches along the way (missing `trustAnchor`,
`tenant.me()` vs `tenant.tenant.me()`, a `.claim()` 500 reproduced on this
identity too). Full writeup with real output and request IDs: see
[README.md](README.md).
