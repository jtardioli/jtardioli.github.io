---
layout: post
title: 'Compound V2: Introduction'
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2]     # TAG names should always be lowercase
date: 2026-02-25 14:40 -0500
image:
  path: /assets/img/compoundv2.png
  alt: Compound V2
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

## What's Next

That's the gist of what will be covered. In the next post, we'll get into the actual mechanics, starting with `cTokens` and how interest accrual works under the hood.