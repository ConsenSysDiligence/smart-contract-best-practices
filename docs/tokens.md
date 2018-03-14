
# Token Implementation Best Practice
Although, implementing Tokens should comply with other best practices, Tokens has there unique details.

## Comply with the latest standard
Generally speaking, smart contracts of tokens should follow the latest stable standard. The current widely adopted consideration is the ERC20 standard. You can find the original discussion of it on GitHub: https://github.com/ethereum/EIPs/issues/20. 

An example of a standard ERC20 token implementation is [OmiseGo](https://etherscan.io/address/0xd26114cd6EE289AccF82350c8d8487fedB8A0C07#code). On the other hand, the  [DOT Allocation Indicator](https://etherscan.io/address/0xb59f67a8bff5d8cd03f6ac17265c550ed8f33907#code) is an example of a token that is not ERC20 compliant because, for some purpose, it implements some but not all of ERC20 functions.

## Prevent transferring tokens to the 0x0 address
If a code allows human mistakes, the odds are high that it will happen. By the time of writing this document, the address "0x0000000000000000000000000000000000000000" contains tokens with a value of more than 80$ million. And this is, unfortunately, increasing in both count and value. 
Please, reference for example: AMIS, Arcade, LAToken and many others, that are stuck at the [0x0](https://etherscan.io/address/0x0000000000000000000000000000000000000000) address. This is a result of a careless code of those smart contracts.

However, as you can see, there are also more than 7 thousand Ether stuck at the [0x0](https://etherscan.io/address/0x0000000000000000000000000000000000000000) address. But, Ether could be resend back to its original addresses with a special or the next hard-fork. Whereas, sending the tokens back to there owners should be managed and implemented on a case-by-case bases, because every token has its own code. So, consider that for the sake of both yourself and your token's holders.

## Prevent transferring tokens to the contract address
Another best practice consideration would be to prevent the transfer of tokens to the same address of the smart contract. To reference an example of the harm, please check [EOS token smart contract](https://etherscan.io/address/0x86fa049857e0209aa7d9e616f7eb3b3b78ecfdb0) where more than 90,000 tokens are stuck at the contract address. Another example is the [LATToken.](https://etherscan.io/token/0xe50365f5d679cb98a1dd62d6f6e58e59321bcddf) Tokens were just transferred to the contract address by mistake and the contract was not designed to prevent this.
### Example
To see a good example of implementing both: **Prevent transfer to the 0x0 address** and **Prevent transfer to the contract address**, you can check the smart contract of **[Kyber Network Crystal](https://etherscan.io/address/0xdd974d5c2e2928dea5f71b9825b8b646686bd200#code)** token.
It is utilizing the following modifier; Which validate that the "to" address is not 0x0 nor the smart contract address:

    modifier validDestination( address to ) {
        require(to != address(0x0));
        require(to != address(this) );
        _;
    }
And the modifier should be applied on **both** "transfer" and "transferFrom" methods. The following is an example:

    function transfer(address _to, uint _value)
        validDestination(_to)
        [other modifiers]
        returns (bool) {
        (... your logic ...)
    }

    function transferFrom(address _from, address _to, uint _value)
        validDestination(_to)
        [other modifiers]
        returns (bool) {
        (... your logic ...)
    }
To reference an example of a poor ERC20 token code that is mintable with fair complexity, but does not take in consideration the above mentioned, check [LAToken](https://etherscan.io/address/0xe50365f5d679cb98a1dd62d6f6e58e59321bcddf#code).
