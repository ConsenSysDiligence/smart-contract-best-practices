Be aware of the tradeoffs between **abstract contracts** and **interfaces**.

Both interfaces and abstract contracts provide one with a customizable and re-usable approach for
smart contracts. Interfaces, which were introduced in Solidity 0.4.11, are similar to abstract
contracts but cannot have any functions implemented. Interfaces also have limitations such as not
being able to access storage or inherit from other interfaces which generally makes abstract
contracts more practical. Although, interfaces are certainly useful for designing contracts prior
to implementation. Additionally, it is important to keep in mind that if a contract inherits from
an abstract contract it must implement all non-implemented functions via overriding or it will be
abstract as well.
