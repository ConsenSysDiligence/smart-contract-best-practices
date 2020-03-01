As we discussed in the [General Philosophy](/general_philosophy/) section, it is not enough to protect yourself against the known attacks. Since the cost of failure on a blockchain can be very high, you must also adapt the way you write software, to account for that risk.

The approach we advocate is to "prepare for failure". It is impossible to know in advance whether your code is secure. However, you can architect your contracts in a way that allows them to fail gracefully, and with minimal damage. This section presents a variety of techniques that will help you prepare for failure.

Note: There's always a risk when you add a new component to your system. A badly designed fail-safe could itself become a vulnerability - as can the interaction between a number of well designed fail-safes. Be thoughtful about each technique you use in your contracts, and consider carefully how they work together to create a robust system.

### Upgrading Broken Contracts

!!! warning
    Smart Contract upgradeability is an active area of research. There are many important questions, and risks related to smart contract upgradeability. Do your research into the state of the art. We welcome discussion on the [related issue](https://github.com/ConsenSys/smart-contract-best-practices/issues/164).


Code will need to be changed if errors are discovered or if improvements need to be made. It is no good to discover a bug, but have no way to deal with it.

Designing an effective upgrade system for smart contracts is an area of active research, and we won't be able to cover all of the complications in this document. However, there are two basic approaches that are most commonly used. The simpler of the two is to have a registry contract that holds the address of the latest version of the contract. A more seamless approach for contract users is to have a contract that forwards calls and data onto the latest version of the contract.

Whatever the technique, it's important to have modularization and good separation between components, so that code changes do not break functionality, orphan data, or require substantial costs to port. In particular, it is usually beneficial to separate complex logic from your data storage, so that you do not have to recreate all of the data in order to change the functionality.

It's also critical to have a secure way for parties to decide to upgrade the code. Depending on your contract, code changes may need to be approved by a single trusted party, a group of members, or a vote of the full set of stakeholders. If this process can take some time, you will want to consider if there are other ways to react more quickly in case of an attack, such as an [emergency stop or circuit-breaker](#circuit-breakers-pause-contract-functionality).

Regardless of your approach, it is important to have some way to upgrade your contracts, or they will become unusable when the inevitable bugs are discovered in them.


#### Example 1: Use a registry contract to store latest version of a contract

In this example, the calls aren't forwarded, so users should fetch the current address each time before interacting with it.

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

1. Users must always look up the current address, and anyone who fails to do so risks using an old version of the contract
2. You will need to think carefully about how to deal with the contract data when you replace the contract

The alternate approach is to have a contract forward calls and data to the latest version of the contract:

#### Example 2: [Use a `DELEGATECALL`](http://ethereum.stackexchange.com/questions/2404/upgradeable-contracts) to forward data and calls

This approach relies on using the [fallback function](https://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) (in `Relay` contract) to forward the calls to a target contract (`LogicContract`) using [delegatecall](https://solidity.readthedocs.io/en/latest/introduction-to-smart-contracts.html#delegatecall-callcode-and-libraries). Remember that `delegatecall` is a special function in Solidity that executes the logic of the called address (`LogicContract`) in the context of the calling contract (`Relay`), so *"storage, current address and balance still refer to the calling contract , only the code is taken from the called address"*.

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

This simple version of the pattern cannot return values from `LogicContract`'s functions, only forward them, which limits its applicability. More complex implementations attempt to solve this with in-line assembly code and a registry of return sizes. They are commonly referred to as [Proxy Patterns](https://blog.openzeppelin.com/proxy-patterns/), but are also known as [Router](https://github.com/PeterBorah/ether-router), [Dispatcher](https://gist.github.com/Arachnid/4ca9da48d51e23e5cfe0f0e14dd6318f) and Relay. Each implementation variant introduces a different set of complexity, risks and limitations.

You must be extremely careful with how you store data with this method. If your new contract has a different storage layout than the first, your data may end up corrupted. When using more complex implementations of `delegatecall`, you should carefully consider and understand\*:

- How the EVM handles the [layout of state variables in storage](https://solidity.readthedocs.io/en/latest/miscellaneous.html#layout-of-state-variables-in-storage), including packing multiple variables into a single storage slot if possible
- How and why [the order of inheritance](https://github.com/OpenZeppelin/openzeppelin-sdk/issues/37) impacts the storage layout
- Why the called contract (`LogicContract`) must have the same storage layout of the calling contract (`Relay`), and only append new variables to the storage (see [Background on delegatecall](https://blog.trailofbits.com/2018/09/05/contract-upgrade-anti-patterns/))
- Why a new version of the called contract (`LogicContract`) [must have the same storage layout as the previous version](https://github.com/OpenZeppelin/openzeppelin-sdk/issues/37), and only append new variables to the storage
- [How a contract's constructor can affect upgradeability](https://blog.openzeppelin.com/towards-frictionless-upgradeability/)
- How the ABI specifies [function selectors](https://solidity.readthedocs.io/en/v0.4.24/abi-spec.html?highlight=signature#function-selector) and how [function-name collision](https://medium.com/nomic-labs-blog/malicious-backdoors-in-ethereum-proxies-62629adf3357) can be used to exploit a contract that uses `delegatecall`
- How `delegatecall` to a non-existent contract will return true even if the called contract does no exist. For more details see [Breaking the proxy pattern](https://blog.trailofbits.com/2018/09/05/contract-upgrade-anti-patterns/) and Solidity docs on [Error handling](https://solidity.readthedocs.io/en/latest/control-structures.html#error-handling-assert-require-revert-and-exceptions).
- Remember the [importance of immutability to achieve truslessness](https://diligence.consensys.net/blog/2019/01/upgradeability-is-a-bug/)

\* *Extended from [Proxy pattern recommendations section](https://blog.trailofbits.com/2018/09/05/contract-upgrade-anti-patterns/)*

### Circuit Breakers (Pause contract functionality)

Circuit breakers stop execution if certain conditions are met, and can be useful when new errors are discovered. For example, most actions may be suspended in a contract if a bug is discovered, and the only action now active is a withdrawal. You can either give certain trusted parties the ability to trigger the circuit breaker or else have programmatic rules that automatically trigger the certain breaker when certain conditions are met.

Example:

```sol
bool private stopped = false;
address private owner;

modifier isAdmin() {
    require(msg.sender == owner);
    _;
}

function toggleContractActive() isAdmin public {
    // You can add an additional modifier that restricts stopping a contract to be based on another action, such as a vote of users
    stopped = !stopped;
}

modifier stopInEmergency { if (!stopped) _; }
modifier onlyInEmergency { if (stopped) _; }

function deposit() stopInEmergency public {
    // some code
}

function withdraw() onlyInEmergency public {
    // some code
}
```

### Speed Bumps (Delay contract actions)

Speed bumps slow down actions, so that if malicious actions occur, there is time to recover. For example, [The DAO](https://github.com/slockit/DAO/) required 27 days between a successful request to split the DAO and the ability to do so. This ensured the funds were kept within the contract, increasing the likelihood of recovery. In the case of the DAO, there was no effective action that could be taken during the time given by the speed bump, but in combination with our other techniques, they can be quite effective.

Example:

```sol
struct RequestedWithdrawal {
    uint amount;
    uint time;
}

mapping (address => uint) private balances;
mapping (address => RequestedWithdrawal) private requestedWithdrawals;
uint constant withdrawalWaitPeriod = 28 days; // 4 weeks

function requestWithdrawal() public {
    if (balances[msg.sender] > 0) {
        uint amountToWithdraw = balances[msg.sender];
        balances[msg.sender] = 0; // for simplicity, we withdraw everything;
        // presumably, the deposit function prevents new deposits when withdrawals are in progress

        requestedWithdrawals[msg.sender] = RequestedWithdrawal({
            amount: amountToWithdraw,
            time: now
        });
    }
}

function withdraw() public {
    if(requestedWithdrawals[msg.sender].amount > 0 && now > requestedWithdrawals[msg.sender].time + withdrawalWaitPeriod) {
        uint amountToWithdraw = requestedWithdrawals[msg.sender].amount;
        requestedWithdrawals[msg.sender].amount = 0;

        require(msg.sender.send(amountToWithdraw));
    }
}
```

### Rate Limiting

Rate limiting halts or requires approval for substantial changes. For example, a depositor may only be allowed to withdraw a certain amount or percentage of total deposits over a certain time period (e.g., max 100 ether over 1 day) - additional withdrawals in that time period may fail or require some sort of special approval. Or the rate limit could be at the contract level, with only a certain amount of tokens issued by the contract over a time period.

Example:

```
uint internal period; // how many blocks before limit resets
uint internal limit; // max ether to withdraw per period
uint internal currentPeriodEnd; // block which the current period ends at
uint internal currentPeriodAmount; // amount already withdrawn this period

constructor(uint _period, uint _limit) public {
    period = _period;
    limit = _limit;
    
    currentPeriodEnd = block.number + period;
}
    
function withdraw(uint amount) public {
    // Update period before proceeding
    updatePeriod();
    
    // Prevent overflow
    uint totalAmount = currentPeriodAmount + amount;
    require(totalAmount >= currentPeriodAmount, 'overflow');
    
    // Disallow withdraws that exceed current rate limit
    require(currentPeriodAmount + amount < limit, 'exceeds period limit');
    currentPeriodAmount += amount;
    msg.sender.transfer(amount);
}

function updatePeriod() internal {
    if(currentPeriodEnd < block.number) {
        currentPeriodEnd = block.number + period;
        currentPeriodAmount = 0;
    }
}
```

### Contract Rollout

Contracts should have a substantial and prolonged testing period - before substantial money is put at risk.

At minimum, you should:

- Have a full test suite with 100% test coverage (or close to it)
- Deploy on your own testnet
- Deploy on the public testnet with substantial testing and bug bounties
- Exhaustive testing should allow various players to interact with the contract at volume
- Deploy on the mainnet in beta, with limits to the amount at risk

##### Automatic Deprecation

During testing, you can force an automatic deprecation by preventing any actions, after a certain time period. For example, an alpha contract may work for several weeks and then automatically shut down all actions, except for the final withdrawal.

```sol
modifier isActive() {
    require(block.number <= SOME_BLOCK_NUMBER);
    _;
}

function deposit() public isActive {
    // some code
}

function withdraw() public {
    // some code
}

```
##### Restrict amount of Ether per user/contract

In the early stages, you can restrict the amount of Ether for any user (or for the entire contract) - reducing the risk.

### Bug Bounty Programs

Some tips for running bounty programs:

- Decide which currency bounties will be distributed in (BTC and/or ETH)
- Decide on an estimated total budget for bounty rewards
- From the budget, determine three tiers of rewards:
    - smallest reward you are willing to give out
    - highest reward that's usually awardable
    - an extra range to be awarded in case of very severe vulnerabilities
- Determine who the bounty judges are (3 may be ideal typically)
- Lead developer should probably be one of the bounty judges
- When a bug report is received, the lead developer, with advice from judges, should evaluate the severity of the bug
- Work at this stage should be in a private repo, and the issue filed on Github
- If it's a bug that should be fixed, in the private repo, a developer should write a test case, which should fail and thus confirm the bug
- Developer should implement the fix and ensure the test now passes; writing additional tests as needed
- Show the bounty hunter the fix; merge the fix back to the public repo is one way
- Determine if bounty hunter has any other feedback about the fix
- Bounty judges determine the size of the reward, based on their evaluation of both the *likelihood* and *impact* of the bug.
- Keep bounty participants informed throughout the process, and then strive to avoid delays in sending them their reward

For an example of the three tiers of rewards, see [Ethereum's Bounty Program](https://bounty.ethereum.org):

> The value of rewards paid out will vary depending on severity of impact. Rewards for minor 'harmless' bugs start at 0.05 BTC. Major bugs, for example leading to consensus issues, will be rewarded up to 5 BTC. Much higher rewards are possible (up to 25 BTC) in case of very severe vulnerabilities.
