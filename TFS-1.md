# TFS-1 — Transaction Fidelity Standard (WIP)

**Components**
- **ERC-XXXX – Intent (Wallet Layer):** user-side declaration of expected outcomes.  
- **EIP-YYYY – Intent Verification (Client Layer):** execution-layer validation of those outcomes.

---

## 1. Motivation

### 1.1 Transaction Fidelity

The goal of Transaction Fidelity is to guarantee that a user **only gets what they asked for**, and that **nothing else happens** during execution — following the principle of *What You See Is What You Get (WYSIWYG)*.

TFS-1 establishes mechanisms ensuring that a transaction cannot produce effects beyond those explicitly declared, authorized, and bounded by the user through their signed Intent.


### 1.2 Transaction Contractualization

TFS-1 introduces **Transaction Contractualization**:  
a formal contract between the **user** and the **chain** describing *what effects the user accepts*, and *what deviations invalidate the authorization*.

This contract includes:

- a **canonical and deterministic representation** of expected effects (Intent),  
- **user-defined acceptance criteria** (slippage, tolerances, restrictions).

The chain (or verifier acting on behalf of the chain) MUST evaluate whether a proposed execution **respects this contract**.  
Only executions whose *observed effects* fall within the user-approved constraints are considered valid.

> Transaction Contractualization is the prerequisite for **Transaction Fidelity**, **predictability of outcomes**, and full **auditability**, without introducing undescriminated censorship.

---

## 2. Scope

### In scope
- The **definition of Intent**, an EIP-712 typed data structure.  
- Wallet-layer display rules ensuring users understand the declared effects.  
- A **cross-layer mapping model** binding a signed Intent to its execution attempt.  
- The **equivalence principle** ensuring executed effects match the declared Intent.  
- Compatibility with **all execution formats**:
  - raw EVM transactions  
  - or any off-chain message later materialized on-chain  
    (EIP-712 meta-transactions, ERC-4337 UserOperations, EIP-7702 delegated EOAs)
- A unified semantics of **effect-level comparison** across execution types.

### Out of scope
- PBS impacts.

---

## 3. Threat Model

- **TOCTOU gap** — divergence between simulated and executed state  
  (simulation spoofing, malicious MEV, frontrunning, sandwiching, state-dependent races).

- **Market evolution / benign drift** — natural variation in prices or pool ratios; mitigated via user-defined tolerances.

- **Predictability & auditability** — without a verifiable mapping between Intent and execution, the chain cannot prove outcome fidelity.  
  TFS-1 provides this linkage.

- **Intent authenticity** — an execution request claims to reference an Intent not actually signed by the user.  
  TFS-1 prevents this via EIP-712 signing by the issuer.

- **Intent integrity** — modification, redaction, or reordering of an Intent.  
  TFS-1 enforces canonical hashing and strict field ordering.

- **Intent replay attacks** — reusing a valid Intent in a different execution context.  
  TFS-1 binds Intents via `(issuer, issuerNonce)`, chain ID, and `deadline` to ensure single-use validity.

---

## 4. Intent Abstraction (ERC-XXXX)

The **Intent** is an EIP-712 typed structure defining the **expected effects** of an execution request.  
It serves as the canonical, user-authored declaration of acceptable outcomes.

An Intent contains:
- a set of **outcomes** (effects the user authorizes),
- **tolerances and restrictions**, fully **configured by the user**,
- **validity windows** (deadline),
- **issuer identity and nonce** for uniqueness and replay protection,
- and the EIP-712 domain, ensuring **authenticity** and deterministic representation.

### 4.1 User-defined configuration
Tolerances, restrictions, and acceptance criteria MUST be shown to the user, who MUST explicitly approve or modify them prior to signing.  
Wallets MUST provide UI capable of capturing these inputs.

### 4.2 Intent Security
Intents MUST be signed as EIP-712 typed data to guarantee:
- **Authenticity** — user actually signed the Intent  
- **Integrity** — no modifications  
- **Replay protection** — scoped via `(issuer, issuerNonce, chainId, deadline)`  
- **Deterministic representation** — consistent across wallets and verifiers.

### 4.3 Execution model compatibility
Intents are compatible with:
- direct EOA transactions, or  
- any off-chain message later submitted as an execution request.

Across all models, the Intent is the authoritative expression of user-approved effects and constraints.

---

## 5. Intent Verification Principles (EIP-YYYY)

Verification occurs at the **execution layer**.  
Clients or verifiers MUST confirm that a transaction's observed effects match its signed Intent.

Core principles:
- Verification occurs **pre-inclusion** (local simulation) or **onchain**.  
- Each declared outcome is compared against normalized on-chain effects.  
- Deviations beyond tolerance thresholds cause **rejection**.  
- Outputs MUST be reproducible and suitable for public audit.

---

## 6. Cross-Layer Mapping Model

A transaction and its Intent are linked via a deterministic mapping:

```
(issuer, issuerNonce) ↔ (tx.from, tx.nonce)
```


- EOAs: `issuer = tx.from`  
- EIP-7702: `issuer = logical sender` from delegation  

This guarantees replay-safety and traceability.

---

## 7. New Transaction Type — Typed Transaction Envelope (EIP-2718)

TFS-1 uses EIP-712 for Intent formatting, but requires that a **transaction and its Intent be treated as one atomic object**.

A new typed transaction envelope under **EIP-2718** ensures:

- a transaction cannot be executed **without** its Intent,  
- an Intent cannot be **replaced, stripped, or modified**,  
- verifiers always receive the exact **transaction + intent pair** needed for validation.

This mechanism guarantees secure Intent-bound execution while preserving compatibility with EOAs, smart accounts, UserOperations, and delegated EOAs.

Wherever verification occurs (client-side or onchain), the envelope ensures access to both:
1. the transaction to be executed, and  
2. the full canonical Intent with all user-defined restrictions.

---

## 8. Conformance Matrix

| Actor | MUST for ERC | MUST for EIP |
|:--|:--|:--|
| **Wallet** | Build & sign Intent + restrictions; display outcomes & tolerances | — |
| **Client / Verifier** | — | Enforce mapping & equivalence rules |
| **Auditor / Indexer** | Record & hash Intent domain/message | Recompute effects for verification |

---

## 9. Versioning & Profiles

- **v1:** EOAs and delegated EOAs (EIP-7702).  
- **v2:** AA flows & solver-generated intents.  

Field ordering within all typed structures is normative.  
Any breaking schema change MUST increment the version.

---

## 10. Security & UX Principles (TBD)

- Explicit `beneficiary` prevents redirection attacks (e.g., malicious approvals).  
- `slippagePpm = 0` denotes exact operations (mints, burns, static transfers).  
