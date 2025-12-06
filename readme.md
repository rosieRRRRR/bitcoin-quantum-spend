# Quantum Resistant Bitcoin Spend
OTQH-M + Two-Secret Multi-Child Reveal (TS-MRC)

Version: v1.0.0
Status: Draft for review
Author: rosiea
Contact: [PQRosie@proton.me](mailto:PQRosie@proton.me)
Date: December 2025
Licence: Apache License 2.0 — Copyright 2025 rosiea



---

## 1. Status

Status: Draft for review
Scope: Bitcoin Script, P2WSH, HD wallets (including PQHD), public Bitcoin network
Consensus changes required: None

---

## 2. Abstract

This specification defines a Bitcoin spend pattern that:

* Enforces a **one-time, non-replayable spend** for a protected UTXO (modulo normal chain reorgs).
* Provides **operational quantum resistance** on the public Bitcoin network by shrinking and hardening the effective attack window.
* **Obstructs mempool attackers** by hiding critical structure and secrets until shortly before confirmation.
* Uses only existing Script primitives (CLTV, CSV, hashlocks, multisig).
* Requires **no consensus changes**.

The core security property is that the effective exposure window is shorter than the time an attacker needs to construct and propagate a rival transaction:

```math
\text{Exposure\_Window}(\Delta t_{\text{eff}}) < \text{Time}_{\text{rival}}
```

where:

* (\Delta t_{\text{eff}}) is the time from when the parent transaction reveals its script and secrets ((S_1) and ephemeral pubkeys) until a spend of the protected UTXO is confirmed.
* (\text{Time}_{\text{rival}}) includes:

  * classical overhead to parse, construct, and propagate a rival transaction, and
  * time to recover multiple independent private keys, potentially using quantum resources.

The construction uses:

* **OTQH-M:** an M-of-M multisig over independent ephemeral keys that are **destroyed after signing**.
* **TS-MRC:** a two-secret structure where the parent spend is gated by secret (S_1) and multiple child transactions are gated by secret (S_2), with hashlock linkage and script contents hidden until reveal.

Optional:

* **HGSP:** a CSV-based cooldown / spend delay for downstream robustness (not required for the core quantum mitigation).

The security guarantees are operational and statistical. This is an engineered mitigation against mempool replacement attacks under realistic assumptions, not a formal cryptographic proof. 

---

## 3. Quick Start (Informative)

For a minimal OTQH-M + TS-MRC deployment:

1. **Ephemeral keys**

   Generate 3 independent ephemeral keys: (K_1), (K_2), (K_3).

2. **Secrets**

   Generate two 256-bit secrets: (S_1), (S_2).

   Compute:

   ```math
   H_{S1} = \text{SHA256}(S_1), \quad H_{S2} = \text{SHA256}(S_2)
   ```

3. **Parent**

   Build a P2WSH parent redeemScript:

   ```text
   T_lock OP_CHECKLOCKTIMEVERIFY OP_DROP
   OP_SHA256 H_S1 OP_EQUALVERIFY
   3 K1_pub K2_pub K3_pub 3 OP_CHECKMULTISIG
   ```

   Fund this P2WSH with the protected UTXO.
   Sign the parent with (K_1), (K_2), (K_3).
   **Irrecoverably delete all ephemeral private keys.**

4. **Child**

   Build a P2WSH child redeemScript:

   ```text
   OP_SHA256 H_S2 OP_EQUALVERIFY
   <CHILD_POLICY>       # e.g. 2 A1_pub A2_pub 2 OP_CHECKMULTISIG
   ```

   Create a child transaction spending the parent’s (H_{S2})-committing output.

5. **Broadcast timing**

   * Broadcast the parent (with (S_1) revealed in the witness) about 10–20 seconds before the target block satisfying `T_lock`.
   * Broadcast the child(ren) (with (S_2) revealed in the witness) about 1–3 seconds before that same block.

6. **Monitor**

   * Ensure the parent + child package is included as a CPFP package.
   * If timing or fees fail, rebuild with new secrets and ephemeral keys.

