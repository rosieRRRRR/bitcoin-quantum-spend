# BIP-360 Execution-Layer Hardening

**Quantum-Hardened Execution Patterns (QH-EP) with Optional EPOCH Spend Path**

Version: v2.0.0
Status: Advanced Specification
Author: rosiea
Date: December 2025
Licence: Apache License 2.0 — Copyright 2025 rosiea

---

## Summary

BIP-360-style script-only outputs reduce long-range, dormant-UTXO quantum exposure by removing the Taproot key path. This specification addresses a different surface: the execution window during a spend, when scripts and public keys are disclosed and the transaction resides in the public mempool.

This document defines Bitcoin-current-consensus execution hardening patterns that:

* deny pre-construction of valid competing spends prior to execution disclosure;
* increase attacker work during the broadcast-to-confirmation window;
* provide explicit, pre-constructed recovery paths for stalled spends;
* require no consensus changes and no new opcodes.

This work does not claim deterministic prevention of mempool replacement, miner inclusion guarantees, or on-chain post-quantum signatures.

---

## 1. Status

### 1.1 Scope

This specification defines optional execution hardening patterns for BIP-360-style script-only outputs implemented as Taproot/Tapscript spend paths.

It specifies:

* spend-time secret gating (denial of pre-construction);
* secret-keyed key ordering (work amplification against sequential targeting);
* authorization modes (ECC M-of-M and an experimental hash-based mode);
* witness construction rules;
* operational discipline (fee strategy, key independence, dry-wallet barrier);
* dual recovery model (timeout return and post-confirmation fallback);
* wallet behavior requirements and prohibitions to preserve the security model.

### 1.2 Consensus changes required

None.
All mechanisms herein are valid under current Bitcoin consensus rules (BIP-340/341/342).

---

## 2. Execution Scope

This specification applies only to:

* the execution-time, public mempool exposure window;
* spend construction and wallet/operator discipline immediately prior to broadcast;
* explicit, pre-constructed recovery paths.

This specification does not define custody authority, governance, or temporal oracle semantics.

---

## 3. Threat Model (Normative)

### 3.1 Assumptions

* The attacker can observe the public mempool and attempt replacement via higher-fee spends.
* During execution, scripts and authorization material may be disclosed.
* A quantum-capable attacker may accelerate recovery of ECC private keys during the broadcast-to-confirmation window.
* Bitcoin does not provide deterministic anti-replacement guarantees for general transactions.

### 3.2 Defended Against

This specification mitigates:

* pre-construction of competing spends prior to spend-time disclosure;
* replacement attempts after disclosure of execution material;
* quantum-assisted short-range key recovery within the exposure window;
* operational adversaries exploiting slow propagation and long exposure.

### 3.3 Out of Scope

This specification does not address:

* deterministic prevention of mempool replacement;
* miner-level adversaries or inclusion guarantees;
* covenant-grade destination binding without covenants;
* long-range dormant-UTXO exposure (addressed by BIP-360-style approaches);
* full post-quantum signature security under Bitcoin consensus today.

## 3.4 Multi-Input Fee Sponsorship and Replacement (Normative)

Bitcoin transactions may contain multiple inputs, each governed by independent scripts. QH-EP does not and cannot prevent an adversary who satisfies the authorization requirements of a protected input from adding additional, externally funded inputs to pay fees or to construct a competing transaction.

QH-EP’s security objective is to increase the cost, coordination, and time required to satisfy authorization during the execution window, and to enforce fail-closed wallet discipline. It does not provide consensus-level prevention of replacement, consensus-level restriction of fee funding sources, or deterministic exclusion of multi-input constructions.

---

## 4. Design Principles (Normative)

Mechanisms MUST be:

* consensus-valid on Bitcoin today;
* explicitly scoped to execution-time hardening;
* compatible with Taproot/Tapscript spend paths;
* conservative about guarantees;
* fail-closed at the wallet/policy layer where applicable.

Mechanisms MUST NOT be presented as “quantum-proof” or as deterministic prevention of mempool replacement.

---

## 5. Core Definitions (Normative)

### 5.1 Spend-Time Secret (S1)

`S1` is a random, one-time secret revealed only at execution.

