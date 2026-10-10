<p align="center">
  <a href="https://three.ws"><img src="https://three.ws/three-ws-mcp-icon.svg" alt="three.ws" width="88" height="88"></a>
</p>

<h1 align="center">@three-ws/portfolio-mcp</h1>

<p align="center"><strong>An AI agent's own trading state over MCP — portfolio value, PnL, live balances, the public trade feed, and one real write: a signed on-chain Solana transfer.</strong></p>

<p align="center">
  <a href="https://www.npmjs.com/package/@three-ws/portfolio-mcp"><img alt="npm" src="https://img.shields.io/npm/v/@three-ws/portfolio-mcp?logo=npm&color=cb3837"></a>
  <img alt="license" src="https://img.shields.io/npm/l/@three-ws/portfolio-mcp?color=3b82f6">
  <img alt="node" src="https://img.shields.io/node/v/@three-ws/portfolio-mcp?color=339933&logo=node.js">
  <a href="https://registry.modelcontextprotocol.io/?q=io.github.nirholas"><img alt="MCP Registry" src="https://img.shields.io/badge/MCP%20Registry-io.github.nirholas-0ea5e9"></a>
  <a href="https://three.ws"><img alt="three.ws" src="https://img.shields.io/badge/built%20by-three.ws-000"></a>
</p>

---

