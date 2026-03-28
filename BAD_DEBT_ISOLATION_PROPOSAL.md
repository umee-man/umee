# Proposal: Bad Debt Isolation & Repayment Mechanism

> [!CAUTION]
> **The Problem:** In the current `x/leverage` architecture, when a user's collateral is fully liquidated but borrow balances remain, the remaining debt is flagged as "Bad Debt". However, this debt remains tracked as standard borrow shares. Because interest in Umee is calculated globally via an `InterestScalar` acting on all borrow shares, **bad debt continues to accrue interest indefinitely.** This infinitely increases the protocol's deficit and artificially inflates the utilization rate, driving up Borrow APY for healthy users.

## 1. Executive Summary

This proposal suggests an architectural shift to isolate bad debt. By completely removing uncollateralized debt from the standard borrow tracking system and moving it to a static, global ledger, the protocol can freeze the debt size (stopping interest accrual) while continuing to pay it down programmatically from newly accrued protocol reserves. 

## 2. Proposed Architecture

The solution requires two primary state transitions: isolating the debt to freeze its value, and updating the global sweep mechanism to pay it down efficiently.

### A. The Isolation Mechanism (Freezing the Debt)

Currently, the liquidation hook `postLiquidate` flags an `address|denom` when collateral hits zero. 

**Proposed Change:**
Instead of just flagging the address, the protocol should execute a **debt write-off** during the liquidation state transition:
1. **Calculate Final Debt:** Convert the user's remaining borrow shares into a static base token amount (e.g., `100 USDC`).
2. **Erase the Position:** Delete the user's borrow position entirely so they no longer hold borrow shares.
3. **Decrease Total Borrowed:** Subtract the static amount from the leverage module's `TotalBorrowed` state. *This immediately fixes the utilization rate calculation, ensuring healthy borrowers are not penalized with high interest rates caused by dead liquidity.*
4. **Move to Isolated State:** Add the static amount to a new global variable: `IsolatedBadDebt[denom]`.

### B. Modified Reserve Sweep Procedure (Automatic Repayment)

Currently, the `SweepBadDebts` function (called in `EndBlocker`) iterates over all flagged `address|denom` pairs across the network, attempting to repay each individually. This is highly inefficient.

**Proposed Change:**
The `SweepBadDebts` function should be refactored to operate globally rather than individually:
1. In `EndBlocker`, the protocol checks if `IsolatedBadDebt[denom] > 0`.
2. It checks if the module has accumulated `Reserves[denom] > 0` from recent interest accruals.
3. If both are true, it calculates the repayment amount: `Repayment = min(IsolatedBadDebt, Reserves)`.
4. It subtracts `Repayment` from both the `IsolatedBadDebt` tracker and the `Reserves` pool in a single `O(1)` state operation.

## 3. Key Benefits

| Benefit Area | Description |
| :--- | :--- |
| **Protocol Solvency** | Prevents bad debt from growing exponentially via compound interest, allowing the protocol's reserves to actually catch up and eliminate the deficit. |
| **Market Fairness** | Removing dead debt from `TotalBorrowed` restores accurate utilization rates. Healthy borrowers will no longer pay artificially inflated APYs caused by bad actors. |
| **Performance** | Replacing an `O(N)` network-wide iteration over individual bad debt addresses with a single `O(1)` global math operation saves significant computation overhead in the `EndBlocker`. |
| **User Experience** | Liquidated users no longer see an infinitely growing negative balance that they can never realistically repay. |

## 4. Implementation Steps

> [!TIP]
> **Developer Action Items:**
> 1. Introduce a new KVStore prefix for `IsolatedBadDebt` structured by `denom`.
> 2. Update `x/leverage/keeper/liquidate.go` -> `postLiquidate()` to wipe the borrow position and map it to `IsolatedBadDebt`.
> 3. Update `x/leverage/keeper/iter.go` -> `SweepBadDebts()` to utilize the global `O(1)` repayment logic.
> 4. Ensure `ReserveFactor` continues accruing from healthy `TotalBorrowed` balances to fund the repayment.
