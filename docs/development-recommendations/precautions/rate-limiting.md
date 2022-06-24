Rate limiting halts or requires approval for substantial changes. For example, a depositor may only
be allowed to withdraw a certain amount or percentage of total deposits over a certain time period
(e.g., max 100 ether over 1 day) - additional withdrawals in that time period may fail or require
some sort of special approval. Or the rate limit could be at the contract level, with only a
certain amount of tokens issued by the contract over a time period.

Example:

```sol
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
