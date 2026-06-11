# Transaction Signing

{% hint style="info" %}
This doc describes the underlying methodology and process of handling transaction signing.

Official SDK coming soon for various common programming languages.
{% endhint %}

`signature` is not a signature over the JSON text. The write path reconstructs the canonical unsigned transaction payload from `action`, `nonce`, `agent_epoch`, and `expires_after_ms`, then verifies the recoverable secp256k1 signature over that exact binary payload. For order and modify actions, public JSON `price` and `quantity` are display decimals; the signed binary action contains the raw atoms obtained from market `price_decimals` and `base_quantity_decimals`. The JSON action and signed binary action must describe the same action.

Canonical unsigned payload:

```
string("NATIVE_CORE_TX_SIGNING_V1")
u32(1)                  // tx codec version
u32(696969)             // Native Core chain id
u64(nonce)
option<u64>(agent_epoch)
option<u64>(expires_after_ms)
action_bytes
```

Encoding rules:

* Integers are unsigned big-endian.
* `string` and byte vectors encode as `u32 byte_length` followed by raw bytes.
* `option<T>` encodes as `u8(0)` for `null`/omitted, or `u8(1)` followed by `T`.
* Hex values are raw bytes after removing the `0x` prefix.
* The signing digest is `keccak256(unsigned_payload_bytes)`.
* `signature` is `0x` + 65 bytes encoded as `r || s || v`; `v` may be `0/1` or `27/28`. High-s signatures are rejected.

The legacy scheme above applies to trading actions. Authorization-sensitive actions use EIP-712 (next section).

### EIP-712 signing (auth_scheme: "eip712")

Public `withdraw`, `settle`, and `repay` must be submitted with `auth_scheme: "eip712"`. This is a **direct cutover**: the moment the new binary is live, legacy signatures over these public actions are rejected (`legacy_signature_not_accepted`), and there is no config switch, height activation, or grace window — public `withdraw`/`settle`/`repay` clients must switch at deploy. Conversely, `auth_scheme="eip712"` on any non-target action is rejected (`eip712_not_allowed_for_action`), and an EIP-712 request may not carry `agent_epoch` (`eip712_agent_epoch_not_allowed`).

The signature covers an EIP-712 typed-data digest, not a binary payload. Clients sign the **v4** scheme, which is MetaMask-compatible: the domain is `EIP712Domain{name:"Native Core", version:"1", verifyingContract:0x0000…0000}` — **no `chainId`** — so a wallet can sign while connected to any EVM chain. The Native chain id is instead a signed message field, `nativeChainId`, so mainnet/testnet replay separation is preserved. Each target action has its own primary type whose fields mirror the action, prefixed by the common fields `uint256 nativeChainId, uint256 authKind, uint256 authScope, uint256 nonce, bool expiresAfterMsPresent, uint256 expiresAfterMs`. `nativeChainId` is the Native Core chain id; `authKind` is `1` (single) and `authScope` is `0` for these public user actions. Amounts are signed as canonical atoms; addresses as `address`; an optional `cloid` as `bool cloidPresent` + `bytes16 cloid`. The presence flags keep an absent value distinct from an explicit `0`. The transaction **authority** is the recovered signer, exactly as for legacy single-signature actions.

A superseded **v3** EIP-712 scheme (domain included `chainId`; no `nativeChainId` field) is retained only for historical decode/replay and is **not accepted at submit**. Because `/trade` carries no codec-version field, a request whose signature was produced under the old v3 scheme is assembled as v4 and recovers a different address, so it fails with a signature/authority error — re-sign with the v4 scheme.

`withdraw` keeps an optional `cloid` at the protocol level (legacy WAL records may omit it), but the public gateway JSON requires `cloid`; the EIP-712 `cloidPresent` flag models the optionality.

Public action tags:

