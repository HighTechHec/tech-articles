# Commit-Reveal Voting on Midnight: Private Ballots with Verifiable Results

Voting systems on public blockchains face a fundamental tension: transparency and auditability require that votes be visible on-chain, but visible votes enable vote-buying, coercion, and strategic last-minute bandwagon voting. These aren't theoretical concerns — they're documented failure modes in DAO governance systems.

Midnight's ZK proof infrastructure enables a different design: commit-reveal voting. Voters commit to a choice in a provably concealed form during the voting period. After polls close, they reveal their vote in a way that's verifiably consistent with their original commitment. The result is a voting system where ballots are private during the vote, verifiable after, and tamper-proof throughout.

This tutorial builds a complete commit-reveal voting contract in Compact, walks through the cryptographic design, and covers the TypeScript integration layer.

---

## The Commit-Reveal Pattern

Commit-reveal works in two phases:

**Commit phase**: The voter submits a commitment — a hash of their vote combined with a secret nonce. The on-chain state records this hash but reveals nothing about the actual vote. Even if someone watches every transaction, they see only `sha256(vote || nonce)`.

**Reveal phase**: After voting closes, the voter submits their original vote and nonce. The contract verifies that `sha256(vote || nonce)` matches their earlier commitment. If it does, the vote is counted. If the reveal doesn't match, it's rejected.

The nonce is critical: without it, an attacker could brute-force `sha256(yes)` vs `sha256(no)` since there are only two choices. A 32-byte random nonce makes preimage attacks infeasible.

The ZK component is what Midnight adds: rather than revealing the nonce and vote in plaintext on-chain, the reveal is done inside a circuit. The voter proves they know a `(vote, nonce)` pair that hashes to their commitment, without exposing either value to the chain. The chain learns only that the proof is valid and which bucket to increment.

---

## Contract Design

The contract tracks:

- A mapping of voter keys to their commitments (commit phase)
- A deadline separating commit and reveal phases
- Vote tallies for each option (reveal phase)
- A registry of which voters have already revealed (prevent double-reveal)

```compact
pragma language_version >= 1.0.0;

import CompactStandardLibrary;

// Ledger state
export ledger commitments: Map<Bytes<32>, Bytes<32>>;
export ledger vote_counts: Map<Uint<8>, Uint<64>>;
export ledger revealed: Map<Bytes<32>, Boolean>;
export ledger commit_deadline: Uint<64>;
export ledger reveal_deadline: Uint<64>;
export ledger num_options: Uint<8>;
export ledger total_reveals: Counter;

// Initialize the vote
export circuit initialize(
  commit_end: Uint<64>,
  reveal_end: Uint<64>,
  options: Uint<8>
): [] {
  assert options >= 2uL as Uint<8>;
  assert options <= 16uL as Uint<8>;
  assert commit_end < reveal_end;
  commit_deadline = commit_end;
  reveal_deadline = reveal_end;
  num_options = options;
  total_reveals.increment(0uL);
}

// Phase 1: Submit commitment
export circuit commit(
  voter_key: Bytes<32>,
  commitment: Bytes<32>,
  current_time: Uint<64>
): [] {
  // Must be in commit phase
  assert current_time <= commit_deadline;
  // Each voter commits exactly once
  assert commitments.lookup(voter_key).isNone();
  commitments.insert(voter_key, commitment);
}

// Phase 2: Reveal vote (ZK proof verifies vote+nonce → commitment)
witness vote_value(voter: Bytes<32>): Uint<8>;
witness vote_nonce(voter: Bytes<32>): Bytes<32>;

export circuit reveal(
  voter_key: Bytes<32>,
  current_time: Uint<64>
): [] {
  // Must be in reveal phase
  assert current_time > commit_deadline;
  assert current_time <= reveal_deadline;

  // Voter must have committed
  const prior_commitment = commitments.lookup(voter_key).value();

  // Voter must not have revealed yet
  assert revealed.lookup(voter_key).isNone();

  // Pull private values from witness (stays off-chain)
  const vote = vote_value(voter_key);
  const nonce = vote_nonce(voter_key);

  // Verify the commitment: hash(vote_byte || nonce) == prior_commitment
  // Pack vote as first byte of a 33-byte input
  const vote_bytes = pad<33>(Bytes<1>(vote));
  const preimage = concat<1, 32>(Bytes<1>(vote), nonce);
  const computed_hash = sha256(preimage);
  assert computed_hash == prior_commitment;

  // Vote must be in valid range
  assert vote < num_options;

  // Count the vote
  const current_count = vote_counts.lookup(vote).orDefault(0uL);
  vote_counts.insert(vote, current_count + 1uL);

  // Mark as revealed
  revealed.insert(voter_key, true);
  total_reveals.increment(1uL);
}

// Read results (usable after reveal phase ends)
export circuit get_count(option: Uint<8>): Uint<64> {
  return vote_counts.lookup(option).orDefault(0uL);
}

export circuit is_commit_phase(current_time: Uint<64>): Boolean {
  return current_time <= commit_deadline;
}

export circuit has_committed(voter_key: Bytes<32>): Boolean {
  return commitments.lookup(voter_key).isSome();
}

export circuit has_revealed(voter_key: Bytes<32>): Boolean {
  return revealed.lookup(voter_key).isSome();
}
```

