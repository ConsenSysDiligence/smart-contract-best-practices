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
