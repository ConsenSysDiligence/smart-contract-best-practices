When utilizing multiple inheritance in Solidity, it is important to understand how the compiler
composes the inheritance graph.

```sol

contract Final {
    uint public a;
    function Final(uint f) public {
        a = f;
    }
}

contract B is Final {
    int public fee;

    function B(uint f) Final(f) public {
    }
    function setFee() public {
        fee = 3;
    }
}

contract C is Final {
    int public fee;

    function C(uint f) Final(f) public {
    }
    function setFee() public {
        fee = 5;
    }
}

contract A is B, C {
  function A() public B(3) C(5) {
      setFee();
  }
}
```

When a contract is deployed, the compiler will *linearize* the inheritance from right to left
(after the keyword _is_ the parents are listed from the most base-like to the most derived). Here
is contract A's linearization:

**Final \<- B \<- C \<- A**

The consequence of the linearization will yield a `fee` value of 5, since C is the most derived
contract. This may seem obvious, but imagine scenarios where C is able to shadow crucial functions,
reorder boolean clauses, and cause the developer to write exploitable contracts. Static analysis
currently does not raise issue with overshadowed functions, so it must be manually inspected.

For more on security and inheritance, check out this
[article](https://pdaian.com/blog/solidity-anti-patterns-fun-with-inheritance-dag-abuse/)

To help contribute, Solidity's Github has a
[project](https://github.com/ethereum/solidity/projects/9#card-8027020) with all
inheritance-related issues.

See [SWC-125](https://swcregistry.io/docs/SWC-125)
