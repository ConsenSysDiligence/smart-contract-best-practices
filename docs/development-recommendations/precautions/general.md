As we discussed in the [General Philosophy](../../general-philosophy/index.md) section, it is not enough to
protect yourself against the known attacks. Since the cost of failure on a blockchain can be very
high, you must also adapt the way you write software, to account for that risk.

The approach we advocate is to "prepare for failure". It is impossible to know in advance whether
your code is secure. However, you can architect your contracts in a way that allows them to fail
gracefully, and with minimal damage. This section presents a variety of techniques that will help
you prepare for failure.

Note: There's always a risk when you add a new component to your system. A badly designed fail-safe
could itself become a vulnerability - as can the interaction between a number of well-designed
fail-safes. Be thoughtful about each technique you use in your contracts, and consider carefully
how they work together to create a robust system.
