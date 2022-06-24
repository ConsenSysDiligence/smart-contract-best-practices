Beware of coding an invariant that strictly checks the balance of a contract.

An attacker can forcibly send ether to any account and this cannot be prevented (not even with a
fallback function that does a `revert()`).

The attacker can do this by creating a contract, funding it with 1 wei, and invoking
`selfdestruct(victimAddress)`. No code is invoked in `victimAddress`, so it cannot be prevented.
This is also true for block reward which is sent to the address of the miner, which can be any
arbitrary address.

Also, since contract addresses can be precomputed, ether can be sent to an address before the
contract is deployed.

See [SWC-132](https://swcregistry.io/docs/SWC-132)
