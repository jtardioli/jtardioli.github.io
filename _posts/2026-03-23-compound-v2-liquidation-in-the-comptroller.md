---
layout: post
title: 'Compound V2: Comptroller Liquidation'
date: 2026-03-23 10:00 -0500
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, comptroller, liquidation, close factor]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 16
---

> This article is part of the Compound V2 series. See the [series index](/posts/compound-v2-introduction/#series-index) for a full list of articles.

The previous article covered the Comptroller's permission hooks but skipped the liquidation-related functions. This article walks through the three that were left out: `liquidateBorrowAllowed`, `seizeAllowed`, and `liquidateCalculateSeizeTokens`.

---

## `liquidateBorrowAllowed`

This function decides whether a liquidation can proceed, before any state changes happen.

### Market Listing Check

```
if (!markets[cTokenBorrowed].isListed || !markets[cTokenCollateral].isListed) {
    return uint(Error.MARKET_NOT_LISTED);
}
```

Both the borrowed market and the collateral market must be listed in the Comptroller. If either one has not been registered, the liquidation is blocked.

### Fetching the Borrow Balance

```
uint borrowBalance = CToken(cTokenBorrowed).borrowBalanceStored(borrower);
```

The borrower's debt is retrieved in underlying terms. This value feeds into both the deprecated market path and the close factor calculation further down.

### Deprecated Market Path

```
if (isDeprecated(CToken(cTokenBorrowed))) {
    require(borrowBalance >= repayAmount, "Can not repay more than the total borrow");
}
```

Compound governance can mark a market as deprecated when it wants to wind it down. Three conditions must hold:

- `collateralFactor == 0` -> the asset can no longer be used as collateral
- `borrowGuardianPaused == true` -> no new borrows are allowed
- `reserveFactor == 1e18` -> 100% of interest goes to reserves, suppliers earn nothing

Once marked as deprecated, the only check is that the repay amount does not exceed the debt. Even healthy positions can be liquidated. Liquidators still receive the incentive bonus, so there is profit in it regardless. This lets governance clean out winding-down markets before positions go stale.

### Non-Deprecated Market Path: Shortfall Requirement

```
(Error err, , uint shortfall) = getAccountLiquidityInternal(borrower);
if (err != Error.NO_ERROR) {
    return uint(err);
}

if (shortfall == 0) {
    return uint(Error.INSUFFICIENT_SHORTFALL);
}
```

For non-deprecated markets, the borrower must have a shortfall. This calls `getAccountLiquidityInternal` rather than the `getHypotheticalAccountLiquidityInternal` covered in the account liquidity article. Internally it just calls the same function with zeroed-out parameters, so no hypothetical action is simulated. If the account is solvent and `shortfall` is zero, no liquidation is allowed.

### Close Factor Cap

```
uint maxClose = mul_ScalarTruncate(Exp({mantissa: closeFactorMantissa}), borrowBalance);
if (repayAmount > maxClose) {
    return uint(Error.TOO_MUCH_REPAY);
}
```

The close factor limits how much of a borrower's debt can be repaid in a single liquidation, expressed as a percentage of the total borrow balance.

Say a borrower owes 100,000 USDC and the close factor is 50%. The maximum a liquidator can repay in one transaction would be

```
maxClose = 100,000 * 0.50 = 50,000 USDC
```

This maximum amount is the `maxClose`. The shortfall amount plays no role in the `maxClose` calculation. A borrower with 100,000 USDC of debt and a shortfall of just $1 can still be liquidated for up to 50,000 USDC. This might seem harsh, but the protocol needs liquidations profitable enough to attract liquidators. Too little incentive and positions go unliquidated, pushing the protocol toward insolvency.

---

## `seizeAllowed`

After the liquidator repays part of the debt, the protocol seizes collateral. `seizeAllowed` runs before that seizure happens.

### Pause Check

```
require(!seizeGuardianPaused, "seize is paused");
```

Seizing can be paused globally as an emergency lever.

### Market Listing

```
if (!markets[cTokenCollateral].isListed || !markets[cTokenBorrowed].isListed) {
    return uint(Error.MARKET_NOT_LISTED);
}
```

Both markets must be listed, same check as `liquidateBorrowAllowed`.

### Comptroller Mismatch

```
if (CToken(cTokenCollateral).comptroller() != CToken(cTokenBorrowed).comptroller()) {
    return uint(Error.COMPTROLLER_MISMATCH);
}
```

Both cToken contracts must reference the same Comptroller. `liquidateBorrowAllowed` and `seizeAllowed` exist separately because each gets called by a different market. The borrow market calls `liquidateBorrowAllowed`, the collateral market calls `seizeAllowed`. In practice they are usually the same Comptroller, but the architecture does not assume that. This check makes sure they agree on who is in charge.

### COMP Distribution

```
updateCompSupplyIndex(cTokenCollateral);
distributeSupplierComp(cTokenCollateral, borrower);
distributeSupplierComp(cTokenCollateral, liquidator);
```

The COMP reward flywheel is updated for both the borrower and the liquidator. Collateral cTokens are about to change hands, so accrued rewards need to be settled before the balances shift. The flywheel will be covered in a later article.

---

## `liquidateCalculateSeizeTokens`

`liquidateBorrowAllowed` caps how much the liquidator can repay. This function determines how many cTokens they get in return.

### Fetching Prices

```
uint priceBorrowedMantissa = oracle.getUnderlyingPrice(CToken(cTokenBorrowed));
uint priceCollateralMantissa = oracle.getUnderlyingPrice(CToken(cTokenCollateral));
if (priceBorrowedMantissa == 0 || priceCollateralMantissa == 0) {
    return (uint(Error.PRICE_ERROR), 0);
}
```

Oracle prices are fetched for both assets in USD terms. A zero price would corrupt the seizure calculation, so the function returns an error if either is missing.

### The Seizure Formula

The formula the function implements is

```
seizeTokens = actualRepayAmount * liquidationIncentive * priceBorrowed
              --------------------------------------------------------
                          priceCollateral * exchangeRate
```

The numerator converts the repaid amount into its USD value and scales it up by the liquidation incentive. The denominator converts from USD into collateral cTokens.

### The Implementation

```
numerator = mul_(Exp({mantissa: liquidationIncentiveMantissa}), Exp({mantissa: priceBorrowedMantissa}));
denominator = mul_(Exp({mantissa: priceCollateralMantissa}), Exp({mantissa: exchangeRateMantissa}));
ratio = div_(numerator, denominator);

seizeTokens = mul_ScalarTruncate(ratio, actualRepayAmount);
```

The mantissa-wrapped `Exp` structs and `mul_` calls and optimizing the order of operations for solidity make this harder to read than it needs to be. The worked example below makes the actual math clear.

### Worked Example

Suppose a liquidator repays 5,000 USDC of a borrower's debt and the collateral is ETH.

```
actualRepayAmount      = 5,000
priceBorrowed (USDC)   = 1
priceCollateral (ETH)  = 2,500
liquidationIncentive   = 1.08  (8% bonus)
exchangeRate (cETH)    = 0.02  (1 cETH = 0.02 ETH)
```

```
1. Convert repayment to USD value
   5,000 * 1 = $5,000

2. Apply the liquidation incentive
   $5,000 * 1.08 = $5,400

3. Convert to ETH
   $5,400 / $2,500 = 2.16 ETH

4. Convert to cETH
   2.16 / 0.02 = 108 cETH
```

The liquidator receives 108 cETH. They paid $5,000 worth of USDC and received $5,400 worth of ETH in cToken form, pocketing the 8% incentive as profit.

The code structures the arithmetic as `(liquidationIncentive * priceBorrowed) / (priceCollateral * exchangeRate)` rather than dividing in separate steps. Grouping all multiplications before the single division minimizes precision loss from integer rounding.

---

## Conclusion

These three functions complete the Comptroller's side of the liquidation flow. The cToken handles the mechanics of repaying debt and moving collateral, while the Comptroller decides when that is allowed and how much collateral the liquidator gets.