A few design decisions worth noting:

**`commitments.lookup(voter_key).value()`** — This will panic if the key doesn't exist. The `assert` on the previous line guarantees it's present, so this is safe. Using `.value()` after a `.isSome()` check (or here, after asserting the commit exists) is the correct Compact pattern.

**`revealed.lookup(voter_key).isNone()`** rather than checking `!revealed.lookup(voter_key).orDefault(false)` — The None/Some distinction is important. A voter who was never registered returns `None`; a voter who has revealed returns `Some(true)`. Checking `.isNone()` is more precise than treating absence and `false` identically.

**`vote_counts.lookup(option).orDefault(0uL)`** — Options with zero votes don't appear in the map. `orDefault` handles this cleanly without requiring pre-initialization.

---

## The Commitment Hash Construction

The `reveal` circuit constructs the commitment hash from the witness values. This is the cryptographic core: if the hash matches, the voter knew their vote and nonce at commit time.

```compact
const preimage = concat<1, 32>(Bytes<1>(vote), nonce);
const computed_hash = sha256(preimage);
assert computed_hash == prior_commitment;
```

`Bytes<1>(vote)` converts the `Uint<8>` vote choice to its byte representation. `concat<1, 32>` concatenates the 1-byte vote with the 32-byte nonce into a 33-byte preimage. `sha256` produces the 32-byte commitment hash.

The TypeScript side must produce the same hash format when computing commitments:

```typescript
import { sha256 } from '@midnight-ntwrk/compact-runtime';

function computeCommitment(vote: number, nonce: Uint8Array): Uint8Array {
  const preimage = new Uint8Array(33);
  preimage[0] = vote;
  preimage.set(nonce, 1);
  return sha256(preimage);
}
```

This exact byte layout must match what the circuit expects. If the TypeScript uses a different encoding (e.g., little-endian vote, or a different concatenation order), the hashes won't match and reveals will fail.

---

## The Witness Functions

`vote_value` and `vote_nonce` are declared as witnesses:

```compact
witness vote_value(voter: Bytes<32>): Uint<8>;
witness vote_nonce(voter: Bytes<32>): Bytes<32>;
```

Witnesses are functions that return private values. From the circuit's perspective, they're just function calls. The values they return are never written to the blockchain — they live only in the ZK proof's private inputs. The circuit uses them in computations, and the resulting proof attests to the computations without revealing the inputs.

The TypeScript integration provides these witness values at proof generation time:

```typescript
import { WitnessContext } from '@midnight-ntwrk/compact-runtime';

interface VoteWitnesses {
  vote_value: (voter: Uint8Array) => number;
  vote_nonce: (voter: Uint8Array) => Uint8Array;
}

function createRevealWitnesses(
  vote: number,
  nonce: Uint8Array
): VoteWitnesses {
  return {
    vote_value: (_voter) => vote,
    vote_nonce: (_voter) => nonce,
  };
}
```

The voter passes these witnesses when calling `reveal`. The private server generates the ZK proof using them.

---

## TypeScript Integration

### Setting Up

