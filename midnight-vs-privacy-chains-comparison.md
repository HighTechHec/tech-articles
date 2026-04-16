# Midnight vs Other Privacy Chains: Architecture Comparison for Developers

Privacy-preserving smart contracts are evolving fast, and each major platform has made fundamentally different design decisions. If you're a developer choosing where to build, the differences matter more than the marketing. This comparison focuses on what actually affects your day-to-day experience: the programming model, how state is handled, how privacy is enforced, and what the ZK toolchain looks like.

We'll cover five platforms: Midnight, Aztec, Aleo, Mina, and Zcash. Each represents a different philosophy about the right way to build a privacy-first blockchain.

---

## Midnight: Compact + ZK Circuits + Dual Ledger

Midnight is IOG's privacy-focused blockchain, built as a partner chain to Cardano. Its central insight is that most applications need both public and private state — and the two should be first-class, explicitly separated.

### The Dual Ledger Model

Every Midnight dApp operates on two ledgers simultaneously:

- **Public ledger**: Smart contract state visible to all (balances, counters, public records)
- **Shielded ledger**: UTXO-based private state, visible only to the owning wallet

This dual-ledger design means you don't have to choose between "fully private" and "fully transparent." A compliance attestation contract can store a public flag ("this address is KYC'd") while keeping the underlying identity data private. The shielded UTXO model is borrowed from Zcash's Sapling but integrated directly into the smart contract layer.

### Compact: Purpose-Built for ZK Contracts

Midnight introduces a new language, Compact, specifically designed for this dual-ledger environment. Compact looks like a TypeScript subset but compiles to ZK circuits.

```compact
pragma language_version >= 1.0.0;

// Public contract state
export ledger counter: Counter;

// Circuit: proves a valid increment without revealing who called it
circuit increment(proof: ZkProof): [] {
  assert counter.current < 100;
  counter.increment(1);
}
```

Key properties of Compact:

- **Typed privacy**: Public state uses `ledger`, private state uses `witness`
- **Circuit compilation**: Every function is a ZK circuit that generates a proof
- **TypeScript integration**: The off-chain runtime is TypeScript/JavaScript, so the same team writes both sides
- **ZSwap**: Native token protocol for shielded transfers, built into the language

The developer experience is unusual. You write Compact for the on-chain circuit logic and TypeScript for the wallet/frontend. The Midnight SDK bridges them, handling proof generation transparently.

### Privacy Model

