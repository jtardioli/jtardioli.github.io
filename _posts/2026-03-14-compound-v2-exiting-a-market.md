---
layout: post
title: 'Compound V2: Exiting a Market'
date: 2026-03-14 19:36 -0400
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, comptroller, market, exit]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 13
---

> This article is part of the Compound V2 series. See the [series index](/posts/compound-v2-introduction/#series-index) for a full list of articles.

The previous article covered how markets are listed and how users enter them. This article covers the inverse: how a user exits a market they have entered.

Entering a market opts a user's supply into the liquidity calculation, allowing it to serve as collateral. Exiting removes it. Once a user exits a market, their cToken balance in that market is no longer counted toward their borrow power. This also means those cTokens can no longer be seized in a liquidation.

---

## `exitMarket`

The function breaks into three stages: validation, the membership check, and removing the user from the data structures.

### Validation

```
(uint oErr, uint tokensHeld, uint amountOwed, ) = cToken.getAccountSnapshot(msg.sender);
require(oErr == 0, "exitMarket: getAccountSnapshot failed");

if (amountOwed != 0) {
    return fail(Error.NONZERO_BORROW_BALANCE, FailureInfo.EXIT_MARKET_BALANCE_OWED);
}

uint allowed = redeemAllowedInternal(cTokenAddress, msg.sender, tokensHeld);
if (allowed != 0) {
    return failOpaque(Error.REJECTION, FailureInfo.EXIT_MARKET_REJECTION, allowed);
}
```

1. `getAccountSnapshot` retrieves the user's cToken balance (`tokensHeld`) and outstanding borrow (`amountOwed`) in this market.
2. If the user has any outstanding borrow in this market, the exit is rejected. A user cannot stop using a market as collateral while still borrowing from it.
3. `redeemAllowedInternal` simulates what would happen if the user redeemed all of their cTokens in this market. If withdrawing that collateral would leave the account insolvent, meaning their remaining collateral across other markets cannot support their borrows, the exit is blocked. The next article explains how this solvency check works in detail.

---

### Membership Check

```
if (!marketToExit.accountMembership[msg.sender]) {
    return uint(Error.NO_ERROR);
}
```

If the user is not currently in the market, the function returns success immediately. This is not an error. Calling `exitMarket` on a market the user never entered is a no-op.

---

### Removing from the Data Structures

As covered in the previous article, market membership is stored in two places: the `accountMembership` mapping for O(1) lookups and the `accountAssets` array for iteration. Both must be updated.

```
delete marketToExit.accountMembership[msg.sender];
```

The mapping entry is deleted first.

```
CToken[] memory userAssetList = accountAssets[msg.sender];
uint len = userAssetList.length;
uint assetIndex = len;
for (uint i = 0; i < len; i++) {
    if (userAssetList[i] == cToken) {
        assetIndex = i;
        break;
    }
}

assert(assetIndex < len);

CToken[] storage storedList = accountAssets[msg.sender];
storedList[assetIndex] = storedList[storedList.length - 1];
storedList.pop();
```

Removing from the array is more involved. Solidity arrays do not support removing an element by index without leaving a gap. The workaround is a swap-and-pop: copy the last element into the slot being removed, then pop the last element off. This avoids shifting every element after the removed one, keeping the operation O(1) instead of O(n).

The `assert(assetIndex < len)` is a sanity check. If the user passed the membership check above, the cToken must exist in the array. If it does not, the redundant data structures are out of sync and the assertion halts execution.

The array is first copied into memory for the search loop. Memory reads are cheaper than storage reads, so iterating over the memory copy saves gas. The actual swap-and-pop then operates on the storage array directly.

---

## Conclusion

Exiting a market is the inverse of entering one. The key constraint is that a user cannot exit if doing so would leave their account insolvent or if they still have an outstanding borrow in that market.
