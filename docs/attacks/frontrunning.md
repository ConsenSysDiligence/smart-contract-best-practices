Since all transactions are visible in the mempool for a short while before being executed,
observers of the network can see and react to an action before it is included in a block. An
example of how this can be exploited is with a decentralized exchange where a buy order transaction
can be seen, and second order can be broadcast and executed before the first transaction is
included. Protecting against this is difficult, as it would come down to the specific contract
itself.

Front-running, coined originally for traditional financial markets, is the race to order the chaos
to the winner's benefit. In financial markets, the flow of information gave birth to intermediaries
that could simply profit by being the first to know and react to some information. These attacks
mostly had been within stock market deals and early domain registries, such as whois gateways.

!!! cite "front-run·ning (/ˌfrəntˈrəniNG/)"
    *noun*: front-running;

    1. *STOCK MARKET*

        > the practice by market makers of dealing on advance information provided by their brokers and investment analysts, before their clients have been given the information.
        <!-- [[OXFORD](https://www.lexico.com/en/definition/front-running)] -->

### Taxonomy

By defining a [taxonomy](https://arxiv.org/abs/1902.05164) and differentiating each group from
another, we can make it easier to discuss the problem and find solutions for each group.

We define the following categories of front-running attacks:

1. Displacement
1. Insertion
1. Suppression

#### Displacement

In the first type of attack, *a displacement attack*, it is **not important** for Alice’s (User)
function call to run after Mallory (Adversary) runs her function. Alice’s can be orphaned or run
with no meaningful effect. Examples of displacement include:

- Alice trying to register a domain name and Mallory registering it first;
- Alice trying to submit a bug to receive a bounty and Mallory stealing it and submitting it first;
- Alice trying to submit a bid in an auction and Mallory copying it.

This attack is commonly performed by increasing the `gasPrice` higher than network average, often
by a multiplier of 10 or more.

#### Insertion

For this type of attack, it is **important** to the adversary that the original function call runs
after her transaction. In an insertion attack, after Mallory runs her function, the state of the
contract is changed and she needs Alice’s original function to run on this modified state. For
example, if Alice places a purchase order on a blockchain asset at a higher price than the best
offer, Mallory will insert two transactions: she will purchase at the best offer price and then
offer the same asset for sale at Alice’s slightly higher purchase price. If Alice’s transaction is
then run after, Mallory will profit on the price difference without having to hold the asset.

As with displacement attacks, this is usually done by outbidding Alice's transaction in the gas
price auction.

!!! info "Transaction Order Dependence"
    Transaction Order Dependence is equivalent to race
    condition in smart contracts. An example, if one function sets the reward percentage, and the
    withdraw function uses that percentage; then withdraw transaction can be front-run by a change
    reward function call, which impacts the amount that will be withdrawn eventually.

    See [SWC-114](https://swcregistry.io/docs/SWC-114)

<!-- Based on Geth default ordering, it's easy to sandwich a transaction by sending two transactions each with 1 wei higher or lower. -->

<!-- Cite theo/daniel's talk -->

#### Suppression

In a suppression attack, a.k.a *Block Stuffing* attacks, after Mallory runs her function, she tries
to delay Alice from running her function.

This was the case with the first winner of the "Fomo3d" game and some other on-chain hacks. The
attacker sent multiple transactions with a high `gasPrice` and `gasLimit` to custom smart contracts
that assert (or use other means) to consume all the gas and fill up the block's `gasLimit`.

!!! note "Variants"
    Each of these attacks has two variants, *asymmetric* and *bulk*.

    In some cases, Alice and Mallory are performing different operations. For example, Alice is trying to cancel an offer, and Mallory is trying to fulfill it first. We call this *asymmetric displacement*. In other cases, Mallory is trying to run a large set of functions: for example, Alice and others are trying to buy a limited set of shares offered by a firm on a blockchain. We call this *bulk displacement*.


### Mitigations

Front-running is a pervasive issue on public blockchains such as Ethereum.

The best remediation is to **remove the benefit of front-running in your application**, mainly by
removing the importance of transaction ordering or time. For example, in markets, it would be
better to implement batch auctions (this also protects against high-frequency trading concerns).
Another way is to use a pre-commit scheme (“I’m going to submit the details later”). A third option
is to mitigate the cost of front-running by specifying a maximum or minimum acceptable price range
on a trade, thereby limiting price slippage.

**Transaction Ordering:** Go-Ethereum (Geth) nodes, order the transactions based on their
`gasPrice` and address nonce. This, however, results in a gas auction between participants in the
network to get included in the block currently being mined.

**Confidentiality:** Another approach is to limit the visibility of the transactions, this can be
done using a "commit and reveal" scheme.

<!-- cite and properly define commit and reveal -->

A simple implementation is to store the keccak256 hash of the data in the first transaction, then
reveal the data and verify it against the hash in the second transaction. However note that the
transaction itself leaks the intention and possibly the value of the collateralization. There are
enhanced commit and reveal schemes that are more secure, however require more transactions to
function, e.g. [submarine sends](https://libsubmarine.org/).