```typescript
import { WalletFacade } from '@midnight-ntwrk/wallet-sdk-facade';
import { MidnightProvider } from '@midnight-ntwrk/midnight-js-providers';
import { firstValueFrom, filter, timeout, throttleTime } from 'rxjs';
import { createHash } from 'crypto';

const CONTRACT_ABI = require('./build/voting.abi.json');

interface VotingContract {
  callTx: {
    initialize: (commitEnd: bigint, revealEnd: bigint, options: number) => Promise<any>;
    commit: (voterKey: Uint8Array, commitment: Uint8Array, now: bigint) => Promise<any>;
    reveal: (voterKey: Uint8Array, now: bigint) => Promise<any>;
  };
  query: {
    get_count: (option: number) => Promise<bigint>;
    has_committed: (voterKey: Uint8Array) => Promise<boolean>;
    has_revealed: (voterKey: Uint8Array) => Promise<boolean>;
  };
}
```

### Generating a Commitment Off-Chain

```typescript
function generateVoteCommitment(vote: number, options: number): {
  commitment: Uint8Array;
  nonce: Uint8Array;
} {
  if (vote < 0 || vote >= options) {
    throw new Error(`Invalid vote ${vote} — must be 0..${options - 1}`);
  }
  // 32 bytes of cryptographic randomness
  const nonce = crypto.getRandomValues(new Uint8Array(32));
  const preimage = new Uint8Array(33);
  preimage[0] = vote;
  preimage.set(nonce, 1);

  // Node.js SHA-256
  const hash = createHash('sha256');
  hash.update(preimage);
  const commitment = new Uint8Array(hash.digest());

  return { commitment, nonce };
}
```

**Store the nonce securely.** If the voter loses their nonce, they cannot reveal their vote. In a production system, the nonce is saved encrypted in the voter's local storage or wallet. For testing, save it to a file.

### Committing

```typescript
async function submitCommit(
  contract: VotingContract,
  wallet: WalletFacade,
  voterKey: Uint8Array,
  vote: number,
  options: number
): Promise<{ nonce: Uint8Array }> {
  const now = BigInt(Math.floor(Date.now() / 1000));
  const { commitment, nonce } = generateVoteCommitment(vote, options);

  const tx = await contract.callTx.commit(voterKey, commitment, now);
  await wallet.submitTransaction(tx);

  console.log(`Committed vote ${vote} with nonce ${Buffer.from(nonce).toString('hex').slice(0, 16)}...`);
  return { nonce };
}
```

### Revealing

```typescript
async function submitReveal(
  contract: VotingContract,
  wallet: WalletFacade,
  voterKey: Uint8Array,
  vote: number,
  nonce: Uint8Array
): Promise<void> {
  const now = BigInt(Math.floor(Date.now() / 1000));

  // Attach witnesses — these stay private
  const witnesses = createRevealWitnesses(vote, nonce);
  const tx = await contract.callTx.reveal(voterKey, now, witnesses);
  await wallet.submitTransaction(tx);

  console.log(`Revealed vote for ${Buffer.from(voterKey).toString('hex').slice(0, 8)}...`);
}
```

The witnesses are passed as an additional argument to the circuit call. The exact API depends on the SDK version — in `midnight-js` 0.2.x, witnesses are injected via a `WitnessContext` that's attached to the contract instance rather than passed per-call. Check your SDK version's witness injection pattern.

### Reading Results After Reveal Phase

```typescript
async function getResults(
  contract: VotingContract,
  numOptions: number
): Promise<{ option: number; count: bigint }[]> {
  const results = [];
  for (let i = 0; i < numOptions; i++) {
    const count = await contract.query.get_count(i);
    results.push({ option: i, count });
  }
  return results.sort((a, b) => (b.count > a.count ? 1 : -1));
}
```

---

## Handling the Unrevealed Vote Problem

A classic commit-reveal problem: a voter commits but never reveals. Their vote is locked but uncounted. This is usually acceptable — it's their choice not to reveal — but some designs need to know the final participation rate.

Midnight's `total_reveals` counter tracks how many votes were actually counted. The total commits can be derived from `commitments.lookup()` calls, but that's expensive off-chain. A better design adds a commit counter:

```compact
export ledger total_commits: Counter;

// In commit circuit:
total_commits.increment(1uL);
```

Then participation rate is `total_reveals.value / total_commits.value`. If reveal participation is low, the result may not be legitimate even if the contract accepts it.

A quorum threshold can be enforced in a separate circuit:

```compact
export circuit is_quorum_met(threshold_pct: Uint<8>): Boolean {
  const commits = total_commits.value;
  if commits == 0uL {
    return false;
  }
  const reveals = total_reveals.value;
  // reveals * 100 >= commits * threshold_pct
  return reveals * 100uL >= commits * (threshold_pct as Uint<64>);
}
```