---

## 4. Motivation

Standard Bitcoin spends reveal public keys upon broadcast. A sufficiently capable quantum attacker monitoring the mempool could:

* Observe a transaction spending a valuable UTXO.
* Recover the private key from the exposed public key.
* Construct and propagate a higher-fee rival transaction spending the same UTXO.

The OTQH-M + TS-MRC pattern reduces this risk by:

* Using multiple independent ephemeral keys dedicated to a single spend.
* Destroying these ephemeral private keys immediately after signing.
* Withholding secrets that link parent and children until late in the process.
* Structuring broadcast timing so the window between “all necessary secrets visible” and “confirmation” is as short as possible.

The attacker’s effective time budget is reduced, while their work (multiple key recoveries plus full transaction construction and propagation) is increased.

---

## 5. Threat Model

### 5.1 Baseline Assumptions

* The attacker has full mempool visibility.
* The attacker can construct arbitrary Bitcoin transactions and pay higher fees.
* The attacker may have access to a quantum computer capable of breaking an ECDSA key in seconds or minutes, but not instantaneously.
* The attacker does not control consensus rules.
* The attacker does not possess local private keys, PQHD internal state, or any out-of-band wallet state.

### 5.2 Network and Topology Assumptions

* No privileged miner access: the attacker does not have private low-latency channels directly to miners that bypass normal P2P propagation.
* Non-zero propagation overhead: classical parsing and propagation of a rival transaction takes a non-trivial time (T_{\text{overhead}}) (for example a few seconds), which may be deployment-specific.
* Standard miner behaviour: miners generally prioritise higher-fee transactions, but do not enforce arbitrary inclusion policies for a specific attacker beyond fee and policy norms.

### 5.3 Quantum Capability Assumptions

* The attacker can run a limited number (K) of concurrent quantum ECDLP (key-breaking) instances.
* Time to break one ECDSA key is (T_Q).

Effective break time to recover (M) keys is approximated as:

```math
T_{\text{keys}} \approx \left\lceil \frac{M}{K} \right\rceil \times T_Q
```

### 5.4 Defended Against

* Mempool replacement attacks using quantum-assisted key recovery after parent broadcast.
* Front-running attempts that try to double-spend the same protected UTXO via mempool sniping.
* Observers trying to infer downstream spend structure and policy before reveal.

### 5.5 Out of Scope

* Compromise of all key material before signing.
* Near-instantaneous quantum capability that breaks multiple keys in sub-second time.
* Attackers with direct, privileged channels to major miners combined with large-scale parallel quantum capacity.
* Operator error in timing, fee selection, or key-handling procedures.

### 5.6 Threat Model Summary (Informative)

| Threat                               | Mitigation                                     | Key Assumptions                                               |
| ------------------------------------ | ---------------------------------------------- | ------------------------------------------------------------- |
| Quantum key recovery after broadcast | M-of-M ephemeral keys, key destruction, timing | Attacker cannot break (M) keys within (\Delta t_{\text{eff}}) |
| Mempool replacement / fee sniping    | High-fee CPFP package, short exposure window   | Standard miner fee behaviour, no private inclusion deals      |
| Mempool structure analysis           | TS-MRC hides (S_2) and downstream scripts      | Attackers see graph, not hashlock linkage                     |
| Reuse of signing keys                | Strict ephemeral key lifecycle, PQHD policies  | Implementation correctly enforces deletion                    |
| Reorg-based replays                  | Rebuild with new keys/secrets after reorg      | Operator monitors chain state                                 |

---

## 6. Design Goals

* **Single effective spend** per protected UTXO.
* **Operational quantum mitigation:** make it operationally infeasible for an attacker to construct and propagate a rival transaction within the effective exposure window.
* **Mempool obstruction:** delay disclosure of secrets and script contents to limit adversarial insight into the structure of the spend.
* **No consensus changes:** only existing opcodes and features (CLTV, CSV, hashlocks, multisig, P2WSH).
* **Wallet-agnostic design:** works with classical HD wallets; integrates cleanly with PQHD where available.
* **Clear operational procedures:** explicit lifecycle for key generation, signing, deletion, broadcast, and failure handling.

