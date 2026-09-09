---
description: The /trade error model — read the response body, not the HTTP status.
---

# Error responses

`POST /trade` is **synchronous** and returns the **same JSON trade-response shape** whatever the HTTP status (`200`, `400`, `429`, `503`, `504`). Branch on the body, not the status line. A business rejection (below minimum, bad precision, insufficient balance) is **data**, not a transport error.

The envelope carries `submission_status` and, on `accepted`, a [`response` envelope](post-trade.md#what-accepted-carries) with the per-order outcome. **A failed order does not always produce a top-level `error`** — for order-ish actions the failure code lives in an `{"error":"<code>"}` leaf inside `response` while `submission_status` stays `accepted`. See [Execution-level failures](#execution-level-failures).

(One body never has this shape: a request over the 256 KiB `/trade` limit is refused before the handler runs and comes back as plain text. Guard your JSON parse.)

Only the transport layer raises — a non-trade-response `4xx`/`5xx` body, or a wire failure before any response arrived. A decodable trade response is always returned to you as-is.

## submission_status

The call blocks for the on-chain outcome and returns it directly.

There are exactly three values.

| `submission_status` | When | Do next |
| --- | --- | --- |
| `accepted` | The **transaction** landed and executed. For an order that covers rested, filled, a benign IOC/FOK/self-trade/no-liquidity cancel, **and a genuine per-action failure whose code sits in a `response` leaf**; for a non-order action it committed. No top-level `error` in any of those cases. | Not done — read `response.status` before you trust it. The leaf says what happened to the order: `filled` carries `total_sz`, `avg_px` and the `oid`; `open` and `cancelled` carry the `oid` and `cloid`; an `{"error": …}` leaf carries only the code. |
| `rejected` | The write was refused — request-shaping, gateway (rate limit / suspension / expiry), or a check before inclusion — **or** it failed at execution. `error.code` says why; `tx_hash` is present once canonical bytes exist. | If `RateLimited`, back off `error.retry_after_ms` and resend the same signed action. Otherwise fix the cause and submit a **fresh** action. |
| `timeout` | The outcome wasn't observed within the 3-second wait budget, or the submission couldn't be routed to a node. | Depends on `error.code` — `Unavailable` (503) never reached a node, so resubmit rather than lose the write; everything else may still land, so reconcile by `cloid` and **never** resubmit under a new nonce. |

{% hint style="warning" %}
`timeout` is not `rejected` — the transaction may still commit in a later block, and resubmitting under a new nonce is the one move that can double-fill you. Reconcile an order by `cloid` via [`orderStatus`](post-info.md#orderstatus). Not `txStatusByCloid`: only funding and admin actions are indexed there, so an order cloid always comes back `found: false`.
{% endhint %}

A `timeout` has three shapes, and only one of them is safe to resubmit:

| `error.code` | HTTP | Reached a node? | Do next |
| --- | --- | --- | --- |
| *(none)* — the wait budget elapsed | 200 | Yes, it is executing | Reconcile by `cloid` |
| `Unavailable` | 503 | No — refused before the write left the API | Resubmit the same signed bytes; nothing was delivered |
| `NodeUnreachable` | 504 | Unknown — the connection broke mid-submission | Reconcile by `cloid` |

## Where a code comes from

Every `error.code` comes from one of four layers, and the **spelling tells you which**:

| Layer | Style | Examples | HTTP | `submission_status` |
| --- | --- | --- | --- | --- |
| Request-shaping | `CamelCase` | `InvalidJson`, `InvalidQuantityPrecision`, `MissingCloid` | 400 | `rejected` (no `tx_hash`) |
| Gateway | `CamelCase` | `RateLimited`, `PlaceOrderSuspended`, `ExpiredTx`, `Unavailable`, `NodeUnreachable` | 429 / 503 / 504 / 200 | `rejected` or `timeout` |
| Checked before inclusion | `CamelCase` | `MinTradeSpotNtl`, `DuplicateCloid`, `InsufficientSpotBalance`, `AccountFrozen` | 200 | `rejected` |
| Execution — order-ish | lowercase (the variant name) | `tick`, `insufficientspotbalance`, | 200 | `accepted`, code in the `response` leaf |
| Execution — envelope | CamelCase display form | `BadNonce`, `BadSignature`, `ExpiredTx` | 200 | `rejected` |

One condition can surface at two layers with different spellings — an under-minimum order is usually caught at admission as `MinTradeSpotNtl`, but the same failure at execution reads `mintradespotntl`. Match on the code you actually receive.

## Errors you'll actually hit

The wire carries `error.code`, not display copy — the **Message** column is illustrative UI text you'd render from the code. This is the curated subset you meet in practice; the [full catalog](#full-trade-error-code-reference) is at the bottom of this page.

| Code | What it means | Message a user sees | Fix |
| --- | --- | --- | --- |
| `RateLimited` | Over quota on either limiter, never admitted. HTTP `429`, carries `error.retry_after_ms`. **`tx_hash` tells them apart**: the per-IP budget (1 req/s) is enforced before the body is parsed, so the reply has no `tx_hash`; the per-signer rate (1000 req/s) is enforced after canonicalization, so it does. | "Too many requests — retrying shortly." | Back off `retry_after_ms`, then resend the same signed action — one of the three rejections that is safe to resend unchanged, with `TooManyPending` and `QueryLagBackpressure`. |
| `PlaceOrderSuspended` | The write path is degraded, so order placement is suspended: `order`, `modify`, and any `batch` that contains a non-cancel item are refused. Only `cancel` / `cancelAll` — and a `batch` whose **every** item is `cancel` / `cancelAll` — still go through; an empty `batch` is also refused. HTTP `503`, `error.retry_after_ms: 1000`. | "Placing orders is paused — try again shortly." | Back off and retry; keep cancelling if you need to reduce exposure. |
| `Unavailable` (`timeout`) | The write plane could not take the transaction and did not run it. `submission_status: "timeout"`, HTTP `503`, `error.retry_after_ms: 1000`. | "Reconnecting — retrying your order." | Back off `retry_after_ms` and resubmit the same signed bytes. There is nothing to reconcile — see [submission_status](#submission_status). |
| `NodeUnreachable` (`timeout`) | The transaction could not be delivered to the chain; `submission_status: "timeout"`, HTTP `504`. The action may still land. | "Order submitted — confirming status." | Reconcile by `cloid`; never resubmit under a new nonce. |
| `MinTradeSpotNtl` | Order, modify replacement, or batch item is below the market's quote-asset minimum notional. Market orders use their protection price. | "Order must have a minimum value of 10 USDC." | Size up so `price × quantity` clears the minimum. |
| `InvalidPricePrecision` / `InvalidQuantityPrecision` | `price` / `quantity` has more fractional digits than the market's `price_decimals` / `base_quantity_decimals`. | "Price has too many decimal places for this market." | Snap to market precision before signing; send strings, never floats. |
| `DuplicateCloid` | The `cloid` is already open for this owner and market, or the tx repeats a `(market_id, cloid)`. | "An order with this ID already exists." | Use a fresh `cloid` per order. When reconciling a `timeout`, look the existing one up — don't resend. |
| `ExpiredTx` | The signed `expires_after_ms` had already passed when execution reached the tx. | "Order expired before it was placed." | Widen `expires_after_ms`, re-sign, resubmit. |
| `InsufficientSpotBalance` | Balance-mode precheck found too little available balance for the order reserve. | "Insufficient balance." | Fund the account's quote asset — deposit from your main wallet in the web app. |
| `DirectSignerIsActiveAgent` | An active API-wallet key signed in direct-owner mode. An agent key may never sign as an owner. | "This API wallet can't trade as an owner." | Sign in agent mode — send `agent_epoch` with the agent-signed request. The owner address is never sent on the wire — the API recovers it from the signature. |
| `AgentEpochMismatch` | The signed `agent_epoch` doesn't match the live agent-slot epoch. | "Session expired — reconnecting." | Re-resolve `agent_epoch` from [`userAgents`](post-info.md#useragents) and resubmit. If it persists, the API wallet was revoked or re-approved — create a new one in the web app. |
| `AccountFrozen` | The account was frozen by an operator; while frozen, only `cancel` / `cancelAll` are admitted — new orders and modifies are rejected. | "Your account is frozen — trading is paused." | Stop placing orders until the freeze is lifted; cancels still go through. |

## Batch

A [`batch`](post-trade.md#batch) is one `/trade` call under one envelope nonce, so the envelope gets **one** `submission_status`. That status reflects the batch's admission and overall outcome — it does **not** report each item. Items execute in array order, and **each item may individually succeed or fail** inside the batch execution result. On `accepted`, read the per-item outcomes straight out of [`response.statuses[]`](post-trade.md#batch), in item order — no `/info` lookup needed. Fall back to one [`orderStatus`](post-info.md#orderstatus) per item only when the envelope came back `timeout`. Don't infer item outcomes from the envelope status.

## Execution-level failures

An admitted action still runs against the book and **can fail at execution**. Because `/trade` is synchronous, that failure comes back on the `/trade` response — but **where** it appears depends on the action, and getting this wrong reads a failed order as a success.

* **Order-ish actions** (`order`, `cancel`, `cancelAll`, `modify`, `batch`) stay `submission_status: "accepted"` with **no** top-level `error`. The code appears only as a leaf inside the [`response` envelope](post-trade.md#what-accepted-carries), as `{"error":"<code>"}`. How deep that leaf sits follows the action: `response.status.error` for an `order`, `cancel`, or `modify`; `response.statuses[i].error` for a `cancelAll`; `response.statuses[i].status.error` for a [`batch`](post-trade.md#batch) item. This covers `insufficientspotbalance`, `mintradespotntl`, `tick`, `missingorder`, and the rest.
* **Non-order actions** (`transfer` / `activateFor` / `withdraw` / `settle` / `repay` / `approveAgent` / `revokeAgent`) do map an execution failure to `submission_status: "rejected"` with a top-level `error.code`.
* **Six envelope-level failures** demote any action to `rejected` because they invalidate the transaction itself: `badnonce`, `badsignature`, `expiredtx`, `malformedtx`, `invalidbatchlength`, `featuredisabled`. These surface in their CamelCase display form — `BadNonce`, `BadSignature`, and so on.

{% hint style="warning" %}
**The same failure has two spellings, and which one you get depends on where you read it.** `error.code` at the top level is always CamelCase; a leaf code inside `response` is always lowercase with no separator. An order rejected for balance reads `insufficientspotbalance` in the leaf, while the same failure on a `withdraw` reads `InsufficientSpotBalance` at the top level. Match each field against its own vocabulary.

`error.code` at the top level is never a lowercase execution code for an order. If you are matching on `error.code == "tick"`, you will never hit it — look in the `response` leaf instead.

Whether you can look the order up afterwards depends on how far it got:

| Leaf code | What it leaves behind |
| --- | --- |
| `tick` `insufficientspotbalance` `mintradespotntl` | A status row keyed by `cloid` with no `oid`. Reconciling the `cloid` finds it |
| `missingorder` | **Nothing.** A cancel carries no client order intent, so there is nothing to key a row on. Reconciling finds nothing and times out |
| `badalopx` `insufficientspotcredit` | An [`orderStatus`](post-info.md#orderstatus) row under that lowercase status, and an `orderUpdates` frame (`badAloPxRejected`) |
| `ioccancel` `fokcancel` `selftradepreventioncancel` `marketordernoliquidity` | The same, and benign: the order simply did not fill |

Either way, read the leaf on the response, then send a corrected order under a **new** `cloid`.
{% endhint %}

| Code | Where it appears | What it means | Fix |
| --- | --- | --- | --- |
| `tick` | the `response` leaf, with `submission_status: "accepted"` | A non-integer `price` exceeded the market's `max_price_sig_figs`. The transaction landed; the order never entered the book. | Snap the price to the market's `price_decimals` / `max_price_sig_figs` before signing. The [Python SDK](python-sdk/README.md) checks this locally (`LocalValidationError`) and never sends it; see [Decimals & units](decimals-units.md#valid-invalid-examples). |
| `insufficientspotbalance` / `mintradespotntl` / `badalopx` / `missingorder` | the `response` leaf, with `submission_status: "accepted"` | The order failed at execution for the stated reason. | Same handling as the CamelCase admission form of the condition — the difference is only which layer caught it. |
| `BadNonce` / `BadSignature` / `ExpiredTx` / `MalformedTx` / `InvalidBatchLength` / `FeatureDisabled` | top-level `error.code`, with `submission_status: "rejected"` | The transaction envelope itself was invalid, so nothing executed. `InvalidBatchLength` is the execution-layer guard on an empty or over-long `batch`; over `POST /trade` you will not normally see it, because an oversized batch fails at canonicalization first and returns HTTP 400 with `error.code: "EncodeLengthOverflow"` and no `tx_hash`. | Re-sign correctly and submit a fresh action. |

## Full /trade error-code reference

Every `error.code` `/trade` can return. **Request-shaping** codes are `CamelCase` (HTTP 400, no `tx_hash`); the gateway operational codes at the end of the first table carry their own HTTP status. **Node-admission** codes are CamelCase, returned verbatim.

Request-shaping and gateway errors:

| Code | Meaning |
| --- | --- |
| `InvalidJson` | The request body was not valid JSON. |
| `MissingAuthFields` | Required envelope fields were omitted. |
| `AmbiguousAuthFields` | Both `signature` and `signatures` were present, or neither. |
| `SignaturesNotAllowedForAction` | `signatures` (multisig) was sent for an action whose type does not accept a multisig proof. |
| `InvalidSignaturesLen` | The `signatures` array was empty or exceeded 32 entries. |
| `SignaturesRequiredForAction` | A single `signature` was sent for a multisig-only action (e.g. `deposit`/ACCOUNTING, or an admin action under an active admin multisig policy). |
| `InsufficientSignatures` | Fewer `signatures` than the required admin multisig threshold. |
| `LegacySignatureNotAccepted` | A legacy (`auth_scheme` absent or `"legacy"`) signature was sent for an EIP-712 cutover action (`transfer` / `activateFor` / `withdraw` / `settle` / `repay` / `approveAgent` / `revokeAgent`, or an operator `deposit`/`admin*`). These require `auth_scheme:"eip712"`. |
| `Eip712NotAllowedForAction` | `auth_scheme:"eip712"` was sent for a non-target action; only legacy is accepted for those. |
| `Eip712AgentEpochNotAllowed` | An `auth_scheme:"eip712"` request carried `agent_epoch`, which EIP-712 forbids. |
| `UnknownMarket` | The request referenced a market that does not exist. |
| `UnknownAsset` | The request referenced an asset that does not exist. |
| `InvalidMarketId` | A market id was a valid `u64` JSON value but exceeded the protocol `u32` range. |
| `InvalidAssetId` | An asset id was a valid `u64` JSON value but exceeded the protocol `u32` range. |
| `InvalidDstAddress` | `withdraw.dst_address` was not a 20-byte hex address. |
| `InvalidDstChainId` | `withdraw.dst_chain_id` exceeded the protocol `u32` range. A **zero** chain id passes this check and is rejected later at admission as `InvalidWithdraw` (HTTP 200). |
| `MissingCloid` | An action that requires a client id omitted `cloid`. |
| `InvalidCloid` | A `cloid` was not a 16-byte hex value. |
| `InvalidSide` | `side` was not `bid`, `ask`, `buy`, or `sell`. |
| `InvalidOrderType` | `order_type` was not `limit` or `market`. |
| `InvalidTif` | `tif` was not `gtc`, `ioc`, `fok`, or `alo`. |
| `MissingOidOrCloid` | A `cancel` or `modify` target omitted both `oid` and `cloid`. Does not apply to `cancelAll`, which carries no `oid`/`cloid`. |
| `InvalidPrice` | `price` was numerically too large to parse as decimal conversion input. Malformed decimal strings are rejected before this response shape. |
| `InvalidPricePrecision` | `price` had more fractional digits than the market's `price_decimals`. |
| `InvalidPriceOverflow` | Decimal-to-atom conversion for `price` overflowed `u64`. |
| `InvalidQuantity` | `quantity` was numerically too large to parse as decimal conversion input. Malformed decimal strings are rejected before this response shape. |
| `InvalidQuantityPrecision` | `quantity` had more fractional digits than the market's `base_quantity_decimals`. |
| `InvalidQuantityOverflow` | Decimal-to-atom conversion for `quantity` overflowed `u64`. |
| `InvalidSignatureHex` | `signature` was not hex or did not decode to exactly 65 bytes. |
| `Encode…` | The write path could not assemble canonical signed tx bytes, for example because a batch length exceeded codec limits. |
| `EmptyTxBytes` | Defensive guard: canonical byte assembly produced an empty byte vector. This should not occur for normal JSON requests. |
| `DecodeCodec…` / `DecodeSignature…` / `DecodeMultisigProof…` | The write path assembled bytes but could not decode them or recover the authorization (single signature, or a multisig proof — empty/too-many/duplicate/unsorted recovered signers). For public JSON this is the usual shape for a malformed or unrecoverable signature. |
| `RateLimited` | Over quota on either limiter: the per-IP budget (1 req/s per endpoint by default, enforced before parsing — no `tx_hash`) or the per-signer rate (1000/s over a 1-second window, enforced after canonicalization — carries `tx_hash`). HTTP `429`; includes `retry_after_ms`. |
| `TooManyPending` | Too many synchronous `/trade` waits are already in flight on this instance. HTTP `503`, `submission_status: "rejected"`, `retry_after_ms: 50` — transient, retry immediately. Distinct from the same-named node-admission code below. |
| `PlaceOrderSuspended` | Order placement is suspended while the write path is degraded. Admitted: `cancel`, `cancelAll`, and a `batch` whose **every** item is `cancel` / `cancelAll`. Refused: `order`, `modify`, any `batch` that mixes in a non-cancel item, and an empty `batch`. HTTP `503`, `retry_after_ms: 1000`. |
| `ExpiredTx` | The envelope's `expires_after_ms` was already past at the gateway clock; fast-failed before the node hop. HTTP `200`, `submission_status: "rejected"`. |
| `Unavailable` | The write plane could not take the transaction, and did not run it. HTTP `503`, `submission_status: "timeout"`, `retry_after_ms: 1000`. Safe to resend unchanged. |
| `NodeUnreachable` | The transaction could not be delivered to the chain. HTTP `504`, `submission_status: "timeout"`. |

Errors returned before the transaction is included in a block:

| Code | Meaning |
| --- | --- |
| `QueryLagBackpressure` | The node is briefly behind and not accepting writes. Wait a moment and retry the same signed request. |
| `DuplicateTxHash` | The same transaction hash is already pending in ingress. |
| `DuplicateAuthorityNonce` | The same authority/nonce pair is already pending in ingress (authority is the recovered signer for single-sig, or the policy authority for multisig). |
| `MalformedTx` | The node could not decode canonical transaction bytes. Public JSON normally fails earlier if bytes cannot be built. |
| `BadSignature` | The node could not recover a signer from the canonical transaction signature. Public JSON normally fails earlier during signer recovery. |
| `AuthorityHintMismatch` | The decoded authority does not match the submit-path `authority_hint` (recovered signer for single-sig, derived policy authority for multisig). The hint did not match the canonical transaction. |
| `WrongChainId` | The signed payload's chain id did not match the node's configured chain id. |
| `TooManyPending` | Global pending capacity or per-owner pending capacity was reached at the node. (The API also emits this code itself, at HTTP `503` with `retry_after_ms: 50`, when its own synchronous-write concurrency is saturated — see the gateway table above.) |
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
