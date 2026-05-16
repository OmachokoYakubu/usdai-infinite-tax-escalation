# Remediation Strategy: Infinite Locked Shares Tax

## Vulnerability Overview
The `StakedUSDai` vault allows the `LOCKED_SHARES` (inflation protection) to be re-triggered whenever `totalShares()` drops below `1e6`. This happens because redemptions eventually remove shares from the `totalSupply` and `pending` queue, resetting the state to "initial deposit".

## Mitigation Plan

### 1. Persistent Initialization Flag
The most robust fix is to ensure that the `LOCKED_SHARES` are only deducted once in the vault's entire history.

**Proposed Change:**
Add a `bool` flag in `StakedUSDaiStorage.sol` to track if the initial tax has been collected.

```diff
// src/StakedUSDaiStorage.sol

struct StakedUSDaiStorageLayout {
    // ... existing fields ...
+   bool initialSharesLocked;
}
```

**Implementation in `convertToShares`:**

```solidity
function convertToShares(uint256 assets) public view returns (uint256) {
    bool initialDeposit = !_getInitialSharesLocked(); // Check persistent flag

    uint256 shares = ((assets * FIXED_POINT_SCALE) / depositSharePrice());

    if (initialDeposit && shares <= LOCKED_SHARES) revert InvalidAmount();

    return shares - (initialDeposit ? LOCKED_SHARES : 0);
}
```

**Implementation in `_deposit`:**

```solidity
function _deposit(...) internal ... {
    uint256 shares = convertToShares(amount);
    
    // ... existing logic ...

+   if (!_getInitialSharesLocked()) {
+       _setInitialSharesLocked(true);
+   }
    
    _mint(receiver, shares);
    // ...
}
```

### 2. Alternative: Permanent Dead-Address Minting
Instead of deducting the shares from the user's return value, the vault could mint the `LOCKED_SHARES` to a dead address (e.g., `address(0xdead)`) during the very first deposit. This makes `totalShares()` stay above the threshold forever.

**Pros:**
*   Standard ERC4626 pattern (like OpenZeppelin's `_decimalsOffset` or common inflation protection).
*   Simpler logic in `convertToShares`.

**Cons:**
*   Requires a one-time slightly higher gas cost for the first depositor.

## Verification of Fix
After applying the fix, the `PoC_Infinite_Tax.t.sol` should be updated to expect Charlie's shares to be the full amount (no tax) even if the vault was previously emptied.
