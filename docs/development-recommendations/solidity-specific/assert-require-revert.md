#### Enforce invariants with `assert()`

An assert guard triggers when an assertion fails - such as an invariant property changing. For
example, the token to ether issuance ratio, in a token issuance contract, may be fixed. You can
verify that this is the case at all times with an `assert()`. Assert guards should often be
combined with other techniques, such as pausing the contract and allowing upgrades. (Otherwise, you
may end up stuck, with an assertion that is always failing.)

Example:

```sol
contract Token {
    mapping(address => uint) public balanceOf;
    uint public totalSupply;

    function deposit() public payable {
        balanceOf[msg.sender] += msg.value;
        totalSupply += msg.value;
        assert(address(this).balance >= totalSupply);
    }
}
```

Note that the assertion is *not* a strict equality of the balance because the contract can be
[forcibly sent ether](#remember-that-ether-can-be-forcibly-sent-to-an-account) without going
through the `deposit()` function!


#### Use `assert()`, `require()`, `revert()` properly

!!! Info
    The convenience functions **assert** and **require** can be used to check for conditions and throw an exception if the condition is not met.

The **assert** function should only be used to test for internal errors, and to check invariants.

The **require** function should be used to ensure valid conditions, such as inputs, or contract state variables are met, or to validate return values from calls to external contracts. <sup><a href='https://solidity.readthedocs.io/en/latest/control-structures.html#error-handling-assert-require-revert-and-exceptions'>\*</a></sup>

Following this paradigm allows formal analysis tools to verify that the invalid opcode can never be
reached: meaning no invariants in the code are violated and that the code is formally verified.

```sol
pragma solidity ^0.5.0;

contract Sharer {
    function sendHalf(address payable addr) public payable returns (uint balance) {
        require(msg.value % 2 == 0, "Even value required."); //Require() can have an optional message string
        uint balanceBeforeTransfer = address(this).balance;
        (bool success, ) = addr.call.value(msg.value / 2)("");
        require(success);
        // Since we reverted if the transfer failed, there should be
        // no way for us to still have half of the money.
        assert(address(this).balance == balanceBeforeTransfer - msg.value / 2); // used for internal error checking
        return address(this).balance;
    }
}
```

See [SWC-110](https://swcregistry.io/docs/SWC-110) & [SWC-123](https://swcregistry.io/docs/SWC-123)