* `S1` MUST be unique per spend attempt.
* `S1` MUST NOT be reused.
* `H_S1 = SHA256(S1)` MUST be committed in the tapleaf script.
* `S1` MUST be revealed in the witness only at execution.

### 5.2 Authorization Commitment (AUTH_COMMITMENT)

`AUTH_COMMITMENT` binds the authorization mode and parameters and MUST commit to:

* selected authorization mode (ECC or hash-based);
* all public parameters needed to validate the mode;
* key ordering and counts where applicable.

### 5.3 Optional Epoch Value (E)

`E` is an optional wallet-selected coordination value used for auditability and multi-signer alignment.

* Bitcoin Script cannot validate `E` as time.
* Wallet policy MAY select and validate `E` using a temporal authority.
* `E` MUST be canonically encoded (byte-stable across implementations).

Epoch binding further limits execution-layer replay by scoping authorization material to a wallet-defined temporal context.

Authorization material disclosed during one epoch becomes policy-invalid once the epoch advances, even if Script validity would otherwise permit reuse. This prevents deferred or cross-attempt reuse of previously observed execution material following failed, delayed, or abandoned spend attempts.

This mechanism does not prevent consensus-level re-broadcast of transactions. It prevents reuse of execution authorization across time or attempts by enforcing freshness at the wallet and policy layer.


### 5.4 Optional Context Commitment (H_ctx)

If `E` is used, define:

`H_ctx = SHA256("QHEP-CTX-V1" || S1 || E || AUTH_COMMITMENT)`

`"QHEP-CTX-V1"` is a domain separator.

---

## 6. Normative Mechanisms

### 6.1 Denial of Pre-Construction (S1 Gate) (Normative)

All spends implementing this specification MUST include a spend-time secret gate:

`OP_SHA256 <H_S1> OP_EQUALVERIFY`

Security objective:

* prevent third parties from constructing a valid competing spend prior to `S1` disclosure.

### 6.2 Optional Context Binding (EPOCH-Style Commitment) (Normative–Optional)

If `E` is used, the script MUST verify `H_ctx` immediately after `S1` verification:

`OP_SHA256 <H_ctx> OP_EQUALVERIFY`

Requirements:

* `E` MUST be canonically encoded.
* `AUTH_COMMITMENT` MUST bind the authorization mode and parameters.
* Wallets MUST refuse signing unless local policy for `E` is satisfied.

Security objective:

* tamper-evident binding of intended execution context without claiming Script oracle properties.

### 6.3 Secret-Keyed Ordering (Normative)

If a script uses multiple public keys for authorization, the ordering of those keys MUST be derived from a one-time secret `S1` revealed only at execution.

Define:

* `ordering_secret = SHA256("QHEP-Key-Order" || S1)`
* `tag_i = HMAC-SHA256(ordering_secret, pubkey_i_bytes)`
* `ordered_keys = sort_by(tag_i)`

The script MUST enforce `S1` via the `S1` gate prior to any signature checks.

Security objective:

* prevent predictable sequential targeting;
* force parallel recovery effort and reduce the value of pre-targeting.

---

## 7. Authorization Modes (Normative)

### 7.1 Production Mode (ECC, M-of-M) (Normative)

Authorization uses M-of-M Schnorr checks via `OP_CHECKSIGADD`:

```
<K1_xonly> OP_CHECKSIG
<K2_xonly> OP_CHECKSIGADD
...
<KM_xonly> OP_CHECKSIGADD
<M> OP_NUMEQUAL
```

Requirements:

* `AUTH_COMMITMENT` MUST bind the ordered x-only pubkeys and M-of-M parameters.
* Keys MUST be independent (see §10.2).

Properties:

* compatible with Taproot/Tapscript tooling;
* increases attacker workload proportional to `M`;
* ECC is exposed during execution.

### 7.2 Research Mode (Hash-Based Authorization) (Normative–Experimental)

Replace ECC signatures with SHA-256 preimage verification:

```
OP_SHA256 <H_qpk> OP_EQUALVERIFY
OP_DUP OP_SHA256 <expected_hash_i> OP_EQUALVERIFY OP_DROP
...
OP_TRUE
```

Requirements:

