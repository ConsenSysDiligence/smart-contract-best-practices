
These are attacks which are no longer possible due to changes in the protocol or improvements to
solidity. They are recorded here for posterity and awareness.

### Call Depth Attack (deprecated)

As of the [EIP 150](https://github.com/ethereum/EIPs/issues/150) hardfork, call depth attacks are
no longer
relevant<sup><a href='http://ethereum.stackexchange.com/questions/9398/how-does-eip-150-change-the-call-depth-attack'>\*</a></sup>
(all gas would be consumed well before reaching the 1024 call depth limit).

### Constantinople Reentrancy Attack

On January 16th, 2019, Constantinople protocol upgrade was delayed due to a security vulnerability
enabled by [EIP 1283](https://eips.ethereum.org/EIPS/eip-1283). _EIP 1283: Net gas metering for
SSTORE without dirty maps_ proposes changes to reduce excessive gas costs on dirty storage writes.

This change led to possibility of a new reentrancy vector making previously known secure withdrawal
patterns (`.send()` and `.transfer()`) unsafe in specific
situations<sup><a href='https://medium.com/chainsecurity/constantinople-enables-new-reentrancy-attack-ace4088297d9'>\*</a></sup>,
where the attacker could hijack the control flow and use the remaining gas enabled by EIP 1283,
leading to vulnerabilities due to reentrancy.
