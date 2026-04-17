# Contract-State Accounting vs UTXO Tokens on Midnight: Two Models for On-Chain Value

When you need to track value in a Midnight smart contract, you have two architectural options: the **UTXO-layer** model using native shielded tokens, or **ledger-state accounting** using Counter and Map fields inside the contract. They're not interchangeable — each has a specific role, and choosing the wrong one either breaks your privacy model or breaks your contract entirely.

This tutorial explains both approaches in depth, shows working Compact code for each, and gives you a clear decision framework for when to reach for which.

---

## The Core Distinction

Midnight has two layers where value can live:

**The shielded UTXO layer** (Zswap) is Midnight's native token layer. It works like Zcash's Sapling: value exists as encrypted notes committed to a Merkle tree. Nobody knows who holds what, or how much. When a contract receives shielded tokens via `receiveShielded`, it takes custody of a note. When it sends them via `sendShielded`, it produces a new note for the recipient.

**The contract ledger** is the smart contract state — the fields you declare with `export ledger`. These are public: anyone can query them. Counter, Map, Boolean fields live here. When you track a balance internally using a Map, that balance is visible on-chain.

The critical constraint: **shielded UTXO operations and contract-initiated token moves don't work the same way**. If a user wants to send shielded tokens *into* a contract, they use `receiveShielded`. If the contract wants to send tokens *out*, it uses `sendShielded`. Neither of these goes through the contract ledger.

This sounds simple but has non-obvious implications that will bite you if you mix the models.

---

## Model 1: UTXO-Layer Tokens

### When shielded tokens make sense

Use this model when:
- You're building a real token (fungible, transferable, with supply constraints)
- Privacy of holdings is a first-class requirement
- You need mint/burn operations
- You're building payment functionality (escrow, settlement, atomic swaps)

### The Compact primitives

`receiveShielded` accepts a shielded coin into the contract's custody. The coin comes from the user's wallet as a `ShieldedCoinInfo` struct, which is the off-chain representation of a UTXO note.

`sendShielded` releases tokens to a recipient. The recipient is either a `ZswapCoinPublicKey` (a user address) or a `ContractAddress` (another contract).

`mintShieldedToken` creates new tokens in the shielded layer. It requires a domain separator (so each token type is distinct) and a nonce (to prevent replay).

Here's a minimal deposit contract that accepts shielded tokens and records custody:

```compact
pragma language_version >= 1.0.0;

import CompactStandardLibrary;

export ledger total_deposited: Counter;
export ledger reward: QualifiedShieldedCoinInfo;

export circuit deposit(coin: ShieldedCoinInfo): [] {
  // Accept the shielded coin into contract custody
  receiveShielded(disclose(coin));

  // Record where the coin is for later withdrawal
  reward.writeCoin(disclose(coin), right<ZswapCoinPublicKey, ContractAddress>(kernel.self()));

  // Track deposit count publicly
  total_deposited.increment(1uL);
}
```

Notice the split: `receiveShielded` handles the actual UTXO-layer custody, while `total_deposited` is public accounting. The coin amount itself is *not* stored on the public ledger — only the count of deposits. This is intentional. If you stored the amount in `export ledger balance: Uint<64>`, you'd leak the transaction amount publicly.

For minting and immediate transfer to a user:

```compact
pragma language_version >= 1.0.0;

import CompactStandardLibrary;

export ledger domain_separator: Bytes<32>;

export circuit mint_to(
  domainSep: Bytes<32>,
  amount: Uint<64>,
  nonce: Bytes<32>,
  recipient: ZswapCoinPublicKey
): ShieldedSendResult {
  // Mint into contract custody
  const coin = mintShieldedToken(
    disclose(domainSep),
    disclose(amount),
    disclose(nonce),
    right<ZswapCoinPublicKey, ContractAddress>(kernel.self())
  );

  // Immediately send to recipient
  return sendImmediateShielded(
    coin,
    left<ZswapCoinPublicKey, ContractAddress>(disclose(recipient)),
    disclose(amount as Uint<128>)
  );
}
```

The `mintShieldedToken` call creates a coin owned by the contract (`right<..., ContractAddress>(kernel.self())`). `sendImmediateShielded` transfers it to the recipient in the same transaction. The `left<ZswapCoinPublicKey, ...>` wraps the recipient's public key.

