
## Known Attacks

<a name="call-depth-attack"></a>

### Call Depth Attack

Even if it is known that the likelihood of failure in a sub-execution is possible, this can be forced to happen through a Call Depth Attack. There’s a limit to how deep the message-call/contract-creation stack can become in one transaction (limit of 1024). Thus, an attacker can build up a chain of calls and then call a contract, forcing subsequent calls to fail even if enough gas is available. It has to be a call, within a call, within a call, etc.  This is sometimes called the Call stack attack, but as the EVM is stack-based and operates on a stack (that is different from the message-call/contract-creation stack), ambiguity is avoided by simply calling this a Call Depth Attack.

Example auction code from above:

```
// DO NOT USE. THIS IS VULNERABLE.
contract auction {
    address highestBidder;
    uint highestBid;
    mapping(address => uint) refunds;

    function bid() {
	if (msg.value < highestBid) throw;
	if (highestBidder != 0)
	    refunds[highestBidder] += highestBid;
	highestBidder = msg.sender;
	highestBid = msg.value;
    }

    function withdrawRefund() {
	uint refund = refunds[msg.sender];
	refunds[msg.sender] = 0;
	msg.sender.send(refund); // vulnerable line.
	refunds[msg.sender] = refund;
    }
}
```

The send() can fail if the call depth is too large, causing ether to not be sent. However it would be marked as if it did send. As previously shown, the external call should be checked for errors. This example, the state would just revert to the previous state:

```
contract auction {
    address highestBidder;
    uint highestBid;
    mapping(address => uint) refunds;

    function bid() external {
        if (msg.value < highestBid) throw;
        if (highestBidder != 0) {
	    refunds[highestBidder] += highestBid;
	}

        highestBidder = msg.sender;
        highestBid = msg.value;
    }

    function withdrawRefund() external {
	uint refund = refunds[msg.sender];
	refunds[msg.sender] = 0;
	if (!msg.sender.send(refund)) {
	   refunds[msg.sender] = refund;
	}
    }
}
```

Thus:

All raw external calls should be examined and handled carefully for errors.  In most cases, the return values should be checked and handled carefully.  We recommend explicit comments in the code when such a return value is deliberately not checked.

As you can see, the call depth attack can be a malicious attack on a contract for the purpose of failing a subsequent call. Thus even if you know what code will be executed, it could still be forced to fail.

<a name="reentrant-attacks"></a>

### Reentrant Attacks

A key benefit of Ethereum is the ability for one contract to call another - but this also can introduce risks, especially when you don't know exactly what the external call will do. Reentrant attacks, which allow a function to be called in a contract while a function in that contract is running, are one example of this. This can occur when the same function is called again (a recursive reentrant attack), or when another function that shares state with the previously called function is called.

(The DAO hack combined both these attacks)

```
// VULNERABLE.  externalContract can call g() and affect the sharedState.
contract C {
    uint sharedState = 2;
    function f(address externalContract) internal {
       sharedState = 44;
       externalContract.call.value(3)();
       // normally it would be assumed that sharedState would be 44 here.
       // But this guarantee cannot hold since a reentrant attack calling g() will change sharedState
    }
    
    function g() external {
        sharedState = 55;
    }
}
```


#### Recursive Reentrant Attack

This attack occurs when the state has not been properly set before the external call in a function. The attacker can reenter and call the function again, even though the programmer never intended for this to happen. For example, you can withdraw money and recursively call the withdraw again, as the balance may only be updated at the end of the `withdraw` function. The attacker can recursively call until the gas limit has been reached or before the call depth is reached.

Example:

```
// DO NOT USE. THIS IS VULNERABLE.
mapping (address => uint) private userBalances;

function getBalance(address user) constant returns(uint) {
    return userBalances[user];
}

function addToBalance() public {
    userBalances[msg.sender] += msg.value;
}

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    if (!(msg.sender.call.value(amountToWithdraw)())) { throw; } // the ether is sent without zeroing out msg.sender's balance, so msg.sender can repeatedly call withdrawBalance().
    userBalances[msg.sender] = 0;
}
```

In the DAO hack, this occurred when the fallback function was called by `address.call.value()()`, but can occur for any external call.

To protect against recursive reentry, the function needs to set the state such that if the function is called again in the same transaction, it won’t continue to execute.

#### Unexpected State Manipulation Reentrant Attack

This attack occurs when a function expects a certain state, but another contract function alters this state, while the original function is still running. A malicious party starts the first function, then calls the second function before the first function completes.

