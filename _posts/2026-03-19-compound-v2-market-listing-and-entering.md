---
layout: post
title: 'Compound V2: Market Listing and Entering'
date: 2026-03-19 10:00 -0500
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, comptroller, market, collateral]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 12
---

> This article is part of the Compound V2 series. See the [series index](/posts/compound-v2-introduction/#series-index) for a full list of articles.

This article covers two related topics: how Compound admins list new assets as markets, and how users enter those markets to supply collateral or borrow.

## Adding a Market

Before users can interact with any asset in Compound, that asset must first be listed as a market. Only the Compound admin can do this, through `_supportMarket`.

```
function _supportMarket(CToken cToken) external returns (uint) {
    if (msg.sender != admin) {
        return fail(Error.UNAUTHORIZED, FailureInfo.SUPPORT_MARKET_OWNER_CHECK);
    }

    if (markets[address(cToken)].isListed) {
        return fail(Error.MARKET_ALREADY_LISTED, FailureInfo.SUPPORT_MARKET_EXISTS);
    }

    cToken.isCToken();

    Market storage newMarket = markets[address(cToken)];
    newMarket.isListed = true;
    newMarket.isComped = false;
    newMarket.collateralFactorMantissa = 0;

    _addMarketInternal(address(cToken));
    _initializeMarket(address(cToken));

    emit MarketListed(cToken);

    return uint(Error.NO_ERROR);
}
```

The function opens with two access guards. The first rejects any caller that is not the admin. The second rejects a cToken that has already been listed, preventing the same market from being registered twice. After the guards, `cToken.isCToken()` is called as a sanity check to confirm the address actually implements the cToken interface rather than being an arbitrary contract.

With the checks passed, the function writes the initial market state and then delegates to two helper functions before emitting `MarketListed` and returning.

### `_addMarketInternal`

```
function _addMarketInternal(address cToken) internal {
    for (uint i = 0; i < allMarkets.length; i++) {
        require(allMarkets[i] != CToken(cToken), "market already added");
    }
    allMarkets.push(CToken(cToken));
}
```

`allMarkets` is a global array that tracks every listed cToken in the protocol. `_addMarketInternal` appends the new cToken to it. The loop before the push checks that the cToken is not already present. In practice, a market cannot reach this point twice because `_supportMarket` already rejects listed markets, but the redundant check makes the invariant explicit at the array level as well.

### `_initializeMarket`

`_initializeMarket` sets up the initial state for COMP distribution for the new market. The details belong to the COMP flywheel, which will be covered in a later article.

## Entering a Market

Once a market is listed, users can enter it. Supplying an asset to Compound does not require entering its market. You can mint cTokens and earn yield without ever entering. Entering is what determines how that supply is treated by the Comptroller. Until a user enters a market, their supplied balance in that market is invisible to the liquidity calculation and cannot be used as collateral. To borrow from any market, the user must also enter it first, even if they have deposited nothing into it. You cannot borrow DAI without entering the DAI market.

Exiting a market is more involved and is covered in the next article.

### Redundant State for Efficient Lookups

When a user enters a market, `addToMarketInternal` records that membership in two places simultaneously:

```
// Comptroller.addToMarketInternal()
marketToJoin.accountMembership[borrower] = true;
accountAssets[borrower].push(cToken);
```

Both state variables represent the same information. The duplication is a deliberate optimization. `accountAssets` is the right structure when the protocol needs to iterate over all markets a user has entered. `marketToJoin.accountMembership` is the right structure for a targeted boolean check. The mapping answers that in O(1) without touching the array. Liquidity checks happen on every borrow, redeem, and transfer, so having both structures available avoids unnecessary iteration across high-traffic accounts.

## Conclusion

Market listing and market entry are the two gates that control access to Compound. Listing is an admin operation that registers a new asset and initializes its state. Entering is a user operation that opts an account into a market, either to use it as collateral or to borrow from it.
