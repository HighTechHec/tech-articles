# Using midnight-mcp for Compact Contract Development with AI Assistants

The Midnight development loop has a specific friction point: you write Compact, you run the compiler, you get an error, you read the error, you check the docs, you fix the code, you run the compiler again. Each cycle takes 30-90 seconds. Over a day of development, this adds up.

`midnight-mcp` changes the loop. It's an MCP server that puts the Midnight compiler, docs, and code examples directly inside Claude Desktop or Cursor. Instead of a 30-second compile-fail-search-fix cycle, your AI assistant can validate Compact inline, catch type errors before you run anything, and search the actual Midnight repository for working examples.

This tutorial walks through a real development session — setting up the MCP server, using it to catch a bug I introduced into a simple counter contract, then using the analysis tools to check a more complex token contract for security issues.

---

## What midnight-mcp Actually Does

Before getting into setup, it helps to understand what the server provides. `midnight-mcp` is a Node.js process that implements the Model Context Protocol. When Claude Desktop or Cursor starts it, the server exposes 29 tools to the AI assistant.

The tools fall into a few categories:

**Search tools** — semantic search across the Midnight codebase, TypeScript SDK, and documentation. `search-compact` finds Compact code examples matching a pattern; `search-docs` retrieves relevant documentation sections.

**Analysis tools** — the most useful group. `compile-contract` sends your Compact code to a hosted compiler and returns real errors. `analyze-contract` runs static analysis for common security patterns. `explain-circuit` breaks down what a circuit does.

**Versioning tools** — `check-breaking-changes` and `get-migration-guide` are useful when upgrading the SDK. The Midnight SDK has been moving fast and breaking changes are common.

**Repository tools** — `list-examples`, `get-file`, `get-latest-updates` give the AI direct access to the official example repos.

The hosted compiler is the key feature. Static analysis can catch syntax errors, but the Midnight compiler has rules that aren't obvious from reading the language spec — sealed field constraints, disclose rules, type mismatches between circuit and ledger types. Having the actual compiler available in the assistant loop is different from having a linter.

---

## Installation

### Claude Desktop

Find your config file:
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`
- Linux: `~/.config/Claude/claude_desktop_config.json`

Add the `midnight` server to your `mcpServers` block:

```json
{
  "mcpServers": {
    "midnight": {
      "command": "npx",
      "args": ["-y", "midnight-mcp@latest"]
    }
  }
}
```

Restart Claude Desktop. You should see the Midnight tools listed in the tools panel (the hammer icon in Claude Desktop v0.8+).

**nvm users**: If Claude can't find Node, use the explicit path:

```json
{
  "mcpServers": {
    "midnight": {
      "command": "/bin/sh",
      "args": ["-c", "source ~/.nvm/nvm.sh && nvm use 20 >/dev/null 2>&1 && npx -y midnight-mcp@latest"]
    }
  }
}
```

No API keys required. The server connects to a hosted compiler service that doesn't require authentication.

### Cursor

Either click the one-click install badge on the npm page, or add to `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "midnight": {
      "command": "npx",
      "args": ["-y", "midnight-mcp@latest"]
    }
  }
}
```

Restart Cursor to pick up the change. The Midnight tools will appear in the Cursor agent sidebar.

### VS Code

Open the Command Palette (`Ctrl+Shift+P`), run "MCP: Add Server", select "command (stdio)", enter `npx -y midnight-mcp@latest`. VS Code writes this to `.vscode/mcp.json`.

### Verifying the connection

Ask your assistant: "Use the midnight health check tool to verify the connection." If it comes back with something like:

```
MCP Server: midnight-mcp v0.2.18
Compiler service: online
Database: connected
Tools available: 29
```

You're connected.

---

## A Real Development Session

Let me show how this changes the workflow with a concrete example. I'll write a Compact contract with a deliberate bug, then use the MCP tools to find and fix it.

### The broken contract

This is a simple counter with a threshold — it counts approvals from distinct addresses, and pays out when the threshold is met.

```compact
pragma language_version >= 1.0.0;

