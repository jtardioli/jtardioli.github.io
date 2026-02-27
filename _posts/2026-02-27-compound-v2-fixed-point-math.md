---
layout: post
title: 'Compound V2: Fixed-Point Math'
date: 2026-02-27 12:17 -0500
categories: [Protocol Breakdowns, Compound V2]
tags: [compound, compound v2, fixed-point math]     # TAG names should always be lowercase
image:
  path: /assets/img/protocols/compoundv2.png
  alt: Compound V2
series: "Compound V2"
series_index: 4
---

In this article we explain the fixed-point math behind Compound. Understanding this is a crucial stepping stone to understanding the rest of the calculations in the protocol. All the code referenced here is from `ExponentialNoError.sol`.

---

## Why Scales Are Needed

Solidity cannot handle decimals. This is surprising for a language used for decentralized finance. Here is what happens to decimal values in Solidity:

```
// Normal math
1 / 2 = 0.5

// Solidity
1 / 2 = 0
```

Solidity truncates everything after the decimal point. A few more examples:

```
// All examples in Solidity

7  / 3  = 2
88 / 14 = 6
10 / 6  = 1
```

Obviously, having `10 / 6` equal `1` will not work for financial applications. The standard workaround in DeFi is to scale numbers up before doing division.

The idea is simple: pick a scaling factor and multiply your value by it before dividing. Here is an example using `100` as the scaling factor:

```
// We want to compute 1 / 2

// Step 1: scale up
1 * 100 = 100

// Step 2: divide
100 / 2 = 50

// Step 3: to recover the real value off-chain, divide by the scale
50 / 100 = 0.5
```


The general term for this strategy of scaling integers to avoid decimal truncation is called fixed-point arithmetic. On-chain we only ever work with scaled values, so division never produces a truncated zero. Off-chain we divide by the scaling factor to get the true value.

---

## Scales in Compound

Compound defines two scaling factors:

```
uint constant expScale    = 1e18;
uint constant doubleScale = 1e36;
```

Instead of multiplying by `100` like in the example above, Compound multiplies by either `1e18` or `1e36`. A quick example:

```
// Representing 1 at expScale
1 * 1e18 = 1e18

// Dividing
1e18 / 2 = 5e17

// Recovering the real value off-chain
5e17 / 1e18 = 0.5
```


---

## The Exp and Double Structs

```
struct Exp {
    uint mantissa;
}

struct Double {
    uint mantissa;
}
```

These structs act as labels more than anything else. They let you always know how a value is currently scaled, which is more convenient than leaving comments everywhere.

- `Exp` means the value is scaled by `1e18`
- `Double` means the value is scaled by `1e36`

`mantissa` is just the name Compound uses for a scaled value. Whenever you see the word `mantissa` in Compound, read it as "scaled value". This is not the canonical mathematical definition of mantissa. Compound just uses it as a convenient label.

---

## Truncate

```
function truncate(Exp memory exp) pure internal returns (uint) {
    return exp.mantissa / expScale;
}
```

`truncate` takes a mantissa scaled at `1e18` and divides by `expScale` to get back to the original value. Because this is Solidity, the decimal gets truncated:

```
11e17 / 1e18 = 1    // 1.1 gets truncated to 1
```

---

## Addition and Subtraction

```
function add_(uint a, uint b) pure internal returns (uint) {
    return a + b;
}

function add_(Exp memory a, Exp memory b) pure internal returns (Exp memory) {
    return Exp({mantissa: add_(a.mantissa, b.mantissa)});
}

function add_(Double memory a, Double memory b) pure internal returns (Double memory) {
    return Double({mantissa: add_(a.mantissa, b.mantissa)});
}
```

```
function sub_(uint a, uint b) pure internal returns (uint) {
    return a - b;
}

function sub_(Exp memory a, Exp memory b) pure internal returns (Exp memory) {
    return Exp({mantissa: sub_(a.mantissa, b.mantissa)});
}

function sub_(Double memory a, Double memory b) pure internal returns (Double memory) {
    return Double({mantissa: sub_(a.mantissa, b.mantissa)});
}
```

