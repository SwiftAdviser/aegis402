# Aegis402 Shield Protocol

> Pay-per-request blockchain security API for AI agents and DeFi automation. Built on x402 + Web3Antivirus.

**Production API:** `https://aegis402.com/v1`

---

## What It Does

Aegis402 wraps the Web3Antivirus scanning engine into three focused HTTP endpoints with automatic pay-per-use billing via the [x402 protocol](https://docs.x402.org). No API key registration, no subscriptions. The first 100 checks per day are free; after that, clients pay small USDC amounts on Base or Solana and get an immediate response.

The service is designed to be called by autonomous agents immediately before a transaction is signed — providing a last-mile safety layer without requiring a human in the loop.

---

## Endpoints and Pricing

| Endpoint | Purpose | Price |
|----------|---------|-------|
| `POST /v1/simulate-tx` | Transaction simulation + risk detection | $0.05 USDC |
| `GET /v1/check-token/:address` | Honeypot detection + token risk scan | $0.01 USDC |
| `GET /v1/check-address/:address` | Address poisoning + reputation check | $0.005 USDC |
| `GET /v1/usage` | Check free tier usage (always free) | Free |
| `GET /health` | Health check (always free) | Free |

**Supported address formats:** EVM (`0x...`) and Solana (base58) on all endpoints.

---

## Quick Start

Install the x402 fetch client:

```bash
npm install @x402/fetch@2.2.0 @x402/evm@2.2.0
```

Make a paid request with an EVM agent wallet:

```typescript
import { x402Client, wrapFetchWithPayment } from '@x402/fetch';
import { ExactEvmScheme } from '@x402/evm/exact/client';

// yourAgenticEvmSigner = any EVM signer (viem, ethers, CDP SDK, etc.)
const client = new x402Client()
  .register('eip155:*', new ExactEvmScheme(yourAgenticEvmSigner));

const fetch402 = wrapFetchWithPayment(fetch, client);

// Check a token for honeypot
const res = await fetch402(
  'https://aegis402.com/v1/check-token/0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48?chain_id=1',
  { headers: { 'X-Client-Fingerprint': 'my-agent-v1' } }
);
console.log(await res.json());
```

For Solana agents:

```bash
npm install @x402/fetch@2.2.0 @x402/svm@2.2.0
```

```typescript
import { x402Client, wrapFetchWithPayment } from '@x402/fetch';
import { ExactSvmScheme } from '@x402/svm/exact/client';

const client = new x402Client()
  .register('solana:*', new ExactSvmScheme(yourAgenticSolanaSigner));

const fetch402 = wrapFetchWithPayment(fetch, client);
```

---

## Free Tier

100 checks/day per client fingerprint. No signup required.

```bash
# Check your remaining free calls
curl https://aegis402.com/v1/usage
```

```json
{
  "freeTier": {
    "enabled": true,
    "dailyLimit": 100,
    "usedToday": 2,
    "remainingChecks": 98,
    "nextResetAt": "2026-02-27T00:00:00.000Z",
    "resetTimezone": "UTC"
  },
  "_meta": {
    "requestId": "550e8400-e29b-41d4-a716-446655440000",
    "tier": "free",
    "latencyMs": 4
  }
}
```

Set `X-Client-Fingerprint: <stable-id>` to get predictable per-agent free-tier accounting. Without it, the service falls back to IP/User-Agent signals.

---

## Payment

Once the free tier is exhausted, the API returns `HTTP 402 Payment Required`. An x402-compatible client automatically pays the required USDC amount and retries the request — no user intervention needed.

**Payment networks:**
- Base Mainnet (EVM, chain ID 8453) — default
- Solana Mainnet — supported when configured

**Payment currency:** USDC
**No subscriptions. No API keys. Pay only for what you use.**

---

## API Reference

### POST /v1/simulate-tx

Simulate a transaction against current chain state and detect risks before signing.

**Price:** $0.05 USDC

**Request body:**

```json
{
  "from": "0x742d35Cc6634C0532925a3b844Bc454e4438f44e",
  "to": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "value": "1000000000000000000",
  "data": "0xa9059cbb000000000000000000000000",
  "chain_id": 1
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `from` | string | Yes | Sender address (0x-prefixed, 40 hex chars) |
| `to` | string | Yes | Recipient or contract address |
| `value` | string | Yes | Amount in wei (decimal or hex) |
| `data` | string | No | Calldata hex (defaults to `0x`) |
| `chain_id` | number | No | Chain to simulate on (default: 1) |

**Response:**

```json
{
  "isSafe": false,
  "riskLevel": "CRITICAL",
  "simulation": {
    "assets": [...]
  },
  "warnings": [
    "Recipient is a known phishing contract",
    "Unusual token approval detected"
  ],
  "_meta": {
    "requestId": "550e8400-e29b-41d4-a716-446655440000",
    "tier": "paid",
    "latencyMs": 312
  }
}
```

| Field | Type | Values |
|-------|------|--------|
| `isSafe` | boolean | `true` / `false` |
| `riskLevel` | string | `SAFE`, `LOW`, `MEDIUM`, `HIGH`, `CRITICAL` |
| `simulation` | object | Asset movement data from Web3Antivirus |
| `warnings` | string[] | Human-readable risk descriptions |

---

### GET /v1/check-token/:address

Scan a token for honeypot mechanics, rug-pull indicators, and other risks.

**Price:** $0.01 USDC

**Supported:** EVM tokens (`0x...`) and Solana tokens (base58)

```bash
# EVM token (USDC on Ethereum)
curl "https://aegis402.com/v1/check-token/0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48?chain_id=1"

# Solana token
curl "https://aegis402.com/v1/check-token/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"
```

**Query params:**

| Param | Type | Description |
|-------|------|-------------|
| `chain_id` | number | EVM chain ID (1=Ethereum, 8453=Base, etc.). Auto-detected for Solana. |

---

### GET /v1/check-address/:address

Check a wallet or contract address for poisoning attacks and reputation flags.

**Price:** $0.005 USDC

**Supported:** EVM addresses (`0x...`) and Solana addresses (base58)

```bash
# EVM address
curl "https://aegis402.com/v1/check-address/0x742d35Cc6634C0532925a3b844Bc454e4438f44e"

# Solana address
curl "https://aegis402.com/v1/check-address/9WzDXwBbmkg8ZTbNMqUxvQRAyrZzDsGYdLVL9zYtAWWM"
```

---

## Supported Chains

`chain_id` is the chain being **scanned** (not the payment rail).

| Chain | ID | simulate-tx | check-token | check-address |
|-------|----|-------------|-------------|---------------|
| Ethereum | 1 | Yes | Yes | Yes |
| Base | 8453 | Yes | Yes | Yes |
| Polygon | 137 | Yes | Yes | Yes |
| Arbitrum | 42161 | Yes | Yes | Yes |
| Optimism | 10 | Yes | Yes | Yes |
| BSC | 56 | Yes | Yes | Yes |
| Avalanche | 43114 | Yes | Yes | Yes |
| Solana | solana | No | Yes | Yes |

---

## Agent Integration Pattern

Recommended pre-transaction safety check for autonomous agents:

```typescript
// Run all checks in parallel before signing
const [addressCheck, simulation] = await Promise.all([
  fetch402(`https://aegis402.com/v1/check-address/${to}`),
  fetch402('https://aegis402.com/v1/simulate-tx', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ from, to, value, data, chain_id }),
  }),
]);

