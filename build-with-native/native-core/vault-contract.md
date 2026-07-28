---
description: The DepositWithdrawVault contract surface an integrator calls — deposit entry points, read gates, the Deposit event, and revert reasons.
---

# Vault Contract

`DepositWithdrawVault` is the EVM side of Native Core funding. One instance is deployed per supported source chain. It takes custody of deposited ERC20s and releases them on withdrawal.

You use three parts of it: the two deposit entry points, the view calls that decide whether a deposit will succeed, and `usedNonces` to confirm a withdrawal was paid out. The end-to-end flows are in [Deposit & withdraw](deposit-and-withdraw.md).

## Addresses

Resolve the vault at runtime from [`accountingDepositContracts`](post-info.md#accountingdepositcontracts). Vaults are redeployed, and a deposit sent to a superseded vault is not credited.

```bash
curl -sS -X POST "https://api.native.org/info" \
  -H 'content-type: application/json' \
  -d '{"type":"accountingDepositContracts"}'
```

The response is keyed by `src_chain_id`, so the same call covers every chain and stays correct across redeployments. Testnet vaults come from the same query against `https://api-test.native.org`.

## Writing

### deposit

```solidity
function deposit(address token, uint256 amount, uint256 actionFlag)
    external payable returns (uint256 wNLPAmount);
```

Pulls `amount` of `token` from `msg.sender` and credits that address's Native Core account. Requires an ERC20 allowance for the vault. Pass `actionFlag: 0`.

`msg.value` carries the one-time account activation fee and must be `0` for an account that already exists. The contract does not validate it — an incorrect value produces a successful transaction that later fails validation off-chain. See [Deposit & withdraw](deposit-and-withdraw.md#2-size-the-activation-fee).

`amount` is in the token's own decimals and must be a whole number of units at `minDepositDecimalByUnderlying(token)` decimal places, which is `8` for every listed token. An 18-decimal token therefore requires `amount % 10**10 == 0`.

### depositFor

```solidity
function depositFor(address token, uint256 amount, address user, uint256 actionFlag)
    external payable returns (uint256 wNLPAmount);
```

Identical to `deposit`, except it credits `user` while the tokens and allowance come from `msg.sender`. Use it to fund an end user from your own contract or relayer.

The activation fee follows `user`, not the caller: read [`accountStatus`](post-info.md#accountstatus) for `user` to decide `msg.value`.

### withdraw

```solidity
function withdraw(
    address token, address user, uint256 amount,
    uint256 nonce, uint256 deadline, uint256 fee, bytes[] signatures
) external;
```

Operator-only. The call carries an M-of-N signature set from Native's signer network, and Native submits it after your `withdraw` action executes on Native Core. It is listed here so you recognize the release transaction — a call from any other sender reverts.

## Reading

Every deposit-blocking condition is readable before you build a transaction. Checking them turns a revert into an explainable error.

| Call                                            | Returns    | Use                                                        |
| ----------------------------------------------- | ---------- | ---------------------------------------------------------- |
| `getSupportedUnderlyings()`                      | `address[]` | Tokens this vault accepts                                   |
| `isSupportedUnderlying(address token)`           | `bool`     | Same check for one token                                    |
| `isDepositPaused(address token)`                 | `bool`     | Per-token deposit pause                                     |
| `emergencyPaused()`                              | `bool`     | Vault-wide pause                                            |
| `minDepositDecimalByUnderlying(address token)`   | `uint256`  | Maximum decimal places a deposit amount may carry — `8`     |
| `wrappedNLPByUnderlying(address token)`          | `address`  | The wrapped-NLP token minted against the deposit            |
| `depositNonce()`                                 | `uint256`  | Deposits recorded by this vault so far                      |
| `getDepositRecord(uint256 nonce)`                | `(address token, address user, uint256 amount, uint256 msgValue)` | Look up a deposit by its nonce; `msgValue` is the activation fee it carried |
| `usedNonces(address user, uint256 nonce)`        | `bool`     | `true` once a withdrawal to `user` with that nonce has been released |

`usedNonces` is the contract's permanent replay guard: set in the same transaction that transfers the tokens, never cleared, keyed by the withdrawal's payout address and the `withdraw_nonce` you signed. That makes it the confirmation surface for a withdrawal — poll it instead of parsing logs.

## Deposit event

Emitted once per successful deposit. `nonce` is the vault's per-deposit id and ties the transaction to Native Core's credit record.

```solidity
event Deposit(
    address indexed underlying,
    address         wrappedNLP,
    address indexed user,
    uint256         amount,
    uint256         wNLPAmount,
    uint256         fee,
    uint256 indexed actionFlag,
    uint256         nonce
);
```

Topic 0 is `0x259af91af89c9a6b13d53607d57f43b151235f69d54d2339133e57cfb62bf4c5`.

* `user` is the credited address — the caller for `deposit`, the `user` argument for `depositFor`.
* `amount` is in the token's decimals; Native credits the same value rescaled to the asset's 8-decimal `balance_decimals`.
* `fee` is the `msg.value` the transaction carried — `0` for a deposit into an existing account.
* `nonce` is what you match against `deposit_nonce` in [`deposits`](post-info.md#deposits).

To confirm a withdrawal, use `usedNonces` and the destination token's ERC20 `Transfer` rather than the vault's `Withdraw` event — `usedNonces` is stable across vault redeployments and unambiguous.

## Reverts

Custom errors on the deposit path, in the order you are likely to hit them:

| Error                                | Meaning                                                              |
| ------------------------------------ | -------------------------------------------------------------------- |
| `UnsupportedUnderlying`              | `token` is not listed on this vault                                   |
| `InvalidUnderlying`                  | `token` is not a valid underlying for this call                       |
| `DepositPaused`                      | Deposits are paused for `token`                                       |
| `InvalidAmount`                      | `amount` carries more decimal places than `minDepositDecimalByUnderlying` |
| `ZeroAmount`                         | `amount` is `0`                                                       |
| `ZeroAddress`                        | An address argument is the zero address                               |
| `SafeERC20FailedOperation(address)`  | The ERC20 transfer failed — usually an insufficient allowance or balance |
| `ReentrancyGuardReentrantCall`       | Re-entrant call into the vault                                        |

Decode these with the ABI below so users see a reason rather than a four-byte selector.

## ABI fragment

The subset of the vault ABI that [Deposit & withdraw](deposit-and-withdraw.md) calls. Paste it into your client:

```json
[
  { "type": "function", "name": "deposit", "stateMutability": "payable",
    "inputs": [
      { "name": "token", "type": "address" },
      { "name": "amount", "type": "uint256" },
      { "name": "actionFlag", "type": "uint256" }
    ],
    "outputs": [{ "name": "wNLPAmount", "type": "uint256" }] },

  { "type": "function", "name": "depositFor", "stateMutability": "payable",
    "inputs": [
      { "name": "token", "type": "address" },
      { "name": "amount", "type": "uint256" },
      { "name": "user", "type": "address" },
      { "name": "actionFlag", "type": "uint256" }
    ],
    "outputs": [{ "name": "wNLPAmount", "type": "uint256" }] },

  { "type": "function", "name": "getSupportedUnderlyings", "stateMutability": "view",
    "inputs": [], "outputs": [{ "name": "", "type": "address[]" }] },
  { "type": "function", "name": "isSupportedUnderlying", "stateMutability": "view",
    "inputs": [{ "name": "", "type": "address" }], "outputs": [{ "name": "", "type": "bool" }] },
  { "type": "function", "name": "isDepositPaused", "stateMutability": "view",
    "inputs": [{ "name": "", "type": "address" }], "outputs": [{ "name": "", "type": "bool" }] },
  { "type": "function", "name": "emergencyPaused", "stateMutability": "view",
    "inputs": [], "outputs": [{ "name": "", "type": "bool" }] },
  { "type": "function", "name": "minDepositDecimalByUnderlying", "stateMutability": "view",
    "inputs": [{ "name": "", "type": "address" }], "outputs": [{ "name": "", "type": "uint256" }] },
  { "type": "function", "name": "usedNonces", "stateMutability": "view",
    "inputs": [{ "name": "", "type": "address" }, { "name": "", "type": "uint256" }],
    "outputs": [{ "name": "", "type": "bool" }] },
  { "type": "function", "name": "getDepositRecord", "stateMutability": "view",
    "inputs": [{ "name": "nonce", "type": "uint256" }],
    "outputs": [{ "name": "record", "type": "tuple", "components": [
      { "name": "token", "type": "address" },
      { "name": "user", "type": "address" },
      { "name": "amount", "type": "uint256" },
      { "name": "msgValue", "type": "uint256" }
    ] }] },

  { "type": "event", "name": "Deposit", "anonymous": false,
    "inputs": [
      { "name": "underlying", "type": "address", "indexed": true },
      { "name": "wrappedNLP", "type": "address", "indexed": false },
      { "name": "user", "type": "address", "indexed": true },
      { "name": "amount", "type": "uint256", "indexed": false },
      { "name": "wNLPAmount", "type": "uint256", "indexed": false },
      { "name": "fee", "type": "uint256", "indexed": false },
      { "name": "actionFlag", "type": "uint256", "indexed": true },
      { "name": "nonce", "type": "uint256", "indexed": false }
    ] },

  { "type": "error", "name": "UnsupportedUnderlying", "inputs": [] },
  { "type": "error", "name": "InvalidUnderlying", "inputs": [] },
  { "type": "error", "name": "DepositPaused", "inputs": [] },
  { "type": "error", "name": "InvalidAmount", "inputs": [] },
  { "type": "error", "name": "ZeroAmount", "inputs": [] },
  { "type": "error", "name": "ZeroAddress", "inputs": [] },
  { "type": "error", "name": "ReentrancyGuardReentrantCall", "inputs": [] },
  { "type": "error", "name": "SafeERC20FailedOperation",
    "inputs": [{ "name": "token", "type": "address" }] }
]
```
