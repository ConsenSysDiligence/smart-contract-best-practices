#### Use caution when making external calls

Calls to untrusted contracts can introduce several unexpected risks or errors. External calls may
execute malicious code in that contract _or_ any other contract that it depends upon. As such,
every external call should be treated as a potential security risk. When it is not possible, or
undesirable to remove external calls, use the recommendations in the rest of this section to
minimize the danger.

______________________________________________________________________

#### Mark untrusted contracts

When interacting with external contracts, name your variables, methods, and contract interfaces in
a way that makes it clear that interacting with them is potentially unsafe. This applies to your
own functions that call external contracts.

```sol
// bad
Bank.withdraw(100); // Unclear whether trusted or untrusted

function makeWithdrawal(uint amount) { // Isn't clear that this function is potentially unsafe
    Bank.withdraw(amount);
}

// good
UntrustedBank.withdraw(100); // untrusted external call
TrustedBank.withdraw(100); // external but trusted bank contract maintained by XYZ Corp

function makeUntrustedWithdrawal(uint amount) {
    UntrustedBank.withdraw(amount);
}
```

______________________________________________________________________

#### Avoid state changes after external calls

Whether using *raw calls* (of the form `someAddress.call()`) or *contract calls* (of the form
`ExternalContract.someMethod()`), assume that malicious code might execute. Even if
`ExternalContract` is not malicious, malicious code can be executed by any contracts *it* calls.

One particular danger is malicious code may hijack the control flow, leading to vulnerabilities due
to reentrancy. (See [Reentrancy](../../attacks/reentrancy.md) for a fuller discussion of this
problem).

If you are making a call to an untrusted external contract, *avoid state changes after the call*.
This pattern is also sometimes known as the
[checks-effects-interactions pattern](http://solidity.readthedocs.io/en/develop/security-considerations.html?highlight=check%20effects#use-the-checks-effects-interactions-pattern).

See [SWC-107](https://swcregistry.io/docs/SWC-107)

______________________________________________________________________

#### Don't use `transfer()` or `send()`.

`.transfer()` and `.send()` forward exactly 2,300 gas to the recipient. The goal of this hardcoded
gas stipend was to prevent [reentrancy vulnerabilities](../../attacks/reentrancy.md), but this only
makes sense under the assumption that gas costs are constant. Recently
[EIP 1884](https://eips.ethereum.org/EIPS/eip-1884) was included in the Istanbul hard fork. One of
the changes included in EIP 1884 is an increase to the gas cost of the `SLOAD` operation, causing a
contract's fallback function to cost more than 2300 gas.

It's recommended to stop using `.transfer()` and `.send()` and instead use `.call()`.

```
// bad
contract Vulnerable {
    function withdraw(uint256 amount) external {
        // This forwards 2300 gas, which may not be enough if the recipient
        // is a contract and gas costs change.
        msg.sender.transfer(amount);
    }
}

// good
contract Fixed {
    function withdraw(uint256 amount) external {
        // This forwards all available gas. Be sure to check the return value!
        (bool success, ) = msg.sender.call.value(amount)("");
        require(success, "Transfer failed.");
    }
}
```

Note that `.call()` does nothing to mitigate reentrancy attacks, so other precautions must be
taken. To prevent reentrancy attacks, it is recommended that you use the
[checks-effects-interactions pattern](https://solidity.readthedocs.io/en/develop/security-considerations.html?highlight=check%20effects#use-the-checks-effects-interactions-pattern).

______________________________________________________________________

#### Handle errors in external calls

Solidity offers low-level call methods that work on raw addresses: `address.call()`,
`address.callcode()`, `address.delegatecall()`, and `address.send()`. These low-level methods never
throw an exception, but will return `false` if the call encounters an exception. On the other hand,
*contract calls* (e.g., `ExternalContract.doSomething()`) will automatically propagate a throw (for
example, `ExternalContract.doSomething()` will also `throw` if `doSomething()` throws).

If you choose to use the low-level call methods, make sure to handle the possibility that the call
will fail, by checking the return value.

```sol
// bad
someAddress.send(55);
someAddress.call.value(55)(""); // this is doubly dangerous, as it will forward all remaining gas and doesn't check for result
someAddress.call.value(100)(bytes4(sha3("deposit()"))); // if deposit throws an exception, the raw call() will only return false and transaction will NOT be reverted

// good
(bool success, ) = someAddress.call.value(55)("");
if(!success) {
    // handle failure code
}

ExternalContract(someAddress).deposit.value(100)();
```

See [SWC-104](https://swcregistry.io/docs/SWC-104)

______________________________________________________________________

#### Favor *pull* over *push* for external calls

External calls can fail accidentally or deliberately. To minimize the damage caused by such
failures, it is often better to isolate each external call into its own transaction that can be
initiated by the recipient of the call. This is especially relevant for payments, where it is
better to let users withdraw funds rather than push funds to them automatically. (This also reduces
the chance of [problems with the gas limit](../../attacks/denial-of-service.md).) Avoid
combining multiple ether transfers in a single transaction.

```sol
// bad
contract auction {
    address highestBidder;
    uint highestBid;

    function bid() payable {
        require(msg.value >= highestBid);

        if (highestBidder != address(0)) {
            (bool success, ) = highestBidder.call.value(highestBid)("");
            require(success); // if this call consistently fails, no one else can bid
        }

       highestBidder = msg.sender;
       highestBid = msg.value;
    }
}

// good
contract auction {
    address highestBidder;
    uint highestBid;
    mapping(address => uint) refunds;

    function bid() payable external {
        require(msg.value >= highestBid);

        if (highestBidder != address(0)) {
            refunds[highestBidder] += highestBid; // record the refund that this user can claim
        }

        highestBidder = msg.sender;
        highestBid = msg.value;
    }

    function withdrawRefund() external {
        uint refund = refunds[msg.sender];
        refunds[msg.sender] = 0;
        (bool success, ) = msg.sender.call.value(refund)("");
        require(success);
    }
}
```

See [SWC-128](https://swcregistry.io/docs/SWC-128)

______________________________________________________________________

#### Don't delegatecall to untrusted code

The `delegatecall` function is used to call functions from other contracts as if they belong to the
caller contract. Thus the callee may change the state of the calling address. This may be insecure.
An example below shows how using `delegatecall` can lead to the destruction of the contract and
loss of its balance.

```sol
contract Destructor
{
    function doWork() external
    {
        selfdestruct(0);
    }
}

contract Worker
{
    function doWork(address _internalWorker) public
    {
        // unsafe
        _internalWorker.delegatecall(bytes4(keccak256("doWork()")));
    }
}
```

If `Worker.doWork()` is called with the address of the deployed `Destructor` contract as an
argument, the `Worker` contract will self-destruct. Delegate execution only to trusted contracts,
and **never to a user supplied address**.

!!! Warning
    Don't assume contracts are created with zero balance An attacker can send ether to the
    address of a contract before it is created. Contracts should not assume that their initial state
    contains a zero balance. See
    [issue 61](https://github.com/ConsenSys/smart-contract-best-practices/issues/61) for more details.

    See [SWC-112](https://swcregistry.io/docs/SWC-112)