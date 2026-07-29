# Transaction Signing

{% hint style="info" %}
This page specifies the byte-level methodology behind `/trade` signing, for building your own client in any language. If you use the [Python SDK](python-sdk/README.md), it signs every action for you — you do not need anything on this page.
{% endhint %}

`signature` is not a signature over the JSON text. The write path reconstructs the canonical unsigned transaction payload from `action`, `nonce`, `agent_epoch`, and `expires_after_ms`, then verifies the recoverable secp256k1 signature over that exact binary payload. For order and modify actions, public JSON `price` and `quantity` are display decimals; the signed binary action contains the raw atoms obtained from market `price_decimals` and `base_quantity_decimals`. The JSON action and signed binary action must describe the same action.

Canonical unsigned payload:

```rust
string("NATIVE_CORE_TX_SIGNING_V1")
u32(1)                  // tx codec version
u32(696969)             // Native Core chain id (mainnet; testnet is 969696)
u64(nonce)
option<u64>(agent_epoch)
option<u64>(expires_after_ms)
action_bytes
```

The chain id is the target deployment's Native chain id (**mainnet: `696969`**, testnet: `969696`). It is folded into the signed bytes but is **never sent on the wire**, so it must equal the deployment you target — signing with the wrong chain id makes the API recover a different authority and reject the write.

Encoding rules:

* Integers are unsigned big-endian.
* `string` and byte vectors encode as `u32 byte_length` followed by raw bytes.
* `option<T>` encodes as `u8(0)` for `null`/omitted, or `u8(1)` followed by `T`.
* Hex values are raw bytes after removing the `0x` prefix.
* The signing digest is `keccak256(unsigned_payload_bytes)`.
* `signature` is `0x` + 65 bytes encoded as `r || s || v`; `v` may be `0/1` or `27/28`. High-s signatures are rejected.

The legacy scheme above applies to trading actions. Authorization-sensitive actions use EIP-712 (next section).

### EIP-712 signing (auth_scheme: "eip712")

Public `withdraw`, `settle`, `repay`, `approveAgent`, and `revokeAgent` must be submitted with `auth_scheme: "eip712"`. (The full cutover set also covers `deposit` and the operator `admin*` writes, which are not part of this public trading contract.) This is a **direct cutover**: the moment the new binary is live, legacy signatures over these actions are rejected (`legacy_signature_not_accepted`), and there is no config switch, height activation, or grace window — clients must switch at deploy. Conversely, `auth_scheme="eip712"` on any non-target action (`order`/`cancel`/`cancelAll`/`modify`/`batch`) is rejected (`eip712_not_allowed_for_action`), and an EIP-712 request may not carry `agent_epoch` (`eip712_agent_epoch_not_allowed`).

The signature covers an EIP-712 typed-data digest, not a binary payload. Clients sign the **v4** scheme, which is MetaMask-compatible: the domain is `EIP712Domain{name:"Native Core", version:"1", verifyingContract:0x0000…0000}` — **no `chainId`** — so a wallet can sign while connected to any EVM chain. The Native chain id is instead a signed message field, `nativeChainId`, so replay separation across environments is preserved. Each target action has its own primary type whose fields mirror the action, prefixed by the common fields `uint256 nativeChainId, uint256 authKind, uint256 authScope, uint256 nonce, bool expiresAfterMsPresent, uint256 expiresAfterMs`. `nativeChainId` is the Native Core chain id; `authKind` is `1` (single) and `authScope` is `0` for these public user actions. Amounts are signed as canonical atoms; addresses as `address`; an optional `cloid` as `bool cloidPresent` + `bytes16 cloid`. The presence flags keep an absent value distinct from an explicit `0`. The transaction **authority** is the recovered signer, exactly as for legacy single-signature actions.

A superseded **v3** EIP-712 scheme (domain included `chainId`; no `nativeChainId` field) is retained only for historical decode/replay and is **not accepted at submit**. Because `/trade` carries no codec-version field, a request whose signature was produced under the old v3 scheme is assembled as v4 and recovers a different address, so it fails with a signature/authority error — re-sign with the v4 scheme.

`withdraw` keeps an optional `cloid` at the protocol level (legacy WAL records may omit it), but the public API JSON requires `cloid`; the EIP-712 `cloidPresent` flag models the optionality.

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

```rust
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

```rust
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

