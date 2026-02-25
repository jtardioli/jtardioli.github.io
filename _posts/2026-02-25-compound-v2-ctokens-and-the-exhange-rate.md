---
layout: post
title: 'Compound V2: cTokens and the Exchange Rate'
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, ctoken, exchange rate, exchangerate]     # TAG names should always be lowercase
date: 2026-02-25 14:51 -0500
image:
  path: /assets/img/compoundv2.png
  alt: Compound V2
---

Compound lets users deposit tokens to earn yield. When you deposit, you receive a cToken in return that represents your position. You can think of a cToken as a receipt for your deposited tokens. Later you will exchange your cTokens for your underlying when you want to redeem them. The exchange rate between cTokens and the underlying token is how that yield is delivered. Over time, each cToken entitles you to more underlying than when you deposited. This article explains what the exchange rate is, how it's calculated, and why it grows.

## Key Definitions

Before we can look at the exchange rate formula, we need four Compound-specific terms. 

**cash**:  The actual token balance held by the cToken contract right now, what's physically sitting in the contract's wallet.

- Increases when users supply or repay borrows
- Decreases when users borrow or redeem
- Not stored as a state variable; it's the live ERC-20 balance of the contract `underlyingToken.balanceOf(address(this))`

**totalBorrows**:  The sum of all outstanding loans from this market, including accrued interest. Essentially this is the total debt

- Increases when users borrow or when interest accrues on existing borrows
- Decreases when users repay
- Represents money that is not in the contract; it's been sent out to borrowers
- Is a cToken storage variable

**totalReserves**:  A protocol-owned cut of interest income, set aside as a safety buffer for Compound governance.

- Can only be withdrawn by the admin
- Subtracted from the pool when calculating cToken value, since suppliers don't benefit from it
- Is a cToken storage variable

**totalSupply**:  The total number of cTokens in existence.

- Increases when users supply underlying
- Decreases when users redeem cTokens for their underlying
- Is a cToken storage variable

So `cash + totalBorrows` is the total underlying assets of the market, and `totalReserves` is the portion of those assets marked for the protocol rather than suppliers. The numerator `cash + totalBorrows - totalReserves` therefore represents the underlying assets that  suppliers collectively own.


## The Exchange Rate Formula

```
// 1 cToken = exchangeRate * 1 underlyingTokens
exchangeRate = (cash + totalBorrows - totalReserves) / totalSupply
```

## Using the Exchange Rate

To calculate how many cTokens you receive when supplying underlying:

```
cTokensReceived = underlyingSupplied / exchangeRate
```

To calculate how much underlying you receive when redeeming cTokens:

```
underlyingReceived = cTokensBurned * exchangeRate
```

## How Suppliers Earn Yield

As time goes on, borrowers pay interest. That interest increases `totalBorrows`, which increases the numerator of the exchange rate formula. A larger numerator means a higher exchange rate, and a higher exchange rate means your cTokens can be redeemed for more underlying than before.

Here's what that looks like in practice:

```
// You supply 110 USDC when the exchange rate is 1.10
cTokensReceived = 110 / 1.10 = 100 cUSDC
```

Over the course of a month, borrowers pay interest and the exchange rate rises:

```
// Exchange rate is now 1.20
underlyingReceived = 100 * 1.20 = 120 USDC
```

You supplied 110 USDC and can now redeem for 120 USDC, a profit of 10 USDC from interest alone, without doing anything after the initial deposit.

The exchange rate is the mechanism that connects borrower interest payments to supplier yield. Every time interest accrues, the numerator grows, the rate ticks up, and your cTokens become redeemable for more underlying.