---

## Testing with CompactSimulator

The Midnight testkit includes `CompactSimulator` for unit-testing circuits without deploying to a node. For commit-reveal, the key test cases are:

1. Commit with valid parameters succeeds
2. Double-commit for same voter fails
3. Reveal before commit phase ends fails
4. Reveal with wrong nonce fails (hash mismatch)
5. Reveal with invalid vote option fails
6. Double-reveal fails
7. Reveal after reveal deadline fails

```typescript
import { CompactSimulator } from '@midnight-ntwrk/compact-runtime/testing';
import { describe, it, expect } from 'vitest';

describe('commit-reveal voting', () => {
  let sim: CompactSimulator;

  beforeEach(async () => {
    sim = await CompactSimulator.create('./build/voting.compact');
    const NOW = 1000n;
    const COMMIT_END = 2000n;
    const REVEAL_END = 3000n;
    await sim.callCircuit('initialize', [COMMIT_END, REVEAL_END, 2]);
  });

  it('rejects reveal with wrong nonce', async () => {
    const voter = new Uint8Array(32).fill(1);
    const { commitment } = generateVoteCommitment(0, 2);

    await sim.callCircuit('commit', [voter, commitment, 1500n]);

    // Try to reveal with wrong nonce
    const wrongNonce = new Uint8Array(32).fill(99);
    sim.setWitness('vote_value', () => 0);
    sim.setWitness('vote_nonce', () => wrongNonce);

    await expect(
      sim.callCircuit('reveal', [voter, 2500n])
    ).rejects.toThrow(); // hash mismatch asserts false
  });

  it('counts votes correctly after valid reveals', async () => {
    const voter1 = new Uint8Array(32).fill(1);
    const voter2 = new Uint8Array(32).fill(2);

    const { commitment: c1, nonce: n1 } = generateVoteCommitment(0, 2);
    const { commitment: c2, nonce: n2 } = generateVoteCommitment(1, 2);

    await sim.callCircuit('commit', [voter1, c1, 1500n]);
    await sim.callCircuit('commit', [voter2, c2, 1500n]);

    sim.setWitness('vote_value', (v) => v[0] === 1 ? 0 : 1);
    sim.setWitness('vote_nonce', (v) => v[0] === 1 ? n1 : n2);

    await sim.callCircuit('reveal', [voter1, 2500n]);
    await sim.callCircuit('reveal', [voter2, 2500n]);

    const count0 = await sim.callCircuit('get_count', [0]);
    const count1 = await sim.callCircuit('get_count', [1]);
    expect(count0).toBe(1n);
    expect(count1).toBe(1n);
  });
});
```

---

## Security Properties

**Vote privacy during commit phase**: Only the voter knows their vote. The commitment hash reveals nothing computationally, given a random nonce.

**Vote binding**: A voter cannot change their vote after committing. Any reveal that doesn't match the commitment is rejected.

**Verifiability**: The ZK proof guarantees the voter computed the hash correctly. The on-chain tally is a verifiable result of all valid reveals.

**Non-malleable commitments**: Because the nonce is 32 random bytes, two different voters committing the same vote produce different commitments (with overwhelming probability). An attacker cannot copy a commitment.

**No coercion resistance beyond commit phase**: This scheme doesn't provide receipt-freeness — a voter can still prove to a third party what they voted by revealing their `(vote, nonce)` pair. If coercion resistance is required, additional mechanisms (re-randomizable commitments or threshold-decrypted schemes) are needed. Commit-reveal is a significant improvement over plaintext voting, not a complete anti-coercion solution.

---

## Summary

Commit-reveal voting on Midnight separates vote privacy from verifiability. Voters commit to a hash during the voting period, then prove knowledge of their vote via ZK circuit during reveal. The result is a ballot system where votes are private while polls are open, verifiable after they close, and tamper-proof throughout.

Key implementation points:
- Hash construction `sha256(vote_byte || nonce_32bytes)` must match exactly between TypeScript and Compact
- Witnesses (`vote_value`, `vote_nonce`) are the private inputs to the reveal circuit — they never touch the chain
- Store nonces durably — a lost nonce means a lost vote
- Track both commits and reveals for quorum enforcement
- `CompactSimulator` is the right tool for testing circuit edge cases before deploying