* `AUTH_COMMITMENT` MUST bind `H_qpk` and the required preimage set shape.
* Implementations MUST treat this as research-grade due to witness size and fee overhead.

Properties:

* Removes ECC keys and signatures from the execution window;
* Shifts security to SHA-256 preimage hardness (Grover-bounded);
* This mode is intended for research and controlled deployments due to witness size, fee overhead, and one-time-use constraints. Once hash preimages are revealed in the witness, they are irreversibly public and MUST be treated as permanently burned. Implementations MUST NOT assume RBF-style fee adjustment is available for hash-based spends and SHOULD require conservative fee selection and an explicit, pre-constructed recovery path.

---

## 8. Script Templates (Normative)

### 8.1 Primary Leaf Prefix (Normative)

All compliant primary tapleaf scripts MUST begin with:

```
OP_SHA256 <H_S1> OP_EQUALVERIFY
[ OPTIONAL: OP_SHA256 <H_ctx> OP_EQUALVERIFY ]
# Authorization follows
```

### 8.2 Example Primary Leaf (ECC, M-of-M) (Normative)

```
OP_SHA256 <H_S1> OP_EQUALVERIFY
OP_SHA256 <H_ctx> OP_EQUALVERIFY
<K1_xonly> OP_CHECKSIG
<K2_xonly> OP_CHECKSIGADD
...
<KM_xonly> OP_CHECKSIGADD
<M> OP_NUMEQUAL
```

---

## 9. Witness Stack Construction (Normative)

### 9.1 Witness Ordering Rule (Normative)

For `OP_CHECKSIGADD` scripts, signatures MUST be pushed to the witness stack in reverse order of the public keys as they appear in the script.

### 9.2 Primary Spend Witness Stack (Normative)

Witness stack (top to bottom):

```
[ sig_M ] ... [ sig_1 ]
[ E (optional) ]
[ S1 ]
[ script ]
[ control_block ]
```

Requirements:

* `S1` MUST satisfy `H_S1`.
* If present, `E` MUST match the encoding used in `H_ctx`.
* Authorization material MUST match `AUTH_COMMITMENT`.

---

## 10. Operational Discipline (Normative)

### 10.1 Dry-Wallet Barrier (Normative)

All signing wallets used in execution hardening MUST maintain a zero on-chain balance.

Fee funding MUST occur from a separate fee wallet that is not used as an execution signer.

Security objective:

* prevent attacker pre-staging of fee-funded replacement transactions from the same signing wallet set;
* force any attacker who wants to spend from signing wallets to create observable funding dependencies.

### 10.2 Key Independence (Normative)

Signers MUST use truly independent signing keys, derived from independent entropy sources and, where practical, separate devices and administrative domains.

### 10.3 Broadcast and Fee Discipline (Normative)

Wallets and operators MUST:

* use a high initial feerate consistent with the desired confirmation target;
* maintain CPFP readiness (including a prepared strategy for fee acceleration);
* monitor for conflicting spends and treat conflicts as a security-relevant event.

Implementations MUST NOT assume universal support for package relay or special relay paths.

#### 10.3.1 Pre-Validity Fee Signalling (Normative)

When execution hardening employs delayed validity (for example via locktime, spend-time secrets, or context commitments), wallets MUST construct the primary spend with a feerate that is economically attractive at first relay.

Miners and relay nodes may observe and economically evaluate a transaction prior to it becoming consensus-valid. Transactions with high feerates may be retained as desirable candidates. Once validity conditions are satisfied, such transactions are more likely to be incorporated promptly into block templates.

This mechanism does not provide deterministic inclusion guarantees and MUST NOT be interpreted as miner cooperation or enforcement. Its purpose is to compress the effective public-mempool exposure window by ensuring economic visibility precedes execution validity.

### 10.4 Signature Binding (Normative)

Production Mode signatures MUST use `SIGHASH_ALL` unless an implementation profile explicitly and normatively specifies an alternative binding mode and demonstrates that the binding remains consistent with the threat model.

---

## 11. Dual Recovery Model (Normative)

### 11.1 Unmined Timeout Return (Normative)

This recovery path activates if the primary spend is broadcast but fails to confirm within the expected timeframe.

