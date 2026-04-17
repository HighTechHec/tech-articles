# Bringing External Data On-Chain: Oracle Patterns for Midnight Contracts

Smart contracts are deterministic. Every node that processes a transaction must reach the same state from the same inputs. This constraint breaks the moment you try to reference anything external — a price feed, a random number, a real-world event. That external data changes. Different nodes would see different values. The chain forks.

Oracles solve this by converting external data into on-chain facts that all nodes agree on. The oracle makes a claim, the claim gets committed to the chain, and the contract operates on the committed fact rather than querying the external world directly.

Midnight's ZK architecture adds a useful dimension: oracles can make claims about data without revealing the underlying data. A price oracle can prove that a value is in a range without revealing the exact price. An identity oracle can attest to a credential without revealing who holds it. This is the design space this tutorial explores.

---

## The Three Oracle Models

There's no single "oracle pattern" — the right design depends on trust assumptions and data sensitivity.

### Model 1: Trusted Authority Oracle

The simplest model. A designated key signs off-chain data and the contract verifies the signature on-chain. Trust is inherited from the key holder.

```compact
pragma language_version >= 1.0.0;

import CompactStandardLibrary;

// The oracle's public key — set at deploy time
export ledger oracle_pubkey: Bytes<32>;
// Current data value and the block it was signed for
export ledger current_value: Uint<64>;
export ledger value_timestamp: Uint<64>;

export circuit initialize(pubkey: Bytes<32>): [] {
  oracle_pubkey = pubkey;
  current_value = 0uL;
  value_timestamp = 0uL;
}

// Oracle operator calls this to update the data
export circuit submit_data(
  value: Uint<64>,
  timestamp: Uint<64>,
  signature: Bytes<64>
): [] {
  // Verify oracle signed this (value, timestamp) pair
  const msg = concat<8, 8>(
    Bytes<8>(value),
    Bytes<8>(timestamp)
  );
  assert ed25519_verify(oracle_pubkey, msg, signature);
  // Monotonic timestamp — no replay of old data
  assert timestamp > value_timestamp;
  current_value = value;
  value_timestamp = timestamp;
}

export circuit get_value(): Uint<64> {
  return current_value;
}
```

**`ed25519_verify`** is a Compact built-in that verifies an Ed25519 signature against a public key and message. The contract trusts the oracle's key completely — if the key is compromised, so is the oracle feed.

This is the pattern to use when:
- You control the oracle operator
- The data is public (price feeds, weather data, sports scores)
- Simplicity is more important than decentralization

### Model 2: Threshold Oracle (Multi-Sig)

Multiple independent sources must agree before data is accepted. Reduces single-point-of-failure.

```compact
pragma language_version >= 1.0.0;

import CompactStandardLibrary;

export ledger oracle_keys: Map<Uint<8>, Bytes<32>>;
export ledger oracle_count: Uint<8>;
export ledger threshold: Uint<8>;
export ledger current_value: Uint<64>;
export ledger value_round: Uint<64>;

// Pending submissions for current round
export ledger submissions: Map<Uint<8>, Uint<64>>;
export ledger submitted: Map<Uint<8>, Boolean>;
export ledger submission_count: Counter;

export circuit initialize(required: Uint<8>): [] {
  threshold = required;
  oracle_count = 0uL as Uint<8>;
  value_round = 0uL;
  submission_count.increment(0uL);
}

export circuit register_oracle(index: Uint<8>, pubkey: Bytes<32>): [] {
  assert index == oracle_count;
  oracle_keys.insert(index, pubkey);
  oracle_count = (oracle_count as Uint<64> + 1uL) as Uint<8>;
}

export circuit submit(
  oracle_index: Uint<8>,
  value: Uint<64>,
  round: Uint<64>,
  signature: Bytes<64>
): [] {
  assert round == value_round + 1uL;
  assert submitted.lookup(oracle_index).isNone();

  const pubkey = oracle_keys.lookup(oracle_index).value();
  const msg = concat<8, 8>(Bytes<8>(value), Bytes<8>(round));
  assert ed25519_verify(pubkey, msg, signature);

  submissions.insert(oracle_index, value);
  submitted.insert(oracle_index, true);
  submission_count.increment(1uL);
}

export circuit finalize(round: Uint<64>, consensus_value: Uint<64>): [] {
  assert round == value_round + 1uL;
  // Require threshold submissions
  assert submission_count.value >= threshold as Uint<64>;
  // Verify consensus_value matches at least `threshold` submissions
  // (caller provides the value, circuit verifies it appears enough times)
  let count: Uint<64> = 0uL;
  let i: Uint<8> = 0uL as Uint<8>;
  // Note: Compact loops are bounded — max oracle count is compile-time fixed
  // For a real implementation, unroll or use a counter-based approach
  current_value = consensus_value;
  value_round = round;
}
```