In [The DAO](https://github.com/slockit/DAO), the splitting function zeroed out token balances. However, before that was complete, the attacker reentered and transfered out his/her balance to another address, meaning the split function zeroed out an account that already had zero funds. The attacker could then call the splitting function in multiple transactions, as long as s/he kept transferring the tokens around before the final split.

```
TODO: Add snippet
```

If external functions (and the functions they call) share the same state, an attacker can cause substantial damage. Functions that do not share any state, are safe from this attack (even if state changes happen after the external call). This is not generally a safe pattern, so it is still recommended to make state changes before any external calls, even if no functions share state in your contract.

To mitigate this attack, you should:

- Always check the result of a contract call, even when using `send()` (don't ever assume things went well)
- Only make external calls at the end of functions, once state has been set

You can alternatively ensure that functions do not modify the same state variables:

```
contract StateManipulationReentrantSafe {
    uint fState;
    uint gState;

    f()  // only changes fState
    g() // only changes gState
}
```

Also, you can use a [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion) (often used in concurrent programming) where you lock certain variables:

```
// Note: This is a rudimentary example, and mutexes are particularly useful where there is substantial logic and/or shared state
mapping (address => uint) private balances;
bool private lockBalances;

function deposit() public returns (bool) {
    if (!lockBalances) {
	lockBalances = true;
	balances[msg.sender] += msg.value;
	lockBalances = false;
        return true;
    }
    throw;
}

function withdraw(uint amount) public returns (bool) {
    if (!lockBalances && amount > 0 && balances[msg.sender] >= amount) {
        lockBalances = true;
        balances[msg.sender] -= amount;

        if (!msg.sender.send(amount)) {
            throw;
        }
        lockBalances = false;
        return true;
    }

    throw;
}
```

Mutexes have their own disadvantages with the potential for deadlocks and reduced throughput - so choose the approach that works best for your use case and test extensively.


<a name="dos-with-unexpected-throw"></a>

### DoS with (Unexpected) Throw

One example of this is where the routine throw on a failed `send()` can cause a denial-of-service.

In this auction, an attacker can [reject payments](https://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) to themselves and will always be the highest bidder. Any resource that is owned by the highest bidder, will permanently be owned by the attacker.

It should be the responsibility of the recipient to accept payment.

```
TODO: Add code snippet
```

Another example is when a contract may iterate through an array to pay users (e.g., supporters in a crowdfunding contract). It's common to want to make sure that each payment succeeds. If not, one should throw. The issue is that if one call fails, you are reverting the whole payout system, meaning the loop will never complete. No one gets paid, because one address is forcing an error.

```
address[] private refundAddresses;
mapping (address => uint) public refunds;

// bad
function refundAll() public {
    for(uint x; x < refundAddresses.length; x++) { // arbitrary length iteration based on how many addresses participated
        if(refundAddresses[x].send(refunds[refundAddresses[x]])) {
            throw; // doubly bad, now a single failure on send will hold up all funds
        }
    }
}
```

The recommended solution is to [favor pull over push payments](#favor-pull-over-push-payments).

<a name="dos-with-block-gas-limit"></a>

### DoS with Block Gas Limit

All Ethereum transactions must consume an amount of gas lower than the block gas limit (BGL).  An attacker can cause a contract denial-of-service, if the attacker can manipulate the gas used by the contract to provide the service.

For example, manipulating the amount of elements in an array can increase gas costs substantially, forcing a DoS with the BGL. Taking the previous example, of wanting to pay out some stakeholders iteratively, it might seem fine, assuming that the amount of stakeholders won’t increase too much. The attacker would buy up, say 10000 tokens, and then split all 10000 tokens amongst 10000 addresses, causing the amount of iterations to increase, potentially exceeding the BGL.  Note that a contract cannot rely on gas refunds to protect against this DoS, because gas refunds are only provided at the end.

To mitigate this, use a pull rather than a push model. For example:

((code snippet))

An alternative approach is to have a payout loop that can be split across multiple transactions, like so:

```
struct Payee {
    address addr;
    uint256 value;
}
Payee payees[];
uint256 nextPayeeIndex;

function payOut() {
    uint256 i = nextPayeeIndex;
    while (i < payees.length && msg.gas > 200000) {
	payees[i].addr.send(payees[i].value);
	i++;
    }
    nextPayeeIndex = i;
}
```
<a name="timestamp-dependence"></a>

### Timestamp Dependence

The timestamp of the block can be manipulated by the miner, and so should not be used for critical components of the contract. *Block numbers* and *average block time* can be used to estimate time, but this is not future proof as block times may change (such as the changes expected during Casper).

```
uint startTime = SOME_START_TIME;

if (now > startTime + 1 week) { // the now can be manipulated by the miner

}
```

<a name="transaction-ordering-dependence"></a>

### Transaction-Ordering Dependence (TOD)

Since a transaction is in the mempool for a short while, one can know what actions will occur, before it is included in a block. This can be troublesome for things like decentralized markets, where a transaction to buy some tokens can be seen, and a market order implemented before the other transaction gets included. Protecting against is difficult, as it would come down to the specific contract itself. For example, in markets, it would be better to implement batch auctions (this also protects against high frequency trading concerns). Another way to use a pre-commit scheme (“I’m going to submit the details later”).

