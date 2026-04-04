# Conduit Protocol Specification

> **Status**: Draft v0.1
> **Date**: 2026-04-04
> **Token**: $CDT (ERC-20)

## 1. Overview

Conduit is a decentralized protocol for accessing RPC and LLM inference endpoints. It replaces the session-based, per-request metering model (used by protocols like Pocket Network) with a **lease-based, direct-access** model that eliminates gateway proxies, reduces on-chain overhead, and enables anyone with excess capacity to monetize it.

### Core Differentiators

| Dimension | Pocket Network (PATH) | Conduit |
|-----------|----------------------|---------|
| Access model | Sessions (block-height-based, auto-rollover) | Leases (time-based, DHCP-style renewal) |
| Pricing | Per-relay metering | Time-based access fee |
| Request path | Consumer → Gateway → Relay Miner → Node | Consumer → Provider (direct) |
| Auth | Per-relay cryptographic signing | JWT token (oracle-issued, time-scoped) |
| QoS | In-process reputation scoring in gateway | Oracle-driven, on-chain accountability |
| Provider onboarding | Run relay miner + stake + configure PATH | Call `registerEndpoint()` + stake |
| Data storage | On-chain sessions + in-memory state | On-chain registry + DA layer for bulk data |
| Gas overhead | Per-session on-chain activity | One-time registration + batched lease settlements |

### Design Principles

1. **Direct access**: No proxy or gateway in the request path. Consumers connect to providers directly.
2. **Lease over sessions**: Pay for time-based access, not per-request metering. Eliminates relay counting, sequence numbering, and per-request signature overhead.
3. **Minimal on-chain footprint**: Only store what needs trustless verification on-chain. Everything else goes to DA.
4. **Permissionless provider onboarding**: Anyone with excess RPC or inference capacity can register an endpoint and start earning $CDT.
5. **Oracle-driven operations**: JWT issuance, QoS monitoring, and usage aggregation handled by oracles — not embedded in a centralized gateway.
6. **Production-grade reliability**: QoS oracle with health checks, latency monitoring, and provider slashing — baked into the protocol, not optional.

## 2. Architecture

### 2.1 System Overview

```
                                    ┌──────────────────┐
                                    │   DA Layer        │
                                    │ (EigenDA/Celestia)│
                                    │                   │
                                    │ - JWT payloads    │
                                    │ - QoS history     │
                                    │ - Usage logs      │
                                    └────────┬──────────┘
                                             │ read/write
                    ┌────────────────────────┼────────────────────────┐
                    │                        │                        │
             ┌──────┴───────┐       ┌────────┴────────┐      ┌───────┴──────┐
             │  JWT Oracle   │       │  QoS Oracle     │      │ Settlement   │
             │              │       │                 │      │ Oracle       │
             │ - Issue JWTs │       │ - Health checks │      │              │
             │ - Refresh    │       │ - Latency probe │      │ - Batch      │
             │ - Revoke     │       │ - Score publish │      │   usage      │
             └──────┬───────┘       └────────┬────────┘      │ - Trigger    │
                    │                        │               │   payments   │
                    │                        │               └───────┬──────┘
                    ▼                        ▼                       ▼
        ┌───────────────────────────────────────────────────────────────┐
        │                        L2 Smart Contracts                     │
        │                                                               │
        │  ┌─────────────────┐  ┌──────────────┐  ┌─────────────────┐  │
        │  │ ProviderRegistry │  │ LeaseManager │  │   CDT Token     │  │
        │  │                 │  │              │  │   (ERC-20)      │  │
        │  │ - endpoints[]   │  │ - leases[]   │  │                 │  │
        │  │ - stakes        │  │ - renewals   │  │ - stake/slash   │  │
        │  │ - services      │  │ - pricing    │  │ - lease payment │  │
        │  └─────────────────┘  └──────────────┘  └─────────────────┘  │
        └───────────────────────────────────────────────────────────────┘
                    ▲                                        ▲
                    │ register                               │ acquireLease
                    │                                        │
             ┌──────┴───────┐                       ┌────────┴────────┐
             │   Provider    │◄──── direct HTTP ────►│   Consumer      │
             │              │      (with JWT)       │   Sidecar       │
             │ - RPC node   │                       │                 │
             │ - LLM host   │                       │ - Lease mgmt   │
             │ - JWT verify │                       │ - Token refresh │
             └──────────────┘                       │ - Failover     │
                                                    └─────────────────┘
```

