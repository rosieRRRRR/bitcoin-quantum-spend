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

This specification is compatible with classical HD wallets; however, claims of enforced key destruction, non-replaceability, and operational quantum mitigation assume use of PQHD Custody (Baseline). Transactional or single-device profiles MUST NOT be considered sufficient for the guarantees described herein.

---

## 2. Abstract

This specification defines a Bitcoin spend pattern that:

* Enforces a one-time, non-replayable spend for a protected UTXO (modulo normal chain reorgs).
* Provides operational quantum resistance on the public Bitcoin network by shrinking and hardening the effective attack window.
* Obstructs mempool attackers by hiding critical structure and secrets until shortly before confirmation.
* Uses only existing Script primitives (CLTV, CSV, hashlocks, multisig).
* Requires no consensus changes.

When implemented using a PQHD wallet, this specification assumes PQHD Custody (Baseline) as the minimum conformance level; single-device or non-custodial profiles are insufficient for the security properties claimed.

The core security property is that the effective exposure window is shorter than the time an attacker needs to construct and propagate a rival transaction:

```math
Exposure_Window(Δt_eff) < Time_rival
```

where:

* Δt_eff is the time from when the relevant spend transaction(s) reveal their scripts and secrets (S1 and/or S2) until confirmation of the intended spend(s).
* Time_rival includes:

  * classical overhead to parse, construct, and propagate a rival transaction, and
  * time to recover multiple independent private keys, potentially using quantum resources.

The construction uses:

* OTQH-M: an M-of-M multisig over independent ephemeral keys that are destroyed after signing.
* TS-MRC: a two-secret structure where the spend of the OTQH-M output is gated by secret S1 and downstream spend(s) are gated by secret S2, with linkage and script contents hidden until reveal.

Optional:

* HGSP: a CSV-based cooldown or spend delay for downstream robustness (not required for the core quantum mitigation).

In this specification, “operational quantum resistance” refers to time-bounded attack mitigation and elimination of signing authority during active spends, not cryptographic resistance to quantum key recovery.

The security guarantees are operational and statistical. This is an engineered mitigation against mempool replacement attacks under realistic assumptions, not a formal cryptographic proof.

---

## 2.1 Additional Security Context (PQHD Integration)

Claims of enforced non-retention, non-replaceability, and operational quantum mitigation in this specification assume use of **PQHD Custody (Baseline)**. When paired with PQHD, custody authorisation via the unified custody predicate, deterministic policy evaluation, and runtime integrity MUST evaluate to true **before** any OTQH-M + TS-MRC execution-time hardening is generated or applied.

PQHD-enforced **non-retention-by-construction** for ephemeral signing material (single-use, volatile-only, fail-closed) strengthens the time-bounded mitigation claims of this construction and ensures OTQH-M + TS-MRC is used strictly as execution-time hardening and MUST NOT be used as, or interpreted to create, an alternative authority path.

---

## 3. Quick Start (Informative)

For a minimal OTQH-M + TS-MRC deployment:

1. Ephemeral keys
   Generate 3 independent ephemeral keys: K1, K2, K3.

2. Secrets
   Generate two 256-bit secrets: S1, S2.

   ```math
   H_S1 = SHA256(S1),   H_S2 = SHA256(S2)
   ```

3. Funding transaction (Tx_F)
   Build a P2WSH output whose redeemScript enforces OTQH-M and commits to H_S1 (and optionally CLTV).
   Tx_F MUST NOT reveal S1 or S2.

4. OTQH-M spend transaction (Tx_S1)
   Spend the OTQH-M output created by Tx_F.
   Tx_S1 reveals S1 and creates one or more outputs committing to H_S2.

5. TS-MRC spend transaction(s) (Tx_S2)
   Spend the TS-MRC output(s).
   Tx_S2 reveals S2.

6. Broadcast targets
   Broadcast Tx_S1 near the intended CLTV satisfaction window if used.
   Broadcast Tx_S2 after Tx_S1 according to relay and miner behaviour.

   These are operational targets only. Implementations MUST NOT assume universal package relay or same-block inclusion.