---

## 7. Components

### 7.1 OTQH-M (Multi-Ephemeral Protected Spend)

The parent transaction uses an M-of-M multisig:

```math
\text{M } \langle K1_{\text{pub}} \rangle \langle K2_{\text{pub}} \rangle \ldots \langle KM_{\text{pub}} \rangle \text{ M OP\_CHECKMULTISIG}
```

where:

* All (K_i) are independent ephemeral signing keys created for a single protected spend.
* Ephemeral private keys are destroyed immediately after signing the parent.
* No long-term user key participates in the parent multisig. Long-term keys (K_{\text{user}}) are reserved for downstream policy or change outputs.

Optional additions in the parent redeemScript:

* `OP_CHECKLOCKTIMEVERIFY` over (T_{lock}).
* Hashlock over (H_{S1}) (SHA256 of (S_1)).

### 7.2 TS-MRC (Two-Secret Multi-Child Reveal)

TS-MRC uses two independent secrets:

* (S_1): gates the parent spend (for example via a hashlock in the parent script).
* (S_2): gates multiple child spends (for example via hashlocks in their scripts and funding outputs).

Properties:

* Children appear as independent hashlocked P2WSH spends until (S_2) is revealed in their witnesses.
* Parent and children can be constructed so that their hashlock linkage and script contents are not directly inferable until reveal.

### 7.3 Visibility Properties

| Hidden until reveal                                   | Visible to mempool observers                     |
| ----------------------------------------------------- | ------------------------------------------------ |
| The value of (S_2)                                    | Transaction graph topology (child spends parent) |
| Downstream script contents (P2WSH redeemScripts)      | CPFP relationships and effective package feerate |
| Explicit hashlock linkage between parent and children | Timing patterns and fee signals                  |

TS-MRC obscures secrets and policy, not the existence of graph relationships.

### 7.4 HGSP (Optional Cooldown)

HGSP (Hash-Gated Spend Path) uses `OP_CHECKSEQUENCEVERIFY` (CSV) and/or chained hashlocks to enforce relative cooldowns on downstream spends. HGSP is recommended for multi-hop or multi-stage policies but not required for the core quantum-resistance logic.

### 7.5 Timing Model

Reference block: the first block at or after which (T_{lock}) is satisfiable for the parent CLTV.

Definitions:

* (\Delta t_1): time between parent broadcast and expected (T_{lock})-satisfying block (for example 10–20 seconds).
* (\Delta t_2): time between child broadcast and expected (T_{lock})-satisfying block (for example 1–3 seconds).

These are operational targets, not guarantees.

---

## 8. Core Construction

### 8.1 Keys and Secrets

Keys:

* (K_{\text{user}}): long-term wallet key (optional in this spend pattern).
* (K_1, \ldots, K_M): ephemeral keys for the parent M-of-M multisig.

Secrets:

* (S_1, S_2): independent, uniformly random 256-bit values.
* (S_2) must not be derivable from (S_1) or any public values.

Hashes:

```math
H_{S1} = \text{SHA256}(S_1), \quad H_{S2} = \text{SHA256}(S_2)
```

### 8.2 Mandatory Ephemeral Key Lifecycle

| Time                     | Action                                                                         |
| ------------------------ | ------------------------------------------------------------------------------ |
| T-0: Generation          | Generate (K1_{\text{priv}}, \ldots, KM_{\text{priv}}) in a secure environment. |
| T-1: Sign parent         | Sign (Tx_P) with all (M) ephemeral keys.                                       |
| T-2: Destruction         | Irrecoverably delete all ephemeral private keys from all storage and memory.   |
| T-3: Waiting period      | Keys no longer exist; no broadcast yet.                                        |
| T-4: Parent broadcast    | Broadcast (Tx_P) with (S_1) at approximately (T_{lock} - \Delta t_1).          |
| T-5: Network propagation | (Tx_P) propagates; miners learn script, pubkeys, and (S_1).                    |
| T-6: Child broadcast     | Broadcast children with (S_2) at approximately (T_{lock} - \Delta t_2).        |

