---
layout: post
title: 'Compound V2: CErc20 and CEther'
date: 2026-03-14 19:34 -0400
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, cerc20, cether, erc20, transfer]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 9
---

> This article is part of the Compound V2 series. See the [series index](/posts/compound-v2-introduction/#series-index) for a full list of articles.

`CToken` is an abstract contract. It defines the core logic for minting, redeeming, borrowing, repaying, and liquidating, but it leaves three functions unimplemented:

- `getCashPrior`
- `doTransferIn`
- `doTransferOut`

These three functions are the only meaningful difference between `CErc20` and `CEther`. Everything else is inherited from `CToken`. `CErc20` implements them for ERC-20 underlying assets, and `CEther` implements them for ETH.

---

## CEther

### getCashPrior

```
function getCashPrior() override internal view returns (uint) {
    return address(this).balance - msg.value;
}
```

`getCashPrior` returns the contract's ETH balance before the current transaction. For ETH, that is simply `address(this).balance` minus `msg.value`. The subtraction is necessary because by the time this function is called, the incoming ETH has already been added to the contract's balance. Subtracting `msg.value` recovers what the balance was before the current call.

### doTransferIn

```
function doTransferIn(address from, uint amount) override internal returns (uint) {
    require(msg.sender == from, "sender mismatch");
    require(msg.value == amount, "value mismatch");
    return amount;
}
```

For ETH, there is no token transfer to perform. The ETH arrives with the transaction via `msg.value`. `doTransferIn` simply validates that the sender matches `from` and that `msg.value` matches the expected `amount`, then returns `amount` directly. There is no fee-on-transfer concern with ETH, so the actual and expected amounts are always equal.

### doTransferOut

```
function doTransferOut(address payable to, uint amount) virtual override internal {
    to.transfer(amount);
}
```

`doTransferOut` sends ETH to the recipient using `transfer`, which forwards a fixed 2300 gas stipend and reverts on failure.

---

## CErc20

### getCashPrior

```
function getCashPrior() virtual override internal view returns (uint) {
    EIP20Interface token = EIP20Interface(underlying);
    return token.balanceOf(address(this));
}
```

For ERC-20 tokens, `getCashPrior` returns the contract's current token balance via `balanceOf`. What makes this "prior" is not the function itself but where it is called: it is always called before `doTransferIn`. If it were called after, it would include the just-transferred tokens and no longer represent the prior balance.

### doTransferIn

```
function doTransferIn(address from, uint amount) virtual override internal returns (uint) {
    address underlying_ = underlying;
    EIP20NonStandardInterface token = EIP20NonStandardInterface(underlying_);
    uint balanceBefore = EIP20Interface(underlying_).balanceOf(address(this));
    token.transferFrom(from, address(this), amount);

    bool success;
    assembly {
        switch returndatasize()
            case 0 {                       // This is a non-standard ERC-20
                success := not(0)          // set success to true
            }
            case 32 {                      // This is a compliant ERC-20
                returndatacopy(0, 0, 32)
                success := mload(0)        // Set success = returndata of external call
            }
            default {                      // This is an excessively non-compliant ERC-20, revert.
                revert(0, 0)
            }
    }
    require(success, "TOKEN_TRANSFER_IN_FAILED");

    uint balanceAfter = EIP20Interface(underlying_).balanceOf(address(this));
    return balanceAfter - balanceBefore;
}
```

`doTransferIn` pulls ERC-20 tokens from `from` into the contract. The implementation is more involved than the ETH version for two reasons: non-standard token return values, and fee-on-transfer tokens.

**Non-standard return values**

The ERC-20 standard requires `transfer` and `transferFrom` to return a `bool`. In practice, some widely used tokens, notably USDC at the time Compound was deployed, do not return a boolean. If the call fails, they revert. If it succeeds, they return nothing.

Solidity's default ABI decoder would revert when trying to decode a missing return value, so the assembly block handles both cases manually. `returndatasize()` returns the byte length of the return data from the most recent external call, in this case the `transferFrom`.

- If `returndatasize()` is `0`, no value was returned. The function treats this optimistically and sets `success` to `not(0)`, which is `true` in assembly.
- If `returndatasize()` is `32`, a standard boolean was returned. The assembly copies it into scratch space and reads it into `success`.
- Any other size reverts, as the token is behaving in an unexpected way.

After the assembly block, `success` is checked with `require`.

**Fee-on-transfer tokens**

Rather than trusting that the contract received exactly `amount`, `doTransferIn` measures the actual balance change by comparing `balanceBefore` and `balanceAfter`. If the underlying token charges a fee on transfer, the contract may have received less than `amount`. Returning the difference ensures that the rest of the protocol only credits what was actually received.

### doTransferOut

```
function doTransferOut(address payable to, uint amount) virtual override internal {
    EIP20NonStandardInterface token = EIP20NonStandardInterface(underlying);
    token.transfer(to, amount);

    bool success;
    assembly {
        switch returndatasize()
            case 0 {                      // This is a non-standard ERC-20
                success := not(0)          // set success to true
            }
            case 32 {                     // This is a compliant ERC-20
                returndatacopy(0, 0, 32)
                success := mload(0)        // Set success = returndata of external call
            }
            default {                     // This is an excessively non-compliant ERC-20, revert.
                revert(0, 0)
            }
    }
    require(success, "TOKEN_TRANSFER_OUT_FAILED");
}
```

`doTransferOut` uses the same assembly pattern as `doTransferIn` to handle non-standard return values. The only difference is direction: `transfer` is called instead of `transferFrom`, sending tokens from the contract to `to`. There is no balance check here because the amount going out is already known and validated upstream.

---

## Conclusion

`CErc20` and `CEther` are thin wrappers around `CToken` that handle the practical differences between transferring ETH and ERC-20 tokens. The assembly in `doTransferIn` and `doTransferOut` is the most unusual code in the codebase, but its purpose is straightforward: work correctly with tokens that do not follow the ERC-20 standard precisely.
