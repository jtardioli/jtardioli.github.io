---
layout: post
title: 'Compound V2: The Proxy Pattern'
date: 2026-03-14 19:36 -0400
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, proxy, delegatecall, cerc20]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 10
---

> This article is part of the Compound V2 series. See the [series index](/posts/compound-v2-introduction/#series-index) for a full list of articles.

The blockchain is immutable. Once a contract is deployed, there is no way to update it. This is by design, but it creates a practical problem: if a bug is discovered or an optimization identified, an entirely new contract must be deployed. Every user and frontend that stored the old address has to migrate.

Proxy patterns solve this. They allow a contract's logic to be updated while keeping its address fixed. Users and frontends interact with one stable address, and the underlying logic can be swapped without anyone needing to update their references.

This convenience comes with a real tradeoff: proxies can be abused. If a protocol can update its implementation without restriction, it can introduce malicious logic, drain funds, or rug users. This is an important consideration when evaluating any protocol that uses upgradeable contracts.

This article covers how Compound uses the proxy pattern in `CErc20Delegator` and `CErc20Delegate`. If you are not already familiar with `delegatecall`, read up on it before continuing. [RareSkills has a good primer.](https://rareskills.io/post/delegatecall)

---

## How the Proxy Pattern Works

The pattern splits one contract into two:

- The **proxy** (called the **delegator** in Compound's terminology) holds all the storage.
- The **implementation** (called the **delegate**) holds all the logic.

When a user calls the proxy, the proxy forwards the call to the implementation using `delegatecall`. Because `delegatecall` executes the implementation's code in the context of the proxy's storage, meaning any state reads and writes operate on the proxy's storage slots, not the implementation's, all state changes land in the proxy.

The result is that users only ever need to know the proxy's address. When the protocol deploys a new implementation, it updates a single pointer inside the proxy. Every subsequent call is automatically routed to the new implementation.

---

## Storage Layout

The critical constraint when using `delegatecall` is that both contracts must share the same storage layout. Storage in the EVM is slot-based: the first declared variable occupies slot 0, the second slot 1, and so on. If the proxy declares `address admin` in slot 0 and the implementation declares `uint256 totalSupply` in slot 0, any write to `totalSupply` in the implementation will silently overwrite `admin` in the proxy.

In Compound's case, both `CErc20Delegator` and `CErc20Delegate` inherit their storage variables from the same base contracts. This guarantees their layouts remain in sync automatically.

---

## A Closer Look at CErc20Delegator

### Wrapper Functions

Most functions in `CErc20Delegator` follow the same pattern:

```
function _setInterestRateModel(InterestRateModel newInterestRateModel) override public returns (uint) {
    bytes memory data = delegateToImplementation(abi.encodeWithSignature("_setInterestRateModel(address)", newInterestRateModel));
    return abi.decode(data, (uint));
}
```

These are typed wrapper functions. Each one ABI-encodes a function signature and its arguments, forwards the call to the implementation via `delegatecall`, then decodes and returns the result. The wrappers exist because they give callers compile-time type checking and allow external contracts to interact with the proxy as if it were the implementation directly.

### delegateTo

`delegateTo` is the core internal function that performs the actual `delegatecall`:

```
function delegateTo(address callee, bytes memory data) internal returns (bytes memory) {
    (bool success, bytes memory returnData) = callee.delegatecall(data);
    assembly {
        if eq(success, 0) {
            revert(add(returnData, 0x20), returndatasize())
        }
    }
    return returnData;
}
```

It issues a `delegatecall` to `callee` with `data`, then checks whether the call succeeded. If it did not, it reverts with the raw error bytes. The `add(returnData, 0x20)` offset skips the ABI-encoded length prefix of the `bytes` array, so the revert bubbles up the underlying error data rather than the length-prefixed wrapper.

### delegateToImplementation

`delegateToImplementation` is a thin wrapper around `delegateTo` that hardcodes the implementation address:

```
function delegateToImplementation(bytes memory data) public returns (bytes memory) {
    return delegateTo(implementation, data);
}
```

This is what the typed wrappers call internally.

### delegateToViewImplementation

```
function delegateToViewImplementation(bytes memory data) public view returns (bytes memory) {
    (bool success, bytes memory returnData) = address(this).staticcall(abi.encodeWithSignature("delegateToImplementation(bytes)", data));
    assembly {
        if eq(success, 0) {
            revert(add(returnData, 0x20), returndatasize())
        }
    }
    return abi.decode(returnData, (bytes));
}
```

The Solidity compiler will not allow a function marked `view` to call `delegatecall` internally, because it has no way to verify that the target function does not modify state. The workaround is to wrap the call in a `staticcall`. A `staticcall` enforces read-only behavior at the EVM level: any state change inside it causes an automatic revert. This satisfies the compiler and is safe in practice.

This matters because ERC-20 requires functions like `balanceOf` to be marked `view`. Since the actual logic lives in the implementation, this mechanism is needed to call through `delegatecall` while satisfying that constraint.

The `abi.decode` at the end strips one layer of encoding. The return data gets encoded twice: once by the inner `delegatecall` inside `delegateToImplementation`, and again by the outer `staticcall`. The decode leaves the raw return data from the `delegatecall`, consistent with what the other functions return.

---

## The Constructor and Initialization

The constructor sets the admin to whoever deployed the proxy:

```
admin = payable(msg.sender);
```

It then calls `initialize` on the implementation via `delegatecall`, so the initialization logic executes inside the proxy's storage:

```
delegateTo(
    implementation_,
    abi.encodeWithSignature(
        "initialize(address,address,address,uint256,string,string,uint8)",
        underlying_,
        comptroller_,
        interestRateModel_,
        initialExchangeRateMantissa_,
        name_,
        symbol_,
        decimals_
    )
);
```

The reason `initialize` is used rather than a constructor on the implementation is that constructor code runs at deployment time and is not stored on-chain. There is nothing to `delegatecall` into. Using an `initialize` function is the standard workaround.

### CErc20.initialize

`CErc20.initialize` calls `super.initialize` and then sets the underlying token address:

```
function initialize(
    address underlying_,
    ComptrollerInterface comptroller_,
    InterestRateModel interestRateModel_,
    uint initialExchangeRateMantissa_,
    string memory name_,
    string memory symbol_,
    uint8 decimals_
) public {
    super.initialize(comptroller_, interestRateModel_, initialExchangeRateMantissa_, name_, symbol_, decimals_);
    underlying = underlying_;
    EIP20Interface(underlying).totalSupply();
}
```

The call to `EIP20Interface(underlying).totalSupply()` is a sanity check confirming that the underlying token contract is live and conforms to ERC-20. If the address is invalid, this line reverts and the entire initialization fails.

### CToken.initialize

`super.initialize` lands in `CToken.initialize`:

```
function initialize(
    ComptrollerInterface comptroller_,
    InterestRateModel interestRateModel_,
    uint initialExchangeRateMantissa_,
    string memory name_,
    string memory symbol_,
    uint8 decimals_
) public {
    require(msg.sender == admin, "only admin may initialize the market");
    require(accrualBlockNumber == 0 && borrowIndex == 0, "market may only be initialized once");

    initialExchangeRateMantissa = initialExchangeRateMantissa_;
    require(initialExchangeRateMantissa > 0, "initial exchange rate must be greater than zero.");

    uint err = _setComptroller(comptroller_);
    require(err == NO_ERROR, "setting comptroller failed");

    accrualBlockNumber = getBlockNumber();
    borrowIndex = mantissaOne;

    err = _setInterestRateModelFresh(interestRateModel_);
    require(err == NO_ERROR, "setting interest rate model failed");

    name = name_;
    symbol = symbol_;
    decimals = decimals_;

    _notEntered = true;
}
```

The key guard is `require(msg.sender == admin, ...)`. Because this function runs via `delegatecall`, `msg.sender` is the original external caller of the proxy constructor: the deployer. This prevents any other party from calling `initialize` and hijacking the market setup.

One known risk with this pattern: if `initialize` on the implementation contract is never called directly, anyone can call it and set themselves as admin on the implementation. This does not affect the proxy's storage, but it can open up other attack surfaces. It is worth checking whether the implementation's `initialize` has been called when auditing a proxy-based protocol.

The `require(accrualBlockNumber == 0 && borrowIndex == 0, ...)` guard ensures `initialize` can only run once.

`_notEntered` is set to `true` rather than leaving it at its default of `false` to initialize the reentrancy guard. Writing to a non-zero storage slot costs significantly less gas than writing to a zero slot, so by starting at `true` and toggling to `false` during execution, every re-entrancy check after the first one is cheaper.

---

## Upgrading the Implementation

To upgrade to a new implementation, the admin calls `_setImplementation`. The core of the function is:

```
address oldImplementation = implementation;
implementation = implementation_;
```

The `implementation` pointer in the proxy is updated to the new contract address. All subsequent calls are routed to the new implementation.

Before updating the pointer, the proxy calls `_resignImplementation()` on the old implementation. After updating, it calls `_becomeImplementation()` on the new one. These hooks give each implementation a chance to perform any necessary cleanup or setup during a transition.

Only the admin can call `_setImplementation`. This means the security of the upgrade mechanism depends entirely on the security of the admin key. If the admin is a single EOA, a compromised key means a malicious upgrade. In practice, Compound's admin is a timelock contract controlled by governance, which provides an additional layer of protection.

---

## The Fallback Function

If a call arrives at the proxy with calldata that does not match any of the proxy's own function signatures, the fallback function forwards it directly to the implementation:

```
fallback() external payable {
    require(msg.value == 0, "CErc20Delegator:fallback: cannot send value to fallback");

    (bool success, ) = implementation.delegatecall(msg.data);

    assembly {
        let free_mem_ptr := mload(0x40)
        returndatacopy(free_mem_ptr, 0, returndatasize())
        switch success
        case 0 { revert(free_mem_ptr, returndatasize()) }
        default { return(free_mem_ptr, returndatasize()) }
    }
}
```

The assembly block copies the full return data from the `delegatecall` into memory at the free memory pointer, then either reverts or returns it. Raw assembly is used rather than Solidity's return handling because Solidity would add an extra ABI encoding layer, whereas the assembly passes the raw bytes back to the caller exactly as the implementation returned them.

The typed wrapper functions handle calls the proxy knows about at compile time. The fallback catches everything else. Together they ensure no call is ever dropped.

The `require(msg.value == 0)` guard rejects raw Ether. This is a `CErc20` proxy backed by an ERC-20 token. There is no mechanism to handle ETH.

---

## Conclusion

`CErc20Delegator` is a proxy that holds all state, and `CErc20Delegate` is the implementation that holds all logic. The pattern keeps the user-facing address stable across upgrades. The cost is that the upgrade mechanism itself becomes a trust assumption: whoever controls the admin key controls the implementation.
