# Understanding Wallet Sync: Why Your Midnight Deploy Fails Before It Starts

The most common source of confusion when starting with Midnight development isn't the ZK proofs or Compact syntax — it's wallet sync. Developers run the deploy script, hit an error on `balanceUnboundTransaction`, and have no idea why. The error gives you nothing useful. The wallet appears connected. But the transaction fails before the contract even touches the blockchain.

This tutorial explains what wallet sync actually means in Midnight, how the three sub-wallets work, where the known bugs are, and the correct pattern for waiting until a wallet is ready.

---

## What Wallet Sync Does

When you connect a Midnight wallet, it doesn't instantly know your balance or available UTXOs. The wallet needs to scan the blockchain from its last known block (or genesis, for a fresh wallet) and process every shielded note it can decrypt. This is the sync phase.

The reason it has to work this way: shielded transactions on Midnight are encrypted. The wallet has to try decrypting every note on the chain to discover which ones belong to it. This is the same design Zcash uses — full-chain scan by default, with block filtering as an optimization.

Until this scan completes, the wallet's view of your balance is incomplete. Calling `balanceUnboundTransaction` before sync means asking the wallet to select UTXOs from an incomplete inventory. It will either throw an error or — worse — silently succeed with incorrect results.

---

## The Three Sub-Wallets

When you look at `WalletFacade.state()`, the emitted state object has three nested components:

```typescript
interface WalletState {
  shielded: {
    state: {
      progress: SyncProgress;
      balances: Record<string, bigint>;
    };
  };
  unshielded: {
    progress: SyncProgress;
    balances: Record<string, bigint>;
  };
  dust: {
    state: {
      progress: SyncProgress;
    };
    balance: (date: Date) => Record<string, bigint>;
  };
}
```

Each sub-wallet tracks a different type of value on Midnight:

**Shielded wallet** manages shielded (private) UTXOs — the Zswap notes that hold NIGHT tokens with privacy. This is what `receiveShielded` and `sendShielded` interact with. The shielded wallet must scan encrypted note commitments from every block.

**Unshielded wallet** manages transparent UTXOs. These are visible on-chain, similar to Bitcoin's UTXO model but without privacy. When a Midnight transaction has a transparent fee component or moves NIGHT in an unshielded way, it shows up here.

**Dust wallet** manages very small token amounts that don't justify a full UTXO — they're consolidated automatically. The dust wallet has a quirk: its balance depends on a timestamp via `state.dust.balance(new Date(Date.now()))`. This is because dust recombination is time-keyed.

All three must complete sync before the wallet is ready to use. If you check only `shielded.isStrictlyComplete()`, you might proceed while the unshielded wallet is still scanning, which can cause transaction failures for fee payment.

---

## The Safe Sync Pattern

The correct way to wait for wallet readiness is using RxJS operators on the `wallet.state()` observable:

```typescript
import { WalletFacade } from '@midnight-ntwrk/wallet-sdk-facade';
import { firstValueFrom, filter, timeout, tap, throttleTime } from 'rxjs';

async function waitForWalletSync(
  wallet: WalletFacade,
  timeoutMs = 90_000
): Promise<void> {
  await firstValueFrom(
    wallet.state().pipe(
      // Rate-limit the state emissions during scan (can fire very frequently)
      throttleTime(2_000),
      // Optional: log progress
      tap((state) => {
        const s = state.shielded.state.progress.isStrictlyComplete();
        const u = state.unshielded.progress.isStrictlyComplete();
        const d = state.dust.state.progress.isStrictlyComplete();
        console.log(`Sync progress — shielded:${s} unshielded:${u} dust:${d}`);
      }),
      // Wait until ALL THREE are complete
      filter((state) =>
        state.shielded.state.progress.isStrictlyComplete() &&
        state.unshielded.progress.isStrictlyComplete() === true &&
        state.dust.state.progress.isStrictlyComplete()
      ),
      // Fail loudly after timeout instead of hanging forever
      timeout({
        each: timeoutMs,
        with: () => {
          throw new Error(`Wallet sync timeout after ${timeoutMs}ms`);
        }
      })
    )
  );
}

// Usage
const wallet = await WalletFacade.create(config);
await waitForWalletSync(wallet);

// Now safe to use
const tx = await contract.balanceUnboundTransaction(deployParams);
```

