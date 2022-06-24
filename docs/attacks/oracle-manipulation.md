Protocols sometimes require additional information from outside the realm of the blockchain to function correctly.
Such off-chain information is provided by [oracles](https://ethereum.org/en/developers/docs/oracles/), which often are smart contracts themselves.

A vulnerability arises when protocols relying on oracles automatically execute actions even though the oracle-provided data feed is incorrect.
An oracle with deprecated or even malicious contents can have disastrous effects on all processes connected to the data feed.
In practice, manipulated data feeds can cause significant damage, from unwarranted liquidations to malicious arbitrage trades.
The following sections provide examples illustrating common vulnerabilities and malfunctions involving oracles.


## Spot Price Manipulation

A classic vulnerability comes from the world of on-chain price oracles: Trusting the spot price of a decentralized exchange.

The scenario is simple. A smart contract needs to determine the price of an asset, e.g., when a user deposits ETH into its system.
To achieve this price discovery, the protocol consults its respective Uniswap pool as a source.
Exploiting this behavior, an attacker can take out a flash loan to drain one side of the Uniswap pool.
Due to the lack of data source diversity, the protocol's internal price is directly manipulated, e.g., to 100 times the original value.
The attacker can now perform an action to capture this additional value.
For example, an arbitrage trade on top of the newly created price difference or an advantageous position in the system can be gained.

The problems are two-fold:

1. The use of a single price feed source smart contract allows for easy on-chain manipulation using flash loans.
2. Despite a notable anomaly, the smart contracts consuming the price information continue to operate on the manipulated data.

A more concrete example is provided by the Visor Hack.
The [following code](https://github.com/VisorFinance/hypervisor/blob/e772228ed5e27239161c3173c550265b5548e9f5/contracts/Hypervisor.sol#L102-L103) shows that on deposit, the price feed is fetched directly from Uniswap:

```solidity
uint160 sqrtPrice = TickMath.getSqrtRatioAtTick(currentTick());
uint256 price = FullMath.mulDiv(uint256(sqrtPrice).mul(uint256(sqrtPrice)), PRECISION, 2**(96 * 2));
```

Here, `currentTick()` directly fetches the [current price tick](https://github.com/VisorFinance/hypervisor/blob/e772228ed5e27239161c3173c550265b5548e9f5/contracts/Hypervisor.sol#L515-L518) from a Uniswap pool:

```solidity
// @return tick Uniswap pool's current price tick
function currentTick() public view returns (int24 tick) {
    (, tick, , , , , ) = pool.slot0();
}
```

As this price data is fetched from an on-chain dependency, and the price data is determined in the current transaction context, this spot price can be manipulated in the same transaction.

1. An attacker can take out a flash loan on the incoming asset A and on the relevant Uniswap pool, swap asset A for asset B with a large volume.
2. This trade will increase the price of asset B (increased demand) and reduce the cost of asset A (increased supply).
3. When asset B is deposited into the above function, its price is still pumped up by the flash loan.
4. Consequentially, asset B gives the attacker an over-proportional amount of shares.
5. These shares can be withdrawn, giving the attacker equal parts of asset A and asset B from the pool.
6. Repeating this process will drain the vulnerable pool of all funds.
7. With the money gained from the withdrawal of their shares, the attacker can repay the flash loan.

!!! Warning
    Under no circumstances should a decentralized exchange's spot price be used directly for price discovery.
    Secure price calculation can be performed, e.g., by using time-weighted average prices (TWAPs) across longer time intervals.
    Assuming sufficient liquidity, this severely increases the cost of a price manipulation attack, making it unfeasible.
    An example for facilitating secure price discovery is the [Uniswap V3 `OracleLibrary`](https://docs.uniswap.org/protocol/reference/periphery/libraries/OracleLibrary) docs.


## Off-Chain Infrastructure

By definition, a data feed transporting off-chain information into a smart contract requires traditional software to run.
From the sensor hardware or manual entry to authenticated APIs submitting data on-chain, it is not uncommon for a plethora of software to be involved.

Depending on the concrete implementation, attacks on access control, cryptographic implementation, transport, and database security, among others, can be performed.
As a result, software providing oracle services must be hardened and adhere to security best practices such as the [OWASP Secure Coding Practices](https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/migrated_content).
Especially oracles which don't offer a community-driven dispute phase must be hardened as their compromise will directly affect dependent applications.

!!! Info
    Eskandari et al. divided the concept of an oracle into the following six modules:

    - Ground Truth
    - Data Sources
    - Data Feeders
    - Selection of Data Feeders
    - Aggregation
    - Dispute Phase

    Their publication provides a great read on design principles, attacks, and mitigations.

    ```
    Shayan Eskandari, Mehdi Salehi, Wanyun Catherine Gu, and Jeremy Clark. 2021.
    SoK: oracles from the ground truth to market manipulation.
    Proceedings of the 3rd ACM Conference on Advances in Financial Technologies.
    Association for Computing Machinery, New York, NY, USA, 127–141.
    DOI:https://doi.org/10.1145/3479722.3480994
    ```

An excellent example of an off-chain component malfunction affecting on-chain oracle data feeds is the Synthetix sKRW incident.
Synthetix aggregates multiple related price-feeds to accurately price their derivatives and surfaces the aggregate through a smart contract on-chain.
With a value erroneously reported 1000 times higher than the original, the price of the Korean Won was reported significantly higher, even though the aggregation.
An arbitrage bot used this effect, which promptly earned it a profit of over 1B USD.
While on-chain aggregation and price reporting worked correctly, an off-chain component failure resulted in the incident.

!!! Info
    samczsun wrote a [great article](https://www.paradigm.xyz/2020/11/so-you-want-to-use-a-price-oracle) on the Paradigm blog elaborating this incident and various other price oracle related incidents.


## Centralized Oracles and Trust

Projects can also choose to implement a centralized oracle.
Such a smart contract's update method can, e.g., be protected by an `onlyOwner` modifier and require users to trust in the correct and timely submission of data.
Depending on the size and structure of the system, this centralized trust can lead to the authorized user(s) getting incentivized to submit malicious data and abuse their position of power.

Additionally, such centralized systems can have an inherent risk due to compromised private keys.


## Decentralized Oracle Security

Decentralized oracles aim to diversify the group of data collectors to a point where disrupting a quorum of participants becomes unfeasible for an attacker.
In a decentralized scenario, further security considerations stem from how participants are incentivized and what sort of misbehavior if left unpunished.
Participants providing (valid) data to the oracle system are economically rewarded.
Aiming to maximize their profit, the participants are incentivized to provide the cheapest version of their service possible.

### Freeloading

**Freeloading** attacks are the simplest form to save work and maximize profit.
A node can leverage another oracle or off-chain component (such as an API) and simply copy the values without validation.
For example, an oracle providing weather data might expect data providers to measure temperature and wind speed in a specific location.
Nodes are, however, incentivized to use a publicly available weather data API and simply surface their data to the system.
Besides the apparent data source centralization issue, freeloading attacks at scale can also severely affect the data's correctness.
This effect is most visible when sampling rates vary, e.g., the on-chain oracle expects a sampling rate of 10 minutes while freeloading nodes provide data from an API that is updated once every hour.

Freeloading in decentralized oracle data marketplaces can amplify a price race to the bottom as freeloading only requires a simple lookup. At the same time, proper data provisioning might involve a more significant computational overhead.
With less competition in cheaper price ranges, a few freeloading nodes could even be able to take over a data feed.
Freeloading attacks can be easily prevented for more complex data feeds by implementing a commit-reveal scheme.
This security measure will prevent oracle system participants from peeking into each other's data.
For simpler data provisioning, consistency checks punishing nodes that obviously copy data from well-known public services can be implemented.
Data collectors contributing to the centralization of the overall service will be disincentivized.

### Mirroring

**Mirroring** attacks are a flavor of Sybil attacks and can go hand-in-hand with freeloading.
Similarly, misbehaving nodes aim to save work by reading from a centralized data source, optionally with a reduced sampling rate.
A single node reading from the centralized data source then replicates its values across other participants who mirror that data.
With a single data read, the reward for providing the information is multiplied by the number of participants.
As the number of mirroring participants grows, this increased weight on a single data point can significantly deteriorate error correction mechanisms.
A similar outcome to a mirroring attack can happen accidentally when a large, uninformed part of a community relies on a single data source.

A commit-reveal scheme is ineffective to mitigate (purposeful) mirroring attacks as it does not consider private data transfers between Sybil nodes.
Due to the lack of transparency in Sybil communications, mirroring attacks can be very hard to detect in practice.


## Solutions

Currently, the easiest ways to solve the oracle problem are decentralized oracles, such as:

* [Chainlink](https://chain.link/) is the largest decentralized oracle provider, and the Chainlink network can be leveraged to bring decentralized data on-chain.
* [Tellor](https://tellor.io/) is an oracle that provides censorship-resistant data, secured by economic incentives, ensuring data can be provided by anyone, anytime, and checked by everyone.
* [Witnet](https://witnet.io/) leverages state-of-the-art cryptographic and economic incentives to provide smart contracts with off-chain data.

Using a median of multiple oracles provides heightened security since it is harder and more expensive to attack various oracles.
It also ensures that a smart contract gets the data it needs even if one oracle or API call fails. 

Another standard solution is to use a time-weighted average price feed so that price is averaged out
over X periods and multiple sources.
Not only does this prevent oracle manipulation, but it also reduces the chance you can be front-run, as an order executed right before cannot have as drastic of an impact on the price.
This condition does not apply for low liquidity assets, which are generally cheaper to manipulate, even for a prolonged time.
Uniswap v2 provides a [sliding window example](https://github.com/Uniswap/v2-periphery/blob/2efa12e0f2d808d9b49737927f0e416fafa5af68/contracts/examples/ExampleSlidingWindowOracle.sol).
