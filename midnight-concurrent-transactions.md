# Concurrent Transactions on Midnight: UTXO Race Conditions and How to Handle Them

When two users submit transactions to the same contract simultaneously, one of them is going to fail. This isn't a bug — it's a fundamental property of UTXO-based blockchains. Understanding exactly why it happens and how to design around it is the difference between a Midnight contract that works under load and one that randomly fails in production.

This tutorial explains the UTXO race condition, shows you how to reproduce it in a test, walks through the patterns that mitigate it, and covers the specific Midnight SDK tooling for handling submission failures.

---

## Why Race Conditions Happen

Midnight transactions consume UTXOs. A UTXO can only be consumed once. When you call `balanceUnboundTransaction`, the wallet selects UTXOs from its current view of your balance to fund the transaction. If two transactions are built simultaneously — each selecting the same UTXO as an input — only the first one to be included in a block will succeed. The second will be rejected because its input UTXO has already been spent.

This is identical to Bitcoin's double-spend prevention, applied to application-layer transactions.

For private (shielded) transactions on Midnight, the same principle holds for Zswap notes. A shielded note is a one-time-use UTXO. If two transactions reference the same note, only one can succeed.

The second dimension: contract ledger state. Even if both transactions use different UTXOs, if they both try to modify the same ledger field (a Counter, a Map entry), the second one may fail if the first's state change invalidates the second's ZK proof. The ZK proof was generated against a specific ledger state. If the ledger has moved since proof generation, the proof may no longer be valid.

---

## Reproducing the Race Condition

Understanding the failure mode requires seeing it happen. Here's a minimal test that produces it:

```typescript
import { WalletFacade } from '@midnight-ntwrk/wallet-sdk-facade';
import { TestWalletProvider } from '@midnight-ntwrk/testkit-js';

describe('race condition demo', () => {
  it('demonstrates concurrent submission failure', async () => {
    const wallet = await TestWalletProvider.create();
    await waitForSync(wallet);

    // A shared counter contract
    const contract = await deployCounterContract(wallet);

    // Two independent wallets, same contract
    const alice = await TestWalletProvider.create();
    const bob = await TestWalletProvider.create();
    await fundWallet(alice, 1000n);
    await fundWallet(bob, 1000n);
    await waitForSync(alice);
    await waitForSync(bob);

    // Both build transactions at the same time
    // (same ledger state, same block height)
    const [aliceTx, bobTx] = await Promise.all([
      contract.callTx.increment(),
      contract.callTx.increment(),
    ]);

    // Submit both simultaneously
    const results = await Promise.allSettled([
      alice.submitTransaction(aliceTx),
      bob.submitTransaction(bobTx),
    ]);

    const succeeded = results.filter(r => r.status === 'fulfilled').length;
    const failed = results.filter(r => r.status === 'rejected').length;

    // One or both will fail if they share UTXO inputs
    console.log(`Succeeded: ${succeeded}, Failed: ${failed}`);
    // In practice: either 1/1 or 0/2, rarely 2/2
  });
});
```

The outcome depends on timing and UTXO selection. In a test environment with a single funded wallet, both transactions will often select the same UTXO as a fee input, causing one to fail. In a production environment with multiple UTXOs in a wallet, the failure rate depends on how many UTXOs are available versus how many concurrent transactions are in flight.

---

## The Three Failure Modes

### Failure Mode 1: UTXO Conflict (Fee Inputs)

The most common failure. Both transactions select the same shielded note as a fee input. The second transaction is rejected with something like `"note already spent"` or `"invalid UTXO"`.

**Detection**: The rejection happens at submission time, not at ZK proof generation time. The proof is valid — the error is at the blockchain layer.

**Rate**: High when a wallet has few UTXOs. A wallet with 1 UTXO that funds every transaction will fail 100% of concurrent submissions. A wallet with 100 UTXOs will fail much less often.

### Failure Mode 2: Ledger State Conflict

