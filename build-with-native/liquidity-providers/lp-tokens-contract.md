# LP Tokens Contract

### Overview

Native LP Token contract is a yield-bearing vault that represent the shares and positions for all liquidity providers in Native.

The contract also helps distribute yield from the funding interests, collected from private market makers, who utilise the liquidity to facilitate trading with Credit-Based swaps of the Native trading engine.

Mind that the `balance` of the token directly represents the amount of underlying tokens in users' addresses. As well as the `totalSupply` number. To get shares, use `shares` and `totalShares` .

### **Inherits**

ERC20, Ownable, ReentrancyGuardTransient, Pausable

### State Variables <a href="#state-variables" id="state-variables"></a>

#### underlying <a href="#underlying" id="underlying"></a>

The underlying token that this LP token represents

```solidity
IERC20Metadata public underlying;
```

#### creditVault <a href="#creditvault" id="creditvault"></a>

The address of the credit vault contract

```solidity
address public creditVault;
```

#### \_decimals <a href="#decimals" id="decimals"></a>

The number of decimals for this token, matching the underlying token's decimals

```solidity
uint8 private _decimals;
```

#### earlyWithdrawFeeBips <a href="#earlywithdrawfeebips" id="earlywithdrawfeebips"></a>

Early withdrawal fee in basis points (1 bip = 0.01%)

_Applied to prevent front-running by users who deposit right before yield distribution and immediately redeem after_

```solidity
uint256 public earlyWithdrawFeeBips;
```

#### totalUnderlying <a href="#totalunderlying" id="totalunderlying"></a>

Total amount of underlying assets deposited by LPs

```solidity
uint256 public totalUnderlying;
```

#### totalShares <a href="#totalshares" id="totalshares"></a>

Total number of shares issued

```solidity
uint256 public totalShares;
```

#### minRedeemInterval <a href="#minredeeminterval" id="minredeeminterval"></a>

Minimum time interval between deposit and redeem (in seconds)

```solidity
uint256 public minRedeemInterval = 8 hours;
```

#### minDeposit <a href="#mindeposit" id="mindeposit"></a>

Minimum amount required for deposits

```solidity
uint256 public minDeposit;
```

#### shares <a href="#shares" id="shares"></a>

Mapping of user addresses to their share balances

```solidity
mapping(address => uint256) public shares;
```

#### lastDepositTimestamp <a href="#lastdeposittimestamp" id="lastdeposittimestamp"></a>

Mapping of user addresses to their last deposit timestamp

```solidity
mapping(address => uint256) public lastDepositTimestamp;
```

### Functions <a href="#functions" id="functions"></a>

#### constructor <a href="#constructor" id="constructor"></a>

```solidity
constructor(
    string memory _name,
    string memory _symbol,
    address _underlying,
    address _creditVault
) ERC20(_name, _symbol);
```

#### deposit <a href="#deposit" id="deposit"></a>

Deposit underlying tokens to mint LP tokens

```solidity
function deposit(
    uint256 amount
) external nonReentrant whenNotPaused returns (uint256 sharesToMint);
```

**Parameters**

| Name     | Type      | Description                            |
| -------- | --------- | -------------------------------------- |
| `amount` | `uint256` | Amount of underlying tokens to deposit |

**Returns**

| Name           | Type      | Description                   |
| -------------- | --------- | ----------------------------- |
| `sharesToMint` | `uint256` | Amount of shares to be minted |

#### redeem <a href="#redeem" id="redeem"></a>

Redeem LP tokens for underlying tokens

```solidity
function redeem(
    uint256 sharesToBurn
) external nonReentrant whenNotPaused returns (uint256 underlyingAmount);
```

**Parameters**

| Name           | Type      | Description              |
| -------------- | --------- | ------------------------ |
| `sharesToBurn` | `uint256` | Amount of shares to burn |

**Returns**

| Name               | Type      | Description                          |
| ------------------ | --------- | ------------------------------------ |
| `underlyingAmount` | `uint256` | Amount of underlying tokens received |

#### distributeYield <a href="#distributeyield" id="distributeyield"></a>

Distributes yield to LP token holders