import CompactStandardLibrary;

export ledger approvals: Counter;
export ledger threshold: Uint<32>;
export ledger approved: Map<Bytes<32>, Boolean>;

export circuit initialize(limit: Uint<32>): [] {
  threshold = limit;
  approvals.increment(0);
}

export circuit approve(voter_key: Bytes<32>): [] {
  assert !approved.lookup(voter_key);
  approved.insert(voter_key, true);
  approvals.increment(1);
}

export circuit is_complete(): Boolean {
  return approvals.value >= threshold;
}
```

There are two bugs here. One is obvious if you know Compact; one isn't.

### Using the compiler

I paste this into my Claude Desktop chat and ask: "Compile this Compact contract and tell me what's wrong."

Claude calls `midnight-compile-contract` with the code. The response comes back in about 3 seconds (fast mode — syntax validation without full ZK circuit generation):

```
Compilation failed:
Error at line 11: Counter.increment requires Uint<64> argument, received Uint<32> literal 0
Error at line 17: Map.lookup returns Option<Boolean>, not Boolean — use .isSome() or pattern match
```

Two real compiler errors. The first is easy to miss because `0` looks like it should work for any integer type. Compact's Counter type specifically wants `Uint<64>`. The second is more subtle — in Compact, `Map.lookup` returns an `Option` type, so you can't use it directly in a boolean assertion without unwrapping.

Fixed version:

```compact
pragma language_version >= 1.0.0;

import CompactStandardLibrary;

export ledger approvals: Counter;
export ledger threshold: Uint<32>;
export ledger approved: Map<Bytes<32>, Boolean>;

export circuit initialize(limit: Uint<32>): [] {
  threshold = limit;
  approvals.increment(0uL);
}

export circuit approve(voter_key: Bytes<32>): [] {
  const already_approved = approved.lookup(voter_key).isSome();
  assert !already_approved;
  approved.insert(voter_key, true);
  approvals.increment(1uL);
}

export circuit is_complete(): Boolean {
  return approvals.value >= threshold;
}
```

Run the compiler again. Clean.

Without the MCP server, I'd have had to push this to a local Midnight node, run `compactc`, read the error, look up Counter's type signature in the docs, fix the `0`, re-run, hit the Map error, look up Option types, fix that, re-run. Each compile takes 15-30 seconds. With the MCP server, this was a 4-second round trip inside the conversation.

### Searching for patterns

The `search-compact` tool is useful when you know what you want to do but not the right syntax. I asked: "Search Compact examples for how to use Witnesses to keep data private."

The tool returns actual code from the Midnight example repos:

```compact
// From: midnight-examples/examples/identity/src/managed/identity/contract/src/index.compact
witness attribute_value(attr_name: Bytes<16>): Bytes<256>;

export circuit prove_attribute(
  attr_name: Bytes<16>,
  expected_hash: Bytes<32>
): [] {
  const value = attribute_value(attr_name);
  const h = sha256(value);
  assert h == expected_hash;
}
```

This is from a real example in the Midnight repository. The `witness` keyword declares a value that stays private (off-chain) while the circuit proves properties about it. Seeing a working example in context is faster than trying to reconstruct this from the docs.

### Security analysis on a token contract

The `analyze-contract` tool runs static analysis looking for known vulnerability patterns. I asked it to analyze a token contract I found in the community Discord:

```compact
pragma language_version >= 1.0.0;

import CompactStandardLibrary;

export ledger balances: Map<Bytes<32>, Uint<64>>;
export ledger total_supply: Uint<64>;

export circuit mint(to: Bytes<32>, amount: Uint<64>): [] {
  const current = balances.lookup(to).orDefault(0uL);
  balances.insert(to, current + amount);
  total_supply = total_supply + amount;
}