### The TypeScript side

On the TypeScript side, constructing a deposit transaction requires building the shielded coin info from the wallet:

```typescript
import { WalletProvider } from '@midnight-ntwrk/midnight-js-wallet';
import { ContractAddress } from '@midnight-ntwrk/ledger';

async function depositToContract(
  wallet: WalletProvider,
  contractAddress: ContractAddress,
  amount: bigint
): Promise<void> {
  // Wallet selects a shielded coin to use
  const coinInfo = await wallet.getShieldedCoinForAmount(amount);

  // Build the deposit call
  const tx = await contract.deposit(coinInfo);
  await wallet.submitTransaction(tx);
}
```

The user's wallet manages coin selection — which UTXO to spend, how to construct the proof. Your TypeScript code doesn't handle the cryptography directly; it passes through the coin info that the wallet generates.

### Important constraint: `receiveShielded` + `addCalls` (v4.0.1 caveat)

If you're on midnight-js v4.0.1, there's a known bug: the `addCalls` API introduced in that version fails for contracts that use `receiveShielded`, `sendShielded`, or `writeCoin`. You'll get `"expected a cell, received null"` errors. The fix is in v4.0.2, which reverts to the original `createUnprovenLedgerCallTx` approach. Always check your SDK version before debugging shielded operation failures.

---

## Model 2: Ledger-State Accounting

### When internal accounting makes sense

Use this model when:
- Token operations aren't available or are temporarily blocked (e.g., KYC gates, time locks)
- You're tracking internal state that isn't actual token ownership
- You're building off-chain-style accounting that mirrors on-chain reality without moving UTXO tokens
- Performance matters more than perfect privacy (ledger reads are cheaper than UTXO operations)
- You need aggregate statistics visible to other contracts or indexers

### The Compact primitives

Counter and Map are the workhorses here.

`Counter` is an unsigned 64-bit integer that increments via `.increment(n: Uint<64>)`. It has a `.value` field for reading.

`Map<K, V>` is a persistent hash map. `.lookup(key)` returns `Option<V>`, not `V` — you must unwrap it. `.insert(key, value)` writes. `.member(key)` returns Boolean.

Here's a ledger-state token implementation:

```compact
pragma language_version >= 1.0.0;

import CompactStandardLibrary;

export ledger balances: Map<Bytes<32>, Uint<64>>;
export ledger total_supply: Counter;
export ledger admin: Bytes<32>;

witness caller_key(): Bytes<32>;

export circuit initialize(admin_key: Bytes<32>): [] {
  admin = admin_key;
}

export circuit mint(to: Bytes<32>, amount: Uint<64>): [] {
  // Access control: only admin can mint
  const caller = caller_key();
  assert caller == admin;

  const current = balances.lookup(to).orDefault(0uL);
  balances.insert(to, current + amount);
  total_supply.increment(amount);
}

export circuit transfer(from: Bytes<32>, to: Bytes<32>, amount: Uint<64>): [] {
  // Caller must prove they are 'from'
  const caller = caller_key();
  assert caller == from;

  const from_bal = balances.lookup(from).orDefault(0uL);
  assert from_bal >= amount;

  balances.insert(from, from_bal - amount);
  const to_bal = balances.lookup(to).orDefault(0uL);
  balances.insert(to, to_bal + amount);
}

export circuit balance_of(addr: Bytes<32>): Uint<64> {
  return balances.lookup(addr).orDefault(0uL);
}
```

Several things to note:

**`Option` unwrapping is required.** `Map.lookup` never returns a `V` directly — it returns `Option<V>`. Using `.orDefault(0uL)` is the idiomatic way to get a zero-initialized default. Passing a `Map.lookup` result directly to an assertion or arithmetic operation is a compile error.

**`caller_key()` is a witness.** Access control in Compact works through witnesses — values the caller provides off-chain that the circuit can use without revealing them on-chain. The `caller_key()` witness lets the circuit assert that the caller knows their own private key, without exposing the key publicly.

**Everything is public.** The `balances` Map is fully visible on-chain. Anyone can query who holds what. This is fundamentally different from shielded tokens.

### A practical hybrid: internal accounting for gated operations

Sometimes you need ledger-state accounting as a temporary holding pattern before token operations are unlocked. This is common in pre-launch contracts:

```compact
pragma language_version >= 1.0.0;

import CompactStandardLibrary;

export ledger credits: Map<Bytes<32>, Uint<64>>;
export ledger tokens_unlocked: Boolean;
export ledger admin: Bytes<32>;

witness caller_key(): Bytes<32>;

export circuit earn_credits(user: Bytes<32>, amount: Uint<64>): [] {
  const caller = caller_key();
  assert caller == admin;

  const current = credits.lookup(user).orDefault(0uL);
  credits.insert(user, current + amount);
}

export circuit unlock_tokens(): [] {
  const caller = caller_key();
  assert caller == admin;
  tokens_unlocked = true;
}

export circuit redeem(
  user: Bytes<32>,
  recipient_key: ZswapCoinPublicKey,
  domain_sep: Bytes<32>,
  mint_nonce: Bytes<32>
): ShieldedSendResult {
  assert tokens_unlocked;

  const caller = caller_key();
  assert caller == user;

  const amount = credits.lookup(user).orDefault(0uL);
  assert amount > 0uL;

  // Zero out the ledger credit before minting
  credits.insert(user, 0uL);

  // Mint real shielded tokens to user
  const coin = mintShieldedToken(
    disclose(domain_sep),
    disclose(amount),
    disclose(mint_nonce),
    right<ZswapCoinPublicKey, ContractAddress>(kernel.self())
  );

  return sendImmediateShielded(
    coin,
    left<ZswapCoinPublicKey, ContractAddress>(disclose(recipient_key)),
    disclose(amount as Uint<128>)
  );
}
```

This pattern is useful for point/credit systems where real tokens are unlocked later. The credits accumulate as public ledger state. On unlock, each user calls `redeem` to convert their credit balance into actual shielded tokens. The `credits.insert(user, 0uL)` before minting prevents double-redemption — it's the functional equivalent of a nullifier check.

---

## Understanding `disclose`: The Bridge Between Layers

Both models require understanding `disclose`, one of the most important keywords in Compact.

In Midnight, circuit inputs are private by default. When you write `export circuit deposit(coin: ShieldedCoinInfo)`, the `coin` parameter is a private witness — the value exists in the ZK proof but not on-chain. This is what makes shielded operations private.

But `receiveShielded`, `sendShielded`, and related functions need to *commit* to values on the public blockchain (so the network can verify the state transition). `disclose` marks a value as public — it passes through the ZK proof boundary and becomes visible in the transaction.

```compact
// Private parameter — coin details stay in the proof
export circuit deposit(coin: ShieldedCoinInfo): [] {
  // disclose makes the commitment visible to the network
  receiveShielded(disclose(coin));
  // total_deposited is a public ledger field — no disclose needed
  total_deposited.increment(1uL);
}
```

The rule of thumb: anything passed to `receiveShielded`, `sendShielded`, or `mintShieldedToken` requires `disclose`. Anything written to an `export ledger` field is already public. The compiler will catch mismatches, but understanding the principle helps you design circuits correctly from the start.

**What `disclose` doesn't do**: It doesn't reveal the coin's value or owner to observers. It reveals the commitment — a hash that proves the coin exists in the tree, without revealing its contents. The actual amount and ownership remain hidden.

---

## State Machine Pattern: Combining Both Models

A common pattern in real Midnight dApps is to use ledger state to control lifecycle phases, with UTXO tokens handling the actual value at each phase. Here's a staking contract that demonstrates this:

```compact
pragma language_version >= 1.0.0;

import CompactStandardLibrary;

// Public state — viewable by anyone
export ledger staking_open: Boolean;
export ledger total_stakers: Counter;
export ledger stake_records: Map<Bytes<32>, Boolean>;
export ledger admin: Bytes<32>;

witness caller_key(): Bytes<32>;

export circuit open_staking(): [] {
  assert caller_key() == admin;
  staking_open = true;
}

export circuit stake(
  user_key: Bytes<32>,
  coin: ShieldedCoinInfo
): [] {
  // Phase gate: staking must be open
  assert staking_open;

  // Prevent double-staking
  assert !stake_records.member(user_key);

  // Verify caller is the user staking
  assert caller_key() == user_key;

  // Accept the shielded coin (private amount, private holder)
  receiveShielded(disclose(coin));

  // Record that this user staked (public, but only a boolean)
  stake_records.insert(user_key, true);
  total_stakers.increment(1uL);
}

export circuit unstake(
  user_key: Bytes<32>,
  recipient_key: ZswapCoinPublicKey,
  coin_to_return: QualifiedShieldedCoinInfo,
  amount: Uint<128>
): ShieldedSendResult {
  assert caller_key() == user_key;
  assert stake_records.lookup(user_key).orDefault(false);

  // Remove stake record
  stake_records.insert(user_key, false);

  // Return tokens privately to user's wallet
  return sendShielded(
    coin_to_return,
    left<ZswapCoinPublicKey, ContractAddress>(disclose(recipient_key)),
    disclose(amount)
  );
}
```

