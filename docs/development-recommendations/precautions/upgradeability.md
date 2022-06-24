!!! warning
    Smart Contract upgradeability is an active area of research. There are many important
    questions, and risks related to smart contract upgradeability. Do your research into the state of
    the art. We welcome discussion on the
    [related issue](https://github.com/ConsenSys/smart-contract-best-practices/issues/164).

Code will need to be changed if errors are discovered or if improvements need to be made. It is no
good to discover a bug, but have no way to deal with it.

Designing an effective upgrade system for smart contracts is an area of active research, and we
won't be able to cover all of the complications in this document. However, two basic approaches are
most commonly used. The simpler of the two is to have a registry contract that holds the address of
the latest version of the contract. A more seamless approach for contract users is to have a
contract that forwards calls and data onto the latest version of the contract.

Whatever the technique, it's important to have modularization and good separation between
components, so that code changes do not break functionality, orphan data, or require substantial
costs to port. In particular, it is usually beneficial to separate complex logic from your data
storage, so that you do not have to recreate all of the data in order to change the functionality.

It's also critical to have a secure way for parties to decide to upgrade the code. Depending on
your contract, code changes may need to be approved by a single trusted party, a group of members,
or a vote of the full set of stakeholders. If this process can take some time, you will want to
consider if there are other ways to react more quickly in case of an attack, such as an
[emergency stop or circuit-breaker](./circuit-breakers.md).

Regardless of your approach, it is important to have some way to upgrade your contracts, or they
will become unusable when the inevitable bugs are discovered in them.

#### Example 1: Use a registry contract to store the latest version of a contract

In this example, the calls aren't forwarded, so users should fetch the current address each time
before interacting with it.

```sol
pragma solidity ^0.5.0;

contract SomeRegister {
    address backendContract;
    address[] previousBackends;
    address owner;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner)
        _;
    }

    function changeBackend(address newBackend) public
    onlyOwner()
    returns (bool)
    {
        if(newBackend != address(0) && newBackend != backendContract) {
            previousBackends.push(backendContract);
            backendContract = newBackend;
            return true;
        }

        return false;
    }
}
```

There are two main disadvantages to this approach:

1. Users must always look up the current address, and anyone who fails to do so risks using an old
   version of the contract
1. You will need to think carefully about how to deal with the contract data when you replace the
   contract

The alternate approach is to have a contract forward calls and data to the latest version of the
contract:

#### Example 2: [Use a `DELEGATECALL`](http://ethereum.stackexchange.com/questions/2404/upgradeable-contracts) to forward data and calls

This approach relies on using the
[fallback function](https://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) (in
`Relay` contract) to forward the calls to a target contract (`LogicContract`) using
[delegatecall](https://solidity.readthedocs.io/en/latest/introduction-to-smart-contracts.html#delegatecall-callcode-and-libraries).
Remember that `delegatecall` is a special function in Solidity that executes the logic of the
called address (`LogicContract`) in the context of the calling contract (`Relay`), so *"storage,
current address and balance still refer to the calling contract , only the code is taken from the
called address"*.

```sol
pragma solidity ^0.5.0;

contract Relay {
    address public currentVersion;
    address public owner;

    modifier onlyOwner() {
        require(msg.sender == owner);
        _;
    }

    constructor(address initAddr) {
        require(initAddr != address(0));
        currentVersion = initAddr;
        owner = msg.sender; // this owner may be another contract with multisig, not a single contract owner
    }

    function changeContract(address newVersion) public
    onlyOwner()
    {
        require(newVersion != address(0));
        currentVersion = newVersion;
    }

    fallback() external payable {
        (bool success, ) = address(currentVersion).delegatecall(msg.data);
        require(success);
    }
}
```

```sol
contract LogicContract {
    address public currentVersion;
    address public owner;
    uint public counter;

    function incrementCounter() {
        counter++;
    }
}
```

This simple version of the pattern cannot return values from `LogicContract`'s functions, only
forward them, which limits its applicability. More complex implementations attempt to solve this
with in-line assembly code and a registry of return sizes. They are commonly referred to as
[Proxy Patterns](https://blog.openzeppelin.com/proxy-patterns/), but are also known as
[Router](https://github.com/PeterBorah/ether-router),
[Dispatcher](https://gist.github.com/Arachnid/4ca9da48d51e23e5cfe0f0e14dd6318f) and Relay. Each
implementation variant introduces a different set of complexity, risks and limitations.

You must be extremely careful with how you store data with this method. If your new contract has a
different storage layout than the first, your data may end up corrupted. When using more complex
implementations of `delegatecall`, you should carefully consider and understand\*:

- How the EVM handles the
  [layout of state variables in storage](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html),
  including packing multiple variables into a single storage slot if possible
- How and why
  [the order of inheritance](https://github.com/OpenZeppelin/openzeppelin-sdk/issues/37) impacts
  the storage layout
- Why the called contract (`LogicContract`) must have the same storage layout of the calling
  contract (`Relay`), and only append new variables to the storage (see
  [Background on delegatecall](https://blog.trailofbits.com/2018/09/05/contract-upgrade-anti-patterns/))
- Why a new version of the called contract (`LogicContract`)
  [must have the same storage layout as the previous version](https://github.com/OpenZeppelin/openzeppelin-sdk/issues/37),
  and only append new variables to the storage
- [How a contract's constructor can affect upgradeability](https://blog.openzeppelin.com/towards-frictionless-upgradeability/)
- How the ABI specifies
  [function selectors](https://solidity.readthedocs.io/en/v0.4.24/abi-spec.html?highlight=signature#function-selector)
  and how
  [function-name collision](https://medium.com/nomic-labs-blog/malicious-backdoors-in-ethereum-proxies-62629adf3357)
  can be used to exploit a contract that uses `delegatecall`
- How `delegatecall` to a non-existent contract will return true even if the called contract does
  not exist. For more details see
  [Breaking the proxy pattern](https://blog.trailofbits.com/2018/09/05/contract-upgrade-anti-patterns/)
  and Solidity docs on
  [Error handling](https://solidity.readthedocs.io/en/latest/control-structures.html#error-handling-assert-require-revert-and-exceptions).
- Remember the
  [importance of immutability to achieve truslessness](https://diligence.consensys.net/blog/2019/01/upgradeability-is-a-bug/)

\* *Extended from
[Proxy pattern recommendations section](https://blog.trailofbits.com/2018/09/05/contract-upgrade-anti-patterns/)*
