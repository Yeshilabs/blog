---
title: "ZK state channels on Mina Protocol"
pubDate: 2024-12-29
description: " A Formal Specification of Zero-Knowledge State Channels for Mina Protocol"
author: "Yofi"
---
# z2zlib: A Formal Specification of Zero-Knowledge State Channels for Mina Protocol

## Overview

This specification defines a zero-knowledge state channel construction for the Mina Protocol. It allows two participants to run an off-chain state machine, advancing the state optimistically without blockchain interaction. Disputes are resolved by submitting zero-knowledge proofs on-chain to confirm or refute the validity of disputed state transitions. The design leverages Mina’s recursive proofs and efficient verification, ensuring minimal on-chain overhead and strong security guarantees.

## Formal Notation and Definitions

### Participants and Keys

Let $\mathcal{P} = \{A, B\}$ be the set of channel participants:
- $A$: Alice
- $B$: Bob

Each participant $p \in \mathcal{P}$ has:
- $(K_p, K_p^{-1})$: Public/private key pair suitable for signatures and on-chain identity.
- $\sigma_p : \mathcal{M} \to \Sigma$: Signature function, producing a signature $s$ on message $m$.
- $Ver_p : \mathcal{M} \times \Sigma \to \{0,1\}$: Signature verification function.

### State Representation

The channel maintains a sequence of states:
- $S_i \in \mathcal{S}$: The state at step $i$.
- Each $S_i$ is represented on-chain by a hash $h(S_i) \in \mathbb{F}$, where $h: \mathcal{S} \rightarrow \mathbb{F}$ is a collision-resistant hash (e.g., Poseidon).

A move (or action) $m_i$ transforms $S_i$ into $S_{i+1}$:
- $m_i: \mathcal{S} \to \mathcal{S}$ is the state transition function at step $i$.
- $T : \mathcal{S} \times \mathcal{S} \to \{0,1\}$: A predicate for validating transitions. $T(S_i, S_{i+1})=1$ iff $S_{i+1}=m_i(S_i)$ is a valid move according to the application’s rules.

### Turn Taking

A turn mapping function $\phi : \mathbb{Z}_2 \to \{K_A, K_B\}$ defines whose turn it is at step $i$:

$$φ(0) = K_A$$
$$φ(1) = K_B$$

For a generalization, this can be extended to more complex turn orderings encoded into the state or the circuit.

### Zero-Knowledge Proofs

We use zero-knowledge proofs for:
- **State Transition Proof ($\pi_{st}$)**: Proves that a given transition from $S_i$ to $S_{i+1}$ is valid without revealing internal state.
- **Fraud Proof ($\pi_{fraud}$)**: Proves that a claimed transition is invalid.
- **Final Proof ($\pi_{final}$)**: Aggregates and proves that a final state $S_f$ is the result of a series of valid transitions from $S_0$ to $S_f$.

We assume primitives:
- $ZK.Prove(\phi) \to \pi$: Given a statement $\phi$, produce a proof $\pi$.
- $ZK.Verify(\phi, \pi) \to \{0,1\}$: Verify that $\pi$ proves $\phi$.

### Settlement Contract

A smart contract $SC : (\mathcal{S}, \Pi) \to \{0,1\}$ manages disputes and finalization:
- `SC.settle(S_f, π_{final})`: Finalize the channel state if $π_{final}$ proves all transitions were valid.
- `SC.submit(S_i, π_{fraud})`: Resolve disputes by verifying a proof of invalidity for a proposed transition.

The contract stores minimal data (e.g., the latest agreed-upon state hash), ensuring scalability.

---

## Protocol Specification

### 1. Setup Phase

1. **Key Exchange**:
   
   $$A → B : K_A$$
   $$B → A : K_B$$

   Both parties know each other’s public keys.

2. **Initial State Agreement**:
   - Alice proposes initial state $S_0$:
     
     $$M_0^A = (0, h(S_0), σ_A(0 || h(S_0)))$$
     $$A → B : M_0^A$$
     
   - Bob verifies and countersigns:
     
     $$Ver_A(0 || h(S_0), σ_A(...)) = 1$$
     $$M_0^B = (M_0^A, σ_B(M_0^A))$$
     $$B → A : M_0^B$$
     
   
   Now both agree on $S_0$ and its hash $h(S_0)$.