```rust
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

```rust
u8(side)
u8(order_type)
u8(tif)
option<u64>(price)
u64(quantity)
option<bytes16>(cloid)
```

CancelAll action bytes:

```rust
u16(26)
u32(market_id)
```

`cancelAll` carries no `oid` and no `cloid`. The signed payload is a single `u32` market id; the JSON body must match (`{ "type": "cancelAll", "market_id": "<id>" }`).

Batch action bytes:

```rust
u16(18)
u32(item_count)
item_0
item_1
...
```

Each batch item starts with a `u8` item tag followed by the item payload without the top-level `u16` action tag:

```rust
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
  chainId: 696_969, // Native Core mainnet chain id (testnet: 969_696) — must match the deployment you sign for

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
const request = signer.sign(
  {
    type: "order",
    market_id: "2",
    side: "bid",
    order_type: "limit",
    tif: "alo",
    price: "3500.00",
    quantity: "1.0000",
    cloid: "0x11111111111111111111111111111111",
  },
  // Trading actions are signed by the API wallet (an agent key), so agent_epoch
  // is REQUIRED. Read it from POST /info `userAgents` (the `epoch` of the slot
  // whose `agent` is your API-wallet address). Omit it only for owner-signed actions.
  { agentEpoch: 3n },
);

await fetch(`${API_URL}/trade`, {
  method: "POST",
  headers: { "content-type": "application/json" },
  body: JSON.stringify(request),
});
```

`agent_epoch` is mandatory for these API-wallet–signed trading actions: an API wallet is always an active agent, so a `/trade` write that omits `agent_epoch` is treated as direct-owner mode and rejected with `DirectSignerIsActiveAgent`. Read the current epoch from [`userAgents`](post-info.md#useragents).

`/trade` is synchronous, so the response carries the outcome. `submission_status` is `accepted` (the transaction landed and executed), `rejected` (refused, or failed at execution), or `timeout`. `/trade` reports that the order landed, not its fill state — read [`orderStatus`](post-info.md#orderstatus) to see whether it rested or filled. See [POST /trade](post-trade.md) and [error responses](error-responses.md) for the full model.

Order outcome (the order landed):

```json
{
  "submission_status": "accepted",
  "tx_hash": "0x..."
}
```

Rejected response (node-admission code, returned verbatim):

```json
{
  "submission_status": "rejected",
  "tx_hash": "0x...",
  "error": {
    "code": "MinTradeSpotNtl"
  }
}
```

Request-shaping rejections usually have no `tx_hash` because canonical bytes were not assembled; admission and execution rejections include it. Rate-limit responses also include `error.retry_after_ms`. A `submission_status: "timeout"` means the outcome was not observed in time — either the wait budget elapsed (HTTP `200`, no `error`) or the submission could not be routed (`HandoffTimeout` / `HandoffBufferFull:*` / `HandoffMultipleActive` at HTTP `503`, or `node_unreachable: …` at HTTP `504`). The two cases need opposite handling — see [Handle outcomes & timeouts](handle-timeouts.md#reconciling-a-timeout).

Malformed or non-decodable JSON is handled by the API and returns the `TradeResponse` shape with `error.code = "invalid_json"`. This includes invalid action type tags and invalid field types. Negative numeric strings and malformed decimal strings are also `invalid_json` — they are rejected inside deserialization, before any field-specific check runs. Failures caught after decoding, such as a precision violation, return their specific code in the same response shape. All such responses include `x-trace-id`.

### `/trade` error codes

Every `/trade` `error.code` — request-shaping, gateway, node-admission, and execution — is cataloged in [Error responses](error-responses.md#full-trade-error-code-reference). Node-admission codes are returned **verbatim** (CamelCase) in top-level `error.code`. A lowercase execution code such as `tick` never appears there — for an order-ish action it comes back inside the [`response` envelope](post-trade.md#what-accepted-carries) while `submission_status` stays `accepted`.

Supported public top-level action types:

* `order`
* `cancel`
* `cancelAll`
* `modify`
* `batch`
* `withdraw` (user single-signature, EIP-712 `auth_scheme:"eip712"`; see [withdraw](post-trade.md#withdraw))
* `settle` (user single-signature, EIP-712 `auth_scheme:"eip712"`)
* `repay` (user single-signature, EIP-712 `auth_scheme:"eip712"`)
* `approveAgent` (owner single-signature, EIP-712 `auth_scheme:"eip712"`; see [approveAgent](post-trade.md#approveagent))
* `revokeAgent` (owner single-signature, EIP-712 `auth_scheme:"eip712"`)

Operator/accounting writes (`deposit`, `adminSetAccountingWithdrawTokens`, `admin*`, `setMultisigPolicy`, `addAsset`, `openMarket`, …) are also accepted by the API but require operator/admin authority and are not part of this public trading contract.
