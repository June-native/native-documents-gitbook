---
description: The /trade error model — read the response body, not the HTTP status.
---

# Error responses

`POST /trade` is **synchronous** and always returns the **same JSON trade-response shape**, whatever the HTTP status (`200`, `400`, `429`, `503`, `504`). Branch on the body, not the status line. The envelope carries `submission_status`, and any non-successful outcome carries an `error.code`. A business rejection (below minimum, bad precision, insufficient balance) is **data**, not a transport error: read `error.code` and act on it.

Only the transport layer raises — a non-trade-response `4xx`/`5xx` body, or a wire failure before any response arrived. A decodable trade response is always returned to you as-is.

## submission_status

The call blocks for the on-chain outcome and returns it directly.

There are exactly three values.

| `submission_status` | When | Do next |
| --- | --- | --- |
| `accepted` | The transaction landed and executed — for an order that covers rested, filled, or a benign IOC/FOK/self-trade/no-liquidity cancel; for a non-order action it committed. No `error`. | Done. The `/trade` response does not carry the fill state — read [`orderStatus`](post-info.md#orderstatus) to see whether an order rested or filled. |
| `rejected` | The write was refused — request-shaping, gateway (rate limit / suspension / expiry), node admission — **or** it failed at execution. `error.code` says why; `tx_hash` is present once canonical bytes exist. | If `RateLimited`, back off `error.retry_after_ms` and resend the same signed action. Otherwise fix the cause and submit a **fresh** action. |
| `timeout` | The outcome wasn't observed within the wait budget, or the submission couldn't be routed to the active node. **May still land.** | Reconcile by `cloid`. **Never** resubmit under a new nonce — double-fill risk. |

{% hint style="warning" %}
`timeout` is not `rejected` — the transaction may still commit in a later block. Reconcile by `cloid` (via [`orderStatus`](post-info.md#orderstatus) / [`txStatusByCloid`](post-info.md#orderstatus)); resubmitting under a new nonce is the one move that can double-fill you.
{% endhint %}

A `timeout` has two shapes: the wait budget elapsed → HTTP `200` with **no** `error`; the submission couldn't be routed → HTTP `503`/`504` with an `error.code` (see the gateway codes below).

## Where a code comes from

Every `error.code` comes from one of four layers, and the **spelling tells you which**:

| Layer | Style | Examples | HTTP | `submission_status` |
| --- | --- | --- | --- | --- |
| Request-shaping | lowercase `snake_case` | `invalid_json`, `invalid_quantity_precision`, `missing_cloid` | 400 | `rejected` (no `tx_hash`) |
| Gateway | `CamelCase` / prefixed | `RateLimited`, `PlaceOrderSuspended`, `ExpiredTx`, `HandoffTimeout`, `node_unreachable: …` | 429 / 503 / 504 / 200 | `rejected` or `timeout` |
| Node admission | `CamelCase`, verbatim from the node | `MinTradeSpotNtl`, `DuplicateCloid`, `InsufficientSpotBalance`, `AccountFrozen` | 200 | `rejected` |
| Execution | lowercase (the variant name) | `tick`, `badnonce`, `insufficientspotbalance` | 200 | `rejected` |

One condition can surface at two layers with different spellings — an under-minimum order is usually caught at admission as `MinTradeSpotNtl`, but the same failure at execution reads `mintradespotntl`. Match on the code you actually receive.

## Errors you'll actually hit

The wire carries `error.code`, not display copy — the **Message** column is illustrative UI text you'd render from the code. This is the curated subset you meet in practice; the [full catalog](#full-trade-error-code-reference) is at the bottom of this page.

| Code | What it means | Message a user sees | Fix |
| --- | --- | --- | --- |
| `RateLimited` | Per-signer request rate exceeded (50 req/s per signer); never admitted. HTTP `429`, carries `error.retry_after_ms`. | "Too many requests — retrying shortly." | Back off `retry_after_ms`, then resend the same signed action. The only safe resend. |
| `PlaceOrderSuspended` | The write path is degraded, so **place-order** actions (`order` / `modify` / a `batch` with one) are suspended; `cancel` / `cancelAll` still go through. HTTP `503`, `error.retry_after_ms: 1000`. | "Placing orders is paused — try again shortly." | Back off and retry; keep cancelling if you need to reduce exposure. |
| `HandoffTimeout` / `HandoffBufferFull:{request_count\|bytes\|signer}` / `HandoffMultipleActive` (`timeout`) | The submission couldn't be routed to a single writable node (leadership handoff / backpressure). `submission_status: "timeout"`, HTTP `503`, `error.retry_after_ms: 1000`. The action may still land. | "Order submitted — confirming status." | Reconcile by `cloid`; never resubmit under a new nonce. |
| `node_unreachable: …` (`timeout`) | Node admission couldn't be reached; `submission_status: "timeout"`, HTTP `504`. The action may still land. | "Order submitted — confirming status." | Reconcile by `cloid`; never resubmit under a new nonce. |
| `MinTradeSpotNtl` | Order, modify replacement, or batch item is below the market's quote-asset minimum notional. Market orders use their protection price. | "Order must have a minimum value of 10 USDC." | Size up so `price × quantity` clears the minimum. |
| `invalid_price_precision` / `invalid_quantity_precision` | `price` / `quantity` has more fractional digits than the market's `price_decimals` / `base_quantity_decimals`. | "Price has too many decimal places for this market." | Snap to market precision before signing; send strings, never floats. |
| `DuplicateCloid` | The `cloid` is already open for this owner and market, or the tx repeats a `(market_id, cloid)`. | "An order with this ID already exists." | Use a fresh `cloid` per order. When reconciling a `timeout`, look the existing one up — don't resend. |
| `ExpiredTx` | The signed `expires_after_ms` had already passed when execution reached the tx. | "Order expired before it was placed." | Widen `expires_after_ms`, re-sign, resubmit. |
| `InsufficientSpotBalance` | Balance-mode precheck found too little available balance for the order reserve. | "Insufficient balance." | Fund the account's quote asset — deposit from your main wallet (no faucet; bring Arbitrum Sepolia testnet assets). |
| `DirectSignerIsActiveAgent` | An active API-wallet key signed in direct-owner mode. An agent key may never sign as an owner. | "This API wallet can't trade as an owner." | Sign in agent mode — pass the owner `accountAddress` alongside the agent key. |
| `AgentEpochMismatch` | The signed `agent_epoch` doesn't match the live agent-slot epoch. | "Session expired — reconnecting." | Re-resolve `agent_epoch` from [`userAgents`](post-info.md#useragents) and resubmit. If it persists, the API wallet was revoked or re-approved — create a new one in the web app. |
| `AccountFrozen` | The account was frozen by an operator; while frozen, only `cancel` / `cancelAll` are admitted — new orders and modifies are rejected. | "Your account is frozen — trading is paused." | Stop placing orders until the freeze is lifted; cancels still go through. |

## Batch

A [`batch`](post-trade.md#batch) is one `/trade` call under one envelope nonce, so the envelope gets **one** `submission_status`. That status reflects the batch's admission and overall outcome — it does **not** report each item. Items execute in array order, and **each item may individually succeed or fail** inside the batch execution result. Reconcile **per item** by `cloid` via [`batchOrderStatus`](post-info.md#batchorderstatus) (up to 20 lookups per call, which covers a full `1..=10`-item batch). Don't infer item outcomes from the envelope status.

## Execution-level failures

An admitted action still runs against the book and **can fail at execution**. Because `/trade` is synchronous, that failure comes back **on the `/trade` response itself** — `submission_status: "rejected"` with a lowercase `error.code`, for both order and non-order actions. The same outcome is also readable afterward via [`orderStatus`](post-info.md#orderstatus) by `cloid`, which you only need when the write came back `timeout`.

Execution codes are the lowercase variant names — e.g. `tick`, `badnonce`, `insufficientspotbalance` — distinct from the CamelCase admission codes above.

| Code | What it means | Fix |
| --- | --- | --- |
| `tick` | A non-integer `price` exceeded the market's `max_price_sig_figs`. The order was admitted, then rejected at execution — `/trade` returns `submission_status: "rejected"` with `error.code: "tick"`. | Snap the price to the market's `price_decimals` / `max_price_sig_figs` before signing. The [Python SDK](python-sdk/README.md) checks this locally (`LocalValidationError`) and never sends it; see [Decimals & units](decimals-units.md#valid-invalid-examples). |

## Full /trade error-code reference

Every `error.code` `/trade` can return. **Request-shaping** codes are lowercase `snake_case` (HTTP 400, no `tx_hash`); the gateway operational codes at the end of the first table carry their own HTTP status. **Node-admission** codes are CamelCase, returned verbatim.

Request-shaping and gateway errors:

| Code | Meaning |
| --- | --- |
| `invalid_json` | The request body was not valid JSON. |
| `must provide action + nonce and a signature or signatures` | Required envelope fields were omitted. |
| `must provide exactly one of signature or signatures` | Both `signature` and `signatures` were present, or neither. |
| `signatures_not_allowed_for_action` | `signatures` (multisig) was sent for an action whose type does not accept a multisig proof. |
| `invalid_signatures_len` | The `signatures` array was empty or exceeded 32 entries. |
| `signatures_required_for_action` | A single `signature` was sent for a multisig-only action (e.g. `deposit`/ACCOUNTING, or an admin action under an active admin multisig policy). |
| `insufficient_signatures` | Fewer `signatures` than the required admin multisig threshold. |
| `legacy_signature_not_accepted` | A legacy (`auth_scheme` absent or `"legacy"`) signature was sent for an EIP-712 cutover action (`withdraw` / `settle` / `repay` / `approveAgent` / `revokeAgent`, or an operator `deposit`/`admin*`). These require `auth_scheme:"eip712"`. |
| `eip712_not_allowed_for_action` | `auth_scheme:"eip712"` was sent for a non-target action; only legacy is accepted for those. |
| `eip712_agent_epoch_not_allowed` | An `auth_scheme:"eip712"` request carried `agent_epoch`, which EIP-712 forbids. |
| `market_metadata_unavailable` | The write path needed market decimal metadata from node query state, but the metadata query failed or returned an invalid shape. |
| `asset_metadata_unavailable` | The write path needed asset decimal metadata from node query state, but the metadata query failed or returned an invalid shape. |
| `unknown_market` | The request referenced a market absent from refreshed market metadata. |
| `unknown_asset` | The request referenced an asset absent from refreshed asset metadata. |
| `invalid_market_id` | A market id was a valid `u64` JSON value but exceeded the protocol `u32` range. |
| `invalid_asset_id` | An asset id was a valid `u64` JSON value but exceeded the protocol `u32` range. |
| `invalid_dst_address` | `withdraw.dst_address` was not a 20-byte hex address. |
| `invalid_dst_chain_id` | `withdraw.dst_chain_id` was zero or exceeded the protocol `u32` range. |
| `invalid_cloid` | A `cloid` was not a 16-byte hex value. |
| `invalid_side` | `side` was not `bid`, `ask`, `buy`, or `sell`. |
| `invalid_order_type` | `order_type` was not `limit` or `market`. |
| `invalid_tif` | `tif` was not `gtc`, `ioc`, `fok`, or `alo`. |
| `missing_oid_or_cloid` | A `cancel` or `modify` target omitted both `oid` and `cloid`. Does not apply to `cancelAll`, which carries no `oid`/`cloid`. |
| `invalid_price` | `price` was numerically too large to parse as decimal conversion input. Malformed decimal strings are rejected before this response shape. |
| `invalid_price_precision` | `price` had more fractional digits than the market's `price_decimals`. |
| `invalid_price_overflow` | Decimal-to-atom conversion for `price` overflowed `u64`. |
| `invalid_quantity` | `quantity` was numerically too large to parse as decimal conversion input. Malformed decimal strings are rejected before this response shape. |
| `invalid_quantity_precision` | `quantity` had more fractional digits than the market's `base_quantity_decimals`. |
| `invalid_quantity_overflow` | Decimal-to-atom conversion for `quantity` overflowed `u64`. |
| `invalid_signature_hex` | `signature` was not hex or did not decode to exactly 65 bytes. |
| `encode_error: <TxCodecError>` | The write path could not assemble canonical signed tx bytes, for example because a batch length exceeded codec limits. |
| `empty_tx_bytes` | Defensive guard: canonical byte assembly produced an empty byte vector. This should not occur for normal JSON requests. |
| `decode_error: <TxDecodeError>` | The write path assembled bytes but could not decode them or recover the authorization (single signature, or a multisig proof — empty/too-many/duplicate/unsorted recovered signers). For public JSON this is the usual shape for a malformed or unrecoverable signature. |
| `RateLimited` | The signer exceeded the per-signer request rate (50/s over a 1-second window). HTTP `429`; includes `retry_after_ms`. |
| `PlaceOrderSuspended` | Place-order actions (`order` / `modify` / a `batch` with one) are suspended while the write path is degraded; `cancel` / `cancelAll` still admit. HTTP `503`, `retry_after_ms: 1000`. |
| `ExpiredTx` | The envelope's `expires_after_ms` was already past at the gateway clock; fast-failed before the node hop. HTTP `200`, `submission_status: "rejected"`. |
| `HandoffTimeout` / `HandoffBufferFull:{request_count\|bytes\|signer}` / `HandoffMultipleActive` | The submission could not be routed to a single writable node (leadership handoff / backpressure). HTTP `503`, `submission_status: "timeout"`, `retry_after_ms: 1000`. |
| `node_unreachable: <tonic error>` | The submit path could not complete node admission. HTTP `504`, `submission_status: "timeout"`. |

Node admission pass-through errors:

| Code | Meaning |
| --- | --- |
| `QueryLagBackpressure` | The node's query view is missing or more than four blocks behind execution; retry the same signed request after projection catches up. |
| `DuplicateTxHash` | The same transaction hash is already pending in ingress. |
| `DuplicateAuthorityNonce` | The same authority/nonce pair is already pending in ingress (authority is the recovered signer for single-sig, or the policy authority for multisig). |
| `MalformedTx` | The node could not decode canonical transaction bytes. Public JSON normally fails earlier if bytes cannot be built. |
| `BadSignature` | The node could not recover a signer from the canonical transaction signature. Public JSON normally fails earlier during signer recovery. |
| `AuthorityHintMismatch` | The decoded authority does not match the submit-path `authority_hint` (recovered signer for single-sig, derived policy authority for multisig). The hint did not match the canonical transaction. |
| `WrongChainId` | The signed payload's chain id did not match the node's configured chain id. |
| `TooManyPending` | Global pending capacity or per-owner pending capacity was reached. |
| `InvalidIngressConfig` | The node ingress configuration was invalid. |
| `DirectSignerIsActiveAgent` | A signer currently registered as an active agent attempted direct-owner mode. |
| `OwnerDoesNotExist` | Direct-owner admission resolved to an owner account that does not exist. |
| `UnknownAgent` | Agent-mode submission used a signer that is not an active agent. |
| `AgentEpochMismatch` | Agent-mode submission used an epoch that does not match the active agent slot epoch. |
| `AgentActionNotAllowed` | Agent-mode submission attempted an action kind not allowed for agent signatures. |
| `OracleUnavailable` | A non-cancel SpotCreditAccount action was submitted while the oracle status was unavailable. |
| `SpotCreditAccountFrozen` | A non-cancel action was submitted for a frozen SpotCreditAccount, including settle by a frozen margin signer. |
| `AccountNotFunded` | Balance-mode precheck found no balance row for the asset required by the order reserve. |
| `InsufficientSpotBalance` | Balance-mode precheck found clearly insufficient available balance for an order reserve or repay debit. |
| `MinTradeSpotNtl` | An order, modify replacement, or batch order/replacement was below the current quote asset minimum notional. Market orders use their submitted protection price for this precheck. |
| `InsufficientSpotCredit` | Spot-credit precheck showed the single-order risk leg or settle post-position value would take available credit below zero. |
| `DuplicateCloid` | The submitted order or modify replacement `cloid` is already open for the same owner and market, or the tx contains duplicate `(market_id, cloid)` intents. |
| `MarketNotFound` | The latest acceptable QueryView has no referenced market. |
| `OracleMarkPriceMissing` | A SpotCreditAccount order path or settle post-position check needs a fresh mark price that is absent or stale. |
| `AccountNotFound` | Settle/repay admission found a required account missing after owner admission resolution. |
| `BalanceOverflow` | Admission proved a settle credit would overflow the destination balance. |
| `ActionNotAllowedForSpotCreditAccount` | Withdraw admission found the signer is a SpotCreditAccount where only balance-mode accounts are allowed. |
| `InvalidWithdraw` | Withdraw payload is invalid, for example zero amount, zero destination chain, or zero destination address. |
| `WithdrawUnknownChainToken` | Withdraw destination chain/asset is not configured, or the asset is missing. |
| `WithdrawUnknownUser` | Withdraw admission could not resolve the signer owner account. |
| `WithdrawAmountBelowMinimum` | Withdraw amount is below the configured minimum for `(dst_chain_id, asset_id)`. |
| `WithdrawAmountNotAboveFee` | Withdraw amount is not greater than the asset's configured withdraw fee. |
| `WithdrawDuplicateNonce` | Withdraw business nonce is already committed or currently pending admission. |
| `WithdrawInsufficientBalance` | Withdraw admission proved the signer has insufficient available balance. |
| `InvalidSettle` | Settle payload, account shape, or margin position is invalid. |
| `InvalidRepay` | Repay payload, account shape, or target position is invalid. |
| `AssetNotFound` | A settle/repay asset reference was absent from the current committed state. |
| `Overflow` | Admission precheck arithmetic or id allocation overflowed. |
| `AccountFrozen` | The account was frozen by an operator (`adminFreezeAccount`); while frozen, only `cancel` / `cancelAll` are admitted. |
| `V3SignatureSuperseded` | The signature used the superseded **v3** EIP-712 scheme for a `withdraw` / `settle` / `repay`; only the **v4** scheme is accepted at submit — re-sign with v4. |

Node-admission codes are returned **verbatim** (CamelCase). A failure at execution comes back synchronously with a **lowercase** execution code (e.g. `tick`) — see [Execution-level failures](#execution-level-failures) above.

## See also

{% content-ref url="transaction-signing.md" %}
[transaction-signing.md](transaction-signing.md)
{% endcontent-ref %}

{% content-ref url="api-access.md" %}
[api-access.md](api-access.md)
{% endcontent-ref %}
