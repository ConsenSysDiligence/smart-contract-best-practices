Explicitly label the visibility of functions and state variables. Functions can be specified as
being `external`, `public`, `internal` or `private`. Please understand the differences between
them, for example, `external` may be sufficient instead of `public`. For state variables,
`external` is not possible. Labeling the visibility explicitly will make it easier to catch
incorrect assumptions about who can call the function or access the variable.

- `External` functions are part of the contract interface. An external function `f` cannot be
  called internally (i.e. `f()` does not work, but `this.f()` works). External functions are
  sometimes more efficient when they receive large arrays of data.
- `Public` functions are part of the contract interface and can be either called internally or via
  messages. For public state variables, an automatic getter function (see below) is generated.
- `Internal` functions and state variables can only be accessed internally, without using `this`.
- `Private` functions and state variables are only visible for the contract they are defined in and
  not in derived contracts. **Note**: Everything that is inside a contract is visible to all
  observers external to the blockchain, even `Private`
  variables.<sup><a href='https://solidity.readthedocs.io/en/develop/contracts.html?#visibility-and-getters'>\*</a></sup>

```sol
// bad
uint x; // the default is internal for state variables, but it should be made explicit
function buy() { // the default is public
    // public code
}

// good
uint private y;
function buy() external {
    // only callable externally or using this.buy()
}

function utility() public {
    // callable externally, as well as internally: changing this code requires thinking about both cases.
}

function internalAction() internal {
    // internal code
}
```

See [SWC-100](https://swcregistry.io/docs/SWC-100) and
[SWC-108](https://swcregistry.io/docs/SWC-108)
