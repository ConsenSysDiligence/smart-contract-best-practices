### Security Related EIPs

The following EIPs are important to be aware of, either for understanding how the EVM works, or to inform proper best practices when developing a smart contract system. 

Please do not assume this is an exhaustive list. 

### Final

- [EIP 155](https://eips.ethereum.org/EIPS/eip-155) Simple replay attack protection - Provides a way to send transactions that work on Ethereum mainnet without working on other chains such as Ethereum Classic, various testnets or consortium chains.
- [EIP 214](https://eips.ethereum.org/EIPS/eip-214) New opcode STATICCALL - Adds a new opcode that can be used to call another
contract (or itself) while disallowing any modifications to the state during the call.
- [EIP 607](https://eips.ethereum.org/EIPS/eip-607) Hardfork Meta: Spurious Dragon - Implemented the Simple Replay Attack Protection from EIP 155.
- [EIP 779](https://eips.ethereum.org/EIPS/eip-779) Hardfork Meta: DAO Fork - Documents the changes included in the hard fork named
“DAO Fork”. Unlike other hard forks, the DAO Fork did not change the protocol. Rather, the DAO Fork was an “irregular state change”
that transferred ether balances from a list of accounts (“child DAO” contracts) into a specified account (the “WithdrawDAO”
contract).

### Draft

- [EIP 1470](https://eips.ethereum.org/EIPS/eip-1470) Smart Contract Weakness Classification - Proposes a classification scheme for
security weaknesses in Ethereum smart contracts.(SWC)
- [EIP 1051](https://eips.ethereum.org/EIPS/eip-1051) Overflow checking for the EVM - Adds two new opcodes that permit efficient
detection and prevention of overflows.
- [EIP 1271](https://eips.ethereum.org/EIPS/eip-1271) Standard Signature Validation Method for Contracts - The current design of 
many smart contracts prevents contract based accounts from interacting with them, since contracts do not possess private keys and 
therefore can not directly sign messages. The proposal here outlines a standard way for contracts to verify if a provided signature 
is valid when the account is a contract.