Midnight uses zk-SNARKs for proof generation. Transactions include a proof that the state transition is valid without revealing which UTXO was spent. The shielded ledger uses a Merkle tree of commitments (like Zcash's note commitment tree).

**Selective disclosure** is first-class: you can generate a proof that "I am in this allowlist" without revealing your identity, or "my balance is above X" without revealing the balance. This is encoded in the Compact type system.

### Summary

| Dimension | Midnight |
|-----------|----------|
| Language | Compact (custom, ZK-aware) |
| Privacy model | Dual ledger: public + shielded UTXO |
| ZK scheme | zk-SNARKs |
| State model | Ledger (public) + Witness (private) |
| Off-chain SDK | TypeScript |
| Consensus | Proof-of-stake (partner chain to Cardano) |

---

## Aztec: Noir + Hybrid UTXO

Aztec is an L2 on Ethereum that brings programmable privacy through its Noir language and hybrid execution model.

### Noir: Privacy-First, Ethereum-Native

Noir was designed to lower the barrier to ZK circuit development. It reads like Rust but compiles to circuits (currently targeting Barretenberg's PLONK backend).

```noir
use dep::std;

fn main(
    secret_input: Field,
    public_hash: pub Field
) {
    let computed = std::hash::pedersen_hash([secret_input]);
    assert(computed[0] == public_hash);
}
```

Noir's key advantage: it's backend-agnostic. The same Noir program can compile to PLONK, Groth16, or other proving systems. This portability means the ecosystem can upgrade its ZK backend without rewriting application code.

### Hybrid State: Notes vs Storage

Aztec's state model is fundamentally UTXO-based for private state (called "notes") and account-based for public state:

- **Private notes**: Encrypted, UTXO-like, owned by individual users
- **Public storage**: EVM-style key/value storage, visible on-chain
- **Aztec.nr contracts**: Written in Noir, can mix both state types

The duality creates complexity. When a private function needs to read public state, or vice versa, you must explicitly handle the transition with `enqueue_public_function_call`. This is more explicit than Midnight's model but also more complex.

```rust
// Aztec.nr (simplified)
contract Token {
    use dep::aztec::prelude::*;

    #[aztec(private)]
    fn transfer(to: AztecAddress, amount: Field) {
        // Private note consumption
        let sender_note = get_note(storage.balances, context.msg_sender());
        sender_note.assert_balance_sufficient(amount);
        // Enqueue public update
        context.call_public_function(
            context.this_address(),
            comptime { FunctionSelector::from_signature("_transfer_public((Field),(Field))") },
            [to.to_field(), amount]
        );
    }
}
```

### Privacy Model

Aztec uses recursive zk-SNARKs (Honk, a variant of PLONK). Transaction proofs are generated client-side and batched into an L2 rollup proof. The sequencer never sees private state — only encrypted notes.

Private/public bridging goes through the "portal" pattern: private functions can call public ones (enqueued, one-block delay), but public functions cannot call private ones.

### Developer Experience

Aztec's documentation and tooling are well-developed. The sandbox (local devnet) runs in Docker, and the workflow mirrors Hardhat for Ethereum developers. Noir has a growing standard library and syntax that Solidity developers find approachable.

| Dimension | Aztec |
|-----------|-------|
| Language | Noir |
| Privacy model | Encrypted notes (UTXO) + public storage |
| ZK scheme | Honk (PLONK variant), recursive |
| State model | Notes (private) + Storage (public) |
| Off-chain SDK | TypeScript (Aztec.js) |
| Consensus | Proof-of-stake L2 on Ethereum |

---

## Aleo: Leo + Records

Aleo takes a different approach: everything is private by default, with optional public disclosure. Its record model makes it conceptually closer to functional programming than either Midnight or Aztec.

### Leo: A Strongly-Typed ZK Language

Leo is Rust-inspired but simpler, designed specifically for Aleo's snarkVM. It compiles to Aleo Instructions (similar to assembly) and then to R1CS constraints for Groth16 proofs.

```leo
program token.aleo {
    record Token {
        owner: address,
        amount: u64,
    }

    transition mint(
        public receiver: address,
        public amount: u64,
    ) -> Token {
        return Token {
            owner: receiver,
            amount: amount,
        };
    }

    transition transfer(
        token: Token,
        receiver: address,
        amount: u64,
    ) -> (Token, Token) {
        let remaining: u64 = token.amount - amount;
        let sender_token: Token = Token { owner: token.owner, amount: remaining };
        let receiver_token: Token = Token { owner: receiver, amount: amount };
        return (sender_token, receiver_token);
    }
}
```

### Record Model

Aleo's fundamental data unit is the **record**: an encrypted object owned by an address. Records are consumed and produced by transitions (similar to Zcash's UTXO model but typed). There's no mutable shared state — transitions produce new records from old ones.

This immutability makes reasoning about privacy straightforward: you can prove you own a record and that a transition was valid, without revealing the record's contents. But it makes stateful applications (like AMMs or lending protocols) harder to implement, since you're working against the grain of the record model.

**Mappings** provide optional public storage:

```leo
program counter.aleo {
    mapping counts: address => u64;

    async transition increment() -> Future {
        return finalize_increment(self.caller);
    }

    async function finalize_increment(addr: address) {
        let current: u64 = counts.get_or_use(addr, 0u64);
        counts.set(addr, current + 1u64);
    }
}
```

The `async` transition + `finalize` pattern handles the public state update in a separate phase, making the privacy/transparency boundary explicit.

### Privacy Model

Aleo uses Groth16 proofs on a BLS12-377 curve. Proof generation happens client-side via snarkVM. The record commitment scheme hides the record contents while proving ownership.

**Private vs. public**: Leo has `private` (encrypted, off-chain) and `public` fields/parameters. Public fields appear in the transaction on-chain; private ones do not.

| Dimension | Aleo |
|-----------|------|
| Language | Leo |
| Privacy model | Records (private UTXO) + Mappings (public) |
| ZK scheme | Groth16 |
| State model | Records (consumed/produced) + Mappings |
| Off-chain SDK | JavaScript/Rust (aleo-sdk) |
| Consensus | Proof-of-stake (AleoBFT) |

---

## Mina: o1js + Recursive SNARKs

Mina's headline feature is its constant-size blockchain: ~22KB regardless of history. This is achieved through recursive zk-SNARKs that fold the entire chain history into a single proof. Smart contracts (called zkApps) participate in this recursive proving.

### o1js: JavaScript-First ZK

o1js is Mina's TypeScript library for writing zkApps. Unlike the other platforms, there's no separate language — you write TypeScript classes decorated with Mina-specific primitives.

```typescript
import { Field, SmartContract, State, state, method, UInt64 } from 'o1js';

class Counter extends SmartContract {
    @state(Field) count = State<Field>();

    @method async increment() {
        const currentCount = this.count.getAndRequireEquals();
        const newCount = currentCount.add(1);
        this.count.set(newCount);
    }

    @method async assertAbove(threshold: Field) {
        const currentCount = this.count.getAndRequireEquals();
        currentCount.assertGreaterThan(threshold);
    }
}
```

The `@method` decorator marks functions that compile to ZK circuits. `Field` is the base type (a 255-bit finite field element). Type safety is enforced — you can't accidentally mix circuit and off-circuit code.

### Recursive SNARKs and the Constant Chain

Mina's key innovation is **proof recursion**: a zkApp proof can include proofs of other zkApp proofs. This means you can build systems where one contract verifies the output of another without re-executing the computation on-chain. The chain state is itself a proof.

This architecture has trade-offs:
- **Pro**: Any application on Mina benefits from the constant-size blockchain
- **Con**: Proof generation is computationally expensive (minutes for complex circuits)
- **Con**: o1js's circuit model has footguns (easy to accidentally write non-circuit code that silently doesn't get proven)

