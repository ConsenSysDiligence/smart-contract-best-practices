While much of your programming experience will be relevant to Ethereum programming, there are some
pitfalls to be aware of.

- Be extremely careful about external contract calls, which may execute malicious code and change
  control flow.
- Understand that your public functions are public, and may be called maliciously and in any order.
  The private data in smart contracts is also viewable by anyone.
- Keep gas costs and the block gas limit in mind.
- Be aware that timestamps are imprecise on a blockchain, miners can influence the time of
  execution of a transaction within a margin of several seconds.
- Randomness is non-trivial on blockchain, most approaches to random number generation are gameable
  on a blockchain.