### 2.2 Component Summary

| Component | Type | Purpose |
|-----------|------|---------|
| **ProviderRegistry** | Smart Contract (L2) | On-chain directory of provider endpoints, staking, service metadata |
| **LeaseManager** | Smart Contract (L2) | Lease lifecycle: acquire, renew, expire. Tracks consumer-provider relationships |
| **CDT Token** | Smart Contract (L2, bridgeable to L1) | ERC-20 for lease payments, provider staking, and protocol governance |
| **JWT Oracle** | Off-chain Oracle | Issues and refreshes JWT tokens for active leases; posts to DA layer |
| **QoS Oracle** | Off-chain Oracle | Periodic health checks, latency probes, scoring; publishes to DA + on-chain suspensions |
| **Settlement Oracle** | Off-chain Oracle | Aggregates usage data from DA, triggers batched payment settlements on-chain |
| **Consumer Sidecar** | Client-side daemon | Local agent that manages leases, refreshes tokens, selects endpoints, handles failover |
| **DA Layer** | External infrastructure | Stores JWT payloads, QoS history, and detailed usage logs off-chain |

## 3. Smart Contract Architecture

### 3.1 ProviderRegistry

The registry is the entry point for anyone wanting to offer RPC or LLM inference capacity.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IProviderRegistry {
    struct Endpoint {
        string url;           // Static, immutable endpoint URL
        bytes32 serviceId;    // e.g., keccak256("evm-mainnet"), keccak256("llm-hermes-70b")
        uint256 stakedAmount; // CDT staked as collateral
        uint8 tier;           // 0=unranked, 1=verified, 2=premium (set by QoS oracle)
        bool active;          // Can be suspended by QoS oracle
        uint256 registeredAt;
    }

    /// @notice Register a new endpoint. Requires minimum stake.
    /// @param serviceId The service identifier
    /// @param url The static URL for this endpoint
    function registerEndpoint(bytes32 serviceId, string calldata url) external;

    /// @notice Remove an endpoint and begin stake unlock period
    function deregisterEndpoint(bytes32 serviceId) external;

    /// @notice Get all active endpoints for a service (view, no gas)
    function getEndpoints(bytes32 serviceId) external view returns (Endpoint[] memory);

    /// @notice Increase stake on an existing endpoint
    function addStake(bytes32 serviceId) external payable;

    /// @notice Withdraw stake after unlock period
    function withdrawStake(bytes32 serviceId) external;

    /// @notice Called by QoS oracle to suspend/unsuspend or update tier
    function updateEndpointStatus(
        address provider,
        bytes32 serviceId,
        bool active,
        uint8 tier
    ) external; // onlyOracle

    /// @notice Called by QoS oracle to slash a provider's stake
    function slashProvider(
        address provider,
        bytes32 serviceId,
        uint256 amount,
        bytes32 reason
    ) external; // onlyOracle
}
```

**Key design decisions:**
- URL is immutable once registered — the provider sets up their endpoint, it never changes
- Minimum stake required (prevents spam registrations and provides slashing collateral)
- Tier system managed by QoS oracle, not self-reported
- Stake unlock period (e.g., 7 days) to prevent slash-and-run
- `getEndpoints()` is a view function — consumers read it for free

### 3.2 LeaseManager

Leases replace Pocket's sessions. A lease grants a consumer access to all endpoints for a service for a duration.

```solidity
interface ILeaseManager {
    struct Lease {
        address consumer;
        bytes32 serviceId;
        uint256 startTime;
        uint256 endTime;
        uint256 paidAmount;      // CDT paid for this lease period
        bytes32 jwtCommitment;   // Hash of current JWT (posted to DA by oracle)
        bool autoRenew;
        bool active;
    }

    /// @notice Acquire a lease for a service. Pays CDT for duration.
    /// @param serviceId The service to lease
    /// @param duration Lease duration in seconds
    /// @param autoRenew Whether to auto-renew when expiring
    function acquireLease(
        bytes32 serviceId,
        uint256 duration,
        bool autoRenew
    ) external;

    /// @notice Renew an existing lease. Can be called by consumer or oracle (for auto-renew).
    function renewLease(uint256 leaseId) external;

    /// @notice Cancel a lease. Prorated refund if time remains.
    function cancelLease(uint256 leaseId) external;

