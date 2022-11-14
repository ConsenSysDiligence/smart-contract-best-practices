Some tips for running bounty programs:

- Decide which currency bounties will be distributed in (BTC and/or ETH)
- Decide on an estimated total budget for bounty rewards
- From the budget, determine three tiers of rewards:
  - smallest reward you are willing to give out
  - highest reward that's usually awardable
  - an extra range to be awarded in case of very severe vulnerabilities
- Determine who the bounty judges are (3 may be ideal typically)
- Lead developer should probably be one of the bounty judges
- When a bug report is received, the lead developer, with advice from judges, should evaluate the
  severity of the bug
- Work at this stage should be in a private repo, and the issue filed on Github
- If it's a bug that should be fixed, in the private repo, a developer should write a test case,
  which should fail and thus confirm the bug
- Developer should implement the fix and ensure the test now passes; writing additional tests as
  needed
- Show the bounty hunter the fix; merge the fix back to the public repo is one way
- Determine if bounty hunter has any other feedback about the fix
- Bounty judges determine the size of the reward, based on their evaluation of both the
  *likelihood* and *impact* of the bug.
- Keep bounty participants informed throughout the process, and then strive to avoid delays in
  sending them their reward

For an example of the three tiers of rewards, see
[Ethereum's Bounty Program](https://bounty.ethereum.org):

> The value of rewards paid out will vary depending on severity of impact. Rewards for minor
> 'harmless' bugs start at 0.05 BTC. Major bugs, for example leading to consensus issues, will be
> rewarded up to 5 BTC. Much higher rewards are possible (up to 25 BTC) in case of very severe
> vulnerabilities.
