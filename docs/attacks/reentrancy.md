One of the major dangers of [calling external contracts](../development-recommendations/general/external-calls.md) is that
they can take over the control flow, and make changes to your data that the calling function wasn't
expecting. This class of bugs can take many forms, and both of the major bugs that led to the DAO's
collapse were bugs of this sort.

### Reentrancy on a Single Function

The first version of this bug to be noticed involved functions that could be called repeatedly,
before the first invocation of the function was finished. This may cause the different invocations
of the function to interact in destructive ways.

```sol
// INSECURE
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    (bool success, ) = msg.sender.call.value(amountToWithdraw)(""); // At this point, the caller's code is executed, and can call withdrawBalance again
    require(success);
    userBalances[msg.sender] = 0;
}
```

Since the user's balance is not set to 0 until the very end of the function, the second (and later)
invocations will still succeed and will withdraw the balance over and over again.

!!! Factoid
    A DAO is a Decentralized Autonomous Organization. Its goal is to codify the rules and
    decision-making apparatus of an organization, eliminating the need for documents and people in
    governing, creating a structure with decentralized control.

```
On June 17th 2016, [The DAO](https://www.coindesk.com/understanding-dao-hack-journalists) was hacked and 3.6 million Ether ($50 Million) were stolen using the first reentrancy attack.

Ethereum Foundation issued a critical update to rollback the hack. This resulted in Ethereum being forked into Ethereum Classic and Ethereum.
```

In the example given, the best way to prevent this attack is to make sure you don't call an
external function until you've done all the internal work you need to do:

```sol
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    userBalances[msg.sender] = 0;
    (bool success, ) = msg.sender.call.value(amountToWithdraw)(""); // The user's balance is already 0, so future invocations won't withdraw anything
    require(success);
}
```

Note that if you had another function which called `withdrawBalance()`, it would be potentially
subject to the same attack, so you must treat any function which calls an untrusted contract as
itself untrusted. See below for further discussion of potential solutions.

### Cross-function Reentrancy

An attacker may also be able to do a similar attack using two different functions that share the
same state.

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
    (bool success, ) = msg.sender.call.value(amountToWithdraw)(""); // At this point, the caller's code is executed, and can call transfer()
    require(success);
    userBalances[msg.sender] = 0;
}
```

In this case, the attacker calls `transfer()` when their code is executed on the external call in
`withdrawBalance`. Since their balance has not yet been set to 0, they are able to transfer the
tokens even though they already received the withdrawal. This vulnerability was also used in the
DAO attack.

The same solutions will work, with the same caveats. Also note that in this example, both functions
were part of the same contract. However, the same bug can occur across multiple contracts, if those
contracts share state.

### Pitfalls in Reentrancy Solutions

Since reentrancy can occur across multiple functions, and even multiple contracts, any solution
aimed at preventing reentrancy with a single function will not be sufficient.

Instead, **we have recommended finishing all internal work (ie. state changes) first, and only then
calling the external function**. This rule, if followed carefully, will allow you to avoid
vulnerabilities due to reentrancy. However, you need to not only avoid calling external functions
too soon, but also avoid calling functions which call external functions. For example, the
following is insecure:

```sol
// INSECURE
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function withdrawReward(address recipient) public {
    uint amountToWithdraw = rewardsForA[recipient];
    rewardsForA[recipient] = 0;
    (bool success, ) = recipient.call.value(amountToWithdraw)("");
    require(success);
}

function getFirstWithdrawalBonus(address recipient) public {
    require(!claimedBonus[recipient]); // Each recipient should only be able to claim the bonus once

    rewardsForA[recipient] += 100;
    withdrawReward(recipient); // At this point, the caller will be able to execute getFirstWithdrawalBonus again.
    claimedBonus[recipient] = true;
}
```

Even though `getFirstWithdrawalBonus()` doesn't directly call an external contract, the call in
`withdrawReward()` is enough to make it vulnerable to a reentrancy. You therefore need to treat
`withdrawReward()` as if it were also untrusted.

```sol
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function untrustedWithdrawReward(address recipient) public {
    uint amountToWithdraw = rewardsForA[recipient];
    rewardsForA[recipient] = 0;
    (bool success, ) = recipient.call.value(amountToWithdraw)("");
    require(success);
}

function untrustedGetFirstWithdrawalBonus(address recipient) public {
    require(!claimedBonus[recipient]); // Each recipient should only be able to claim the bonus once

    claimedBonus[recipient] = true;
    rewardsForA[recipient] += 100;
    untrustedWithdrawReward(recipient); // claimedBonus has been set to true, so reentry is impossible
}
```

In addition to the fix making reentry impossible,
[untrusted functions have been marked](../development-recommendations/general/external-calls.md). This same
pattern repeats at every level: since `untrustedGetFirstWithdrawalBonus()` calls
`untrustedWithdrawReward()`, which calls an external contract, you must also treat
`untrustedGetFirstWithdrawalBonus()` as insecure.

Another solution often suggested is a [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion). This
allows you to "lock" some state so it can only be changed by the owner of the lock. A simple
example might look like this:

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

    (bool success, ) = msg.sender.call.value(amount)("");

    if (success) { // Normally insecure, but the mutex saves it
      balances[msg.sender] -= amount;
    }

    lockBalances = false;
    return true;
}
```

If the user tries to call `withdraw()` again before the first call finishes, the lock will prevent
it from having any effect. This can be an effective pattern, but it gets tricky when you have
multiple contracts that need to cooperate. The following is insecure:

```sol
// INSECURE
contract StateHolder {
    uint private n;
    address private lockHolder;

    function getLock() {
        require(lockHolder == address(0));
        lockHolder = msg.sender;
    }

    function releaseLock() {
        require(msg.sender == lockHolder);
        lockHolder = address(0);
    }

    function set(uint newState) {
        require(msg.sender == lockHolder);
        n = newState;
    }
}
```

An attacker can call `getLock()`, and then never call `releaseLock()`. If they do this, then the
contract will be locked forever, and no further changes will be able to be made. If you use mutexes
to protect against reentrancy, you will need to carefully ensure that there are no ways for a lock
to be claimed and never released. (There are other potential dangers when programming with mutexes,
such as deadlocks and livelocks. You should consult the large amount of literature already written
on mutexes, if you decide to go this route.)

See [SWC-107](https://swcregistry.io/docs/SWC-107)

______________________________________________________________________

Above were examples of reentrancy involving the attacker executing malicious code *within a single
transaction*. The following are a different type of attack inherent to Blockchains: the fact that
*the order of transactions themselves* (e.g. within a block) is easily subject to manipulation.