7. Monitor
   Monitor mempool and chain state. Rebuild with fresh material on failure.

---

## 4. Motivation

Standard Bitcoin spends reveal public keys upon broadcast. A sufficiently capable quantum attacker monitoring the mempool could recover private keys and propagate a higher-fee rival transaction.

The OTQH-M + TS-MRC pattern reduces this risk by:

* Using multiple independent ephemeral keys.
* Destroying ephemeral private keys immediately after signing.
* Withholding secrets that link spend stages until late.
* Minimising the window between full revelation and confirmation.

---

## 5. Threat Model

### 5.1 Baseline Assumptions

* Full mempool visibility by the attacker.
* Ability to construct arbitrary transactions and pay higher fees.
* Possible access to quantum resources capable of breaking ECDSA keys in seconds or minutes.
* No control over consensus rules.
* No access to local wallet state or PQHD internal state.
* If PQHD is used, the wallet meets PQHD Custody (Baseline), including multi-device quorum and mandatory runtime attestation.

### 5.2 Network and Topology Assumptions

* No privileged miner access.
* Non-zero propagation overhead.
* Standard miner fee-based selection behaviour.

### 5.3 Quantum Capability Assumptions

* Bounded concurrency K of quantum key recovery.
* Time per ECDSA break T_Q.

```math
T_keys ≈ ceil(M / K) × T_Q
```

### 5.4 Defended Against

* Mempool replacement after spend broadcast.
* Fee-sniping front-running.
* Early inference of downstream policy.

### 5.5 Out of Scope

* Pre-signing compromise.
* Near-instantaneous quantum breaks.
* Miner collusion with large-scale quantum capacity.
* Operator error.

---

## 6. Design Goals

* Single effective spend per protected UTXO.
* Operational quantum mitigation.
* Mempool obstruction via delayed disclosure.
* No consensus changes.
* Wallet-agnostic with strengthened guarantees under PQHD Custody (Baseline).
* Explicit operational procedures.

---

## 7. Components

### 7.1 OTQH-M

An M-of-M multisig using ephemeral keys:

```math
M <K1_pub> <K2_pub> … <KM_pub> M OP_CHECKMULTISIG
```

Optional additions:

* OP_CHECKLOCKTIMEVERIFY over T_lock.
* Hashlock over H_S1.

### 7.2 TS-MRC

Two independent secrets:

* S1 gates the OTQH-M spend.
* S2 gates downstream spends.

### 7.3 Visibility Properties

Hidden until reveal:

* S2
* Downstream redeemScripts
* Explicit linkage

Visible:

* Transaction graph topology
* Fee signals

### 7.4 HGSP (Optional)

CSV-based cooldowns for downstream policies.

### 7.5 Timing Model

Δt_1 and Δt_2 are operational targets only and not validity conditions.

---

## 8. Core Construction

### 8.1 Keys and Secrets

Ephemeral keys K1 … KM.
Secrets S1, S2.

```math
H_S1 = SHA256(S1),   H_S2 = SHA256(S2)
```

### 8.2 Ephemeral Key Lifecycle

Where PQHD is used, PQHD Custody (Baseline) is required.

| Phase       | Action                            |
| ----------- | --------------------------------- |
| Generation  | Create ephemeral keys             |
| Signing     | Sign Tx_S1                        |
| Destruction | Delete all ephemeral private keys |
| Broadcast   | Broadcast spend transactions      |

### 8.3 Funding Transaction (Tx_F)

Creates the OTQH-M P2WSH output.
MUST NOT reveal S1 or S2.

### 8.4 OTQH-M Spend Transaction (Tx_S1)

RedeemScript:

```text
<T_lock> OP_CHECKLOCKTIMEVERIFY
OP_DROP
OP_SHA256 <H_S1> OP_EQUALVERIFY
M <K1_pub> <K2_pub> ... <KM_pub> M OP_CHECKMULTISIG
```

Witness:

```text
OP_0
<sig_K1>
<sig_K2>
...
<sig_KM>
<S1>
<RedeemScript_OTQH>
```

If CLTV is used, nLockTime ≥ T_lock and nSequence < 0xFFFFFFFE.

