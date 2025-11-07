---
ERC: XXXX
Title: IntentGuard — Transaction Contractualization for Execution Fidelity
Author: Jean-Loïc Mugnier (IPSProtocol) <contact@ipsprotocol.org>
Status: Draft
Type: Standards Track
Category: ERC
Created: 2025-08-25
Discussions-To: <link to GitHub issue or ethresear.ch thread>
Requires: EIP-712 (recommended)
License: CC0-1.0
---

# 1. Abstract

This ERC defines **Intent**, an EIP-712 typed structure for declaring and signing the **expected outcome** of an Ethereum transaction.  
By standardizing how wallets represent these outcomes — such as token transfers, approvals, or value movements — this specification enables verifiable comparison between the **user-approved intent** and the **actual on-chain execution** within user-defined tolerances.  
It establishes the foundation for **Transaction Fidelity**, the *“What You See Is What You Get”* principle for blockchain transactions.

---

# 2. Specification

## 2.1. Data Structures (EIP-712 Types)

> *EVM-native:* `kind = keccak256(canonical event signature)`  
> Example: `keccak256("Transfer(address,address,uint256)")`

```solidity
struct EventParameter {
  string  paramType; // MUST — ABI type string (e.g., "address", "uint256", "bytes32")
  string  name;      // OPTIONAL — parameter name if available
  bytes   value;     // MUST — ABI-encoded value
}

struct Outcome {
  bytes32 kind;              // MUST — event signature hash (topic0)
  address asset;             // MUST — contract emitting the event; 0x0 for ETH
  address source;            // MUST — origin of the state or value change
  address beneficiary;       // MUST — final recipient, spender, or owner
  EventParameter[] params;   // OPTIONAL — ABI-decoded parameters for rendering
}
struct Intent {
  uint256   version;      // MUST
  address   issuer;       // MUST
  uint256   issuerNonce;  // MUST (== tx.nonce)
  uint256   deadline;     // MUST
  Outcome[] outcomes;     // MUST
}
```

---

## 2.2. EIP-712 Types (Canonical)

> **Field order is normative.** Implementations MUST preserve this exact order.

```json
{
  "EventParameter": [
    {"name":"paramType","type":"string"},
    {"name":"name","type":"string"},
    {"name":"value","type":"bytes"}
  ],
  "Outcome": [
    {"name":"kind","type":"bytes32"},
    {"name":"asset","type":"address"},
    {"name":"source","type":"address"},
    {"name":"beneficiary","type":"address"},
    {"name":"params","type":"EventParameter[]"}
  ],
  "Intent": [
    {"name":"version","type":"uint256"},
    {"name":"issuer","type":"address"},
    {"name":"issuerNonce","type":"uint256"},
    {"name":"deadline","type":"uint256"},
    {"name":"outcomes","type":"Outcome[]"}
  ]
}
```

---

## 2.3. EIP-712 Signing Rules

**Domain (normative):**

```json
{
  "name": "Intent",
  "version": "1",
  "chainId": <TARGET_CHAIN_ID>,
  "verifyingContract": "0x0000000000000000000000000000000000000000"
}
```

Wallets **MUST** display before signing:

- All declared outcomes in human-readable form.  
- Each outcome’s asset (token or “ETH” for 0x0), direction (inflow/outflow), beneficiary, and amount (token-formatted if available).  
- Optional contextual data such as `chainId`, sender account, and estimated gas cost.

## 2.4 Parameter Semantics

- paramType follows canonical Solidity type strings (e.g., uint256, address, bytes32, tuple, etc.).
- value is the ABI-encoded representation of the parameter.
- name MAY be omitted if unavailable from ABI metadata.
- The params array enables wallets and verifiers to reconstruct the complete semantic context of each event.

---

# 3. Signing and Display Requirements

Wallets MUST render and display all declared outcomes of the Intent before signing.
Each outcome SHOULD be presented in a clear, human-readable format indicating the involved asset, the direction (inflow or outflow), and the beneficiary address.

Wallets SHOULD allow users to review and configure a slippage tolerance for each outcome prior to signing, expressed as a percentage or parts-per-million (slippagePpm).
Default values MAY be provided by the dApp or simulation plugin, but the user MUST have the ability to modify them.

Wallet interfaces SHOULD also include contextual information (e.g., chainId, sender account, estimated gas cost) but MAY omit internal fields such as issuerNonce or other non-user-facing metadata.

Wallets MAY reuse simulation outputs defined under Outcome-Based Transaction Simulation (ERC-XXXX) to prefill outcomes before signing.

Implementations MUST preserve field order exactly as defined when encoding or signing the typed data.

---

# 4. Conformance

| Actor             | MUST / SHOULD requirements                                                                                                         |
| :---------------- | :--------------------------------------------------------------------------------------------------------------------------------- |
| **Wallet**        | Display all outcomes (with `src` and `beneficiary`), allow per-outcome slippage configuration, and sign the exact EIP-712 message. |
| **dApp** | maintain their own event to impact translation service          |
| **Verifier** | Match onchain emitted events against the signed Intent parameters and structure.  |


## 4.1 Multiple 'Similar' Outcomes 

If a transaction emits multiple events of the same `kind/asset/src/beneficiary`, clients SHOULD **aggregate** values (sum), producing a single Outcome with the net `amount`.

# 5. Intent–Execution Linkage

Verifiers or nodes linking an Intent to an executed transaction MUST match by (issuer, issuerNonce) and verify that all emitted events correspond to declared Outcome.kind and Outcome.asset.
Parameter comparison is deterministic: paramType and value MUST match exactly, within any contextual tolerances defined externally.

A future EIP may define the onchain attestation process whereby a verifier publishes a record linking:

# 6. Security Considerations

- `src` and `beneficiary` MUST both be explicit to prevent misdirection or invisible redirection attacks.(eg. malicious approvals)
- `issuer` and `issuerNonce` bind the record to a unique transaction scope, enabling controlled transaction updates and preventing replay.  
- `deadline` defines the intent’s validity window and prevents accounts from remaining locked by expired or obsolete records.  
- Hiding internal fields (e.g., nonces) from the UI is permitted as long as they remain part of the signed data.  
- Hardware wallets may require dual confirmations (EIP-712 message and on-chain transaction).  

---

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
