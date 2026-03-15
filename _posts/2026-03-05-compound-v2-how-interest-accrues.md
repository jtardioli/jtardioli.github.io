---
layout: post
title: 'Compound V2: Interest Accrual'
date: 2026-03-05 09:48 -0500
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, interest, accrueInterest]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 5
---

> This article is part of the Compound V2 series. See the [series index](/posts/compound-v2-introduction/#series-index) for a full list of articles.

The `accrueInterest` function is the backbone of the Compound protocol. It is called before most actions in the protocol to keep all debt up to date. Make sure you have already read the articles on the [exchange rate](https://cursedid0l.github.io/posts/compound-v2-ctokens-and-the-exhange-rate/), the [borrow index](https://cursedid0l.github.io/posts/compound-v2-tracking-interest-with-the-borrow-index/), and [fixed-point math](https://cursedid0l.github.io/posts/compound-v2-fixed-point-math/).

The main goal of `accrueInterest` is to update these four state variables:

- `totalBorrows`
- `totalReserves`
- `borrowIndex`
- `accrualBlockNumber`

At this point you should already be familiar with `totalBorrows`, `totalReserves`, and `borrowIndex`. The only new variable is `accrualBlockNumber`, which stores the block number of the last time `accrueInterest` was called. Its purpose will become clear as the function is broken down below.

---

## Step 1: Check if Interest Has Already Been Accrued This Block

```
uint currentBlockNumber = getBlockNumber();
uint accrualBlockNumberPrior = accrualBlockNumber;

// Short-circuit accumulating 0 interest
if (accrualBlockNumberPrior == currentBlockNumber) {
    return NO_ERROR;
}
```

The function fetches the current block number and compares it to the block number of the last accrual. If they are the same, it returns early. This means all transactions in a block share the same interest snapshot. Interest ticks up block by block, not transaction by transaction.

---

## Step 2: Load State and Calculate Block Delta

```
// Read the previous values out of storage
uint cashPrior = getCashPrior();
uint borrowsPrior = totalBorrows;
uint reservesPrior = totalReserves;
uint borrowIndexPrior = borrowIndex;

// Calculate the current borrow interest rate
uint borrowRateMantissa = interestRateModel.getBorrowRate(cashPrior, borrowsPrior, reservesPrior);
require(borrowRateMantissa <= borrowRateMaxMantissa, "borrow rate is absurdly high");

// Calculate the number of blocks elapsed since the last accrual
uint blockDelta = currentBlockNumber - accrualBlockNumberPrior;
```

The state variables are loaded into local `prior` variables because the state will be updated at the end of the function. Reading from storage once and caching in memory is cheaper than reading from storage multiple times, so this is a common Solidity gas optimization pattern.

The borrow rate is fetched from the `interestRateModel`. How it is calculated will be covered in a later article. For now, you should already understand what the borrow rate represents from the [borrow index article](https://cursedid0l.github.io/posts/compound-v2-tracking-interest-with-the-borrow-index/).

The `require` check on `borrowRateMaxMantissa` is a sanity guard that prevents a buggy or malicious interest rate model from setting an absurdly high rate and draining the protocol.

Finally, `blockDelta` is the number of blocks that have passed since the last accrual.

---

## Step 3: Calculate `simpleInterestFactor`

```
// simpleInterestFactor = borrowRate * blockDelta
Exp memory simpleInterestFactor = mul_(Exp({mantissa: borrowRateMantissa}), blockDelta);
```

Interest accrues linearly between accruals. Multiplying the per-block borrow rate by the number of elapsed blocks gives the total rate for the entire period. The result is called `simpleInterestFactor`.

`simpleInterestFactor` is stored as an `Exp` struct, which is the fixed-point type used throughout the protocol. Unlike `mul_ScalarTruncate`, `mul_` returns a full `Exp` without truncating, preserving precision for the calculations that follow.

For example, if the per-block borrow rate is `0.0001` and 10 blocks have passed, `simpleInterestFactor` is `0.001`.

Note that this is simple (linear) interest, not compound interest. The compounding happens at a higher level via the `borrowIndex`.

---

## Step 4: Calculate Interest Accumulated and Update `totalBorrows`

```
// interestAccumulated = simpleInterestFactor * totalBorrows
uint interestAccumulated = mul_ScalarTruncate(simpleInterestFactor, borrowsPrior);

// totalBorrowsNew = interestAccumulated + totalBorrows
uint totalBorrowsNew = interestAccumulated + borrowsPrior;
```

Multiplying `simpleInterestFactor` by `totalBorrows` gives the total interest accumulated since the last accrual. `totalBorrows` is the sum of all individual principals, and `simpleInterestFactor` is the borrow rate for the period. This is equivalent to `principal * borrowRate`, done in aggregate rather than one borrower at a time.

This works because interest accrues proportionally. Summing all principals and applying a single rate produces the same result as computing interest per borrower and summing.

The accumulated interest is then added to `borrowsPrior` to get the updated `totalBorrows`.

---

## Step 5: Update `totalReserves`

```
// totalReservesNew = (interestAccumulated * reserveFactor) + totalReserves
uint totalReservesNew = mul_ScalarTruncateAddUInt(Exp({mantissa: reserveFactorMantissa}), interestAccumulated, reservesPrior);
```

The protocol does not distribute 100% of accrued interest to suppliers. A portion is kept by the protocol in `totalReserves` to cover potential losses or as protocol revenue. The `reserveFactor` is the percentage of interest that goes to reserves, which means suppliers effectively receive `(1 - reserveFactor)` of all accrued interest.

For example, if the reserve factor is 5%:

```
interestAccumulated * reserveFactor = reservesEarned
100                 * 0.05          = 5
```

That amount is added to `totalReserves` so it accumulates over time.

---

## Step 6: Update `borrowIndex`

```
// borrowIndexNew = simpleInterestFactor * borrowIndex + borrowIndex
uint borrowIndexNew = mul_ScalarTruncateAddUInt(simpleInterestFactor, borrowIndexPrior, borrowIndexPrior);
```

The borrow index is updated by multiplying it by `simpleInterestFactor` and adding the result to the prior index. This is equivalent to:

```
borrowIndexNew = borrowIndexPrior * (1 + simpleInterestFactor)
```

Each accrual scales the index up by the interest rate for that period. Individual borrower debt can then be calculated at any time by comparing the index at the time of their last interaction to the current index.

---

## Step 7: Write State and Emit Event

```
// EFFECTS & INTERACTIONS
// (No safe failures beyond this point)

accrualBlockNumber = currentBlockNumber;
borrowIndex = borrowIndexNew;
totalBorrows = totalBorrowsNew;
totalReserves = totalReservesNew;

emit AccrueInterest(cashPrior, interestAccumulated, borrowIndexNew, totalBorrowsNew);

return NO_ERROR;
```

All four state variables are written to storage at once, at the end of the function.

The `AccrueInterest` event is emitted with `cashPrior`, `interestAccumulated`, `borrowIndexNew`, and `totalBorrowsNew`. Finally, `NO_ERROR` is returned to signal that the function completed successfully.

---

## Conclusion

`accrueInterest` is a bookkeeping function. Every time it runs, it asks: how many blocks have passed, what was the borrow rate over that period, and how much interest does that imply? It then distributes that interest across `totalBorrows`, `totalReserves`, and `borrowIndex` in a single update.

Because it is called before most other actions in the protocol, every interaction with Compound starts from an up-to-date interest state. No interest is ever silently stale.