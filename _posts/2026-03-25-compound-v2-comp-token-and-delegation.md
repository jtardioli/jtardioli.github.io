---
layout: post
title: 'Compound V2: COMP Token and Delegation'
date: 2026-03-25 10:00 -0500
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, governance, comp, delegation, checkpoints]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 18
---

> This article is part of the Compound V2 series. See the [series index](/posts/compound-v2-introduction/#series-index) for a full list of articles.

Previous articles in this series mentioned that Compound governance can update interest rate models, list new markets, close markets, and more. None of them covered who governs Compound or how the process actually works.

Compound uses a token-based governance model. Anyone can create a proposal, and anyone holding COMP tokens can vote on it. If enough votes are cast in favor, the proposal is executed on-chain. For example, a proposal might update the interest rate model on a particular market.

The governance system is managed entirely by smart contracts. This article focuses on the COMP token itself, found in `contracts/Governance/Comp.sol`. It implements the ERC-20 standard directly, with additional governance-specific functions.

## Delegation

One COMP token equals one vote, but owning tokens alone does not give you voting power. You must first delegate your tokens. Think of delegating as "activating" your tokens for voting.

As the name suggests, you can delegate your tokens to someone else. They can then vote with your tokens even while you hold them. Even if you want to vote yourself, you still need to delegate to your own address.

For example, Alice holds 100 COMP but has not delegated. She has 0 voting power. She calls `delegate(alice)` and now has 100 votes.

```
function _delegate(address delegator, address delegatee) internal {
    address currentDelegate = delegates[delegator];
    uint96 delegatorBalance = balances[delegator];
    delegates[delegator] = delegatee;

    emit DelegateChanged(delegator, currentDelegate, delegatee);

    _moveDelegates(currentDelegate, delegatee, delegatorBalance);
}
```

The `delegator` is the token owner and the `delegatee` is the person who will vote with those tokens. The function updates the `delegates` mapping with the new delegatee. Notice that each delegator can only have one delegatee at a time. The function then calls `_moveDelegates` to transfer the voting power from the old delegatee to the new one.

## Checkpoints

Before looking at `_moveDelegates`, checkpoints need to be understood. Each time a user's delegation changes, the contract takes a snapshot (checkpoint) of their voting power and records it along with the block number. Two mappings track this:

```
struct Checkpoint {
    uint32 fromBlock; // the block number of the snapshot
    uint96 votes; // number of votes at snapshot
}
```

- `checkpoints`: maps an address and an index to a `Checkpoint` struct
- `numCheckpoints`: maps an address to the number of checkpoints they have

Checkpoints exist to prevent two attacks. When you vote, your tokens are not burned or locked. You still have full control of them. This opens two attack vectors:

1. Alice votes with 100 tokens, then transfers them to Bob, who votes again with the same 100 tokens.
2. A user takes out a flash loan for a huge amount of COMP, votes with all of it, then returns the tokens at the end of the transaction.

To prevent both attacks, the COMP token snapshots voting power every time delegation changes. When a vote happens, it uses your balance from the most recent snapshot *before* the voting process began, not your current balance. This means transferring tokens after a proposal is created does not give the receiver voting power for that proposal, and flash-loaned tokens have no voting power because no snapshot exists for them before the proposal.

## `_moveDelegates`

```
function _moveDelegates(address srcRep, address dstRep, uint96 amount) internal {
    if (srcRep != dstRep && amount > 0) {
```

The function only runs if the source and destination are different and the amount is greater than zero.

```
        if (srcRep != address(0)) {
            uint32 srcRepNum = numCheckpoints[srcRep];
            uint96 srcRepOld = srcRepNum > 0 ? checkpoints[srcRep][srcRepNum - 1].votes : 0;
            uint96 srcRepNew = sub96(srcRepOld, amount, "Comp::_moveVotes: vote amount underflows");
            _writeCheckpoint(srcRep, srcRepNum, srcRepOld, srcRepNew);
        }
```

For the source (old delegatee), it gets the number of checkpoints, looks up the most recent vote count (`numCheckpoints - 1` is the index of the latest entry), subtracts the amount being moved, and writes a new checkpoint.

```
        if (dstRep != address(0)) {
            uint32 dstRepNum = numCheckpoints[dstRep];
            uint96 dstRepOld = dstRepNum > 0 ? checkpoints[dstRep][dstRepNum - 1].votes : 0;
            uint96 dstRepNew = add96(dstRepOld, amount, "Comp::_moveVotes: vote amount overflows");
            _writeCheckpoint(dstRep, dstRepNum, dstRepOld, dstRepNew);
        }
    }
}
```

The destination (new delegatee) follows the same pattern, except it adds the voting power instead of subtracting.

## `_writeCheckpoint`

The `_writeCheckpoint` function writes a new checkpoint with an important optimization:

```
function _writeCheckpoint(address delegatee, uint32 nCheckpoints, uint96 oldVotes, uint96 newVotes) internal {
    uint32 blockNumber = safe32(block.number, "Comp::_writeCheckpoint: block number exceeds 32 bits");

    if (nCheckpoints > 0 && checkpoints[delegatee][nCheckpoints - 1].fromBlock == blockNumber) {
        checkpoints[delegatee][nCheckpoints - 1].votes = newVotes;
    } else {
        checkpoints[delegatee][nCheckpoints] = Checkpoint(blockNumber, newVotes);
        numCheckpoints[delegatee] = nCheckpoints + 1;
    }

    emit DelegateVotesChanged(delegatee, oldVotes, newVotes);
}
```

If the most recent checkpoint was already written in the current block, it overwrites the vote count instead of creating a new entry. Notice that in this case `numCheckpoints` is not incremented, only the existing checkpoint's `votes` value is updated. This prevents multiple checkpoints from accumulating in the same block.

## `getPriorVotes`

Voting uses the most recent snapshot before a proposal began. `getPriorVotes` is the function that finds the voting power for an account at a specific block number.

```
function getPriorVotes(address account, uint blockNumber) public view returns (uint96) {
    require(blockNumber < block.number, "Comp::getPriorVotes: not yet determined");

    uint32 nCheckpoints = numCheckpoints[account];
    if (nCheckpoints == 0) {
        return 0;
    }
```

First it requires that the block number is in the past as a sanity check. Then it gets the number of checkpoints and returns zero if there are none.

```
    // First check most recent balance
    if (checkpoints[account][nCheckpoints - 1].fromBlock <= blockNumber) {
        return checkpoints[account][nCheckpoints - 1].votes;
    }

    // Next check implicit zero balance
    if (checkpoints[account][0].fromBlock > blockNumber) {
        return 0;
    }
```

Before doing any expensive searching, the function tries two quick checks. If the most recent checkpoint is from before the target block, that's the answer (early return). If the first checkpoint is from after the target block, the account had zero votes at that time (early return).

```
    uint32 lower = 0;
    uint32 upper = nCheckpoints - 1;
    while (upper > lower) {
        uint32 center = upper - (upper - lower) / 2; // ceil, avoiding overflow
        Checkpoint memory cp = checkpoints[account][center];
        if (cp.fromBlock == blockNumber) {
            return cp.votes;
        } else if (cp.fromBlock < blockNumber) {
            lower = center;
        } else {
            upper = center - 1;
        }
    }
    return checkpoints[account][lower].votes;
}
```

If neither shortcut works, the function runs a binary search over the checkpoints array:

1. Start with `lower = 0` and `upper = nCheckpoints - 1` as the search range.
2. Find the midpoint `center`.
3. If the checkpoint at `center` matches the target block exactly, return it.
4. If `fromBlock` at `center` is less than the target, the answer is at `center` or later, so `lower` moves up to `center`.
5. If `fromBlock` at `center` is greater than the target, the answer is before `center`, so `upper` moves down to `center - 1`.
6. Repeat until `lower` and `upper` converge. The checkpoint at `lower` is the most recent one at or before the target block.

For a visual explanation of how binary search works, [this short video from Harvard CS50](https://www.youtube.com/watch?v=YzT8zDPihmc) is a good resource.

## `_transferTokens`

`_transferTokens` handles all token transfers.

```
function _transferTokens(address src, address dst, uint96 amount) internal {
    require(src != address(0), "Comp::_transferTokens: cannot transfer from the zero address");
    require(dst != address(0), "Comp::_transferTokens: cannot transfer to the zero address");

    balances[src] = sub96(balances[src], amount, "Comp::_transferTokens: transfer amount exceeds balance");
    balances[dst] = add96(balances[dst], amount, "Comp::_transferTokens: transfer amount overflows");
    emit Transfer(src, dst, amount);

    _moveDelegates(delegates[src], delegates[dst], amount);
}
```

The amount is subtracted from the sender's balance and added to the receiver's balance. The important line is the last one. `_moveDelegates` is called with the sender's delegatee and the receiver's delegatee. When someone transfers tokens, voting power automatically moves from the sender's delegatee to the receiver's delegatee.

For example, if Alice has delegated to Charlie and transfers 50 COMP to Bob who has delegated to Dave, then Charlie loses 50 votes and Dave gains 50 votes. Neither Alice nor Bob's delegation preferences change, just the voting power behind them.

## `delegateBySig`

`delegateBySig` is an alternative way to delegate where someone else submits the transaction and pays the gas. The delegator signs a message off-chain, and a relayer (a bot, a frontend, any third party) submits it on-chain.

The process works like this:

1. The delegator constructs a message with three fields: `delegatee` (who to delegate to), `nonce` (prevents replay attacks), and `expiry` (when the signature expires).
2. The message is hashed using [EIP-712](https://eips.ethereum.org/EIPS/eip-712), a standard for structured data signing. The hash includes a domain separator (contract name, chain ID, contract address) so the signature cannot be reused on a different contract or chain.
3. The delegator signs the hash with their private key. The wallet produces three values: `v` (recovery byte, 27 or 28), `r` and `s` (the two halves of the ECDSA signature).
4. The delegator gives `(delegatee, nonce, expiry, v, r, s)` to the relayer.

Once the relayer has the signature, they call `delegateBySig` with the provided data.

```
function delegateBySig(address delegatee, uint nonce, uint expiry, uint8 v, bytes32 r, bytes32 s) public {
    bytes32 domainSeparator = keccak256(abi.encode(DOMAIN_TYPEHASH, keccak256(bytes(name)), getChainId(), address(this)));
    bytes32 structHash = keccak256(abi.encode(DELEGATION_TYPEHASH, delegatee, nonce, expiry));
    bytes32 digest = keccak256(abi.encodePacked("\x19\x01", domainSeparator, structHash));
```

The first line builds the EIP-712 domain separator from the contract name, chain ID, and contract address. The second line hashes the delegation data (`delegatee`, `nonce`, `expiry`). The third line combines them into a final `digest`, prefixed with `\x19\x01`. The `\x19\x01` prefix is part of the EIP-712 standard. `\x19` means "this is not a transaction" and `\x01` indicates "EIP-712 structured data." This prevents someone from crafting a delegation message that also happens to be a valid Ethereum transaction.

```
    address signatory = ecrecover(digest, v, r, s);
    require(signatory != address(0), "Comp::delegateBySig: invalid signature");
```

Using the `digest` and the `v`, `r`, `s` values provided by the delegator, `ecrecover` recovers the signer's address. `ecrecover` is a built-in Solidity function that derives the address that produced a given hash and ECDSA signature. If the signature is valid, this returns the delegator's address. If invalid, it returns `address(0)`, which the require catches.

```
    require(nonce == nonces[signatory]++, "Comp::delegateBySig: invalid nonce");
    require(block.timestamp <= expiry, "Comp::delegateBySig: signature expired");
    return _delegate(signatory, delegatee);
}
```

The nonce is checked and incremented to prevent replay attacks. The expiry is checked to make sure the signature has not expired. Finally, `_delegate` is called to update state.


## Conclusion

The COMP token is an ERC-20 with built-in governance. Holding tokens alone does nothing. Delegation activates voting power, checkpoints snapshot it at every change, and `getPriorVotes` uses binary search to look up historical balances. Token transfers automatically move voting power between delegatees.
