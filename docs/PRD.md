# Conduit Protocol — Product Requirements Document

> **Status**: Draft v0.1
> **Date**: 2026-04-04
> **Authors**: TBD

## 1. Problem Statement

Decentralized RPC and AI inference access is broken. Current solutions like Pocket Network suffer from:

1. **High latency**: Requests route through centralized gateways (PATH) before reaching nodes — adding unnecessary network hops
2. **Session overhead**: Block-height-based sessions require constant rollover, relay counting, and per-request cryptographic signing
3. **Provider complexity**: Running a relay miner requires significant infrastructure beyond the actual node
4. **Reliability**: Enterprise users avoid decentralized RPC in production because QoS is not guaranteed at the protocol level
5. **No LLM support**: Existing decentralized RPC protocols were not designed for inference workloads (streaming, long-running requests, model-specific QoS)
6. **Underutilized capacity**: Many node operators and GPU owners have excess capacity but the barrier to monetize it is too high

## 2. Vision

**Conduit makes it trivially easy to share and consume decentralized RPC and LLM inference capacity.**

For providers: register your endpoint with one transaction and start earning.
For consumers: acquire a time-based lease and connect directly — no gateway, no session management, no per-request fees.

## 3. Target Users

### 3.1 Providers (Supply Side)

| Persona | Description | Motivation |
|---------|-------------|------------|
| **Hobbyist node operator** | Runs an Ethereum full node at home for validation/personal use | Has excess RPC capacity; wants passive income from idle resources |
| **Small infrastructure company** | Operates nodes for internal use across multiple chains | Monetize overcapacity without building a commercial RPC product |
| **GPU owner** | Has consumer or datacenter GPUs running LLM inference | Sell excess inference capacity when not in use |
| **Existing RPC provider** | Already runs commercial nodes | Additional revenue channel with minimal integration effort |

### 3.2 Consumers (Demand Side)

| Persona | Description | Need |
|---------|-------------|------|
| **DApp developer** | Building on Ethereum/L2s | Reliable, affordable RPC without vendor lock-in |
| **AI application builder** | Building apps that call LLM inference | Decentralized, censorship-resistant LLM access |
| **Protocol/DAO** | Needs infrastructure for protocol operations | Self-sovereign infrastructure access, no single point of failure |
| **Researcher** | Academic or independent researcher | Affordable access to LLM inference at scale |

## 4. User Stories

### Provider Stories

- **P1**: As a node operator, I want to register my endpoint with a single transaction so that I can start earning without setting up complex infrastructure
- **P2**: As a provider, I want to see my QoS score and earnings in a dashboard so that I can optimize my service
- **P3**: As a provider, I want to deregister and withdraw my stake when I no longer want to participate
- **P4**: As a provider, I want the authentication to be lightweight so that it doesn't add latency to my responses
- **P5**: As a GPU owner, I want to register an LLM inference endpoint alongside RPC endpoints using the same system

### Consumer Stories

- **C1**: As a developer, I want to acquire a lease for RPC access and immediately start making calls without managing sessions
- **C2**: As a consumer, I want my access to auto-renew like DHCP so there's no interruption in service
- **C3**: As a consumer, I want to connect directly to the provider with no proxy so I get the lowest possible latency
- **C4**: As a consumer, I want the sidecar to handle failover automatically if my primary endpoint goes down
- **C5**: As a consumer, I want to see which providers are most reliable before choosing a service tier
- **C6**: As an AI app builder, I want to acquire a lease for LLM inference and stream responses directly from the provider

## 5. Product Requirements

### 5.1 Smart Contracts (Must Have — Phase 1)

| ID | Requirement | Priority |
|----|-------------|----------|
| SC-1 | ProviderRegistry contract with registerEndpoint, deregisterEndpoint, getEndpoints | P0 |
| SC-2 | Minimum stake enforcement on registration | P0 |
| SC-3 | LeaseManager contract with acquireLease, renewLease, cancelLease | P0 |
| SC-4 | CDT ERC-20 token with staking and transfer functionality | P0 |
| SC-5 | Oracle role-based access control on state-mutating functions | P0 |
| SC-6 | Prorated refund on lease cancellation | P1 |
| SC-7 | Stake unlock period (cooldown before withdrawal) | P1 |
| SC-8 | Service ID governance (adding new service types) | P2 |

### 5.2 JWT Oracle (Must Have — Phase 1)

| ID | Requirement | Priority |
|----|-------------|----------|
| JO-1 | Listen for LeaseAcquired events and generate JWT | P0 |
| JO-2 | Sign JWT with oracle key pair | P0 |
| JO-3 | Post JWT to DA layer and update on-chain commitment | P0 |
| JO-4 | Refresh JWT before TTL expiry | P0 |
| JO-5 | Batch commitment updates to reduce gas | P1 |
| JO-6 | Revocation list management on DA layer | P1 |
| JO-7 | Threshold signing (multi-signer JWT issuance) | P2 |

