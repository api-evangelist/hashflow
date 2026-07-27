---
name: Get an RFQ quote and execute a swap on Hashflow
description: Discover market makers, request a signed request-for-quote (RFQ), and settle the trade on-chain via the HashflowRouter using the Taker API v3.
api: https://docs.hashflow.com/hashflow/taker/getting-started-api-v3
base_url: https://api.hashflow.com/taker/v3
operations:
  - "GET /market-makers"
  - "GET /price-levels"
  - "POST /rfq"
  - "GET /restrictions"
method: generated
source: https://docs.hashflow.com/hashflow/taker/getting-started-api-v3
---

# Get an RFQ quote and swap on Hashflow

Hashflow is an RFQ DEX: instead of an AMM curve, professional market makers
return **signed** quotes that settle on-chain with zero slippage. Follow these
steps to go from price discovery to execution.

## Authentication
Send your provider-issued credential key in the `Authorization` header on every
request. Include a `source` string identifying your integration; single-wallet
integrators use `source: "api"` plus a `wallet` parameter. See
`authentication/hashflow-authentication.yml`.

## Steps

1. **List market makers** — `GET /market-makers` with `source`, `baseChainType`
   ("evm" | "solana"), and `baseChainId` (e.g. "1" for Ethereum). Returns
   `{ marketMakers: string[] }`.

2. **Discover prices** — `GET /price-levels` with `source`, chain params, the
   `marketMakers[]` from step 1, and optionally `baseToken`/`quoteToken`
   addresses (native token = `0x0000000000000000000000000000000000000000`). This
   can be polled every ~1s and is cache-friendly. Quantities/prices are in whole
   units, not wei.

3. **Request a signed quote** — `POST /rfq` with a `rfqs[]` array specifying
   `baseToken`, `quoteToken`, one of `baseTokenAmount`/`quoteTokenAmount`, and
   `trader`. Set `options.doNotRetryWithOtherMakers: false` to allow fallback
   routing. The response returns `quotes[]`, each with `quoteData` (including
   `nonce`, `quoteExpiry`, `pool`, `txid`), a `signature`, and optionally
   `calldata`/`targetContract`/`value` for execution.

4. **Execute on-chain before `quoteExpiry`** — call `tradeRFQT` (single-chain) or
   `tradeXChainRFQT` (cross-chain) on the `HashflowRouter` contract
   (github.com/hashflownetwork/x-protocol) with the signed `quoteData`. On Solana,
   call the `trade` instruction. Send `value` when the base token is native.

## Error & rate-limit handling
- Responses carry `status: "success" | "fail"`; on `fail`, read the `error`
  string (see `errors/hashflow-error-codes.yml`).
- If quotes are refused, check `GET /restrictions` — a trader can be rate-limited
  (`isTraderRestricted: true`, reason `rate_limit`, with `expiryTimestampMs`).
- Quotes are single-use: each carries a `nonce` and `quoteExpiry`; do not reuse.
