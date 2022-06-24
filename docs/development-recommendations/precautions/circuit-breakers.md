Circuit breakers stop execution if certain conditions are met, and can be useful when new errors
are discovered. For example, most actions may be suspended in a contract if a bug is discovered,
and the only action now active is a withdrawal. You can either give certain trusted parties the
ability to trigger the circuit breaker or else have programmatic rules that automatically trigger
the certain breaker when certain conditions are met.

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
