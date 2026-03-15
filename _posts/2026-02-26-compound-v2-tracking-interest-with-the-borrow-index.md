---
layout: post
title: 'Compound V2: The Borrow Index'
date: 2026-02-26 10:54 -0500
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, borrow index, interest]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 3
---

> This article is part of the Compound V2 series. See the [series index](/posts/compound-v2-introduction/#series-index) for a full list of articles.


To build a lending protocol, you need a way to track how much debt each borrower owes as it accumulates over time. Compound tracks interest on a block-by-block basis, meaning debt accrues with each new Ethereum block (roughly every 12-15 seconds). Getting this right matters because without accurate debt tracking, borrowers could underpay and lenders would not receive the interest they are owed. This article explains how Compound handles this efficiently using a single accumulating value called the `borrowIndex`.



---



## Compounding Interest Per Block



Every borrower's debt grows by a fixed rate each block. Two values drive the calculation:



- **`principal`**: the initial amount borrowed

- **`borrowRate`**: the percentage debt grows each block



In the calculations below, when you see `(1 + borrowRate)` it means keep 100% of what you had, plus add interest on top. For example, if you had 10 apples and wanted 30% more, the intuitive calculation is `10 * 1.30`, which can be rewritten as `10 * (1 + 0.30)`. Interest works the same way: `(1 + 0.01) = 1.01` means your balance is now 101% of what it was, your original amount plus 1% more.



This simple example shows how interest accumulates over blocks:



```

principal = 1

borrowRate = 0.01 (1%)



debtAfter1Block = principal * (1 + borrowRate)

= 1 * 1.01

= 1.01



debtAfter2Blocks = debtAfter1Block * (1 + borrowRate)

= 1.01 * 1.01

= 1.01^2

= 1.0201

```



Notice the pattern: after 2 blocks the result is `1.01^2`. Each block the debt is multiplied by `1.01` again, so the exponent just counts the blocks elapsed. The step-by-step multiplication can be skipped entirely using `principal * (1 + borrowRate)^n`, where `n` is the number of blocks:



```

principal = 1

borrowRate = 0.01 (1%)

n = 5 (5 blocks elapsed)



debt = principal * (1 + borrowRate)^n

= 1 * 1.01^5

= 1.0510

```



This is standard compound interest. Each block you pay interest not just on your original debt, but on the interest that has already accumulated. That is what makes the exponent grow rather than a simple multiplication by `n`.



---



## The Scaling Problem



Calculating the interest owed each block by 1 borrower is simple enough, but a scaling problem emerges as the number of borrowers grows.



```

principal_alice = 1000

principal_bob = 500

principal_carol = 250

borrowRate = 0.01 (1%)



# Block 1

alice_debt = 1000 * 1.01 = 1010

bob_debt = 500 * 1.01 = 505

carol_debt = 250 * 1.01 = 252.5



# Block 2

alice_debt = 1010 * 1.01 = 1020.1

bob_debt = 505 * 1.01 = 510.05

carol_debt = 252.5 * 1.01 = 255.025

```



With just the 3 borrowers from this example, there are already 6 updates over 2 blocks. Compound has thousands of borrowers. Updating each position on every block would be extremely expensive in gas. That's where the `borrowIndex` comes in.



---

## The Borrow Index



The key insight is that every dollar borrowed accumulates debt at the same rate. If Alice and Bob each borrow $1 at a 1% borrow rate, after one block they both owe $1.01. The growth factor is the same regardless of principal.



Instead of tracking each user's balance separately, the protocol tracks a single value: how much $1 borrowed at block 0 would be worth right now. This is the `borrowIndex`.



### How the Borrow Index Updates



The `borrowIndex` starts at 1 and multiplies by `(1 + borrowRate)` each block:



```

newBorrowIndex = oldBorrowIndex * (1 + borrowRate)

```



```

borrowRate = 0.01 (1%)

borrowIndex = 1 # initial value



# After block 1

borrowIndex = 1 * (1 + 0.01) = 1.01



# After block 2

borrowIndex = 1.01 * (1 + 0.01) = 1.0201

```



To find any user's current debt, multiply their principal by the current `borrowIndex`:



```

userDebt = principal * borrowIndex

```



```

aliceDebt = 1.0201 * 1000 = 1020.1

bobDebt = 1.0201 * 500 = 510.05

carolDebt = 1.0201 * 250 = 255.025

```



Now only one value updates per block instead of calculating every borrower's debt. Any user's current debt can be calculated on demand by multiplying their `principal` by the current `borrowIndex`. This means there is no need to store each user's running debt each block.



---



## Handling Different Entry Points



The formula above assumes all users borrowed at block 0. In practice, borrowers enter at different blocks, each with a different accumulated `borrowIndex` at the time they borrow.



Consider this example:



```

// This example uses unrealistic borrowIndex values to make the math more intuitive

block 0: borrowIndex = 1

block 1: borrowIndex = 2

block 2: borrowIndex = 4 ← Alice borrows here, snapshots borrowIndex = 4

block 3: borrowIndex = 8

block 4: borrowIndex = 16 ← current block

```




Alice borrowed when the `borrowIndex` was 4. The current `borrowIndex` is 16. The goal is to find the multiplier that took 4 to 16:



```

4 * x = 16

x = 16 / 4

x = 4

```



Between block 2 and block 4, every dollar of debt grew by a factor of 4. If Alice borrowed $20, she now owes `20 * 4 = 80`.



This generalizes to:



```

principalMultiplier = currentBorrowIndex / borrowIndexWhenUserBorrowed

userDebt = principal * principalMultiplier

= principal * (currentBorrowIndex / borrowIndexWhenUserBorrowed)

```



In the Compound contracts, `borrowIndexWhenUserBorrowed` is stored per user and is called `interestIndex`. The `interestIndex` is set to the global `borrowIndex` at the moment the user's borrow snapshot is created or updated. So the final debt formula is:



```

userDebt = principal * (currentBorrowIndex / interestIndex)

```



---



## The Contract Data Structures



To summarize, calculating a user's debt at any point requires three things:



- The user's `principal`

- The global `borrowIndex` right now

- The `interestIndex`: the snapshot of the global `borrowIndex` taken when the user last updated their borrow position



In the smart contract this looks like:



```

uint public borrowIndex;



struct BorrowSnapshot {

uint principal;

uint interestIndex;

}



mapping(address => BorrowSnapshot) internal accountBorrows;

```




The global `borrowIndex` accrues every block, growing to reflect the total interest accumulated since the protocol launched. When a user borrows, a snapshot is taken of two values: their `principal` and the current `borrowIndex`, stored as their `interestIndex`. From that point on, their debt can be calculated at any time by comparing where the `borrowIndex` is now to where it was when they borrowed.



That is the `borrowIndex`: a single accumulating value that lets you calculate any user's current debt regardless of when they borrowed.



---




## Conclusion



|Variable|What it represents|

|---|---|

|`borrowIndex`|Global accumulator: how much $1 borrowed at inception is worth now|

|`principal`|Amount the user originally borrowed|

|`interestIndex`|Snapshot of `borrowIndex` taken when user borrowed|



**Debt formula:**



```

userDebt = principal * (currentBorrowIndex / interestIndex)

```
