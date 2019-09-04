The following is a list of known attacks which you should be aware of, and defend against when writing smart contracts.

## Reentrancy

One of the major dangers of [calling external contracts](./recommendations#external-calls) is that they can take over the control flow, and make changes to your data that the calling function wasn't expecting. This class of bug can take many forms, and both of the major bugs that led to the DAO's collapse were bugs of this sort.

### Reentrancy on a Single Function

The first version of this bug to be noticed involved functions that could be called repeatedly, before the first invocation of the function was finished. This may cause the different invocations of the function to interact in destructive ways.

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

Since the user's balance is not set to 0 until the very end of the function, the second (and later) invocations will still succeed, and will withdraw the balance over and over again.

!!! Factoid
    A DAO is a Decentralized Autonomous Organization. Its goal is to codify the rules and decisionmaking apparatus of an organization, eliminating the need for documents and people in governing, creating a structure with decentralized control.

    On June 17th 2016, [The DAO](https://www.coindesk.com/understanding-dao-hack-journalists) was hacked and 3.6 million Ether ($50 Million) were stolen using the first reentrancy attack.

    Ethereum Foundation issued a critical update to rollback the hack. This resulted in Ethereum being forked into Ethereum Classic and Ethereum.

In the example given, the best way to prevent this attack is to make sure you don't call an external function until you've done all the internal work you need to do:

```sol
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    userBalances[msg.sender] = 0;
    (bool success, ) = msg.sender.call.value(amountToWithdraw)(""); // The user's balance is already 0, so future invocations won't withdraw anything
    require(success);
}
```

Note that if you had another function which called `withdrawBalance()`, it would be potentially subject to the same attack, so you must treat any function which calls an untrusted contract as itself untrusted. See below for further discussion of potential solutions.

### Cross-function Reentrancy

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
    (bool success, ) = msg.sender.call.value(amountToWithdraw)(""); // At this point, the caller's code is executed, and can call transfer()
    require(success);
    userBalances[msg.sender] = 0;
}
```

In this case, the attacker calls `transfer()` when their code is executed on the external call in `withdrawBalance`. Since their balance has not yet been set to 0, they are able to transfer the tokens even though they already received the withdrawal. This vulnerability was also used in the DAO attack.

The same solutions will work, with the same caveats. Also note that in this example, both functions were part of the same contract. However, the same bug can occur across multiple contracts, if those contracts share state.

### Pitfalls in Reentrancy Solutions

Since reentrancy can occur across multiple functions, and even multiple contracts, any solution aimed at preventing reentrancy with a single function will not be sufficient.

Instead, **we have recommended finishing all internal work (ie. state changes) first, and only then calling the external function**. This rule, if followed carefully, will allow you to avoid vulnerabilities due to reentrancy. However, you need to not only avoid calling external functions too soon, but also avoid calling functions which call external functions. For example, the following is insecure:

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

Even though `getFirstWithdrawalBonus()` doesn't directly call an external contract, the call in `withdrawReward()` is enough to make it vulnerable to a reentrancy. You therefore need to treat `withdrawReward()` as if it were also untrusted.

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

In addition to the fix making reentry impossible, [untrusted functions have been marked](./recommendations#mark-untrusted-contracts). This same pattern repeats at every level: since `untrustedGetFirstWithdrawalBonus()` calls `untrustedWithdrawReward()`, which calls an external contract, you must also treat `untrustedGetFirstWithdrawalBonus()` as insecure.

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

    (bool success, ) = msg.sender.call(amount)("");

    if (success) { // Normally insecure, but the mutex saves it
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

An attacker can call `getLock()`, and then never call `releaseLock()`. If they do this, then the contract will be locked forever, and no further changes will be able to be made. If you use mutexes to protect against reentrancy, you will need to carefully ensure that there are no ways for a lock to be claimed and never released. (There are other potential dangers when programming with mutexes, such as deadlocks and livelocks. You should consult the large amount of literature already written on mutexes, if you decide to go this route.)

----------------


##  Front-Running (AKA Transaction-Ordering Dependence)

Above were examples of reentrancy involving the attacker executing malicious code *within a single transaction*. The following are a different type of attack inherent to Blockchains: the fact that *the order of transactions themselves* (e.g. within a block) is easily subject to manipulation.

Since a transaction is in the mempool for a short while, one can know what actions will occur before it is included in a block. This can be troublesome for things like decentralized markets, where a transaction to buy some tokens can be seen, and a market order implemented before the other transaction gets included. Protecting against this is difficult, as it would come down to the specific contract itself. For example, in markets, it would be better to implement batch auctions (this also protects against high frequency trading concerns). Another way to use a pre-commit scheme (“I’m going to submit the details later”).

---------------

## Timestamp Dependence
Be aware that the timestamp of the block can be manipulated by the miner, and all direct and indirect uses of the timestamp should be considered.

!!! Note
    See the [Recommendations](./recommendations/#timestamp-dependence) section for design considerations related to Timestamp Dependence.

----------------

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

If a balance reaches the *maximum uint value (2^256)* it will circle back to zero which checks for the condition. This may or may not be relevant, depending on the implementation. Think about whether or not the `uint` value has an opportunity to approach such a large number. Think about how the `uint` variable changes state, and who has authority to make such changes. If any user can call functions which update the `uint` value, it's more vulnerable to attack. If only an admin has access to change the variable's state, you might be safe. If a user can increment by only 1 at a time, you are probably also safe because there is no feasible way to reach this limit.

The same is true for underflow. If a uint is made to be less than zero, it will cause an underflow and get set to its maximum value.

Be careful with the smaller data-types like uint8, uint16, uint24...etc: they can even more easily hit their maximum value.

!!! Warning
    Be aware there are around [20 cases for overflow and underflow](https://github.com/ethereum/solidity/issues/796#issuecomment-253578925).

One simple solution to mitigate the common mistakes for overflow and underflow is to use `SafeMath.sol` [library](https://github.com/OpenZeppelin/openzeppelin-solidity/blob/master/contracts/math/SafeMath.sol) for arithmetic functions.


### Underflow in Depth: Storage Manipulation

 [Doug Hoyte's submission](https://github.com/Arachnid/uscc/tree/master/submissions-2017/doughoyte) to the 2017 underhanded solidity contest received [an honorable mention](http://u.solidity.cc/). The entry is interesting, because it raises the concerns about how C-like underflow might affect Solidity storage. Here is a simplified version:

```sol
contract UnderflowManipulation {
    address public owner;
    uint256 public manipulateMe = 10;
    function UnderflowManipulation() {
        owner = msg.sender;
    }

    uint[] public bonusCodes;

    function pushBonusCode(uint code) {
        bonusCodes.push(code);
    }

    function popBonusCode()  {
        require(bonusCodes.length >=0);  // this is a tautology
        bonusCodes.length--; // an underflow can be caused here
    }

    function modifyBonusCode(uint index, uint update)  {
        require(index < bonusCodes.length);
        bonusCodes[index] = update; // write to any index less than bonusCodes.length
    }

}
```

In general, the variable `manipulateMe`'s location cannot be influenced without going through the `keccak256`, which is infeasible. However, since dynamic arrays are stored sequentially, if a malicious actor wanted to change `manipulateMe` all they would need to do is:

 * Call `popBonusCode` to underflow (Note: `array.pop()` method [was added](https://github.com/ethereum/solidity/blob/v0.5.0/Changelog.md) in Solidity 0.5.0)
 * Compute the storage location of `manipulateMe`
 * Modify and update `manipulateMe`'s value using `modifyBonusCode`

!!! Note
    In practice, this array would be immediately pointed out as fishy, but buried under more complex smart contract architecture, it can arbitrarily allow malicious changes to constant variables, such as total supply of a token.

When considering use of a dynamic array, a container data scructure is a good practice. The Solidity CRUD [part 1](https://medium.com/@robhitchens/solidity-crud-part-1-824ffa69509a) and [part 2](https://medium.com/@robhitchens/solidity-crud-part-2-ed8d8b4f74ec) articles are good resources.


----------------

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

If attacker bids using a smart contract which has a fallback function that reverts any payment, the attacker can win any auction. When it tries to refund the old leader, it reverts if the refund fails. This means that a malicious bidder can become the leader while making sure that any refunds to their address will *always* fail. In this way, they can prevent anyone else from calling the `bid()` function, and stay the leader forever. A recommendation is to set up a [pull payment system](./recommendations#favor-pull-over-push-for-external-calls) instead, as described earlier.

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

Again, the recommended solution is to [favor pull over push payments](./recommendations#favor-pull-over-push-for-external-calls).

-------------

## DoS with Block Gas Limit

Each block has an upper bound on the amount of gas that can be spent, and thus the amount computation that can be done. This is the Block Gas Limit. If the gas spent exceeds this limit, the transaction will fail. This leads to a couple possible Denial of Service vectors:

### Gas Limit DoS on a Contract via Unbounded Operations

You may have noticed another problem with the previous example: by paying out to everyone at once, you risk running into the block gas limit.

This can lead to problems even in the absence of an intentional attack. However, it's especially bad if an attacker can manipulate the amount of gas needed. In the case of the previous example, the attacker could add a bunch of addresses, each of which needs to get a very small refund. The gas cost of refunding each of the attacker's addresses could, therefore, end up being more than the gas limit, blocking the refund transaction from happening at all.

This is another reason to [favor pull over push payments](./recommendations#favor-pull-over-push-for-external-calls).

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

### Gas Limit DoS on the Network via Block Stuffing

Even if your contract does not contain an unbounded loop, an attacker can prevent other transactions from being included in the blockchain for several blocks by placing computationally intensive transactions with a high enough gas price.

To do this, the attacker can issue several transactions which will consume the entire gas limit, with a high enough gas price to be included as soon as the next block is mined. No gas price can guarantee inclusion in the block, but the higher the price is, the higher is the chance.

If the attack succeeds, no other transactions will be included in the block. Sometimes, an attacker's goal is to block transactions to a specific contract prior to specific time.

This attack [was conducted](https://osolmaz.com/2018/10/18/anatomy-block-stuffing) on Fomo3D, a gambling app. The app was designed to reward the last address that purchased a "key". Each key purchase extended the timer, and the game ended once the timer went to 0. The attacker bought a key and then stuffed 13 blocks in a row until the timer was triggered and the payout was released. Transactions sent by attacker took 7.9 million gas on each block, so the gas limit allowed a few small "send" transactions (which take 21,000 gas each), but disallowed any calls to the `buyKey()` function (which costs 300,000+ gas).

A Block Stuffing attack can be used on any contract requiring an action within a certain time period. However, as with any attack, it is only profitable when the expected reward exceeds its cost. Cost of this attack is directly proportional to the number of blocks which need to be stuffed. If a large payout can be obtained by preventing actions from other participants, your contract will likely be targeted by such an attack.

----------------

## Insufficient gas griefing

This attack may be possible on a contract which accepts generic data and uses it to make a call another contract (a 'sub-call') via the low level `address.call()` function, as is often the case with multisignature and transaction relayer contracts.

If the call fails, the contract has two options:

1. revert the whole transaction
2. continue execution.

Take the following example of a simplified `Relayer` contract which continues execution regardless of the outcome of the subcall:

```sol
contract Relayer {
    mapping (bytes => bool) executed;

    function relay(bytes _data) public {
        // replay protection; do not call the same transaction twice
        require(executed[_data] == 0, "Duplicate call");
        executed[_data] = true;
        innerContract.call(bytes4(keccak256("execute(bytes)")), _data);
    }
}
```

This contract allows transaction relaying. Someone who wants to make a transaction but can't execute it by himself (e.g. due to the lack of ether to pay for gas) can sign data that he wants to pass and transfer the data with his signature over any medium. A third party "forwarder" can then submit this transaction to the network on behalf of the user.

If given just the right amount of gas, the `Relayer` would complete execution recording the `_data`argument in the `executed` mapping, but the subcall would fail because it received insufficient gas to complete execution.

!!! Note
    When a contract makes a sub-call to another contract, the EVM limits the gas forwarded to [to 63/64 of the remaining gas](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-150.md),

An attacker can use this to censor transactions, causing them to fail by sending them with a low amount of gas. This attack is a form of "[griefing](https://en.wikipedia.org/wiki/Griefer)": It doesn't directly benefit the attacker, but causes grief for the victim. A dedicated attacker, willing to consistently spend a small amount of gas could theoretically censor all transactions this way, if they were the first to submit them to `Relayer`.

One way to address this is to implement logic requiring forwarders to provide enough gas to finish the subcall. If the miner tried to conduct the attack in this scenario, the `require` statement would fail and the inner call would revert. A user can specify a minimum gasLimit along with the other data (in this example, typically the `_gasLimit` value would be verified by a signature, but that is ommitted for simplicity in this case).

```sol
// contract called by Relayer
contract Executor {
    function execute(bytes _data, uint _gasLimit) {
        require(gasleft() >= _gasLimit);
        ...
    }
}
```

Another solution is to permit only trusted accounts to relay the transaction.

-------------

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

!!! Warning
    It is also possible to [precompute](https://github.com/Arachnid/uscc/tree/master/submissions-2017/ricmoo) a contract's address and send Ether to that address before deploying the contract.

-------------

## Deprecated/historical attacks

These are attacks which are no longer possible due to changes in the protocol or improvements to solidity. They are recorded here for posterity and awareness.

### Call Depth Attack (deprecated)

As of the [EIP 150](https://github.com/ethereum/EIPs/issues/150) hardfork, call depth attacks are no longer relevant<sup><a href='http://ethereum.stackexchange.com/questions/9398/how-does-eip-150-change-the-call-depth-attack'>\*</a></sup> (all gas would be consumed well before reaching the 1024 call depth limit).

### Constantinople Reentrancy Attack

On January 16th, 2019, Constantinople protocol upgrade was delayed due to a security vulnerability enabled by [EIP 1283](https://eips.ethereum.org/EIPS/eip-1283). _EIP 1283: Net gas metering for SSTORE without dirty maps_ proposes changes to reduce excessive gas costs on dirty storage writes.

This change led to possibility of a new reentrancy vector making previously known secure withdrawal patterns (`.send()` and `.transfer()`) unsafe in specific situations<sup><a href='https://medium.com/chainsecurity/constantinople-enables-new-reentrancy-attack-ace4088297d9'>\*</a></sup>, where the attacker could hijack the control flow and use the remaining gas enabled by EIP 1283, leading to vulnerabilities due to reentrancy.


## Other Vulnerabilities

The [Smart Contract Weakness Classification Registry](https://smartcontractsecurity.github.io/SWC-registry/) offers a complete and up-to-date catalogue of known smart contract vulnerabilities and anti-patterns along with real-world examples. Browsing the registry is a good way of keeping up-to-date with the latest attacks.
