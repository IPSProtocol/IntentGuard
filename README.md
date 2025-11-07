# IntentGuard

**A phased approach to achieving Transaction Fidelity and protecting users from malicious transactions, phishing attacks, and MEV exploitation.**

---

## Overview

IntentGuard presents a two-phase solution to one of blockchain's most fundamental user experience and security challenges: ensuring that **what users see before signing a transaction is what actually executes on-chain** — the *"What You See Is What You Get" (WYSIWYG)* principle for blockchain transactions.

The protocol addresses critical vulnerabilities including:
- **Transaction Simulation Spoofing (TSP)** — where malicious dApps show users one outcome during simulation but execute something entirely different
- **Time-of-Check to Time-of-Use (TOCTOU)** attacks — exploiting the gap between transaction preview and execution
- **Malicious MEV** — including sandwich attacks and frontrunning that harm users
- **Phishing and hidden approvals** — tricking users into granting unauthorized token access

---

## The Two-Phase Approach

### Phase 1: Quick Wins for Wallet UX 🛡️

**Wallet-Level Protection**

Phase 1 provides **immediate, implementable security improvements** that wallets can deploy today to protect users from the most common attack vectors, without requiring any consensus changes or ecosystem-wide coordination.

📄 **Specification**: [`ERC - phase 1/erc-outcome-based-simulation.md`](ERC%20-%20phase%201/erc-outcome-based-simulation.md)

#### What It Does

- **Outcome-Based Simulation**: Instead of showing users cryptic function calls, wallets simulate transactions and extract **plain-language summaries** of what will actually happen (e.g., "Transfer 100 USDC to 0x123...")
- **Local Policy Engine**: Users can lock specific assets or define per-token access rules that act as a safety net, blocking unauthorized operations even if the user misreads the simulation
- **Offchain Signature Protection**: Extends the same protections to off-chain signatures like EIP-2612 `permit`, preventing "blind signing" of malicious delegation requests

#### Key Benefits

✅ **No protocol changes required** — pure wallet-side implementation  
✅ **Immediate deployment** — wallets can implement today  
✅ **Stops 90%+ of phishing attacks** — ice-phishing, hidden approvals, malicious permits  
✅ **Dramatically improved UX** — users see outcomes, not function calls  
✅ **Asset-level control** — users can lock high-value tokens from any interaction  

#### Limitations

⚠️ Phase 1 operates **before transaction submission** and cannot prevent:
- Manipulation after the transaction leaves the wallet
- TOCTOU exploits where contract state changes between simulation and execution
- Malicious MEV attacks (sandwiching, frontrunning)
- Transaction reordering by builders/validators

---

### Phase 2: Long-Term Transaction Fidelity Solution 🔒

**Protocol-Level Enforcement (Complete Specification)**

Phase 2 presents the **comprehensive, long-term solution** that achieves true *transaction fidelity* — cryptographically verifiable proof that execution matches user intent. This requires ecosystem coordination but provides complete protection.

📄 **Specifications**:
- [`ERC & EIP - phase 2/erc-intent.md`](ERC%20%26%20EIP%20-%20phase%202/erc-intent.md) — Intent standard (ERC-XXXX)
- [`ERC & EIP - phase 2/eip-intent-verification.md`](ERC%20%26%20EIP%20-%20phase%202/eip-intent-verification.md) — Verification layer (EIP-YYYY)
- [`ERC & EIP - phase 2/TFS-1.md`](ERC%20%26%20EIP%20-%20phase%202/TFS-1.md) — Transaction Fidelity Standard overview

#### What It Does

1. **Intent Declaration**: Users sign an EIP-712 structured `Intent` that declares the expected outcomes of their transaction (transfers, approvals, amounts, etc.)
2. **Pre-Inclusion Verification**: Before a transaction is included in a block, verifiers (clients, builders, or validators) check that the **actual execution effects** match the signed `Intent`, within user-defined tolerances
3. **Rejection on Mismatch**: If execution diverges from intent (beyond slippage bounds), the transaction is rejected before inclusion

#### Architecture

```
┌──────────────┐
│     User     │
└──────┬───────┘
       │
       │ 1. Reviews simulated outcomes
       ├─ 2. Signs Intent (EIP-712)
       │  
       ▼
┌──────────────────────┐
│   Wallet / dApp      │
│  (ERC-XXXX Intent)   │
└──────┬───────────────┘
       │
       │ 3. Submits transaction + signed Intent
       │
       ▼
┌────────────────────────┐
│  Verifier / Builder    │
│  (EIP-YYYY)            │
│  - Simulates execution │
│  - Normalizes effects  │
│  - Compares to Intent  │
└──────┬─────────────────┘
       │
       │ 4a. Match → Include
       │ 4b. Mismatch → Reject
       │
       ▼
┌──────────────┐
│   On-Chain   │
└──────────────┘
```