_Can only be called by the credit vault_

```solidity
function distributeYield(
    uint256 yieldAmount
) external;
```

**Parameters**

| Name          | Type      | Description                   |
| ------------- | --------- | ----------------------------- |
| `yieldAmount` | `uint256` | Amount of yield to distribute |

#### balanceOf <a href="#balanceof" id="balanceof"></a>

Gets the underlying token balance of an account

```solidity
function balanceOf(
    address account
) public view override returns (uint256);
```

**Parameters**

| Name      | Type      | Description                          |
| --------- | --------- | ------------------------------------ |
| `account` | `address` | The address to check the balance for |

**Returns**

| Name     | Type      | Description                                                  |
| -------- | --------- | ------------------------------------------------------------ |
| `<none>` | `uint256` | The amount of underlying tokens the account effectively owns |

#### totalSupply <a href="#totalsupply" id="totalsupply"></a>

Gets the total supply of underlying tokens in the pool

```solidity
function totalSupply() public view override returns (uint256);
```

**Returns**

| Name     | Type      | Description                                                    |
| -------- | --------- | -------------------------------------------------------------- |
| `<none>` | `uint256` | The total amount of underlying tokens managed by this contract |

#### sharesOf <a href="#sharesof" id="sharesof"></a>

Gets the number of shares owned by an account

```solidity
function sharesOf(
    address account
) public view returns (uint256);
```

**Parameters**

| Name      | Type      | Description                     |
| --------- | --------- | ------------------------------- |
| `account` | `address` | The address to check shares for |

**Returns**

| Name     | Type      | Description                               |
| -------- | --------- | ----------------------------------------- |
| `<none>` | `uint256` | The number of shares owned by the account |

#### getUnderlyingByShares <a href="#getunderlyingbyshares" id="getunderlyingbyshares"></a>

Calculates the underlying token amount for a given number of shares

```solidity
function getUnderlyingByShares(
    uint256 sharesAmount
) public view returns (uint256);
```

**Parameters**

| Name           | Type      | Description                     |
| -------------- | --------- | ------------------------------- |
| `sharesAmount` | `uint256` | The number of shares to convert |

**Returns**

| Name     | Type      | Description                                   |
| -------- | --------- | --------------------------------------------- |
| `<none>` | `uint256` | The corresponding amount of underlying tokens |

#### getSharesByUnderlying <a href="#getsharesbyunderlying" id="getsharesbyunderlying"></a>

Calculates the number of shares for a given amount of underlying tokens

```solidity
function getSharesByUnderlying(
    uint256 underlyingAmount
) public view returns (uint256);
```

**Parameters**

| Name               | Type      | Description                                |
| ------------------ | --------- | ------------------------------------------ |
| `underlyingAmount` | `uint256` | The amount of underlying tokens to convert |

**Returns**

| Name     | Type      | Description                        |
| -------- | --------- | ---------------------------------- |
| `<none>` | `uint256` | The corresponding number of shares |

#### exchangeRate <a href="#exchangerate" id="exchangerate"></a>

Gets the current exchange rate between shares and underlying tokens

```solidity
function exchangeRate() public view returns (uint256);
```

**Returns**

| Name     | Type      | Description                                   |
| -------- | --------- | --------------------------------------------- |
| `<none>` | `uint256` | The exchange rate scaled by 1e18 (1:1 = 1e18) |

#### decimals <a href="#decimals" id="decimals"></a>

Gets the number of decimals for this token

```solidity
function decimals() public view override returns (uint8);
```

**Returns**

| Name     | Type    | Description                                           |
| -------- | ------- | ----------------------------------------------------- |
| `<none>` | `uint8` | The number of decimals, matching the underlying token |

#### pause <a href="#pause" id="pause"></a>

Pauses all deposit and redeem operations

_Can only be called by the owner_

```solidity
function pause() external onlyOwner;
```

#### unpause <a href="#unpause" id="unpause"></a>

Unpauses all deposit and redeem operations

_Can only be called by the owner_

```solidity
function unpause() external onlyOwner;
```

#### setMinDeposit <a href="#setmindeposit" id="setmindeposit"></a>