> A [Model Context Protocol](https://modelcontextprotocol.io) server that gives an AI agent programmatic access to its **own** trading state over stdio: what it holds, how that value moves over time, a single token's position with live market data, the platform-wide feed of closed trades and their **realized PnL**, live on-chain balances for any wallet — and one real write, a **signed Solana transfer** of SOL or any SPL token.

Reads hit the live three.ws API and Solana RPC — nothing is mocked. The account-scoped reads run against the agent wallets **you own** (resolved from your three.ws session). The one write, `send_transfer`, signs **locally** with your own key and moves real funds on mainnet.

## Install

```bash
npm install @three-ws/portfolio-mcp
```

Or run with `npx` (no install):

```bash
npx @three-ws/portfolio-mcp
```

## Quick start

**Claude Code**, one line:

```bash
claude mcp add portfolio -- npx -y @three-ws/portfolio-mcp
```

**Claude Desktop / Cursor** (`claude_desktop_config.json` or `mcp.json`):

```json
{
	"mcpServers": {
		"portfolio": {
			"command": "npx",
			"args": ["-y", "@three-ws/portfolio-mcp"],
			"env": {
				"THREE_WS_SESSION": "<your __Host-sid cookie>",
				"SOLANA_SECRET_KEY": "<base58 secret of the sending wallet>"
			}
		}
	}
}
```

`THREE_WS_SESSION` is only needed for the account-scoped reads; `SOLANA_SECRET_KEY` is only needed for `send_transfer`. The trade feed and on-chain balance reads work with no configuration at all.

Inspect the surface with the MCP Inspector:

```bash
npx -y @modelcontextprotocol/inspector npx @three-ws/portfolio-mcp
```

## Tools

| Tool                     | Type                  | What it does                                                                                                          |
| ------------------------ | --------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `get_portfolio_summary`  | read · session        | Live holdings + USD value across every agent wallet you own, with a per-wallet breakdown. Optional snapshot.         |
| `get_portfolio_history`  | read · session        | Your total portfolio USD value over time, from stored snapshots — the performance chart.                            |
| `get_portfolio_asset`    | read · session        | One token deep-dive: which wallets hold it, your total amount/USD, plus live market data and a price-history chart.  |
| `get_trades_feed`        | read · public         | The platform-wide feed of closed positions and their **realized PnL** — newest exit first, filterable by coin.       |
| `get_wallet_balances`    | read · public         | Live on-chain SOL + SPL (incl. Token-2022) balances for any Solana address, straight from the RPC.                  |
| `send_transfer`          | **write · destructive** | Sign and broadcast a real, irreversible Solana mainnet transfer of SOL or any SPL token from your own wallet.      |

Everything reads live state — balances, feeds, and broadcasts all move between calls — so no tool is idempotent. `send_transfer` is the only tool that mutates anything.

### Input parameters

**`get_portfolio_summary`** — `snapshot` (bool, default false — also persist a history point).

**`get_portfolio_history`** — `days` (1–365, default 90).

**`get_portfolio_asset`**: `chain` (`solana` | `evm`, required), `id` (`native` or the base58 mint / 0x contract, required), `days` (1 to 365, default 30), `chain_id` (optional EVM chain id: 8453 Base, 1 Ethereum, 42161 Arbitrum, 10 Optimism, 137 Polygon, 56 BNB, and the other chains the platform indexes). Leave `chain_id` off and the API prices the contract on whatever chain your wallets hold it on, defaulting to Base; set it when the same address exists on more than one chain.

**`get_trades_feed`** — `network` (`mainnet` | `devnet`, default mainnet), `window` (`1h` | `6h` | `24h` | `7d` | `30d` | `all`, default 24h), `min_pnl_pct` (default 10), `limit` (1–80, default 40), `cursor` (ISO timestamp for pagination), `mint` (filter to one coin).

**`get_wallet_balances`** — `address` (base58, required), `include_tokens` (bool, default true).

**`send_transfer`** — `recipient` (base58, required), `amount` (decimal string, required), `mint` (base58 SPL mint; omit / `native` for SOL), `secret` (base58, overrides `SOLANA_SECRET_KEY`), `priorityMicroLamports` (0–50,000,000), `confirm` (must be `true` to execute).

## Realized vs. live value

Two different questions, two different tools:

- **"What am I worth right now?"** → `get_portfolio_summary` / `get_portfolio_asset` — live holdings priced in USD.
- **"How have my closed trades done?"** → `get_trades_feed` — realized PnL per closed position (entry/exit in SOL, profit %, multiple, hold time).

## The write tool — `send_transfer`

`send_transfer` **broadcasts a real, irreversible transaction on Solana mainnet** and moves funds out of the signing wallet. It signs locally with the per-call `secret` or `SOLANA_SECRET_KEY` — the three.ws API never holds your key. Guardrails:

- **`confirm: true` required** while `REQUIRE_CONFIRM` is on (the default) — the first call without it returns a refusal, not a broadcast.
- **`MAX_SOL_PER_TX`** caps native SOL sends (default 0.5 SOL).
- **`RECIPIENT_ALLOWLIST`**, when set, restricts the destination to a known set.
- The `mint` is **runtime input** — pass the `$THREE` mint or any SPL token you hold. No mint is hardcoded.

For an SPL transfer it reads the mint's decimals on-chain, verifies your balance, and auto-creates the recipient's associated token account when missing. On a confirmation timeout it returns `tx_unconfirmed` with the signature rather than silently retrying — check Solscan before resending to avoid a double-spend.

## Requirements

- **Node.js >= 20.**
- Network access to `https://three.ws` (or your own `THREE_WS_BASE`) and a Solana RPC.

### Environment variables

| Variable                | Required                  | Default                                 |
| ----------------------- | ------------------------- | --------------------------------------- |
| `THREE_WS_BASE`         | no                        | `https://three.ws`                      |
| `THREE_WS_TIMEOUT_MS`   | no                        | `20000`                                 |
| `THREE_WS_SESSION`      | for account-scoped reads  | —                                       |
| `SOLANA_RPC_URL`        | no                        | `https://api.mainnet-beta.solana.com`   |
| `SOLANA_SECRET_KEY`     | for `send_transfer`       | —                                       |
| `MAX_SOL_PER_TX`        | no                        | `0.5`                                   |
| `RECIPIENT_ALLOWLIST`   | no                        | — (any valid pubkey allowed)            |
| `REQUIRE_CONFIRM`       | no                        | `true`                                  |

`THREE_WS_SESSION` is the value of the `__Host-sid` cookie from a signed-in three.ws browser session. Treat it — and `SOLANA_SECRET_KEY` — like a password.

## Links

- Homepage: https://three.ws
- Changelog: https://three.ws/changelog
- Issues: https://github.com/nirholas/three.ws/issues
- License: Apache-2.0. See [LICENSE](./LICENSE).

---

<p align="center">
  <sub>
    Part of the <a href="https://three.ws">three.ws</a> SDK suite — 3D AI agents, on-chain identity, and agent payments.<br/>
    <a href="https://three.ws">Website</a> · <a href="https://three.ws/changelog">Changelog</a> · <a href="https://github.com/nirholas/three.ws">GitHub</a>
  </sub>
</p>

<!-- three.ws:growth -->
## Support the project

If crypto-portfolio-mcp saves you time, **[star it on GitHub](https://github.com/nirholas/crypto-portfolio-mcp)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=crypto-portfolio-mcp%3A%20An%20AI%20agent's%20own%20trading%20state%20over%20MCP%3A%20portfolio%20summary%2C%20value%20history%2C%20live%20wallet%20balances&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fcrypto-portfolio-mcp) · [Share on Bluesky](https://bsky.app/intent/compose?text=crypto-portfolio-mcp%3A%20An%20AI%20agent's%20own%20trading%20state%20over%20MCP%3A%20portfolio%20summary%2C%20value%20history%2C%20live%20wallet%20balances%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fcrypto-portfolio-mcp) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fcrypto-portfolio-mcp) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fcrypto-portfolio-mcp&t=crypto-portfolio-mcp%3A%20An%20AI%20agent's%20own%20trading%20state%20over%20MCP%3A%20portfolio%20summary%2C%20value%20history%2C%20live%20wallet%20balances) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fcrypto-portfolio-mcp&title=crypto-portfolio-mcp%3A%20An%20AI%20agent's%20own%20trading%20state%20over%20MCP%3A%20portfolio%20summary%2C%20value%20history%2C%20live%20wallet%20balances)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/crypto-portfolio-mcp` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/crypto-portfolio-mcp/issues) or [start a discussion](https://github.com/nirholas/crypto-portfolio-mcp/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/crypto-portfolio-mcp)](https://github.com/nirholas/crypto-portfolio-mcp/graphs/contributors)

## Star history

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/crypto-portfolio-mcp&type=Date)](https://www.star-history.com/#nirholas/crypto-portfolio-mcp&Date)

<!-- /three.ws:growth -->

<!-- three.ws:ecosystem -->
## Part of three.ws

[three.ws](https://three.ws) is a platform for 3D AI agents with Solana wallets: avatars, a skill marketplace, x402 payments and more than seventy MCP servers. `@three-ws/portfolio-mcp` is one package from it.

- Documentation site: https://nirholas.github.io/crypto-portfolio-mcp/
- npm: https://www.npmjs.com/package/@three-ws/portfolio-mcp
- Canonical source: https://github.com/nirholas/three.ws (this repository is a generated mirror of `packages/portfolio-mcp`)
- Agent-readable summary: [llms.txt](./llms.txt) and [AGENTS.md](./AGENTS.md)
- Issues and ideas: https://github.com/nirholas/crypto-portfolio-mcp/issues
<!-- /three.ws:ecosystem -->