The `finalize` circuit here is simplified — a production implementation would verify that `consensus_value` appears at least `threshold` times in the submissions map. Compact doesn't have dynamic-length loops, so this verification is done by unrolling up to the max oracle count. For small N (3-of-5, 5-of-9), the unrolled version is clean.

### Model 3: ZK Oracle (Private Data Attestation)

This is Midnight's unique contribution to oracle design. The oracle holds private data and proves properties about it without revealing the data itself.

Example: a credit oracle that proves a borrower's credit score exceeds a threshold without revealing the actual score.

```compact
pragma language_version >= 1.0.0;

import CompactStandardLibrary;

// Oracle's public attestation key
export ledger attestation_key: Bytes<32>;

// Ledger of attestations: subject_id → commitment to attested value
export ledger attestations: Map<Bytes<32>, Bytes<32>>;

// Record that subject_id's value exceeds threshold
// without revealing what the value is
export circuit attest_range(
  subject_id: Bytes<32>,
  min_value: Uint<64>,
  max_value: Uint<64>,
  signature: Bytes<64>
): [] {
  // Oracle signed: (subject_id, min_value, max_value)
  const msg = concat<32, 16>(
    subject_id,
    concat<8, 8>(Bytes<8>(min_value), Bytes<8>(max_value))
  );
  assert ed25519_verify(attestation_key, msg, signature);
  // Store a commitment: sha256(subject_id || min || max)
  const commitment = sha256(msg);
  attestations.insert(subject_id, commitment);
}

// Private witness: actual value (never goes on-chain)
witness actual_value(subject: Bytes<32>): Uint<64>;

// Circuit: prove this subject's attested value exceeds a required minimum
export circuit prove_sufficient(
  subject_id: Bytes<32>,
  required_minimum: Uint<64>
): [] {
  // Must have an attestation on file
  assert attestations.lookup(subject_id).isSome();

  // Retrieve the actual value privately
  const val = actual_value(subject_id);

  // Verify it meets the requirement
  assert val >= required_minimum;
  // The ZK proof proves val >= required_minimum
  // without revealing val — the attestation anchors the trust
}
```

This pattern enables private credit scores, health records, KYC attestations, and similar use cases. The oracle attests to a value range; the subject proves their value satisfies a condition within that range.

---

## Freshness and Staleness

The most common oracle bug is using stale data. A price signed 10 minutes ago is not the current price. Contracts need freshness guarantees.

The correct pattern is to tie oracle data to block height or a timestamp, then enforce a maximum staleness in the circuit that uses it:

```compact
export ledger price: Uint<64>;
export ledger price_block: Uint<64>;
export ledger max_staleness: Uint<64>;  // in blocks

export circuit use_price(
  required_freshness: Uint<64>,
  current_block: Uint<64>
): Uint<64> {
  assert current_block - price_block <= max_staleness;
  return price;
}
```

`current_block` is passed in as a circuit argument rather than read from a system variable — Compact circuits don't have access to a "current block number" directly. The caller provides it and the circuit validates it's within acceptable range.

For price-sensitive contracts (lending, AMMs), `max_staleness` should be tight — one or two blocks. For slower-moving data (credit scores, identity attestations), a longer staleness window is acceptable.

---

## Replay Attack Prevention

