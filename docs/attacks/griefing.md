This attack may be possible on a contract which accepts generic data and uses it to make a call
another contract (a 'sub-call') via the low level `address.call()` function, as is often the case
with multisignature and transaction relayer contracts.

If the call fails, the contract has two options:

1. revert the whole transaction
1. continue execution.

Take the following example of a simplified `Relayer` contract which continues execution regardless
of the outcome of the subcall:

```sol
contract Relayer {
    mapping (bytes => bool) executed;

    function relay(bytes _data) public {
        // replay protection; do not call the same transaction twice
        require(executed[_data] == 0, "Duplicate call");
        executed[_data] = true;
        innerContract.call(bytes4(keccak256("execute(bytes)")), _data);
    }
}
```

This contract allows transaction relaying. Someone who wants to make a transaction but can't
execute it by himself (e.g. due to the lack of ether to pay for gas) can sign data that he wants to
pass and transfer the data with his signature over any medium. A third party "forwarder" can then
submit this transaction to the network on behalf of the user.

If given just the right amount of gas, the `Relayer` would complete execution recording the
`_data`argument in the `executed` mapping, but the subcall would fail because it received
insufficient gas to complete execution.

!!! Note
    When a contract makes a sub-call to another contract, the EVM limits the gas forwarded to
    [to 63/64 of the remaining gas](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-150.md),

An attacker can use this to censor transactions, causing them to fail by sending them with a low
amount of gas. This attack is a form of "[griefing](https://en.wikipedia.org/wiki/Griefer)": It
doesn't directly benefit the attacker, but causes grief for the victim. A dedicated attacker,
willing to consistently spend a small amount of gas could theoretically censor all transactions
this way, if they were the first to submit them to `Relayer`.

One way to address this is to implement logic requiring forwarders to provide enough gas to finish
the subcall. If the miner tried to conduct the attack in this scenario, the `require` statement
would fail and the inner call would revert. A user can specify a minimum gasLimit along with the
other data (in this example, typically the `_gasLimit` value would be verified by a signature, but
that is omitted for simplicity in this case).

```sol
// contract called by Relayer
contract Executor {
    function execute(bytes _data, uint _gasLimit) {
        require(gasleft() >= _gasLimit);
        ...
    }
}
```

Another solution is to permit only trusted accounts to relay the transaction.
