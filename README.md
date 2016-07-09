# Ethereum Contract Security Techniques and Tips

[The DAO](https://github.com/slockit/DAO), a crowdfunded investment contract that had substantial security flaws, highlights the importance of security and proper software engineering of blockchain-based contracts. This document outlines collected security tips and techniques for smart contract development. This material is provided as is - and may not reflect best practice. Pull requests are welcome.

**Currently, this document is an early draft - and likely has substantial omissions or errors. This message will be removed in the future once a number of community members have reviewed this document.**

#### Note for contributors

This document is designed to provide a starting security baseline for intermediate Solidity programmers. It includes security philosophies, code idioms, known attacks, and software engineering techniques for blockchain contract programming - and aims to cover all communities, techniques, and tools that improve smart contract security. At this stage, this document is focused primarily on Solidity, a javascript-like language for Ethereum, but other languages are welcome.

To contribute, see our [Contribution Guidelines](CONTRIBUTING.md).

#### Additional Requested Content

We especially welcome content in the following areas:

- Testing Solidity code (structure, frameworks, common test idioms)
- Software engineering practices for smart contracts and/or blockchain-based programming

## General Philosophy

Ethereum and complex blockchain programs are new and therefore you should expect an ongoing number of bugs, security risks, and changing best practice - even if you follow the security practices noted in this document. Further, blockchain programming requires a different engineering mindset as it is much closer to hardware programming or financial services programming with high cost to failure and limited release opportunities, unlike the rapid and forgiving iteration cycles of web or mobile development.

As a result, beyond protecting yourself against currently known hacks, it's critical to follow a different philosophy of development:

- **Practice defensive programming**; any non-trivial contract will have errors in it. This means that you likely need to build contracts where you can:
  - pause the contract ('circuit breaker')
  - manage the amount of money at risk (rate limiting, maximum usage)
  - fix and iterate on the code when errors are discovered
  - provide superuser power to a party or many parties for contract administration

- **Conduct a thoughtful and carefully staged rollout**
  - Test contracts thoroughly, adding in all newly discovered failure cases
  - Provide hacking bounties starting from alpha testnet releases
  - Rollout in phases, with increasing usage and testing in each phase

- **Keep contracts simple** - complexity increases the likelihood of errors
  - Ensure the contract logic is simple, especially at first when the code is untested - or lightly used
  - Modularize code, minimizing performance optimizations at the cost of readability; legibility increases audibility
  - Put logic that requires decentralization on the blockchain, and put other logic off; this allows you to continue rapid iteration off the blockchain

- **Follow all key security resources** (Gitter, Twitter, blogs, Reddit accounts of key members), as newly discovered issues can be quickly exploited

- **Be careful about calling external contracts** (especially, at this stage)
  - look at the dependencies of external contracts
  - remember that many levels of dependencies are particularly problematic

## Security Notifications

This is a list of resources that will often highlight discovered exploits in Ethereum or Solidity:

- [Ethereum Blog](https://blog.ethereum.org/): The official Ethereum blog
  - [Ethereum Blog - Security only](https://blog.ethereum.org/category/security/): All blog posts that are tagged *Security*
- [Ethereum Gitter](https://gitter.im/orgs/ethereum/rooms) chat rooms
  - [Solidity](https://gitter.im/ethereum/solidity)
  - [Go-Ethereum](https://gitter.im/ethereum/go-ethereum)
  - [CPP-Ethereum](https://gitter.im/ethereum/cpp-ethereum)
  - [Research](https://gitter.im/ethereum/research)
- [Reddit](https://www.reddit.com/r/ethereum)
- [Network Stats](https://ethstats.net/)

It's highly recommended that you *regularly* read all these sources, as exploits they note may impact your contracts.

Additionally, here is a list of Ethereum core developers who may write about security, and see the [bibliography](https://github.com/ConsenSys/smart-contract-best-practices#smart-contract-security-bibliography) for more from the community.

- **Vitalik Buterin**: [Twitter](https://twitter.com/vitalikbuterin), [Github](https://github.com/vbuterin), [Reddit](https://www.reddit.com/user/vbuterin), [Ethereum Blog](https://blog.ethereum.org/author/vitalik-buterin/)
- **Dr. Christian Reitwiessner**: [Twitter](https://twitter.com/ethchris), [Github](https://github.com/chriseth), [Ethereum Blog](https://blog.ethereum.org/author/christian_r/)
- **Dr. Gavin Wood**: [Twitter](https://twitter.com/gavofyork), [Blog](http://gavwood.com/), [Github](https://github.com/gavofyork)
- **Vlad Zamfir**: [Twitter](https://twitter.com/vladzamfir), [Github](https://github.com/vladzamfir), [Ethereum Blog](https://blog.ethereum.org/author/vlad/)

## Recommendations for Smart Contract Security in Solidity

<a name="avoid-external-calls"></a>

### Avoid external calls, when possible

External calls (including raw `call()`, `callcode()`, `delegatecall()`) can introduce several unexpected risks or errors. For calls to untrusted contracts, you may be executing malicious code in that contract _or_ any other contract that it depends upon. As such, it is strongly encouraged to minimize external calls. Over time, it is likely that a paradigm will develop that leads to safer external calls - but the risk currently is high.

If you must make an external call, ensure that external calls are the last call in a function - and that you've finalized your contract state before the call is made.

When possible, avoid external Contract calls (eg `ExternalContract.doSomething()`), including raw `call()`, `callcode()`, `delegatecall()`.

<a name="use-external-calls-safely"></a>

### Use external calls safely

*Raw calls* (`address.call()`, `address.callcode()`, `address.delegatecall()`) never throw an exception, but will return `false` if the call encounters an exception. On the other hand, *contract calls* (e.g., `ExternalContract.doSomething()`) will automatically propogate a throw (for example, `ExternalContract.doSomething()` will also `throw` if `doSomething()` throws).

Whether using *raw calls* or *contract calls*, assume that malicious code will execute if `ExternalContract` is untrusted.  Additionally, if you trust an `ExternalContract`, you also trust any contracts it calls.  If any malicious contract exists in the call chain, the malicious contract can attack you (see [Reentrant Attacks](https://github.com/ConsenSys/smart-contract-best-practices/blob/master/smart-contracts.md#reentrant-attacks)).

<a name="avoid-call-value"></a>

### Use `send()`, avoid `call.value()`

When sending Ether, use `someAddress.send()`, avoid `someAddress.call.value()()`.

As noted, external calls, such as `someAddress.call.value()()` are potentially dangerous because they always trigger code. `send()` also triggers code, specifically the [fallback function](https://github.com/ConsenSys/smart-contract-best-practices/blob/master/smart-contracts.md#keep-fallback-functions-simple), but `send()` is safe because it only has access to gas stipend of 2,300 gas. This is insufficient for the send recipient to trigger any state changes (the intention of the 2,300 gas stipend was to allow the recipient to log an event).

```
// bad
someAddress.call.value(100)(); // this is doubly dangerous, as it will forward all remaining gas and doesn't check for result
someAddress.call.value(100)(bytes4(sha3("deposit()"))); // if deposit throws an exception, the raw call() will only return false and transaction will NOT be reverted

// good
if(!someAddress.send(100)) {
    // Some failure code
}

ExternalContract(someAddress).deposit.value(100); // raw call() is avoided, so if deposit throws an exception, the whole transaction IS reverted
```

### Always test if `send()` and other raw calls have succeeded

`send()` and raw external calls can fail (e.g., when the call depth of 1024 is breached), so you should always test if it succeeded. If you don't test the result, it's recommended to note in a comment.

If you throw on a `send()` failure, be careful as you may create a [denial-of-service](https://github.com/ConsenSys/smart-contract-best-practices#dos-with-unexpected-throw) vulnerability.

```
// bad
someAddress.send(55);
someAddress.call.value(55)(); // this is doubly dangerous, as it will forward all remaining gas and doesn't check for result

if(!someAddress.call.value(55)(calldata)) { // checks the return value of call() but is not recommended since call() should be avoided
    // Some failure code
}

// good
if(!someAddress.send(55)) {
    // Some failure code
}
```

<a name="favor-pull-over-push-payments"></a>

### Favor *pull* payments over *push* payments

As noted, payments can fail for multiple reasons - including if the call stack exceeds 1024 or a failure on externally. To prevent payments failing, create a balance in an action that can be separately withdrawn by the external party using another action. This requires two actions on external party's part - a request to withdraw and a withdrawal - but reduces a number of potential errors.

In the future, with lower block times and having to interact across shards, this asynchronous payment might become a more important pattern for reasons other than security.

```
// bad
contract auction {
    address highestBidder;
    uint highestBid;

    function bid() {
        if (msg.value < highestBid) throw;

        if (highestBidder != 0) {
            if (!highestBidder.send(highestBid)) { // if this call consistently fails, no one else can bid
                throw;
            }
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

    function bid() external {
        if (msg.value < highestBid) throw;

        if (highestBidder != 0) {
            refunds[highestBidder] += highestBid; // record the refund that this user can claim
        }

        highestBidder = msg.sender;
        highestBid = msg.value;
    }

    function withdrawRefund() external {
        uint refund = refunds[msg.sender];
        refunds[msg.sender] = 0;
        if (!msg.sender.send(refund)) {
            refunds[msg.sender] = refund; // reverting state changes because send failed
        }
    }
}
```

<a name="keep-fallback-functions-simple"></a>

### Keep fallback functions simple

[Fallback functions](http://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) are called when a contract is sent a message with no arguments (or when no function matches), and only has access to 2,300 gas when called from a `.send()` call. The most you should do in most fallback functions is call an event. Use a proper function if a computation or more gas is required.

```
// bad
function() { balances[msg.sender] += msg.value; }

// good
function() { throw; }
function() { LogDepositReceived(msg.sender); }
function deposit() external { balances[msg.sender] += msg.value; }
```

<a name="mark-visibility"></a>

### Explicitly mark visibility in functions and state variables

Explicitly label the visibility of functions and state variables. Functions can be specified as being `external`, `public`, `internal` or `private`. For state variables, `external` is not possible.

```
// not great
uint x; // the default is private for state variables, but it should be made explicit
function transfer() { // the default is public

}

// good
uint private y;
function transfer() public {

}

function internalAction() internal {

}
```
<a name="mark-untrusted-contracts"></a>

### Mark untrusted contracts

Mark which contracts are untrusted. For example, some abstract contracts are implementable by 3rd parties. Use a prefix, or at minimum a comment, to highlight untrusted contracts.

```
// bad
Bank.withdraw(100); // Unclear whether trusted or untrusted

// good
ExternalBank.withdraw(100); // untrusted external call
TrustedBank.withdraw(100); // external but trusted bank contract maintained by XYZ Corp
```

<a name="beware-rounding-with-integer-division"></a>

### Beware rounding with integer division

All integer divison rounds down to the nearest integer - use a multiplier to keep track, or use the future fixed point data types.

```
// bad
uint x = 5 / 2; // Result is 2, all integer divison rounds DOWN to the nearest integer

// good
uint multiplier = 10;
uint x = (5 * multiplier) / 2;

// in the near future, Solidity will have fixed point data types like fixed, ufixed
```

<a name="beware-division-by-zero"></a>

### Beware division by zero

Currently, Solidity [returns zero](https://github.com/ethereum/solidity/issues/670) and does not
`throw` an exception when a number is divided by zero.

<a name="differentiate-functions-events"></a>

### Differentiate functions and events

Favor capitalization and a prefix in front of events (we suggest *Log*), to prevent the risk of confusion between functions and events. For functions, always start with a lowercase letter, except for the constructor.

```
// bad
event Transfer() {}
function transfer() {}

// good
event LogTransfer() {}
function transfer() external {}
```

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

A key benefit of Ethereum is the ability for one contract to call another - but this also can introduce risks, especially when you don't know exactly what the external call will do. Reentrant attacks, which allow a function to be called in a contract while a function in that contract is running, are one example of ths. This can occur when the same function is called again (a recursive reentrant attack), or when another function that shares state with the previously called function is called.

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


## Software Engineering Techniques

Designing your contract for unknown, often unknowable, failure scenarios is a key aspect of defensive programming, which aims to reduce the risk from newly discovered bugs. We list potential techniques you can use to mitigate various unknown failure scenarios - and many of these can be used together.

Be thoughtful about what techniques you incorporate, as certain techniques require more Solidity code, leading to a greater risk of bugs.

### Permissioned Guard (changing code once deployed)

Code will need to be changed if errors are discovered or if improvements need to be made. The simplest technique is to have a registry contract that holds the address of the latest contract. A more seamless approach for contract users is to have a contract that forwards calls and data onto the latest version of the contract.

Whatever the technique, it's important to have modularization and good separation between components (data, logic) - so that code changes do not break functionality, orphan data, or require substantial costs to port.

It's also critical to have a secure way for parties to decide to upgrade the code - and code changes may be approved by a single trusted party, a group of members, or a vote of the full set of stakeholders. You may often combine a permissioned guard with a contract pause or lock, ensuring that activity does not occur while changes are being made - or after a contract is upgraded.

**Example 1: Use a registry contract to store latest version of a contract**

In this example, the calls aren't forwarded, so users should fetch the current address each time before interacting with it.

```
contract SomeRegister {
    address backendContract;
    address[] previousBackends;
    address owner;

    function SomeRegister() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        if (msg.sender != owner) {
            throw;
        }
        _
    }

    function changeBackend(address newBackend) public
    onlyOwner()
    returns (bool)
    {
        if(newBackend != backendContract) {
            previousBackends.push(backendContract);
            backendContract = newBackend;
            return true;
        }

        return false;
    }
}
```

**Example 2: [Use a `DELEGATECALL`](http://ethereum.stackexchange.com/questions/2404/upgradeable-contracts) to forward data and calls**

```
contract Relay {
    address public currentVersion;
    address public owner;

    modifier onlyOwner() {
        if (msg.sender != owner) {
            throw;
        }
        _
    }

    function Relay(address initAddr) {
        currentVersion = initAddr;
        owner = msg.sender; // this owner may be another contract with multisig, not a single contract owner
    }

    function changeContract(address newVersion) public
    onlyOwner()
    {
        currentVersion = newVersion;
    }

    function() {
        if(!currentVersion.delegatecall(msg.data)) throw;
    }
}
```

### Circuit Breakers (Pause contract functionality)

Circuit breakers stop execution if certain conditions are met, and can be useful when new errors are discovered. For example, most actions may be suspended in a contract if a bug is discovered, and the only action now active is a withdrawal. They can come at the cost of injecting some level of trust, though smart design can minimize the trust required.

A circuit breaker can also be combined with assert guards, automatically pausing the contract if certain assertions fail (e.g., ether to token peg changes).

Example:

```
bool private stopped = false;
address private owner;

function toggleContractActive() public
isAdmin() {
    // You can add an additional modifier that restricts stopping a contract to be based on another action, such as a vote of users
    stopped = !stopped;
}

modifier isAdmin() {
    if(msg.sender != owner) {
        throw;
    }
    _
}

modifier stopInEmergency { if (!stopped) _ }
modifier onlyInEmergency { if (stopped) _ }

function deposit() public
stopInEmergency() {
    // some code
}

function withdraw() public
onlyInEmergency() {
    // some code
}
```

### Speed Bumps (Delay contract actions)

Speed bumps slow down actions, so that if malicious actions occur, there is time to recover. For example, [The DAO](https://github.com/slockit/DAO/) required 27 days between a successful request to split the DAO and the ability to do so. This ensured the funds were kept within the contract, increasing the likelihood of recovery (other fundamental flaws made this functionality useless without a fork in Ethereum). Speed bumps can be combined with other techniques (like circuit breakers or root access) for maximal effectiveness.

Example:

```
struct RequestedWithdrawal {
    uint amount;
    uint time;
}

mapping (address => uint) private balances;
mapping (address => RequestedWithdrawal) private requestedWithdrawals;
uint constant withdrawalWaitPeriod = 28 days; // 4 weeks

function requestWithdrawal() public {
    if (balances[msg.sender] > 0) {
	uint amountToWithdraw = balances[msg.sender];
	balances[msg.sender] = 0; // for simplicity, we withdraw everything;
	// presumably, the deposit function prevents new deposits when withdrawals are in progress

	requestedWithdrawals[msg.sender] = RequestedWithdrawal({
	    amount: amountToWithdraw,
	    time: now
	  });
    }
}

function withdraw() public {
    if(requestedWithdrawals[msg.sender].amount > 0 && now > requestedWithdrawals[msg.sender].time + withdrawalWaitPeriod) {
        uint amountToWithdraw = requestedWithdrawals[msg.sender].amount;
        requestedWithdrawals[msg.sender].amount = 0;

        if(!msg.sender.send(amountToWithdraw)) {
            throw;
        }
    }
}
```

### Rate Limiting

Rate limiting halts or requires approval for substantial changes. For example, a depositor may only be allowed to withdraw a certain amount or percentage of total deposits over a certain time period (e.g., max 100 ether over 1 day) - additional withdrawals in that time period may fail or require approval of an administrator. Or the rate limit could be at the contract level, with only a certain amount of tokens issued by the contract over a time period.

[Example](https://gist.github.com/PeterBorah/110c331dca7d23236f80e69c83a9d58c#file-circuitbreaker-sol)

### Assert Guards

An assert guard triggers when an assertion fails - such as an invariant property changing. For example, the token to ether issuance ratio, in a token issuance contract, may be fixed. You can verify that this is the case at all times with an assertion. Assert guards should often be combined with other techniques, such as pausing the contract and allowing upgrades.

Assert guards can also be combined with automated bug bounties that payout if the ratio changes in a test contract.

The following example reverts transactions if the ratio of ether to total number of tokens changes:

```
contract TokenWithInvariants {
    mapping(address => uint) public balanceOf;
    uint public totalSupply;

    modifier checkInvariants {
        _
        if (this.balance < totalSupply) throw;
    }

    function deposit(uint amount) public checkInvariants {
        // intentionally vulnerable
        balanceOf[msg.sender] += amount;
        totalSupply += amount;
    }

    function transfer(address to, uint value) public checkInvariants {
        if (balanceOf[msg.sender] >= value) {
            balanceOf[to] += value;
            balanceOf[msg.sender] -= value;
        }
    }

    function withdraw() public checkInvariants {
        // intentionally vulnerable
        uint balance = balanceOf[msg.sender];
        if (msg.sender.call.value(balance)()) {
            totalSupply -= balance;
            balanceOf[msg.sender] = 0;
        }
    }
}
```

### Contract Rollout

Contracts should have a substantial and prolonged testing period - before substantial money is put at risk.

At minimum, you should:

- Have a full test suite with 100% test coverage (or close to it)
- Deploy on your own testnet
- Deploy on the public testnet with substantial testing and bug bounties
- Exhaustive testing should allow various players to interact with the contract at volume
- Deploy on the mainnet in beta, with limits to the amount at risk

##### Automatic Deprecation

During testing, you can force an automatic deprecation by preventing any actions, after a certain time period. For example, an alpha contract may work for several weeks and then automatically shut down all actions, except for the final withdrawal.

```
modifier isActive() {
    if (now > SOME_BLOCK_NUMBER) {
        throw;
    }
    _
}

function deposit() public
isActive() {
    // some code
}

function withdraw() public {
    // some code
}

```
##### Restrict amount of Ether per user/contract

In the early stages, you can restrict the amount of Ether for any user (or for the entire contract) - reducing the risk.

## Security-related Documentation and Procedures
When launching a contract that will have substantial funds or is required to be mission critical, it is important to include proper documentation. Some documentation related to security includes:

**Status**

- Where current code is deployed
- Current status of deployed code (including outstanding issues, performance stats, etc.)

**Known Issues**

- Key risks with contract
  - e.g., You can lose all your money, hacker can vote for certain outcomes
- All known bugs/limitations
- Potential attacks and mitigants
- Potential conflicts of interest (e.g., will be using yourself, like Slock.it did with the DAO)

**History**

- Testing (including usage stats, discovered bugs, length of testing)
- People who have reviewed code (and their key feedback)

**Procedures**

- Notification process if bug is discovered
- Wind down process if something goes wrong (e.g., funders will get percentage of your balance before attack, from remaining funds)
- If hacker bounty*provided for discovered bugs, responsible disclosure policy, where to report, etc
- Recourse in case of failure (e.g., insurance, penalty fund, no recourse)

**Contact Information**

- Who to contact with issues
- Names of programmers and/or other important parties
- Chat room where questions can be asked

## Key Security Tools

- [Oyente](http://www.comp.nus.edu.sg/~loiluu/papers/oyente.pdf), an upcoming tool, will analyze Ethereum code to find common vulnerabilities (e.g., Transaction Order Dependence, no checking for exceptions)

## Future improvements
- **Editor Security Warnings**: Editors will soon alert for common security errors, not just compilation errors. Browser Solidity is getting these features soon.

- **New functional languages that compile to EVM bytecode**: Functional languages gives certain guarantees over procedural languages like Solidity, namely immutability within a function and strong compile time checking. This can reduce the risk of errors by providing deterministic behavior. (for more see [this](https://plus.google.com/u/0/events/cmqejp6d43n5cqkdl3iu0582f4k), Curry-Howard correspondence, and linear logic)

## Smart Contract Security Bibliography

A lot of this document contains code, examples and insights gained from various parts already written by the community. 
Here are some of them.  Feel free to add more.

##### By Ethereum core developers

- [How to Write Safe Smart Contracts](https://chriseth.github.io/notes/talks/safe_solidity) (Christian Reitwiessner)
- [Smart Contract Security](https://blog.ethereum.org/2016/06/10/smart-contract-security/) (Christian Reitwiessner)
- [Thinking about Smart Contract Security](https://blog.ethereum.org/2016/06/19/thinking-smart-contract-security/) (Vitalik Buterin)
- [Solidity](http://solidity.readthedocs.io)
- [Solidity Security Considerations](http://solidity.readthedocs.io/en/latest/security-considerations.html)

##### By Community

- http://forum.ethereum.org/discussion/1317/reentrant-contracts
- http://hackingdistributed.com/2016/06/16/scanning-live-ethereum-contracts-for-bugs/
- http://hackingdistributed.com/2016/06/18/analysis-of-the-dao-exploit/
- http://hackingdistributed.com/2016/06/22/smart-contract-escape-hatches/
- http://martin.swende.se/blog/Devcon1-and-contract-security.html
- http://publications.lib.chalmers.se/records/fulltext/234939/234939.pdf
- http://vessenes.com/deconstructing-thedao-attack-a-brief-code-tour
- http://vessenes.com/ethereum-griefing-wallets-send-w-throw-considered-harmful
- http://vessenes.com/more-ethereum-attacks-race-to-empty-is-the-real-deal
- https://blog.blockstack.org/simple-contracts-are-better-contracts-what-we-can-learn-from-the-dao-6293214bad3a
- https://blog.slock.it/deja-vu-dao-smart-contracts-audit-results-d26bc088e32e
- https://github.com/Bunjin/Rouleth/blob/master/Security.md
- https://github.com/LeastAuthority/ethereum-analyses
- https://medium.com/@ConsenSys/assert-guards-towards-automated-code-bounties-safe-smart-contract-coding-on-ethereum-8e74364b795c
- https://medium.com/@coriacetic/in-bits-we-trust-4e464b418f0b
- https://medium.com/@hrishiolickel/why-smart-contracts-fail-undiscovered-bugs-and-what-we-can-do-about-them-119aa2843007
- https://medium.com/@peterborah/we-need-fault-tolerant-smart-contracts-ec1b56596dbc
- https://pdaian.com/blog/chasing-the-dao-attackers-wake
- http://www.comp.nus.edu.sg/~loiluu/papers/oyente.pdf

## Reviewers

The following people have reviewed this document (date and commit they reviewed in parentheses):

-

## License

Licensed under [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International](https://creativecommons.org/licenses/by-nc-sa/4.0/)