#### Key Benefits

✅ **Eliminates TOCTOU attacks** — execution is verified against intent pre-inclusion  
✅ **Stops malicious MEV** — sandwiches and frontrunning that violate intent are rejected  
✅ **Enforces transaction fidelity** — cryptographic proof of execution correctness  
✅ **Preserves DeFi flexibility** — supports slippage tolerances for price-sensitive operations  
✅ **Fully auditable** — intent and execution hashes are logged for transparency  
✅ **Backward compatible** — wallets can fall back to traditional flow for legacy transactions  

#### Requirements

This phase requires:
- Client-side support for intent verification (EIP-YYYY)
- Builder/validator integration to enforce pre-inclusion checks
- Wallet adoption of the Intent standard (ERC-XXXX)
- Ecosystem coordination and gradual rollout

---

## Implementation Roadmap

### For Wallets

**Now (Phase 1)**
1. Implement outcome-based simulation using event extraction
2. Add local policy engine for asset-level controls
3. Display plain-language transaction summaries to users
4. Extend to off-chain signature previews

**Future (Phase 2)**
1. Implement Intent signing (EIP-712) in transaction flow
2. Allow users to configure per-outcome slippage tolerances
3. Submit transactions with signed Intent metadata
4. Display verification status and rejection reasons

### For Node Operators / Builders

**Future (Phase 2)**
1. Implement effect normalization from traces/logs (EIP-YYYY)
2. Add intent verification to block building pipeline
3. Reject transactions that fail equivalence checks
4. Log verification reports for auditability

### For dApp Developers

**Now (Phase 1)**
- Provide translation templates for your protocol's events
- Sign event-to-language mappings for verification

**Future (Phase 2)**
- Pre-generate Intent structures based on user actions
- Display expected outcomes with slippage suggestions
- Handle graceful fallback for legacy wallets

---

## Technical Context

### The Core Problem

The **Time-of-Check to Time-of-Use (TOCTOU)** gap is a systemic weakness in all blockchains:

1. **Simulation Time**: User previews the transaction in their wallet (T₀)
2. **Signing Time**: User approves and signs (T₁)
3. **Mempool Time**: Transaction propagates through the network (T₂)
4. **Execution Time**: Transaction is included and executed on-chain (T₃)

Between T₀ and T₃, malicious actors can:
- Frontrun the transaction
- Modify contract state via upgrades or governance
- Reorder transactions to extract value
- Spoof simulation results entirely

**Existing solutions are insufficient:**
- Private RPCs/mempools → centralized, no execution guarantees
- Risk scoring → cannot detect runtime discrepancies
- EIP-7730 (human-readable transactions) → makes *calls* readable, not *outcomes*
- Transaction batching → doesn't prevent *unintended* actions

---

## Why This Matters

### For Users
- **See exactly what will happen** before signing, in plain language
- **Protection from phishing** and hidden approvals
- **Assurance against manipulation** between signing and execution
- **Control over assets** with lockable permissions

### For the Ecosystem
- **Restores trust** in wallet and dApp interactions
- **Reduces support burden** from scam victims
- **Enables safer DeFi** with verifiable execution
- **Foundation for future innovations** (solvers, account abstraction)

---

## Status & Contributions

**Phase 1**: ✅ Specification complete — ready for wallet implementation  
**Phase 2**: ✅ Specification complete — ready for client/EIP process

We welcome:
- Wallet implementations and feedback
- Security reviews and formal analysis
- Client/builder prototype implementations
- Community discussion and refinement

---

## Learn More

- **Context & Motivation**: [`context.md`](context.md)
- **Phase 1 Specification**: [`ERC - phase 1/erc-outcome-based-simulation.md`](ERC%20-%20phase%201/erc-outcome-based-simulation.md)
- **Phase 2 Specifications**: [`ERC & EIP - phase 2/`](ERC%20%26%20EIP%20-%20phase%202/)
  - Intent Standard: `erc-intent.md`
  - Verification Layer: `eip-intent-verification.md`
  - Transaction Fidelity Standard: `TFS-1.md`

---

## License

All specifications are released under [CC0-1.0](https://creativecommons.org/publicdomain/zero/1.0/) — public domain dedication.

---

**IntentGuard** — Making "What You See Is What You Get" a reality for blockchain transactions.