Addition and subtraction are straightforward. The base functions just add or subtract two values. The overloaded versions for `Exp` and `Double` do the same thing — they just attach the appropriate type label to the result. The scaling factor does not change when adding or subtracting.

---

## Multiplication

Multiplication requires extra care because multiplying two scaled values together breaks the scale:

```
1e18 * 1e18 = 1e36
```

If we now divide by `1e18` to recover the original value we get `1e18`, not `1`. The scale is broken. The fix is to divide by the scaling factor once after multiplying two scaled values:

```
// Rule: scaledA * scaledB / scale = scaledProduct

1e18 * 1e18 / 1e18 = 1e18    // represents 1 * 1 = 1
3e18 * 5e18 / 1e18 = 15e18   // represents 3 * 5 = 15
```

Dividing these results by the scale now returns the correct original values:

```
 1e18 / 1e18 =  1
15e18 / 1e18 = 15
```

This adjustment only applies when multiplying two scaled values together. If you multiply a scaled value by a plain constant, no correction is needed:

```
1e18 * 5 = 5e18
5e18 / 1e18 = 5    // still correct
```

---

## Division

Division has the opposite problem. Dividing two scaled values collapses the scale, so you need to multiply by the scaling factor once after dividing to restore it:

```
// Without correction the scale is lost
15e18 / 3e18 = 5

// With correction the scale is preserved
15e18 / 3e18 * 1e18 = 5e18
```

Due to order of operations you can also write this as:

```
15e18 * 1e18 / 3e18 = 5e18
```

Both are equivalent, but in Compound you will typically see the second form. This is because multiplying before dividing avoids precision loss. If you divide first and the result truncates to zero, multiplying afterward recovers nothing.

```
scaledValue1 * scalingFactor / scaledValue2
```


As with multiplication, if you are dividing by a plain constant rather than another scaled value, no correction is needed:

```
15e18 / 3 = 5e18    // scale is preserved automatically
```

---

## Fraction

The `fraction` function is similar to the division functions with one difference: it takes two plain (unscaled) integers as inputs and scales them to `doubleScale` before dividing.

```
function fraction(uint a, uint b) pure internal returns (Double memory) {
    return Double({mantissa: div_(mul_(a, doubleScale), b)});
}
```


The function multiplies `a` by `doubleScale` (`1e36`) before dividing by `b`. Here is a concrete example:

```
// We want to compute 1 / 3

// Without scaling
1 / 3 = 0    // truncates to zero

// With doubleScale
1 * 1e36 / 3 = 333333333333333333333333333333333333 

// 0.333... at 1e36 precision
```

The reason `doubleScale` is used instead of `expScale` is precision. When computing a ratio between two small integers, `1e18` may not give enough decimal places to represent the result accurately. Scaling to `1e36` first gives 36 digits of precision, which is enough for most sensitive calculations.

---

## Safe Type Conversion Guards

```
function safe224(uint n, string memory errorMessage) pure internal returns (uint224) {
    require(n < 2**224, errorMessage);
    return uint224(n);
}

function safe32(uint n, string memory errorMessage) pure internal returns (uint32) {
    require(n < 2**32, errorMessage);
    return uint32(n);
}
```

These are simple guards for downcasting integer types. A `uint32` can hold values up to `2^32 - 1`, a `uint224` can hold values up to `2^224 - 1`. These functions check that the value fits before casting, preventing silent overflow.

---

## Conclusion

With this foundation you should be able to understand every function in `ExponentialNoError.sol`. The two rules worth keeping in mind as you move forward: when multiplying two scaled values, divide by the scale once after; when dividing two scaled values, multiply by the scale once before. Everything else in the file is just addition and subtraction, which need no correction at all. Fixed-point arithmetic is used across virtually every DeFi protocol, so understanding it well will pay off well beyond Compound.