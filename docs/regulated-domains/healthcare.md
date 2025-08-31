# Healthcare and Regulated-Data Considerations

**Goal:** help engineers avoid putting regulated data on-chain and ship auditable controls for health and similar regulated workflows.

## Do not put regulated data on-chain
- Treat all L1/L2 calldata and logs as public and permanent. Never store PHI/PII, medical record numbers, claim IDs, diagnosis codes, or free-text notes on-chain.
- Ban human-entered identifiers in transaction memos and events.
- Use off-chain encrypted storage with access control; anchor only commitments or hashes where strictly necessary.

## Prefer privacy-preserving patterns
- Commit–reveal, Merkle proofs, nullifiers, or other ZK patterns to prove authorization without revealing patient data.
- Verifiable Credentials (VCs) and Decentralized Identifiers (DIDs) for off-chain consent and role proofs; on-chain checks verify proofs or commitments, not raw attributes.

## Events and logging hygiene
- Emit non-identifying codes only. No dates of service, names, addresses, or rare condition flags.
- Scrub revert strings and error messages of sensitive context.
- Hashes are not anonymous if source data is small or structured. Salt where possible and avoid reversible encodings.

## Consent and authorization flows
- Use EIP-712 typed data for consent/authorization messages with strict domain separation and correct chainId, verifyingContract, and purpose-specific primaryType.
- Ensure nonces and expirations; reject replay across chains and contracts.

## Access control and key management
- Principle of least privilege. Separate signer roles for deploy, upgrade, pause, and data-adjacent actions.
- Rotate keys, document custodianship, and define break-glass procedures with bounded scope and post-incident review.

## Protocol threats with health impact
- Price oracle manipulation and front-running can cascade into harmful outcomes if tied to benefits or coverage; treat them as safety issues, not only financial risk.
- Denial-of-service on consent or credential verification endpoints is a clinical safety risk; budget gas and include retries and circuit-breakers.

## Minimal pre-audit checklist
- [ ] No PHI/PII in storage, calldata, or events; unit tests assert absence.
- [ ] EIP-712 domains use correct chainId and verifyingContract; nonces and expirations enforced.
- [ ] No tx.origin auth; role-based access only.
- [ ] All external calls checked; reentrancy protections around state-changing flows that touch regulated workflows.
- [ ] Break-glass functions gated, timelocked, and monitored.
- [ ] Off-chain stores encrypted, access-logged, and backed up; on-chain holds only commitments.
- [ ] Incident hooks: on detect → pause bounded scope, rotate keys, notify off-chain handlers.

## Slither/Mythril test ideas
- Slither: detect external call patterns around consent updates; flag reentrancy and missing checks.
- Mythril: simulate replay without nonce/expiry on EIP-712 messages; assert failures.

## References
- ConsenSys Diligence Smart Contract Best Practices
- NIST Privacy Framework; NIST SP 800-53; NIST SP 800-122
- HIPAA Security Rule (45 CFR 164 Subpart C)
- HL7 FHIR Security; SMART on FHIR
- W3C VC and DID Core; OIDC4VC / OIDC4VP