Example pattern:

```
<H_timeout> OP_CHECKLOCKTIMEVERIFY OP_DROP
<R1_pub> OP_CHECKSIG
<R2_pub> OP_CHECKSIGADD
...
<RR_pub> OP_CHECKSIGADD
<R> OP_NUMEQUAL
```

Requirements:

* Recovery keys MUST be independent from execution keys.
* `H_timeout` MUST be selected conservatively to avoid premature recovery races.
* This leaf MUST be explicitly committed at output creation time.

### 11.2 Post-Confirmation Fallback (Normative–Optional)

If used, this path MUST be:

* explicitly pre-constructed and policy-approved;
* bound to recovery conditions that do not create an alternative authority path;
* designed so that confirmation of the primary spend clearly transitions control to the fallback policy domain.

### 11.3 Recovery Timing Parameters (Normative)

Implementations MUST define and enforce conservative timing parameters, including:

* `H_timeout` for the unmined timeout return leaf;
* any downstream fallback activation constraints.

### 11.4 Recovery Key Independence (Normative)

Recovery keys MUST be derived and stored independently from execution keys.

---

## 12. Wallet Behavior Requirements (Normative)

### 12.1 State Tracking (Normative)

Wallets MUST explicitly track:

1. whether the primary spend has been broadcast;
2. whether the primary spend has confirmed;
3. current block height relative to `H_timeout` and any fallback parameters.

### 12.2 Alternative Spend Prohibition (Normative)

Once the primary spend transaction has been broadcast, the wallet MUST NOT create ad-hoc alternative spends of the same UTXO outside the defined recovery paths.

### 12.3 Failure Handling (Normative)

On any failure:

* discard all ephemeral execution material used for the attempt (including `S1`);
* invalidate the PSBT for that attempt;
* re-evaluate policy and re-construct under fresh randomness prior to any reattempt.

Partial reuse is forbidden.

For hash-based authorization spends, failure handling MUST assume that any revealed preimages are burned regardless of confirmation outcome; reattempts MUST use fresh authorization material and MUST rely only on explicitly pre-constructed recovery paths where applicable.


## 13. Quantitative Condition (Informative)

QH-EP provides benefit when:

`ceil(M / K) × T_Q > Δt_eff`

---

## 14. Non-Goals (Normative)

This specification does not:

* provide deterministic prevention of mempool replacement;
* guarantee miner inclusion;
* provide covenants or destination enforcement;
* claim on-chain post-quantum signature security.

---

## 15. Relationship to BIP-360 (Informative)

BIP-360-style script-only outputs reduce long-range quantum exposure at rest by removing the key path. This specification complements that approach by hardening the execution window during spends, where disclosure and mempool exposure remain unavoidable.

At-rest protection and execution-time hardening are distinct and non-substitutable.

---

## Annex A — Further Execution Hardening Enhancements (Informative)

This annex documents optional, non-consensus mechanisms that MAY be layered on top of the execution-layer hardening patterns defined in this specification.

The mechanisms described here:

* are **not required** for BIP-360 compatibility;
* do **not** alter Bitcoin consensus behavior;
* do **not** redefine custody authority;
* do **not** attempt to address at-rest key exposure, which is the sole concern of the BIP-360 output type itself;
* MUST NOT be interpreted as providing deterministic guarantees.

Nothing in this annex modifies or weakens the normative requirements of the main specification.

---

### A.1 Epoch-Bound Execution (EPOCH Protocol) (Informative)

The **EPOCH Protocol** is an execution-time coordination mechanism that binds spend authorization to a wallet-selected epoch value, enforced entirely by wallet policy and committed on-chain for tamper-evident integrity.

When layered on top of QH-EP, EPOCH provides:

* denial of pre-construction via spend-time secrets;
* explicit binding of execution context through a context commitment;
* coordination and auditability across distributed or threshold signers;
* replay collapse by scoping authorization material to a wallet-defined temporal context.

Bitcoin Script **cannot** validate epochs or clocks. Epoch enforcement is strictly wallet-side and MUST NOT be interpreted as miner enforcement, consensus enforcement, or deterministic inclusion control.

