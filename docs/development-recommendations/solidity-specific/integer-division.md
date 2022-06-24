All integer division rounds down to the nearest integer. If you need more precision, consider using
a multiplier, or store both the numerator and denominator.

(In the future, Solidity will have a
[fixed-point](https://solidity.readthedocs.io/en/develop/types.html#fixed-point-numbers) type,
which will make this easier.)

```sol
// bad
uint x = 5 / 2; // Result is 2, all integer division rounds DOWN to the nearest integer
```

Using a multiplier prevents rounding down, this multiplier needs to be accounted for when working
with x in the future:

```sol
// good
uint multiplier = 10;
uint x = (5 * multiplier) / 2;
```

Storing the numerator and denominator means you can calculate the result of `numerator/denominator`
off-chain:

```sol
// good
uint numerator = 5;
uint denominator = 2;
```