| JSON action                | Canonical tag | Notes                                                                                                                         |
| -------------------------- | ------------: | ----------------------------------------------------------------------------------------------------------------------------- |
| `order`                    |           `0` | Top-level order.                                                                                                              |
| `cancel` with `oid`        |           `2` | If both `oid` and `cloid` are present, `oid` wins.                                                                            |
| `cancel` with only `cloid` |           `4` | Canonical cancel-by-cloid.                                                                                                    |
| `modify` with `oid`        |           `6` | If both `oid` and `cloid` are present, `oid` wins.                                                                            |
| `modify` with only `cloid` |           `8` | Canonical modify-by-cloid.                                                                                                    |
| `cancelAll`                |          `26` | Cancel every open order for the effective owner in one market. No `cloid`.                                                    |
| `batch`                    |          `18` | Batch item tags are `order=0`, `cancel by oid=1`, `modify by oid=2`, `modify by cloid=3`, `cancel by cloid=4`, `cancelAll=5`. |

Order action bytes:

```
u16(action_tag)
u32(market_id)
u8(side)          // bid=0, ask=1
u8(order_type)    // limit=0, market=1
u8(tif)           // gtc=0, ioc=1, fok=2, alo=3
option<u64>(price)
u64(quantity)
option<bytes16>(cloid)
```

Cancel action bytes:

```
// by oid
u16(2)
u32(market_id)
u64(oid)

// by cloid
u16(4)
u32(market_id)
bytes16(cloid)
```

Modify action bytes:

```
// by oid
u16(6)
u32(market_id)
u64(oid)
replacement_order

// by cloid
u16(8)
u32(market_id)
bytes16(cloid)
replacement_order
```

`replacement_order` uses the same fields as `order` after `market_id`:

```
u8(side)
u8(order_type)
u8(tif)
option<u64>(price)
u64(quantity)
option<bytes16>(cloid)
```

CancelAll action bytes:

```
u16(26)
u32(market_id)
```

`cancelAll` carries no `oid` and no `cloid`. The signed payload is a single `u32` market id; the JSON body must match (`{ "type": "cancelAll", "market_id": "<id>" }`).

Batch action bytes:

```
u16(18)
u32(item_count)
item_0
item_1
...
```

Each batch item starts with a `u8` item tag followed by the item payload without the top-level `u16` action tag:

```
0: order payload
1: cancel-by-oid payload
2: modify-by-oid payload
3: modify-by-cloid payload
4: cancel-by-cloid payload
5: cancelAll payload (u32 market_id)
```

TypeScript signing helper example for Node.js with `ethers`. The example uses Node's global `Buffer`; import it from `node:buffer` only if your TypeScript setup requires explicit Buffer types.