Sets the minimum deposit amount

_Can only be called by the owner_

```solidity
function setMinDeposit(
    uint256 _minDeposit
) external onlyOwner;
```

**Parameters**

| Name          | Type      | Description                |
| ------------- | --------- | -------------------------- |
| `_minDeposit` | `uint256` | New minimum deposit amount |

#### setMinRedeemInterval <a href="#setminredeeminterval" id="setminredeeminterval"></a>

Sets the minimum time interval required between deposit and redeem

_Can only be called by the owner_

```solidity
function setMinRedeemInterval(
    uint256 _interval
) external onlyOwner;
```

**Parameters**

| Name        | Type      | Description                     |
| ----------- | --------- | ------------------------------- |
| `_interval` | `uint256` | New minimum interval in seconds |

#### setEarlyWithdrawFeeBips <a href="#setearlywithdrawfeebips" id="setearlywithdrawfeebips"></a>

Sets the early withdrawal fee in basis points (BIPs)

_Can only be called by the owner_

```solidity
function setEarlyWithdrawFeeBips(
    uint256 _earlyWithdrawFeeBips
) external onlyOwner;
```

**Parameters**

| Name                    | Type      | Description                      |
| ----------------------- | --------- | -------------------------------- |
| `_earlyWithdrawFeeBips` | `uint256` | New early withdrawal fee in BIPs |

#### \_mintShares <a href="#mintshares" id="mintshares"></a>

```solidity
function _mintShares(address to, uint256 shareAmount) internal;
```

#### \_burnShares <a href="#burnshares" id="burnshares"></a>

```solidity
function _burnShares(address from, uint256 shareAmount) internal;
```

#### \_transferShares <a href="#transfershares" id="transfershares"></a>

```solidity
function _transferShares(address from, address to, uint256 _shares) internal;
```

#### \_transfer <a href="#transfer" id="transfer"></a>

Override ERC20's \_transfer to handle yield-bearing LP token transfers

_Since this is a yield-bearing token, the actual transfer is done by transferring shares rather than token amounts directly. The shares represent the user's proportion of the total underlying assets including yield._

```solidity
function _transfer(address from, address to, uint256 amount) internal override;
```

**Parameters**

| Name     | Type      | Description                             |
| -------- | --------- | --------------------------------------- |
| `from`   | `address` | The address to transfer from            |
| `to`     | `address` | The address to transfer to              |
| `amount` | `uint256` | The underlying token amount to transfer |

### Events <a href="#events" id="events"></a>

#### YieldDistributed <a href="#yielddistributed" id="yielddistributed"></a>

Event emitted when yield is distributed to LP holders

```solidity
event YieldDistributed(uint256 yieldAmount);
```

#### MinRedeemIntervalUpdated <a href="#minredeemintervalupdated" id="minredeemintervalupdated"></a>

Event emitted when minimum redeem interval is updated

```solidity
event MinRedeemIntervalUpdated(uint256 newInterval);
```

#### TransferShares <a href="#transfershares" id="transfershares"></a>

Event emitted when shares are transferred between addresses

```solidity
event TransferShares(address indexed from, address indexed to, uint256 shares);
```

#### SharesMinted <a href="#sharesminted" id="sharesminted"></a>

Event emitted when new shares are minted

```solidity
event SharesMinted(address indexed user, uint256 shares, uint256 underlyingAmount);
```

#### SharesBurned <a href="#sharesburned" id="sharesburned"></a>

Event emitted when shares are burned

```solidity
event SharesBurned(address indexed user, uint256 shares, uint256 underlyingAmount);
```

#### MinDepositUpdated <a href="#mindepositupdated" id="mindepositupdated"></a>

Event emitted when minimum deposit amount is updated

```solidity
event MinDepositUpdated(uint256 oldAmount, uint256 newAmount);
```

#### EarlyWithdrawFeeBipsUpdated <a href="#earlywithdrawfeebipsupdated" id="earlywithdrawfeebipsupdated"></a>

Event emitted when early withdraw fee is updated

```solidity
event EarlyWithdrawFeeBipsUpdated(uint256 oldFeeBips, uint256 newFeeBips);
```