## 2. Program Execution

During the program’s execution phase, the state channel progresses off-chain using a combination of signatures and, when required, zero-knowledge proofs. The protocol supports two main execution flows:

1. **Optimistic Execution (Signatures-Only)**: When no private inputs or complex verifications are required, transitions are performed and verified using digital signatures alone.

2. **ZK Proof-Based Execution**: When a state transition depends on private inputs or requires complex verification, a zero-knowledge proof of the state transition is generated and verified off-chain before acceptance.

If disagreements arise over a proposed transition, participants can resort to on-chain dispute resolution using fraud proofs.


### 2.1 Optimistic Execution (Signatures-Only)

This is the simplest scenario, relying solely on signatures and local verification of state transitions:

- Suppose at state $S_0$, it is Bob’s turn to act. Bob computes:
  
    $$S_1 = m_0(S_0)$$
  
  and constructs:
  
  $$M_0 = (0, S_1, m_0, \sigma_B(0 \Vert S_1 \Vert m_0))$$
  
  Bob sends $M_0$ to Alice.

- Upon receiving $M_0$, Alice verifies:
  - $\sigma_B$ is valid.
  - $T(S_0, S_1)=1$, ensuring $m_0$ is a legal and correct move.
  
  If valid, Alice proceeds with her move:
  
  $$S_2 = m_1(S_1)$$
  
  She then sends:
  
  $$M_1 = (1, S_2, m_1, \sigma_A(1 \Vert S_2 \Vert m_1))$$
  
  to Bob.

- This pattern continues optimistically. If all moves are straightforward and verifiable without revealing any hidden data, no zero-knowledge proofs are necessary at this stage.

  
### 2.2 ZK Proof-Based Execution (Private Inputs)

When a move requires private inputs—e.g., secret randomness, confidential financial data, or hidden game elements—a signature alone is insufficient to assure the other party of correctness. In this case, the participant generating the move also produces a zero-knowledge state transition proof $\pi_{st}$ that the move is valid without revealing the secret input.

- Suppose it is Alice’s turn to move from $S_1$ to $S_2$ using a move $m_1$ that depends on secret information (e.g., a hidden card in a game):
  
  $$S_2 = m_1(S_1)$$
  
  
- Alice generates a zero-knowledge proof:
  
  $$\pi_{st} = ZK.Prove(T(S_1, S_2)=1)$$
  
  This proof ensures that $S_2$ is a valid next state from $S_1$ given the confidential move $m_1$, without exposing $m_1$’s private details.

- Alice sends:
  
  $$M_1 = (1, S_2, \pi_{st}, \sigma_A(1 \Vert S_2 \Vert \pi_{st}))$$
  
  to Bob. Here, $M_1$ includes the zero-knowledge proof $\pi_{st}$ instead of directly disclosing the private input.

- Bob verifies:
  - $Ver_A(1 \Vert S_2 \Vert \pi_{st}, \sigma_A(...))=1$
  - $ZK.Verify(T(S_1, S_2)=1, \pi_{st})=1$

  If the verification passes, Bob is convinced that the transition is valid, even though he does not know the private inputs that produced it. Bob then proceeds with his turn.

This approach preserves privacy and trust: complex or secret-dependent transitions can be validated without revealing secrets.

  
### 2.3 Fraud Handling (On-Chain Dispute Resolution)

If at any point a participant suspects fraud—either in the optimistic or ZK proof-based approach—their remedy is to produce a succinct zero-knowledge fraud proof and submit it on-chain:

- The accusing participant constructs:
  
  $$\pi_{fraud} = ZK.Prove(T(S_i, S_{i+1})=0)$$
  
  demonstrating that the claimed transition is invalid.

- Submit to the settlement contract:
  
  $$SC.submit(S_i, \pi_{fraud})$$
  
- If $ZK.Verify(T(S_i, S_{i+1})=0, \pi_{fraud})=1$, the contract settles the dispute in favor of the accuser, penalizing the cheater. Thus, on-chain dispute resolution ensures that no fraudulent transition can be enforced.


**Summary of Execution Scenarios**:

