# Triage Defense Playbook: Infinite 'Initial Deposit' Tax

## 1. Triage Classification
*   **Vulnerability Type**: Economic Logic / Persistence Failure
*   **Severity**: High (Permanent Fund Leakage)
*   **Impacted Component**: `StakedUSDai.sol` -> `convertToShares` / `convertToAssets`

## 2. Evidence of Vulnerability
The vulnerability is visible in the asymmetric handling of the "initial deposit" state.
*   **Location**: `StakedUSDai.sol#L422`
*   **Logic**: The state is determined by a live balance check (`totalShares() < 1e6`) rather than a persistent lifecycle flag.

## 3. Anticipated Developer Counter-Arguments
*   *"The amount (1e6 shares) is negligible."*
    *   **Defense**: The severity is not the dollar amount, but the **failure of the accounting invariant**. Direct theft of any amount of user principal due to a recurring logic error is a High severity finding in professional audits.
*   *"Users should just check the share price before depositing."*
    *   **Defense**: The vault is an ERC-4626 implementation. Users expect standard behavior where the exchange rate is predictable and not subject to a hidden "entry fee" that resets.

## 4. Developer Masking Analysis
This bug was likely masked by:
1.  **Standard Pattern Bias**: Developers copied the "Locked Shares" pattern from other protocols (like OpenZeppelin or Solady) but failed to account for the unique **asynchronous redemption queue** in USDai which allows `totalShares` to drop.
2.  **Test Coverage Gap**: Standard tests usually only verify the *first* deposit and assume the vault remains populated thereafter.

## 5. Critical Invariants to Monitor
*   **Invariant-01**: `LOCKED_SHARES` should be deducted at most once per contract lifetime.
*   **Invariant-02**: `totalAssets()` / `totalShares()` should never lead to a share price increase that wasn't caused by yield or donations.
