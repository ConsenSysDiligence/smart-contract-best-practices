Avoid using `extcodesize` to check for Externally Owned Accounts.

The following modifier (or a similar check) is often used to verify whether a call was made from an
externally owned account (EOA) or a contract account:

```sol
// bad
modifier isNotContract(address _a) {
  uint size;
  assembly {
    size := extcodesize(_a)
  }
    require(size == 0);
     _;
}
```

The idea is straightforward: if an address contains code, it's not an EOA but a contract account.
However, **a contract does not have source code available during construction**. This means that
while the constructor is running, it can make calls to other contracts, but `extcodesize` for its
address returns zero. Below is a minimal example that shows how this check can be circumvented:

```sol
contract OnlyForEOA {    
    uint public flag;

    // bad
    modifier isNotContract(address _a){
        uint len;
        assembly { len := extcodesize(_a) }
        require(len == 0);
        _;
    }

    function setFlag(uint i) public isNotContract(msg.sender){
        flag = i;
    }
}

contract FakeEOA {
    constructor(address _a) public {
        OnlyForEOA c = OnlyForEOA(_a);
        c.setFlag(1);
    }
}
```

Because contract addresses can be pre-computed, this check could also fail if it checks an address
which is empty at block `n`, but which has a contract deployed to it at some block greater than
`n`.

!!! Warning "This issue is nuanced."
    If your goal is to prevent other contracts from being able to call your contract, the `extcodesize` check is probably sufficient. An alternative approach is to check the value of `(tx.origin == msg.sender)`, though this also [has drawbacks](../../development-recommendations/solidity-specific/tx-origin.md).

    There may be other situations in which the `extcodesize` check serves your purpose. Describing all of them here is out of scope. Understand the underlying behaviors of the EVM and use your Judgement.
