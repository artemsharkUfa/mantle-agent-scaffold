# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Development

```bash
npm run build          # tsc → dist/
npm run dev            # tsc --watch
npm run typecheck      # tsc --noEmit (no emit, just check)
npm run start          # node dist/index.js (MCP stdio server)
```

**Skills submodule** (required before first build):
```bash
npm run skills:init    # git submodule init + update
npm run skills:sync    # pull latest skills content
```

## Testing

```bash
npm test                    # vitest run (runs build first via pretest)
npm run test:e2e            # vitest with e2e config (5min timeout per test)
npm run test:e2e:required   # E2E with E2E_REQUIRE_LIVE=true
```

- Unit tests: `tests/**/*.test.ts` — vitest, node environment
- E2E tests: `e2e/**/*.test.ts` — LLM-driven, require env vars (`E2E_LLM_PROVIDER`, `E2E_LLM_API_KEY`, `E2E_LLM_MODEL`); see `.env.example`
- Run a single test: `npx vitest run tests/chain.test.ts`
- Run a single E2E test: `npx vitest run --config vitest.e2e.config.ts e2e/defi-read.test.ts`

## Architecture

ES Module TypeScript project (`"type": "module"`, target ES2022, `NodeNext` module resolution). Two entry points:

### MCP Server (`src/index.ts` → `src/server.ts`)
- stdio-only MCP server using `@modelcontextprotocol/sdk`
- Capabilities: tools, resources, prompts
- `ListTools` / `CallTool` handlers dispatch to tool modules via `allTools` record
- All errors normalized through `toErrorPayload()` → `{ error, code, message, suggestion, details }`

### CLI (`cli/index.ts`)
- Commander-based CLI (`mantle-cli`) mirroring MCP tool functionality
- 7 command groups matching tool categories
- Global options: `--network`, `--json`, `--no-color`, `--rpc-url`

### Tool Registration Pattern
Each `src/tools/<category>.ts` exports a `Record<string, Tool>` where `Tool` has `name`, `description`, `inputSchema`, and `handler`. All merged in `src/tools/index.ts` → `allTools`.

**Tool categories** (7 files, 19 tools):
- `chain.ts` — chain info, live status
- `account.ts` — MNT balance, ERC-20 balances, allowances
- `token.ts` — resolve, metadata, prices (DexScreener → DefiLlama fallback)
- `registry.ts` — address resolution against trusted contract registry
- `defi-read.ts` — swap quotes (Agni/Merchant Moe), pool data, TVL, Aave v3 lending markets (largest file ~2300 lines)
- `indexer.ts` — GraphQL subgraph + read-only SQL queries (endpoint policy enforced)
- `diagnostics.ts` — RPC health, capabilities listing

### Supporting Layers
- `src/config/` — static chain configs (mainnet 5000, sepolia 5003), token addresses, protocol contract addresses
- `src/lib/clients.ts` — viem `PublicClient` cached per network; RPC URL from env or config
- `src/lib/endpoint-policy.ts` — URL validation (blocks private IPs, non-HTTPS, SQL mutations)
- `src/lib/token-registry.ts` — symbol/address → resolved token with quick-ref fallback
- `src/lib/registry.ts` — trusted contract registry (address → label, category, status)
- `src/resources.ts` — 7 MCP resources (`mantle://chain/*`, `mantle://registry/*`, `mantle://docs/*`)
- `src/prompts.ts` — 3 MCP prompts (portfolioAudit, mantleBasics, gasConfiguration)

## Critical Domain Rules

- **Gas token is MNT, not ETH** — all gas estimates and native balances are in MNT
- **Never hold private keys** — transaction tools return unsigned payloads only
- **Always resolve before use** — `mantle_resolveAddress` / `mantle_resolveToken` before any tx-building tool
- **Read-only SQL** — `ensureReadOnlySql()` blocks INSERT/UPDATE/DELETE/DROP/ALTER/CREATE/TRUNCATE
- **Endpoint policy** — `ensureEndpointAllowed()` blocks non-HTTPS (except localhost when `MANTLE_ALLOW_HTTP_LOCAL_ENDPOINTS=true`), private IPs, metadata endpoints

## Environment Variables

| Variable | Purpose | Default |
|----------|---------|---------|
| `MANTLE_MCP_TRANSPORT` | Transport mode (only `stdio`) | — |
| `MANTLE_RPC_URL` | Mainnet RPC override | `https://rpc.mantle.xyz` |
| `MANTLE_SEPOLIA_RPC_URL` | Sepolia RPC override | `https://rpc.sepolia.mantle.xyz` |
| `MANTLE_ALLOW_HTTP_LOCAL_ENDPOINTS` | Allow http:// for localhost | `false` |
| `MANTLE_ALLOWED_ENDPOINT_DOMAINS` | Comma-separated domain allowlist for indexer | — |

## Knowledge Wiki (`knowledge/`)

Claude обновляет `knowledge/` только если причина/решение не очевидны из кода:

- **Баг с нетривиальным root cause** -> `troubleshooting.md`
- **Недокументированное поведение chain/RPC** -> `chain-quirks.md`

## CI

GitHub Actions (`.github/workflows/ci.yml`): checkout → Node 20 → skills:init → install → typecheck → test → docs:build