    /// @notice Called by JWT oracle to update the JWT commitment for a lease
    function updateJwtCommitment(
        uint256 leaseId,
        bytes32 newCommitment
    ) external; // onlyOracle

    /// @notice Get active lease details (view, no gas)
    function getLease(uint256 leaseId) external view returns (Lease memory);

    /// @notice Get the current JWT commitment for a lease (view, no gas)
    function getActiveJwtCommitment(uint256 leaseId) external view returns (bytes32);

    /// @notice Get all active leases for a consumer (view, no gas)
    function getConsumerLeases(address consumer) external view returns (Lease[] memory);
}
```

**Key design decisions:**
- Lease is time-based, not request-based — no relay counting
- DHCP-style auto-renewal: oracle triggers renewal if balance allows
- JWT commitment stored on-chain (hash only), full JWT on DA layer
- Prorated refund on cancellation — consumer-friendly
- Pricing can be flat-rate per service or dynamic based on provider count/demand

### 3.3 CDT Token

```solidity
interface ICDTToken {
    // Standard ERC-20 + additional protocol functions

    /// @notice Stake CDT as a provider
    function stakeForProvider(address provider, bytes32 serviceId) external;

    /// @notice Slash staked CDT (called by ProviderRegistry)
    function slash(address provider, uint256 amount) external; // onlyRegistry

    /// @notice Pay for a lease (called by LeaseManager)
    function payLease(address consumer, uint256 amount) external; // onlyLeaseManager

    /// @notice Distribute rewards to providers from lease payments
    function distributeRewards(
        bytes32 serviceId,
        address[] calldata providers,
        uint256[] calldata amounts
    ) external; // onlySettlementOracle
}
```

## 4. Oracle Architecture

### 4.1 JWT Oracle

**Purpose**: Issue, refresh, and revoke JWT tokens for active leases.

**Flow**:
```
1. LeaseManager emits LeaseAcquired(leaseId, consumer, serviceId, duration)
2. JWT Oracle listens for event
3. Oracle generates JWT:
   {
     "sub": consumerAddress,
     "sid": serviceId,
     "lid": leaseId,
     "iat": now,
     "exp": leaseEndTime,
     "endpoints": [...activeEndpointsForService],
     "iss": "conduit-jwt-oracle",
     "jti": uniqueTokenId
   }
4. Oracle signs JWT with its private key
5. Oracle posts full JWT to DA layer
6. Oracle calls LeaseManager.updateJwtCommitment(leaseId, keccak256(jwt))
7. Consumer sidecar fetches JWT from DA layer using commitment as pointer
```

**Refresh cycle**:
- JWT TTL: configurable per service (default 1 hour)
- Oracle refreshes at TTL/2 intervals
- On refresh: new JWT posted to DA, new commitment on-chain
- Batch commitment updates to minimize gas (e.g., every 30 min for all active leases)

**Revocation**:
- On lease cancellation: oracle adds JWT ID to revocation list on DA
- Providers check revocation list periodically (lightweight, off-chain)
- Emergency revocation: oracle can update commitment to null on-chain

### 4.2 QoS Oracle

**Purpose**: Monitor provider endpoint health, score performance, enforce SLAs.

Inspired by Pocket PATH's reputation system but implemented as an oracle rather than in-process.

**Health check cycle**:
```
Every N minutes (configurable per service):
1. For each registered endpoint in a service:
   a. Send probe request (eth_blockNumber for RPC, simple prompt for LLM)
   b. Measure: latency, correctness, availability
   c. Record observation to DA layer
