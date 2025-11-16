# WORK IN PROGRESS
---
EIP: YYYY
title: Intent Verification — Pre-Inclusion Effect Equivalence for Transaction Fidelity
author: Jean-Loïc Mugnier (IPSProtocol) <contact@ipsprotocol.org>
status: Draft
type: Standards Track
category: Core
discussions-to: <link to Magicians thread or GH issue>
requires: EIP-712, ERC-XXXX (Intent)
license: CC0-1.0
---

# 1. Abstract

This EIP specifies client-side verification semantics to ensure a transaction is only included if its **observed effects** match a user-signed **Intent** (ERC-XXXX), within user-defined tolerances.  
It defines: (a) a canonical **effect normalization** procedure, (b) an **equivalence rule**, and (c) a minimal **verification interface** usable pre-inclusion via candidate block evaluation.

---

# 2. Motivation

Wallets can collect and sign an Intent (expected outcomes); however, without client-side verification, there is no verifiable linkage between **exxpected** and **actual** effects.  
This EIP provides a standard way for clients to **normalize effects** and **compare** them to the signed Intent, enabling Transaction Fidelity while remaining compatible with existing inclusion pipelines.

---

# 3. Definitions

- **Intent (ERC-XXXX)**: EIP-712 message with `Intent` and `Outcome[]` where each outcome has `(kind, asset, origin, beneficiary, amount, slippagePpm)`.
- **Outcome**: An executed runtime observation derived from tx or block simulation, normalized to match the `Outcome` shape.
- **Normalization**: Deterministic extraction of effects from traces/logs, including a synthetic representation for native ETH movement.

---

# 4. Effect Normalization

Clients MUST normalize executed behavior into a set of **Outcomes** suitable for comparison with `Outcome[]`.

## 4.1 Event-centric extraction

- For token/contract events, use:
  - `kind` = `topic0` (keccak256 of canonical event signature)
  - `asset` = `log.address`
  - Map ABI fields per canonical semantics to fill `origin` and `beneficiary`:
    - `Transfer(address from, address to, uint256 value)`  
      `origin = from`, `beneficiary = to`, `amount = value`:
    - Protocol-specific events MAY be supported if their semantics are registered canonically.

- For approvals or allowance-like effects, clients SHOULD normalize to `kind = keccak256("Approval(address,address,uint256)")` with:
  - `asset` = token contract
  - `origin` = owner
  - `beneficiary` = spender
  - `amount` = target allowance

## 4.2 Native ETH movement

The EVM does not emit an event for ETH transfers.

ETH_TRANSFER_KIND = keccak256("ETH.Transfer(address,address,uint256)")

For any value movement detected in call frames:
- `kind = ETH_TRANSFER_KIND`
- `asset = 0x0000000000000000000000000000000000000000`
- `origin = msg.sender in the context of the recipient`
- `beneficiary = msg.value recipient`
- `amount = value`


## 4.4 Sign Convention

Let `issuer` be the Intent `issuer`. For each normalized Effect:
- If the effect increases the issuer’s holdings relative to the asset/beneficiary pair, set `amount > 0`.
- If the effect decreases the issuer’s holdings (issuer == `origin` and tokens/value flow out), set `amount < 0`.

> Rationale: preserves parity with the ERC’s `Outcome.amount` sign semantics.

---

# 5. Equivalence Rule

Let `A = |Outcome.amount|` (declared magnitude) and `E = |Effect.amount|` (executed magnitude).  
An effect **matches** an outcome iff:

1. kind == outcome.kind
2. asset == outcome.asset
3. origin == outcome.origin
4. beneficiary == outcome.beneficiary

5. amount:
```python
    if slippagePpm == 0: 
        E == A
    else: 
        |E - A| ≤ A * slippagePpm / 1_000_000
```

All declared outcomes MUST be matched by corresponding effects (1:1 after aggregation).  
Undeclared extra effects COULD cause failure if involve the transaction issuer (eg: tx.sender)


# 6. Mapping and Validity

Before equivalence, the verifier MUST check:

- `tx.chainId == intent.domain.chainId`
- `(intent.issuer, intent.issuerNonce)` ↔ `(tx.from, tx.nonce)` mapping holds for EOAs;  
  for EIP-7702 delegated transactions, `issuer` equals the logical sender from the delegation profile.
- `block.timestamp ≤ intent.deadline`

If any check fails, verification fails.


# 7. Verification Interface

This EIP defines a minimal programmatic interface. JSON-RPC method names are informative; clients MAY expose them under local or IPC namespaces.

## 7.1 Types (normalized)

```json
{
  "OUt": {
    "kind": "0x<32b hash>",
    "asset": "0x<20b>",
    "origin": "0x<20b>",
    "beneficiary": "0x<20b>",
    "amount": "<int256 decimal string>"
  },
  "VerificationReport": {
    "equivalent": true,
    "reasonCode": 0,
    "recordHash": "0x<32b>",   // keccak256(typed Intent)
    "effectsHash": "0x<32b>"   // keccak256(canonicalized Effects array)
  }
}
```


# 8. Failure Semantics (Reason Codes)

| Code | Meaning                         |
|------|---------------------------------|
| 0    | OK                              |
| 1    | OUT_OF_TOLERANCE                |
| 2    | UNEXPECTED_USER_EFFECT               |
| 3    | MISSING_EFFECT                  |
| 4    | EXPIRED_OR_DEADLINE             |
| 5    | CHAINID_MISMATCH                |
| 6    | MAPPING_MISMATCH                |
| 8    | MALFORMED_INTENT                |
| 9    | ENGINE_ERROR                    |

Implementations SHOULD log structured metadata to facilitate audit (tx hash, recordHash, matched pairs).


# 9. Policy Notes

- **Extra effects** (not declared in the Intent) SHOULD cause failure only if impact user.(code 2)
- **Approvals/allowances** SHOULD require exact value match (`slippagePpm = 0`).  
- **Aggregation** (multi-event netting) MUST be deterministic.


# 10. Relationship to ERC-XXXX (Intent)

This EIP defines the **verifier linkage layer** for ERC-XXXX.  
While ERC-XXXX standardizes how **users declare intents**, this EIP specifies how **verifiers** — either local clients, simulation nodes, or block builders — perform **pre-inclusion effect comparison** to ensure execution fidelity.

Verifiers apply the normalization and equivalence rules defined here to evaluate candidate transactions against signed Intents, forming the final step in the  transaction fidelity model:

User → Wallet → Intent (ERC-XXXX)
↓
Verifier (EIP-YYYY)
↓
Execution (onchain inclusion)

In the ERC-XXXX specification, this linkage is referenced as the external **verification layer** responsible for enforcing pre-inclusion equivalence.  
This EIP provides the concrete semantics and interface for that verification process.


# 11. Security Considerations

- Verifiers MUST perform equivalence checks deterministically and auditable under replay.  
- Builders integrating this process MUST ensure rejection or quarantine of transactions with failed verification.  
- Intent hashes and Effects hashes SHOULD be logged for reproducibility.  
- A malicious verifier cannot alter inclusion outcomes but could censor; redundancy across verifiers mitigates this.  
- Pre-inclusion verification SHOULD be optional but verifiable, allowing fallback to traditional mempool behavior for compatibility.


# 13. Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
