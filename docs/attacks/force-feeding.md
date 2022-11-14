Forcing a smart contract to hold an Ether balance can influence its internal accounting and security assumptions.
There are multiple ways a smart contract can receive Ether. The hierarchy is as follows:

1. Check whether a payable external `receive` function is defined.
2. If not, check whether a payable external `fallback` function is defined.
3. Revert.

The precedence of each function is explained in this great graphic from the [Solidity by Example](https://solidity-by-example.org/sending-ether/) article:

```
Which function is called, fallback() or receive()?

           send Ether
               |
         msg.data is empty?
              / \
            yes  no
            /     \
receive() exists?  fallback()
         /   \
        yes   no
        /      \
    receive()   fallback()
```

Consider the following example:

```sol
pragma solidity ^0.8.13;

contract Vulnerable {
    receive() external payable {
        revert();
    }

    function somethingBad() external {
        require(address(this).balance > 0);
        // Do something bad
    }
}
```

The contract's logic seemingly disallows direct payments and prevents "something bad" from happening.
However, calling `revert` in both `fallback` and `receive` **cannot prevent the contract from receiving Ether**.
The following techniques can be used to force-feed Ether to a smart contract.

### Selfdestruct

When the `SELFDESTRUCT` opcode is called, funds of the calling address are sent to the address on the stack, and execution is immediately halted.
Since this opcode works on the EVM-level, Solidity-level functions that might block the receipt of Ether [will not be executed](https://solidity.readthedocs.io/en/develop/security-considerations.html#sending-and-receiving-ether).


### Pre-calculated Deployments

Additionally, the target address of newly deployed smart contracts is generated in a deterministic fashion.
The address generation can be looked up in any EVM implementation, such as the [py-evm reference implementation](https://github.com/ethereum/py-evm/blob/e924f63992a35212616b4e20355d161bc4348925/eth/_utils/address.py#L17-L18) by the Ethereum Foundation:

```python
def generate_contract_address(address: Address, nonce: int) -> Address:
    return force_bytes_to_address(keccak(rlp.encode([address, nonce])))
```

An attacker can send funds to this address before the deployment has happened.
This is also illustrated by [this 2017 Underhanded Solidity Contest submission](https://github.com/Arachnid/uscc/tree/master/submissions-2017/ricmoo).


### Block Rewards and Coinbase

Depending on the attacker's capabilities, they can also start proof-of-work mining.
By setting the target address to their `coinbase`, block rewards will be added to its balance.
As this is yet another EVM-level capability, checks performed by Solidity are ineffective.


### Solution

The above effects illustrate that relying on exact comparisons to the contract's Ether balance is unreliable.
The smart contract's business logic must consider that the actual balance associated with it can be higher than the internal accounting's value.

**In general, we strongly advise against using the contract's balance as a guard.**

More information can be found in [SWC-132](https://swcregistry.io/docs/SWC-132).
