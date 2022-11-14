
Consider a simple token transfer:

```sol
mapping (address => uint256) public balanceOf;

// INSECURE
function transfer(address _to, uint256 _value) {
    /* Check if sender has balance */
    require(balanceOf[msg.sender] >= _value);
    /* Add and subtract new balances */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}

// SECURE
function transfer(address _to, uint256 _value) {
    /* Check if sender has balance and for overflows */
    require(balanceOf[msg.sender] >= _value && balanceOf[_to] + _value >= balanceOf[_to]);

    /* Add and subtract new balances */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}
```

If a balance reaches the *maximum uint value (2^256)* it will circle back to zero which checks for
the condition. This may or may not be relevant, depending on the implementation. Think about
whether or not the `uint` value has an opportunity to approach such a large number. Think about how
the `uint` variable changes state, and who has authority to make such changes. If any user can call
functions which update the `uint` value, it's more vulnerable to attack. If only an admin has
access to change the variable's state, you might be safe. If a user can increment by only 1 at a
time, you are probably also safe because there is no feasible way to reach this limit.

The same is true for underflow. If a uint is made to be less than zero, it will cause an underflow
and get set to its maximum value.

Be careful with the smaller data-types like uint8, uint16, uint24...etc: they can even more easily
hit their maximum value.

!!! Warning
    Be aware there are around [20 cases for overflow and underflow](https://github.com/ethereum/solidity/issues/796#issuecomment-253578925).

    One simple solution to mitigate the common mistakes for overflow and underflow is to use
    `SafeMath.sol`
    [library](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/ae54e6de1d77c60881e4c85ffbdb7f9d76b71efe/contracts/utils/math/SafeMath.sol)
    for arithmetic functions. Solidity automatically reverts on integer overflow and underflow, as of
    version 0.8.0.

    See [SWC-101](https://swcregistry.io/docs/SWC-101)
