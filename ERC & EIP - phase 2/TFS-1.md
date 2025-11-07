# TFS-1 — Transaction Fidelity Standard

**Components**
- **ERC-XXXX – Intent (Wallet Layer):** user-side declaration of expected outcomes.  
- **EIP-YYYY – Intent Verification (Client Layer):** execution-layer validation of those outcomes.

---

## 1. Scope & Non-Goals

**In scope**
- Definition of the intent abstraction and its canonical representation (EIP-712).  
- Wallet-layer display rules and user-defined slippage semantics.  
- Mapping model linking a transaction (`tx`) to its signed `Intent`.  
- Cross-layer validation principle ensuring that execution effects match the declared intent.  
- Alignment of terminology and interoperability between wallet and client layers.

**Out of scope**
- Solver- or Account-Abstraction–based intent generalization (reserved for **TFS-2**).  
- Economic incentives, staking, or slashing mechanisms.  
- PBS or builder integration specifics.  

---

## 2. Threat Model

- **TOCTOU gap** — divergence between simulated and executed state.  
  Includes *transaction simulation spoofing*, *malicious MEV* (frontrunning, sandwiching), and other state-dependent race conditions.  

- **Market evolution / benign drift** — natural variation of prices or pool ratios between simulation and inclusion; mitigated via user-defined tolerances.  

- **Predictability and auditability** — without a verifiable mapping between user intent and execution, users and auditors cannot prove outcome fidelity.  
  TFS-1 establishes this linkage, enabling decentralized transparency and ex-post verification of transaction correctness.

---

## 3. Intent Abstraction (ERC-XXXX)

The canonical **Intent** is an EIP-712 typed data structure representing the expected effects of a transaction.  
It is declared, displayed, and signed by the wallet before submission.

ERC-XXXX defines:
- The **field schema** and canonical order of the `Intent` structure.  
- How outcomes (effects) such as transfers or approvals are encoded.  
- How tolerances (`slippagePpm`) express acceptable deviation.  
- How wallets must display all declared outcomes before signing.

For detailed schema and field definitions, see [`./erc-intent.md`](./erc-intent.md).

---

## 4. Intent Verification Principles (EIP-YYYY)

Validation is performed at the **execution layer**.  
EIP-YYYY specifies how clients or verifiers confirm that a transaction’s observed effects match its signed `Intent`.

Core principles:
- Verification occurs pre-inclusion or within a simulation context.  
- Each declared outcome is compared against emitted on-chain effects.  
- Deviations beyond the declared tolerance cause the transaction to be **rejected**.  
- Verification outputs are transparent and reproducible for public audit.

For detailed client-level validation semantics, see [`./eip-intent-verification.md`](./eip-intent-verification.md).

---

## 5. Cross-Layer Mapping Model

A transaction and its signed intent are linked via a deterministic mapping:
```
(issuer, issuerNonce) ↔ (tx.from, tx.nonce)
```

- For EOAs: `issuer = tx.from`.  
- For delegated (EIP-7702) transactions: `issuer` corresponds to the logical sender defined in the delegation.  
- Each `(issuer, issuerNonce)` pair binds to a single canonical transaction, ensuring replay-safety and fidelity traceability across layers.

---

## 6. Conformance Matrix

| Actor | MUST for ERC | MUST for EIP |
|:--|:--|:--|
| **Wallet** | Build & sign `Intent`; display outcomes and tolerances | — |
| **Client / Verifier** | — | Enforce mapping and equivalence rules |
| **Auditor / Indexer** | Record and hash domain/message for mapping | Recompute effects for verification |

---

## 7. Versioning & Profiles

- **v1:** EOAs and delegated EOAs (EIP-7702).  
- **v2:** Account-Abstraction and solver-generated intents.  
Field order within all typed structures is normative.  
Breaking schema changes MUST increment the `version`.

---

## 8. Security & UX Principles

- Explicit `beneficiary` eliminates major phishing and redirection risks (eg. malicious approvals)
- `slippagePpm = 0` denotes exact operations (mints, burns, static transfers).
