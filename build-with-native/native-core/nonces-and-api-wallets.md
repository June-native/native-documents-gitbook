---
description: How a bot authorizes and replay-protects Native Core writes — the API-wallet agent key and the per-authority nonce.
---

# Nonces & API Wallets

Every `POST /trade` write is a client-signed transaction. Two things make it safe to run one from an unattended bot: the key it signs with can place and cancel orders but can **never move funds**, and each signed action carries a **nonce** that stops it from being replayed. This page covers both — the API-wallet (agent) key that signs, and the nonce that protects what it signs.

## Background — why writes need a nonce

The API does not authenticate a session; it recovers the signer from each signature and treats that recovered address as the transaction **authority**. Without a nonce, anyone who observed a signed `/trade` body could resend the exact bytes and replay your order. The nonce makes each signed action single-use: execution records consumed nonces per authority and rejects a repeat, so a captured request cannot be replayed and a retry you did not intend cannot double-fill.

## API wallets

Writes are authorized by an **API wallet**: a protocol-level *agent* key scoped only to placing and cancelling orders. It can trade your account's balance but can **never** move funds off Native. That scoping — plus the fact that it is revocable — is what makes it safe to embed in a bot.

You create one by generating an agent keypair locally and authorizing it with a single owner-signed `approveAgent` — sign it with your **main wallet** directly against the API, or let the **[Native web app](https://app.native.org/markets/ETH-USDT?agentWallets=agents)** do it and hand you a one-time **connection bundle** with the agent key. Deposit from your main wallet to create the trading account first. See [API Access](api-access.md#access-model-api-wallets) for the full flow and the bundle shape; revoke or rotate the agent any time.

Anything that moves value or manages agents is **owner-signed** and outside the API wallet's reach: `withdraw`, `settle`, `repay`, and the agent-lifecycle `approveAgent` / `revokeAgent` must be signed by your **main wallet** under `auth_scheme:"eip712"`. These *are* real `/trade` action types — the API accepts them directly — but the API-wallet (agent) key cannot sign them; sign them with your **main wallet**, directly against the API or through the web app. The API wallet's allowlist is exactly `order`, `cancel`, `cancelAll`, `modify`, and `batch`.

| Concept | Web app label | Signs | Can move funds? |
| --- | --- | --- | --- |
| **Agent** — API wallet | *API wallet* | `order` · `cancel` · `cancelAll` · `modify` · `batch` (legacy) | No |
| **Owner** — trading account | *Account* | `withdraw` · `settle` · `repay` · `approveAgent` · `revokeAgent` (EIP-712) | Yes |

The API-wallet setup and connection-bundle shape are on the API access page. If you integrate with the Python SDK, its quickstart walks the same web-app setup click-by-click.

{% content-ref url="api-access.md" %}
[api-access.md](api-access.md)
{% endcontent-ref %}

{% content-ref url="python-sdk/getting-started.md" %}
[getting-started.md](python-sdk/getting-started.md)
{% endcontent-ref %}

## Nonce mechanics

The nonce is a decimal-string `u64` **Unix millisecond timestamp** — use `Date.now()` / current Unix ms. It is validated **authority-scoped**: keyed on the recovered signer, so one API wallet is one nonce namespace. Execution enforces these rules against the committed block timestamp:

| Rule | Behavior |
| --- | --- |
| **Acceptance window** | A nonce must fall within `block_timestamp_ms - 2 days` through `block_timestamp_ms + 1 day`. Outside that window it is rejected. |
| **No duplicates** | A nonce already consumed for this authority is rejected. |
| **Retention** | Execution retains the latest **100** consumed nonces per authority. |
| **Monotonic-when-full** | Once the retained set is full, a new nonce must be **greater than the current minimum retained nonce**. |

Because the window is measured against the *block* timestamp and only the latest 100 are retained, a monotonic-from-now clock always satisfies every rule: each new millisecond timestamp is larger than everything retained and comfortably inside the window.

**Using the Python SDK?** It manages this for you: its nonce is a per-instance, lock-guarded, monotonic millisecond counter, so a single `Exchange` is safe to share across threads. The one rule to keep is to construct **one** `Exchange` per API wallet and share it — never one per worker, or two instances on the same key hand out colliding nonces. See [One Exchange per API wallet](python-sdk/core-concepts.md#one-exchange-per-api-wallet).

The nonce rides in the `/trade` envelope alongside the action, and it is one of the fields folded into the canonical signed payload — so tampering with it invalidates the signature.

{% content-ref url="post-trade.md" %}
[post-trade.md](post-trade.md)
{% endcontent-ref %}

{% content-ref url="transaction-signing.md" %}
[transaction-signing.md](transaction-signing.md)
{% endcontent-ref %}

## `agent_epoch`

An agent-signed request also carries `agent_epoch`, a decimal `u64` identifying the live approval **generation** (epoch) for the API wallet — distinct from its `slot_id` (`0`–`3`). The current epoch is not something you hardcode — it is resolved live from the `userAgents` read (the `epoch` of the slot holding your API-wallet address), so a stale value from an old bundle does no harm. If the epoch has rotated underneath you, node admission rejects the write with `AgentEpochMismatch`; the SDK catches that once, re-resolves the epoch from `userAgents`, and retries the same action under a fresh nonce. Omit `agent_epoch` for owner-signed requests.

## Safety

{% hint style="warning" %}
**Do not reuse a decommissioned address.** Once an authority's retained nonce set prunes, an old signed action from that address could re-enter the acceptance window and be replayed. Rotate to a fresh API wallet rather than reusing an address you have retired.
{% endhint %}

{% hint style="warning" %}
**Read by the owner address, not the agent address.** Pass your account (**owner**) address as `user` to every `POST /info` query — in the SDK that is `exchange.effective_account`. Querying by the API-wallet / agent address returns empty results, because orders act on the owner, not the signer.
{% endhint %}
