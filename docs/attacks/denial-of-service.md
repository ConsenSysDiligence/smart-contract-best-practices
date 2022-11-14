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

If attacker bids using a smart contract which has a fallback function that reverts any payment, the
attacker can win any auction. When it tries to refund the old leader, it reverts if the refund
fails. This means that a malicious bidder can become the leader while making sure that any refunds
to their address will *always* fail. In this way, they can prevent anyone else from calling the
`bid()` function, and stay the leader forever. A recommendation is to set up a
[pull payment system](../development-recommendations/general/external-calls.md) instead, as
described earlier.

Another example is when a contract may iterate through an array to pay users (e.g., supporters in a
crowdfunding contract). It's common to want to make sure that each payment succeeds. If not, one
should revert. The issue is that if one call fails, you are reverting the whole payout system,
meaning the loop will never complete. No one gets paid because one address is forcing an error.

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

Again, the recommended solution is to
[favor pull over push payments](../development-recommendations/general/external-calls.md).

See [SWC-113](https://swcregistry.io/docs/SWC-113)


## DoS with Block Gas Limit

Each block has an upper bound on the amount of gas that can be spent, and thus the amount
computation that can be done. This is the Block Gas Limit. If the gas spent exceeds this limit, the
transaction will fail. This leads to a couple of possible Denial of Service vectors:

### Gas Limit DoS on a Contract via Unbounded Operations

You may have noticed another problem with the previous example: by paying out to everyone at once,
you risk running into the block gas limit.

This can lead to problems even in the absence of an intentional attack. However, it's especially
bad if an attacker can manipulate the amount of gas needed. In the case of the previous example,
the attacker could add a bunch of addresses, each of which needs to get a very small refund. The
gas cost of refunding each of the attacker's addresses could, therefore, end up being more than the
gas limit, blocking the refund transaction from happening at all.

This is another reason to
[favor pull over push payments](../development-recommendations/general/external-calls.md).

If you absolutely must loop over an array of unknown size, then you should plan for it to
potentially take multiple blocks, and therefore require multiple transactions. You will need to
keep track of how far you've gone, and be able to resume from that point, as in the following
example:

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

You will need to make sure that nothing bad will happen if other transactions are processed while
waiting for the next iteration of the `payOut()` function. So only use this pattern if absolutely
necessary.

### Gas Limit DoS on the Network via Block Stuffing

Even if your contract does not contain an unbounded loop, an attacker can prevent other
transactions from being included in the blockchain for several blocks by placing computationally
intensive transactions with a high enough gas price.

To do this, the attacker can issue several transactions which will consume the entire gas limit,
with a high enough gas price to be included as soon as the next block is mined. No gas price can
guarantee inclusion in the block, but the higher the price is, the higher is the chance.

If the attack succeeds, no other transactions will be included in the block. Sometimes, an
attacker's goal is to block transactions to a specific contract prior to specific time.

This attack [was conducted](https://solmaz.io/2018/10/18/anatomy-block-stuffing/) on Fomo3D, a
gambling app. The app was designed to reward the last address that purchased a "key". Each key
purchase extended the timer, and the game ended once the timer went to 0. The attacker bought a key
and then stuffed 13 blocks in a row until the timer was triggered and the payout was released.
Transactions sent by attacker took 7.9 million gas on each block, so the gas limit allowed a few
small "send" transactions (which take 21,000 gas each), but disallowed any calls to the `buyKey()`
function (which costs 300,000+ gas).

A Block Stuffing attack can be used on any contract requiring an action within a certain time
period. However, as with any attack, it is only profitable when the expected reward exceeds its
cost. The cost of this attack is directly proportional to the number of blocks which need to be
stuffed. If a large payout can be obtained by preventing actions from other participants, your
contract will likely be targeted by such an attack.

See [SWC-128](https://swcregistry.io/docs/SWC-128)