### 8.3 Parent Transaction (Tx_P)

Parent redeemScript (RedeemScript_P):

```text
<T_lock> OP_CHECKLOCKTIMEVERIFY
OP_DROP
OP_SHA256 <H_S1> OP_EQUALVERIFY
M <K1_pub> <K2_pub> ... <KM_pub> M OP_CHECKMULTISIG
```

Parent witness:

```text
<empty>
<sig_K1>
<sig_K2>
...
<sig_KM>
<S1>
<RedeemScript_P>
```

(Tx_P) must include at least one output whose script commits to (H_{S2}) (for example a P2WSH redeemScript including a hashlock over (H_{S2})).

### 8.4 Child Transactions

Example child redeemScript (RedeemScript_A):

```text
OP_SHA256 <H_S2> OP_EQUALVERIFY
2 <A1_pub> <A2_pub> 2 OP_CHECKMULTISIG
```

Child witness:

```text
<empty>
<sig_A1>
<sig_A2>
<S2>
<RedeemScript_A>
```

### 8.5 Broadcast Sequence

1. Build all scripts and transactions.
2. Sign (Tx_P) with all ephemeral keys.
3. Destroy ephemeral private keys.
4. Broadcast parent (Tx_P) with (S_1) at (T_{lock} - \Delta t_1).
5. Broadcast children at (T_{lock} - \Delta t_2) with (S_2).
6. Aim for confirmation at or shortly after (T_{lock}).

### 8.6 Rival Transaction Requirements

Attacker must:

1. Observe (Tx_P) in the mempool and parse its witness.
2. Extract all ephemeral pubkeys and (S_1).
3. Recover all (M) ephemeral private keys.
4. Construct a rival transaction spending the same input, with valid signatures, correct locktime and CLTV, and (S_1).
5. Propagate the rival transaction fast enough that miners include it instead of the honest spend.

### 8.7 Operational Security Constraint

Critical exposure window:

```math
\Delta t_{\text{eff}} \approx \Delta t_1 + \text{block timing variance} + \text{propagation delays}
```

Attacker required time:

```math
\text{Time}_{\text{rival}} = T_{\text{overhead}} + T_{\text{keys}},
\qquad
T_{\text{keys}} \approx \left\lceil \frac{M}{K} \right\rceil \times T_Q
```

Operational requirement:

```math
\Pr[ \Delta t_{\text{eff}} < \text{Time}_{\text{rival}} ] \approx 1
```

In practice:

* (M) is the main security knob.
* (\Delta t_1) should be kept small to reduce (\Delta t_{\text{eff}}).
* Implementations should monitor network conditions to estimate realistic (T_{\text{overhead}}).

---

## 9. Example Transactions

### 9.1 Parameters

* (T_{lock} = 500000)
* (M = 3)
* (\Delta t_1 = 20) seconds
* (\Delta t_2 = 2) seconds

### 9.2 Parent RedeemScript

```text
500000 OP_CHECKLOCKTIMEVERIFY
OP_DROP
OP_SHA256 0xH_S1 OP_EQUALVERIFY
3 0xK1_pub 0xK2_pub 0xK3_pub 3 OP_CHECKMULTISIG
```

### 9.3 Example Child RedeemScript

```text
OP_SHA256 0xH_S2 OP_EQUALVERIFY
2 0xA1_pub 0xA2_pub 2 OP_CHECKMULTISIG
```

Funding output for this child is a P2WSH scriptPubKey:

```text
OP_0 <32-byte sha256(RedeemScript_Child)>
```

---

## 10. Security Analysis

### 10.1 Non-Replayability

Once the parent transaction confirming the protected UTXO is included in a block on the best chain, that UTXO is consumed. Replay of that exact transaction on another tip fails because the UTXO no longer exists there.

### 10.2 Non-Replaceability (Practical)