### Privacy Model

Mina's zkApps don't have native asset privacy. On-chain state is public (similar to Ethereum). Privacy comes from keeping computation off-chain: you prove "I know a secret that satisfies these constraints" without revealing the secret, but the on-chain state itself is visible.

For true transaction privacy, you'd layer something like Tornado Cash's approach on top. Mina doesn't have built-in shielding at the application layer.

| Dimension | Mina |
|-----------|------|
| Language | o1js (TypeScript) |
| Privacy model | Off-chain computation, on-chain state public |
| ZK scheme | Kimchi (Mina's variant of PLONK), recursive |
| State model | On-chain fields (public), off-chain computation |
| Off-chain SDK | TypeScript (o1js) |
| Consensus | Ouroboros Samasika (PoS) |

---

## Zcash: Sapling and the UTXO Foundation

Zcash is the original privacy blockchain, and its Sapling protocol directly influenced Midnight's shielded UTXO model. Unlike the others, Zcash doesn't have general smart contracts — it's a payment network with privacy as the core feature.

### Sapling: Shielded Transactions

Zcash's Sapling protocol (and its successor Orchard/Unified Addresses) provides shielded payments using zk-SNARKs (Groth16 for Sapling, PLONK for Orchard). The note commitment tree and nullifier mechanism are the same ideas Midnight adapted for its shielded ledger.

```
// Zcash transaction (conceptual, not actual code)
sapling_spend:
  value commitment: <commits to amount>
  nullifier: <proves note is unspent>
  spend_proof: <zk-SNARK: I know the spending key>
  
sapling_output:
  value commitment: <commits to output amount>
  note_commitment: <for recipient's commitment tree>
  encrypted_note: <only recipient can decrypt>
```

Developers building on Zcash use the **Zcash SDK** (available in Swift, Kotlin, Rust) primarily for wallet applications. There's no programmable contract layer.

### Developer Perspective

Zcash is the least programmable of the five. You can build wallets and payment applications, but you can't write arbitrary logic. The privacy primitives are excellent — the cryptographic engineering behind Sapling is industry-leading — but they're not exposed as a general-purpose computing platform.

**Zcashd** (deprecated) and **Zebra** (the new full node, Rust) are the reference implementations. If you're building ZK applications, Zcash's cryptographic research (PLONK, Halo2) has influenced all the other platforms but the application-layer ecosystem is thin.

| Dimension | Zcash |
|-----------|-------|
| Language | None (SDKs only) |
| Privacy model | Shielded UTXO (Sapling/Orchard) |
| ZK scheme | Groth16 (Sapling), PLONK/Halo2 (Orchard) |
| State model | Transparent + Shielded UTXO |
| Off-chain SDK | Swift, Kotlin, Rust (wallet-focused) |
| Consensus | Proof-of-work (moving to PoS) |

---

## Side-by-Side Comparison

| | Midnight | Aztec | Aleo | Mina | Zcash |
|--|---------|-------|------|------|-------|
| **Smart contracts** | Yes (Compact) | Yes (Noir) | Yes (Leo) | Yes (o1js) | No |
| **Language style** | TypeScript-like DSL | Rust-like | Rust-like | TypeScript | N/A |
| **Private state** | Shielded UTXO + Ledger | Notes (UTXO) | Records (UTXO) | Off-chain only | Shielded UTXO |
| **Public state** | Ledger | Storage | Mappings | On-chain fields | Transparent UTXO |
| **Native tokens** | NIGHT | ETH (L2) | ALEO | MINA | ZEC |
| **ZK scheme** | zk-SNARKs | Honk/PLONK (recursive) | Groth16 | Kimchi/PLONK (recursive) | Groth16 / Halo2 |
| **Off-chain runtime** | TypeScript | TypeScript | JavaScript/Rust | TypeScript | Platform SDKs |
| **Layer** | L1 (Cardano partner) | L2 (Ethereum) | L1 | L1 | L1 |
| **Toolchain maturity** | Early (active dev) | Moderate | Moderate | Mature | Mature |

---

## Which Should You Choose?

**Choose Midnight if:**
- You need both public and private state in the same contract
- Your team is TypeScript-heavy
- You want first-class selective disclosure (proving properties without revealing data)
- You're building on or near the Cardano ecosystem

**Choose Aztec if:**
- You're building on Ethereum and want L2 privacy
- Your team knows Rust and Solidity patterns
- You need proven EVM compatibility
- You want a mature sandbox dev environment today

**Choose Aleo if:**
- Privacy-by-default is a requirement (records are private by default)
- Your use case maps naturally to a pure UTXO model
- You're building financial primitives where the record model fits
- You prefer a standalone L1 with a clean slate

**Choose Mina if:**
- Off-chain computation with on-chain verification is your pattern
- You want constant-chain-size guarantees for your users
- Your team prefers TypeScript over new languages
- Recursive proof composition is core to your architecture

**Choose Zcash if:**
- You're building a payment wallet, not general smart contracts
- Maximum payment privacy is the primary requirement
- You want to leverage battle-tested cryptographic primitives (Sapling/Orchard)

---

## The Shared Foundations

Despite their differences, these platforms converge on several ideas:

1. **The UTXO/note/record pattern** for private state — all draw from Zcash's Sapling. Private state can't be "read and modified" atomically from the chain's perspective; it's consumed and re-created.

2. **Commitment + nullifier scheme** — to spend a private note, you prove you know the secret without revealing it, and publish a nullifier so it can't be double-spent.

3. **Client-side proof generation** — in all five systems, the actual ZK proof is computed by the user's device or wallet. The chain verifies proofs but doesn't generate them.

4. **Dual state tension** — every platform grapples with the public/private boundary. The biggest design differences are in how explicitly that boundary is encoded in the language.

The practical implication: if you understand one of these systems well, you'll recognize the architectural patterns in the others. The cryptography is largely shared; the differences are in language design, developer experience, and where on the public/private spectrum each platform sits by default.

---

## Conclusion

Midnight's dual-ledger model with Compact is the most explicit about the public/private boundary — you declare in the type system what is public and what is private, and the compiler enforces it. Aztec takes a similar approach but in a UTXO-heavy Ethereum L2 context. Aleo's record model makes privacy the default. Mina's recursive SNARKs solve a different problem (chain size) and bolt privacy on at the application layer. Zcash remains the most proven but least programmable.

For developers building privacy-sensitive applications today, Aztec has the most mature toolchain. For those willing to bet on emerging platforms, Midnight's explicit dual-ledger design and TypeScript integration make it the most ergonomic for applications that genuinely need both public and private state.