A few things to note:

**`throttleTime(2_000)`** is important. The wallet state observable can emit hundreds of times per second during a sync as each block is processed. Without throttling, the `tap` calls and filter checks pile up, slowing the sync. Throttling to 2 seconds keeps the log readable and the event loop clear.

**`state.unshielded.progress.isStrictlyComplete() === true`** uses strict equality. The unshielded progress object's `isStrictlyComplete()` method returns `boolean | undefined` in some SDK versions — when the unshielded wallet hasn't initialized yet, it returns `undefined`. Using `=== true` is safer than a truthy check here.

**`timeout` with `with:`** throws an observable error that `firstValueFrom` converts to a rejected promise. The `each:` form applies the timeout to each emission gap, not the total time — meaning if the wallet is steadily progressing, it won't time out even if total sync takes longer than `timeoutMs`. For a hard total-time limit, use `first:` instead.

---

## The `isStrictlyComplete()` Bug on Idle Chains

There's a known issue with the dust wallet on chains that have low activity: `dust.state.progress.isStrictlyComplete()` never returns `true`, causing the sync to hang indefinitely.

The root cause: `isStrictlyComplete()` checks whether the wallet has caught up to the chain's current head block. On an idle chain (like a local devnet with no transactions), the dust wallet's scan can complete but the progress tracker doesn't register it as "strictly complete" because there are no dust consolidation events to process.

The workaround is to not require dust wallet completion in dev/test environments, or to add a fallback timeout:

```typescript
async function waitForWalletSyncWithDustFallback(
  wallet: WalletFacade,
  timeoutMs = 90_000
): Promise<void> {
  const isDevnet = process.env.NODE_ENV !== 'production';

  await firstValueFrom(
    wallet.state().pipe(
      throttleTime(2_000),
      filter((state) => {
        const shieldedReady = state.shielded.state.progress.isStrictlyComplete();
        const unshieldedReady = state.unshielded.progress.isStrictlyComplete() === true;
        const dustReady = state.dust.state.progress.isStrictlyComplete();

        // On devnet, skip dust requirement to avoid the idle chain bug
        return shieldedReady && unshieldedReady && (dustReady || isDevnet);
      }),
      timeout({
        each: timeoutMs,
        with: () => {
          throw new Error(`Wallet sync timeout — check if chain is producing blocks`);
        }
      })
    )
  );
}
```

This is also the correct pattern for integration tests. Running a full local Midnight node with no activity will trigger the dust bug. Making dust sync optional in test environments keeps your CI from hanging.

If you're seeing this in production (mainnet or testnet), the fix is the same — but the root cause is probably a very slow chain or network connectivity issues. Add logging before the filter to see which sub-wallet is blocked:

```typescript
tap((state) => {
  if (!state.dust.state.progress.isStrictlyComplete()) {
    console.warn('Dust wallet not synced — if this persists, may be the idle chain bug');
  }
})
```

---

## What Happens When You Call `balanceUnboundTransaction` Before Sync

The failure mode isn't always an obvious error. There are three outcomes depending on how far the sync has progressed:

**Case 1: Wallet not started** — `balanceUnboundTransaction` throws immediately with something like `"wallet not initialized"` or a null reference error. This is the easy case.

**Case 2: Partially synced** — The wallet has scanned some blocks but not all. `balanceUnboundTransaction` succeeds, returns a transaction, but it has an incorrect UTXO selection. The transaction might under-pay fees, double-spend a note already used elsewhere, or reference a note that doesn't exist in the current chain state. The transaction is built but fails at submission.

