It is currently possible to [shadow](https://en.wikipedia.org/wiki/Variable_shadowing) built-in
globals in Solidity. This allows contracts to override the functionality of built-ins such as `msg`
and `revert()`. Although this [is intended](https://github.com/ethereum/solidity/issues/1249), it
can mislead users of a contract as to the contract's true behavior.

```sol
contract PretendingToRevert {
    function revert() internal constant {}
}

contract ExampleContract is PretendingToRevert {
    function somethingBad() public {
        revert();
    }
}
```

Contract users (and auditors) should be aware of the full smart contract source code of any
application they intend to use.
