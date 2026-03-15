---
layout: post
title: 'Compound V2: Introduction'
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2]     # TAG names should always be lowercase
date: 2026-02-25 14:40 -0500
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 1
---

## Why Compound V2

  

Compound V2 came out in 2019, so you might wonder why I'm writing about it now. In my opinion, it is the best starting point for going deeper into DeFi mechanics. It covers a wide range of foundational concepts all in one place.

  

Compound is an extremely influential protocol. It inspired Aave and is [one of the most forked projects in DeFi history](https://defillama.com/forks), with 147 forks and $1.95B in TVL across them as of writing. If you understand Compound well, you automatically understand all those forks, and you're set to quickly understand virtually any lending protocol you encounter in the future. It's the basis for how most lending protocols are structured.

  

Here's what this series will cover:

  

- The lending mechanics: `cTokens`, interest accrual, liquidations, and more

- The governance system: Compound's on-chain governance structure, which influenced many of the governance frameworks used today

- Proxy patterns: Compound uses an early proxy pattern, and understanding it here will make every other proxy pattern you encounter easier to follow

  

---

  

## How Compound Works

  

Compound is a lending protocol where you can take out loans. Since it is decentralized, there is no central authority that can pursue you if you don't repay. For this reason, you must first deposit an asset as security, which is called collateral. Your collateral will be seized if you do not repay your loan to cover the protocol's losses. You can only borrow a fraction of your collateral's value, a ratio called the loan-to-value ratio, or LTV. The protocol always holds more than it lends out, which protects it if asset prices move against you.

  

On the other side of every loan is a lender. Lenders deposit assets into the protocol and earn yield on them, paid by borrowers in the form of interest. This is how Compound generates returns for its depositors.

  

---

  

## Series Index

- **CTokens**
  - [The Exchange Rate](/posts/compound-v2-ctokens-and-the-exhange-rate/)
  - [The Borrow Index](/posts/compound-v2-tracking-interest-with-the-borrow-index/)
  - [Fixed-Point Math](/posts/compound-v2-fixed-point-math/)
  - [Interest Accrual](/posts/compound-v2-how-interest-accrues/)
  - [Minting and Redeeming](/posts/compound-v2-minting-and-redeeming/)
  - [Borrowing and Repaying](/posts/compound-v2-borrowing-and-repaying/)
  - [Liquidation](/posts/compound-v2-liquidation/)
  - [CErc20 and CEther](/posts/compound-v2-cerc20-and-cether/)
  - [The Proxy Pattern](/posts/compound-v2-the-proxy-pattern/)
- **The Comptroller**
  - [The Comptroller](/posts/compound-v2-the-comptroller/)
  - [Market Listing and Entering](/posts/compound-v2-market-listing-and-entering/)
  - [Exiting a Market](/posts/compound-v2-exiting-a-market/)
  - [Account Liquidity](/posts/compound-v2-account-liquidity/)
  - [Comptroller Permission System](/posts/compound-v2-comptroller-permission-system/)
  - [Comptroller Liquidation](/posts/compound-v2-liquidation-in-the-comptroller/)
- **Interest Rate Models**
  - [Interest Rate Models](/posts/compound-v2-interest-rate-models/)
- **Governance**
  - [COMP Token and Delegation](/posts/compound-v2-comp-token-and-delegation/)
  - [Proposals and Voting](/posts/compound-v2-proposals-and-voting/)
  - [Proposal Execution and Timelock](/posts/compound-v2-proposal-execution-and-timelock/)
- **Rewards**
  - [COMP Reward Distribution](/posts/compound-v2-comp-reward-distribution/)