The public ledger (`staking_open`, `total_stakers`, `stake_records`) tracks who's participating and whether staking is active — information that should be queryable. The actual token amounts are entirely in the UTXO layer: the contract holds the shielded coin, and `sendShielded` returns it without revealing the amount on-chain.

Note that `stake_records` only stores a Boolean — whether the user has staked. The amount is not stored. This is deliberate: storing the amount would reveal it publicly. If you need to return exactly what was staked, you pass the `QualifiedShieldedCoinInfo` back through the `unstake` circuit, which identifies which specific coin in the contract's custody to return.

---

## Comparison Table

| Property | UTXO-Layer Tokens | Ledger-State Accounting |
|----------|------------------|------------------------|
| **Privacy** | Amounts + holders private | All balances public |
| **Complexity** | Higher (UTXO proofs, wallet integration) | Lower (simple reads/writes) |
| **Transferability** | Native (send to any address) | Custom (contract must implement transfer) |
| **Gas efficiency** | More expensive (ZK proof generation) | Cheaper (ledger state ops) |
| **Composability** | Interops with Zswap layer | Isolated to contract |
| **Auditability** | Private by default, selective disclosure | Fully transparent |
| **Use case** | Real tokens, payments, DeFi | Bookkeeping, points, gates |

---

## Decision Framework

### Use UTXO-layer tokens when:

**You're building actual financial value transfer.** If users expect to hold, send, and receive tokens with privacy guarantees, UTXO tokens are correct. Internal accounting tokens can't be sent to arbitrary wallets without custom contract logic.

**Privacy of amounts is a requirement.** With ledger-state accounting, every balance is on-chain and queryable. If a compliance attestation system shouldn't reveal exactly how much an address holds, UTXO tokens are the only option.

**You're integrating with Zswap.** If your contract needs to interact with other Midnight protocols that use native tokens, you need to use `receiveShielded` and `sendShielded`. Ledger-state maps are invisible to the broader Zswap ecosystem.

### Use ledger-state accounting when:

**Token operations are blocked.** If your use case involves KYC gates, launch phases, or other scenarios where you want to earn credits before tokens exist, ledger-state accounting is the right intermediate layer. Convert to UTXO tokens at the right moment via the mint-and-send pattern.

**You need public on-chain statistics.** If a governance contract needs to count votes publicly, or a staking contract needs a queryable total staked amount, ledger-state is correct. UTXO operations produce opaque on-chain effects.

**You're building internal accounting that shouldn't be transferable.** Reputation systems, point balances tied to specific accounts, or access counters that shouldn't be tradeable are better as Map entries. Making them UTXO tokens would make them transferable, breaking the intended semantics.

**You're prototyping.** Ledger-state is simpler to develop and test. Start there, then migrate value-bearing state to UTXO tokens when you need privacy or interoperability.

---

## Common Mistakes

### Storing amounts in public ledger when using `receiveShielded`

```compact
// BAD: leaks the deposit amount on-chain
export ledger last_deposit_amount: Uint<64>;

export circuit deposit(coin: ShieldedCoinInfo): [] {
  receiveShielded(disclose(coin));
  last_deposit_amount = coin.value;  // coin.value is private — this won't compile
}
```

`coin.value` is a private witness value. The compiler will reject assigning it to a public ledger field without `disclose`. If you write `last_deposit_amount = disclose(coin.value)`, it will compile but it leaks the deposit amount, defeating the purpose of shielded tokens.

### Using `Map.lookup` without unwrapping

