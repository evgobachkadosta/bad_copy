# Architecture Document: Pure ZK Anonymous Voting System

This document outlines the architecture for our zero-knowledge (ZK) voting system. It relies entirely on zk-SNARKs (specifically via the Rust `arkworks` ecosystem compiled to WASM) to guarantee voter eligibility, anonymity, and double-vote prevention. 

To maintain scope for the hackathon, this design omits Homomorphic Encryption. Votes are submitted in plaintext alongside a cryptographic proof of eligibility.

---

## Core Cryptographic Primitives

### Block 1: Cryptographic Hash
A hash function takes an input and produces a fixed-size, one-way fingerprint. We use a ZK-friendly hash function like Poseidon.
* `Hash("secret_A") → "0x7f3a..."`
* **Property:** It is computationally infeasible to reverse `"0x7f3a..."` back to `"secret_A"`.

### Block 2: Commitment
A commitment acts as a sealed envelope containing the voter's secret. 
* `Commitment = Hash(secret)`
* The server stores this commitment to register the voter without knowing the secret.



### Block 3: Merkle Tree
A data structure that represents a large list of commitments as a single root hash.
* To prove a commitment exists in the tree, a voter only needs the sibling hashes (the "Merkle path") up to the root, not the entire list of voters.
* The election authority publishes one public `ROOT` hash representing all eligible voters.

### Block 4: Nullifier
A one-way fingerprint of the voter's secret used exclusively to prevent double voting.
* `Nullifier = Hash(secret + "nullify_domain")`
* It is deterministically tied to the secret, but mathematically unlinked from the Commitment. The server stores used nullifiers. If a duplicate is submitted, the vote is rejected.

---

## System Flow

The protocol executes across 4 phases: **Setup → Registration → Voting → Tally**

### Phase 1: Setup (System Initialization)
The system initializes the parameters required for the election and the ZK circuits.

1.  **Define Parameters:** Set the voting period and candidate options.
2.  **Initialize State:** Create an empty Merkle tree. `ROOT = Hash(0)`.
3.  **Generate ZK Keys:** Perform the trusted setup for the circuit to generate the Proving Key (used by clients) and Verification Key (used by the server/auditors).

### Phase 2: Voter Registration
This phase establishes the anonymity set.

1.  **Local Generation:** The voter's machine generates a cryptographically secure random number (`secret`). This never leaves the client.
2.  **Commitment Derivation:** The client computes `leaf = Hash(secret)`.
3.  **Registration:** The voter authenticates with the central authority and submits the `leaf`. 
4.  **State Update:** The authority inserts the `leaf` into the Merkle tree and updates the public `ROOT`. The voter receives their specific Merkle path.

### Phase 3: Voting (Pure ZK)
The voting period opens. All computation happens client-side in the browser via WASM.



**Step 1: Input Preparation**
* **Private Inputs (kept secret):** `secret`, `merkle_path`
* **Public Inputs (revealed to server):** `merkle_root`, `nullifier`, `vote` (e.g., $1$ for Yes, $0$ for No)

**Step 2: Proof Generation**
The client runs the ZK prover. The circuit enforces the following mathematical constraints:
1.  The `secret` hashes to a leaf that, combined with the `merkle_path`, perfectly reconstructs the public `merkle_root`.
2.  The `nullifier` is correctly derived from the `secret`.
3.  The `vote` is strictly a binary value: $vote \times (1 - vote) = 0$.

**Step 3: Submission**
The client sends the following payload to the server:
* `proof` (The ~200 byte zk-SNARK)
* `nullifier`
* `vote` (Plaintext integer)

### Phase 4: Verification and Tally
The server processes the incoming payloads in real-time.

**Step 1: Server Verification**
For each submission, the server executes three checks:
1.  `verify_proof(proof, public_inputs) == true`: Confirms the cryptography.
2.  `nullifier NOT IN spent_nullifiers_db`: Prevents double voting.
3.  `vote` matches the predefined valid options (redundant but safe).

**Step 2: Recording**
If all checks pass:
* The `nullifier` is inserted into the `spent_nullifiers_db`.
* The plaintext `vote` is added to the public tally database.

**Step 3: Public Auditability**
Because the system relies on public verification keys and a public Merkle root, anyone can audit the election by verifying the cryptographic proofs against the published list of nullifiers and the final vote count.
