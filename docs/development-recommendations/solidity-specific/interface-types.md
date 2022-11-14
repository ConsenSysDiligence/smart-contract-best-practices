When a function takes a contract address as an argument, it is better to pass an interface or
contract type rather than a raw `address`. If the function is called elsewhere within the source
code, the compiler will provide additional type safety guarantees.

Here we see two alternatives:

```sol
contract Validator {
    function validate(uint) external returns(bool);
}

contract TypeSafeAuction {
    // good
    function validateBet(Validator _validator, uint _value) internal returns(bool) {
        bool valid = _validator.validate(_value);
        return valid;
    }
}

contract TypeUnsafeAuction {
    // bad
    function validateBet(address _addr, uint _value) internal returns(bool) {
        Validator validator = Validator(_addr);
        bool valid = validator.validate(_value);
        return valid;
    }
}
```

The benefits of using the `TypeSafeAuction` contract above can then be seen from the following
example. If `validateBet()` is called with an `address` argument, or a contract type other than
`Validator`, the compiler will throw this error:

```sol
contract NonValidator{}

contract Auction is TypeSafeAuction {
    NonValidator nonValidator;

    function bet(uint _value) {
        bool valid = validateBet(nonValidator, _value); // TypeError: Invalid type for argument in function call.
                                                        // Invalid implicit conversion from contract NonValidator
                                                        // to contract Validator requested.
    }
}
```