const addr = await addressCheck.json();
const sim = await simulation.json();

// Block if any check returns HIGH or CRITICAL risk
if (!sim.isSafe || ['HIGH', 'CRITICAL'].includes(sim.riskLevel)) {
  throw new Error(`Transaction blocked: ${sim.warnings.join(', ')}`);
}
```

For token swaps and approvals, also run `check-token` in parallel:

```typescript
const [addressCheck, tokenCheck, simulation] = await Promise.all([
  fetch402(`https://aegis402.com/v1/check-address/${to}`),
  fetch402(`https://aegis402.com/v1/check-token/${tokenAddress}?chain_id=${chain_id}`),
  fetch402('https://aegis402.com/v1/simulate-tx', { method: 'POST', ... }),
]);
```

---

## Error Handling

All responses include `_meta.requestId` for debugging.

| Status | Meaning | Action |
|--------|---------|--------|
| 200 | Success | Use response |
| 400 | Invalid parameters | Check `error` field for details |
| 402 | Free tier exhausted | x402 client auto-pays and retries |
| 500 | Service error | Retry once; include `requestId` in bug reports |
| 502/503/504 | Upstream error | Retry with backoff |

---

## Use Cases

**AI trading agents** — Screen every transaction before execution. Catch phishing contracts, honeypot tokens, and poisoned addresses at $0.005–$0.05 per check.

**DeFi automation protocols** — Validate user-submitted transactions server-side before forwarding to the chain.

**Wallet applications** — Add built-in scam detection without managing security infrastructure. Pay only when users are active.

**AI agent frameworks** (ELIZA, LangChain, etc.) — Drop-in security tool. [SKILL.md](https://github.com/SwiftAdviser/aegis402/blob/main/SKILL.md) is formatted for direct agent consumption.

**MEV bots and arbitrage agents** — Pre-simulate transactions to avoid traps and losing execution to sandwiching on malicious contracts.

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Runtime | Bun / Node.js (Express 5) |
| Language | TypeScript (strict) |
| Payment protocol | x402 v2 (`@x402/express`, `@x402/evm`, `@x402/svm`) |
| Payment facilitator | Coinbase CDP |
| Payment network | Base Mainnet (EVM), Solana Mainnet |
| Security data | Web3Antivirus API |
| Resilience | Circuit breaker (opossum) + axios-retry |
| Analytics | Supabase (Postgres) |

---

## Self-Hosting

```bash
git clone https://github.com/SwiftAdviser/aegis-402-shield-protocol
cd aegis-402-shield-protocol

bun install

cp .env.example .env
# Edit .env:
# WEB3_ANTIVIRUS_API_KEY=your_key
# X402_WALLET_ADDRESS=0xYourEVMWallet
# X402_NETWORK_ID=8453
# X402_FACILITATOR_URL=https://facilitator.example.com
# X402_SOLANA_ADDRESS=YourSolanaAddress (optional)

bun --watch src/index.ts   # development
bun run build && bun start  # production
```

For development without real payments:

```bash
DEV_MODE=true bun --watch src/index.ts
```

---

## Feedback

Agents and developers can submit structured feedback via the API (free, non-consuming):

```bash
curl -X POST "https://aegis402.com/v1/feedback" \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "issue",
    "summary": "check-token returned unexpected result",
    "endpoint": "/v1/check-token/0x...",
    "status_code": 200,
    "chain_id": "8453",
    "agent": { "name": "my-trading-bot", "version": "2.1.0" }
  }'
```

`kind` must be one of: `issue`, `feedback`, `expectation`.

---

## Links

- Production API: https://aegis402.com/v1
- x402 Protocol docs: https://docs.x402.org
- x402 ecosystem: https://www.x402.org/ecosystem

## Community

- X: https://x.com/aegis402
- Telegram channel: https://t.me/aegis402_channel
- Developer chat: https://t.me/aegis402_chat

---

Built for the agentic economy. Powered by x402 Protocol.