```compact
// BAD: won't compile
const balance = balances.lookup(user);
assert balance >= amount;   // balance is Option<Uint<64>>, not Uint<64>

// CORRECT
const balance = balances.lookup(user).orDefault(0uL);
assert balance >= amount;
```

This is one of the most common Compact compile errors for developers coming from Solidity, where map lookups return zero by default.

### No access control on mint

```compact
// BAD: anyone can mint
export circuit mint(to: Bytes<32>, amount: Uint<64>): [] {
  const current = balances.lookup(to).orDefault(0uL);
  balances.insert(to, current + amount);
}
```

Compact circuits are publicly callable by default. There's no `msg.sender` equivalent — callers don't automatically prove their identity. Add a `witness caller_key(): Bytes<32>` and assert `caller_key() == admin` in any privileged circuit.

---

## Practical Recommendation

Start with the question: "Does this value need to be private, transferable, or interoperable with the broader Midnight token ecosystem?"

If yes to any of those: **UTXO tokens**.

If no — if you're tracking internal state, game scores, points, credits, or any value that's specific to your contract's logic and doesn't need to leave: **ledger-state accounting**.

For most production contracts, you'll end up using both. The public ledger tracks statistics and state flags. Shielded UTXO handles the actual value. The deposit/redemption pattern, where credits accumulate as ledger state and convert to shielded tokens on demand, is a clean way to bridge the two.

---

## Testing Both Models

Testing UTXO-layer contracts requires a different approach than testing ledger-state contracts.

For ledger-state accounting, you can test purely in the Compact simulator. The Map and Counter fields behave like their TypeScript equivalents, and you can write unit tests that call circuits and inspect the resulting state directly.

For UTXO contracts, you need the `@midnight-ntwrk/midnight-js-testing` package, which provides a test wallet and a local Midnight node. The workflow:

```typescript
import { TestWalletProvider } from '@midnight-ntwrk/midnight-js-testing';
import { TestNode } from '@midnight-ntwrk/midnight-js-node-api';

describe('Staking contract', () => {
  let node: TestNode;
  let wallet: TestWalletProvider;

  beforeEach(async () => {
    node = await TestNode.start();
    wallet = await TestWalletProvider.create(node, {
      initialShieldedBalance: 1000n  // fund test wallet with shielded tokens
    });
  });

  it('should accept a stake and return it', async () => {
    const contract = await deployContract(wallet);

    // Fund the wallet with shielded tokens, stake them
    const stakeTx = await contract.stake(wallet.publicKey, wallet.getShieldedCoin(100n));
    await wallet.submitTransaction(stakeTx);

    // Verify staking recorded on public ledger
    const staked = await contract.contractState();
    expect(staked.stake_records.get(wallet.publicKey)).toBe(true);
    expect(staked.total_stakers.value).toBe(1n);

    // Unstake and verify tokens returned
    const unstakeTx = await contract.unstake(wallet.publicKey, wallet.coinPublicKey);
    await wallet.submitTransaction(unstakeTx);

    const afterState = await contract.contractState();
    expect(afterState.stake_records.get(wallet.publicKey)).toBe(false);
  });
});
```

The key difference: testing UTXO operations requires an actual wallet to hold and spend shielded coins. You can't fake a `ShieldedCoinInfo` — the proof would fail verification. This is slower than pure unit tests but is the only way to verify shielded operations actually work end-to-end.

For ledger-state circuits, you can test more simply:

```typescript
import { CompactSimulator } from '@midnight-ntwrk/compact-runtime';

const sim = new CompactSimulator(CounterContract);
await sim.call('mint', ['admin_key', 'user_key', 100n]);
const state = await sim.getState();
expect(state.balances.get('user_key')).toBe(100n);
expect(state.total_supply.value).toBe(100n);
```

Start with simulator tests for ledger-state logic. Add integration tests with a real wallet and node for anything touching UTXO operations.

---

## Summary

Midnight gives you two orthogonal value-tracking systems. The UTXO layer (`receiveShielded`, `sendShielded`, `mintShieldedToken`) provides real privacy at the cost of complexity. The contract ledger (`Counter`, `Map`) provides simple, transparent accounting with no ZK overhead.

Neither is always correct. The architecture that serves you best is usually a combination: use ledger state for what needs to be queryable, and shielded tokens for what needs to be private. Understanding when value belongs on which layer is the fundamental skill for building Midnight dApps that work correctly.
