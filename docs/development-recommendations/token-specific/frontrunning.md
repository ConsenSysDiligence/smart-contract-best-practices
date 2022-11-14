The EIP-20 token's `approve()` function creates the potential for an approved spender to spend more
than the intended amount. A
[front running attack](../../attacks/frontrunning.md) can be
used, enabling an approved spender to call `transferFrom()` both before and after the call to
`approve()` is processed. More details are available on the
[EIP](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md#approve), and in
[this document](https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit).
