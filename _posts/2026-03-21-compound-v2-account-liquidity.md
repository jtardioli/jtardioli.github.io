---
layout: post
title: 'Compound V2: Account Liquidity'
date: 2026-03-21 10:00 -0500
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, comptroller, liquidity, collateral factor, solvency]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 14
---

> This article is part of the Compound V2 series. See the [series index](/posts/compound-v2-introduction/#series-index) for a full list of articles.

One function underpins nearly every permission check in the Comptroller: `getHypotheticalAccountLiquidityInternal`. The previous articles walked through minting, borrowing, and redeeming while treating the Comptroller as a black box. This section opens that box. Specifically, this function answers two questions:

1. Is an account currently solvent? That is, does it have enough collateral to cover what it has borrowed?
2. What is the maximum additional amount an account can borrow or redeem without becoming insolvent?

If an account is insolvent it cannot borrow more and becomes eligible for liquidation.

The function is also used to simulate hypothetical actions. Before Compound allows a user to borrow more or redeem collateral, it runs this function with those hypothetical amounts to verify the account would remain solvent after the action. That is what "hypothetical" refers to in the name.

Almost all of the calculations here use mantissa-scaled fixed-point values. The `mul_` functions shown below handle the fixed-point arithmetic internally. If you have read the fixed-point math article in this series, the mechanics will be familiar. This walkthrough focuses on the higher-level picture rather than the scaling details.

## What the Function Computes

The function computes two running totals across every market the user has entered:

- `sumCollateral`: the total borrow power in USD the user currently has, given their deposits across all entered markets.
- `sumBorrowPlusEffects`: the total amount the user is currently borrowing across all markets, also denominated in USD.

An account is solvent if `sumCollateral > sumBorrowPlusEffects`. The surplus is the amount the user can still borrow. If the relationship flips, the account is insolvent and the shortfall is the gap that must be covered before borrowing is possible again.

Translating everything into USD is what makes it possible to sum across markets. ETH and DAI cannot be added directly, but their USD values can.

## The Main Loop

The bulk of `getHypotheticalAccountLiquidityInternal` is a loop over every market the account has entered. Each iteration computes the contribution of one market to `sumCollateral` and `sumBorrowPlusEffects`, then accumulates those into the running totals. A user who has entered the ETH market and the DAI market will have two iterations, one for each.

```
CToken[] memory assets = accountAssets[account];

for (uint i = 0; i < assets.length; i++) {
    CToken asset = assets[i];

    (oErr, vars.cTokenBalance, vars.borrowBalance, vars.exchangeRateMantissa) = asset.getAccountSnapshot(account);
    if (oErr != 0) {
        return (Error.SNAPSHOT_ERROR, 0, 0);
    }

    vars.collateralFactor = Exp({mantissa: markets[address(asset)].collateralFactorMantissa});
    vars.exchangeRate     = Exp({mantissa: vars.exchangeRateMantissa});

    vars.oraclePriceMantissa = oracle.getUnderlyingPrice(asset);
    if (vars.oraclePriceMantissa == 0) {
        return (Error.PRICE_ERROR, 0, 0);
    }
    vars.oraclePrice = Exp({mantissa: vars.oraclePriceMantissa});
```

The first part of the loop retrieves the values needed for the calculation:

- `vars.cTokenBalance`: the user's cToken balance for this market.
- `vars.borrowBalance`: the amount of underlying tokens the user has borrowed in this market, with interest accrued to the current block.
- `vars.exchangeRateMantissa`: the current exchange rate between cTokens and underlying tokens.
- `vars.oraclePrice`: the USD price of the underlying asset, sourced from the oracle.

## The Collateral Factor

`vars.collateralFactor` has not appeared in the series yet and deserves explanation before moving forward. The collateral factor is Compound's equivalent of a loan-to-value (LTV) ratio. It is a value between 0 and 1, stored as a mantissa, that controls how much borrow power each dollar of supply in a given market generates.

Stable assets tend to have higher collateral factors, such as 0.8 or 0.9. Volatile assets tend to have lower ones, such as 0.3 or 0.4. The reasoning is straightforward: a volatile asset can lose value quickly, so Compound requires a larger buffer between its market value and the borrow limit. This buffer gives liquidators time to act before a position goes underwater.

As a concrete example:

```
// Depositing $1,000 USDC with a collateral factor of 0.9
borrow_power = 1000 * 0.9 = $900

// Depositing 1 ETH at $10,000 (one can dream) with a collateral factor of 0.8
borrow_power = 10000 * 0.8 = $8,000
```

## The Oracle Price

```
vars.oraclePriceMantissa = oracle.getUnderlyingPrice(asset);
if (vars.oraclePriceMantissa == 0) {
    return (Error.PRICE_ERROR, 0, 0);
}
vars.oraclePrice = Exp({mantissa: vars.oraclePriceMantissa});
```

The oracle price is the USD price of the underlying asset for the current market. This is how the function converts cToken balances and borrow balances into a common denomination so they can be summed across markets. A zero price is treated as an error because it would corrupt every downstream calculation.

## `tokensToDenom` and `sumCollateral`

```
vars.tokensToDenom = mul_(mul_(vars.collateralFactor, vars.exchangeRate), vars.oraclePrice);
```

`tokensToDenom` is a precomputed conversion factor. For each cToken the user holds, it gives the USD borrow power that cToken represents. The derivation in steps:

```
underlying_amount = 1 cToken * exchangeRate
usdc_value        = underlying_amount * oraclePrice
borrow_power      = usdc_value * collateralFactor
```

All three multiplications are combined into `tokensToDenom` so the value can be reused across the loop without recomputing. As a numeric example: suppose 1 cDAI has an exchange rate of 0.02 DAI per cDAI, a DAI oracle price of $1, and a collateral factor of 0.8. Then:

```
tokensToDenom = 0.02 * 1 * 0.8 = 0.016 USD of borrow power per cDAI
```

To get the market's full contribution to `sumCollateral`, multiply `tokensToDenom` by the user's total cToken balance and accumulate it into the running total:

```
vars.sumCollateral = mul_ScalarTruncateAddUInt(vars.tokensToDenom, vars.cTokenBalance, vars.sumCollateral);
```

## `sumBorrowPlusEffects`

Borrow balances are already denominated in underlying tokens, so the only conversion needed is to USD:

```
vars.sumBorrowPlusEffects = mul_ScalarTruncateAddUInt(vars.oraclePrice, vars.borrowBalance, vars.sumBorrowPlusEffects);
```

Underlying amount multiplied by oracle price gives USD value. That value is accumulated into `sumBorrowPlusEffects` across all markets.

## Simulating Hypothetical Actions

```
if (asset == cTokenModify) {
    vars.sumBorrowPlusEffects = mul_ScalarTruncateAddUInt(vars.tokensToDenom, redeemTokens, vars.sumBorrowPlusEffects);
    vars.sumBorrowPlusEffects = mul_ScalarTruncateAddUInt(vars.oraclePrice, borrowAmount, vars.sumBorrowPlusEffects);
}
```

This block is triggered only for the specific market being hypothetically modified. The borrow simulation is intuitive: add the hypothetical borrow amount in USD to `sumBorrowPlusEffects`, then check whether the result still falls under `sumCollateral`.

The redeem simulation requires more thought. Redeeming cTokens reduces your collateral, which should reduce `sumCollateral`. Instead, the code adds the redeemed value to `sumBorrowPlusEffects`. These are equivalent for the purposes of the final comparison:

```
sumCollateral - redeemValue > sumBorrowPlusEffects
sumCollateral               > sumBorrowPlusEffects + redeemValue
```

Both forms produce the same solvency verdict. For example, with `sumCollateral = 1000`, `sumBorrowPlusEffects = 500`, and `redeemValue = 300`:

```
1000 - 300 > 500   →   700 > 500   ✓
1000       > 500 + 300  →  1000 > 800  ✓
```

Compound uses the second form because subtracting from `sumCollateral` would require an underflow check. Adding to `sumBorrowPlusEffects` is simpler and produces the same answer.

## Returning the Result

```
if (vars.sumCollateral > vars.sumBorrowPlusEffects) {
    return (Error.NO_ERROR, vars.sumCollateral - vars.sumBorrowPlusEffects, 0);
} else {
    return (Error.NO_ERROR, 0, vars.sumBorrowPlusEffects - vars.sumCollateral);
}
```

The function returns a tuple of `(error, liquidity, shortfall)`. Exactly one of `liquidity` or `shortfall` will be nonzero. `liquidity` is the surplus borrow power available. `shortfall` is the amount by which the account is undercollateralized. The calling functions use these values to decide whether to allow or revert the action.

## Conclusion

With this function in hand, the remaining Comptroller logic becomes straightforward. Every pre-action hook in the Comptroller, whether for borrowing, redeeming, or transferring, ultimately calls back to this calculation.
