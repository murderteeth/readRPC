# readRPC

A fast, minimal, read-only proxy for Ethereum JSON-RPC requests.

## Local Development

```bash
bun i
cp .dev.vars.example .dev.vars  # Add your RPC URLs
bun dev                          # Runs on Cloudflare's edge (requires auth)
```

Note: Local workerd requires glibc 2.34+. The default `bun dev` uses `--remote` to run on Cloudflare's edge instead.

## Deployment

### 1. Authentication

Create an API token at https://dash.cloudflare.com/profile/api-tokens with the **"Edit Cloudflare Workers"** template.

### 2. Deploy

```bash
CLOUDFLARE_API_TOKEN=your_token bun run deploy
```

### 3. Set RPC URLs

Push environment variables updates via CLI during deploy:

```bash
CLOUDFLARE_API_TOKEN=your_token bunx wrangler deploy --var RPC_URI_FOR_1:https://eth.llamarpc.com
```

Or set them in the Cloudflare dashboard: **Worker → Settings → Variables and Secrets → Environment Variables**

For sensitive envars, use secrets instead:

```bash
CLOUDFLARE_API_TOKEN=your_token bunx wrangler secret put SHH_SECRET_SHH
```

### 4. Cleanup

Destroy personal tokens after use.

## Configuration

| Variable | Description | Default |
| --- | --- | --- |
| `RPC_URI_FOR_[chainId]` | Upstream RPC URL for chain | - |
| `LATEST_TTL` | Cache TTL for `latest` block queries (seconds) | `3` |
| `HISTORICAL_TTL` | Cache TTL for numeric block queries (seconds) | `30` |

## Endpoint

```
POST /chain/[chainId]
```

## Supported Methods

| Method | Description |
| --- | --- |
| `eth_blockNumber` | Returns the current block number |
| `eth_chainId` | Returns the chain ID |
| `eth_gasPrice` | Returns the current gas price |
| `eth_call` | Executes a call without creating a transaction |
| `eth_getBalance` | Returns the balance of an address |
| `eth_getCode` | Returns the code at an address |
| `eth_getStorageAt` | Returns storage at a position |
| `eth_getBlockByNumber` | Returns block by number |
| `eth_getBlockByHash` | Returns block by hash |
| `eth_getTransactionByHash` | Returns transaction by hash |
| `eth_getTransactionReceipt` | Returns transaction receipt |
| `eth_getTransactionCount` | Returns transaction count (nonce) |
| `eth_getLogs` | Returns logs matching filter |
| `eth_estimateGas` | Estimates gas for a transaction |

## Example Request

```bash
curl -X POST https://your-worker.workers.dev/chain/1 \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "eth_blockNumber",
    "params": [],
    "id": 1
  }'
```

## Example Response

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": "0x134a5b2"
}
```

## Caching

Uses Cloudflare's Cache API to cache POST responses at the edge:

- Queries with `latest`, `pending`, or absent block parameters are cached for `LATEST_TTL` seconds (default: 3)
- Queries with numeric block parameters are cached for `HISTORICAL_TTL` seconds (default: 30)
- Batch requests cache each item individually for maximum cache efficiency

Cache keys are generated from the chain ID, method name, and a SHA-256 hash of canonicalized parameters.

## Rate Limiting

Configure rate limiting via Cloudflare's WAF in the dashboard:

**Setup:** Workers → your-worker → Settings → Security → Rate Limiting

## Usage with Libraries

```typescript
// viem
const client = createPublicClient({
  transport: http('https://your-worker.workers.dev/chain/1')
})

// ethers
const provider = new JsonRpcProvider('https://your-worker.workers.dev/chain/1')
```
