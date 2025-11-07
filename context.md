  ## Abstract

  This ERC defines a multi-component protocol for **Transaction Contractualization**, introducing **Transaction Fidelity** — the *“What You See Is What You Get” (WYSIWYG)* principle for blockchain transactions.  
  The protocol establishes a verifiable link between the **expected outcome displayed to the user before transaction signing** and the **actual on-chain execution**, ensuring that the transaction performs exactly what the user approved and signed.  

  Given the nature of **DeFi transactions**, where market conditions may evolve between simulation and inclusion, this ERC introduces the concept of **off-chain slippage bounds** — allowing limited, user-defined variation between simulated and executed results while preserving verifiable fidelity.  

  Participants MAY verify that the realized outcomes of a candidate transaction are **equivalent** (or within acceptable tolerances) to the declared intent.  
  A transaction **MUST only be included** if its on-chain effects faithfully reproduce the **user-approved intent** within the specified bounds.


  ## Motivation

  Wallets and dApps commonly perform transaction simulations to preview the expected outcome before the user signs.  
  Some wallets focus on decoding and displaying what is being called — using ABI decoding or emerging standards such as **EIP-7730** — while others may choose to show a summarized simulation outcome.  

  However, these simulations are **purely informational and not contractual**: they are based on local state and carry no enforceable relationship with the eventual on-chain execution.  
  Between simulation, user signing, and final transaction inclusion, there exists an important **gap in time** that allows for market evolution and adversarial behaviors, including **Transaction Simulation Spoofing (TSP)** and **MEV sandwich attacks**.  

  This exposes the well-known **Time-of-Check to Time-of-Use (TOCTOU)** problem — a systemic weakness present in **all blockchains**, EVM-based or not.  
  As a result, users cannot verify that what they observed during simulation is what will actually occur on-chain.  

  After evaluating the aforementioned constraints, dynamics, and adversarial activities (such as MEV and TSP), it becomes evident that protecting user transactions with **predictability and fidelity** requires a protocol capable of verifying — *before transaction inclusion* — that the transaction indeed performs **what the user signed off for** (the declared **user intent**).  

  When examining the current technology stack between wallets and the proposers (validator), it becomes clear that there is **no simple or standardized way** to perform these verifications.  


  ## Existing Approaches and Industry Context

  Various strategies have emerged to reduce discrepancies between simulated and executed transaction outcomes and to mitigate related front-running or spoofing attacks:

  1. **Private RPCs and Order-Flow Channels**  
    Private endpoints and relays are used to route transactions outside the public mempool to reduce exposure to mempool-based attacks such as front-running and MEV extraction.  
    *Limitation:* These solutions rely on **centralized off-chain intermediaries** and provide **no verifiable proof** that the final on-chain execution matches the simulated outcome.  
    They also **only protect against front-runs occurring within the public mempool**, not against **Transaction Simulation Spoofing (TSP)** attacks, which can drain user wallets by modifying the target contract logic *after* the user has signed the transaction — an attack vector that originates at the **dApp layer**, not in the mempool.

  2. **Transaction Risk Scoring Systems**  
    Wallets and security providers integrate static analysis, signature databases, or heuristic scanning of known malicious contracts and addresses.  
    *Limitation:* These methods depend on **curated datasets** and cannot scale to unknown or dynamically deployed contracts.  
    They analyze code or metadata rather than **runtime behavior**, and therefore cannot detect **Time-of-Check to Time-of-Use (TOCTOU)** discrepancies between simulation and execution.

  3. **Human-Readable Transaction Standards**  
    (e.g., **EIP-7730**)  
    Provide structured, human-readable descriptions of calldata to improve user comprehension.  
    *Limitation:* These descriptions are **static** and depend on centralized or registered translation systems.  
    They make the *call* contractual, but cannot enforce or verify the **resulting outcome**.  
    Moreover, they cannot interpret dynamic calldata, nested delegate calls, or upgraded contract logic, and thus fail to reflect real-time runtime behavior.

  4. **Private Sequencers and Encrypted Mempools**  
    Some rollups and L2 systems use private or encrypted mempools to hide transaction contents until inclusion.  
    *Limitation:* While these models reduce visibility to external actors, they **reintroduce centralized trust assumptions** and still do not prevent manipulation at the application layer.  
    Like private RPCs, they protect against public mempool front-runs but not against **TSP-style attacks** that occur at the **dApp layer**, where adversaries can front-run user transactions by modifying the underlying contract logic before execution.

  5. **Bundled or Batched Transactions (EIP-7702 and related patterns)**  
    Techniques that bundle dependent transactions (on-chain or off-chain) can improve determinism and reduce race conditions, allowing partial verification of expected outcomes.  
    *Limitation:* While such batching can confirm that *what was expected to happen does happen*, it is **technically unfeasible to verify that something that was not supposed to happen did not happen**—for example, a phishing dApp silently inserting a malicious token approval.  
    These mechanisms enhance atomicity but do not protect against **unintended or malicious actions** introduced by compromised or deceptive dApps.



  Collectively, these approaches improve user safety and experience but still lack a standardized mechanism to ensure predictability and execution fidelity in DeFi transactions.