```ts
import { SigningKey, getBytes, keccak256 } from "ethers";

const NativeCore = {
  chainId: 696_969,
  txCodecVersion: 1,
  signingDomain: "NATIVE_CORE_TX_SIGNING_V1",
} as const;

const ActionTag = {
  order: 0,
  cancelByOid: 2,
  cancelByCloid: 4,
  modifyByOid: 6,
  modifyByCloid: 8,
  batch: 18,
  cancelAll: 26,
} as const;

const BatchItemTag = {
  order: 0,
  cancelByOid: 1,
  modifyByOid: 2,
  modifyByCloid: 3,
  cancelByCloid: 4,
  cancelAll: 5,
} as const;

type TradeAction = OrderAction | CancelAction | CancelAllAction | ModifyAction | BatchAction;
type BatchItem = OrderAction | CancelAction | CancelAllAction | ModifyAction;
type Side = "bid" | "ask" | "buy" | "sell";
type OrderType = "limit" | "market";
type Tif = "gtc" | "ioc" | "fok" | "alo";

type OrderAction = {
  type: "order";
  market_id: string;
  side: Side;
  order_type: OrderType;
  tif: Tif;
  price?: string | null;
  quantity: string;
  cloid?: string | null;
};

type CancelAction = {
  type: "cancel";
  market_id: string;
  oid?: string;
  cloid?: string;
};

type CancelAllAction = {
  type: "cancelAll";
  market_id: string;
};

type ReplacementOrder = Omit<OrderAction, "type" | "market_id">;

type ModifyAction = {
  type: "modify";
  market_id: string;
  oid?: string;
  cloid?: string;
  replacement: ReplacementOrder;
};

type BatchAction = {
  type: "batch";
  items: BatchItem[];
};

type MarketScale = {
  price_decimals: number;
  base_quantity_decimals: number;
};

type SignOptions = {
  nonce?: bigint;
  agentEpoch?: bigint;
  expiresAfterMs?: bigint;
};

class Writer {
  chunks: Uint8Array[] = [];

  bytes() { return Buffer.concat(this.chunks.map((x) => Buffer.from(x))); }
  raw(x: Uint8Array) { this.chunks.push(x); }
  u8(x: number) { const b = Buffer.alloc(1); b.writeUInt8(x); this.raw(b); }
  u16(x: number) { const b = Buffer.alloc(2); b.writeUInt16BE(x); this.raw(b); }
  u32(x: number) { const b = Buffer.alloc(4); b.writeUInt32BE(x); this.raw(b); }
  u64(x: bigint) { const b = Buffer.alloc(8); b.writeBigUInt64BE(x); this.raw(b); }
  string(x: string) { const b = Buffer.from(x, "utf8"); this.u32(b.length); this.raw(b); }
  none() { this.u8(0); }
  some(f: () => void) { this.u8(1); f(); }
}

class NativeCoreTradeSigner {
  constructor(
    private readonly privateKey: string,
    private readonly markets: Record<string, MarketScale>,
  ) {}

  sign(action: TradeAction, options: SignOptions = {}) {
    const nonce = options.nonce ?? BigInt(Date.now());
    const unsignedPayload = this.encodeUnsignedPayload(action, nonce, options);
    const signature = this.signPayload(unsignedPayload);
    return {
      action,
      nonce: nonce.toString(),
      ...(options.agentEpoch !== undefined
        ? { agent_epoch: options.agentEpoch.toString() }
        : {}),
      ...(options.expiresAfterMs !== undefined
        ? { expires_after_ms: options.expiresAfterMs.toString() }
        : {}),
      signature,
    };
  }

  private encodeUnsignedPayload(
    action: TradeAction,
    nonce: bigint,
    options: SignOptions,
  ): Buffer {
    const out = new Writer();
    out.string(NativeCore.signingDomain);
    out.u32(NativeCore.txCodecVersion);
    out.u32(NativeCore.chainId);
    out.u64(nonce);
    this.writeOptionU64(out, options.agentEpoch);
    this.writeOptionU64(out, options.expiresAfterMs);
    out.raw(this.encodeAction(action));
    return out.bytes();
  }

  private encodeAction(action: TradeAction): Buffer {
    const out = new Writer();
    if (action.type === "order") {
      out.u16(ActionTag.order);
      this.writeOrder(out, action);
    } else if (action.type === "cancel") {
      this.writeCancel(out, action, true);
    } else if (action.type === "cancelAll") {
      this.writeCancelAll(out, action, true);
    } else if (action.type === "modify") {
      this.writeModify(out, action, true);
    } else {
      out.u16(ActionTag.batch);
      out.u32(action.items.length);
      for (const item of action.items) {
        this.writeBatchItem(out, item);
      }
    }
    return out.bytes();
  }

  private writeBatchItem(out: Writer, item: BatchItem): void {
    if (item.type === "order") {
      out.u8(BatchItemTag.order);
      this.writeOrder(out, item);
    } else if (item.type === "cancel") {
      this.writeCancel(out, item, false);
    } else if (item.type === "cancelAll") {
      this.writeCancelAll(out, item, false);
    } else {
      this.writeModify(out, item, false);
    }
  }

  private writeCancelAll(out: Writer, action: CancelAllAction, topLevel: boolean): void {
    if (topLevel) {
      out.u16(ActionTag.cancelAll);
    } else {
      out.u8(BatchItemTag.cancelAll);
    }
    out.u32(this.u32String(action.market_id, "market_id"));
  }

  private writeCancel(out: Writer, action: CancelAction, topLevel: boolean): void {
    if (action.oid !== undefined) {
      topLevel ? out.u16(ActionTag.cancelByOid) : out.u8(BatchItemTag.cancelByOid);
      out.u32(this.u32String(action.market_id, "market_id"));
      out.u64(BigInt(action.oid));
    } else if (action.cloid !== undefined) {
      topLevel ? out.u16(ActionTag.cancelByCloid) : out.u8(BatchItemTag.cancelByCloid);
      out.u32(this.u32String(action.market_id, "market_id"));
      out.raw(this.hexBytes(action.cloid, 16));
    } else {
      throw new Error("cancel requires oid or cloid");
    }
  }

  private writeModify(out: Writer, action: ModifyAction, topLevel: boolean): void {
    if (action.oid !== undefined) {
      topLevel ? out.u16(ActionTag.modifyByOid) : out.u8(BatchItemTag.modifyByOid);
      out.u32(this.u32String(action.market_id, "market_id"));
      out.u64(BigInt(action.oid));
    } else if (action.cloid !== undefined) {
      topLevel ? out.u16(ActionTag.modifyByCloid) : out.u8(BatchItemTag.modifyByCloid);
      out.u32(this.u32String(action.market_id, "market_id"));
      out.raw(this.hexBytes(action.cloid, 16));
    } else {
      throw new Error("modify requires oid or cloid");
    }
    this.writeReplacement(out, action.replacement, action.market_id);
  }

  private writeOrder(out: Writer, order: OrderAction): void {
    out.u32(this.u32String(order.market_id, "market_id"));
    this.writeReplacement(out, order, order.market_id);
  }

  private writeReplacement(out: Writer, order: ReplacementOrder, marketId: string): void {
    const market = this.market(marketId);
    out.u8(this.sideTag(order.side));
    out.u8(this.orderTypeTag(order.order_type));
    out.u8(this.tifTag(order.tif));
    this.writeOptionU64(
      out,
      order.price == null ? undefined : this.decimalToAtoms(order.price, market.price_decimals),
    );
    out.u64(this.decimalToAtoms(order.quantity, market.base_quantity_decimals));
    if (order.cloid == null) out.none();
    else out.some(() => out.raw(this.hexBytes(order.cloid!, 16)));
  }

  private writeOptionU64(out: Writer, value?: bigint): void {
    if (value === undefined) out.none();
    else out.some(() => out.u64(value));
  }

  private signPayload(unsignedPayload: Buffer): string {
    const digest = Buffer.from(getBytes(keccak256(unsignedPayload)));
    const sig = new SigningKey(this.privateKey).sign(digest);
    return (
      "0x" +
      Buffer.concat([
        Buffer.from(getBytes(sig.r)),
        Buffer.from(getBytes(sig.s)),
        Buffer.from([sig.yParity]),
      ]).toString("hex")
    );
  }

  private hexBytes(hex: string, length: number): Buffer {
    const bytes = Buffer.from(getBytes(hex));
    if (bytes.length !== length) throw new Error(`expected ${length} bytes`);
    return bytes;
  }

  private u32String(value: string, field: string): number {
    const n = Number(value);
    if (!Number.isInteger(n) || n < 0 || n > 0xffff_ffff) {
      throw new Error(`${field} must be a u32 decimal string`);
    }
    return n;
  }

  private sideTag(side: Side): number {
    if (side === "bid" || side === "buy") return 0;
    if (side === "ask" || side === "sell") return 1;
    throw new Error(`invalid side ${side}`);
  }

  private orderTypeTag(orderType: OrderType): number {
    if (orderType === "limit") return 0;
    if (orderType === "market") return 1;
    throw new Error(`invalid order_type ${orderType}`);
  }

  private tifTag(tif: Tif): number {
    return { gtc: 0, ioc: 1, fok: 2, alo: 3 }[tif];
  }

  private market(marketId: string): MarketScale {
    const market = this.markets[marketId];
    if (!market) throw new Error(`missing market metadata for ${marketId}`);
    return market;
  }

  private decimalToAtoms(value: string, decimals: number): bigint {
    if (!/^[0-9]+(?:\.[0-9]+)?$/.test(value)) throw new Error(`invalid decimal ${value}`);
    const [whole, fractional = ""] = value.split(".");
    if (fractional.length > decimals) throw new Error(`too much decimal precision: ${value}`);
    return BigInt(whole) * 10n ** BigInt(decimals) +
      BigInt(fractional.padEnd(decimals, "0") || "0");
  }
}

const signer = new NativeCoreTradeSigner("0x...", {
  "2": { price_decimals: 2, base_quantity_decimals: 4 },
});
const request = signer.sign({
  type: "order",
  market_id: "2",
  side: "bid",
  order_type: "limit",
  tif: "alo",
  price: "3500.00",
  quantity: "1.0000",
  cloid: "0x11111111111111111111111111111111",
});

await fetch(`${API_URL}/trade`, {
  method: "POST",
  headers: { "content-type": "application/json" },
  body: JSON.stringify(request),
});
```

