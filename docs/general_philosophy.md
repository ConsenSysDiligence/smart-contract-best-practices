Ethereum and complex blockchain programs are new and highly experimental. Therefore, you should expect constant changes in the security landscape, as new bugs and security risks are discovered, and new best practices are developed. Following the security practices in this document is therefore only the beginning of the security work you will need to do as a smart contract developer.

Smart contract programming requires a different engineering mindset than you may be used to. The cost of failure can be high, and change can be difficult, making it in some ways more similar to hardware programming or financial services programming than web or mobile development. It is therefore not enough to defend against known vulnerabilities. Instead, you will need to learn a new philosophy of development:

## Prepare for failure

Any non-trivial contract will have errors in it. Your code must, therefore, be able to respond to bugs and vulnerabilities gracefully.

  - Pause the contract when things are going wrong ('circuit breaker')
  - Manage the amount of money at risk (rate limiting, maximum usage)
  - Have an effective upgrade path for bugfixes and improvements

## Rollout carefully

It is always better to catch bugs before a full production release.

  - Test contracts thoroughly, and add tests whenever new attack vectors are discovered
  - Provide [bug bounties](software_engineering.md#bug-bounty-programs) starting from alpha testnet releases
  - Rollout in phases, with increasing usage and testing in each phase

## Keep contracts simple

Complexity increases the likelihood of errors.

  - Ensure the contract logic is simple
  - Modularize code to keep contracts and functions small
  - Use already-written tools or code where possible (eg. don't roll your own random number generator)
  - Prefer clarity to performance whenever possible
  - Only use the blockchain for the parts of your system that require decentralization

## Stay up to date

Use the resources listed in the next section to keep track of new security developments.

  - Check your contracts for any new bug that's discovered
  - Upgrade to the latest version of any tool or library as soon as possible
  - Adopt new security techniques that appear useful

## Be aware of blockchain properties

While much of your programming experience will be relevant to Ethereum programming, there are some pitfalls to be aware of.

  - Be extremely careful about external contract calls, which may execute malicious code and change control flow.
  - Understand that your public functions are public, and may be called maliciously. Your private data is also viewable by anyone.
  - Keep gas costs and the block gas limit in mind.
  - Be aware that timestamps are imprecise on a blockchain, miners can influence the time of execution of a transaction within a margin of several seconds.
  - Randomness is non-trivial on blockchain, most approaches to random number generation are gameable on a blockchain.

## Fundamental Tradeoffs: Simplicity versus Complexity cases

There are multiple fundamental tradeoffs to consider when assessing the structure and security of a smart contract system.  The general recommendation for any smart contract system is to identify the proper balance for these fundamental tradeoffs.

An ideal smart contract system from a software engineering bias is modular, reuses code instead of duplicating it, and supports upgradeable components.  An ideal smart contract system from a secure architecture bias may share this mindset, especially in the case of more complex smart contract systems.

However, there are important exceptions where security and software engineering best practices may not be aligned.  In each case, the proper balance is obtained by identifying the optimal mix of properties along contract system dimensions such as:

- Rigid versus Upgradeable
- Monolithic versus Modular
- Duplication versus Reuse

### Rigid versus Upgradeable

While multiple resources, including this one, emphasize malleability characteristics such as Killable, Upgradeable or Modifiable patterns there is a *fundamental tradeoff* between malleability and security.

Malleability patterns by definition add complexity and potential attack surfaces.  Simplicity is particularly effective over complexity in cases where the smart contract system performs a very limited set of functionality for a pre-defined limited period of time, for example, a governance-free finite-time-frame token-sale contract system.

### Monolithic versus Modular

A monolithic self-contained contract keeps all knowledge locally identifiable and readable.  While there are few smart contract systems held in high regard that exist as monoliths, there is an argument to be made for extreme locality of data and flow - for example, in the case of optimizing code review efficiency.

As with the other tradeoffs considered here, security best practices trend away from software engineering best practices in simple short-lived contracts and trend toward software engineering best practices in the case of more complex perpetual contract systems.

### Duplication versus Reuse

A smart contract system from a software engineering perspective wishes to maximize reuse where reasonable.  There are many ways to reuse contract code in Solidity.  Using proven previously-deployed contracts *which you own* is generally the safest manner to achieve code reuse.

Duplication is frequently relied upon in cases where self-owned previously-deployed contracts are not available.  Efforts such as [Zeppelin Solidity](https://github.com/OpenZeppelin/zeppelin-solidity) seek to provide patterns such that secure code can be re-used without duplication.  Any contract security analyses must include any re-used code that has not previously established a level of trust commensurate with the funds at risk in the target smart contract system.