Transaction A and Transaction B both read the current value of a Counter, compute proofs based on it, then try to submit. A succeeds and increments the counter. B's proof was generated against the old counter value — the proof is now invalid.

**Detection**: The rejection includes a reference to the proof being invalid or the state being inconsistent. This can surface as a ZK verification failure at the node level.

**Rate**: Depends on how many state reads the circuit does. Contracts that read only from Map entries they themselves write are less vulnerable than contracts that read shared global state.

### Failure Mode 3: Sequence Number Conflict

Some contract patterns use an explicit sequence number or nonce to order operations. If two transactions both increment sequence number 5 to 6, only one can succeed.

**Detection**: Usually looks like an assertion failure inside the contract — the `assert sequence == next_expected` fails.

**Rate**: 100% in any concurrent scenario — by design. Sequence numbers are the anti-concurrency mechanism.

---

## Mitigation Pattern 1: Retry with Backoff

The simplest mitigation. When a submission fails due to a UTXO conflict, rebuild the transaction (selecting fresh UTXOs from the updated wallet state) and resubmit.

```typescript
async function submitWithRetry(
  wallet: WalletFacade,
  buildTx: () => Promise<any>,
  maxRetries = 5,
  baseDelayMs = 500
): Promise<any> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      // Rebuild the transaction each time — don't reuse the old one
      const tx = await buildTx();
      return await wallet.submitTransaction(tx);
    } catch (err) {
      const isRetryable = isUTXOConflict(err) || isStateConflict(err);
      if (!isRetryable || attempt === maxRetries - 1) {
        throw err;
      }
      // Exponential backoff with jitter
      const delay = baseDelayMs * Math.pow(2, attempt) + Math.random() * 100;
      console.warn(`Tx attempt ${attempt + 1} failed, retrying in ${delay.toFixed(0)}ms`);
      await new Promise(r => setTimeout(r, delay));
      // Wait for wallet to see the conflicting tx and update state
      await waitForWalletSync(wallet);
    }
  }
}

function isUTXOConflict(err: unknown): boolean {
  const msg = String(err);
  return msg.includes('note already spent') ||
         msg.includes('invalid UTXO') ||
         msg.includes('double spend');
}

function isStateConflict(err: unknown): boolean {
  const msg = String(err);
  return msg.includes('invalid proof') ||
         msg.includes('state mismatch') ||
         msg.includes('inconsistent state');
}
```

**Important**: Rebuild the transaction on every retry. Do not resubmit the same `BalancedTransaction` object. The old transaction's UTXO inputs are already spent (or conflicted). The new transaction must select fresh inputs from the wallet's updated state.

The `waitForWalletSync` call before retry is critical. Without it, the wallet hasn't processed the block that spent the conflicting UTXO, so it will select the same (now-spent) UTXO again on the next attempt.

### Choosing Retry Parameters

| Scenario | Max Retries | Base Delay |
|----------|-------------|------------|
| Interactive UI | 3 | 1000ms |
| Background service | 10 | 200ms |
| High-frequency bot | 20 | 50ms |

For interactive applications, fail fast after 3 retries and show an error. For background services, retry more aggressively. The base delay of 200-500ms gives the mempool time to process and the wallet time to sync.

---

## Mitigation Pattern 2: UTXO Pool Management

For high-throughput applications (NFT mints, airdrop distribution, batch processing), a single wallet becomes a bottleneck. The solution is a pool of wallets, each with independent UTXOs.