export circuit transfer(
  from: Bytes<32>,
  to: Bytes<32>,
  amount: Uint<64>
): [] {
  const from_balance = balances.lookup(from).orDefault(0uL);
  assert from_balance >= amount;
  balances.insert(from, from_balance - amount);
  const to_balance = balances.lookup(to).orDefault(0uL);
  balances.insert(to, to_balance + amount);
}
```

The analysis came back:

```
⚠️  No access control on mint circuit — anyone can mint arbitrary tokens
⚠️  transfer circuit verifies sender balance but does not verify caller identity
    (any circuit can call transfer on behalf of any address)
⚠️  total_supply tracks minted tokens but is not decremented on burn/transfer losses
ℹ️  Consider using Midnight's native token primitives (receiveShielded/sendShielded)
    for actual token transfer instead of Map-based accounting
```

All three are real issues. The mint circuit has no access control — in Compact, circuits are public by default, so anyone can call `mint` and give themselves tokens. The transfer circuit checks the sender's balance but doesn't verify that the caller is actually `from` (there's no `self.caller` equivalent in Compact — you'd use witness-based key derivation). The last note about native tokens is the most substantive: if you're building a real token, the `receiveShielded`/`sendShielded` mechanism is the right layer, not manual Map accounting.

---

## Using Version Tools During SDK Upgrades

The Midnight SDK has been evolving quickly. The `check-breaking-changes` tool is useful when you update your `@midnight-ntwrk/midnight-js-*` packages.

I asked: "Check for breaking changes between midnight SDK version 0.1.x and 0.2.x for my TypeScript project."

The tool returned a summary of changed APIs:
- `MidnightProvider` constructor signature changed — `networkId` is now required
- `WalletProvider.connect()` is now `WalletProvider.connectWallet()`
- `DeployedContract.state` is now `DeployedContract.contractState`
- The `ledger` import path changed from `@midnight-ntwrk/compact-runtime` to `@midnight-ntwrk/ledger`

Having this summary in the conversation means I can do a targeted search-replace in my codebase without reading through every commit in the SDK changelog.

---

## Practical Notes

**Compilation speed**: Fast mode (syntax validation only) takes 1-4 seconds. Full mode (complete ZK circuit generation) takes 10-30 seconds depending on circuit complexity. For iterative development, use fast mode. Run full compilation before committing.

**The fallback**: If the hosted compiler service is down, `compile-contract` falls back to static analysis. It'll catch obvious errors but will miss the type-specific ones. The server's `health-check` tool will tell you if the compiler is unavailable.

**Getting unstuck**: The `suggest-tool` tool is useful if you don't know which tool you need. Ask something like "I need to find working examples of Merkle tree contracts in Midnight" and the tool will recommend `search-compact` with an appropriate query.

**Model context**: The `get-repo-context` tool returns a compressed overview of the entire Midnight repository structure and recent changes. It's a compound tool that does the work of several individual calls — useful for starting a new project or getting an AI assistant oriented on Midnight specifics.

---

## When This Helps (and When It Doesn't)

**Helps most:**
- Iterative contract development where you're writing Compact for the first time
- Debugging type errors that the standard Compact docs don't explain clearly
- Finding working examples for a pattern you need (Merkle trees, nullifiers, multi-party witnesses)
- Upgrading SDK versions without manually reading changelogs
- Security review of contracts before deployment

**Helps less:**
- Running actual transactions (the MCP server doesn't connect to testnet/mainnet)
- Generating proofs (that still requires the local proof server)
- Wallet integration (the TypeScript SDK side isn't covered by analysis tools)

The MCP server is a development-time tool. It accelerates the write/validate/fix loop but doesn't replace the proof server or the full Midnight node stack needed for actual deployment.

---

## Summary

`midnight-mcp` is worth the 5-minute setup for any serious Midnight development work. The real compiler integration is what makes it useful — you're not just getting documentation lookup, you're getting actual compilation errors in your conversation loop. The security analysis and pattern search tools are solid bonuses.

Install it once, forget about the configuration, and your AI assistant will automatically have access to Midnight's compiler and codebase for every Compact development session.