Accepted response:

```json
{
  "submission_status": "accepted",
  "tx_hash": "0x..."
}
```

Rejected response:

```json
{
  "submission_status": "rejected",
  "tx_hash": "0x...",
  "error": {
    "code": "MinTradeSpotNtl"
  }
}
```

Rejected request-local validation responses usually have no `tx_hash` because canonical bytes were not assembled. Rejections after canonical byte assembly include `tx_hash`. Rate-limit responses also include `error.retry_after_ms`. If node admission cannot be reached, the response is `submission_status: "timeout"` with an `error.code` beginning with `node_unreachable: `.

Malformed or non-decodable JSON is handled by the gateway and returns the `TradeResponse` shape with `error.code = "invalid_json"`. This includes invalid action type tags and invalid field types. Request-local validation failures after decoding, such as negative numeric strings or malformed decimal strings, return their specific error code in the same response shape. All such responses include `x-trace-id`.

### `/trade` error codes

Request-shaping errors:

| Code | Meaning |
| --- | --- |
| `invalid_json` | The request body was not valid JSON. |
| `must provide action + nonce and a signature or signatures` | Required envelope fields were omitted. |
| `must provide exactly one of signature or signatures` | Both `signature` and `signatures` were present, or neither. |
| `signatures_not_allowed_for_action` | `signatures` (multisig) was sent for an action whose type does not accept a multisig proof. |
| `invalid_signatures_len` | The `signatures` array was empty or exceeded 32 entries. |
| `legacy_signature_not_accepted` | A legacy (`auth_scheme` absent or `"legacy"`) signature was sent for `withdraw`, `settle`, or `repay`. These require `auth_scheme:"eip712"`. |
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
| `RateLimited` | The signer exceeded the per-signer request rate. The response includes `retry_after_ms`. |
| `node_unreachable: <tonic error>` | The submit path could not complete node admission. The HTTP status is `504` and `submission_status` is `"timeout"`. |

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

Execution remains authoritative, so an accepted response means the tx entered the node pipeline; it may still later execute as a failed transaction.

Supported top-level action types:

* `order`
* `cancel`
* `cancelAll`
* `modify`
* `batch`
* `withdraw` (user single-signature, EIP-712 `auth_scheme:"eip712"`; see [withdraw](post-trade.md#withdraw))
* `settle` (user single-signature, EIP-712 `auth_scheme:"eip712"`)
* `repay` (user single-signature, EIP-712 `auth_scheme:"eip712"`)