**Case 3: Shielded synced, unshielded not** — Common when the shielded wallet scans quickly but the unshielded wallet lags. The transaction is built correctly for the shielded component but fails on the fee payment, which uses unshielded UTXOs. Error message: `"insufficient unshielded balance"` even though you have funds.

Case 2 and 3 are the frustrating ones. The wallet appears ready, the code doesn't throw, but the transaction fails with a misleading error message. This is why the safe sync pattern matters: it's not just defensive coding, it prevents real transaction failures.

---

## Checking Balance After Sync

Once synced, reading balances correctly requires understanding which sub-wallet holds the tokens you care about:

```typescript
async function getWalletBalances(wallet: WalletFacade): Promise<{
  shielded: bigint;
  unshielded: bigint;
  dust: bigint;
}> {
  const state = await firstValueFrom(wallet.state());
  const now = new Date(Date.now());

  return {
    // NIGHT tokens in shielded notes — private
    shielded: Object.values(state.shielded.state.balances ?? {})
      .reduce((sum, n) => sum + n, 0n),
    // Transparent NIGHT tokens
    unshielded: Object.values(state.unshielded.balances ?? {})
      .reduce((sum, n) => sum + n, 0n),
    // Dust accumulations — time-keyed
    dust: Object.values(state.dust.balance(now) ?? {})
      .reduce((sum, n) => sum + n, 0n),
  };
}
```

For most development work, `shielded` is what you want. The unshielded balance is relevant when you're debugging fee payment issues. Dust is rarely relevant unless you're building dust consolidation logic.

---

## Complete Deploy Pattern

Putting it together: the safe deploy flow that handles sync correctly.

```typescript
import { WalletFacade } from '@midnight-ntwrk/wallet-sdk-facade';
import { NetworkId } from '@midnight-ntwrk/midnight-js-network-id';
import { firstValueFrom, filter, throttleTime, timeout } from 'rxjs';

async function deployContract(config: DeployConfig): Promise<ContractAddress> {
  // 1. Create and connect wallet
  const wallet = await WalletFacade.create({
    networkId: NetworkId.TestNet,
    indexerWsUrl: config.indexerUrl,
    proofServerUrl: config.proofServerUrl,
  });

  // 2. Wait for full sync — don't skip this
  console.log('Waiting for wallet sync...');
  await waitForWalletSync(wallet);
  console.log('Wallet synced.');

  // 3. Log balances so you know what you're working with
  const { shielded, unshielded } = await getWalletBalances(wallet);
  console.log(`Balances — shielded: ${shielded}, unshielded: ${unshielded}`);

  if (shielded === 0n) {
    throw new Error('No shielded balance — get testnet NIGHT from the faucet first');
  }

  // 4. Now safe to deploy
  const deployTx = await contract.deploy(deployParams);
  const result = await wallet.submitTransaction(deployTx);

  return result.contractAddress;
}
```

The `if (shielded === 0n)` check is worth adding explicitly. A common first-run failure is forgetting to fund the test wallet from the faucet. The error from `balanceUnboundTransaction` with zero balance is not always obvious.

---

## Summary

Wallet sync on Midnight involves three independent sub-wallets: shielded, unshielded, and dust. All three must complete before calling `balanceUnboundTransaction` or any transaction-building API. The safe pattern is `wallet.state().pipe(filter(s => allThreeComplete(s)))` with RxJS operators.

The dust wallet has a known bug on idle chains where `isStrictlyComplete()` never returns `true`. Work around it by making dust sync optional in dev/test environments.

Partial sync failures — where only some sub-wallets have synced — produce misleading errors at transaction submission time rather than at the `balanceUnboundTransaction` call. This makes them harder to debug. The correct response is not to add retry logic around `balanceUnboundTransaction` but to fix the sync wait before it.