After ephemeral private keys are destroyed, the wallet cannot produce alternative signatures for the parent input. A rival transaction requires recovery of all (M) ephemeral private keys or compromise of signing infrastructure before deletion.

As long as the attacker cannot break all ephemeral keys within (\Delta t_{\text{eff}}), practical replacement becomes infeasible.

### 10.3 Operational Quantum Resistance

Attacker must:

* Break enough keys to satisfy the M-of-M multisig.
* Fit key recovery and construction/propagation of a rival transaction within (\Delta t_{\text{eff}}).

In the bounded-parallelism model:

```math
\text{Time}_{\text{rival}} \approx T_{\text{overhead}} + \left\lceil \frac{M}{K} \right\rceil \times T_Q
```

We aim for:

```math
\Delta t_{\text{eff}} \ll \text{Time}_{\text{rival}}
```

### 10.4 Mempool Obstruction

TS-MRC hides:

* script contents and (S_2) until children are broadcast, and
* policy linkage between parent and children until late.

Graph structure and fee dynamics remain visible.

### 10.5 Limitations and Operational Caveats

* Quantum parallelism: if attackers can break all (M) keys in parallel with negligible overhead, effective break time tends toward (\max(T_Q)).
* Miner collusion / privileged channels: private inclusion deals can reduce (T_{\text{overhead}}).
* Block timing variance: blocks can arrive earlier or later than predicted, affecting (\Delta t_{\text{eff}}).
* Operator error: mistakes in timing, fee setting, or key deletion can weaken protection.

### 10.6 Parallel Attack Assumption

We assume a bounded concurrency model:

* Attacker can run at most (K) concurrent quantum key-recovery processes.
* Effective key-breaking time grows with (\lceil M / K \rceil).

Deployments assuming stronger attackers must increase (M) and tighten timing accordingly.

---

## 11. Wallet and Implementation Considerations

### 11.1 Classical Wallets

Needs:

* Generation and management of separate ephemeral keys.
* Construction of custom P2WSH scripts.
* Control over `nLockTime` and `nSequence`.
* Full control of witness construction and transaction assembly.
* Careful key-handling to guarantee deletion after use.

### 11.2 PQHD Wallets

PQHD can:

* Derive ephemeral keys (K_1 \ldots K_M) deterministically under a quantum-hardened spend role.
* Enforce single-use semantics for ephemeral keys.
* Automatically and verifiably destroy ephemeral private keys after signing.
* Monitor network conditions and fee markets to keep (\Delta t_1, \Delta t_2) realistic.
* Express OTQH-M + TS-MRC as a policy template.

### 11.3 Timing Control

Wallets should:

* Estimate arrival time of the next block satisfying (T_{lock}).
* Schedule parent and child broadcasts to meet target (\Delta t_1) and (\Delta t_2).
* Use feerates in the top 5–10% of the mempool at broadcast time.
* Monitor mempool and block arrivals and adjust if behaviour deviates.

### 11.4 Failure Modes

* Parent not seen: rebroadcast with adjusted fees or rebuild using new keys and secrets.
* Children not seen or not confirmed: rebroadcast children; if only the parent confirms, downstream policy may require a new step.
* Missed (T_{lock}): rebuild the spend with fresh ephemeral keys and secrets; do not reuse ephemeral material.

---

## 12. Annex A — Script Templates

**Parent redeemScript**

```text
<T_lock> OP_CHECKLOCKTIMEVERIFY
OP_DROP
OP_SHA256 <H_S1> OP_EQUALVERIFY
M <K1_pub> <K2_pub> ... <KM_pub> M OP_CHECKMULTISIG
```

**Child funding redeemScript**

```text
OP_SHA256 <H_S2> OP_EQUALVERIFY
<CHILD_POLICY>
```

**Example child redeemScript**

```text
OP_SHA256 <H_S2> OP_EQUALVERIFY
2 <A1_pub> <A2_pub> 2 OP_CHECKMULTISIG
```

---

## 13. Annex B — Pseudocode

