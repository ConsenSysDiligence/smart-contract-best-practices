The code inside a modifier is usually executed before the function body, so any state changes or
external calls will violate the
[Checks-Effects-Interactions](https://solidity.readthedocs.io/en/develop/security-considerations.html#use-the-checks-effects-interactions-pattern)
pattern. Moreover, these statements may also remain unnoticed by the developer, as the code for
modifier may be far from the function declaration. For example, an external call in modifier can
lead to the reentrancy attack:

```sol
contract Registry {
    address owner;

    function isVoter(address _addr) external returns(bool) {
        // Code
    }
}

contract Election {
    Registry registry;

    modifier isEligible(address _addr) {
        require(registry.isVoter(_addr));
        _;
    }

    function vote() isEligible(msg.sender) public {
        // Code
    }
}
```

In this case, the `Registry` contract can make a reentrancy attack by calling `Election.vote()`
inside `isVoter()`.

!!! Note
    Use [modifiers](https://solidity.readthedocs.io/en/develop/contracts.html#function-modifiers) to
    replace duplicate condition checks in multiple functions, such as `isOwner()`, otherwise use
    `require` or `revert` inside the function. This makes your smart contract code more readable and
    easier to audit.