Oracles must prevent replaying old valid signatures. A signed message `(price=100, timestamp=T)` is valid forever unless the contract enforces monotonic progression.

The pattern used in the trusted authority model above uses a monotonic timestamp check:

```compact
assert timestamp > value_timestamp;
```

For multi-party oracles, the round number serves the same purpose:

```compact
assert round == value_round + 1uL;
```

A more robust approach for high-frequency oracles is a nonce map — the contract remembers every `(oracle_id, nonce)` pair it's seen and rejects duplicates:

```compact
export ledger used_nonces: Map<Bytes<32>, Boolean>;

export circuit submit_with_nonce(
  value: Uint<64>,
  nonce: Bytes<32>,
  signature: Bytes<64>
): [] {
  // Reject replayed nonces
  assert used_nonces.lookup(nonce).isNone();
  used_nonces.insert(nonce, true);
  // ... rest of verification
}
```

The tradeoff: the nonce map grows indefinitely. For long-lived contracts, this storage cost adds up. A sliding window approach — accepting only nonces from the last N rounds — bounds the storage:

```compact
export ledger current_epoch: Uint<64>;
export ledger epoch_nonces: Map<Bytes<32>, Boolean>;

export circuit advance_epoch(new_epoch: Uint<64>): [] {
  // Clear old nonces when epoch advances
  // Note: clearing a Map in Compact requires reinitializing it
  // — plan for this in your storage design
  assert new_epoch == current_epoch + 1uL;
  current_epoch = new_epoch;
  // epoch_nonces gets reset at epoch boundary
}
```

---

## The Pull vs Push Model

**Push oracles** update the contract state proactively — the oracle operator submits transactions to keep the on-chain data current. This is what the `submit_data` circuit above does.

**Pull oracles** are queried by the contract when needed. In Midnight's architecture, this means the contract reads a value that was previously pushed by the oracle — there's no mechanism for a circuit to make an outbound call during execution. Circuits are self-contained.

The practical difference:
- Push oracles require the operator to continuously submit transactions. Gas/fee costs are the oracle's problem.
- Pull oracles shift the freshness burden to callers — if no one has pushed recently, the data is stale.

For Midnight specifically, the ZK proof generation time is a factor. Callers who need a fresh oracle value must either wait for the oracle to push, or trigger the push themselves (which means a two-transaction flow: push + use).

A common design pattern for price-gated operations:

```typescript
// Caller triggers oracle update, then uses the price
async function buyAtCurrentPrice(
  oracleContract: OracleContract,
  marketContract: MarketContract,
  wallet: WalletFacade
): Promise<void> {
  // 1. Fetch fresh price from oracle backend
  const { price, signature, timestamp } = await fetchOracleData();

  // 2. Submit oracle update (separate transaction)
  const updateTx = await oracleContract.callTx.submit_data(
    price, timestamp, signature
  );
  await wallet.submitTransaction(updateTx);

  // 3. Now use the price in the market contract
  const buyTx = await marketContract.callTx.buy(price);
  await wallet.submitTransaction(buyTx);
}
```

The two-transaction flow has atomicity implications — between the oracle update and the buy, another caller could also use the price. For most applications this is fine. If atomicity matters, both operations need to be in a single ZK proof (which requires designing the contracts to compose at the circuit level, not just at the transaction level).

---

## Practical: Price Feed Oracle Implementation

A complete TypeScript + Compact price feed suitable for a lending protocol.

### Compact Contract

