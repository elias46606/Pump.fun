# backend: opt-in local ban check for `GET /moderation/ban/address/{address}`

## Problem

`moderation.read` is the only scope with a single operation and it has no native
handler: `GET /v1/moderation/ban/address/{address}` always falls through to the
generic replay and returns `501 COMPATIBILITY_NOT_IMPLEMENTED`. PROGRESS.md lists
`moderation.read` 0/1.

## Requirement served

PRD §1: "Replace the website backend with independently implemented behavior
compatible with every agreed in-scope API operation and state." The operation is
one of the 67 documented developer operations in `contracts/developer.openapi.json`.

## Implementation

Built **only from the public OpenAPI 0.294.0 contract** (SHA-256 `ad2fdc2b…73ad73`).
No captured website code, factory or upstream response is used.

- `src/moderation-store.ts`: `ModerationStore` over an explicitly supplied ban
  list (`schemaVersion`, `dataSource`, `observedAt`, `bans[{address, expiresAt}]`).
  `expiresAt: null` = permanent; a timestamp is active strictly before that instant
  and is evaluated per request through an injectable clock. Addresses match
  verbatim (no case folding, trimming or chain inference). Strict validation
  rejects unknown keys, duplicates, sparse arrays and non-canonical timestamps.
  `loadModerationStore` is a bounded (8 MiB), symlink-refusing, fatal-UTF-8
  startup read like the payouts loader.
- `src/moderation-routes.ts`: registered inside the existing `apiKeyGate`; returns
  the bare JSON boolean the schema declares, with `Cache-Control: no-store` and
  the usual `X-Reconstruction-*` headers (`compatibility: unqualified`). Any query
  string returns local `501 MODERATION_QUERY_UNSUPPORTED` since none is declared.
- `app.ts` / `server.ts` / `.env.example`: `moderationStore` option and
  `MODERATION_BANS_FILE`. **Absent store = unchanged fixture replay / 501.**
- `MODERATION.md`: contract, data model and explicit local boundaries.

Why this approach: it mirrors the existing opt-in stores (payouts, user trades,
positions), so the default harness and all fixture behavior stay byte-identical,
and nothing pretends to be a production moderation verdict. Alternatives
considered: a default synthetic seed (rejected: would change default behavior
and fixture tests) and returning `false` without a store (rejected: would
fabricate a verdict).

## Evidence

Before (base `381dc30`):

```
GET /v1/moderation/ban/address/SyntheticBannedWallet111
501 {"error":"COMPATIBILITY_NOT_IMPLEMENTED","method":"GET","path":"/v1/moderation/ban/address/SyntheticBannedWallet111"}
```

After (real server, `MODERATION_BANS_FILE` with synthetic list, `curl`):

```
SyntheticPermanentBan…  -> 200 application/json  true
SyntheticExpiredBan…    -> 200 application/json  false
SyntheticNeverBanned…   -> 200 application/json  false
(no API key)            -> 401
headers: cache-control: no-store, x-reconstruction-data-source: synthetic,
         x-reconstruction-compatibility: unqualified
```

Tests (Node 22.22.0, pnpm 10.24.0):

- `pnpm --filter @pump-reconstruction/backend test`: base 964/964, after 972/972
  (8 new in `test/moderation.test.ts`; responses validated against the OpenAPI
  200 schema with the existing `assertSchema`).
- Mutation control: making expiry inclusive (`<=`) fails 2 of the new tests.
- `pnpm --filter @pump-reconstruction/backend typecheck`: clean.
- `pnpm check:progress`: OK (no ledger or percentage changes claimed).

## Test steps

```bash
cd app && pnpm install --frozen-lockfile
pnpm --filter @pump-reconstruction/backend test
pnpm --filter @pump-reconstruction/backend typecheck
pnpm check:progress
```

## Limits

- Not a native-compatibility qualification: no differential test against the
  live API, so `moderation.read` stays 0/1 qualified and no feature credit is
  added.
- No ban issuing/lifting, reasons, audit history or rate-limit headers.
- All addresses in tests and docs are synthetic.

<!-- Footer: run locally and paste unchanged:
node <skill-directory>/scripts/run-receipt.mjs disclose \
  --provider Anthropic --model claude-opus-5-5 --client "Claude Code (claude.ai/code cloud session)"
-->
