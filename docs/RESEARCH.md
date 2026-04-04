# Conduit Protocol — Research Document

> **Status**: Draft v0.1 — Living document, updated as research progresses
> **Date**: 2026-04-04

## Table of Contents

1. [Competitive Landscape](#1-competitive-landscape)
2. [Pocket Network Deep Dive (PATH Analysis)](#2-pocket-network-deep-dive)
3. [Oracle Frameworks](#3-oracle-frameworks)
4. [Data Availability Layers](#4-data-availability-layers)
5. [JWT in Decentralized Systems](#5-jwt-in-decentralized-systems)
6. [Token Economics Research](#6-token-economics-research)
7. [L2 Deployment Analysis](#7-l2-deployment-analysis)
8. [Hermes Model Analysis](#8-hermes-model-analysis)
9. [Open Research Questions](#9-open-research-questions)

---

## 1. Competitive Landscape

### 1.1 Decentralized RPC Providers

#### Pocket Network (POKT)

**Overview**: The largest decentralized RPC protocol. Operates a relay network where consumers send requests through gateways that route to node operators (suppliers).

**Architecture**:
- Shannon protocol (V1) with PATH gateway
- Session-based access: sessions tied to block height, auto-rollover
- Per-relay metering: every request is counted, signed, and settled
- Centralized gateway (PATH) proxies all requests

**Key Issues**:
- **Reliability**: Not considered production-grade by enterprises. Session rollover can cause brief outages. QoS is gateway-dependent, not protocol-enforced.
- **Latency**: Extra hops through gateway + relay miner add 50-200ms+ overhead
- **Provider complexity**: Running a relay miner requires significant infrastructure beyond the actual node
- **Session overhead**: Constant rollover, relay counting, per-request signing creates computational and economic overhead

**Token**: $POKT — inflationary model, used for staking by both apps and nodes

**What to learn from**: Their QoS scoring model (0-100 with tiers) is well-designed. Their health check approach tests the full relay path. TLD diversity in endpoint selection is smart.

**What to avoid**: Sessions, relay mining complexity, centralized gateway dependency, per-request overhead.

#### Lava Network

**Overview**: Multi-chain RPC access protocol with a focus on incentive alignment between providers and consumers.

**Architecture**:
- Providers (called "providers") stake LAVA tokens per chain they support
- Consumers pair with providers through on-chain pairing mechanism
- QoS scoring affects provider rewards
- Relay proofs submitted on-chain for payment settlement

**Differentiator vs Pocket**: Stronger focus on multi-chain from day one; pairing mechanism differs from Pocket's session model.

**Token**: $LAVA

#### dRPC

**Overview**: Decentralized RPC network focusing on reliability through provider redundancy.

**Architecture**: Aggregates multiple RPC providers (both centralized and decentralized) behind a unified API.

**Relevance to Conduit**: Shows market demand for reliability over pure decentralization.

#### Ankr

**Overview**: Hybrid centralized/decentralized RPC. Runs its own nodes but also allows independent node operators.

**Token**: $ANKR — used for staking and payment

**Relevance**: Demonstrates the "excess capacity" model we're targeting. Node operators can monetize idle capacity.

### 1.2 Decentralized LLM Inference

#### Bittensor (TAO)

**Overview**: Decentralized AI network organized into subnets. Each subnet focuses on a specific AI task (text generation, image generation, etc.).

**Architecture**:
- Subnet model: validators assess miner (provider) output quality
- Incentive mechanism rewards best-performing miners
- Validators act as QoS oracles (similar to our concept)

**Token**: $TAO — high market cap, used for staking and rewards

**Relevance to Conduit**: Their subnet/validator model is closest to our QoS oracle concept. However, they're focused on AI-specific tasks, not general RPC. Their validator scoring could inform our QoS oracle design.

**Key difference**: Bittensor is AI-first; Conduit is infrastructure-first (RPC + inference).

#### Ritual Network

**Overview**: On-chain AI inference. Focuses on bringing AI computation results on-chain with verifiable proofs.

**Architecture**: Coprocessor model — AI computations happen off-chain with proofs verified on-chain.

**Relevance**: Their verifiable inference approach could inform our LLM QoS checks.

#### Akash Network

**Overview**: Decentralized compute marketplace. Not specific to RPC or inference but provides the underlying GPU/CPU capacity.

**Token**: $AKT

**Relevance**: Akash is lower in the stack (raw compute). Conduit is higher (specific endpoints for RPC/inference). Could potentially run on Akash infrastructure.

#### io.net

**Overview**: GPU aggregation network. Pools GPU capacity from data centers, miners, and individuals.

**Relevance**: Potential integration target — providers on Conduit could source GPU capacity from io.net for LLM inference.

### 1.3 Centralized Comparison Points

#### Alchemy / Infura / QuickNode

**Why enterprises use them**:
- SLA guarantees (99.9%+ uptime)
- Consistent latency
- Simple API key auth
- Dashboard with analytics
- Support team

**What Conduit needs to match**:
- Reliability (our QoS oracle addresses this)
- Simple onboarding (our lease model addresses this)
- Low latency (direct connections address this)
- Analytics (consumer sidecar metrics address this)

### 1.4 Competitive Positioning

```
                    Decentralized ◄──────────────────► Centralized
                         │                                  │
                    Pocket Network                    Alchemy/Infura
                    Lava Network                      QuickNode
                    Bittensor (AI)
                         │
                    ┌────┴────┐
                    │ CONDUIT │  ← Decentralized reliability
                    └────┬────┘    with centralized UX
                         │
                    Direct access
                    Lease-based
                    Oracle QoS
```

---

## 2. Pocket Network Deep Dive (PATH Analysis)

> Based on direct analysis of https://github.com/pokt-network/path

### 2.1 Architecture Overview

PATH (Protocol Access Through HTTP) is Pocket's centralized gateway component. Every consumer request flows through it.

**Request flow**:
```
Consumer HTTP Request
  → Router (router/router.go)
  → Gateway.HandleServiceRequest (gateway/gateway.go)
  → QoS Layer (qos/evm/qos.go) — parse request, determine requirements
  → Protocol Layer (protocol/shannon/context.go) — get session, build relay
  → buildAndSignRelayRequest() — sign with gateway key
  → executeRelayRequestStrategy() — send to endpoint
  → receiveRelayResponse() — process response
  → Heuristic analysis: checkResponseSuccess()
  → Record observations (metrics, QoS signals)
  → Return to consumer
```

### 2.2 Session Model (What We're Replacing)

**Session lifecycle** (protocol/shannon/session.go):
- Sessions are retrieved from on-chain state tied to block height
- Each session contains: app address, suppliers (endpoints), session height, session ID
- Sessions auto-rollover when block height advances (fullnode_session_rollover.go)
- Extended session validity for grace periods during rollover
- App must delegate to gateway; gateway signs relays on behalf of app

**Problems with this model**:
1. **Constant rollover**: Sessions expire with block height, requiring periodic refresh
2. **App delegation**: Apps must explicitly delegate to gateways (extra on-chain tx)
3. **Relay counting**: Every relay incremented a sequence number, validated by relay miner
4. **Signature overhead**: Every relay request signed by gateway, verified by relay miner
5. **Single point of failure**: PATH gateway is centralized despite the "decentralized" branding

### 2.3 QoS / Reputation System (What We're Making an Oracle)

**Scoring** (reputation/reputation.go):
- Score range: 0-100, initial score: 80
- Signal-based: success (+1), minor error (-3), major error (-10), critical (-25), fatal (-50)
- Latency bonuses: fast <100ms (+2), penalties: slow >500ms (-1), very slow >2s (-3)
- Tier thresholds: Tier 1 (≥70), Tier 2 (≥50), Tier 3 (≥30), Probation (<30)
- Probation endpoints get 10% traffic for recovery testing

**Health checks** (gateway/health_check_executor.go):
- Leader election (Redis-based) — only one PATH instance runs checks
- Tests full relay path (not just ping)
- Supports jsonrpc, rest, websocket check types
- Configurable per service via YAML or external URL

**Endpoint selection** (qos/evm/endpoint_selection.go):
- Filter by validity (not recently invalid, block number within sync allowance, chain ID match)
- Archival filtering for historical queries
- TLD diversity preference (avoid clustering on one hosting provider)
- Fallback to stale endpoints if no valid ones available

**What to adopt**:
- Tiered scoring model (simple, effective)
- Signal classification with weights
- TLD diversity in selection
- Testing the full request path, not just ping
- Archival detection for RPC

**What to change**:
- Move from in-process to oracle (decentralized, verifiable)
- Publish scores to DA layer (transparent, auditable)
- On-chain enforcement (suspension, slashing) not just in-memory scoring
- Multi-region probing (PATH runs from one location)

### 2.4 Request Routing

- Service identified by `Target-Service-Id` header
- RPC type detection: JSON-RPC, REST, CometBFT, WebSocket
- Batch requests supported (max configurable per service, default 5500)
- Parallel relay execution via worker pool
- Optional supplier targeting via `Target-Suppliers` header

**Relevance**: Our consumer sidecar handles routing locally. The provider endpoint is called directly — no intermediate routing needed. This is fundamentally simpler.

### 2.5 Monetization

- PATH doesn't handle payment directly — it's a relay gateway
- Relay miner validates relay count against session limits
- Payment settled on-chain per session (app pays from stake)
- Metrics: relay counts, latency histograms, success/failure rates via Prometheus

**What we're replacing this with**: Lease-based flat payments, settled in batches by the Settlement Oracle.

---

## 3. Oracle Frameworks

### 3.1 Chainlink

**Chainlink Automation (formerly Keepers)**:
- Periodic on-chain execution — good fit for JWT refresh cycles and QoS updates
- Three trigger types: time-based (cron), custom logic (`checkUpkeep()`/`performUpkeep()`), log-based (react to events)
- Cost: `[tx.gasPrice * gasUsed * (1 + premium%) + (gasOverhead * tx.gasPrice)] / [LINK/NativeRate]`
- Note (Dec 2025): Time-based upkeeps now require unique forwarder authorization

**Chainlink Functions**:
- Smart contracts send JavaScript to DON for sandboxed execution
- DON aggregates return values and sends result on-chain
- Threshold-encrypted secrets for API keys (only decryptable via multi-party decryption)
- Returns up to 256 bytes per response
- Cost: gas cost (paid in LINK) + premium fee

**Chainlink Runtime Environment (CRE)** — *this is the successor architecture (live on mainnet 2025)*:
- Replaces separate Automation + Functions with modular, composable workflows
- Compose capabilities: HTTP calls, on-chain reads/writes, signing, consensus
- JPMorgan and UBS already building on CRE
- Privacy features (confidential computing) planned for early 2026
- **This could unify our JWT refresh, QoS checks, and settlement into a single CRE workflow**

**Recommendation for Conduit**:
- Phase 1: Chainlink Functions + Automation (proven, documented)
- Phase 2: Migrate to CRE workflows (unified, lower overhead)

### 3.2 Chronicle Protocol

**Key innovation: Scribe** — uses Schnorr aggregated signatures (MuSig2):
- Combines multiple validator signatures into a single "super-signature"
- Shifts computation off-chain
- **68% cheaper on L2, 60% cheaper on L1 than Chainlink**

Concrete gas comparison (Q1 2025):

| Oracle | Gas Units (ETH/USD feed) | Cost per Update |
|--------|--------------------------|-----------------|
| Chronicle | 67,700 | $2.27 |
| Chainlink | 184,800 | $6.61 |

**Scribe Optimistic** mode: Single validator vouches, challenge period follows. Incorrect data = validator banned + challenger rewarded. Analogous to optimistic rollup security.

Live on Arbitrum and OP Mainnet. 22 validator operators including Infura, Etherscan, Gnosis.

**Relevance**: If we need price feeds (CDT/USD for fiat-denominated leases), Chronicle is significantly cheaper. For custom compute (QoS checks), Chainlink Functions/CRE is still needed.

### 3.3 API3

**Airnode**: First-party oracles — API providers run their own oracle nodes.
- Serverless, "set and forget" — runs on cloud (AWS)
- Providers sign responses with their own private keys (strongest proof of data authenticity)
- Requester-specific wallets: each requester gets dedicated wallet, enabling priority gas management
- dAPIs aggregate multiple Airnode streams, remove outliers, average into single feed

**Relevance**: If Conduit providers run Airnodes to self-report health metrics, this is the most trust-minimized approach. The protocol aggregates via dAPIs.

### 3.4 Custom Oracle Network

For Conduit's specific needs, a custom oracle network might be optimal at scale:

- **JWT Oracle**: Threshold signature scheme (TSS) with multiple signers
  - 3-of-5 or 5-of-9 signers
  - Each signer independently generates JWT claims, then collectively sign
  - Key rotation via Chainlink Automation trigger, threshold-encrypted key generation

- **QoS Oracle**: Multiple independent probers
  - Each prober runs health checks from different geographic locations
  - Aggregate scores published via consensus (median of probers)
  - Probers stake CDT and earn rewards for accurate reporting

### 3.5 Oracle Cost on L2s

| Environment | Avg. Gas Cost | Oracle Update Cost (est.) |
|-------------|---------------|---------------------------|
| Ethereum L1 | ~15-30 gwei | $5-15 per update |
| Arbitrum | ~0.01 gwei | $0.01-0.10 per update |
| Base | ~0.001 gwei | $0.005-0.05 per update |
| Optimism | ~0.01 gwei | $0.01-0.15 per update |

L2 deployment cuts oracle costs by **90-99%** versus L1.

### 3.6 Can Oracles Post Directly to DA Layers?

No established pattern exists. Current architecture: oracles → L1/L2 contracts; rollups → DA layers. However, a custom approach is feasible: oracle off-chain compute calls DA layer API, posts bulk data (QoS reports, JWT metadata), then submits only the DA commitment hash on-chain. Novel but technically sound.

### 3.7 Recommendation

| Phase | Oracle Stack |
|-------|-------------|
| **Phase 1 (MVP)** | Chainlink Functions + Automation. Fastest path to working oracles. |
| **Phase 2** | Migrate to CRE workflows. Unified, composable. |
| **Phase 3** | Custom oracle network with CDT-staked operators + Chronicle for price feeds. |

---

## 4. Data Availability Layers

### 4.1 EigenDA

**How it works**:
- Data broken into fragments (blobs), dispersed among restaked ETH operators
- Cryptographic proof committed to Ethereum
- Nodes download only small fragments, use validity proofs for verification
- Throughput: 15 MB/s live, claims 100 MB/s max with DAC model

**Cost**:
- Free tier: Up to 1.28 KiB/s for 12 months (good for development)
- 10x price reduction announced in 2025
- Payment in ETH, EIGEN, or native tokens
- Specific per-MB pricing is negotiated per customer

**SDK**: EigenDA Proxy (Go), conforming to OP Alt-DA server spec. Integrations for Arbitrum Orbit, OP Stack, ZK Stack, Polygon CDK.

**Posting/reading data**: POST endpoint → receive DA certificate (valid 2 weeks). GET endpoint → submit certificate, receive decoded blob.

**Finality**: 12-15 minutes (tied to Ethereum finality)

**On-chain verification**: DAC model — committee attests to availability. Introduces trust assumption but enables higher throughput.

### 4.2 Celestia

**How it works**:
- First modular DA network. Separates data ordering/availability from execution
- Erasure coding + Data Availability Sampling (DAS) + Namespaced Merkle Trees
- Light nodes verify data availability without downloading entire blocks
- Post-Matcha upgrade (Jan 2026): 128 MB blocks

**Cost**: **~$0.81/MB** (vs. Ethereum blobs at $20.56/MB, Ethereum calldata at ~$100/MB). Eclipse paid $0.07/MB for equivalent data costing $3.83/MB on Ethereum blobs — **55x cheaper**.

**SDK (Go)**:
```go
// Create namespace and blob
ns, _ := share.NewV0Namespace(namespaceBytes)
blob, _ := blob.NewBlobV0(ns, data)
// Submit
height, _ := client.Blob.Submit(ctx, []*blob.Blob{b}, blob.NewSubmitOptions())
// Read
blobs, _ := client.Blob.GetAll(ctx, height, []share.Namespace{ns})
```

**Finality**: 6-second block time. ~10 minutes effective finality including Blobstream bridge fraud proof window.

**Blobstream (on-chain verification)**: Posts data root commitments to Ethereum via SP1 ZK proofs. Smart contracts on L2s verify blob inclusion using `DAVerifier` library. This is the most mature on-chain DA verification mechanism available.

**Scale**: 160+ GB published, 56+ rollups using Celestia.

### 4.3 Avail

**How it works**: Purpose-built DA blockchain using KZG polynomial commitments, erasure coding, and DAS. Substrate-based.

**Performance (2025-2026)**:
- Turbo mode: **250ms pre-confirmations** (fastest in industry)
- DA finality reduced from 40s to 20s
- Encrypted DA for privacy-preserving use cases
- 4 MB/block current, roadmap to 10 GB blocks

**SDK**: Nexus SDK for cross-chain flows. Integration commitments from Arbitrum, Optimism, Polygon, StarkWare, zkSync.

### 4.4 DA Layer Comparison

| Feature | EigenDA | Celestia | Avail |
|---------|---------|----------|-------|
| **Live Throughput** | 15 MB/s | 1.33 MB/s | 0.2 MB/s |
| **Cost per MB** | Tiered/negotiated | ~$0.81 | Formula-based |
| **Finality** | 12-15 min | 6s block / ~10min effective | 20s block |
| **Pre-confirmations** | No | No | 250ms (Turbo) |
| **Primary SDK** | Go (eigenda-proxy) | Go (celestia-node) | Substrate/Nexus SDK |
| **On-chain Verification** | DAC attestation | Blobstream (ZK proofs) | Validity proofs |
| **Ethereum Alignment** | Restaked ETH security | Independent chain | Independent chain |
| **Trust Model** | DAC (trust committee) | DAS (trustless sampling) | DAS + validity proofs |

### 4.5 Recommendation

**Primary: Celestia** — strongest option for Conduit given:
- Lowest documented cost (~$0.81/MB)
- Mature Blobstream bridge for on-chain verification of DA commitments
- Go SDK with simple blob submit/retrieve API
- Namespace model (organize Conduit data by service type)
- Largest ecosystem (56+ rollups, 160+ GB posted)

**Alternative: EigenDA** — if Ethereum alignment and restaking security are priorities. Free tier covers development. Better if we build as an EigenLayer AVS.

**Consideration: Avail** — 250ms pre-confirmations could be valuable for real-time QoS data, but ecosystem is less mature.

---

## 5. JWT in Decentralized Systems

### 5.1 Existing Precedents

- **DID-JWT (Decentralized Identity Foundation)**: Signs/verifies JWTs using ES256K and EdDSA. Issuer resolved via DIDs, verification checks issuer's DID and public key on-chain, queries on-chain revocation registry.
- **SD-JWT Verifiable Credentials**: Three-party lifecycle (Issuer, Holder, Verifier) with selective disclosure of claims. Governed by OID4VCI/OID4VP standards.
- **Lit Protocol**: JWT-like tokens signed by a threshold network for access control.
- **NFT-based auth (2025 research)**: Framework proposing replacing JWTs with NFTs for enterprise microservices — adds traceability, revocation, protection against session attacks.

### 5.2 JWT Issuance from Oracle

**Recommended architecture**:

1. **On-chain**: Contract stores current JWT signing public key hash, authorized provider list, lease state. Emits `LeaseAcquired` event on activation.
2. **Oracle layer (Chainlink Functions/CRE)**: Off-chain compute generates JWT:
   - Reads lease parameters from contract
   - Constructs JWT payload (consumer ID, provider endpoint, expiry, permissions)
   - Signs with key held in threshold-encrypted secrets
   - Posts JWT hash on-chain as proof of issuance
   - Delivers full JWT to DA layer
3. **Verification**: Provider validates JWT against on-chain public key hash. Revocation checked against on-chain registry.

The JWT itself never goes on-chain (too large, contains routing data). Only commitments/hashes go on-chain.

### 5.3 Key Rotation in Decentralized Context

1. **Key generation**: Oracle DON generates new signing key pair via threshold cryptography
2. **Announcement**: New public key hash posted on-chain with activation timestamp (e.g., effective in 24 hours)
3. **Grace period**: Both old and new keys valid during transition window
4. **JWKS equivalent**: Smart contract maintains `mapping(keyId => publicKeyHash => activationTime => expirationTime)` — analogous to JWKS endpoint
5. **Retirement**: Old key marked expired after grace period
6. **Trigger**: Chainlink Automation (time-based upkeep) triggers rotation on schedule

### 5.4 Provider-Side Validation

```
Provider receives request with JWT:
1. Decode JWT header → get key ID (kid)
2. Fetch oracle public key from cache (refresh from contract if kid unknown)
3. Verify signature
4. Check claims: exp, sid, sub
5. (Optional) Check jti against revocation list from DA layer
6. Serve request
```

**Latency overhead**: ~0.1ms for signature verification (negligible)

### 5.5 Revocation

**Challenge**: JWTs are stateless. Once issued, they're valid until expiry.

**Solution layers**:
1. **Short TTL** (1 hour): Limits damage window
2. **Revocation list on DA**: Oracle publishes revoked JTIs; providers check periodically
3. **Emergency on-chain revocation**: Oracle nullifies commitment on-chain; providers check commitment before serving (adds ~50ms for on-chain read, only for high-security mode)

### 5.6 ERC-4337 Patterns for Consumer Sidecar

**Key insight**: ERC-4337 (Account Abstraction) maps directly to the consumer sidecar concept.

- **Smart account as consumer identity**: Each consumer deploys an ERC-4337 smart account as their on-chain identity. Sidecar manages it.
- **Paymaster for gas abstraction**: Protocol Paymaster sponsors gas fees — consumers pay only in CDT, not ETH.
- **Batch operations**: UserOperations can batch `deposit CDT + activate lease + request JWT` into a single transaction.
- **Session keys**: Smart accounts authorize session keys with limited permissions and expiry — maps directly to JWT-like time-limited access. Sidecar holds session key, not master key.
- **EIP-7702 (Pectra, May 2025)**: Existing EOAs can temporarily execute smart contract code — lowers barrier for consumers with existing wallets.

**Scale**: 40+ million smart accounts deployed, 100+ million UserOperations processed as of 2025.

---

## 6. Token Economics Research

### 6.1 Lessons from Existing Tokens

| Token | Model | What Works | What Doesn't |
|-------|-------|------------|--------------|
| $POKT | Inflationary (mint per relay) | Aligns rewards with usage | Inflation dilutes holders; complex burn mechanics |
| $LAVA | Inflationary with burn | Provider staking + consumer staking | Dual staking creates friction |
| $TAO | Fixed supply, emission schedule | Scarcity drives value | High concentration in early miners |
| $ANKR | Fixed supply | Simple utility token | Limited staking utility |
| $AKT | Inflationary with take rate | Marketplace fee model | Low demand-side adoption |

### 6.2 Recommendations for $CDT

**Supply model**: Fixed supply with emission schedule for provider rewards
- Total supply: TBD (e.g., 1 billion CDT)
- Bootstrap emissions: Decreasing schedule over 5 years
- After bootstrap: Protocol fees fund ongoing rewards (sustainable)

**Staking**:
- Providers stake CDT per endpoint (minimum varies by service type)
- Higher stake = priority in endpoint listings (tie-breaker, not exclusive)
- Slash conditions tied to QoS oracle findings (objective, not subjective)

**Lease pricing**:
- Provider-set pricing with market dynamics
- Minimum price floor (prevents race to bottom)
- Price discovery via on-chain order book or simple posted prices

**Fee distribution**:
- 90% to providers (weighted by QoS tier and uptime)
- 5% to oracle operators
- 5% to protocol treasury

### 6.3 Burn-and-Mint Equilibrium (BME) — Strong Alternative

Research revealed BME as the emerging standard (adopted by Akash March 2026, Render 2025):
- Consumers pay CDT for leases → **tokens burned**
- Providers receive **newly minted CDT** as rewards
- Direct deflationary pressure from usage
- Cleaner than fee-split model — supply directly tracks demand

**This may be superior to our current 90/5/5 fee-split design. Warrants a tokenomics revision.**

### 6.4 ERC-4626 Vault for Payment Pool

Research uncovered ERC-4626 tokenized vault pattern as a fit for lease payments:
- Consumers deposit stablecoins (USDC) into an ERC-4626 vault
- Receive share tokens representing credit balance
- Vault can deploy idle funds to yield strategies (Aave, Compound) during period between deposit and provider payment
- When lease settles, vault redeems shares and pays provider
- Auto-compounding benefits all depositors

This could work alongside or as an alternative to direct CDT payment — consumers deposit USDC, protocol handles CDT conversion internally.

---

## 7. L2 Deployment Analysis

### 7.1 Cost Comparison

| L2 | Avg. Transaction | DeFi Swap | Contract Deploy (est.) |
|----|-----------------|-----------|----------------------|
| **Base** | ~$0.01 | ~$0.01 | ~$0.10-0.50 |
| **Arbitrum** | ~$0.05-0.30 | ~$0.03 | ~$0.50-2.00 |
| **Optimism** | ~$0.15-0.50 | ~$0.05 | ~$0.50-3.00 |
| **Polygon zkEVM** | ~$0.05-0.50 | ~$0.10 | ~$1.00-5.00 |

All L2s benefited from Ethereum's Dencun upgrade (March 2024) — 50-90% reduction in data posting costs.

### 7.2 Oracle Support by L2

| Oracle | Arbitrum | Base | Optimism | Polygon zkEVM |
|--------|----------|------|----------|----------------|
| Chainlink (full stack) | Full | Full | Full | Partial |
| Chronicle (Scribe) | Live | Limited | Live | No |
| API3 | Yes | Yes | Yes | Yes |
| Pyth | Yes | Yes | Yes | Yes |

### 7.3 DA Layer Integration by L2 Framework

| L2 Framework | Ethereum Blobs | Celestia | EigenDA | Avail |
|--------------|---------------|----------|---------|-------|
| **Arbitrum Orbit** | Default | Supported | Supported | Supported |
| **OP Stack** | Default | Supported | Supported (proxy) | Supported |
| **Base** | Ethereum blobs only | Via custom config | No native | No native |
| **Polygon CDK** | Default | Supported | Supported | Supported |

### 7.4 Recommendation — REVISED

**Primary: Arbitrum One** (or Arbitrum Orbit chain)

Rationale:
1. $0.05-0.30 per transaction, dynamic pricing prevents spikes
2. Full Chainlink stack (Automation, Functions, CRE) + Chronicle Scribe (68%+ gas savings)
3. Arbitrum Orbit natively supports Celestia, EigenDA, and Avail as DA options
4. Largest L2 DeFi ecosystem, most battle-tested
5. Full ERC-4337 support with active account abstraction ecosystem
6. Option to launch custom Orbit chain with Celestia DA in Phase 2

**Secondary: Base**

Rationale: Lowest fees (~$0.01), fastest-growing TVL (46.6% of all L2 DeFi TVL), Coinbase ecosystem. But DA layer options are more limited (Ethereum blobs only by default).

---

## 8. Hermes Model Analysis

> Query sent to `nousresearch/hermes-3-llama-3.1-405b` via OpenRouter (DeepInfra)

### 8.1 Query

Asked Hermes for technical analysis of the Conduit protocol design covering smart contract architecture, oracle design, token economics, security, DA layer choice, attack vectors, cold start problem, and comparison with Pocket Network.

### 8.2 Key Insights from Hermes

**Smart Contract Architecture**:
- Confirmed three-contract model: ProviderRegistry, LeaseContract, CDToken
- Recommended `mapping` + `struct` storage patterns
- Suggested `leasePayment(leaseId, amount)` as a dedicated payment method on the token contract

**Oracle Design**:
- Recommended Chainlink for reliability and adoption
- Confirmed JWT refresh via event monitoring → generate → sign → deliver flow
- Suggested QoS data submission on-chain for transparency

**Token Economics**:
- Suggested provider-set lease pricing based on market demand
- Confirmed staking/slashing model for provider accountability
- Recommended performance-based reward distribution

**Security**:
- Emphasized secure key management and rotation for oracle signing keys
- Suggested consumer insurance fund for protection against provider failure
- Recommended dispute resolution mechanism beyond automated slashing

**Cold Start**:
- Higher rewards for early providers, lower staking requirements
- Discounts/promotions for early consumers
- Collaborate with existing decentralized projects

**Comparison with Pocket**:
- Confirmed lease-based model provides more predictable pricing
- Direct connections reduce latency and single points of failure
- Off-chain DA minimizes gas costs
- Oracle-based QoS is more flexible than in-process monitoring

### 8.3 API Details

- Model: `nousresearch/hermes-3-llama-3.1-405b`
- Provider: DeepInfra (via OpenRouter)
- Cost: $0.001452 (351 prompt tokens, 1,101 completion tokens)
- Demonstrates: Low-cost LLM inference via decentralized routing — exactly what Conduit will enable

---

## 9. Open Research Questions

### 9.1 Technical

1. **JWT vs. alternative auth**: Should we consider EIP-4361 (SIWE) sessions or ERC-4337 session keys as alternatives to JWT?
2. **DA layer data retention**: EigenDA has ~14-day retention. Need archival strategy for audit trails. IPFS pinning? Arweave for permanent storage?
3. **Oracle decentralization timeline**: At what network size does custom oracle network become more cost-effective than Chainlink?
4. **Cross-chain state**: If deployed on multiple L2s, how do we keep provider registry in sync? Hyperlane? LayerZero? Canonical chain + mirrors?
5. **LLM output verification**: How do we verify LLM inference quality without running the same model? Probabilistic checks? Output hashing with known prompts?

### 9.2 Economic

1. **Lease pricing discovery**: Provider-set vs. algorithmic vs. auction-based?
2. **Minimum viable stake**: What's the right minimum stake per service type to prevent spam without excluding small providers?
3. **Bootstrap incentive structure**: How much of the CDT supply should go to bootstrap rewards? What's the decay curve?
4. **Oracle operator economics**: Is 5% of lease fees sufficient to attract oracle operators? What's the break-even?

### 9.3 Market

1. **Target market entry**: Start with RPC (proven demand) or LLM inference (growing demand, less competition)?
2. **Enterprise adoption**: What SLA guarantees do enterprises need? Can we formalize SLAs in smart contracts?
3. **Geographic distribution**: Should we incentivize provider distribution or let the market decide?
4. **Regulatory**: How does selling time-based access to infrastructure via a token interact with securities law?

---

## Appendix: Research Sources

- Pocket Network PATH gateway: https://github.com/pokt-network/path (direct code analysis)
- Nous Hermes 3 405B: Queried via OpenRouter API for protocol design analysis
- Competitive analysis: Web research on Pocket, Lava, Bittensor, Ritual, Akash, Ankr, dRPC
- Oracle frameworks: Chainlink docs, API3 docs, custom oracle research
- DA layers: EigenDA docs, Celestia docs, Avail docs
- Token economics: Analysis of POKT, LAVA, TAO, ANKR, AKT token models

---

## 10. Strategic Insights from Landscape Analysis

> Full competitive landscape with market data: [LANDSCAPE.md](LANDSCAPE.md)

### 10.1 Market Opportunity

The decentralized RPC market is **severely undervalued**: POKT ($30M) + LAVA ($15M) + ANKR ($50M) combined < $100M market cap. Alchemy alone is valued at $10.2B. This gap is either a market failure or an opportunity — likely the latter, given that Infura is actively decentralizing.

### 10.2 Critical Strategic Decisions

**1. Build as an EigenLayer AVS?**

Infura DIN is building as an EigenLayer AVS. This validates the model. Benefits:
- Inherit Ethereum's economic security via restaking
- Slashing mechanism provides cryptoeconomic QoS guarantees
- Tap into $8.7B TVL restaking ecosystem
- Aligns with institutional narrative

Trade-off: More complex architecture, dependency on EigenLayer. Worth a deeper evaluation — this could replace our custom staking with restaking.

**2. Adopt Burn-and-Mint Equilibrium (BME) for $CDT?**

Both Akash and Render adopted BME in 2025-2026. The model:
- Consumers pay CDT for leases → tokens burned
- Providers receive newly minted CDT as rewards
- Direct deflationary pressure from usage

This is cleaner than our current spec's "90/5/5 fee split" and has proven market acceptance. **Strong candidate for a tokenomics revision.**

**3. Quality-linked rewards are essential**

Lava's model prevents race-to-bottom: higher QoS score = larger reward share. Bittensor's Yuma consensus evaluates output quality. Our QoS oracle + tier-weighted distribution already does this, but we should make the reward differential steeper.

### 10.3 Key Competitive Metrics (April 2026)

| Project | Market Cap | Category | Threat Level |
|---------|-----------|----------|-------------|
| Alchemy | $10.2B (private) | Centralized RPC | Benchmark |
| Bittensor (TAO) | $3.3B | Decentralized AI | Low (different focus) |
| Render (RENDER) | $750M | GPU/AI | Low (lower in stack) |
| Celestia (TIA) | $265M | DA Layer | Infrastructure partner |
| The Graph (GRT) | $260M | Indexing | Infrastructure partner |
| Akash (AKT) | $140M | Decentralized Compute | Potential partner |
| EigenLayer (EIGEN) | $105M | Restaking/AVS | Infrastructure partner |
| Ankr (ANKR) | $50M | Hybrid RPC | Direct competitor |
| io.net (IO) | $32M | GPU Marketplace | Potential partner |
| Pocket (POKT) | $30M | Decentralized RPC | **Primary competitor** |
| Lava (LAVA) | $15M | Decentralized RPC | **Primary competitor** |

### 10.4 Enterprise Adoption Gap (Our Biggest Opportunity)

The #1 barrier to enterprise adoption of decentralized RPC is: **no single SLA counterparty with slashable collateral**.

Enterprises need:
- Enforceable SLAs (not paper promises)
- Sub-100ms P99 latency guarantees
- SOC2/ISO compliance
- 24/7 support
- Fiat-denominated billing

Conduit's answer: **Cryptoeconomic SLAs** — on-chain service agreements backed by staked CDT. Provider stake is real collateral, not a paper guarantee. QoS oracle provides real-time enforcement. This is the gap no one has fully filled yet.

### 10.5 Network Optimization Findings

For minimizing consumer-to-provider latency:
- **Cloudflare Tunnel** recommended for providers (free, backbone routing, DDoS protection)
- **Geographic-aware endpoint selection** in sidecar (local latency probes)
- **P99 latency** is what matters, not P50 — current decentralized networks focus on averages but enterprises care about tail latency
- Africa, South Asia, Latin America face 300-500ms penalties — geographic incentives may be needed

---

## Appendix: Research Sources

- Pocket Network PATH gateway: https://github.com/pokt-network/path (direct code analysis)
- Nous Hermes 3 405B: Queried via OpenRouter API for protocol design analysis
- Competitive landscape analysis: [LANDSCAPE.md](LANDSCAPE.md) — full report with 17 projects, April 2026 market data
- Oracle frameworks: Chainlink docs, API3 docs, custom oracle research
- DA layers: EigenDA docs, Celestia docs, Avail docs
- Token economics: Analysis of POKT, LAVA, TAO, ANKR, AKT, RENDER token models