### 5.3 QoS Oracle (Must Have — Phase 1)

| ID | Requirement | Priority |
|----|-------------|----------|
| QO-1 | Periodic health probes for all registered endpoints | P0 |
| QO-2 | Latency measurement and scoring | P0 |
| QO-3 | Correctness validation (block height for RPC) | P0 |
| QO-4 | Tier assignment based on score thresholds | P0 |
| QO-5 | Automatic suspension of endpoints below threshold | P0 |
| QO-6 | Publish QoS reports to DA layer | P1 |
| QO-7 | LLM-specific QoS checks (coherence, throughput) | P2 |
| QO-8 | Multi-region probing for geographic diversity | P2 |

### 5.4 Consumer Sidecar (Must Have — Phase 1)

| ID | Requirement | Priority |
|----|-------------|----------|
| CS-1 | Wallet linking and lease acquisition | P0 |
| CS-2 | JWT fetch and local caching | P0 |
| CS-3 | Local proxy exposing localhost endpoint | P0 |
| CS-4 | Automatic JWT refresh before expiry | P0 |
| CS-5 | Endpoint selection based on QoS data | P0 |
| CS-6 | Failover to next-best endpoint on error | P0 |
| CS-7 | Auto-renew lease before expiration (DHCP-style) | P1 |
| CS-8 | Local metrics and observability | P1 |
| CS-9 | Consumer-side latency reporting to DA | P2 |

### 5.5 Settlement (Phase 2)

| ID | Requirement | Priority |
|----|-------------|----------|
| SO-1 | Settlement oracle aggregating usage data from DA | P1 |
| SO-2 | Batched reward distribution to providers | P1 |
| SO-3 | Fee split: providers / oracles / treasury | P1 |
| SO-4 | Slashing execution based on QoS violations | P1 |

## 6. Non-Functional Requirements

| Category | Requirement |
|----------|-------------|
| **Latency** | Direct consumer-to-provider path must add < 5ms overhead vs direct connection (JWT validation only) |
| **Availability** | Sidecar must handle provider failover within 500ms |
| **Scalability** | Registry must support 10,000+ endpoints per service without gas issues on reads |
| **Gas efficiency** | All oracle writes must be batched; no per-request on-chain transactions |
| **Security** | JWT validation must be stateless on provider side (no on-chain calls per request) |
| **Portability** | Sidecar must run on Linux, macOS, and as a Docker container |

## 7. Success Metrics

### Phase 1 (Testnet)
- 50+ registered provider endpoints across 3+ service types
- 100+ active leases
- QoS oracle achieving 99%+ probe success rate
- Average consumer-to-provider latency within 10ms of direct connection

### Phase 2 (Mainnet Launch)
- 500+ registered endpoints
- 1,000+ active leases
- $CDT trading volume > $100K/day
- 99.9% uptime for Tier 2 (Premium) endpoints

### Phase 3 (Growth)
- 5,000+ endpoints including LLM inference
- Enterprise consumers using Conduit in production
- Provider earnings averaging > $500/month for Tier 2 providers

## 8. Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Oracle centralization | Single point of failure for JWT issuance | Medium | Threshold oracle design in Phase 2; multiple independent operators |
| Cold start (no providers) | No value for consumers | High | Bootstrap incentive program; seed network with team-operated endpoints |
| Cold start (no consumers) | No revenue for providers | High | Free trial leases on testnet; partner with DApp teams |
| JWT oracle key compromise | All JWTs become untrusted | Low | Key rotation mechanism; threshold signing; short JWT TTL limits blast radius |
| L2 downtime | Lease acquisition/renewal blocked | Low | Multi-L2 deployment; sidecar caches current JWT locally |
| Regulatory (token classification) | CDT classified as security | Medium | Utility-focused design; legal review; progressive decentralization |
| Competitor response (Pocket V2) | Feature parity eliminates advantage | Medium | Ship fast; focus on UX and LLM as differentiators |

## 9. Open Questions

1. **Which L2 to deploy on first?** Arbitrum, Base, and Optimism are the leading candidates. Need to evaluate oracle support, DA integration, and ecosystem fit.

2. **Lease pricing model**: Flat rate per service? Dynamic based on demand? Provider-set pricing? Market-based?

3. **Token launch strategy**: Fair launch? IDO? Bootstrap liquidity pool? Need to balance decentralization with pragmatic fundraising.

4. **LLM content policy**: Should providers be required to filter harmful content? Should the protocol be opinionated about this or leave it to providers?

5. **Multi-chain deployment**: Should ProviderRegistry exist on multiple chains, or use a single canonical chain with cross-chain messaging?

6. **Dispute resolution**: How do consumers dispute poor service beyond automatic QoS slashing? Is there a need for a human governance layer?

7. **Provider geographic distribution**: Should the protocol incentivize geographic diversity, or let the market sort it out?