EPOCH MAY be layered on top of QH-EP execution patterns where wallet architectures support policy-enforced execution contexts.

Reference specification and implementations:

* **EPOCH / Epoch Clock repository**
  [https://github.com/rosieRRRRR/epoch-clock](https://github.com/rosieRRRRR/epoch-clock)

---

### A.2 Custody-Level Authority Systems (PQHD) (Informative)

**Post-Quantum Hierarchical Deterministic Custody (PQHD)** is a custody-layer authority system that governs *whether* signing is permitted, independent of *how* execution hardening is performed.

PQHD provides:

* removal of spending authority from classical private key possession;
* unified custody predicates incorporating time, consent, deterministic policy evaluation, device integrity, quorum correctness, ledger continuity, and PSBT equivalence;
* explicit separation between custody authorization and execution mechanics.

PQHD operates **above** execution-layer hardening mechanisms such as QH-EP and EPOCH.
Possession of execution keys alone remains insufficient to authorize a spend under PQHD-style custody systems.

QH-EP and EPOCH MAY be used as execution-time hardening layers **only after** custody authorization evaluates to true.

Reference specification and implementations:

* **PQHD repository**
  [https://github.com/rosieRRRRR/pqhd](https://github.com/rosieRRRRR/pqhd)

---

### A.3 Layered Quantum Threat Mitigation Model (Informative)

BIP-360 defines only the first layer below; subsequent layers are strictly wallet-side and non-consensus.

Advanced wallet architectures MAY adopt a layered security model composed of:

1. **At-rest protection** — BIP-360-style script-only outputs
   Reduces long-range dormant UTXO exposure by removing the Taproot key path.

2. **Execution-time hardening** — QH-EP (this specification)
   Reduces short-range exposure during public mempool broadcast by denying pre-construction, increasing attacker work, and compressing the effective exposure window.

3. **Custody authority** — PQHD-class systems
   Removes spending authority from classical keys entirely through wallet-enforced predicates.

These layers are complementary and non-substitutable. No single layer addresses the full quantum threat surface alone.

---

### A.4 Research Directions (Non-Normative)

The following areas are active research directions and are **not** part of this specification:

* epoch chaining across staged or multi-phase spends;
* hash-only execution authorization at larger scales;
* deterministic execution envelopes and wallet-enforced execution state machines;
* integration with post-quantum signature schemes when supported by Bitcoin consensus.

These directions MUST NOT be presented as guarantees under current Bitcoin consensus rules.

## **Acknowledgements**

This work reflects extensive discussion, review, and iteration within the Bitcoin protocol and cryptography community.

The authors acknowledge **Ethan Heilman**, **Hunter Beast**, and **Isabel Foxen Duke**, whose recent clean-sheet rewrite of BIP-360 clarified scope, terminology, and design intent in response to community feedback. The decision to undertake a full rewrite, rather than incremental revision, materially improved internal coherence and accurately framed BIP-360 as a conservative, consensus-compatible mitigation against potential future cryptanalytic risk.

BIP-360’s design is informed by foundational Bitcoin protocol research and engineering, including:

- **Satoshi Nakamoto**, for the original design of Bitcoin and its trust-minimised consensus model.
- **Pieter Wuille**, for Taproot, Tapscript, and the underlying script-path architecture that enables key-path removal.
- **Greg Maxwell**, for adversarial analysis, cryptographic conservatism, and long-range threat modelling.
- **Andrew Poelstra**, for formal reasoning about Bitcoin script, spend policies, and composability.
- **Jonas Nick**, **Tim Ruffing**, and contributors to Schnorr signatures and Taproot activation.

The authors also acknowledge reviewers and participants in Bitcoin protocol design discussions whose critiques and questions—particularly around naming, script extensibility, and future compatibility—helped sharpen the proposal. Notably, discussion around the **Pay-to-Tapscript-Hash (P2TSH)** naming highlights the community’s continued focus on long-term script evolution and semantic clarity.

This proposal does **not** introduce post-quantum signature schemes. Instead, it represents a measured, incremental step: removing the Taproot key path to reduce exposure should elliptic-curve cryptography be weakened in the future. Any errors, omissions, or remaining ambiguities are the responsibility of the authors.

