---
layout: post
title: 'Compound V2: Proposals and Voting'
date: 2026-03-26 10:00 -0500
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, governance, governor bravo, proposals]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 19
---

> This article is part of the Compound V2 series. See the [series index](/posts/compound-v2-introduction/#series-index) for a full list of articles.

The previous article covered the COMP token, delegation, and checkpoints. This article covers how proposals are created and voted on. All of this is handled by GovernorBravo, found in `contracts/Governance/GovernorBravoDelegate.sol`.

Compound originally used GovernorAlpha. GovernorBravo replaced it with three additions: abstain votes, updatable parameters, and the proxy pattern. The contract is called `GovernorBravoDelegate` because it sits behind a `GovernorBravoDelegator` proxy. This follows a similar proxy structure to the other Compound contracts covered earlier in the series. The contract inherits from `GovernorBravoDelegateStorageV2`, which holds all the state variables.

For an interactive overview of how DAO governance works, try [this governance game](https://dao-game.vercel.app/) I made before reading further.

A proposal goes through several states in its lifecycle:

```
enum ProposalState {
    Pending,
    Active,
    Canceled,
    Defeated,
    Succeeded,
    Queued,
    Expired,
    Executed
}
```

This article covers proposal creation and voting (Pending and Active). The next article will cover what happens after voting ends.

## Creating a Proposal

A proposal is a set of on-chain actions that will execute if it passes. To create one, a user calls `propose` with five parameters: `targets`, `values`, `signatures`, `calldatas`, and a `description`.

The first four are arrays that describe the actions. Each index across the four arrays represents one action. For example, suppose a proposal wants to set a new interest rate model on the USDC market:

- `targets`: the address of the USDC cToken contract
- `values`: 0 (no ETH sent with the call)
- `signatures`: `"_setInterestRateModel(address)"`
- `calldatas`: the ABI-encoded address of the new interest rate model contract

A single proposal can contain multiple actions (up to 10).

```
function propose(
    address[] memory targets,
    uint[] memory values,
    string[] memory signatures,
    bytes[] memory calldatas,
    string memory description
) public returns (uint) {
    require(initialProposalId != 0, "GovernorBravo::propose: Governor Bravo not active");
    require(
        comp.getPriorVotes(msg.sender, sub256(block.number, 1)) > proposalThreshold
            || isWhitelisted(msg.sender),
        "GovernorBravo::propose: proposer votes below proposal threshold"
    );
    require(targets.length == values.length
        && targets.length == signatures.length
        && targets.length == calldatas.length,
        "GovernorBravo::propose: proposal function information arity mismatch"
    );
    require(targets.length != 0, "GovernorBravo::propose: must provide actions");
    require(targets.length <= proposalMaxOperations, "GovernorBravo::propose: too many actions");
```

1. Check that the contract is initialized.
2. Enforce the proposal threshold: the proposer's voting power at the previous block must exceed `proposalThreshold`, or the proposer must be whitelisted. The threshold is governance-configurable (between 1,000 and 100,000 COMP). It prevents spam proposals, though a threshold set too high could limit who can propose. The whitelist gives governance a way to let specific addresses propose regardless of their token holdings.
3. Validate the action arrays. All four arrays must have the same length, there must be at least one action, and there cannot be more than `proposalMaxOperations` (10).

```
    uint latestProposalId = latestProposalIds[msg.sender];
    if (latestProposalId != 0) {
        ProposalState proposersLatestProposalState = state(latestProposalId);
        require(proposersLatestProposalState != ProposalState.Active,
            "GovernorBravo::propose: one live proposal per proposer, found an already active proposal"
        );
        require(proposersLatestProposalState != ProposalState.Pending,
            "GovernorBravo::propose: one live proposal per proposer, found an already pending proposal"
        );
    }
```

4. If the user has proposed before, check that their most recent proposal is not still Active or Pending. Each address can only have one live proposal at a time.

```
    uint startBlock = add256(block.number, votingDelay);
    uint endBlock = add256(startBlock, votingPeriod);
```

5. Calculate the voting window using block numbers. `startBlock` is the current block plus a voting delay (configurable between 1 block and ~1 week). `endBlock` is the start plus the voting period (configurable between ~24 hours and ~2 weeks). For example, if the current block is 1000, the voting delay is 100 blocks, and the voting period is 500 blocks, then voting runs from block 1100 to block 1600.

```
    proposalCount++;
    uint newProposalID = proposalCount;
    Proposal storage newProposal = proposals[newProposalID];

    require(newProposal.id == 0, "GovernorBravo::propose: ProposalID collsion");
```

6. Increment the proposal counter and create a new `Proposal` struct in storage. The collision check is a sanity guard that should never trigger under normal operation.

```
    newProposal.id = newProposalID;
    newProposal.proposer = msg.sender;
    newProposal.eta = 0;
    newProposal.targets = targets;
    newProposal.values = values;
    newProposal.signatures = signatures;
    newProposal.calldatas = calldatas;
    newProposal.startBlock = startBlock;
    newProposal.endBlock = endBlock;
    newProposal.forVotes = 0;
    newProposal.againstVotes = 0;
    newProposal.abstainVotes = 0;
    newProposal.canceled = false;
    newProposal.executed = false;

    latestProposalIds[newProposal.proposer] = newProposal.id;

    emit ProposalCreated(newProposal.id, msg.sender, targets, values, signatures, calldatas, startBlock, endBlock, description);
    return newProposal.id;
}
```

7. Initialize all proposal fields and save to storage. The `eta` field (estimated time of arrival) starts at 0 and gets set later when the proposal is queued in the timelock (Timelock is discussed in detail later). Update `latestProposalIds` so the contract can enforce the one-live-proposal-per-address rule on future calls.

## Voting

Once the voting delay passes and the proposal enters the Active state, token holders can vote. There are three ways to vote:

- `castVote(proposalId, support)` - vote directly
- `castVoteWithReason(proposalId, support, reason)` - vote with an on-chain reason string
- `castVoteBySig(proposalId, support, v, r, s)` - vote via EIP-712 signature so someone else can submit the transaction and pay the gas, the same pattern as `delegateBySig` from the previous article

All three call `castVoteInternal`:

```
function castVoteInternal(address voter, uint proposalId, uint8 support) internal returns (uint96) {
    require(state(proposalId) == ProposalState.Active, "GovernorBravo::castVoteInternal: voting is closed");
    require(support <= 2, "GovernorBravo::castVoteInternal: invalid vote type");
    Proposal storage proposal = proposals[proposalId];
    Receipt storage receipt = proposal.receipts[voter];
    require(receipt.hasVoted == false, "GovernorBravo::castVoteInternal: voter already voted");
    uint96 votes = comp.getPriorVotes(voter, proposal.startBlock);
```

1. The proposal must be Active.
2. The `support` value must be 0 (against), 1 (for), or 2 (abstain).
3. Fetch the proposal and the voter's receipt from storage. If the voter has already voted, the transaction reverts.
4. Look up the voter's voting power at the proposal's `startBlock` using `getPriorVotes` from the COMP token. This is the checkpoint mechanism from the previous article in action: voting power is locked when the proposal becomes active, so token transfers after that point do not affect the vote.

```
    if (support == 0) {
        proposal.againstVotes = add256(proposal.againstVotes, votes);
    } else if (support == 1) {
        proposal.forVotes = add256(proposal.forVotes, votes);
    } else if (support == 2) {
        proposal.abstainVotes = add256(proposal.abstainVotes, votes);
    }

    receipt.hasVoted = true;
    receipt.support = support;
    receipt.votes = votes;

    return votes;
}
```

5. Add the voter's power to the appropriate tally.
6. Update the receipt to record the vote, preventing the same address from voting twice.

## The `state` Function

The `state` function determines which lifecycle state a proposal is currently in. The full function handles all eight states. This article focuses on Pending and Active. The remaining states will be covered in the next article.

```
function state(uint proposalId) public view returns (ProposalState) {
    require(proposalCount >= proposalId && proposalId > initialProposalId,
        "GovernorBravo::state: invalid proposal id"
    );
    Proposal storage proposal = proposals[proposalId];

   ...
    } else if (block.number <= proposal.startBlock) {
        return ProposalState.Pending;
    } else if (block.number <= proposal.endBlock) {
        return ProposalState.Active;
    } else if (...) {
        // remaining states covered in the next article
    }
}
```

## Conclusion

GovernorBravo handles proposal creation and voting. The proposal threshold and one-proposal-per-address rule prevent spam. Voting power is locked at the proposal's start block using the COMP token's checkpoint system, so token transfers during voting have no effect.
