---
ERC: XXXX  
Title: Outcome-Based Transaction Simulation & Offchain Asset Access Control  
Author: Jean-Loïc Mugnier (IPSProtocol) <jmugnier@ipsprotocol.org>  
Status: Draft  
Type: Standards Track  
Category: Wallet / UX / Security  
Created: 2025-11-07  
Optional: ERC-20  
License: CC0-1.0  
---

# 1. Abstract

This ERC defines a standard for **Outcome-Based Transaction Simulation** combined with **Offchain Asset Access Control**, providing users with an additional layer of protection against phishing, hidden approvals, and malicious offchain signatures.  

Wallets implementing this standard simulate pending transactions and extract **runtime outcomes** — such as transfers, approvals, or value movements — derived from emitted events. These outcomes are then translated into **plain-language summaries**, allowing users to clearly understand the actual impact of their transaction before signing.

To strengthen visibility and enforce security over transaction outcomes, the ERC introduces a **local policy layer** — a configurable rule set defining per-token and per-action permissions (e.g., locks, per-asset authorizations). After signing, the simulated outcomes are **evaluated against these local policies** to ensure that even if a user misinterprets the simulation or a translation fails to expose an unexpected outcome, the policy engine serves as a **final safety net**, blocking or requiring re-confirmation of unauthorized operations.

# 2. Motivation

A significant portion of wallet-draining and phishing incidents on Ethereum originate from a single systemic weakness: **users cannot easily understand the outcome of their transactions without deep knowledge of protocol implementation details**.  

Malicious dApps and deceptive front-ends exploit this gap to obtain unauthorized token approvals and offchain signature–based permissions. This weakness persists largely because there is no standardized mechanism to expose or verify what a transaction will actually do, leaving users in a state of blind signing.

Typical attack patterns include:
- **“Ice-phishing”** — tricking users into granting unexpected or unbounded ERC-20 allowances that can later be exploited, a risk further amplified by EIP-7702, which allows multiple calls to be aggregated into a single transaction.  
- **Malicious offchain signatures** — e.g., `permit`, `delegation`, or meta-transaction payloads that silently grant asset control to unauthorized parties.

Even with transaction decoding or localized simulation previews, users remain exposed to human error and interface ambiguity.

This ERC addresses that gap by standardizing how **runtime outcomes** are extracted and represented from simulation results, shifting the focus from **which function is called** to **what results it produces**.

Furthermore, by introducing a **localized access-control layer**, wallets can enforce per-asset or per-action restrictions — allowing users to lock or limit critical permissions.  
This dual mechanism of **outcome-based simulation display** and **offchain rule enforcement** provides a verifiable safety net, even when the user misreads or overlooks a malicious outcome.

The goal of this ERC is pragmatic: to mitigate the most prevalent transaction-level attack surfaces **without altering consensus rules or relying on trusted intermediaries**.

# 3. Definitions

**Issuer** —  
The account whose assets or permissions may be affected by the transaction (typically `tx.from`).

**Outcome** —  
A normalized representation of the transaction’s runtime outcome obtained through simulation, including token transfers, approvals, native-ETH movements, etc..

**Policy** —  
A local, per-account, user-defined rule evaluated by the wallet or plugin to determine whether specific actions are permitted, require explicit confirmation, or are blocked by default.

# 4. Canonical Outcome

Wallets implementing this ERC **MUST** simulate the transaction prior to signing and extract a normalized and deterministic set of **Outcomes** from emitted events and value transfers involving the issuer.  

Wallets **SHOULD** use deterministic simulation environments that mirror mainnet state at a specific block height (`blockTag`), ensuring reproducible outcomes.

Implementations **MAY** rely on third-party simulation APIs, provided responses are cryptographically signed or verifiable.

Each Outcome represents a semantically consistent result derived from the transaction’s runtime behavior, forming the canonical representation of its **runtime outcome**.

## 4.1 Data Model

All Outcomes share the same canonical schema:

```solidity
struct Outcome {
  bytes32 kind;        // MUST — event signature hash (topic0)
  address asset;       // MUST — contract emitting the event
  address origin;      // MUST — initiator of the change
  address beneficiary; // MUST — final recipient, spender, or destination
  bytes   data;        // OPTIONAL — ABI-encoded event parameters
}
```

