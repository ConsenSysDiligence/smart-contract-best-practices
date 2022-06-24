Never use `tx.origin` for authorization, another contract can have a method which will call your
contract (where the user has some funds for instance) and your contract will authorize that
transaction as your address is in `tx.origin`.

```sol
contract MyContract {

    address owner;

    function MyContract() public {
        owner = msg.sender;
    }

    function sendTo(address receiver, uint amount) public {
        require(tx.origin == owner);
        (bool success, ) = receiver.call.value(amount)("");
        require(success);
    }

}

contract AttackingContract {

    MyContract myContract;
    address attacker;

    function AttackingContract(address myContractAddress) public {
        myContract = MyContract(myContractAddress);
        attacker = msg.sender;
    }

    function() public {
        myContract.sendTo(attacker, msg.sender.balance);
    }

}
```

You should use `msg.sender` for authorization (if another contract calls your contract `msg.sender`
will be the address of the contract and not the address of the user who called the contract).

You can read more about it here:
[Solidity docs](https://solidity.readthedocs.io/en/develop/security-considerations.html#tx-origin)

!!! Warning
    Besides the issue with authorization, there is a chance that `tx.origin` will be
    removed from the Ethereum protocol in the future, so code that uses `tx.origin` won't be compatible
    with future releases
    [Vitalik: 'Do NOT assume that tx.origin will continue to be usable or meaningful.'](https://ethereum.stackexchange.com/questions/196/how-do-i-make-my-dapp-serenity-proof/200#200)

    It's also worth mentioning that by using `tx.origin` you're limiting interoperability between
    contracts because the contract that uses tx.origin cannot be used by another contract as a contract
    can't be the `tx.origin`.

    See [SWC-115](https://swcregistry.io/docs/SWC-115)