2. Compute updated scores using exponential moving average
3. If score crosses tier threshold → update on-chain tier
4. If score drops below minimum → suspend endpoint on-chain
5. If provider recovers → unsuspend (probation period)
6. Publish full QoS report to DA layer
```

**Scoring model** (adapted from PATH's reputation system):

| Signal | Score Impact |
|--------|-------------|
| Successful response | +1 |
| Recovery from probation | +15 |
| Minor error (timeout, 5xx) | -3 |
| Major error (wrong chain ID, stale data) | -10 |
| Critical error (malformed response) | -25 |
| Fatal (endpoint unreachable) | -50 |
| Fast response (<100ms for RPC, <500ms for LLM first token) | +2 |
| Slow response (>1s RPC, >5s LLM) | -1 |
| Very slow (>3s RPC, >15s LLM) | -3 |

**Tier thresholds**:
- Tier 2 (Premium): Score >= 85
- Tier 1 (Verified): Score >= 60
- Tier 0 (Unranked): Score >= 30
- Suspended: Score < 30

**LLM-specific checks**:
- Model version verification (compare output metadata)
- Token throughput measurement (tokens/sec)
- Output coherence check (known prompt → expected output hash)
- Context window validation

### 4.3 Settlement Oracle

**Purpose**: Aggregate usage and trigger batched payments from lease fees to providers.

**Flow**:
```
1. Reads active leases from LeaseManager
2. Reads QoS data from DA layer (which providers were active/healthy during lease period)
3. Computes pro-rata distribution:
   - Lease payment distributed to providers proportional to:
     a. Time endpoint was active during lease
     b. QoS tier multiplier (premium earns more)
     c. Number of active providers (more providers = smaller individual share)
4. Calls CDT.distributeRewards() with batched payment data
5. Settlement interval: configurable (e.g., daily, weekly)
```

## 5. Consumer Sidecar

The sidecar is a lightweight daemon that runs alongside the consumer's application.

### 5.1 Responsibilities

```
Consumer Sidecar
├─ Wallet Management
│  ├─ Links consumer's wallet (private key or hardware wallet)
│  └─ Signs lease acquisition/renewal transactions
│
├─ Lease Lifecycle
│  ├─ Acquire lease via LeaseManager contract
│  ├─ Monitor lease expiry
│  ├─ Auto-renew before expiration (DHCP-style)
│  └─ Handle insufficient balance gracefully
│
├─ Token Management
│  ├─ Fetch current JWT from DA layer (using on-chain commitment as pointer)
│  ├─ Cache JWT locally
│  ├─ Refresh before TTL expiry
│  └─ Handle revocation notifications
│
├─ Endpoint Selection
│  ├─ Read QoS scores from DA layer
│  ├─ Rank endpoints by tier + latency
│  ├─ Local latency measurement (supplement oracle data)
│  ├─ Failover to next-best on error
│  └─ TLD diversity preference (avoid single-provider risk)
│
├─ Local Proxy
│  ├─ Expose localhost:PORT for consumer application
│  ├─ Attach JWT to outgoing requests
│  ├─ Route to best endpoint
│  └─ Transparent retry on failure
│
└─ Observability
   ├─ Local metrics (request count, latency, errors)
   └─ Optional: report consumer-side observations to DA (improves QoS data)
```

### 5.2 Consumer Request Flow

```
Application
    │
    │ HTTP POST localhost:8545 (RPC) or localhost:8080 (LLM)
    ▼
Consumer Sidecar
    │
    ├─ 1. Check JWT is valid and not expired
    │     └─ If expired → fetch new JWT from DA (using latest on-chain commitment)
    │
    ├─ 2. Select best endpoint from cached QoS rankings
    │     └─ If primary fails → failover to next tier
    │
    ├─ 3. Attach JWT as Authorization: Bearer <token>
    │
    ├─ 4. Forward request directly to provider endpoint
    │
    └─ 5. Return response to application
         └─ Record local latency observation
```

## 6. Provider Flow

### 6.1 Onboarding

```
1. Provider has excess capacity (RPC node, LLM inference server, etc.)
2. Provider acquires CDT tokens
3. Provider calls ProviderRegistry.registerEndpoint(serviceId, url)
   └─ Transfers minimum stake to contract
4. Provider configures their endpoint to validate Conduit JWTs
   └─ Fetch oracle public key from well-known contract/DA location
5. QoS oracle begins probing the endpoint
6. Once score reaches Tier 1 threshold → endpoint becomes visible to consumers
```

### 6.2 JWT Validation (Provider-Side)

```
On each incoming request:
1. Extract JWT from Authorization header
2. Verify signature against Conduit JWT Oracle public key
3. Check exp claim (not expired)
4. Check sid claim matches this provider's registered serviceId
5. (Optional) Check jti against revocation list from DA layer
6. If valid → serve request
7. If invalid → return 401
```

This is intentionally lightweight — no per-request on-chain calls, no relay counting, no sequence numbers.

## 7. Network Optimization (Minimizing Hops)

Since Conduit eliminates the gateway proxy, the remaining latency factor is raw network distance between consumer and provider. We address this at multiple levels.

### 7.1 Provider-Side: Cloudflare Tunnel (Recommended)

Providers are encouraged (not required) to expose endpoints via Cloudflare Tunnel:

```
Provider's local endpoint (localhost:8545)
  → cloudflared tunnel (local daemon)
  → Cloudflare PoP (nearest to provider)
  → Cloudflare private backbone
  → Cloudflare PoP (nearest to consumer)
  → Consumer sidecar
