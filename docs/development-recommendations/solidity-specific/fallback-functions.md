!!! tip

    For comprehensive insights into secure development practices, consider visiting the [Development Recommendations](https://scsfg.io/developers/) section of the Smart Contract Security Field Guide. This resource provides in-depth articles to guide you in developing robust and secure smart contracts.

#### Keep fallback functions simple

[Fallback functions](http://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) are
called when a contract is sent a message with no arguments (or when no function matches), and only
has access to 2,300 gas when called from a `.send()` or `.transfer()`. If you wish to be able to
receive Ether from a `.send()` or `.transfer()`, the most you can do in a fallback function is log
an event. Use a proper function if a computation of more gas is required.

```sol
// bad
function() payable { balances[msg.sender] += msg.value; }

// good
function deposit() payable external { balances[msg.sender] += msg.value; }

function() payable { require(msg.data.length == 0); emit LogDepositReceived(msg.sender); }
```

#### Check data length in fallback functions

Since the
[fallback functions](http://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) is
not only called for plain ether transfers (without data) but also when no other function matches,
you should check that the data is empty if the fallback function is intended to be used only for
the purpose of logging received Ether. Otherwise, callers will not notice if your contract is used
incorrectly and functions that do not exist are called.

```sol
// bad
function() payable { emit LogDepositReceived(msg.sender); }

// good
function() payable { require(msg.data.length == 0); emit LogDepositReceived(msg.sender); }
```
Solidity now supports a [receive fallback](https://docs.soliditylang.org/en/v0.8.14/contracts.html#receive-ether-function) 
function, which when declared in your code you wouldn't need to worry about making your main fallback function payable.
Receive fallback function will be triggered when Ether is forcefully sent to the contract.<br/>
It can be declared by:
```
receive() external payable { ... }
```

There is now also an explicit [fallback function](https://docs.soliditylang.org/en/v0.8.14/contracts.html#fallback-function), 
that replaces the normal fallback function.
This gets triggered when a function call does not match any of the function signatures declared in a contract.
This fallback function can also be assigned payable to accept Ether.<br/>
It can be declared by:
```
fallback() external { ... }
```