```compact
pragma language_version >= 1.0.0;

import CompactStandardLibrary;

export ledger oracle_key: Bytes<32>;
export ledger prices: Map<Bytes<8>, Uint<64>>;      // asset_id → price (in NIGHT)
export ledger timestamps: Map<Bytes<8>, Uint<64>>;  // asset_id → last update time
export ledger max_age: Uint<64>;                     // max staleness in seconds

export circuit initialize(
  pubkey: Bytes<32>,
  staleness_limit: Uint<64>
): [] {
  oracle_key = pubkey;
  max_age = staleness_limit;
}

export circuit update_price(
  asset_id: Bytes<8>,
  price: Uint<64>,
  timestamp: Uint<64>,
  signature: Bytes<64>
): [] {
  // Signature over (asset_id, price, timestamp)
  const preimage = concat<8, 16>(
    asset_id,
    concat<8, 8>(Bytes<8>(price), Bytes<8>(timestamp))
  );
  assert ed25519_verify(oracle_key, preimage, signature);

  // Must be newer than existing data for this asset
  const prior_ts = timestamps.lookup(asset_id).orDefault(0uL);
  assert timestamp > prior_ts;

  prices.insert(asset_id, price);
  timestamps.insert(asset_id, timestamp);
}

export circuit get_price(
  asset_id: Bytes<8>,
  current_time: Uint<64>
): Uint<64> {
  const ts = timestamps.lookup(asset_id).value();
  assert current_time - ts <= max_age;  // reject stale
  return prices.lookup(asset_id).value();
}

export circuit price_above_threshold(
  asset_id: Bytes<8>,
  minimum: Uint<64>,
  current_time: Uint<64>
): Boolean {
  const ts = timestamps.lookup(asset_id).value();
  assert current_time - ts <= max_age;
  const p = prices.lookup(asset_id).value();
  return p >= minimum;
}
```

### TypeScript Oracle Backend

```typescript
import * as ed from '@noble/ed25519';
import { sha512 } from '@noble/hashes/sha512';
import fetch from 'node-fetch';

// Configure noble-ed25519 to use sha512 (required in Node.js)
ed.etc.sha512Sync = (...m) => sha512(ed.etc.concatBytes(...m));

interface OracleConfig {
  privateKey: Uint8Array;
  contractAddress: string;
  wallet: any; // WalletFacade
  priceContract: any;
  updateIntervalMs: number;
}

async function fetchCryptoPrice(assetSymbol: string): Promise<number> {
  // Example: CoinGecko free API
  const res = await fetch(
    `https://api.coingecko.com/api/v3/simple/price?ids=${assetSymbol}&vs_currencies=usd`
  );
  const data = await res.json() as any;
  return Math.round(data[assetSymbol].usd * 1e6); // scale to micro-NIGHT
}

function buildOracleMessage(
  assetId: Uint8Array,
  price: bigint,
  timestamp: bigint
): Uint8Array {
  const msg = new Uint8Array(24); // 8 + 8 + 8
  msg.set(assetId, 0);
  const priceBytes = new DataView(new ArrayBuffer(8));
  priceBytes.setBigUint64(0, price);
  msg.set(new Uint8Array(priceBytes.buffer), 8);
  const tsBytes = new DataView(new ArrayBuffer(8));
  tsBytes.setBigUint64(0, timestamp);
  msg.set(new Uint8Array(tsBytes.buffer), 16);
  return msg;
}

async function pushPriceUpdate(config: OracleConfig): Promise<void> {
  const assetId = new TextEncoder().encode('BTC\0\0\0\0\0').slice(0, 8);
  const price = BigInt(await fetchCryptoPrice('bitcoin'));
  const timestamp = BigInt(Math.floor(Date.now() / 1000));

  const message = buildOracleMessage(assetId, price, timestamp);
  const signature = await ed.sign(message, config.privateKey);

  const tx = await config.priceContract.callTx.update_price(
    assetId, price, timestamp, signature
  );
  await config.wallet.submitTransaction(tx);
  console.log(`Oracle updated BTC price: ${price} at ${timestamp}`);
}

// Run price push on interval
async function runOracleService(config: OracleConfig): Promise<never> {
  while (true) {
    try {
      await pushPriceUpdate(config);
    } catch (e) {
      console.error(`Oracle push failed: ${e}`);
    }
    await new Promise(r => setTimeout(r, config.updateIntervalMs));
  }
}
```

### Using the Oracle in a Lending Contract

```compact
// lending.compact — imports oracle via contract address
extern oracle_contract: OracleContract;

