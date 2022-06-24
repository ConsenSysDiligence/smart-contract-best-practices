Contracts should be deployed with the same compiler version and flags that they have been tested
the most with. Locking the pragma helps ensure that contracts do not accidentally get deployed
using, for example, the latest compiler which may have higher risks of undiscovered bugs. Contracts
may also be deployed by others and the pragma indicates the compiler version intended by the
original authors.

```sol
// bad
pragma solidity ^0.4.4;


// good
pragma solidity 0.4.4;
```

Note: a floating pragma version (ie. `^0.4.25`) will compile fine with `0.4.26-nightly.2018.9.25`,
however nightly builds should never be used to compile code for production.

!!! Warning
    Pragma statements can be allowed to float when a contract is intended for consumption
    by other developers, as in the case with contracts in a library or EthPM package. Otherwise, the
    developer would need to manually update the pragma in order to compile locally.

    See [SWC-103](https://swcregistry.io/docs/SWC-103)
