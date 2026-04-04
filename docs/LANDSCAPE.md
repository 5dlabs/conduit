# Conduit: Decentralized RPC & LLM Inference Marketplace Research Report

**Date:** April 4, 2026
**Scope:** Competitive landscape, infrastructure options, token economics, and technical pain points

---

## Table of Contents

1. [Decentralized RPC Providers](#1-decentralized-rpc-providers)
2. [Decentralized LLM Inference](#2-decentralized-llm-inference)
3. [Relevant Infrastructure](#3-relevant-infrastructure)
4. [Token Economics Comparison](#4-token-economics-comparison)
5. [Technical Pain Points & Enterprise Adoption Barriers](#5-technical-pain-points--enterprise-adoption-barriers)
6. [Summary Market Metrics Table](#6-summary-market-metrics-table)

---

## 1. Decentralized RPC Providers

### 1.1 Pocket Network (POKT)

**Current State:** Pocket Network completed its Shannon mainnet upgrade in June 2025, transitioning from the legacy Morse protocol to a Cosmos SDK-based architecture. The upgrade was a zero-downtime cutover via Cosmovisor. POKT surged 450% in June 2025 on the announcement.

**Architecture (Shannon):**
- Built on Cosmos SDK with Tendermint (CometBFT) consensus
- Modular architecture enabling horizontal scalability---validator sets and Gateway infrastructure scale independently
- **Session Model:** Sessions are pseudorandomly generated using on-chain block data to assign groups of ~5 nodes to applications. Sessions "tumble" periodically based on block count for security rotation.
- **PATH Gateway:** The open-source demand-side client (Path API & Toolkit Harness) handles request intake, routing, and proof packaging off-chain. Staked on-chain Gateways sign work.
- **RelayMiners:** Off-chain actors that route requests from gateways to suppliers, collect responses, and package them into relay proofs submitted on-chain.
- **Relay Mining & Probabilistic Proofs:** Shannon uses probabilistic proof submission to handle high relay volumes without overwhelming the blockchain.
- **Grove:** The primary gateway operator (Grove.city) through which relay requests are routed to the decentralized provider set.

**Scope Expansion:** The protocol transitioned from blockchain-specific RPC into a "universal open data fabric"---now serving AI model inference, traditional Web2 APIs, indexing services, and oracles through the same decentralized node network.

**Pricing:** Wholesale model---claims >50% savings vs. centralized providers. Usage-based burn economics.

**Market Metrics (April 2026):**
- Market Cap: ~$30M
- Token Price: ~$0.013
- Circulating Supply: ~2.3B POKT

**Criticisms & Issues:**
- June 2025 was essentially a nil revenue month during the Shannon migration, with multiple weeks of low/no production traffic.
- Visible codebase activity on core repositories has slowed, raising questions about developer momentum.
- Exchange disruptions (Bithumb suspended POKT deposits/withdrawals during protocol upgrades).
- Foundation assumed direct operational responsibility for critical infrastructure in Q4 2025, raising centralization concerns.
- Still dependent on Grove as the primary gateway---single-gateway concentration risk.

---

### 1.2 Lava Network (LAVA)

**Architecture:**
- Built on Cosmos SDK with Delegated Proof-of-Stake (DPoS) via CometBFT consensus
- IBC-enabled for cross-chain compatibility
- **Pairing System:** Algorithmically generated PairingLists match consumers to providers based on: (1) provider stake, (2) epoch blockhash, (3) geolocation, (4) consumer address.
- **Specifications ("Specs"):** Governance-defined blueprints that define supported APIs. Specs can be added/updated via on-chain governance, enabling dynamic support for new chains and APIs.
- **Conflict Resolution:** Uses an Honest Majority protocol---a stake-weighted vote of validators resolves data discrepancies. Consumers also use a threshold algorithm to sample API endpoints and probabilistically minimize response conflicts.
- **Subscription Model:** Consumers pay LAVA for subscriptions granting RPC access. Creates a reliable revenue stream for providers.

**Scale:** 50+ supported blockchains, 2B+ daily RPC requests.

**Differentiators vs. Pocket:**
- More focused on RPC quality and provider competition rather than becoming a "universal data fabric"
- Peer-to-peer consumer-provider interaction with on-chain dispute resolution
- Specification system is more modular and governance-driven for adding new chains
- Revenue model driven by subscriptions rather than relay-count minting

**Market Metrics (April 2026):**
- Market Cap: ~$15M
- Token Price: ~$0.03
- Revenue: >$3.5M generated since April 1, 2026 with NEAR, Starknet, Filecoin, Axelar contributing >$1M in payments

**Key Risk:** Small market cap and nascent adoption. Subscription model requires significant demand-side growth.

---

### 1.3 Ankr (ANKR)

**Architecture:**
- Hybrid model: Ankr-operated nodes alongside independent node operators
- 75+ supported blockchains
- **RPCfi (Oct 2025):** Upgraded tokenomics structure tying RPC usage to token value

**Staking Mechanics:**
- Node providers must self-stake 100,000 ANKR as an insurance deposit
- Reward distribution: 70% to the node (49% individual stakers, 51% node providers), 30% to Ankr Treasury (DAO-controlled)

**Differentiators:**
- More centralized than POKT/LAVA---Ankr maintains its own node infrastructure alongside the decentralized network
- Broader product suite: RPC, liquid staking (ankrETH), blockchain creation service, AppChains
- Heurist Chain L2 for AI cloud computing planned for 2026

**Market Metrics (April 2026):**
- Market Cap: ~$50M
- Token Price: ~$0.005
- Circulating Supply: 10B ANKR

---

### 1.4 dRPC

**Architecture:**
- Decentralized routing layer with independent data providers
- Built on **Dshackle**, a customized fork of the open-source library from Emerald, serving as the provider-side adapter
- Dshackle tracks node state, prioritizes by reliability, executes requests with precision
- 7 geo-distributed clusters with multi-region failover
- 95+ supported blockchains, up to 5,000 RPS

**Product Tiers:**
- **NodeCloud:** Instant access, minimal infrastructure overhead
- **NodeCore:** Self-hosted with open-source stack
- **NodeHaus:** Managed service for blockchain foundations with 24/7 support

**Pricing:** Flat-rate method pricing---20 CU per request regardless of method, at $6 per 1M requests.

**Differentiators:**
- No token (as of 2026)---purely service-based business model
- Provider selection process evaluates service quality and reliability (curated, not fully permissionless)
- Focused on developer experience and operational simplicity
- Open routing library approach

**Key Insight:** dRPC demonstrates that a decentralized RPC network can work without a token, using traditional SaaS pricing. This is a relevant model for Conduit.

---

### 1.5 Infura DIN (Decentralized Infrastructure Network)

**Architecture:**
- Built as an EigenLayer AVS (Actively Validated Service)
- Backed by stETH restaking for economic security
- Permissionless participation from node operators, watchers, and restakers
- Currently handling 13B+ requests/month across 30+ networks

**Key Developments:**
- Ephemeral testnet completed March 2025 (9 providers + Watcher onboarded)
- EigenLayer AVS mainnet launched November 2025
- Token component on the roadmap (confirmed by Joseph Lubin)

**Strategic Significance:**
- Infura processes 70-80% of all RPC traffic today---their decentralization effort validates the market thesis
- Leverages EigenLayer's slashing for economic security guarantees
- Could become the largest decentralized RPC network by volume overnight if migration completes

**Key Risk:** Consensys centralization concerns. Token not yet launched. Unclear timeline for full decentralization.

---

### 1.6 MASQ

**Relevance to Conduit: LOW.** MASQ is a decentralized mesh network (VPN/Tor hybrid) with RPC support as a secondary feature. It supports Polygon chain interaction but is not primarily an RPC infrastructure play. Market cap is negligible. Not a meaningful competitor in the decentralized RPC space.

---

### 1.7 Alchemy (Centralized Comparison Point)

**Market Position:** Powers 70% of top Ethereum DApps (OpenSea, Uniswap). Processes $4.2T+ in yearly transaction volume.

**Pricing:**
- Free: 30M Compute Units/month
- Pay-as-you-go: $0.40 per 1M CUs
- Enterprise: Custom pricing with lower per-unit costs

**Why Enterprises Choose Alchemy Over Decentralized:**
- Single accountable vendor for SLA contracts
- 24/7 support lines
- Predictable performance (sub-100ms latency guarantees)
- Enterprise-grade compliance and security certifications
- Custom RPS setups, global scaling

**Benchmark:** This is the bar Conduit must meet or exceed to capture enterprise traffic.

---

## 2. Decentralized LLM Inference

### 2.1 Ritual Network

**Architecture:**
- **Infernet:** Lightweight library connecting off-chain AI computations with on-chain smart contracts. 8,000+ independent Infernet nodes with diverse hardware profiles.
- **Ritual Chain:** Sovereign Layer 1 blockchain building on EVM++ with specialized nodes for TEE execution, ZK verification, and GPU-based AI inference.
- **EVM++ Sidecars:** Dedicated execution environments running model inference outside the execution client, scaling in parallel to the core execution layer.
- **Symphony:** Accelerated consensus using Execute-Once-Verify-Many-Times (EOVMT) model---select compute nodes execute workloads and generate succinct sub-proofs.
- **Verification:** Flexibly supports ZK-ML, TEE attestations, optimistic ML, and probabilistic proof ML natively.

**Funding:** $135M total over 4 rounds (including $25M Series A in June 2024). Founded by Niraj Pant and Akilesh Potti.

**Token:** Not yet launched (as of April 2026).

**Differentiators:**
- Most sophisticated verification approach (multi-modal: ZK, TEE, optimistic, probabilistic)
- Native smart contract integration via Infernet
- Building a dedicated L1 rather than bolting onto existing chains
- Strong VC backing (Greylock, etc.)

**Key Risk:** Pre-token, pre-mainnet for Ritual Chain. Execution risk is high.

---

### 2.2 Bittensor (TAO)

**Architecture:**
- Decentralized ML ecosystem built on Substrate (Polkadot SDK)
- **Subnet Model:** 128+ active subnets, each focusing on unique capabilities (NLP, computer vision, inference optimization, data routing). Each subnet is an independent competitive marketplace.
- **Dynamic TAO (dTAO, Feb 2025):** Each subnet issues its own "alpha" token via subnet-specific liquidity pools. TAO emissions allocated based on net TAO inflow---subnets generating real value receive more emissions.
- **Yuma Consensus (Intelligence Consensus):** Evaluates the usefulness of AI outputs from miners rather than just validating transactions. Participants rewarded based on quality and performance.

**Key 2025-2026 Milestones:**
- First halving December 14, 2025 (daily emissions cut from 7,200 to 3,600 TAO)
- Covenant-72B: 72B-parameter decentralized AI model trained by 70+ contributors using commodity hardware (TAO surged 100% on this in March 2026)
- Grayscale filed for a Bittensor trust (December 2025)
- Plans to expand beyond 128-subnet limit in 2026-2027

**Market Metrics (April 2026):**
- Market Cap: ~$3.3B
- Token Price: ~$310
- Staking Ratio: 76.55% (highest among major PoS networks)
- Hard Cap: 21M TAO (Bitcoin-like scarcity)

**Differentiators:**
- Largest and most mature decentralized AI network
- Subnet competition model creates a Darwinian marketplace for AI capabilities
- dTAO creates strong economic alignment between token value and actual subnet performance
- Institutional interest (Grayscale trust filing)

**Key Risks:**
- Quality variance across subnets is extreme
- Gaming of Yuma consensus (validators colluding)
- Complexity of subnet evaluation for stakers
- Not specifically optimized for inference latency

---

### 2.3 Akash Network (AKT)

**Architecture:**
- Decentralized cloud computing marketplace on Cosmos SDK
- Reverse-auction deployment model (users post requirements, providers bid)
- **Supercloud Platform:** Production-ready as of 2025 with 60% GPU utilization rate
- **Starcluster:** Protocol-owned compute combining centrally managed datacenters with decentralized GPU marketplace. Targeting ~7,200 NVIDIA GB200 GPUs.
- **AkashML:** Managed inference layer simplifying access to decentralized GPU infrastructure

**Key 2025-2026 Milestones:**
- Mainnet 14: Migrated to Cosmos SDK v0.53, eliminated 8 years of technical debt
- NVIDIA Blackwell (B200/B300) support live and scaling
- Consumer GPU validation: Clustered RTX 4090s reducing inference costs by 75% vs. H100s
- **Burn-Mint Equilibrium (BME)** upgrade (March 23, 2026): Overhauled tokenomics to peg supply to network compute demand

**Market Metrics (April 2026):**
- Market Cap: ~$140M
- Token Price: ~$0.60
- 99% demand surge over 30 days (March 2026)

**Differentiators:**
- Most production-ready decentralized compute platform
- General-purpose cloud (not AI-only)---compute, storage, networking
- Starcluster hybrid model (protocol-owned + decentralized) addresses quality concerns
- Consumer GPU clustering is a genuine cost innovation

**Key Risks:**
- Enterprise trust in decentralized cloud for production workloads
- Starcluster introduces centralization
- Competition from io.net, Render for GPU-specific workloads

---

### 2.4 Gensyn

**Architecture:**
- Decentralized protocol for ML training and inference across any device
- **RL Swarm:** Collaborative post-training via reinforcement learning reasoning over the internet. Multiple models communicate, criticize, and improve each other.
- **Verification:** Cryptographic proofs ensure ML computations performed by independent hardware providers are correct and tamper-proof.
- Off-chain execution with on-chain coordination (payments, dispute resolution, task assignment, validation checkpoints)

**Token:** AI token sale December 15-20, 2025. 300M tokens (3% of supply). Valuation cap: $1B FDV ($0.10/token). Same price as a16z venture round.

**Market Metrics (April 2026):**
- Early stage---token distribution and mainnet still ramping
- FDV: ~$1B (at sale price)

**Differentiators:**
- Focused specifically on ML training (not just inference)---unique positioning
- RL Swarm is genuinely novel for collaborative model improvement
- Backed by a16z
- Psyche network / DisTrO optimizer for distributed training

**Key Risks:**
- Very early stage (testnet launched March 2025)
- Training verification is harder than inference verification
- Token liquidity unclear

---

### 2.5 Nous Research / Hermes

**Models:**
- **Hermes 4.3-36B** (December 2025): 36B parameter LLM with 512K context, matching Hermes 4 70B at half the parameter cost. Hybrid reasoning mode with explicit thinking segments.
- Optimized for local deployment---fits in consumer GPU VRAM

**Decentralized Training (Psyche Network):**
- Hermes 4.3 was the first production model post-trained entirely on the Psyche network
- Uses DisTrO optimizer for efficient communication between training nodes over the open internet
- Secured by Solana blockchain consensus
- Significantly reduces training costs

**Relevance to Conduit:** Nous/Hermes models are strong candidates for serving through a decentralized inference marketplace. Their optimization for local/efficient inference and proven decentralized training pipeline make them natural partners. The Psyche network demonstrates viable decentralized ML coordination.

---

### 2.6 io.net (IO)

**Architecture:**
- Decentralized GPU network on Solana blockchain
- Aggregates underutilized GPUs from data centers, crypto miners, and other sources
- **IO Cloud:** Decentralized GPU marketplace
- **IO Intelligence:** AI infrastructure platform
- **Co-Staking Marketplace** (Feb 2025): IO holders stake alongside GPU operators

**Scale:** 327K verified GPUs (up from 60K in March 2024), 5,350 cluster-ready.

**Pricing:** Up to 90% cheaper than traditional providers, 70% cost savings vs. AWS/GCP.

**Market Metrics (April 2026):**
- Market Cap: ~$32M
- Token Price: ~$0.10
- Circulating Supply: 310M IO
- Deflationary tokenomics (revenue-funded buyback and burn)

**Key Risk:** Very low market cap relative to claimed GPU network size. Token unlock pressure (13.29M IO unlocking April 11). Revised token model pending.

---

### 2.7 Render Network (RENDER)

**Architecture:**
- Decentralized GPU rendering platform (originally focused on 3D rendering, expanding to AI)
- Migrated to Solana for higher throughput
- **Burn-and-Mint Equilibrium (BME):** Users pay RENDER for jobs (tokens burned), node operators receive newly minted RENDER as rewards
- **Dispersed Subnet:** Dedicated subnet for AI workloads (~$0.69/GPU hour)
- **RNP-023:** Proposal to integrate Salad's ~60,000 consumer-grade GPUs as an exclusive subnet

**Scale:** ~1.5M frames/month throughput. 35% of all-time frames completed in 2025 alone.

**Market Metrics (April 2026):**
- Market Cap: ~$750M
- Token Price: ~$1.45
- Max Supply: 644.2M RENDER

**Differentiators:**
- Most established decentralized GPU network (strong in rendering/VFX/gaming)
- BME model creates direct deflationary pressure from usage
- Expanding into AI compute creates dual demand driver

**Key Risk:** Historically rendering-focused---AI pivot is unproven at scale. Competition from Akash, io.net.

---

## 3. Relevant Infrastructure

### 3.1 EigenLayer / EigenDA

**EigenLayer Architecture:**
- Restaking protocol allowing ETH validators to reuse staked ETH to secure additional services (AVSs)
- Three core components: restakers (security), operators (infrastructure), AVSs (consumers)
- **Slashing** (live since April 2025): AVSs set custom slashing conditions. Unique Stake Allocation system isolates risk---operators allocate specific ETH percentages per AVS.
- 14-day exit queue for withdrawals
- Only fee-paying AVSs eligible for incentives (anti-passive-farming)

**EigenDA:**
- Data Availability Committee (DAC) model, not a publicly verified blockchain
- V2 achieving 100MB/s throughput (significantly higher than Celestia mainnet)
- Supports native token restaking for L2 networks
- Teams can restake ERC-20 tokens to secure custom quorums

**Market Metrics (April 2026):**
- EIGEN Market Cap: ~$105M
- TVL: $8.7B (third-largest DeFi protocol)
- Ecosystem TVL: $15.8B across restaking protocols

**Relevance to Conduit:**
- Infura DIN is already building as an EigenLayer AVS---this validates the model
- EigenDA could serve as DA layer for off-chain storage of inference results, relay proofs
- Slashing mechanism provides cryptoeconomic QoS guarantees
- Could build Conduit as an AVS to inherit Ethereum's security

---

### 3.2 Celestia

**Architecture:**
- Modular DA layer---decouples consensus and data availability from execution
- Data Availability Sampling (DAS) allows light nodes to verify data without downloading the entire blockchain
- Namespaced Merkle Trees for organizing data by rollup/application

**Key 2025-2026 Developments:**
- **Matcha Upgrade (v6):** 128MB blocks, inflation cut from 5% to 2.5%
- **Fibre Protocol:** Parallel protocol targeting 1 Tbit/s of dedicated blockspace for real-time applications
- 56+ rollups using Celestia, supported by all major rollup frameworks (Arbitrum Orbit, OP Stack, Polygon CDK)

**Market Metrics (April 2026):**
- TIA Market Cap: ~$265M
- Token Price: ~$0.29
- Circulating Supply: 898.6M TIA

**Relevance to Conduit:**
- Alternative DA layer to EigenDA with stronger decentralization properties
- DAS enables light client verification---useful for Conduit nodes verifying data availability without full download
- Fibre's real-time blockspace could serve agentic/streaming AI inference results

**EigenDA vs. Celestia Tradeoff:**
| Factor | EigenDA | Celestia |
|--------|---------|----------|
| Throughput | 100MB/s | Lower (improving with Matcha) |
| Trust Model | DAC (trust committee) | DAS (trustless sampling) |
| Security | Inherits ETH security via restaking | Own validator set |
| Cost | Lower (piggybacking on ETH) | Moderate (own token economics) |
| Decentralization | Lower (committee-based) | Higher (light node verification) |

---

### 3.3 Oracle Solutions

#### Chainlink
- **Market Position:** 67%+ oracle market share, $6.1B market cap
- **Economics 2.0:** 700M+ LINK staked, 4.5-7.2% APY
- **CCIP:** $18B/month cross-chain volume
- **Relevance:** Could handle QoS data feeds, JWT refresh oracle, cross-chain token transfers

#### API3
- **Approach:** DAO-governed, first-party oracle (API providers run their own nodes)
- **Mandatory SLAs** built into governance policies
- **Relevance:** First-party oracle model aligns well with Conduit's provider accountability needs. Providers reporting their own QoS metrics directly.

#### Band Protocol
- **Approach:** Independent BandChain for fast, low-fee data processing with IBC cross-chain support
- **Relevance:** Good for cross-chain data flows but less feature-rich than Chainlink for complex oracle needs

**Recommendation for Conduit:** Chainlink for production reliability and CCIP for cross-chain, potentially API3's first-party model for provider self-reporting QoS data.

---

### 3.4 The Graph

**Architecture:**
- Decentralized indexing protocol for querying blockchain data
- Subgraphs define what data to index and how
- **Horizon Mainnet (Q1 2026):** Unifies Subgraphs, Substreams, and Token API into a single multi-service blockchain

**Scale:** 5B+ monthly queries across thousands of subgraphs.

**Market Metrics (April 2026):**
- GRT Market Cap: ~$260M
- Token Price: ~$0.024

**Relevance to Conduit:**
- Essential for reading on-chain state (provider stakes, session data, relay counts)
- Subgraphs can index Conduit's smart contracts for dashboards and analytics
- Token API beta enables real-time token balance queries across chains

---

## 4. Token Economics Comparison

### 4.1 POKT (Pocket Network)

| Feature | Detail |
|---------|--------|
| **Staking Minimum** | 60,000 POKT for node operators |
| **Reward Source** | Relay fees (minted proportional to relays served) |
| **Deflation** | PIP-41: 2.5% burn per transaction cycle (100% burned, 97.5% re-minted) |
| **Reward Distribution** | Split between service nodes, validator nodes, and DAO |
| **Slashing** | Not prominently documented in Shannon; session rotation handles some security |
| **Annual Yield** | 10-40% for node operators |
| **What Works** | Deflationary model creates genuine scarcity; relay-proportional rewards align incentives |
| **What Doesn't** | Nil revenue months during migrations; gateway concentration (Grove); high staking minimum barriers |

### 4.2 LAVA (Lava Network)

| Feature | Detail |
|---------|--------|
| **Block Rewards** | Fixed 3.4% of total supply, monthly over 4 years, with variable burn |
| **Provider Drops** | 6.6% of total supply over 4 years (bootstrap incentive) |
| **Subscription Revenue** | Providers + restakers earn 95% of subscription rewards |
| **Staking Model** | Delegate to validators (block rewards) + restake to RPC providers (subscription rewards) |
| **QoS Linkage** | Higher performance = larger rewards; more LAVA staked = more selection probability |
| **What Works** | Revenue-driven APR (not inflationary); quality-linked rewards |
| **What Doesn't** | Bootstrapping problem---Provider Drops are temporary; small market cap limits provider incentives |

### 4.3 TAO (Bittensor)

| Feature | Detail |
|---------|--------|
| **Supply Cap** | 21M TAO (Bitcoin-like) |
| **Halving** | First halving Dec 14, 2025 (3,600 TAO/day) |
| **dTAO** | Each subnet has alpha tokens; TAO emissions allocated by net inflow |
| **Yuma Consensus** | Evaluates AI output quality, not just transaction validity |
| **Staking Ratio** | 76.55% (highest among major PoS) |
| **What Works** | Scarcity model + halving creates strong long-term value thesis; dTAO aligns emissions to real value |
| **What Doesn't** | Yuma consensus gaming; extreme quality variance across subnets; complexity deters casual stakers |

### 4.4 ANKR

| Feature | Detail |
|---------|--------|
| **Staking Minimum** | 100,000 ANKR for node providers |
| **Reward Split** | 70% to node (49% stakers / 51% provider), 30% to DAO Treasury |
| **RPCfi (2025)** | Ties RPC usage directly to token value |
| **What Works** | Clear revenue split; DAO treasury for ecosystem funding |
| **What Doesn't** | Hybrid centralized/decentralized model means Ankr retains significant control; low token price limits provider economic security |

### 4.5 Key Token Economics Insights for Conduit

1. **Burn-and-mint equilibrium (BME)** is the emerging standard (Akash, Render adopted it). Creates direct deflationary pressure from usage without relying on inflationary staking rewards.
2. **Revenue-driven staking yields** (Lava model) are healthier than inflationary rewards, but require bootstrapping incentives.
3. **Quality-linked rewards** (Lava, Bittensor) are essential---flat per-relay rewards without QoS weighting lead to a race to the bottom.
4. **Subnet/service-specific token pools** (Bittensor's dTAO) enable market-driven resource allocation.
5. **Hard supply caps** (TAO) create long-term value conviction but may limit network growth flexibility.
6. **Slashing is underutilized** across decentralized RPC networks. EigenLayer's approach (custom per-AVS slashing) is the most sophisticated model available.

---

## 5. Technical Pain Points & Enterprise Adoption Barriers

### 5.1 Known Issues with Decentralized RPC

**Reliability Variance:**
- Performance varies heavily depending on the specific operator. Decentralized RPC trades consistency for resilience.
- P95/P99 latency (not P50 average) is what competitive operations require. Decentralized networks struggle with tail latency.
- Centralized providers maintain <100ms average latency; decentralized endpoints can spike to several seconds or return 429 errors.

**Session Overhead:**
- Pocket's session model adds latency for initial node assignment
- Session tumbling creates periodic disruption
- Provider rotation can route to suboptimal nodes

**Geographic Gaps:**
- Africa, South Asia, Latin America face 300-500ms higher latency despite growing blockchain adoption
- Chicken-and-egg: providers need user density to justify local infrastructure, users need infrastructure for adoption

**Data Consistency:**
- No guarantee all nodes in a decentralized network have identical chain state
- Block height discrepancies, stale data, and reorg handling vary by provider

### 5.2 Why Enterprises Avoid Decentralized RPC in Production

1. **No single SLA counterparty:** Enterprises need one vendor accountable via contract. Decentralized networks have no such entity.
2. **Unenforceable SLAs:** Traditional SLAs are paper contracts offering post-mortem apologies, not real-time guarantees. Breach resolution takes 30-90 days.
3. **Compliance gaps:** Enterprise use cases require SOC2, ISO certifications, data residency guarantees---impossible in a permissionless network.
4. **Unpredictable costs:** Token-denominated pricing introduces FX risk. CU-based pricing from Alchemy/Infura is more predictable.
5. **Support expectations:** Enterprises expect 24/7 support lines and dedicated account managers.
6. **Production pattern:** Most production stacks use a staked/native RPC for critical writes, multi-chain provider for resilience, and decentralized gateway only for fallback.

### 5.3 Challenges for Production-Grade Decentralized Inference

**Latency:**
- Geographic distribution introduces >200ms RTT before inference begins (e.g., prompt from Mumbai served from Oregon)
- Centralized data centers can address this but undermine decentralization
- Model partitioning across nodes adds coordination overhead

**Verification Overhead:**
- ZK-ML proofs are computationally expensive (often more expensive than the inference itself)
- TEE attestations add latency and limit hardware choices
- Optimistic verification has fraud proof windows that delay finality

**Trust and Security:**
- Sybil attacks in open networks
- Data privacy during inference (user prompts may contain sensitive data)
- Model integrity---ensuring the claimed model is actually running

**Hardware Heterogeneity:**
- Consumer GPUs have vastly different capabilities
- Inference latency varies dramatically across hardware
- VRAM limitations constrain model sizes on consumer hardware

### 5.4 What Would Make It Production-Grade

1. **Cryptoeconomic SLAs:** On-chain service agreements with slashing for downtime (EigenLayer model). Staked capital as real collateral, not paper guarantees.
2. **Geographic-aware routing:** Latency-optimized provider selection with geo-distributed node requirements.
3. **Hybrid trust model:** TEE for privacy-sensitive inference + probabilistic verification for throughput (Ritual's approach).
4. **Tiered service classes:** Not all requests need the same QoS. Allow enterprise users to select latency/reliability tiers at different price points.
5. **Provider reputation systems:** On-chain track records with historical latency, uptime, and accuracy data.
6. **Fiat-denominated pricing options:** Abstract away token volatility for enterprise billing.

---

## 6. Summary Market Metrics Table

| Project | Token | Market Cap (Apr 2026) | Token Price | Key Metric | Category |
|---------|-------|-----------------------|-------------|------------|----------|
| Pocket Network | POKT | ~$30M | $0.013 | Universal data fabric | Decentralized RPC |
| Lava Network | LAVA | ~$15M | $0.03 | 2B+ daily RPC requests | Decentralized RPC |
| Ankr | ANKR | ~$50M | $0.005 | 75+ chains supported | Hybrid RPC |
| dRPC | None | Private | N/A | 95+ chains, 5K RPS | Decentralized RPC |
| Infura DIN | Pending | N/A | N/A | 13B+ requests/month | Decentralized RPC |
| Alchemy | None | Private ($10.2B last) | N/A | 70% ETH dApp market share | Centralized RPC |
| Bittensor | TAO | ~$3.3B | $310 | 128+ subnets, 76.5% staked | Decentralized AI |
| Ritual | Pending | N/A (raised $135M) | N/A | 8,000+ Infernet nodes | On-chain AI Inference |
| Akash Network | AKT | ~$140M | $0.60 | 60% GPU utilization | Decentralized Compute |
| Gensyn | AI | ~$1B FDV (at sale) | $0.10 (sale) | RL Swarm testnet | ML Training |
| io.net | IO | ~$32M | $0.10 | 327K verified GPUs | GPU Marketplace |
| Render Network | RENDER | ~$750M | $1.45 | 1.5M frames/month | GPU Rendering/AI |
| Nous/Hermes | None | N/A | N/A | Hermes 4.3-36B model | Open-source AI |
| EigenLayer | EIGEN | ~$105M | $0.18 | $8.7B TVL | Restaking / DA |
| Celestia | TIA | ~$265M | $0.29 | 56+ rollups, 128MB blocks | Data Availability |
| Chainlink | LINK | ~$6.1B | $8.62 | 67%+ oracle market, 700M staked | Oracle |
| The Graph | GRT | ~$260M | $0.024 | 5B+ monthly queries | Indexing |

---

## Key Takeaways for Conduit

### Market Opportunity
1. **The decentralized RPC market is fragmented and undervalued.** POKT ($30M), LAVA ($15M), ANKR ($50M) combined are under $100M market cap. Meanwhile, Alchemy alone is valued at $10.2B. This gap represents either a market failure or an opportunity.

2. **Infura's DIN validates the hybrid model.** The largest centralized provider is actively decentralizing via EigenLayer. This confirms the market direction but also means Conduit will face competition from incumbents.

3. **LLM inference decentralization is pre-product-market-fit.** Bittensor has scale ($3.3B) but quality problems. Ritual has architecture but no mainnet. The first network to deliver consistent, low-latency, verified inference wins.

### Architecture Recommendations
1. **Build as an EigenLayer AVS** to inherit Ethereum's security and tap into the restaking ecosystem. Infura DIN is already doing this.
2. **Use Celestia or EigenDA** for data availability depending on trust model preference.
3. **Adopt Burn-and-Mint Equilibrium** for token economics (proven by Akash, Render).
4. **Implement quality-linked rewards** (Lava model) to prevent race-to-the-bottom pricing.
5. **Support both RPC and LLM inference** from day one---Pocket's pivot to "universal data fabric" validates this convergence.
6. **Offer tiered service with cryptoeconomic SLAs** backed by staked collateral and slashing---this is the single biggest gap in the current market.

### Competitive Moats to Build
1. **Latency optimization:** Geographic-aware routing with P99 latency guarantees (not just P50)
2. **Enterprise billing:** Fiat-denominated pricing with token abstraction
3. **Verification flexibility:** Support multiple verification methods (TEE, ZK, optimistic) per service tier
4. **Provider reputation:** On-chain QoS history that compounds over time
5. **SDK simplicity:** dRPC's success shows that developer experience matters more than decentralization ideology

---

## Sources

### Decentralized RPC
- [Pocket Network Shannon Upgrade](https://pocket.network/shannon-launch-retro/)
- [Pocket Network PNF Q4 2025 Transparency Report](https://forum.pokt.network/t/pnf-q4-2025-january-2026-transparency-report/5626)
- [POKT Token Economics](https://pocket.network/pokt-token/)
- [PATH Gateway Framework](https://pocket.network/path-gateway/)
- [Pocket Network Developer Guide](https://pocket.network/pocket-developer-guide/)
- [What Is Pocket Network - BingX](https://bingx.com/en/learn/article/what-is-pocket-network-pokt-open-api-protocol)
- [Lava Network - How Lava Works](https://www.lavanet.xyz/blog/how-lava-works)
- [Lava Network Technical Architecture - DAIC Capital](https://daic.capital/blog/lava-network-technical-architecture)
- [Lava Rewards and Restaking Docs](https://docs.lavanet.xyz/rewards-restaking/)
- [Ankr Decentralized Infrastructure](https://www.ankr.com/)
- [Ankr Token Staking - DailyCoin](https://dailycoin.com/ankr-launches-ankr-token-staking-allowing-stakers-to-earn-rewards-across-all-rpc-requests-on-the-ankr-network/)
- [Ankr Review 2026](https://cryptoadventure.com/ankr-review-2026-rpc-infrastructure-liquid-staking-security-and-fit/)
- [dRPC Review 2026 - Finbold](https://finbold.com/guide/drpc-review-manage-all-your-blockchain-data-from-one-place/)
- [dRPC - Blockonomi](https://blockonomi.com/drpc/)
- [Chainstack dRPC Overview 2025](https://chainstack.com/drpc-provider-overview-2025/)
- [Infura DIN - EigenLayer AVS Launch](https://www.theblock.co/post/379022/infura-din-avs-eigenlayer)
- [Infura DIN Overview](https://www.infura.io/solutions/decentralized-infrastructure-service)
- [Infura DIN EigenLayer Expansion - The Defiant](https://thedefiant.io/news/infrastructure/consensys-infura-expands-din-to-eigenlayer)
- [Alchemy Pricing](https://www.alchemy.com/pricing)
- [Alchemy Overview 2026 - Chainstack](https://chainstack.com/alchemy-rpc-provider-overview-2026/)

### Decentralized AI/Inference
- [Ritual Network Architecture - Mitosis](https://university.mitosis.org/inside-ritual-network-the-architecture-use-cases-and-community-powering-ritualnet/)
- [Ritual x EigenLayer - EigenCloud Blog](https://blog.eigencloud.xyz/ritual-eigenlayer-ai-x-restaking/)
- [Ritual Introduction](https://ritual.net/blog/introducing-ritual)
- [Bittensor Ultimate Guide 2026](https://www.tao.media/the-ultimate-guide-to-bittensor-2026/)
- [Bittensor Ecosystem Expansion - CoinDesk](https://www.coindesk.com/business/2025/09/13/bittensor-ecosystem-surges-with-subnet-expansion-institutional-access)
- [Bittensor TAO Halving - Gate.com](https://www.gate.com/en-us/learn/articles/bittensor-tao-halving-ahead-why-decentralized-ai-may-be-the-next-big-crypto-trend-in-2026/12117)
- [Akash 2025 Year in Review](https://akash.network/blog/akash-2025-year-in-review/)
- [Akash at NVIDIA GTC 2025](https://www.nvidia.com/en-us/on-demand/session/gtc25-EXS74351/)
- [Gensyn Testnet](https://docs.gensyn.ai/testnet)
- [Gensyn AI Token - CryptoRank](https://cryptorank.io/ico/gensyn)
- [Decentralized AI Inference Markets - BlockEden](https://blockeden.xyz/blog/2025/07/28/decentralized-ai-inference-markets/)
- [Nous Research Hermes 4.3](https://nousresearch.com/introducing-hermes-4-3)
- [Hermes 4.3-36B - HuggingFace](https://huggingface.co/NousResearch/Hermes-4.3-36B)
- [io.net GPU Network](https://io.net)
- [io.net - Nansen Research](https://research.nansen.ai/articles/ionet-does-it-have-what-it-takes)
- [Render Network](https://rendernetwork.com/)
- [Render BME Model](https://know.rendernetwork.com/basics/burn-mint-equilibrium)
- [Render Statistics 2026](https://coinlaw.io/render-statistics/)

### Infrastructure
- [EigenLayer Review 2026 - CoinBureau](https://coinbureau.com/review/eigenlayer-review/)
- [EigenDA Introduction - EigenCloud Blog](https://blog.eigencloud.xyz/intro-to-eigenda-hyperscale-data-availability-for-rollups/)
- [EigenLayer Slashing Launch - CoinDesk](https://www.coindesk.com/tech/2025/04/17/eigenlayer-adds-key-slashing-feature-completing-original-vision/)
- [EigenLayer Tokenomics](https://tokenomics.com/articles/eigenlayer-tokenomics-how-eigen-captures-restaking-revenue)
- [2026 Data Availability Race - BlockEden](https://blockeden.xyz/blog/2026/02/24/modular-blockchain-wars-data-availability/)
- [Celestia DA Layer Docs](https://docs.celestia.org/learn/celestia-101/data-availability/)
- [Celestia Blob Economics - BlockEden](https://blockeden.xyz/blog/2026/01/16/celestia-blob-economics-data-availability-rollup-costs/)
- [Chainlink Economics 2.0 Staking](https://www.openpr.com/news/4442136/chainlink-link-locks-700m-tokens-in-economics-2-0-staking)
- [Top Blockchain Oracles Comparison](https://ecoinimist.com/2025/07/13/top-5-blockchain-oracles-chainlink-band-api3-pyth-and-tellor/)
- [The Graph - CoinMarketCap](https://coinmarketcap.com/cmc-ai/the-graph/what-is/)
- [The Graph 2026 Roadmap - InvestingHaven](https://investinghaven.com/crypto-blockchain/coins/5-reasons-to-buy-the-graph-grt-heading-into-2026/)

### Pain Points & Enterprise Adoption
- [2025 Ethereum RPC Latency Study](https://www.instanodes.io/blogs/2025-ethereum-rpc-latency-study-global-benchmarks-and-emerging-network-performance/)
- [Centralized vs Decentralized RPC - dRPC](https://drpc.org/blog/centralized-vs-decentralized-rpc-nodes-the-ultimate-comparison/)
- [On-Chain SLAs - ChainsCoreLabs](https://www.chainscorelabs.com/en/blog/depin-building-physical-infra-on-chain/network-performance-and-qos/the-future-of-network-slas-from-paper-promises-to-on-chain-guarantees)
- [Decentralized LLM Inference Architecture - Indium Tech](https://www.indium.tech/blog/decentralized-inference-ai/)
- [Best RPC Providers 2025 - GetBlock](https://getblock.io/blog/best-rpc-node-providers-2025-the-practical-comparison-guide/)
