# Ethereum Contract Security Techniques and Tips

[The DAO](https://github.com/slockit/DAO), a crowdfunded investment contract that had substantial security flaws, highlights the importance of security and proper software engineering of blockchain-based contracts. This document outlines collected security tips and techniques for smart contract development. This material is provided as is - and may not reflect best practice. Pull requests are welcome.

**Currently, this document is an early draft - and likely has substantial omissions or errors. This message will be removed in the future once a number of community members have reviewed this document.**

#### Note for contributors

This document is designed to provide a starting security baseline for intermediate Solidity programmers. It includes security philosophies, code idioms, known attacks, and software engineering techniques for blockchain contract programming - and aims to cover all communities, techniques, and tools that improve smart contract security. At this stage, this document is focused primarily on Solidity, a javascript-like language for Ethereum, but other languages are welcome.

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
  - provide superuser power to a party or many parties

- **Conduct a thoughtful and carefully staged rollout**
  - Test contracts thoroughly, adding in all newly discovered failure cases
  - Provide hacking bounties starting from alpha testnet releases
  - Rollout in phases, with increasing usage and testing in each phase

- **Keep contracts simple** - complexity increases the likelihood of errors
  - Ensure the contract logic simple, especially at first when the code is untested
  - Modularize code, minimizing performance optimizations at the cost of readability; legibility increases audibility
  - Put logic that requires decentralization on the blockchain, and put other logic off; this allows you to continue rapid iteration off the blockchain

- **Follow all key security resources** (Gitter, Twitter, blogs, Reddit accounts of key members), as newly discovered issues can be quickly exploited

- **Be careful about calling external contracts** (especially, at this stage)
  - look at the dependencies of external contracts
  - remember that many levels of dependencies are particularly problematic

## Security Notifications

This is a list of resources that will often highlight discovered exploits in Ethereum or Solidity:

- [Ethereum Gitter](https://gitter.im/orgs/ethereum/rooms) chat rooms
  - [Solidity](https://gitter.im/ethereum/solidity)
  - [Go-Ethereum](https://gitter.im/ethereum/go-ethereum)
  - [CPP-Ethereum](https://gitter.im/ethereum/cpp-ethereum)
  - [Research](https://gitter.im/ethereum/research)
- [Ethereum Blog](https://blog.ethereum.org/): The official Ethereum blog
- [Hacking Distributed](http://hackingdistributed.com/): Professor Sirer's blog with regular posts on cryptocurrencies and security
- [Reddit](https://www.reddit.com/r/ethereum)
- [Vessenes.com](http://vessenes.com/): Peter Vessenes blog
- [Network Stats](https://ethstats.net/)

It's highly recommended that you *regularly* read all these sources, as exploits they note may impact your contracts.

Additionally, this is a list of community members who may write about security:

- **Vitalik Buterin**: [Twitter](https://twitter.com/vitalikbuterin), [Github](https://github.com/vbuterin), [Reddit](https://www.reddit.com/user/vbuterin), [Ethereum Blog](https://blog.ethereum.org/author/vitalik-buterin/)
- **Dr. Christian Reitwiessner**: [Twitter](https://twitter.com/ethchris), [Github](https://github.com/chriseth), [Ethereum Blog](https://blog.ethereum.org/author/christian_r/)
- **Dr. Gavin Wood**: [Twitter](https://twitter.com/gavofyork), [Blog](http://gavwood.com/), [Github](https://github.com/gavofyork)
- **Dr. Emin Gun Sirer**: [Twitter](https://twitter.com/el33th4xor)
- **Vlad Zamfir**: [Twitter](https://twitter.com/vladzamfir), [Github](https://github.com/vladzamfir), [Ethereum Blog](https://blog.ethereum.org/author/vlad/)

## Key Security Tools

- [Oyente](http://www.comp.nus.edu.sg/~loiluu/papers/oyente.pdf), an upcoming tool, will analyze Ethereum code to find common vulnerabilities (e.g., Transaction Order Dependence, no checking for exceptions)

## Recommendations for Smart Contract Security in Solidity

### Avoid external calls, when possible

External calls (including `.send`, which triggers the fallback function) can introduce several unexpected risks or errors. For calls to untrusted contracts, you may be executing malicious code in that contract _or_ any other contract that it depends upon. As such, it is strongly encouraged to minimize external calls. Over time, it is likely that a paradigm will develop that leads to safer external calls - but the risk currently is high.

If you must make an external call, ensure that external calls are the last call in a function - and that you've finalized your contract state before the call is made. You should also remember to check the result of all external calls (`.send()` and other raw calls will provide a boolean value, while other external function calls will throw on failure). The throw on failure of external function calls is a feature of Solidity (i.e., `Contract.doFunctionThatWillThrow()` will rethrow).

### Safely using external calls

It should be noted that external calls are potentially dangerous because they can always trigger code, even when using send(). send() can trigger the fallback function, for example. send() is usually more safe as it only has access to gas stipend of 2300 gas, which is insufficient for the send recipient to trigger any state changes (the intention of the 2300 gas stipend was to allow the recipient to register a log). Any external call is potentially vulnerable: especially call(), callcode() & delegatecall().

There is a slight difference in Solidity as to what happens with Contract calls (ie ExternalContract.doSomething()) vs raw calls (address.send() or address.call()).  A raw call never throws an exception: it returns false if the call encounters an exception. Contract calls will automatically propagate a throw (vs a raw call). For example ExternalContract.doSomething() will also throw if doSomething() throws.

The most important point is the same for both, whether using ExternalContract.doSomething() or address.call(), if ExternalContract is untrusted, assume that malicious code will execute.  Note that if you trust an ExternalContract, you also trust any external contracts it calls, and that those call, are all non-malicious.  If any malicious contract exists in the call chain, the malicious contract can attack you (next section on Reentrant Attacks).

### Always test if `.send` and other raw calls have succeeded

Sends and raw external calls can fail (e.g., when the call stack depth of 1024 is breached), so you should always test if it succeeded. If you don't test the result, it's recommended to note in a comment.

Also, if you throw on a `send` failure in an iterator then your loop may never be able to complete.

```
// bad
someAddress.send();
someAddress.call.value()(); // this is doubly dangerous, as it will forward all remaining gas and doesn't check for result

// good
if(!someAddress.send()) {
    // Some failure code
}

if(!someAddress.call.value(100000)()) { // forwards fixed amount of gas
    // Some failure code
}
```

Source: [Smart Contract Security](https://blog.ethereum.org/2016/06/10/smart-contract-security/)

### Design Patterns to avoid external calls: Push vs Pull & Asynchrony

The more ideal scenario is to try and avoid external calls where possible. This is particularly apparent when ether needs to be sent around, across contracts. It’s a potential cascade of calls.


A recommendation around this is to move to a pull vs push system. For example, let’s say a DAO needs to be paid out. The funds are sent to that address. Instead of in the same transaction, sending & splitting the funds when it reaches the DAO, the DAO simply logs that it received funds. Now, each participant who was supposed to receive must go and withdraw their funds from the DAO when they want to. So, now the call stack is reduced and attack space is reduced.

Generally, although asynchrony sometimes requires multiple transactions, it is a safer pattern in general, because you reduce the potential attack space. In the future of Ethereum, with lower block times & potentially having to interact across shards, asynchrony might become a more needed pattern besides just for security concerns.

For example, here's a withdrawRefund example in an auction contract.

```
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
	if (!msg.sender.send(refund))
	   refunds[msg.sender] = refund;
      }
}
```

### Keep fallback functions simple

Fallback functions are the default functions called when a contract is sent a message with no arguments, and only has access to 2,300 gas when called from a `.send()` call. As such, the most you should do in most fallback functions is call an event. Use a proper function if a computation or more gas is required.

```
// bad
function () { balances[msg.sender] += msg.value; }

// good
function() { throw; }
function() { LogSomeEvent(); }
function deposit() { balances[msg.sender] += msg.value; }
```

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

Source:

### Differentiate functions and events

Favor capitalization and a prefix in front of events (we suggest *Log*), to prevent the risk of confusion between functions and events. For functions, always start with a lowercase letter, except for the constructor.

```
// bad
event transferHappened() {}
function Transfer() {}

// good
event LogTransfer() {}
function transfer() {}
```

Source: [Deconstructing the DAO Attack: A Brief Code Tour](http://vessenes.com/deconstructing-thedao-attack-a-brief-code-tour/) (Peter Vessenes)

#### Explicitly mark visibility in functions and state variables

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

### Mark untrusted contracts

Mark which contracts are untrusted.  For example, some abstract contracts are implementable by 3rd parties. Use a prefix, or at minimum a comment, to highlight untrusted contracts.

```
// bad
Bank.withdraw(100); // Unclear whether trusted or untrusted

// good
ExternalBank.withdraw(100); // untrusted external call
Bank.withdraw(100); // external but trusted bank contract maintained by XYZ Corp
```

## Known Attacks

### Call depth attack

(it is sometimes also referred as the call stack attack)

Even if it is known that the likelihood of failure in a sub-execution is possible, this can be forced to happen through a call depth attack. There’s a limit to how deep the call stack can become in one transaction (limit of 1024). Thus an attacker can build up a chain of calls and then call a contract, forcing subsequent calls to fail even if enough gas is available. It has to be a call, within a call, within a call, etc.

For example, looking at the auction code from previously:

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

The send() can fail if the call depth is too large, causing ether to not be sent. However it would be marked as if it did send. As previously shown, the external call should be checked for errors. This example, the state would just revert to the previous state.

```
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
	if (!msg.sender.send(refund))
	   refunds[msg.sender] = refund;
    }
}
```

Thus:

All raw external calls should be examined and handled carefully for errors.  In most cases, the return values should be checked and handled carefully.  We recommend explicit comments in the code when such a return value is deliberately not checked.

As you can see, the call depth attack can be a malicious attack on a contract for the purpose of failing a subsequent call. Thus even if you know what code will be executed, it could still be forced to fail.

### Reentrant Attacks

Allowing contracts to call other contracts is one of the amazing benefits of Ethereum. It allows for emergent behaviour to occur across code. It is a powerful tool to easily attach additional functionality to existing smart contract ecosystems, but could open up malicious attacks. One of these are reentrant attacks: allowing a contract that is called to reenter the contract.

The more complicated issues with external calls (Contract OR raw calls) comes in when one does not know what the other contract might do when it is triggered. This can cause reentrant attacks, which can be separated into a recursive & unexpected state manipulation attack. These can be combined, which is what happened in the DAO hack.

### Recursive Reentrant Attack

This attack occurs when the state has not been properly set before the external call occurs in a function. The attacker can reenter and call the function again, being to recursively call a piece of code, whilst it was expected that they could only do it once. They can only recursively call until the gas limit has been reached or before the call depth is reached.

This is particularly problematic in the case where it is not known that something like address.call.value()() to send ether, can trigger a fallback function. However, this is not just limited to a fallback function. It can happen with any external (Contract or raw).

((code snippet))

To protect against recursive reentry, the function needs to set the state such that if the function is called again in the same transaction, it wouldn’t continue to execute.

### Unexpected State Manipulation Reentrant Attack

Unexpected state manipulation reentrant attack can occur when another function G in the contract shares the state of the current calling function F.

So, let’s imagine an attacker starts off by calling function F. It has an external call in it, which the attacker then uses to call function G. Since F & G share state, calling G, can manipulate the behaviour in F. By the time F is finished, the attacker might have manipulated state by calling G without function F being aware of it. This is one of the attacks that was done on the DAO. When splitting function was called, right at the end (after the external call), the token balances were zeroed out. However, before that was done, the attacker reentered and transfer out his balance to another address, basically then letting the split function zero out an account that already has zero funds in it. This allowed the attacker to again call the splitting function in multiple transactions, as long as he kept transferring the tokens around before the final split.

((code snippet))

To protect against this, the recommendation is again to make sure that any state changes that are to be applied needs to happen BEFORE the attacker can come in and manipulate the state.

If there are more than 2 functions that share the state, it can be seen that an attacker can potentially wreak massive havoc.  Whether an attacker is actually able to gain financially from an attack, is a separate question from an attacker rendering a contract unusable by abusing its state.

Functions that do not share any state, are safe from this attack (even if state changes happen after the external call). It is not generally a safe pattern, so it is still recommended to do state changes before doing any external calls even if no functions share states in your contract.

A non-example? ((insert here))
```
contract ReentrantSafe {
    uint fState;
    uint gState;

    f()  // only changes fState
    g() // only changes gState
}
```

In general, to protect against reentry attacks:

We recommend that external calls (of any sort) should be made at the end of function calls. If it can’t be done, extreme care needs to be taken to make sure that the functions do not share state that can be manipulated by the attacker before proceeding onwards.
In general, for both reentry attacks, the easiest fix is to make sure that you only do external calls as the last action in a function.

If one has an expectation of what type of computations will be done in an external call, to limit the gas appropriately (and not forwarding all the gas by default). This however limits the potential for any emergent useful use cases, so use wisely.

To summarise the protection against external call attacks:
- Always make sure to check the result of the call, even when using send(), or even Contract calls. Under no assumption should it be expected that everything went exactly as planned.
- Move external calls to the end of functions.
If this can’t be done, make extremely sure that state manipulation across functions won’t be possible.

<a name="iterator-abort"></a>

#### Iterator Abort

Loops that depend on external calls succeeding can fail due to a single malicious party.

```
address[] refunds; // assume this array is populated

for (uint x = 0; x < refunds.length; x++) {
  if (refundAddresses[x].send(100)) { // any single address that overrides send to return a failure can now prevent payments for the entire group
    throw;
  }
}
```

The recommended pattern is that each user should withdraw their refund themselves.

### DoS with Block Gas Limit

All Ethereum transactions must consume an amount of gas lower than the block gas limit (BGL).  An attacker can cause a denial-of-service against the contract, if the attacker can manipulate the gas used by the contract to provide the service.

Example: Manipulating the amount of elements in an array can increase gas costs substantially, forcing a DoS with the BGL. Taking the previous example, of wanting to pay out some stakeholders iteratively, it might seem fine, assuming that the amount of stakeholders won’t increase too much. The attacker would buy up, say 10000 tokens, and then split all 10000 tokens amongst 10000 addresses, causing the amount of iterations to increase, potentially exceeding the BGL.  Note that a contract cannot rely on gas refunds to protect against this DoS, because gas refunds are only provided at the end.

To mitigate around this, a pull vs push model comes in handy. For example:

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

### Timestamp Dependence Bug

In Solidity, one has access to the timestamp of the block. If it is used as an important part of the contract, the developer needs to know that it can be manipulated by the miner.

((code snippet))

A way around this could be to use block numbers and estimate time that has passed based on the average block time. However, this is NOT future proof as block times might change in the future (such as the current planned 4 second block times in Casper). So, consider this when using block numbers as as timekeeping mechanism (how long your code will be around).

### Transaction-Ordering Dependence (TOD)

Since a transaction is in the mempool for a short while, one can know what actions will occur, before it is properly recorded (included in a block). This can be troublesome for things like decentralized markets, where a transaction to buy some tokens can be seen, and a market order implemented before the other transaction gets included. Protecting against is difficult, as it would come down to the specific contract itself. For example, in markets, it would be better to implement batch auctions (this also protects against high frequency trading concerns). Another potential way to use a pre-commit scheme (“I’m going to submit the details later”).


### Leech attack

If your contract is an oracle, it may want protection from leeches that will use your contract’s data for free. If not encrypted, the data would always be readable, but one can restrict the usage of this information in other smart contracts.

Part of the solution is to carefully review the visibilities of all function and state variable.


## Software Engineering Techniques

Designing your contract for unknown, often unknowable, failure scenarios is a key aspect of defensive programming, which aims to reduce the risk from newly discovered bugs. We list potential techniques you can use to mitigate many unknown failure scenarios.

### Deployment

Before moving the mainnet, it can be prudent to first deploy the code to the testnet. Here, the code can be tested. In order to incentivize attackers, one can offer bug bounties for finding exploits in the testnet code.

When deploying on the main chain, one might want to the use the optimizer. This reduces the bytecode required and can sometimes be useful to fit large contracts into

### On-Chain

There should be the assumption that even if you used all the tools at hand, bugs could still slip in. Thus, one should be prepared for this, by using various on-chain mechanisms to either revert or replace faulty code.

### Permissioned Guard (changing code once deployed)

Code will need to be changed if errors are discovered or if improvements need to be made - and there are various techniques to do this. The simplest is to have a registry contract that holds the address of the latest contract. A more seamless approach for contract users is to have a contract that forwards calls and data onto the latest version of the contract.

Whatever the technique, it's important to have modularization and good separation between components (data, logic) - so that code changes do not break functionality, orphan data, or require substantial costs to port.

It's also critical to have a secure way for parties to upgrade the code - and code changes may be approved by a single trusted party, a group of members, or a vote of the full set of stakeholders. You may often combine a permissioned guard with a contract pause or lock, ensuring that activity does not occur while changes are being made - or after a contract is upgraded.

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

    function changeBackend(address newBackend)
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

**Example 2: Use a `DELEGATECALL` to forward data and calls**

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

    function changeContract(address newVersion)
    onlyOwner()
    {
        currentVersion = newVersion;
    }

    function() {
        if(!currentVersion.delegatecall(msg.data)) throw;
    }
}
```

Source: [Stack Overflow](http://ethereum.stackexchange.com/questions/2404/upgradeable-contracts)

### Circuit Breakers (Pause contract functionality)

Circuit breakers stop contract code from being executed if certain conditions are met, and can be useful when new errors are discovered. They sometimes come at the cost of injecting some level of trust, though smart design can minimize the trust required. Pausing can protect ether and many other items (e.g., votes). A circuit breaker can also be combined with assert guards, automatically pausing the contract if certain assertions fail (e.g., sum of balances drops below contract ether amount).

Example:

```
bool private paused = false;

function public toggleContractActive()
isAdmin() {
    // You can add an additional modifier that restricts pausing a contract to be based on another action, such as a vote of users
    paused = !paused;
}

modifier isAdmin() {
  if(msg.sender != owner) {
    throw;
  }
  _
}

modifier isActive() {
  if(paused) {
    throw;
  }
  _
}

function transfer()
isActive() {
  // some code
}
```

### Speed Bumps/Rate Limiting (Delay contract actions)

Speed bumps slow down actions, so that if malicious actions occur, there is time to recover. For example, [The DAO](https://github.com/slockit/DAO/) required 27 days between a successful request to split the DAO and the ability to do so. This ensured the funds were kept within the contract, allowing a greater likelihood of recovery (other fundamental flaws made this functionality useless without a fork in Ethereum). Speed bumps can be combined with other techniques (like circuit breakers or root access) for maximal effectiveness.

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

### Assert Guards

Akin to watching for unknown activities, an assert guard performs like a circuit breaker, but instead focuses on scenarios where an attacker can force a set of tests to fail. This however, does mean that the tests have to be written in Solidity as well, and assumes that tests are bug free as well. If an assert failure is triggers, the developers are allowed back in to upgrade the code, and only in those scenarios.

### Contract Rollout

Contracts should have a substantial and prolonged testing period - before substantial money is put at risk.

At minimum, you should:

- Have a full test suite with 100% test coverage
- Deploy on your own testnet
- Deploy on the public testnet with substantial testing and bug bounties
- Exhaustive testing should allow various players to interact with the contract at volume
- Deploy on the mainnet in beta with limits to the amount at risk

##### Automatic Deprecation

During testing, you can force an automatic deprecation by preventing any actions after a certain time period. For example, an alpha contract may work for several weeks and then automatically shut down all actions, except for the final withdrawal.

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

## Future improvements
- **Editor Security Warnings**: Editors will soon alert for common security errors, not just compilation errors. Browser Solidity is getting these features soon.

- **New functional languages that compile to EVM bytecode**: Functional languages gives certain guarantees over procedural languages like Solidity, namely immutability within a function and strong compile time checking. This can reduce the risk of errors by providing deterministic behavior. (for more see [this](https://plus.google.com/u/0/events/cmqejp6d43n5cqkdl3iu0582f4k), Curry-Howard correspondence, and linear logic)

## Noted Security Blog Posts

A lot of this document contains code, examples and insights gained from various parts already written by the community. Here are noted posts.

##### Writing Safer Contracts

- [How to Write Safe Smart Contracts](https://chriseth.github.io/notes/talks/safe_solidity): Blog post from Devcon 1 from the creator or Solidity
- [Making Smart Contracts Smarter](http://www.comp.nus.edu.sg/~loiluu/papers/oyente.pdf) (Loi Luu, Duc-Hiep Chu, Prateek Saxena, Hrishi Olickel, Aquinas Hobor)
- [We need fault-tolerant smart contracts](https://medium.com/@peterborah/we-need-fault-tolerant-smart-contracts-ec1b56596dbc) (Peter Borah)
- [We need Escape Hatches](http://hackingdistributed.com/2016/06/22/smart-contract-escape-hatches/) (Hacking Distributed)
- [Thinking about Smart Contract Security](https://blog.ethereum.org/2016/06/19/thinking-smart-contract-security/) (Vitalik Buterin)
- [Assert Guards: Towards Automated Code Bounties & Safe Smart Contract Coding on Ethereum](https://medium.com/@ConsenSys/assert-guards-towards-automated-code-bounties-safe-smart-contract-coding-on-ethereum-8e74364b795c) (Simon de la Rouviere)
- [Safer Smart Contracts through type-driven development](http://publications.lib.chalmers.se/records/fulltext/234939/234939.pdf) (Jack Pettersson and Robert Edström)
- [Simple Contracts are Better Contracts](https://blog.blockstack.org/simple-contracts-are-better-contracts-what-we-can-learn-from-the-dao-6293214bad3a)
- [In Bits We Trust?](https://medium.com/@coriacetic/in-bits-we-trust-4e464b418f0b) (David Xiao)

##### Common Contract Errors

- [Smart Contract Security](https://blog.ethereum.org/2016/06/10/smart-contract-security/) (Christian Reitwiessner)
- [More Ethereum Attacks: Race-to-empty is the Real Deal](http://vessenes.com/more-ethereum-attacks-race-to-empty-is-the-real-deal/)
- [Devcon1 and Ethereum Contract Security](http://martin.swende.se/blog/Devcon1-and-contract-security.html)
- [Potential Attacks against Rouleth contract](https://github.com/Bunjin/Rouleth/blob/master/Security.md)
- [Ethereum Griefing Wallets: Send w/Throw Is Dangerous](http://vessenes.com/ethereum-griefing-wallets-send-w-throw-considered-harmful/)

##### DAO-related Security Posts

- [Deja Vu DAO Smart Contracts Audit Results](https://blog.slock.it/deja-vu-dao-smart-contracts-audit-results-d26bc088e32e#.x9frbu72d) (Stephen Tual)
- [DAO Call for Moratorium](http://hackingdistributed.com/2016/05/27/dao-call-for-moratorium/) (Dino Mark, Vlad Zamfir, and Emin Gün Sirer)
- [Analysis of the DAO Exploit](http://hackingdistributed.com/2016/06/18/analysis-of-the-dao-exploit/) (Phil Daian/Hacking Distributed)
- [Deconstructing the DAO Attack](http://vessenes.com/deconstructing-thedao-attack-a-brief-code-tour/) (Peter Vessenes)
- [Chasing the DAO Attacker's Wake](https://pdaian.com/blog/chasing-the-dao-attackers-wake/) (Phil Daian)

##### Other

[Least Authority Security Audit](https://github.com/LeastAuthority/ethereum-analyses)

## Reviewers

The following people have reviewed this document (date and commit they reviewed in parentheses):

-

## License

Licensed under [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International](https://creativecommons.org/licenses/by-nc-sa/4.0/)
