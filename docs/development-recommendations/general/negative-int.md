Solidity provides several types to work with signed integers. Like in most programming languages,
in Solidity a signed integer with `N` bits can represent values from `-2^(N-1)` to `2^(N-1)-1`.
This means that there is no positive equivalent for the `MIN_INT`. Negation is implemented as
finding the two's complement of a number, so the negation of the most negative number
[will result in the same number](https://en.wikipedia.org/wiki/Two%27s_complement#Most_negative_number).

This is true for all signed integer types in Solidity (`int8`, `int16`, ..., `int256`).

```sol
contract Negation {
    function negate8(int8 _i) public pure returns(int8) {
        return -_i;
    }

    function negate16(int16 _i) public pure returns(int16) {
        return -_i;
    }

    int8 public a = negate8(-128); // -128
    int16 public b = negate16(-128); // 128
    int16 public c = negate16(-32768); // -32768
}
```

One way to handle this is to check the value of a variable before negation and throw if it's equal
to the `MIN_INT`. Another option is to make sure that the most negative number will never be
achieved by using a type with a higher capacity (e.g. `int32` instead of `int16`).

A similar issue with `int` types occurs when `MIN_INT` is multiplied or divided by `-1`.
