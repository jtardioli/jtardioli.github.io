---
layout: post
title: 'Compound V2: Minting and Redeeming'
date: 2026-03-06 10:49 -0500
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, mint, redeem, minting, redeeming ]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 6
---

> This article is part of the Compound V2 series. See the [series index](/posts/compound-v2-introduction/#series-index) for a full list of articles.


This article walks through two core operations in Compound's `cToken` contract: minting and redeeming. Minting is how a user supplies underlying tokens to the protocol and receives `cTokens` in return. Redeeming is the reverse: the user returns `cTokens` and gets their underlying tokens back. Both operations share a common structure, and understanding one makes the other straightforward.



This article assumes familiarity with the [cToken exchange rate](https://cursedid0l.github.io/posts/compound-v2-ctokens-and-the-exhange-rate/). If you haven't read that article yet, start there.



---



## Minting



### MintInternal



```

function mintInternal(uint mintAmount) internal nonReentrant {

accrueInterest();

// mintFresh emits the actual Mint event if successful and logs on errors

mintFresh(msg.sender, mintAmount);

}

```



`mintInternal` is the entry point for minting. It calls `accrueInterest` to bring interest up to date for the current block, then delegates to `mintFresh` where the actual work happens.



### Comptroller Check



The first thing `mintFresh` does is ask the comptroller whether minting is allowed:



```

uint allowed = comptroller.mintAllowed(address(this), minter, mintAmount);

if (allowed != 0) {

revert MintComptrollerRejection(allowed);

}

```



If the comptroller returns anything other than `0`, minting is rejected and the transaction reverts. The comptroller will be covered separately. For now, treat it as a black box that enforces protocol-level rules.



### Accrual Check



```

if (accrualBlockNumber != getBlockNumber()) {

revert MintFreshnessCheck();

}

```



Before doing anything with amounts, `mintFresh` verifies that `accrualBlockNumber` equals the current block number. This confirms that `accrueInterest` has already run in this block. It is a safety check that reasserts an invariant: `mintInternal` should have already called `accrueInterest`, but `mintFresh` does not take that on trust.



This matters because the exchange rate between `cTokens` and the underlying asset is derived from accrued interest. If interest has not been accrued yet this block, the exchange rate is stale and minting at that rate would produce incorrect token amounts.



### Getting the Exchange Rate



```

Exp memory exchangeRate = Exp({mantissa: exchangeRateStoredInternal()});

```



The exchange rate is fetched and stored locally. It will be used to calculate how many `cTokens` the minter receives for their underlying deposit.



### Transferring In the Underlying Tokens



```

uint actualMintAmount = doTransferIn(minter, mintAmount);

```



`doTransferIn` pulls the underlying tokens from the minter into the contract. It handles the difference between ETH and ERC-20 underlying assets internally: for ERC-20 tokens it calls `transferFrom` on the token contract, and for ETH it reads `msg.value` directly.



The return value, `actualMintAmount`, is the amount the contract actually received. This matters because some ERC-20 tokens have fee-on-transfer mechanics. If 100 tokens are sent, the contract might only receive 98. Using the requested `mintAmount` directly would cause accounting errors, crediting the user for tokens the protocol never received. Using `actualMintAmount` ensures the protocol only credits what it actually holds.



### Calculating cTokens to Mint



```

uint mintTokens = div_(actualMintAmount, exchangeRate);

```



The number of `cTokens` to issue is `actualMintAmount / exchangeRate`. A higher exchange rate means each `cToken` is worth more underlying, so the minter receives fewer `cTokens` for the same deposit.



For example, if the exchange rate is `0.02` and the deposit is `1 USDC`, the minter receives `1 / 0.02 = 50 cUSDC`. A year later, if the exchange rate has risen to `0.022` due to accrued interest, the same `1 USDC` deposit only gets `1 / 0.022 ≈ 45.5 cUSDC`. The earlier 50 `cUSDC` are now worth more than the original deposit. Each existing `cToken` can be redeemed for more underlying than before.



### Updating State



```

totalSupply = totalSupply + mintTokens;

accountTokens[minter] = accountTokens[minter] + mintTokens;

```



The total supply of `cTokens` and the minter's individual balance are both incremented.



### Events



```

emit Mint(minter, actualMintAmount, mintTokens);

emit Transfer(address(this), minter, mintTokens);

```



Two events are emitted. `Mint` records the operation with the actual underlying amount deposited and the `cTokens` issued. `Transfer` records the `cToken` transfer from the contract to the minter, following the ERC-20 standard.



---



## Redeeming



Unlike minting, redeeming has two internal entry points before funneling into the same core function:



```

function redeemInternal(uint redeemTokens) internal nonReentrant {

accrueInterest();

redeemFresh(payable(msg.sender), redeemTokens, 0);

}



function redeemUnderlyingInternal(uint redeemAmount) internal nonReentrant {

accrueInterest();

redeemFresh(payable(msg.sender), 0, redeemAmount);

}

```



`redeemInternal` is for users who want to specify how many `cTokens` to return. `redeemUnderlyingInternal` is for users who want to specify how much underlying they want to receive. Both call `accrueInterest` first, then pass their value into `redeemFresh` with the other argument set to zero. The zero acts as a signal for which input was provided.



### Two Inputs, One at a Time



`redeemFresh` starts with a guard to enforce that exactly one of the two inputs is non-zero:



```

require(redeemTokensIn == 0 || redeemAmountIn == 0, "one of redeemTokensIn or redeemAmountIn must be zero");

```



Then it resolves both values from whichever was provided:



```

Exp memory exchangeRate = Exp({mantissa: exchangeRateStoredInternal()});



uint redeemTokens;

uint redeemAmount;



if (redeemTokensIn > 0) {

redeemTokens = redeemTokensIn;

redeemAmount = mul_ScalarTruncate(exchangeRate, redeemTokensIn);

} else {

redeemTokens = div_(redeemAmountIn, exchangeRate);

redeemAmount = redeemAmountIn;

}

```



If the user supplied `cTokens`, the underlying amount is calculated as `redeemTokensIn × exchangeRate`. If the user supplied the underlying amount, the `cTokens` to burn is calculated as `redeemAmountIn / exchangeRate`. After this block, both `redeemTokens` and `redeemAmount` are known regardless of which path was taken.



For example, if a user holds `50 cUSDC` and the exchange rate is `0.022`, redeeming via `redeemInternal` returns `50 × 0.022 = 1.1 USDC`. Redeeming is the exact inverse of minting.



### Comptroller and Freshness Checks



```

uint allowed = comptroller.redeemAllowed(address(this), redeemer, redeemTokens);

if (allowed != 0) {

revert RedeemComptrollerRejection(allowed);

}



if (accrualBlockNumber != getBlockNumber()) {

revert RedeemFreshnessCheck();

}

```



The comptroller is asked whether the redeem is allowed, and the accrual block number is verified to match the current block. Treat the comptroller as a black box for now. The freshness check serves the same purpose as in minting, the exchange rate used in the calculation above must be up to date.



### Cash Check



```

if (getCashPrior() < redeemAmount) {

revert RedeemTransferOutNotPossible();

}

```



Before modifying any state, the function confirms the contract actually holds enough underlying to cover the redemption. `getCashPrior()` returns the contract's current balance of the underlying asset.



This situation arises when a large portion of the supplied assets have been borrowed out. Compound does not guarantee 100% liquidity, rather it allows utilization up to some limit, so if utilization is very high there may not be enough cash on hand to cover a redemption.



### Updating State



```

totalSupply = totalSupply - redeemTokens;

accountTokens[redeemer] = accountTokens[redeemer] - redeemTokens;

```



The redeemer's `cToken` balance and the total supply are both decremented before the underlying is sent out. State is updated first, then the transfer happens.



### Transferring Out the Underlying



```

doTransferOut(redeemer, redeemAmount);

```



`doTransferOut` sends the underlying tokens to the redeemer. Like `doTransferIn`, it handles both ETH and ERC-20 variants: for ERC-20 it calls `token.transfer(redeemer, redeemAmount)`, and for ETH it calls `redeemer.transfer(redeemAmount)`.



### Events and Verify Hook



```

emit Transfer(redeemer, address(this), redeemTokens);

emit Redeem(redeemer, redeemAmount, redeemTokens);

comptroller.redeemVerify(address(this), redeemer, redeemAmount, redeemTokens);

```



`Transfer` records the `cTokens` moving from the redeemer back to the contract. `Redeem` records the full operation. `redeemVerify` can be ignored for now.



---



## Conclusion



Minting and redeeming are inverse operations built around the same exchange rate. Minting divides underlying by the exchange rate to produce `cTokens`. Redeeming multiplies `cTokens` by the exchange rate to recover underlying. As interest accrues over time the exchange rate rises, meaning each `cToken` is redeemable for more underlying than when it was minted. That appreciation is how suppliers earn yield in Compound.
