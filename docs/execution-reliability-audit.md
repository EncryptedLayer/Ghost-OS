# Ghost OS Execution Reliability Audit

Date: 2026-06-22

This audit documents the current intent-to-finality path and the hardening added in this pass. It is intentionally production-minded: Ghost OS must not claim a crypto action worked unless a real provider built/sent it and a real hash or signature exists.

## Current Architecture

Primary chat route:
- `app/api/ghost/chat/route.ts` receives natural-language commands.
- `lib/ghost-intent/parser.ts` creates the first structured intent and session clarification memory.
- `lib/ghost-brain-v2/index.ts` validates intent and builds the activity lifecycle.
- Concrete wallet commands then route to:
  - `lib/trading/trade-controller.ts` for swaps.
  - `lib/trading/bridge-controller.ts` for bridges.
  - `lib/wallets/embedded/executeGhostTransfer.ts` for transfers.

Manual/prepared execution routes:
- `app/api/swap/prepare/route.ts` builds EVM 0x or Solana Jupiter prepared swap payloads.
- `app/api/swap/execute/route.ts` tracks already-submitted EVM hashes or executes signed Jupiter transactions.
- `app/api/ghost/command/quote/route.ts` and `app/api/ghost/command/execute/route.ts` use the older Ghost command parser and execution engine.

Scanner and history:
- `lib/security/scanner.ts` uses Dexscreener as the no-key base scanner and marks GoPlus/Honeypot as provider-required when keys are absent.
- `lib/ghost-execution/tx-history.ts` writes to Prisma when available and falls back to local storage if Prisma is unavailable.

## Failure Classes Found

1. Capabilities were implicit.
   Swap, bridge, transfer, scan, and portfolio behavior depended on whichever route the command reached. There was no central source of truth for whether a capability was supported, degraded, disabled, or unsupported.

2. Provider readiness was discovered too late.
   Bridge and swap commands could reach runtime execution before Ghost OS clearly checked whether LI.FI, 0x, Jupiter, or chain RPC was configured for that requested chain.

3. Public errors were too generic or too internal.
   Some failures returned raw backend codes or generic messages. Users need to know stage, cause category, retry safety, and the safest next action without seeing secret provider details.

4. Prisma failures were partially handled.
   Transaction history has a local fallback, and Prisma is temporarily cooled down after failures. This is good and should be preserved, but database schema errors still need migrations before full persistent history is guaranteed.

5. Intent and execution paths are duplicated.
   Brain V2, deterministic Ghost Brain, trading parser, manual swap routes, and command execution routes all parse overlapping concepts. The new capability registry gives these routes a shared contract, but future cleanup should converge them further.

## Hardened In This Pass

Added a central typed capability registry:
- File: `lib/capabilities/registry.ts`
- It defines capability IDs, chain support, required providers, permissions, inputs, wallet/simulation/approval requirements, testnet/cancellation support, and status.
- It maps Brain V2 intents and legacy Ghost intents to capability IDs.
- It evaluates provider readiness for critical write capabilities before execution:
  - `dex.swap`
  - `bridge.transfer`
  - `token.transfer`

Added standardized Ghost errors:
- File: `lib/server/ghostError.ts`
- It normalizes errors into:
  - stable `errorCode`
  - execution `stage`
  - failure `category`
  - safe public message
  - retryability
  - suggested action
  - evidence and correlation ID

Updated shared API error responses:
- File: `lib/server/response.ts`
- `fail()` now includes a `ghostError` payload and uses sanitized public messages.

Updated chat execution routing:
- File: `app/api/ghost/chat/route.ts`
- Concrete write commands now check capability readiness before execution.
- Execution failures are caught and returned as clean Ghost chat output instead of raw backend failure cards.
- Brain training cases are logged when a capability is missing or execution fails.

Added regression tests:
- File: `tests/ghost-capabilities.test.mjs`
- Covers capability mapping, bridge readiness, unsupported capability behavior, buy-contract parsing, route metadata, and GhostError sanitization.

## Capability Snapshot

Available or degraded read/analyze capabilities:
- `wallet.scan`
- `token.scan`
- `contract.scan`
- `transaction.scan`
- `portfolio.read`
- `balance.read`
- `market.read`

Write capabilities gated by provider readiness:
- `dex.swap`: requires 0x for EVM swaps and Jupiter for Solana swaps.
- `bridge.transfer`: requires LI.FI.
- `token.transfer`: requires configured chain RPC and Ghost Wallet signer.
- `nft.trade`: requires a configured NFT marketplace order builder, marketplace data provider, wallet signer, RPC, and fee configuration.

NFT execution notes:
- Base NFT actions should resolve intent first, then use OpenSea marketplace data, OpenSea SDK/API order preparation, or a Base MCP OpenSea plugin where configured.
- Base MCP transaction preparation can shape the unsigned transaction or signature payload before wallet-mode routing.
- Ghost Wallet Mode may sign and submit supported NFT approval, buy, list, cancel-listing, accept-offer, and sweep actions only after ownership, balance, price-limit, policy, route, and risk checks pass.
- External Wallet Mode must prepare an unsigned transaction or signature request and wait for the connected wallet. Ghost OS must never sign external-wallet payloads server-side.
- Missing marketplace, Base MCP, order-builder, RPC, signer, or fee configuration must return a clean not-configured state. Ghost OS must not show fake NFT hashes, listings, fills, or confirmations.

Currently disabled/unsupported unless providers are wired:
- `approval.scan`
- `approval.revoke`
- `token.approve`
- `token.deploy`
- `staking.*`
- `lending.*`
- `contract.call`
- `liquidity.add`

## Remaining Production Work

1. Persist execution state machine rows for every write command:
   `RECEIVED -> INTENT_RESOLVED -> VALIDATED -> PREFLIGHT_RUNNING -> QUOTED -> BUILT -> SIMULATED -> RISK_REVIEWED -> AWAITING_APPROVAL -> APPROVED -> SIGNATURE_REQUESTED -> SIGNED -> SUBMITTED -> PENDING -> CONFIRMED -> FINALIZED`.

2. Add per-provider health checks, timeout/retry policies, circuit breakers, and request correlation IDs across 0x, LI.FI, Jupiter, RPC, CoinGecko, and Dexscreener.

3. Unify the older Ghost command parser, Brain V2 intent model, and trading parser so every route consumes the same strict intent schema.

4. Add explicit pre-flight persistence for balance, allowance, quote freshness, gas reserve, simulation result, slippage, price impact, route expiration, and policy result.

5. Add destination-chain finality tracking for bridges. Source-chain submission must not be treated as bridge completion.

6. Run Prisma migrations in the target environment so persistent transaction history and brain training tables are available. The local fallback prevents crashes but is not a replacement for production persistence.

7. Expand the test matrix with provider-mocked execution for swap, bridge, transfer, scan, failure diagnosis, duplicate broadcast prevention, and safe retry.
