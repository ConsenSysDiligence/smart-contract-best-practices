
# Token Implementation Best Practice

Implementing Tokens should comply with other best practices, but also have some unique considerations.

## Comply with the latest standard

Generally speaking, smart contracts of tokens should follow an accepted and stable standard. 

Examples of currently accepted standards include:

* [EIP20](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md)
* [EIP721](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-721.md) (non-fungible token)
* More at [eips.ethereum.org](https://eips.ethereum.org/erc#final)

## Be aware of front running attacks on EIP-20

The EIP-20 token's `approve()` function creates the potential for an approved spender to spend more than the intended amount. A [front running attack](./known_attacks/#transaction-ordering-dependence-tod-front-running) can be used, enabling an approved spender to call `transferFrom()` both before and after the call to `approve()` is processed. More details are available on the [EIP](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md#approve), and in [this document](https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit).


## Prevent transferring tokens to the 0x0 address

At the time of writing, the "zero" address ([0x0000000000000000000000000000000000000000](https://etherscan.io/address/0x0000000000000000000000000000000000000000)) holds tokens with a value of more than 80$ million.


## Prevent transferring tokens to the contract address

Consider also preventing the transfer of tokens to the same address of the smart contract. 

An example of the potential for loss by leaving this open is the [EOS token smart contract](https://etherscan.io/address/0x86fa049857e0209aa7d9e616f7eb3b3b78ecfdb0) where more than 90,000 tokens are stuck at the contract address. 

### Example

An example of implementing both the above recommendations would be to create the following modifier; validating that the "to" address is neither 0x0 nor the smart contract's own address:

```sol
    modifier validDestination( address to ) {
        require(to != address(0x0));
        require(to != address(this) );
        _;
    }
```

The modifier should then be applied to the "transfer" and "transferFrom" methods:

```sol 
    function transfer(address _to, uint _value)
        validDestination(_to)
        returns (bool) 
    {
        (... your logic ...)
    }

    function transferFrom(address _from, address _to, uint _value)
        validDestination(_to)
        returns (bool) 
    {
        (... your logic ...)
    }
```
