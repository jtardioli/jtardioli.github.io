---
layout: post
title: 'Compound V2: COMP Reward Distribution'
date: 2026-03-28 10:00 -0500
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, comp, flywheel, rewards, distribution]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 21
---

> This article is part of the Compound V2 series. See the [series index](/posts/compound-v2-introduction/#series-index) for a full list of articles.

The flywheel is Compound's system for distributing COMP tokens to suppliers and borrowers. Users earn COMP rewards just by using Compound.

The problem is splitting COMP fairly across thousands of users across multiple markets, with balances changing every block. Looping through every user would cost too much gas.

The solution is the same index pattern sued for the `borrowIndex`. Instead of tracking what each user is owed, the contract maintains a single global index that increases over time. Each user stores a snapshot of the index from their last interaction. When they interact again, the difference between the current index and their snapshot tells the contract exactly how much COMP they earned, without looping through other users.

The key difference from the borrow index is the math. The borrow index uses division (`currentIndex / userIndex`) because debt compounds multiplicatively. The flywheel uses subtraction (`currentIndex - userIndex`) because COMP rewards are additive.

Governance controls how much COMP flows to each market via `compSupplySpeeds` and `compBorrowSpeeds`, which set the COMP per block for suppliers and borrowers respectively. A market with a higher speed distributes more COMP to its users.

The COMP tokens themselves must be held by the Comptroller. They get there through the Reservoir contract (`contracts/Reservoir.sol`), which holds a supply of COMP and drips it to the Comptroller at a fixed rate per block. Anyone can call `drip()` on the Reservoir to transfer the accumulated COMP since the last drip. The Comptroller then distributes from its own balance.

## State Variables

All flywheel state lives in the Comptroller. The core struct is `CompMarketState`:

```
struct CompMarketState {
    uint224 index; // cumulative COMP per unit (cToken or borrowed unit)
    uint32 block;  // block number the index was last updated
}
```

The remaining state variables:

- `compSupplyState[cToken]`: a `CompMarketState` for the supply side of each market
- `compBorrowState[cToken]`: same for the borrow side
- `compSupplySpeeds[cToken]`: COMP per block distributed to suppliers of this market (set by governance)
- `compBorrowSpeeds[cToken]`: same for borrowers
- `compSupplierIndex[cToken][user]`: the user's snapshot of the supply index from their last interaction
- `compBorrowerIndex[cToken][user]`: same for the borrow side


- `compAccrued[user]`: total unclaimed COMP a user has earned across all markets
- `compInitialIndex`: `1e36`, the starting value for all indexes

The first four are per-market globals. The next two are per-user per-market. `compAccrued` is per-user across all markets.

### Why `compInitialIndex` Is 1e36

Every market's index starts at `1e36` instead of 0. If it started at 0, there would be no way to distinguish between a user who supplied before COMP rewards were enabled and a user who never interacted at all, since both would have a snapshot of 0. Starting at `1e36` means anyone who has actually interacted has a snapshot of at least `1e36`. A snapshot of 0 tells the contract the user has never been tracked.

The `1e36` baseline cancels out in the math. When Alice first supplies, her snapshot is set to `1e36`. After some blocks, the index might be `1e36 + 500`. Alice's reward is proportional to 500, the difference. The baseline has no effect on the result.

## Updating the Supply Index

The supply index tracks cumulative COMP earned per cToken since the market was created. The formula:

```
supplyIndex = oldIndex + (deltaBlocks * compSupplySpeed) / totalSupply
```

`deltaBlocks * compSupplySpeed` is the total COMP earned by all suppliers in this market since the last update. Dividing by `totalSupply` (the total cTokens in existence) gives the COMP earned per cToken over that period. Adding it to the old index keeps the running total.

```
function updateCompSupplyIndex(address cToken) internal {
    CompMarketState storage supplyState = compSupplyState[cToken];
    uint supplySpeed = compSupplySpeeds[cToken];
    uint32 blockNumber = safe32(getBlockNumber(), "block number exceeds 32 bits");
    uint deltaBlocks = sub_(uint(blockNumber), uint(supplyState.block));
```

Get the current supply state, the speed for this market, and calculate how many blocks have passed since the last update.

```
    if (deltaBlocks > 0 && supplySpeed > 0) {
        uint supplyTokens = CToken(cToken).totalSupply();
        uint compAccrued = mul_(deltaBlocks, supplySpeed);
        Double memory ratio = supplyTokens > 0 ? fraction(compAccrued, supplyTokens) : Double({mantissa: 0});
        supplyState.index = safe224(add_(Double({mantissa: supplyState.index}), ratio).mantissa, "new index exceeds 224 bits");
        supplyState.block = blockNumber;
    } else if (deltaBlocks > 0) {
        supplyState.block = blockNumber;
    }
}
```

If blocks have passed and the market has a supply speed, calculate the COMP accrued (`deltaBlocks * supplySpeed`), divide by total cToken supply to get the per-cToken amount, and add it to the index. The `fraction` function scales both numbers up before dividing to avoid precision loss, the same helper covered in an earlier article. If blocks have passed but the supply speed is zero (rewards are turned off), only the block number is updated.

## Distributing Supplier COMP

`distributeSupplierComp` calculates how much COMP a specific supplier has earned since their last interaction and adds it to their `compAccrued` balance.

```
function distributeSupplierComp(address cToken, address supplier) internal {
    CompMarketState storage supplyState = compSupplyState[cToken];
    uint supplyIndex = supplyState.index;
    uint supplierIndex = compSupplierIndex[cToken][supplier];

    compSupplierIndex[cToken][supplier] = supplyIndex;
```

Get the current global supply index and the user's snapshot. Immediately update the user's snapshot to the current index.

```
    if (supplierIndex == 0 && supplyIndex >= compInitialIndex) {
        supplierIndex = compInitialIndex;
    }
```

If the user's snapshot is 0, they supplied before COMP rewards were enabled for this market. Set their snapshot to `compInitialIndex` so they earn rewards from the point rewards were turned on, not from the beginning of time.

```
    Double memory deltaIndex = Double({mantissa: sub_(supplyIndex, supplierIndex)});

    uint supplierTokens = CToken(cToken).balanceOf(supplier);

    uint supplierDelta = mul_(supplierTokens, deltaIndex);

    uint supplierAccrued = add_(compAccrued[supplier], supplierDelta);
    compAccrued[supplier] = supplierAccrued;

    emit DistributedSupplierComp(CToken(cToken), supplier, supplierDelta, supplyIndex);
}
```

Calculate the delta between the current index and the user's snapshot. Multiply by the user's cToken balance to get their COMP earned. Add it to their running `compAccrued` total. The supply index tracks cumulative COMP per cToken, so a user holding 10% of all cTokens earns 10% of the COMP distributed to that market.

## The Borrow Side

`updateCompBorrowIndex` and `distributeBorrowerComp` follow the same pattern as their supply counterparts. The only difference is what they divide by.

On the supply side, the index tracks COMP per cToken, so it divides by `totalSupply` (total cTokens). On the borrow side, the index tracks COMP per unit of borrowed principal, so it divides by the total borrowed principal.

```
uint borrowAmount = div_(CToken(cToken).totalBorrows(), marketBorrowIndex);
```

`totalBorrows()` returns the total amount borrowed including accrued interest. Dividing by the `marketBorrowIndex` strips out the interest and gives the original borrowed principal. This is the same relationship from Article 3. COMP rewards are proportional to how much you borrowed, not how much interest you owe.

Similarly, `distributeBorrowerComp` uses the borrower's principal (not their balance with interest) when calculating their share:

```
uint borrowerAmount = div_(CToken(cToken).borrowBalanceStored(borrower), marketBorrowIndex);
```

Everything else is identical to the supply side.

## When the Flywheel Runs

The flywheel updates happen automatically as side effects of normal user actions. The Comptroller calls `updateCompSupplyIndex` and `distributeSupplierComp` inside `mintAllowed`, `redeemAllowed`, `seizeAllowed`, and `transferAllowed`. It calls `updateCompBorrowIndex` and `distributeBorrowerComp` inside `borrowAllowed` and `repayBorrowAllowed`.

This means every time a user mints, redeems, borrows, repays, gets liquidated, or transfers cTokens, the flywheel updates for them. Users accumulate COMP without doing anything special.

## Claiming COMP

The functions above accumulate COMP in `compAccrued[user]` but do not transfer any tokens. To actually receive COMP, a user calls `claimComp`.

```
function claimComp(address[] memory holders, CToken[] memory cTokens, bool borrowers, bool suppliers) public {
    for (uint i = 0; i < cTokens.length; i++) {
        CToken cToken = cTokens[i];
        require(markets[address(cToken)].isListed, "market must be listed");
        if (borrowers == true) {
            Exp memory borrowIndex = Exp({mantissa: cToken.borrowIndex()});
            updateCompBorrowIndex(address(cToken), borrowIndex);
            for (uint j = 0; j < holders.length; j++) {
                distributeBorrowerComp(address(cToken), holders[j], borrowIndex);
            }
        }
        if (suppliers == true) {
            updateCompSupplyIndex(address(cToken));
            for (uint j = 0; j < holders.length; j++) {
                distributeSupplierComp(address(cToken), holders[j]);
            }
        }
    }
    for (uint j = 0; j < holders.length; j++) {
        compAccrued[holders[j]] = grantCompInternal(holders[j], compAccrued[holders[j]]);
    }
}
```

The function takes arrays of holders and markets, plus two booleans to select borrow rewards, supply rewards, or both. For each market, it updates the relevant index and distributes COMP to each holder. Notice that anyone can trigger the claim for any address, not just the holder themselves.

After all distributions are calculated, `grantCompInternal` handles the actual transfer:

```
function grantCompInternal(address user, uint amount) internal returns (uint) {
    Comp comp = Comp(getCompAddress());
    uint compRemaining = comp.balanceOf(address(this));
    if (amount > 0 && amount <= compRemaining) {
        comp.transfer(user, amount);
        return 0;
    }
    return amount;
}
```

The function checks if the Comptroller has enough COMP to pay out. If yes, it transfers the tokens and returns 0, which sets `compAccrued[user]` to 0. If the Comptroller does not have enough COMP, it skips the transfer and returns the original amount, so `compAccrued[user]` keeps the balance and the user can try again later. This means the claim silently does nothing if the Comptroller's COMP balance is insufficient. The transaction does not revert, so a user could call `claimComp` and receive nothing without an error.

## Conclusion

The flywheel distributes COMP to suppliers and borrowers using the same index pattern as the borrow interest system. A global index tracks cumulative COMP per unit, each user stores a snapshot, and the difference gives their earned rewards. The flywheel updates automatically on every user interaction, and users can claim their accumulated COMP at any time.