```python
def build_otqh_m_scripts(T_lock, M, ephemeral_pubs, H_S1, H_S2):
    # Parent redeemScript
    redeem_parent = [
        T_lock, "OP_CHECKLOCKTIMEVERIFY", "OP_DROP",
        "OP_SHA256", H_S1, "OP_EQUALVERIFY",
        M, *ephemeral_pubs, M, "OP_CHECKMULTISIG",
    ]

    # Child funding redeemScript (prefix; policy appended later)
    redeem_child_funding_prefix = [
        "OP_SHA256", H_S2, "OP_EQUALVERIFY",
    ]

    return redeem_parent, redeem_child_funding_prefix


def sign_parent_with_ephemerals(tx_p, ephemeral_keys, S1, redeem_parent):
    sigs = []
    for k in ephemeral_keys:
        sigs.append(sign(tx=tx_p, privkey=k.priv))

    # Irrecoverably delete keys
    for k in ephemeral_keys:
        k.priv = None

    witness = [""] + sigs + [S1, redeem_parent]
    tx_p.witness.append(witness)
    return tx_p
```

Child transactions are built similarly, with (S_2) revealed only when broadcasting children.

---

## 14. Annex C — Parameter Recommendations

### 14.1 Recommended Values (Informative)

* (M = 2) minimum, (M = 3) recommended.
* (\Delta t_1): 10–20 seconds.
* (\Delta t_2): 1–3 seconds.
* Fees: choose a package feerate in roughly the top 5–10% of the mempool.

Adjust based on:

* Observed block timing variance.
* Measured propagation and inclusion behaviour.
* Assumed attacker quantum capacity.

### 14.2 Quantum Break Time (Informative)

| Logical qubits | Time per ECDSA key | Notes                      |
| -------------- | ------------------ | -------------------------- |
| ~1 million     | Hours to days      | Current rough estimates    |
| ~10 million    | Minutes            | Optimistic future          |
| ~100 million+  | Seconds            | Aggressive future scenario |

Operational requirement (under bounded (K)):

```math
\left\lceil \frac{M}{K} \right\rceil \times T_Q \gg \Delta t_{\text{eff}}
```

### 14.3 Operational Checklist

1. Generate (S_1) and (S_2).
2. Generate ephemeral keys (K_1 \ldots K_M).
3. Build scripts and construct (Tx_P).
4. Sign (Tx_P) with all ephemeral keys.
5. Destroy all ephemeral private keys.
6. Broadcast parent near (T_{lock} - \Delta t_1).
7. Broadcast children near (T_{lock} - \Delta t_2).
8. Monitor confirmation and handle failures per Section 11.4.

---

## 15. Annex D — PQHD Integration Note

OTQH-M + TS-MRC is compatible with PQHD-style wallets. PQHD can:

* Express this pattern as a policy template.
* Ensure ephemeral keys are derived only for authorised quantum-hardened spends.
* Enforce one-time use and destruction of ephemeral keys.
* Provide auditability of timing, fees, and broadcast behaviour.

PQHD is not required, but strengthens operational guarantees and reduces operator risk.

---

## 16. Annex E — Worked Example Transactions

Network: regtest or signet
Scope: one protected UTXO, one parent (Tx_P), one child (Tx_A)
Parent: (M = 3) ephemeral keys
Amounts are illustrative.

### 16.1 Example Parameters

| Field                 | Value                                               | Notes          |
| --------------------- | --------------------------------------------------- | -------------- |
| Previous UTXO         | `txid_prev`: 1111...11, `vout_prev`: 0              | Protected coin |
| (value_{\text{prev}}) | 1.00000000 BTC                                      |                |
| Timelock              | (T_{lock} = 500000)                                 | Block height   |
| Keys                  | (K1_{\text{pub}}, K2_{\text{pub}}, K3_{\text{pub}}) | Ephemeral      |
| Secrets               | (H_{S1} = 0xf1f1...f1), (H_{S2} = 0xe2e2...e2)      | Hashes         |
| Timing                | (\Delta t_1 = 20) s, (\Delta t_2 = 2) s             | Targets        |
| Fees                  | (Tx_P) fee 5,000 sat; (Tx_A) fee 3,000 sat          | Example        |

