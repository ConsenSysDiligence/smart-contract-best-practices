There are multiple fundamental tradeoffs to consider when assessing the structure and security of a
smart contract system. The general recommendation for any smart contract system is to identify the
proper balance for these fundamental tradeoffs.

An ideal smart contract system from a software engineering bias is modular, reuses code instead of
duplicating it, and supports upgradeable components. An ideal smart contract system from a secure
architecture bias may share this mindset, especially in the case of more complex smart contract
systems.

However, there are important exceptions where security and software engineering best practices may
not be aligned. In each case, the proper balance is obtained by identifying the optimal mix of
properties along contract system dimensions such as:

- Rigid versus Upgradeable
- Monolithic versus Modular
- Duplication versus Reuse

### Rigid versus Upgradeable

While multiple resources, including this one, emphasize malleability characteristics such as
Killable, Upgradeable or Modifiable patterns there is a *fundamental tradeoff* between malleability
and security.

Malleability patterns by definition add complexity and potential attack surfaces. Simplicity is
particularly effective over complexity in cases where the smart contract system performs a very
limited set of functionality for a pre-defined limited period of time, for example, a
governance-free finite-time-frame token-sale contract system.

### Monolithic versus Modular

A monolithic self-contained contract keeps all knowledge locally identifiable and readable. While
there are few smart contract systems held in high regard that exist as monoliths, there is an
argument to be made for extreme locality of data and flow - for example, in the case of optimizing
code review efficiency.

As with the other tradeoffs considered here, security best practices trend away from software
engineering best practices in simple short-lived contracts and trend toward software engineering
best practices in the case of more complex perpetual contract systems.

### Duplication versus Reuse

A smart contract system from a software engineering perspective wishes to maximize reuse where
reasonable. There are many ways to reuse contract code in Solidity. Using proven
previously-deployed contracts *which you own* is generally the safest manner to achieve code reuse.

Duplication is frequently relied upon in cases where self-owned previously-deployed contracts are
not available. Efforts such as
[OpenZeppelin's Solidity Library](https://github.com/OpenZeppelin/openzeppelin-contracts) seek to
provide patterns such that secure code can be re-used without duplication. Any contract security
analysis must include any re-used code that has not previously established a level of trust
commensurate with the funds at risk in the target smart contract system.
