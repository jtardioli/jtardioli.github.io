---
layout: post
title: 'Compound V2: The Comptroller'
date: 2026-03-18 10:00 -0500
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, comptroller, unitroller]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 11
---

> This article is part of the Compound V2 series. See the [series index](/posts/compound-v2-introduction/#series-index) for a full list of articles.

The Comptroller is the risk engine of Compound. While the cToken contracts handle the mechanics of moving tokens in and out, they defer every decision to the Comptroller. Before a user can mint, borrow, redeem, or liquidate, the cToken asks the Comptroller for permission. The Comptroller enforces collateral factors, checks whether a position is solvent, determines how much of a borrow can be liquidated, and distributes COMP rewards.

## The Unitroller

The `Unitroller` is the proxy contract that sits in front of the Comptroller. When a cToken stores the Comptroller address, it is actually storing the address of the `Unitroller`. Calls pass through the proxy to the current Comptroller implementation. There is not much else to say here as the proxy pattern was covered in a previous article.

## The Comptroller Interface

The `ComptrollerInterface` defines all the functions a Comptroller implementation must expose. cTokens do not call the Comptroller directly by its concrete type. They call it through this interface, which means the underlying implementation can be upgraded as long as the interface stays consistent.

## Comptroller Storage

If you look at the codebase you will notice several versioned storage contracts: `ComptrollerV1Storage`, `ComptrollerV2Storage`, and so on. Each new version inherits from the previous one. New variables are only ever appended at the end.

This pattern exists to prevent storage collisions across upgrades. Because the `Unitroller` delegates calls to the Comptroller implementation, both contracts share the same storage layout. If a new Comptroller version reordered or removed variables from storage, it would corrupt the data already written there. Inheriting from the previous version guarantees that existing variable slots are never touched, and new variables always land in fresh slots.

```
ComptrollerV1Storage
        ↓
ComptrollerV2Storage
        ↓
       ...
        ↓
ComptrollerV7Storage
        ↓
      Comptroller
```

Each arrow means "inherits from." `Comptroller` sits at the bottom and inherits the full accumulated storage layout. New variables were only ever added at the bottom of each new version, never inserted or removed from an earlier slot. The codebase also contains `ComptrollerG7`, which is a previous Comptroller implementation that was active before the current one. It inherits from `ComptrollerV5Storage` and contains largely the same logic. The Unitroller now points to `Comptroller`, so `ComptrollerG7` is historical.

## Conclusion

The Comptroller is built on a few straightforward structural pieces. The `Unitroller` acts as a stable proxy address so cTokens never need to be updated when the Comptroller logic changes. The `ComptrollerInterface` decouples the cTokens from any specific implementation. The versioned storage contracts ensure that upgrades never corrupt existing state. With that foundation in place, the next article gets into the actual logic of how the comptroller decides if an account is solvent.