### 16.2 Parent RedeemScript and scriptPubKey

Parent RedeemScript_P:

```text
500000 OP_CHECKLOCKTIMEVERIFY
OP_DROP
OP_SHA256 0xH_S1 OP_EQUALVERIFY
3 0xK1_pub 0xK2_pub 0xK3_pub 3 OP_CHECKMULTISIG
```

Parent scriptPubKey (P2WSH):

```text
OP_0 <32-byte sha256(RedeemScript_P)>
```

### 16.3 Child RedeemScript and scriptPubKey

Child RedeemScript_A:

```text
OP_SHA256 0xH_S2 OP_EQUALVERIFY
2 0xA1_pub 0xA2_pub 2 OP_CHECKMULTISIG
```

Child funding scriptPubKey (P2WSH):

```text
OP_0 <32-byte sha256(RedeemScript_A)>
```

### 16.4 Example Parent Transaction (Tx_P)

| Field    | Value          | Notes                                  |
| -------- | -------------- | -------------------------------------- |
| version  | 2              |                                        |
| locktime | 500000         | Enables CLTV                           |
| vin      | `txid_prev:0`  | Spends protected UTXO                  |
| sequence | 0xfffffffe     | Non-final; enables locktime            |
| vout 0   | 0.99990000 BTC | Commits to (H_{S2})                    |
| vout 1   | 0.00009500 BTC | Change to (K_{\text{user}}) or similar |
| fee      | 0.00000500 BTC | 5,000 sat                              |

Parent witness:

```text
<empty>
<sig_K1>
<sig_K2>
<sig_K3>
<S1>
<RedeemScript_P>
```

### 16.5 Example Child Transaction (Tx_A)

Tx_A spends vout 0 of Tx_P.

| Field    | Value          | Notes                       |
| -------- | -------------- | --------------------------- |
| version  | 2              |                             |
| locktime | 0              | No extra CLTV               |
| vin      | `txid_TxP:0`   | Spends child funding output |
| sequence | 0xffffffff     | Final                       |
| vout 0   | 0.99987000 BTC | Final destination           |
| fee      | 0.00000300 BTC | 3,000 sat                   |

Child witness:

```text
<empty>
<sig_A1>
<sig_A2>
<S2>
<RedeemScript_A>
```

### 16.6 Combined CPFP Package

Tx_P and Tx_A form a CPFP package. Effective package feerate:

```math
f_{\text{pkg}} = \frac{\text{fee}_{Tx_P} + \text{fee}_{Tx_A}}
                      {\text{vsize}_{Tx_P} + \text{vsize}_{Tx_A}}
```

Choose fees so (f_{\text{pkg}}) is competitive in the target block’s fee distribution.

### 16.7 Example Broadcast Timeline (Informative)

Assume:

* (T_{lock} = 500000)
* (\Delta t_1 = 20) s
* (\Delta t_2 = 2) s
* Expected block 500000 around 14:10:00.

| Time     | Action                     | Outcome                                                                                                                          |
| -------- | -------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| 14:09:40 | Broadcast Tx_P with (S_1). | Parent valid in mempool; eligible at height ≥ 500000.                                                                            |
| 14:09:58 | Broadcast Tx_A with (S_2). | Tx_A becomes valid; relationship between Tx_P and Tx_A is now visible. Remaining exposure window (\Delta t_2 \approx 2) seconds. |
| 14:10:00 | Block 500000 mined.        | If package is included, window for rival spend closes.                                                                           |

### 16.8 Notes for Implementers

* Scripts must be correctly serialised to Script bytecode.
* All P2WSH scriptPubKeys are of the form `OP_0 <32-byte SHA256(script)>`.
* Signatures should use `SIGHASH_ALL` unless policy requires otherwise.
* Production implementations should enforce ephemeral key destruction and must not reuse ephemeral keys across spends.

---

If you find this work useful and want to support it, you can do so here:

`bc1q380874ggwuavgldrsyqzzn9zmvvldkrs8aygkw`