- **Optimistic (No Private Input)**: Transitions rely on signatures. Participants trust each other’s verification, resorting to on-chain disputes only if necessary.
- **ZK Proof-Based (Private Input)**: If a transition involves hidden data, a zero-knowledge proof is used off-chain to guarantee correctness. The other party verifies the proof rather than the underlying data.
- **Fraud Handling**: At any point, if a participant believes the other is cheating, they can produce a fraud proof and finalize the dispute on-chain.


### 3. Settlement Phase

When no more updates are desired:

1. **Final State Agreement**:
   Both participants finalize with a last message:
   
   $$M_f = (f, h(S_f), h(m_{f-1}), σ_p(f || h(S_f) || h(m_{f-1})))$$
   $$q \space \text{countersigns:} σ_q(M_f)$$
   
   They now have mutually signed final state data.

2. **On-chain Settlement**:
   They can either:
   - Just submit signatures if they trust each other’s previous steps.
   - Or provide a recursive proof $π_{final}$ that all transitions from $S_0$ to $S_f$ were valid:
     ```
     SC.settle(S_f, π_{final})
     ```
   If $ZK.Verify(\bigwedge_{i=1}^f T(S_{i-1}, S_i)=1, π_{final})=1$, the contract finalizes and releases funds or concludes the channel.

---

## Circuit Specifications

### State Transition Circuit

```
    Circuit StateTransition {
        public inputs:
            prev_state_hash: Field,
            next_state_hash: Field,
            transition_index: Field

        private inputs:
            prev_state: State,
            next_state: State,
            move: Move

        constraints:
            hash(prev_state) == prev_state_hash
            hash(next_state) == next_state_hash
            is_valid_transition(prev_state, next_state, move) == true
            verify_turn(transition_index, prev_state.turn_mapping) == true
    }
```

This ensures that given a previous state and a move, the next state is correct and the right participant made the move.

### Fraud Proof Circuit

```
    Circuit FraudProof {
        public inputs:
            state_hash: Field,
            claimed_next_hash: Field

        private inputs:
            state: State,
            claimed_next: State,
            move: Move

        constraints:
            hash(state) == state_hash
            hash(claimed_next) == claimed_next_hash
            is_valid_transition(state, claimed_next, move) == false
    }
```

This proves a purported transition is invalid.

### Final Aggregated Proof Circuit 

A recursive circuit can combine multiple `StateTransition` proofs:
```
    Circuit FinalProof {
        public inputs:
            initial_state_hash: Field,
            final_state_hash: Field

        private inputs:
            transition_proofs: [π_{st,1}, π_{st,2}, ... π_{st,f}]

        constraints:
            // Recursively verify that each π_{st,i} is valid and chained correctly:
            check_chaining(initial_state_hash, final_state_hash, transition_proofs)
}
```

This yields a single $π_{final}$ proving the entire chain of states is valid.

---

## Security Properties

**Safety**:
- **State Validity**:  
  
  $$∀i>0: T(S_{i-1}, S_i) = 1$$
  
  Ensures no invalid states are accepted.

- **Authorization**:  
  
  $$∀i: Ver_{φ(i)}(M_i) = 1$$
  
  Only the designated participant at each turn can produce the valid signed state.

**Liveness**:
- **Progress**:  
  Honest participants can always produce the next state off-chain without going on-chain, ensuring responsiveness.
- **Finality**:  
  A final state can always be settled on-chain with a proof or mutual signatures.

**Privacy**:
- Internal states and moves remain off-chain. On-chain, only hashes and succinct proofs are revealed, preserving the privacy of the channel’s execution details.

---

## Implementation 

1. **Efficiency in Proof Generation**:
   - Use batching and recursion to reduce overhead.
   - Employ Mina’s SNARK-friendly hash functions (e.g., Poseidon).

2. **State Representation**:
   - Compact states (e.g., Merkle roots) to reduce proving complexity.
   - Efficiently encode moves so the circuit remains small.

3. **On-Chain Data Minimization**:
   - Store only hashes on-chain; never publish full states.
   - Post proofs only when disputes occur or for final settlement.

4. **Rational Incentives**:
   - Fraud proofs deter malicious behavior since cheating can be proved and penalized.
   - Low overhead ensures participants have little incentive to escalate on-chain unless necessary.

