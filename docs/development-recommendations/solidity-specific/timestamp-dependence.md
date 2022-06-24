There are three main considerations when using a timestamp to execute a critical function in a
contract, especially when actions involve fund transfer.

#### Timestamp Manipulation

Be aware that the timestamp of the block can be manipulated by a miner. Consider this
[contract](https://etherscan.io/address/0xcac337492149bdb66b088bf5914bedfbf78ccc18#code):

```sol

uint256 constant private salt =  block.timestamp;

function random(uint Max) constant private returns (uint256 result){
    //get the best seed for randomness
    uint256 x = salt * 100/Max;
    uint256 y = salt * block.number/(salt % 5) ;
    uint256 seed = block.number/3 + (salt % 300) + Last_Payout + y;
    uint256 h = uint256(block.blockhash(seed));

    return uint256((h / x)) % Max + 1; //random number between 1 and Max
}
```

When the contract uses the timestamp to seed a random number, the miner can actually post a
timestamp within 15 seconds of the block being validated, effectively allowing the miner to
precompute an option more favorable to their chances in the lottery. Timestamps are not random and
should not be used in that context.

#### The 15-second Rule

The [Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf) (Ethereum's reference
specification) does not specify a constraint on how much blocks can drift in time, but
[it does specify](https://ethereum.stackexchange.com/a/5926/46821) that each timestamp should be
bigger than the timestamp of its parent. Popular Ethereum protocol implementations
[Geth](https://github.com/ethereum/go-ethereum/blob/4e474c74dc2ac1d26b339c32064d0bac98775e77/consensus/ethash/consensus.go#L45)
and
[Parity](https://github.com/paritytech/parity-ethereum/blob/73db5dda8c0109bb6bc1392624875078f973be14/ethcore/src/verification/verification.rs#L296-L307)
both reject blocks with timestamp more than 15 seconds in future. Therefore, a good rule of thumb
in evaluating timestamp usage is:

!!! Note
    If the scale of your time-dependent event can vary by 15 seconds and maintain integrity,
    it is safe to use a `block.timestamp`.

#### Avoid using `block.number` as a timestamp

It is possible to estimate a time delta using the `block.number` property and
[average block time](https://etherscan.io/chart/blocktime), however this is not future proof as
block times may change (such as
[fork reorganisations](https://blog.ethereum.org/2015/08/08/chain-reorganisation-depth-expectations/)
and the [difficulty bomb](https://github.com/ethereum/EIPs/issues/649)). In a sale spanning days,
the 15-second rule allows one to achieve a more reliable estimate of time.

See [SWC-116](https://swcregistry.io/docs/SWC-116)
