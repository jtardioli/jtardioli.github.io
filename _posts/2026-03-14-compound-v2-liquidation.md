---
layout: post
title: 'Compound V2: Liquidation'
date: 2026-03-14 19:32 -0400
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, liquidation, liquidate, seize, collateral]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 8
---

> This article is part of the Compound V2 series. See the [series index](/posts/compound-v2-introduction/#series-index) for a full list of articles.

This article walks through liquidation in Compound's `cToken` contract. Liquidation is how the protocol recovers bad debt: when a borrower's position becomes insolvent, a third party called the liquidator can repay part of their debt in exchange for a discounted portion of their collateral.

Unlike minting, redeeming, borrowing, and repaying, liquidation spans two separate `cToken` contracts. Understanding why requires a quick recap of how Compound's markets work.

---

## Why Liquidation Involves Two Markets

In Compound, supplying and borrowing happen in separate markets. A user might supply ETH to the ETH market, receiving `cETH` as their collateral receipt, and then borrow USDC from the USDC market. These are two separate `cToken` contracts.

When that position becomes insolvent, the liquidator repays some of the borrower's USDC debt in the borrow market, and in exchange seizes some of the borrower's `cETH` in the collateral market. Because two contracts are involved, both need to have their interest accrued before the liquidation can proceed.

![Liquidation across two markets](/assets/img/protocols/compoundv2/liquidation-across-markets.png)

---

## liquidateBorrowInternal

```
function liquidateBorrowInternal(address borrower, uint repayAmount, CTokenInterface cTokenCollateral) internal nonReentrant {
    accrueInterest();

    uint error = cTokenCollateral.accrueInterest();
    if (error != NO_ERROR) {
        revert LiquidateAccrueCollateralInterestFailed(error);
    }

    // liquidateBorrowFresh emits borrow-specific logs on errors
    liquidateBorrowFresh(msg.sender, borrower, repayAmount, cTokenCollateral);
}
```

`liquidateBorrowInternal` is the entry point. It calls `accrueInterest` on the borrow market, then calls `accrueInterest` on the collateral market via an external call to `cTokenCollateral`. Both markets must be up to date before any calculations proceed. If the collateral market accrual fails, the liquidation reverts.

---

## liquidateBorrowFresh

### Sanity Checks

```
if (borrower == liquidator) {
    revert LiquidateLiquidatorIsBorrower();
}

if (repayAmount == 0) {
    revert LiquidateCloseAmountIsZero();
}

if (repayAmount == type(uint).max) {
    revert LiquidateCloseAmountIsUintMax();
}
```

Three guards run before any state changes. The borrower cannot liquidate themselves. The repay amount cannot be zero. Unlike `repayBorrowFresh`, passing `type(uint).max` is explicitly rejected here rather than being treated as a convenience shorthand for full repayment. This is intentional: liquidators are only permitted to repay up to the close factor, a protocol parameter that caps how much of a position can be liquidated in a single transaction. Allowing `type(uint).max` would bypass that cap.

### Repaying the Debt

```
uint actualRepayAmount = repayBorrowFresh(liquidator, borrower, repayAmount);
```

The liquidator repays part of the borrower's debt by calling `repayBorrowFresh` directly. This is the same function covered in the repaying article. The liquidator is the payer, the borrower is the debtor, and the actual amount transferred is returned for use in the next step.

### Calculating Collateral to Seize

```
(uint amountSeizeError, uint seizeTokens) = comptroller.liquidateCalculateSeizeTokens(
    address(this),
    address(cTokenCollateral),
    actualRepayAmount
);
require(amountSeizeError == NO_ERROR, "LIQUIDATE_COMPTROLLER_CALCULATE_AMOUNT_SEIZE_FAILED");
```

The comptroller calculates how many `cTokens` from the collateral market the liquidator is entitled to seize. This calculation is internal to the comptroller and will be covered separately. The key point is that `seizeTokens` is denominated in the collateral `cToken`, not in underlying. The liquidator receives `cTokens`, not the raw collateral asset.

The amount is bounded by the close factor: there is only so much of a position that can be liquidated at once. If a borrower owes `$100,000` but their position only needs `$5,000` repaid to become solvent again, allowing a full liquidation would be punitive. The close factor enforces a ceiling so that liquidators can only take what is necessary to restore solvency.

### Collateral Balance Check

```
require(cTokenCollateral.balanceOf(borrower) >= seizeTokens, "LIQUIDATE_SEIZE_TOO_MUCH");
```

A safety check to confirm the borrower actually holds enough collateral `cTokens` to cover the seizure.

### Seizing the Collateral

```
if (address(cTokenCollateral) == address(this)) {
    seizeInternal(address(this), liquidator, borrower, seizeTokens);
} else {
    require(cTokenCollateral.seize(liquidator, borrower, seizeTokens) == NO_ERROR, "token seizure failed");
}
```

It is possible for the borrow market and the collateral market to be the same contract, for example if someone supplied and borrowed the same asset. In that case `seizeInternal` is called directly on the current contract to avoid making an external call back into itself. In the more common case where they are different contracts, `seize` is called on the collateral market, which in turn calls `seizeInternal`.

---

## seize and seizeInternal

`seize` is the external entry point called by the borrow market on the collateral market contract. It simply delegates to `seizeInternal`, passing `msg.sender` as the `seizerToken`. Using `msg.sender` rather than a parameter is critical: it prevents a malicious caller from claiming to be a different market and seizing collateral they are not entitled to.

```
function seize(address liquidator, address borrower, uint seizeTokens) override external nonReentrant returns (uint) {
    seizeInternal(msg.sender, liquidator, borrower, seizeTokens);
    return NO_ERROR;
}
```

### Protocol Fee Calculation

```
uint protocolSeizeTokens = mul_(seizeTokens, Exp({mantissa: protocolSeizeShareMantissa}));
uint liquidatorSeizeTokens = seizeTokens - protocolSeizeTokens;
Exp memory exchangeRate = Exp({mantissa: exchangeRateStoredInternal()});
uint protocolSeizeAmount = mul_ScalarTruncate(exchangeRate, protocolSeizeTokens);
uint totalReservesNew = totalReserves + protocolSeizeAmount;
```

The liquidator does not receive 100% of the seized collateral. A small portion goes to the protocol reserves, governed by `protocolSeizeShareMantissa`. The total seized `cTokens` are split: `protocolSeizeTokens` go to reserves and `liquidatorSeizeTokens` go to the liquidator.

The protocol's share is then converted from `cTokens` to underlying using the exchange rate, so that `totalReserves` can be updated in underlying terms. `totalReserves` is denominated in underlying, not in `cTokens`.

### Updating State

```
totalReserves = totalReservesNew;
totalSupply = totalSupply - protocolSeizeTokens;
accountTokens[borrower] = accountTokens[borrower] - seizeTokens;
accountTokens[liquidator] = accountTokens[liquidator] + liquidatorSeizeTokens;
```

Four state variables are updated. `totalReserves` increases by the protocol's share in underlying terms. `totalSupply` decreases by `protocolSeizeTokens` because those `cTokens` are effectively burned: they are converted into underlying and added to reserves rather than transferred to anyone. The borrower's `cToken` balance decreases by the full `seizeTokens`. The liquidator's `cToken` balance increases by `liquidatorSeizeTokens`, their share after the protocol fee.

---

## Conclusion

Liquidation is the most complex operation in this series because it crosses two markets and involves four distinct parties: the borrower, the liquidator, the borrow market, and the collateral market. The flow is:

1. Accrue interest on both markets
2. Verify the liquidation is permitted and within bounds
3. The liquidator repays part of the borrower's debt via `repayBorrowFresh`
4. The comptroller calculates how many collateral `cTokens` that entitles the liquidator to
5. Those `cTokens` are seized from the borrower, with a portion going to protocol reserves and the rest to the liquidator

The seized collateral is always in `cToken` form, not underlying. The liquidator receives `cTokens` that they can later redeem.
