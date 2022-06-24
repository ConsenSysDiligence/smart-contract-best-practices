Speed bumps slow down actions, so that if malicious actions occur, there is time to recover. For
example, [The DAO](https://github.com/slockit/DAO/) required 27 days between a successful request
to split the DAO and the ability to do so. This ensured the funds were kept within the contract,
increasing the likelihood of recovery. In the case of the DAO, there was no effective action that
could be taken during the time given by the speed bump, but in combination with our other
techniques, they can be quite effective.

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
            time: block.timestamp
        });
    }
}

function withdraw() public {
    if(requestedWithdrawals[msg.sender].amount > 0 && block.timestamp > requestedWithdrawals[msg.sender].time + withdrawalWaitPeriod) {
        uint amountToWithdraw = requestedWithdrawals[msg.sender].amount;
        requestedWithdrawals[msg.sender].amount = 0;

        require(msg.sender.send(amountToWithdraw));
    }
}
```
