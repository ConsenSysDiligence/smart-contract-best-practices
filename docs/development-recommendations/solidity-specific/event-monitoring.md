It can be useful to have a way to monitor the contract's activity after it was deployed. One way to
accomplish this is to look at all transactions of the contract, however that may be insufficient,
as message calls between contracts are not recorded in the blockchain. Moreover, it shows only the
input parameters, not the actual changes being made to the state. Also, events could be used to
trigger functions in the user interface.

```sol
contract Charity {
    mapping(address => uint) balances;

    function donate() payable public {
        balances[msg.sender] += msg.value;
    }
}

contract Game {
    function buyCoins() payable public {
        // 5% goes to charity
        charity.donate.value(msg.value / 20)();
    }
}
```

Here, `Game` contract will make an internal call to `Charity.donate()`. This transaction won't
appear in the external transaction list of `Charity`, but only visible in the internal
transactions.

An event is a convenient way to log something that happened in the contract. Events that were
emitted stay in the blockchain along with the other contract data and they are available for future
audit. Here is an improvement to the example above, using events to provide a history of the
Charity's donations.

```sol
contract Charity {
    // define event
    event LogDonate(uint _amount);

    mapping(address => uint) balances;

    function donate() payable public {
        balances[msg.sender] += msg.value;
        // emit event
        emit LogDonate(msg.value);
    }
}

contract Game {
    function buyCoins() payable public {
        // 5% goes to charity
        charity.donate.value(msg.value / 20)();
    }
}

```

Here, all transactions that go through the `Charity` contract, either directly or not, will show up
in the event list of that contract along with the amount of donated money.

______________________________________________________________________

!!! Note "Prefer newer Solidity constructs"
    Prefer constructs/aliases such as `selfdestruct` (over `suicide`) and `keccak256` (over `sha3`).  Patterns like `require(msg.sender.send(1 ether))` can also be simplified to using `transfer()`, as in `msg.sender.transfer(1 ether)`. Check out [Solidity Change log](https://github.com/ethereum/solidity/blob/develop/Changelog.md) for more similar changes.
