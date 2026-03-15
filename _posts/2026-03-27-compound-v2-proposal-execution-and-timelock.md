---
layout: post
title: 'Compound V2: Proposal Execution and Timelock'
date: 2026-03-27 10:00 -0500
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, governance, timelock, execution, cancel]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 20
---

> This article is part of the Compound V2 series. See the [series index](/posts/compound-v2-introduction/#series-index) for a full list of articles.

The previous article covered proposal creation and voting, ending with the Pending and Active states. This article picks up after voting ends and covers the remaining proposal states: Defeated, Succeeded, Queued, Executed, Expired, and Canceled.

The remaining states from the `state` function:

```
// This is the governor state() function
    } else if (proposal.forVotes <= proposal.againstVotes || proposal.forVotes < quorumVotes) {
        return ProposalState.Defeated;
    } else if (proposal.eta == 0) {
        return ProposalState.Succeeded;
    } else if (proposal.executed) {
        return ProposalState.Executed;
    } else if (block.timestamp >= add256(proposal.eta, timelock.GRACE_PERIOD())) {
        return ProposalState.Expired;
    } else {
        return ProposalState.Queued;
    }
```

These checks run after the Pending and Active checks from the previous article. If voting ended and the proposal did not pass, it is Defeated. If it passed and has not been queued yet, it is Succeeded. The remaining states involve the timelock, which is explained next.

## Defeated

A proposal is Defeated if voting ended and either the against votes are greater than or equal to the for votes, or the for votes did not reach the quorum of 400,000 COMP. This is a terminal state. Nothing else happens.

## The Timelock

The timelock needs to be understood before covering the rest of the flow. The timelock (`contracts/Timelock.sol`) is a separate contract that sits between GovernorBravo and the rest of the protocol. All governance actions pass through it.

The timelock enforces a mandatory delay between approval and execution. This gives users time to review upcoming changes and react. For example, if a proposal changes the interest rate model in a way a lender disagrees with, they have time to withdraw their funds before the change takes effect.

The timelock is a separate contract because it is the actual admin of all Compound protocol contracts, not GovernorBravo. GovernorBravo only has permission to queue, execute, and cancel actions in the timelock. This separation means the governance contract can be upgraded or replaced without migrating admin authority across every protocol contract. When Compound upgraded from GovernorAlpha to GovernorBravo, the same timelock stayed in place. Only the timelock's `admin` field was updated to point to the new governor.

Key parameters:

- `delay`: the waiting period before a queued action can be executed (configurable between 2 and 30 days)
- `GRACE_PERIOD`: 14 days after the delay passes during which the action must be executed, or it expires
- `admin`: the GovernorBravo contract

## Queuing a Proposal

After a proposal succeeds, anyone can call `queue` on GovernorBravo to send it to the timelock.

```
function queue(uint proposalId) external {
    require(state(proposalId) == ProposalState.Succeeded, ...);
    Proposal storage proposal = proposals[proposalId];
    uint eta = add256(block.timestamp, timelock.delay());
    for (uint i = 0; i < proposal.targets.length; i++) {
        queueOrRevertInternal(proposal.targets[i], proposal.values[i],
            proposal.signatures[i], proposal.calldatas[i], eta);
    }
    proposal.eta = eta;
}
```

1. Verify the proposal is in the Succeeded state.
2. Calculate the `eta` (estimated time of arrival) by adding the timelock's delay to the current timestamp. This is the earliest the proposal can be executed.
3. Loop through each action and queue it in the timelock via `queueOrRevertInternal`.
4. Store the `eta` on the proposal. This is what moves the proposal from Succeeded to Queued in the `state` function (`eta` is no longer 0).

`queueOrRevertInternal` checks that an identical action is not already queued, then forwards it to the timelock:

```
function queueOrRevertInternal(address target, uint value, string memory signature, bytes memory data, uint eta) internal {
    require(!timelock.queuedTransactions(keccak256(abi.encode(target, value, signature, data, eta))), ...);
    timelock.queueTransaction(target, value, signature, data, eta);
}
```

Inside the timelock contract, `queueTransaction` records the action:

```
// Timelock.sol
function queueTransaction(address target, uint value, string memory signature, bytes memory data, uint eta) public returns (bytes32) {
    require(msg.sender == admin, ...);
    require(eta >= getBlockTimestamp().add(delay), ...);

    bytes32 txHash = keccak256(abi.encode(target, value, signature, data, eta));
    queuedTransactions[txHash] = true;

    emit QueueTransaction(txHash, target, value, signature, data, eta);
    return txHash;
}
```

1. Verify the caller is the admin (GovernorBravo).
2. Verify the `eta` satisfies the delay requirement.
3. Hash the action parameters into a `txHash` and mark it as queued. This hash is used later during execution to verify the action is legitimate.

## Executing a Proposal

Once the timelock delay passes, anyone can call `execute` on GovernorBravo.

```
function execute(uint proposalId) external payable {
    require(state(proposalId) == ProposalState.Queued, ...);
    Proposal storage proposal = proposals[proposalId];
    proposal.executed = true;
    for (uint i = 0; i < proposal.targets.length; i++) {
        timelock.executeTransaction{value: proposal.values[i]}(
            proposal.targets[i], proposal.values[i],
            proposal.signatures[i], proposal.calldatas[i], proposal.eta);
    }
    emit ProposalExecuted(proposalId);
}
```

1. Verify the proposal is in the Queued state.
2. Set `executed` to true.
3. Loop through each action and execute it through the timelock.

Inside the timelock, `executeTransaction` performs the actual on-chain calls:

```
// Timelock.sol
function executeTransaction(address target, uint value, string memory signature, bytes memory data, uint eta) public payable returns (bytes memory) {
    require(msg.sender == admin, ...);

    bytes32 txHash = keccak256(abi.encode(target, value, signature, data, eta));
    require(queuedTransactions[txHash], ...);
    require(getBlockTimestamp() >= eta, ...);
    require(getBlockTimestamp() <= eta.add(GRACE_PERIOD), ...);

    queuedTransactions[txHash] = false;
```

4. Verify the caller is the admin (GovernorBravo).
5. Rebuild the `txHash` and verify it was queued.
6. Verify the timelock delay has passed (`timestamp >= eta`).
7. Verify the grace period has not expired (`timestamp <= eta + GRACE_PERIOD`). If nobody calls `execute` within this 14-day window after the delay, the proposal expires and can never be executed.
8. Set the queued transaction to false so it cannot be executed again.

```
    bytes memory callData;

    if (bytes(signature).length == 0) {
        callData = data;
    } else {
        callData = abi.encodePacked(bytes4(keccak256(bytes(signature))), data);
    }
```

9. Build the calldata from the `signature` and `data` parameters. Proposals store actions as a human-readable function signature (like `"_setInterestRateModel(address)"`) and ABI-encoded arguments separately. The timelock combines them here: it hashes the signature string to get the 4-byte function selector, then prepends it to the encoded arguments.

For example, if the signature is `"_setInterestRateModel(address)"` and the data is the ABI-encoded address of the new model, the resulting calldata is the 4-byte selector of `_setInterestRateModel(address)` followed by that encoded address. This is the same format as a normal function call. If the signature is empty, the raw data is used as-is.

```
    (bool success, bytes memory returnData) = target.call{value: value}(callData);
    require(success, ...);

    emit ExecuteTransaction(txHash, target, value, signature, data, eta);
    return returnData;
}
```

10. Call the target contract with the calldata and ETH value. If the call reverts, the entire execution reverts.

## Canceling a Proposal

A proposal can be canceled at any point before execution. The proposer can always cancel their own proposal. If someone else wants to cancel it, they can only do so if the proposer's voting power has dropped below the proposal threshold. This is possible because the `cancel` function checks the proposer's voting power at `block.number - 1`, not at the proposal's `startBlock`. The voting snapshot from proposal creation is only used for casting votes. The cancel check reflects current delegation state, so if the proposer transferred or undelegated their COMP after proposing, anyone can cancel.

For whitelisted proposers, only the `whitelistGuardian` can cancel, and only if the proposer's voting power has also dropped below the threshold.

```
function cancel(uint proposalId) external {
    require(state(proposalId) != ProposalState.Executed, ...);

    Proposal storage proposal = proposals[proposalId];

    if(msg.sender != proposal.proposer) {
        if(isWhitelisted(proposal.proposer)) {
            require(
                (comp.getPriorVotes(proposal.proposer, sub256(block.number, 1)) < proposalThreshold)
                    && msg.sender == whitelistGuardian, ...);
        } else {
            require(
                (comp.getPriorVotes(proposal.proposer, sub256(block.number, 1)) < proposalThreshold), ...);
        }
    }

    proposal.canceled = true;
    for (uint i = 0; i < proposal.targets.length; i++) {
        timelock.cancelTransaction(proposal.targets[i], proposal.values[i],
            proposal.signatures[i], proposal.calldatas[i], proposal.eta);
    }

    emit ProposalCanceled(proposalId);
}
```

1. The proposal must not already be executed.
2. If the caller is the proposer, skip the permission checks.
3. If the caller is not the proposer, check that the proposer's voting power has dropped below the threshold. This prevents someone from proposing with temporary voting power and having the proposal persist after they lose it.
4. For whitelisted proposers, the `whitelistGuardian` must be the caller.
5. Set `canceled` to true and remove all queued transactions from the timelock.

`cancelTransaction` in the timelock marks the transaction hash as false:

```
// Timelock.sol
function cancelTransaction(address target, uint value, string memory signature, bytes memory data, uint eta) public {
    require(msg.sender == admin, ...);

    bytes32 txHash = keccak256(abi.encode(target, value, signature, data, eta));
    queuedTransactions[txHash] = false;

    emit CancelTransaction(txHash, target, value, signature, data, eta);
}
```

## Conclusion

After voting ends, a proposal is either Defeated or moves into the timelock. The timelock enforces a delay between approval and execution, giving users time to react. Anyone can queue and execute proposals, but cancellation is restricted to the proposer or to anyone if the proposer's voting power drops below the threshold.