### 8.5 TS-MRC Spend Transaction (Tx_S2)

Example redeemScript:

```text
OP_SHA256 <H_S2> OP_EQUALVERIFY
2 <A1_pub> <A2_pub> 2 OP_CHECKMULTISIG
```

Witness:

```text
OP_0
<sig_A1>
<sig_A2>
<S2>
<RedeemScript_TS>
```

### 8.6 Broadcast and Finality Rules

Implementations MUST NOT assume universal package relay, guaranteed transaction ordering, or same-block inclusion.

Fee selection influences inclusion probability but does not guarantee confirmation or ordering; all security claims in this specification assume adversarial fee competition.

Parent-only confirmation of Tx_S1 without the intended confirmation of corresponding Tx_S2 transactions materially enlarges the exposure window beyond the modeled security bounds and invalidates the intended mempool hardening guarantees. Such an outcome is considered a catastrophic failure mode for this construction.

---

### 8.7 Note on Ephemeral Key Destruction

The instruction to irrecoverably delete the ephemeral private keys (K1 to KM) is an operational security mandate for compliant signing runtimes, as defined by PQHD Custody (Baseline) requirements, not a cryptographic assertion of key impossibility.

The corresponding public keys remain permanently visible on-chain and are theoretically vulnerable to recovery via quantum algorithms such as Shor’s algorithm. This specification does not claim cryptographic invulnerability of those public keys.

The security property relied upon by this construction is the absence of signing authority. Deletion of the ephemeral private keys ensures that, even if a quantum attacker were to recover a corresponding private key after Tx_S1 has confirmed, the recovered key material cannot be used to authorise a rival transaction because no compliant signing runtime retains the private key.

Accordingly, the effectiveness of ephemeral key destruction is evaluated in terms of attack time budget and authority elimination, not long-term secrecy. This clarification is consistent with the time-bounded security model and the operational quantum mitigation claims made elsewhere in this specification.


---

## 9. Security Analysis

Replacement requires recovery of all M ephemeral keys within Δt_eff.

```math
Time_rival = T_overhead + ceil(M / K) × T_Q
```

---

## 10. Wallet Considerations

### 10.1 Scope Limitation: Long-Term UTXO Protection

This construction does not provide long-term cryptographic protection for unspent outputs. Public keys and scripts committed on-chain remain theoretically vulnerable to future cryptographic advances, including quantum key recovery.

The security guarantees of this specification are limited to mitigating quantum-assisted replacement and mempool-based attacks during active spend execution. Protection is achieved by reducing the effective exposure window and eliminating signing authority after use, not by rendering on-chain data permanently secure.

Long-term protection of dormant UTXOs is explicitly out of scope for this specification.

---

### 10.2 PQHD Wallets

All PQHD references in this specification refer to PQHD Custody (Baseline).

---

## Annex A — Script Templates

OTQH-M redeemScript:

```text
<T_lock> OP_CHECKLOCKTIMEVERIFY
OP_DROP
OP_SHA256 <H_S1> OP_EQUALVERIFY
M <K1_pub> <K2_pub> ... <KM_pub> M OP_CHECKMULTISIG
```

TS-MRC redeemScript:

```text
OP_SHA256 <H_S2> OP_EQUALVERIFY
<CHILD_POLICY>
```

---

## Annex B — Pseudocode

```python
def sign_spend_otqh(tx_s1, keys, S1, script):
    sigs = [sign(tx_s1, k) for k in keys]
    for i in range(len(keys)):
        keys[i] = None
    witness = ["\x00"] + sigs + [S1, script]
    tx_s1.witness.append(witness)
```

---

## Annex C — Parameter Recommendations

* M = 2 minimum, M = 3 recommended.
* Timing is an operational policy.
* Fee strategy must be competitive.

---

## Annex D — PQHD Integration Note

OTQH-M + TS-MRC assumes PQHD Custody (Baseline). Transactional or single-device PQHD profiles are out of scope. Enterprise PQHD deployments may add governance but are not required for correctness.

---

If you find this work useful and want to support it, you can do so here:

bc1q380874ggwuavgldrsyqzzn9zmvvldkrs8aygkw
