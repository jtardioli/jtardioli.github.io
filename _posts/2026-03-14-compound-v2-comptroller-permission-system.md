---
layout: post
title: 'Compound V2: Comptroller Permission System'
date: 2026-03-14 19:43 -0400
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, comptroller, permissions, hooks]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 15
---

> This article is part of the Compound V2 series. See the [series index](/posts/compound-v2-introduction/#series-index) for a full list of articles.

## The Division of Responsibility

The previous article covered how the Comptroller determines whether an account is solvent. This article covers where that calculation actually gets used. Before executing any state-changing operation, a cToken calls the Comptroller through a pair of hook functions. The first, named with an `*Allowed` suffix, runs before the action and either permits or blocks it. The second, named with a `*Verify` suffix, runs after the action completes and can revert the transaction if something went wrong.

One thing to be aware of while reading the code is that there are references to a flywheel throughout these functions. The flywheel is Compound's COMP reward distribution system and will be covered in a later article. For now, treat any call to `updateCompSupplyIndex`, `distributeSupplierComp`, or similar functions as a black box.

## The `*Allowed` Functions

### `mintAllowed`

```
function mintAllowed(address cToken, address minter, uint mintAmount) override external returns (uint) {
    require(!mintGuardianPaused[cToken], "mint is paused");

    minter;
    mintAmount;

    if (!markets[cToken].isListed) {
        return uint(Error.MARKET_NOT_LISTED);
    }

    updateCompSupplyIndex(cToken);
    distributeSupplierComp(cToken, minter);

    return uint(Error.NO_ERROR);
}
```

`mintAllowed` is the simplest of the permission checks. It verifies two things: that minting has not been paused for this market, and that the market is actually listed in the Comptroller. If either condition fails, the mint is blocked. The pause flags exist as emergency levers that Compound governance can activate if a market is behaving abnormally or a vulnerability is discovered.

### `redeemAllowed`

```
function redeemAllowedInternal(address cToken, address redeemer, uint redeemTokens) internal view returns (uint) {
    if (!markets[cToken].isListed) {
        return uint(Error.MARKET_NOT_LISTED);
    }

    if (!markets[cToken].accountMembership[redeemer]) {
        return uint(Error.NO_ERROR);
    }

    (Error err, , uint shortfall) = getHypotheticalAccountLiquidityInternal(redeemer, CToken(cToken), redeemTokens, 0);
    if (err != Error.NO_ERROR) {
        return uint(err);
    }
    if (shortfall > 0) {
        return uint(Error.INSUFFICIENT_LIQUIDITY);
    }

    return uint(Error.NO_ERROR);
}
```

Redeeming is more involved because withdrawing collateral can push an account toward insolvency. The function first checks that the market is listed, then checks whether the redeemer has entered the market. If they have not, their balance in that market is not being used as collateral, so redeeming it cannot affect their liquidity position and the check passes immediately.

If the redeemer is in the market, the function calls `getHypotheticalAccountLiquidityInternal` with the requested redeem amount to simulate what the account's position would look like after the withdrawal. If the simulation shows a shortfall, meaning the account would be borrowing more than its remaining collateral supports, the redeem is blocked.

### `borrowAllowed`

```
function borrowAllowed(address cToken, address borrower, uint borrowAmount) override external returns (uint) {
    require(!borrowGuardianPaused[cToken], "borrow is paused");

    if (!markets[cToken].isListed) {
        return uint(Error.MARKET_NOT_LISTED);
    }

    if (!markets[cToken].accountMembership[borrower]) {
        require(msg.sender == cToken, "sender must be cToken");

        Error err = addToMarketInternal(CToken(msg.sender), borrower);
        if (err != Error.NO_ERROR) {
            return uint(err);
        }

        assert(markets[cToken].accountMembership[borrower]);
    }

    if (oracle.getUnderlyingPrice(CToken(cToken)) == 0) {
        return uint(Error.PRICE_ERROR);
    }

    uint borrowCap = borrowCaps[cToken];
    if (borrowCap != 0) {
        uint totalBorrows = CToken(cToken).totalBorrows();
        uint nextTotalBorrows = add_(totalBorrows, borrowAmount);
        require(nextTotalBorrows < borrowCap, "market borrow cap reached");
    }

    (Error err, , uint shortfall) = getHypotheticalAccountLiquidityInternal(borrower, CToken(cToken), 0, borrowAmount);
    if (err != Error.NO_ERROR) {
        return uint(err);
    }
    if (shortfall > 0) {
        return uint(Error.INSUFFICIENT_LIQUIDITY);
    }

    Exp memory borrowIndex = Exp({mantissa: CToken(cToken).borrowIndex()});
    updateCompBorrowIndex(cToken, borrowIndex);
    distributeBorrowerComp(cToken, borrower, borrowIndex);

    return uint(Error.NO_ERROR);
}
```

`borrowAllowed` has the most logic of any permission check.

First, the function checks that borrowing has not been paused for this market and that the market is listed. Then it checks whether the borrower has entered the market. Entering the borrow market is required before a user can take out a loan. If they have not entered yet, the function handles it automatically. It requires that the caller is the cToken itself to prevent a third party from enrolling an arbitrary address, then calls `addToMarketInternal` to enter the borrower into the market. The `assert` that follows is a sanity check confirming the enrollment succeeded.

Next, the function verifies the oracle is returning a valid price for the asset. A zero price would corrupt the liquidity calculation and needs to be caught before proceeding.

After the oracle check, the function enforces the borrow cap. Compound can set a maximum total borrow amount per market, and a borrow cap of zero means unlimited. If a cap is set, the function checks that the new borrow would not push total market borrows over the limit.

Finally, `getHypotheticalAccountLiquidityInternal` is called to simulate the account's position after the borrow. If the simulation shows a shortfall, the borrow is blocked.

### `repayBorrowAllowed`

`repayBorrowAllowed` only checks that the market is listed. Repaying a borrow can never make a position worse, so no liquidity check is needed. The rest of the function body is flywheel-related and will be covered in a later article.

### `transferAllowed`

`transferAllowed` checks that transfers have not been paused, then delegates to `redeemAllowedInternal` using the transfer amount. The reasoning is important. cTokens are collateral. If a user's position is undercollateralized, their cTokens are subject to seizure by a liquidator. Allowing them to transfer those cTokens to a different address that has no open borrows would let them escape liquidation entirely, effectively stealing from lenders. By requiring that a transfer passes the same check as a redeem, the protocol ensures users can only move cTokens they are not relying on to back an active borrow.

## The `*Verify` Functions

The verify hooks run after an action completes. In Compound V2, every verify function is a no-op except for `redeemVerify`.

### `redeemVerify`

```
if (redeemTokens == 0 && redeemAmount > 0) {
    revert("redeemTokens zero");
}
```

This check guards against a division rounding edge case in `redeemFresh`. When a user redeems by specifying an underlying amount rather than a cToken amount, the contract calculates how many cTokens to burn.

```
redeemTokens = div_(redeemAmountIn, exchangeRate);
redeemAmount = redeemAmountIn;
```

If the exchange rate is large enough relative to the requested amount, integer division can round `redeemTokens` down to zero while `redeemAmount` remains nonzero. The user would receive underlying tokens while burning no cTokens. `redeemVerify` catches this case after the fact and reverts the transaction before it is finalized.

## Conclusion

The `*Allowed` hooks enforce market status, liquidity requirements, and protocol-level caps before an action executes. The `*Verify` hooks are mostly no-ops, with `redeemVerify` being the only one that does real work, catching a rounding edge case that only becomes visible after the math has run.