```

**Benefits**:
- **Backbone routing**: Requests traverse Cloudflare's private network instead of public internet (fewer hops, lower latency)
- **DDoS protection**: Free, always-on
- **No port exposure**: Provider doesn't need to open ports or expose their IP
- **TLS built-in**: End-to-end encryption without provider-side cert management
- **Anycast**: Consumer hits the nearest Cloudflare PoP automatically

**Cost**: Cloudflare Tunnels are free. Argo Smart Routing (optimized backbone) is ~$0.10/GB — negligible for RPC payloads.

The URL registered on-chain is the tunnel URL (static, immutable). The provider can change their underlying infrastructure without changing the registered URL.

### 7.2 Consumer-Side: Geographic Endpoint Selection

The consumer sidecar performs local latency measurements to supplement QoS oracle data:

```
Sidecar Endpoint Selection
├─ 1. Fetch QoS scores from DA layer (oracle-measured, multi-region)
├─ 2. Local latency probe to top-N endpoints (real RTT from consumer's location)
├─ 3. Composite ranking: QoS tier × (1 / local_latency)
├─ 4. Select best endpoint
└─ 5. Re-probe periodically (every 5 min) to detect changes
```

This means a consumer in Tokyo will naturally prefer providers in Asia, while a consumer in London will prefer European providers — without any protocol-level geographic configuration.

### 7.3 Alternative Network Strategies

| Strategy | Use Case | Trade-off |
|----------|----------|-----------|
| **Cloudflare Tunnel** | Default recommendation | Depends on Cloudflare (centralization risk) |
| **Bare direct (no tunnel)** | Providers who want full control | No DDoS protection, public IP exposed |
| **QUIC/HTTP3** | Latency-sensitive consumers | Both sides need QUIC support |
| **WireGuard tunnel** | Privacy-focused providers | Consumer needs WireGuard client |

The protocol is transport-agnostic — the registered URL just needs to be reachable via HTTP(S). The tunnel strategy is a provider-side choice.

### 7.4 Request Path Comparison

```
Pocket Network (PATH):
Consumer → DNS → Internet → PATH Gateway → Internet → Relay Miner → Internet → Node
Hops: 6+ network traversals, 2 intermediary processes

Conduit (with Cloudflare Tunnel):
Consumer → DNS → CF PoP (nearby) → CF Backbone → CF PoP (near provider) → Provider
Hops: 1 public internet hop (DNS), rest on private backbone

Conduit (bare direct):
Consumer → DNS → Internet → Provider
Hops: 1 public internet hop, no intermediaries
```

## 8. Token Economics ($CDT)

### 8.1 Supply and Distribution

| Allocation | % | Vesting |
|-----------|---|---------|
| Protocol Treasury | 25% | 4-year linear |
| Team | 15% | 1-year cliff, 3-year linear |
| Early Contributors | 10% | 6-month cliff, 2-year linear |
| Liquidity / Ecosystem | 20% | As needed |
| Provider Rewards Bootstrap | 15% | Emitted over 5 years |
| Community / Grants | 15% | Governed by DAO |

### 8.2 Token Utility

1. **Lease payments**: Consumers pay CDT for time-based access to services
2. **Provider staking**: Providers stake CDT as collateral (slashable for bad behavior)
3. **Governance**: CDT holders vote on protocol parameters (minimum stake, lease pricing, oracle configs)
4. **Oracle incentives**: Oracle operators earn CDT for running JWT, QoS, and settlement oracles

### 8.3 Fee Flow

```
Consumer pays CDT for lease
    │
    ├─ 90% → Provider reward pool (distributed by Settlement Oracle)
    │         ├─ Weighted by: uptime × tier multiplier × active time
    │         └─ Higher QoS tier = higher share
    │
    ├─ 5% → Oracle operators (JWT + QoS + Settlement)
    │
    └─ 5% → Protocol treasury (funds development + grants)
```

### 8.4 Staking and Slashing

**Provider staking**:
- Minimum stake: configurable per service (higher for LLM, lower for RPC)
- Stake-weighted visibility: higher stake = shown first in endpoint list (tie-breaker)

**Slashing conditions** (enforced by QoS oracle):
- Endpoint unreachable for > 1 hour: 1% slash
- Serving incorrect data (wrong chain, hallucinated responses): 5% slash
- Repeated SLA violations within 7 days: 10% slash
- Malicious behavior (serving harmful content, data exfiltration attempt): 50% slash

**Slash distribution**:
- 50% burned (deflationary pressure)
- 50% to affected consumers (compensation)

## 9. Supported Service Types

### 9.1 RPC Services

| Service ID | Description | QoS Checks |
|-----------|-------------|------------|
| `evm-mainnet` | Ethereum mainnet RPC | Block height, chain ID, eth_call correctness |
| `evm-arbitrum` | Arbitrum One RPC | Same + L2-specific methods |
| `evm-base` | Base RPC | Same |
| `solana-mainnet` | Solana mainnet RPC | Slot height, getHealth, transaction confirmation |
| `cosmos-hub` | Cosmos Hub RPC | Block height, /status endpoint |

### 9.2 LLM Inference Services

| Service ID | Description | QoS Checks |
|-----------|-------------|------------|
| `llm-hermes-70b` | Nous Hermes 3 70B | Response coherence, tokens/sec, model version |
| `llm-hermes-405b` | Nous Hermes 3 405B | Same |
| `llm-llama-70b` | Llama 3.1 70B | Same |
| `llm-mixtral-8x22b` | Mixtral 8x22B | Same |

New services can be added via governance proposal.

## 10. Security Model

### 10.1 Trust Assumptions

| Component | Trust Level | Accountability |
|-----------|-------------|----------------|
| Smart Contracts | Trustless | Audited, immutable (upgradeable via governance) |
| JWT Oracle | Semi-trusted | Must be decentralized (multi-sig or threshold) |
| QoS Oracle | Semi-trusted | Results verifiable against DA layer history |
| Providers | Untrusted | Staked + slashable + QoS monitored |
| Consumers | Untrusted | Pre-paid via lease |
| DA Layer | Trust DA consensus | Using established DA (EigenDA/Celestia) |

### 10.2 Attack Vectors and Mitigations

| Attack | Mitigation |
|--------|-----------|
| Provider serves stale/wrong data | QoS oracle detects via correctness checks; slashing |
| Consumer shares JWT with unauthorized users | JWT is scoped to consumer address; provider can optionally verify source IP |
| Oracle collusion | Threshold oracle design (e.g., 3-of-5 signers for JWT issuance) |
| Sybil attack on provider registry | Minimum stake requirement makes Sybil expensive |
| Eclipse attack on QoS oracle | Multiple independent probe locations; DA layer provides verifiable history |
| JWT replay after lease cancellation | Revocation list on DA; short JWT TTL limits window |
| Provider front-running (reading consumer requests) | Out of scope for v1; future: TEE-based inference |

### 10.3 Access Control

```
ProviderRegistry:
  registerEndpoint    → any address (with sufficient stake)
  deregisterEndpoint  → only endpoint owner
  updateEndpointStatus → only QoS oracle
  slashProvider       → only QoS oracle

LeaseManager:
  acquireLease        → any address (with sufficient CDT)
  renewLease          → lease owner OR JWT oracle (for auto-renew)
  cancelLease         → lease owner only
  updateJwtCommitment → only JWT oracle

CDT Token:
  transfer            → standard ERC-20
  slash               → only ProviderRegistry
  distributeRewards   → only Settlement Oracle
```

## 11. Comparison with Pocket Network

### 11.1 Architectural Comparison

| Aspect | Pocket (Shannon + PATH) | Conduit |
|--------|------------------------|---------|
| **Request hops** | Consumer → PATH Gateway → Relay Miner → Node | Consumer → Provider (1 hop) |
| **Auth per request** | Relay signed with gateway key, validated by relay miner | JWT in header, validated locally by provider |
| **Session/lease mgmt** | On-chain sessions tied to block height, auto-rollover | Time-based leases with DHCP-style renewal |
| **QoS** | In-process in PATH gateway (reputation 0-100) | Oracle-driven, on-chain accountability |
| **Provider onboarding** | Run relay miner + full node + stake + configure PATH | Register endpoint + stake (one tx) |
| **Payment model** | Per-relay, settled on-chain per session | Per-lease (time-based), settled in batches |
| **Gateway** | Required (centralized component) | None (direct access) |
| **Complexity** | High (sessions, relay counting, signatures, rollover) | Low (lease + JWT + direct call) |

### 11.2 Where Conduit Wins

1. **Latency**: Eliminating the gateway proxy removes at least one network hop
2. **Simplicity**: No relay counting, no session rollover, no per-request signatures
3. **Provider UX**: Register with one transaction vs. running relay miner infrastructure
4. **Gas efficiency**: Batched settlements vs. per-session on-chain activity
5. **Reliability**: Direct connections + oracle QoS vs. gateway-mediated with in-process scoring
6. **LLM support**: First-class support for inference endpoints, not just RPC

### 11.3 Where Pocket Has Advantages (and how we address them)

1. **Established network effects**: Pocket has existing providers and consumers
   - *Mitigation*: Bootstrap program with CDT incentives; target excess capacity providers first
2. **Per-request granularity**: Some consumers want pay-per-request
   - *Mitigation*: Offer short-duration leases (e.g., 1-hour minimum) as approximate equivalent
3. **Proven on-chain settlement**: Pocket's relay proofs are cryptographically verifiable
   - *Mitigation*: Our QoS oracle + DA layer provides verifiable history; threshold oracle design prevents manipulation

## 12. Roadmap

### Phase 1: Foundation (MVP)
- [ ] Deploy ProviderRegistry and LeaseManager contracts on testnet
- [ ] Implement CDT token with basic staking
- [ ] Build JWT Oracle (single operator, Chainlink Functions)
- [ ] Build QoS Oracle for EVM RPC (basic health checks)
- [ ] Consumer sidecar CLI (lease management + local proxy)
- [ ] Support: `evm-mainnet`, `evm-arbitrum`

### Phase 2: Decentralization
- [ ] Threshold JWT oracle (multi-signer)
- [ ] DA layer integration (EigenDA or Celestia)
- [ ] Settlement oracle with batched payments
- [ ] Provider dashboard (stake, earnings, QoS score)
- [ ] Consumer dashboard (leases, usage, spending)
- [ ] Support: Solana, Cosmos RPC

### Phase 3: LLM Inference
- [ ] LLM-specific QoS checks (coherence, throughput, model verification)
- [ ] LLM service types in registry
- [ ] Streaming support (SSE/WebSocket for LLM inference)
- [ ] Support: Hermes, Llama, Mixtral inference endpoints

### Phase 4: Production
- [ ] Mainnet deployment
- [ ] CDT token launch
- [ ] Governance DAO
- [ ] Cross-chain bridge for CDT
- [ ] TEE-based inference (privacy for consumer prompts)
- [ ] SDK for provider integration (drop-in JWT validation middleware)

## Appendix A: Service ID Registry

Service IDs are `bytes32` values computed as `keccak256(serviceString)`:

```
evm-mainnet     → 0x...
evm-arbitrum    → 0x...
evm-base        → 0x...
evm-optimism    → 0x...
solana-mainnet  → 0x...
cosmos-hub      → 0x...
llm-hermes-70b  → 0x...
llm-hermes-405b → 0x...
llm-llama-70b   → 0x...
```

## Appendix B: JWT Claim Schema

```json
{
  "sub": "0xConsumerAddress",
  "sid": "evm-mainnet",
  "lid": 42,
  "iat": 1712200000,
  "exp": 1712203600,
  "endpoints": [
    "https://provider1.example.com/rpc",
    "https://provider2.example.com/rpc"
  ],
  "tier_min": 1,
  "iss": "conduit-jwt-oracle",
  "jti": "unique-token-id-here"
}
```

## Appendix C: Gas Cost Estimates

| Operation | Estimated Gas | Estimated Cost (L2) |
|-----------|--------------|---------------------|
| registerEndpoint | ~150,000 | ~$0.02 |
| acquireLease | ~120,000 | ~$0.015 |
| renewLease | ~80,000 | ~$0.01 |
| updateJwtCommitment (batched, per lease) | ~30,000 | ~$0.004 |
| updateEndpointStatus | ~50,000 | ~$0.006 |
| distributeRewards (batched, 100 providers) | ~500,000 | ~$0.06 |

All read operations (getEndpoints, getLease, etc.) are free (view functions).