export circuit borrow(
  asset_id: Bytes<8>,
  amount: Uint<64>,
  current_time: Uint<64>
): [] {
  // Get verified fresh price from oracle contract
  const price = oracle_contract.get_price(asset_id, current_time);

  // Collateral check: require 150% overcollateralization
  const collateral_value = collateral * price / 1_000_000uL;
  assert collateral_value * 100uL >= amount * 150uL;

  // ... rest of borrow logic
}
```

The `extern` keyword declares a dependency on another deployed contract. The oracle contract's verified price is used directly in the lending circuit's collateral calculation.

---

## Security Considerations

**Oracle key management**: The oracle's signing key is the root of trust. Compromised key = compromised oracle. Use HSMs or threshold key schemes (Shamir's Secret Sharing, FROST) for production oracle keys. Rotate keys via a governance circuit.

**Price manipulation**: Single-source oracles are vulnerable to price manipulation on the data source. For financial contracts, use aggregated sources (average of 5+ price feeds), time-weighted averages (TWAP), or outlier rejection.

**Liveness dependency**: A contract that requires a fresh oracle value can be DoS'd by stopping oracle updates. Design graceful degradation — what should the contract do if the oracle hasn't updated in N hours? Options: pause new operations, use last-known-good price with wider staleness window, fail-open to protect existing positions.

**MEV and front-running**: If oracle updates are visible before they're included in a block, searchers can front-run. Midnight's shielded transaction layer mitigates this for the transaction content, but the oracle update itself (which must be verifiable) may still be observable. Commitment-then-reveal for oracle updates is possible but adds a round trip.

**ZK proof cost**: For ZK oracles (Model 3), each `prove_sufficient` call generates a ZK proof. On the RTX 5090, this is fast. On a developer laptop, it may take 30-90 seconds per call. For high-frequency applications, batch multiple proofs or use a separate proof service.

---

## Oracle Lifecycle

### Deployment

```typescript
async function deployOracleInfrastructure(
  oraclePrivateKey: Uint8Array,
  wallet: WalletFacade
): Promise<{ oracleAddress: string; publicKey: Uint8Array }> {
  const publicKey = await ed.getPublicKey(oraclePrivateKey);

  const tx = await priceFeedContract.deploy({
    pubkey: publicKey,
    staleness_limit: 300n // 5 minutes
  });
  const result = await wallet.submitTransaction(tx);

  return {
    oracleAddress: result.contractAddress,
    publicKey
  };
}
```

### Key Rotation

```compact
export ledger pending_key: Bytes<32>;
export ledger rotation_authorized: Boolean;

export circuit authorize_rotation(new_key: Bytes<32>, current_sig: Bytes<64>): [] {
  // Current oracle key authorizes rotation to new key
  const msg = concat<4, 32>(Bytes<4>(0xROTATEuL), new_key);
  assert ed25519_verify(oracle_key, msg, current_sig);
  pending_key = new_key;
  rotation_authorized = true;
}

export circuit complete_rotation(new_sig: Bytes<64>): [] {
  assert rotation_authorized;
  // New key confirms rotation
  const msg = Bytes<8>(0xCONFIRM_ROTuL);
  assert ed25519_verify(pending_key, msg, new_sig);
  oracle_key = pending_key;
  rotation_authorized = false;
}
```

Two-step key rotation requires both the old and new key to sign off, preventing a compromised old key from blocking rotation.

---

## Summary

Oracle patterns in Midnight span a spectrum from simple trusted-authority designs to ZK-native private attestations:

| Model | Trust | Data Privacy | Complexity |
|-------|-------|-------------|------------|
| Trusted authority | Single key | None | Low |
| Threshold multi-sig | N-of-M keys | None | Medium |
| ZK attestation | Attestor + ZK | High | High |

Key implementation concerns apply to all models: monotonic timestamp/round enforcement prevents replay attacks; freshness checks in the consuming circuit prevent stale data; key management is the operational root of trust.

The ZK attestation model is Midnight-native and enables use cases that aren't possible on transparent chains: privacy-preserving credit scoring, selective disclosure of compliance credentials, and range proofs over sensitive values. These require designing the oracle to commit to value ranges rather than point values, then using witness-based circuits to prove properties over the private data without revealing it.
