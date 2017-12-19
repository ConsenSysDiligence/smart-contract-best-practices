The following is a list of known attacks which you should be aware of, and defend against when writing smart contracts.

## Race Conditions<sup><a href='#footnote-race-condition-terminology'>\*</a></sup>

One of the major dangers of calling external contracts is that they can take over the control flow, and make changes to your data that the calling function wasn't expecting. This class of bug can take many forms, and both of the major bugs that led to the DAO's collapse were bugs of this sort.

### Reentrancy

The first version of this bug to be noticed involved functions that could be called repeatedly, before the first invocation of the function was finished. This may cause the different invocations of the function to interact in destructive ways.

```sol
// INSECURE
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    require(msg.sender.call.value(amountToWithdraw)()); // At this point, the caller's code is executed, and can call withdrawBalance again
    userBalances[msg.sender] = 0;
}
```

Since the user's balance is not set to 0 until the very end of the function, the second (and later) invocations will still succeed, and will withdraw the balance over and over again. A very similar bug was one of the vulnerabilities in the DAO attack.

In the example given, the best way to avoid the problem is to [use `send()` instead of `call.value()()`](https://github.com/ConsenSys/smart-contract-best-practices#send-vs-call-value). This will prevent any external code from being executed.

However, if you can't remove the external call, the next simplest way to prevent this attack is to make sure you don't call an external function until you've done all the internal work you need to do:

```sol
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    userBalances[msg.sender] = 0;
    require(msg.sender.call.value(amountToWithdraw)()); // The user's balance is already 0, so future invocations won't withdraw anything
}
```

Note that if you had another function which called `withdrawBalance()`, it would be potentially subject to the same attack, so you must treat any function which calls an untrusted contract as itself untrusted. See below for further discussion of potential solutions.

### Cross-function Race Conditions

An attacker may also be able to do a similar attack using two different functions that share the same state.

```sol
// INSECURE
mapping (address => uint) private userBalances;

function transfer(address to, uint amount) {
    if (userBalances[msg.sender] >= amount) {
       userBalances[to] += amount;
       userBalances[msg.sender] -= amount;
    }
}

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    require(msg.sender.call.value(amountToWithdraw)()); // At this point, the caller's code is executed, and can call transfer()
    userBalances[msg.sender] = 0;
}
```

In this case, the attacker calls `transfer()` when their code is executed on the external call in `withdrawBalance`. Since their balance has not yet been set to 0, they are able to transfer the tokens even though they already received the withdrawal. This vulnerability was also used in the DAO attack.

The same solutions will work, with the same caveats. Also note that in this example, both functions were part of the same contract. However, the same bug can occur across multiple contracts, if those contracts share state.

### Pitfalls in Race Condition Solutions

Since race conditions can occur across multiple functions, and even multiple contracts, any solution aimed at preventing reentry will not be sufficient.

Instead, we have recommended finishing all internal work first, and only then calling the external function. This rule, if followed carefully, will allow you to avoid race conditions. However, you need to not only avoid calling external functions too soon, but also avoid calling functions which call external functions. For example, the following is insecure:

```sol
// INSECURE
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function withdraw(address recipient) public {
    uint amountToWithdraw = userBalances[recipient];
    rewardsForA[recipient] = 0;
    require(recipient.call.value(amountToWithdraw)());
}

function getFirstWithdrawalBonus(address recipient) public {
    require(!claimedBonus[recipient]); // Each recipient should only be able to claim the bonus once

    rewardsForA[recipient] += 100;
    withdraw(recipient); // At this point, the caller will be able to execute getFirstWithdrawalBonus again.
    claimedBonus[recipient] = true;
}
```

Even though `getFirstWithdrawalBonus()` doesn't directly call an external contract, the call in `withdraw()` is enough to make it vulnerable to a race condition. You therefore need to treat `withdraw()` as if it were also untrusted.

```sol
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function untrustedWithdraw(address recipient) public {
    uint amountToWithdraw = userBalances[recipient];
    rewardsForA[recipient] = 0;
    require(recipient.call.value(amountToWithdraw)());
}

function untrustedGetFirstWithdrawalBonus(address recipient) public {
    require(!claimedBonus[recipient]); // Each recipient should only be able to claim the bonus once

    claimedBonus[recipient] = true;
    rewardsForA[recipient] += 100;
    untrustedWithdraw(recipient); // claimedBonus has been set to true, so reentry is impossible
}
```

In addition to the fix making reentry impossible, [untrusted functions have been marked.](https://github.com/ConsenSys/smart-contract-best-practices#mark-untrusted-contracts) This same pattern repeats at every level: since `untrustedGetFirstWithdrawalBonus()` calls `untrustedWithdraw()`, which calls an external contract, you must also treat `untrustedGetFirstWithdrawalBonus()` as insecure.

Another solution often suggested is a [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion). This allows you to "lock" some state so it can only be changed by the owner of the lock. A simple example might look like this:

```sol
// Note: This is a rudimentary example, and mutexes are particularly useful where there is substantial logic and/or shared state
mapping (address => uint) private balances;
bool private lockBalances;

function deposit() payable public returns (bool) {
    require(!lockBalances);
    lockBalances = true;
    balances[msg.sender] += msg.value;
    lockBalances = false;
    return true;
}

function withdraw(uint amount) payable public returns (bool) {
    require(!lockBalances && amount > 0 && balances[msg.sender] >= amount);
    lockBalances = true;

    if (msg.sender.call(amount)()) { // Normally insecure, but the mutex saves it
      balances[msg.sender] -= amount;
    }

    lockBalances = false;
    return true;
}
```

If the user tries to call `withdraw()` again before the first call finishes, the lock will prevent it from having any effect. This can be an effective pattern, but it gets tricky when you have multiple contracts that need to cooperate. The following is insecure:

```sol
// INSECURE
contract StateHolder {
    uint public n;
    address private lockHolder;

    function getLock() {
        require(lockHolder == 0);
        lockHolder = msg.sender;
    }

    function releaseLock() {
        require(msg.sender == lockHolder);
        lockHolder = 0;
    }

    function set(uint newState) {
        require(msg.sender == lockHolder);
        n = newState;
    }
}
```

An attacker can call `getLock()`, and then never call `releaseLock()`. If they do this, then the contract will be locked forever, and no further changes will be able to be made. If you use mutexes to protect against race conditions, you will need to carefully ensure that there are no ways for a lock to be claimed and never released. (There are other potential dangers when programming with mutexes, such as deadlocks and livelocks. You should consult the large amount of literature already written on mutexes, if you decide to go this route.)

<a name='footnote-race-condition-terminology'></a>

<div style='font-size: 80%; display: inline;'>* Some may object to the use of the term <i>race condition</i> since Ethereum does not currently have true parallelism. However, there is still the fundamental feature of logically distinct processes contending for resources, and the same sorts of pitfalls and potential solutions apply.</div>

## Transaction-Ordering Dependence (TOD) / Front Running

Above were examples of race conditions involving the attacker executing malicious code *within a single transaction*. The following are a different type of race condition inherent to Blockchains: the fact that *the order of transactions themselves* (within a block) is easily subject to manipulation.

Since a transaction is in the mempool for a short while, one can know what actions will occur, before it is included in a block. This can be troublesome for things like decentralized markets, where a transaction to buy some tokens can be seen, and a market order implemented before the other transaction gets included. Protecting against this is difficult, as it would come down to the specific contract itself. For example, in markets, it would be better to implement batch auctions (this also protects against high frequency trading concerns). Another way to use a pre-commit scheme (“I’m going to submit the details later”).

## Timestamp Dependence

Be aware that the timestamp of the block can be manipulated by the miner, and all direct and indirect uses of the timestamp should be considered. *Block numbers* and *average block time* can be used to estimate time, but this is not future proof as block times may change (such as the changes expected during Casper).

```sol
uint someVariable = now + 1;

if (now % 2 == 0) { // the now can be manipulated by the miner

}

if ((someVariable - 100) % 2 == 0) { // someVariable can be manipulated by the miner

}
```

## Integer Overflow and Underflow

Consider a simple token transfer:

```sol
mapping (address => uint256) public balanceOf;

// INSECURE
function transfer(address _to, uint256 _value) {
    /* Check if sender has balance */
    require(balanceOf[msg.sender] >= _value);
    /* Add and subtract new balances */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}

// SECURE
function transfer(address _to, uint256 _value) {
    /* Check if sender has balance and for overflows */
    require(balanceOf[msg.sender] >= _value && balanceOf[_to] + _value >= balanceOf[_to]);

    /* Add and subtract new balances */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}
```

If a balance reaches the maximum uint value (2^256) it will circle back to zero. This checks for that condition. This may or may not be relevant, depending on the implementation. Think about whether or not the uint value has an opportunity to approach such a large number. Think about how the uint variable changes state, and who has authority to make such changes. If any user can call functions which update the uint value, it's more vulnerable to attack. If only an admin has access to change the variable's state, you might be safe. If a user can increment by only 1 at a time, you are probably also safe because there is no feasible way to reach this limit.

The same is true for underflow. If a uint is made to be less than zero, it will cause an underflow and get set to its maximum value.

Be careful with the smaller data-types like uint8, uint16, uint24...etc: they can even more easily hit their maximum value.

Be aware there are around [20 cases for overflow and underflow](https://github.com/ethereum/solidity/issues/796#issuecomment-253578925).

<a name="dos-with-unexpected-revert"></a>

## DoS with (Unexpected) revert

Consider a simple auction contract:

```sol
// INSECURE
contract Auction {
    address currentLeader;
    uint highestBid;

    function bid() payable {
        require(msg.value > highestBid);

        require(currentLeader.send(highestBid)); // Refund the old leader, if it fails then revert

        currentLeader = msg.sender;
        highestBid = msg.value;
    }
}
```

When it tries to refund the old leader, it reverts if the refund fails. This means that a malicious bidder can become the leader while making sure that any refunds to their address will *always* fail. In this way, they can prevent anyone else from calling the `bid()` function, and stay the leader forever. A recommendation is to set up a [pull payment system](https://github.com/ConsenSys/smart-contract-best-practices/#favor-pull-over-push-payments) instead, as described earlier.

Another example is when a contract may iterate through an array to pay users (e.g., supporters in a crowdfunding contract). It's common to want to make sure that each payment succeeds. If not, one should revert. The issue is that if one call fails, you are reverting the whole payout system, meaning the loop will never complete. No one gets paid because one address is forcing an error.


```sol
address[] private refundAddresses;
mapping (address => uint) public refunds;

// bad
function refundAll() public {
    for(uint x; x < refundAddresses.length; x++) { // arbitrary length iteration based on how many addresses participated
        require(refundAddresses[x].send(refunds[refundAddresses[x]])) // doubly bad, now a single failure on send will hold up all funds
    }
}
```

Again, the recommended solution is to [favor pull over push payments](#favor-pull-over-push-payments).

## DoS with Block Gas Limit

You may have noticed another problem with the previous example: by paying out to everyone at once, you risk running into the block gas limit. Each Ethereum block can process a certain maximum amount of computation. If you try to go over that, your transaction will fail.

This can lead to problems even in the absence of an intentional attack. However, it's especially bad if an attacker can manipulate the amount of gas needed. In the case of the previous example, the attacker could add a bunch of addresses, each of which needs to get a very small refund. The gas cost of refunding each of the attacker's addresses could, therefore, end up being more than the gas limit, blocking the refund transaction from happening at all.

This is another reason to [favor pull over push payments](#favor-pull-over-push-payments).

If you absolutely must loop over an array of unknown size, then you should plan for it to potentially take multiple blocks, and therefore require multiple transactions. You will need to keep track of how far you've gone, and be able to resume from that point, as in the following example:

```sol
struct Payee {
    address addr;
    uint256 value;
}

Payee[] payees;
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

You will need to make sure that nothing bad will happen if other transactions are processed while waiting for the next iteration of the `payOut()` function. So only use this pattern if absolutely necessary.

## Forcibly Sending Ether to a Contract

It is possible to forcibly send Ether to a contract without triggering its fallback function. This is an important consideration when placing important logic in the fallback function or making calculations based on a contract's balance. Take the following example:

```sol
contract Vulnerable {
    function () payable {
        revert();
    }
    
    function somethingBad() {
        require(this.balance > 0);
        // Do something bad
    }
}
```

Contract logic seems to disallow payments to the contract and therefore disallow "something bad" from happening. However, a few methods exist for forcibly sending ether to the contract and therefore making its balance greater than zero. 

The `selfdestruct` contract method allows a user to specify a beneficiary to send any excess ether. `selfdestruct` [does not trigger a contract's fallback function](https://solidity.readthedocs.io/en/develop/security-considerations.html#sending-and-receiving-ether). 

It is also possible to [precompute](https://github.com/Arachnid/uscc/tree/master/submissions-2017/ricmoo) a contract's address and send Ether to that address before deploying the contract.

Contract developers should be aware that Ether can be forcibly sent to a contract and should design contract logic accordingly. Generally, assume that it is not possible to restrict sources of funding to your contract. 

## Deprecated/historical attacks

These are attacks which are no longer possible due to changes in the protocol or improvements to solidity. They are recorded here for posterity and awareness. 

### Call Depth Attack (deprecated)

As of the [EIP 150](https://github.com/ethereum/EIPs/issues/150) hardfork, call depth attacks are no longer relevant<sup><a href='http://ethereum.stackexchange.com/questions/9398/how-does-eip-150-change-the-call-depth-attack'>\*</a></sup> (all gas would be consumed well before reaching the 1024 call depth limit).