> *EVM-native:* \`kind = keccak256(canonical event signature)\`  
> Example: \`keccak256("Transfer(address,address,uint256)")\`

## 4.2 Determinism and Ordering

The canonical Outcome array is unordered for semantic equivalence,  
but wallets **SHOULD** preserve original emission order for reproducibility.  
Implementations **MAY** group identical Outcomes for display, provided  
grouping does not alter the canonical data used for hashing or verification.

# 5. Offchain Signature Simulation

Wallets **MUST** simulate off-chain signatures (e.g., EIP-2612 `permit`, Permit2, `delegation`) by reproducing their on-chain execution effects in a sandbox environment identical to transaction simulation.  

Extracted Outcomes from these simulations **MUST** follow the same canonical schema.

# 6. Outcome Rendering Layer

Wallets **SHOULD** render all transaction outcomes in human-readable form prior to signature confirmation.  

To achieve consistent interpretation, a **public translation repository** **SHOULD** be maintained by the protocol owner, mapping canonical event signatures to their plain-language templates.  

Each Outcome **SHALL** be rendered using standardized templates that describe the **user-facing result**, rather than the underlying contract function calls.

Implementations **SHOULD** rely on a public, signed repository of translation templates (e.g., hosted on GitHub).  
Each translation entry **MUST** include:
- `eventSignature`
- `templateText`
- `verifierSignature` — signed by the deployer of the contract emitting the event or by a recognized maintainer registry.

Wallets **MUST** display the verified signer identity or checksum to the user when rendering translations.

## 6.1 Display Rules

- All Outcomes **SHOULD** be displayed. Events that do not involve the user (e.g., emissions related solely to third-party interactions) **MAY** be omitted.  
- Events involving **unknown crypto assets** or **events without transalation** **SHOULD** trigger a clear warning.  
- Wallets **MAY** enhance formatting using known metadata (token symbol, name, decimals) or verified naming sources.  
- Localization and UI adaptations **MAY** be applied, provided that numerical values and semantics remain unaltered.

These translation templates establish the canonical bridge between **onchain events** and **user comprehension**, ensuring consistency, predictability, and auditability across wallet implementations.

## 6.2 Non-User-Relevant Events

Events unrelated to the issuer (e.g., protocol-internal accounting or cross-protocol emissions) **MAY** be omitted in future extensions once standardized heuristics exist for user relevance detection.  

In the current version, all events **MUST** be retained.

# 7. Offchain Asset Access Control

To prevent unintended or malicious operations, wallets implementing this ERC **SHOULD** provide a local, user-configurable **access-control layer** that defines permitted states or actions for each asset.

## 7.1 Policy Model

Wallets **MAY** implement either a coarse-grained **asset-level policy** or a fine-grained **action-based policy**, depending on their threat model and UX design.

```solidity
enum AssetState { LOCKED, UNLOCKED }

struct AssetPolicy {
  AssetState state; // Simplified: lock or unlock a specific asset
}
```

## 7.2 Policy Evaluation Lifecycle

Wallets **MUST** evaluate local policies immediately before signature submission.  
If the transaction violates any active policy, the wallet **MUST** either block the signature or display a **Confirm with Override** prompt.

# 8. Security Considerations

- Outcomes **MUST** be derived directly from the calldata corresponding to the pending transaction; 
- Any transaction substitution **MUST** be simulated again.
- Wallets **MUST NOT** allow external scripts to bypass or override local policy evaluation.  
- Users **Could** back up or export their policy configurations securely to prevent accidental loss of protection.  
- Wallets **MUST** maintain consistency between simulation blockTag and signing state to avoid simulation drift.  

This ERC is consensus-neutral and purely client-side. It standardizes the interface and semantics of the final security layer between transaction simulation and signature authorization.

# 9. Rationale and Future Work

This ERC shifts transaction comprehension from **function-level semantics** to **outcome-level semantics**, abstracting away implementation details and focusing instead on the **functional consequences** for the user.  
It ensures that users review **what will actually happen** rather than merely **what is being invoked**.  

By combining **outcome-based simulation** with **user-governed offchain access control**, wallets can prevent the vast majority of phishing, approval-drain, and hidden permission attacks before a transaction is ever broadcast.

## 9.1 Future Research Directions

- **Transaction Fidelity Verification** — Onchain attestation of simulation hashes and execution equivalence between the user intent and transaction outcome at inclusion, providing verifiable proof that execution matches the user’s intent and protecting against advanced phishing, TOCTOU attacks and malicious MEV — effectively enabling true *“What You See Is What You Get”* transaction fidelity.
- **Expanded Outcome Schemas** — Standardized mappings for domain-specific protocols such as lending, staking, and vault operations to broaden semantic coverage.  
- **Distributed Policy Registries** — Shared, attestable repositories of policy configurations for institutional, custodial, or enterprise-grade wallets.  

Collectively, these extensions move toward a verifiable model of **transaction fidelity** — where intent, simulation, and execution can be cryptographically aligned while preserving the decentralized and client-sovereign nature of Ethereum.

# 10. Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