```typescript
class UTXOPool {
  private wallets: WalletFacade[];
  private available: boolean[];

  constructor(wallets: WalletFacade[]) {
    this.wallets = wallets;
    this.available = new Array(wallets.length).fill(true);
  }

  async acquire(): Promise<{ wallet: WalletFacade; release: () => void }> {
    // Find an available wallet
    while (true) {
      const idx = this.available.findIndex(a => a);
      if (idx !== -1) {
        this.available[idx] = false;
        const wallet = this.wallets[idx];
        const release = () => { this.available[idx] = true; };
        return { wallet, release };
      }
      // All wallets busy — wait
      await new Promise(r => setTimeout(r, 50));
    }
  }
}

// Setup: 5 wallets, each pre-funded
const pool = new UTXOPool(await Promise.all(
  Array.from({ length: 5 }, async () => {
    const w = await WalletFacade.create(config);
    await waitForWalletSync(w);
    return w;
  })
));

// Usage: each concurrent operation gets its own wallet
async function processItem(item: Item): Promise<void> {
  const { wallet, release } = await pool.acquire();
  try {
    const tx = await contract.callTx.process(item.id);
    await wallet.submitTransaction(tx);
  } finally {
    release();
  }
}
```

With 5 independent wallets, up to 5 transactions can be submitted simultaneously without any UTXO conflicts between them. The pool serializes access per-wallet but parallelizes across the pool.

**Funding consideration**: Each pool wallet needs its own shielded balance for fees. Pre-fund each wallet from a treasury wallet before starting the pool. In testnet, use the faucet N times.

---

## Mitigation Pattern 3: UTXO Splitting

A single wallet can increase its parallelism by splitting its balance into many small UTXOs, each capable of funding a transaction.

```typescript
async function splitUTXOs(
  wallet: WalletFacade,
  targetCount: number
): Promise<void> {
  const state = await firstValueFrom(wallet.state());
  const balance = getShieldedBalance(state);
  if (balance === 0n) throw new Error('No balance to split');

  const splitAmount = balance / BigInt(targetCount) - FEE_RESERVE;
  const recipients = Array.from({ length: targetCount - 1 }, () => ({
    address: wallet.address, // send to self
    amount: splitAmount,
  }));

  // Self-transfer splits the UTXO into N pieces
  const tx = await wallet.buildShieldedTransfer(recipients);
  await wallet.submitTransaction(tx);
  await waitForWalletSync(wallet);
  console.log(`Balance split into ${targetCount} UTXOs`);
}
```

After splitting, the wallet has `targetCount` UTXOs, and concurrent transactions that happen to select different UTXOs won't conflict. The conflict probability drops from near-100% to near-1/N² for N UTXOs and 2 concurrent transactions.

**Caution**: Splitting too aggressively (hundreds of dust UTXOs) will slow down wallet sync significantly, since the wallet has to track each note. Keep the split count practical: 10-20 UTXOs for most use cases.

---

## Mitigation Pattern 4: Optimistic Concurrency for Ledger State

For ledger state conflicts (not UTXO conflicts), the retry pattern alone may not be sufficient if the contract uses monotonic state that callers must read before submitting.

The pattern: add a version field to the ledger, include it in the circuit's inputs, and fail fast if the version has moved:

```compact
export ledger data: Map<Bytes<32>, Uint<64>>;
export ledger version: Uint<64>;

export circuit update(
  key: Bytes<32>,
  new_value: Uint<64>,
  expected_version: Uint<64>
): [] {
  // Optimistic concurrency check — fail if state has moved
  assert version == expected_version;
  data.insert(key, new_value);
  version = version + 1uL;
}
```

The caller reads `version` before building the transaction, includes it in the circuit call, and the circuit rejects if anyone else has modified state since then. This turns an opaque ZK proof failure into a deterministic assertion failure, making the retry logic cleaner.

On retry: re-read `version`, rebuild the transaction, resubmit. Exponential backoff applies here too.

---

## Mitigation Pattern 5: Queue-Based Serialization

For operations that must be strictly ordered (and where concurrency is undesirable), use an off-chain queue with a single submitter:

```typescript
class SerializedSubmitter {
  private queue: Array<{
    buildTx: () => Promise<any>;
    resolve: (result: any) => void;
    reject: (err: any) => void;
  }> = [];
  private processing = false;

  submit(buildTx: () => Promise<any>): Promise<any> {
    return new Promise((resolve, reject) => {
      this.queue.push({ buildTx, resolve, reject });
      this.process();
    });
  }

  private async process(): Promise<void> {
    if (this.processing) return;
    this.processing = true;

    while (this.queue.length > 0) {
      const { buildTx, resolve, reject } = this.queue.shift()!;
      try {
        const tx = await buildTx();
        const result = await wallet.submitTransaction(tx);
        resolve(result);
        // Wait for tx to be confirmed before next submission
        await waitForConfirmation(result.txHash);
      } catch (err) {
        reject(err);
      }
    }

    this.processing = false;
  }
}
```

The serialized submitter processes one transaction at a time, waiting for confirmation before submitting the next. This eliminates all concurrency at the cost of throughput. Use for contract initialization, configuration changes, and other infrequent critical operations.

---

## Monitoring Failure Rates

In production, track submission failures to detect concurrency issues:

```typescript
const metrics = {
  submissions: 0,
  conflicts: 0,
  retries: 0,
};

async function submitTracked(
  wallet: WalletFacade,
  buildTx: () => Promise<any>
): Promise<any> {
  metrics.submissions++;
  try {
    return await submitWithRetry(wallet, buildTx, 5, 500);
  } catch (err) {
    if (isUTXOConflict(err) || isStateConflict(err)) {
      metrics.conflicts++;
      throw err;
    }
    throw err;
  }
}
```

A conflict rate above 5% under normal load indicates you need more UTXOs per wallet or a larger wallet pool. Above 20% means the concurrency pattern needs a fundamental redesign.

---

## Testing Concurrent Behavior

The Midnight testkit enables controlled concurrency testing. Key scenarios to test:

```typescript
describe('concurrency', () => {
  it('handles N concurrent submissions from one wallet', async () => {
    const wallet = await createFundedWallet();
    await splitUTXOs(wallet, 10); // ensure 10 UTXOs

    const txCount = 5;
    const results = await Promise.allSettled(
      Array.from({ length: txCount }, () =>
        submitWithRetry(wallet, () => contract.callTx.increment())
      )
    );

    const succeeded = results.filter(r => r.status === 'fulfilled').length;
    // With retry + UTXO split, most should succeed
    expect(succeeded).toBeGreaterThanOrEqual(txCount - 1);
  });

  it('handles N concurrent submissions from N wallets', async () => {
    const wallets = await Promise.all(
      Array.from({ length: 5 }, createFundedWallet)
    );

    const results = await Promise.allSettled(
      wallets.map(w =>
        submitWithRetry(w, () => contract.callTx.increment())
      )
    );

    // Different wallets → different UTXOs → no conflicts
    const succeeded = results.filter(r => r.status === 'fulfilled').length;
    expect(succeeded).toBe(wallets.length);
  });
});
```

---

## Summary

UTXO race conditions on Midnight happen at two layers: the UTXO input layer (conflicting note spends) and the ledger state layer (conflicting ZK proofs against moved state). The failure mode for both is a rejected transaction at submission time.

Mitigation options in order of simplicity:

1. **Retry with backoff**: Always rebuild the transaction on retry. Wait for wallet sync before each attempt. Works for most applications.
2. **UTXO splitting**: Increase the number of UTXOs per wallet to reduce conflict probability in concurrent single-wallet workflows.
3. **Wallet pool**: N independent wallets eliminate UTXO conflicts for N concurrent operations at the cost of pre-funding and management overhead.
4. **Optimistic concurrency versioning**: Use version fields in the contract to convert opaque ZK failures to deterministic assertion failures.
5. **Serialized submission**: For strict ordering requirements, queue transactions and submit sequentially with confirmation waits.

Choose based on your throughput requirements. Single-user applications: retry + UTXO split. Multi-user applications: retry + wallet pool. Low-throughput critical operations: serialized submission.
