# Conduit Protocol

> Decentralized, lease-based access to RPC and LLM inference endpoints.

**$CDT** — The Conduit Token

## What is Conduit?

Conduit replaces session-based decentralized RPC access (Pocket Network, etc.) with a lease-based model that enables direct consumer-to-provider connections with no gateway proxy.

**For providers**: Register your endpoint with one transaction. Start earning $CDT from excess capacity.

**For consumers**: Acquire a time-based lease. Connect directly to providers. No sessions, no per-request fees, no middleman.

## How it works

```
1. Provider registers endpoint on-chain (one tx)
2. Consumer acquires a lease (pays $CDT for time-based access)
3. JWT Oracle issues auth token → stored on DA layer
4. Consumer connects DIRECTLY to provider with JWT
5. QoS Oracle monitors quality → slashes bad actors
6. Settlement Oracle batches payments to providers
```

## Key differences from Pocket Network

| | Pocket | Conduit |
|--|--------|---------|
| Access model | Per-request sessions | Time-based leases |
| Request path | Consumer → Gateway → Relay Miner → Node | Consumer → Provider |
| Provider setup | Relay miner + full node + PATH config | `registerEndpoint()` + stake |
| Auth overhead | Sign every relay | JWT in header |
| Gas cost | Per-session activity | Batched settlements |

## Project structure

```
conduit/
├── docs/
│   ├── SPEC.md              # Protocol specification
│   ├── PRD.md               # Product requirements
│   ├── RESEARCH.md          # Technical research & architecture analysis
│   └── LANDSCAPE.md         # Competitive landscape (17 projects, market data)
├── contracts/
│   └── src/                  # Solidity smart contracts
├── oracle/
│   ├── jwt-oracle/           # JWT issuance and refresh
│   └── qos-oracle/           # Quality of service monitoring
└── sidecar/                  # Consumer sidecar daemon
```

## Status

Early design phase. See [docs/SPEC.md](docs/SPEC.md) for the protocol specification and [docs/PRD.md](docs/PRD.md) for product requirements.

## License

TBD
