---
layout: post
title: 'Compound V2: Borrowing and Repaying'
date: 2026-03-14 19:31 -0400
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, borrow, repay, borrowing, repaying]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 7
---

> This article is part of the Compound V2 series. See the [series index](/posts/compound-v2-introduction/#series-index) for a full list of articles.

This article walks through two core operations in Compound's `cToken` contract: borrowing and repaying. Borrowing is how a user takes underlying tokens out of the protocol against their collateral. Repaying is how a user returns those tokens and clears their debt. Both follow the same defensive structure seen in minting and redeeming.

This article assumes familiarity with the borrow index and `borrowBalanceStoredInternal`. If you haven't read the[ borrow index article](https://cursedid0l.github.io/posts/compound-v2-tracking-interest-with-the-borrow-index/) yet, start there.

---

## Borrowing

### borrowInternal

```
function borrowInternal(uint borrowAmount) internal nonReentrant {
    accrueInterest();
    // borrowFresh emits borrow-specific logs on errors
    borrowFresh(payable(msg.sender), borrowAmount);
}
```

`borrowInternal` is the entry point for borrowing. It calls `accrueInterest` to bring interest up to date for the current block, then delegates to `borrowFresh` where the actual work happens.

### Comptroller Check, Accrual Check, and Cash Check

The comptroller check, accrual check, and cash check follow the same pattern covered in the minting and redeeming article.

### Calculating the New Borrow Balance

```
uint accountBorrowsPrev = borrowBalanceStoredInternal(borrower);
uint accountBorrowsNew = accountBorrowsPrev + borrowAmount;
uint totalBorrowsNew = totalBorrows + borrowAmount;
```

`borrowBalanceStoredInternal` returns the borrower's current debt with accumulated interest factored in. If the borrower has never borrowed before, it returns zero. If they have an existing borrow, it scales their stored principal up to the current borrow index to account for interest that has accrued since their last interaction.

The new borrow balance is the previous balance plus the requested amount. The new protocol-wide total borrows is updated the same way.

### Updating State

```
accountBorrows[borrower].principal = accountBorrowsNew;
accountBorrows[borrower].interestIndex = borrowIndex;
totalBorrows = totalBorrowsNew;
```

Three values are written to storage. The borrower's `principal` is set to `accountBorrowsNew`, their total debt including any previously accumulated interest. The `interestIndex` is set to the current `borrowIndex`. `totalBorrows` is updated to reflect the new protocol-wide debt.

Setting the principal to the full current balance rather than just the new borrow amount is intentional. At any future point, `borrowBalanceStoredInternal` will calculate the borrower's debt as `principal * currentBorrowIndex / storedInterestIndex`. By recording the current index now, the next call will correctly measure only the interest that accrues from this moment forward.

For example: suppose the borrow index is `1.5` when a user borrows `100 USDC`. Their principal is stored as `100` and their interest index as `1.5`. If the borrow index grows to `1.65` by the next interaction, their balance is `100 * 1.65 / 1.5 = 110 USDC`. The `10 USDC` in interest accrued precisely over the period they were borrowing.

### Transferring Out and Emitting

```
doTransferOut(borrower, borrowAmount);

emit Borrow(borrower, borrowAmount, accountBorrowsNew, totalBorrowsNew);
```

`doTransferOut` sends the underlying to the borrower.

---

## Repaying

### repayBorrowInternal and repayBorrowBehalfInternal

```
function repayBorrowInternal(uint repayAmount) internal nonReentrant {
    accrueInterest();
    // repayBorrowFresh emits repay-borrow-specific logs on errors
    repayBorrowFresh(msg.sender, msg.sender, repayAmount);
}

function repayBorrowBehalfInternal(address borrower, uint repayAmount) internal nonReentrant {
    accrueInterest();
    // repayBorrowFresh emits repay-borrow-specific logs on errors
    repayBorrowFresh(msg.sender, borrower, repayAmount);
}
```

There are two internal entry points. `repayBorrowInternal` is for a borrower repaying their own debt. `repayBorrowBehalfInternal` allows a third party to repay on behalf of someone else, for example a liquidator or an automated keeper. Both call `accrueInterest` first and then delegate to `repayBorrowFresh`, passing `msg.sender` as the payer and the appropriate borrower address.

### Comptroller and Accrual Checks

Same pattern as the rest of the series. See the minting and redeeming article for the full rationale.

### Getting the Borrower's Current Balance

```
uint accountBorrowsPrev = borrowBalanceStoredInternal(borrower);
```

`borrowBalanceStoredInternal` returns the borrower's total outstanding debt including all interest accumulated since their last interaction. This is the number the repayment will be subtracted from.

### Handling Full Repayment

```
uint repayAmountFinal = repayAmount == type(uint).max ? accountBorrowsPrev : repayAmount;
```

Passing `type(uint).max` as the repay amount is a convenience shorthand for "repay everything." Rather than requiring the caller to query the contract for the exact amount owed, they can pass the maximum `uint` value and the function resolves it to the full outstanding balance. Any other value is treated as a partial repayment.

### Transferring In

```
uint actualRepayAmount = doTransferIn(payer, repayAmountFinal);
```

`doTransferIn` pulls the repayment from the payer. `actualRepayAmount` may differ from `repayAmountFinal` for fee-on-transfer tokens. See the minting article for the full explanation.

### Updating State

```
uint accountBorrowsNew = accountBorrowsPrev - actualRepayAmount;
uint totalBorrowsNew = totalBorrows - actualRepayAmount;

accountBorrows[borrower].principal = accountBorrowsNew;
accountBorrows[borrower].interestIndex = borrowIndex;
totalBorrows = totalBorrowsNew;
```

The actual repaid amount is subtracted from the borrower's balance and from the protocol-wide total borrows. If the borrower repaid everything, `accountBorrowsNew` is zero. The interest index is updated to the current `borrowIndex` regardless, so that if any dust balance remains, future interest calculations start from the correct reference point.

---

## Conclusion

Borrowing and repaying are inverse operations centered on `accountBorrows`, which stores each borrower's principal and the borrow index at the time of their last interaction. Every time the balance is read or written, it is scaled against the current borrow index to produce a number that includes all interest accrued over the borrower's time in the protocol.
