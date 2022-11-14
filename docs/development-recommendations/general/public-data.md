Many applications require submitted data to be private up until some point in time in order to
work. Games (eg. on-chain rock-paper-scissors) and auction mechanisms (eg. sealed-bid
[Vickrey auctions](https://en.wikipedia.org/wiki/Vickrey_auction)) are two major categories of
examples. If you are building an application where privacy is an issue, make sure you avoid
requiring users to publish information too early. The best strategy is to use
[commitment schemes](https://en.wikipedia.org/wiki/Commitment_scheme) with separate phases: first
commit using the hash of the values and in a later phase revealing the values.

However, care must be taken to ensure that the hashed value stored isn't recognisable (and thus, de-mappable), as this would defeat the second purpose of hashing - preventing the reveal of such values. Here's an example:

Say a smart contract allows 2 players to play rock-paper-scissors, and uses this commit-reveal scheme - both players have to send a hash of their move before either of them sends the last (game ending) transaction. Here's what the keccak256 hash of `rock` is: `10977e4d68108d418408bc9310b60fc6d0a750c63ccef42cfb0ead23ab73d102`. If you were playing, and you saw your opponent commiting this, wouldn't this tell you exactly what move your opponent has committed to? A safer implementation would be to hash not just the name of the move, but also, say, a user chosen salt. That would make the resulting salt non-recognisable.

Examples:

- In rock paper scissors, require both players to submit a hash of their intended move first, then
  require both players to submit their move; if the submitted move does not match the hash throw it
  out.
- In an auction, require players to submit a hash of their bid value in an initial phase (along
  with a deposit greater than their bid value), and then submit their auction bid value in the
  second phase.
- When developing an application that depends on a random number generator, the order should always
  be *(1)* players submit moves, *(2)* random number generated, *(3)* players paid out. The method
  by which random numbers are generated is itself an area of active research; current best-in-class
  solutions include Bitcoin block headers (verified through http://btcrelay.org),
  hash-commit-reveal schemes (ie. one party generates a number, publishes its hash to "commit" to
  the value, and then reveals the value later) and [RANDAO](http://github.com/randao/randao). As
  Ethereum is a deterministic protocol, no variable within the protocol could be used as an
  unpredictable random number. Also, be aware that [miners are in some extent in control of the
  `block.blockhash()`
  value](https://ethereum.stackexchange.com/questions/419/when-can-blockhash-be-safely-used-for-a-random-number-when-would-it-be-unsafe).
