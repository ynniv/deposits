# Bitcoin Deposits: A Layer 3 Protocol for Trust-Minimized Lightning Wallets

Vinny Fiano<br/>
ynniv@ynniv.com<br/>
npub12akj8hpakgzk6gygf9rzlm343nulpue3pgkx8jmvyeayh86cfrus4x6fdh

## Abstract

<span class="newthought">We present Bitcoin Deposits</span>, a Layer 3 protocol utilizing Proof of Over-Reserves that enables users to maintain Lightning wallets without requiring on-chain Bitcoin transactions or channel management. This protocol introduces validated outputs—a new Lightning extension that allows commitment transactions to include outputs subject to agreed-upon validation rules, enforcing protocol compliance at the consensus layer. Through cryptographic authentication, mandatory over-reserve requirements, metrics-based auditor monitoring, and deterministic validation rules, the system creates economic incentives for honest behavior while providing instant Lightning access. Integration with Nostr Wallet Connect enables users to access deposits from any compatible wallet interface. We rely on objective performance metrics and neutral party intervention for recovery scenarios, creating a trust-minimized (but not fully trustless) system that bridges the gap between custodial services and self-custody for users below the UTXO line.

## 1. Introduction

<span class="newthought">The Lightning Network has demonstrated</span> the viability of off-chain Bitcoin transactions, achieving sub-second settlement times with minimal fees. However, adoption remains constrained by the complexity of channel management and the requirement for on-chain transactions to establish payment channels. Each new Lightning user must execute at least one on-chain transaction, creating a scalability bottleneck as Bitcoin's block space is limited.

Critically, millions of potential Bitcoin users exist "below the UTXO line"—their total holdings are worth less than the on-chain fees required to establish self-custody. The 'UTXO line' represents the economic threshold where transaction fees exceed the practical value of creating a UTXO. When fees are $50, anyone with less than $500-1000 faces prohibitive costs.</span> When transaction fees spike to $50 or more, users with $100 in savings face an impossible choice: pay 50% of their wealth in fees or remain on custodial services. This economic reality locks out the very populations that would benefit most from Bitcoin's financial inclusion.

Additionally, Lightning's current architecture assumes users can maintain high availability for channel management and have reliable internet for gossip synchronization. This excludes users in developing regions with intermittent connectivity and those who need simple, phone-based payment solutions.

Existing solutions to this problem involve custodial services that sacrifice Bitcoin's core property of trustlessness. Users must trust operators to maintain reserves and process withdrawals honestly. Chaumian Ecash systems like Cashu provide privacy but require users to trust mints with custody of funds.

We propose a new layer 3 protocol that enables Lightning wallets through validated outputs—a Lightning protocol extension that allows commitment transactions to include outputs subject to agreed-upon validation rules. Users control deposits through cryptographic keys without managing channels, while operators are constrained by protocol rules enforced at the Lightning channel level and monitored by auditors. The system achieves:

- **Zero on-chain footprint** for users through Layer 3 abstraction
- **Offline receiving** where the vault creates invoices on user's behalf
- **Low-connectivity sending** without gossip sync requirements
- **Cryptographic enforcement** via validated outputs in commitment transactions
- **Economic sustainability** through LSP fees and configurable maintenance fees
- **Progressive trust model** from corporate-audited vaults to full self-custody
- **Metrics-based reliability** through objective auditor monitoring
- **Trust-minimized operation** through 120% reserve requirements
- **Flexible deployment** through support for pseudonymous operators and users
- **Universal wallet access** via Nostr Wallet Connect integration

## 2. Validated Outputs Extension

### 2.1 Overview

Bitcoin Deposits requires a new Lightning Network protocol extension called "validated outputs" that enables commitment transactions to include additional outputs subject to validation rules agreed upon by channel partners. This mechanism provides cryptographic enforcement of Layer 3 protocols at the Lightning consensus layer.

### 2.2 Protocol Layers

```
┌─────────────────────┐
│      Layer 3        │  Bitcoin Deposits Protocol
│  (Deposit Wallets)  │  - Deposit management
│                     │  - Payment authorization
├─────────────────────┤
│      Layer 2        │  Lightning Network + Validated Outputs
│ (Payment Channels)  │  - Validation framework
│                     │  - Commitment transactions
├─────────────────────┤
│      Layer 1        │  Bitcoin Blockchain
│    (Settlement)     │  - Proof of Work
│                     │  - UTXO model
└─────────────────────┘
```

### 2.3 Protocol Integration

The validated outputs extension requires:

1. **Feature Negotiation**: New feature bit `option_validated_outputs`
2. **Channel Messages**: Extended to support validator negotiation
3. **Update Protocol**: New message types for validated output operations
4. **Commitment Structure**: Validated outputs included deterministically
5. **Auditor Requirements**: Deposit auditors must approve of vault operators

Both channel partners must validate all outputs before signing commitments, ensuring consensus on protocol rules.

### 2.4 Auditors

In the Bitcoin Deposits protocol, "auditors" serve a dual purpose:
1. **Watchtower role**: Monitor for outdated commitment transactions
2. **Monitoring role**: Validate and monitor vault operator behavior

Auditors are peers that validate commitment transactions but cannot reject them. Instead, they:
- Collect metrics on vault availability and correctness
- Report compliance scores for reputation systems

Critically, auditors can be operated by various entities:
- **Corporate auditors**: Companies like Block, Coinbase, or Strike with established reputations
- **Community auditors**: Bitcoin organizations and trusted community members
- **Independent auditors**: Anonymous operators building reputation over time
- **Commercial auditors**: Paid services competing on reliability and accuracy

Each deposit specifies a set of acceptable auditors, while the channel state tracks which auditors are currently participating. This creates a two-layer system: deposits express auditor preferences, and channels enforce auditor participation.

## 3. Validation Rules

### 3.1 Core Validation Functions

The Bitcoin Deposits validator implements validation with full deposit state visibility

### 3.2 Maintenance Fee Implementation

Maintenance fees are enforced through block-based timing mechanisms similar to HTLC timeout periods

**Fee Application Examples:**
- **Dust cleanup**: `UnitFee=1000, Period=144` → "1000 millisats per day"
- **Percentage fees**: `RateFee=27, Period=52560` → "0.27% per year"

**Timing Mechanism:**
Fees are calculated and applied during commitment transaction creation, ensuring deterministic timing. The block-based approach provides:
- Consistent timing across all implementations
- Integration with existing Lightning timeout mechanisms
- Predictable fee schedules for users and operators
- Support for both absolute and percentage-based fee structures

## 4. Security Model

### 4.1 Trust-Minimized Operation

**Important**: This protocol is trust-minimized, but not fully trustless. Operators retain the ability to steal funds through various mechanisms:
- Force closing with invalid state
- Refusing to process legitimate withdrawals
- Coordinating with malicious channel partners

The protocol minimizes trust requirements through:
- **Proof of Over-Reserves**: Operators must maintain 120% of deposit balances
- **Metrics-based monitoring**: Auditors track objective performance indicators
- **Economic disincentives**: Security deposit at risk for misbehavior
- **Neutral party recovery**: Dispute resolution for force-close scenarios

The system is designed for amounts where these protections provide adequate security—typically $5-200 per deposit. Users can further reduce risk by distributing funds across multiple vaults.

### 4.2 Auditor Monitoring

**Reputation System Framework:**

**Storage and Query:**
- Reports stored by operator public key in distributed systems
- Scores aggregated from last 90 days of activity
- Query interface allows filtering by auditor, timeframe, and violation type
- Historical data enables trend analysis and early warning systems

**Key Insight**: This is not a social reputation system—auditors report objective metrics that can be independently verified. Users don't need to interpret complex scores; wallets can provide simple interfaces showing vault health.

### 4.3 Recovery Mechanisms

**Cooperative Channel Close**:
1. All deposits must be transferred to other vaults or withdrawn
2. Validated output must be removed (zero balance)
3. Standard Lightning cooperative close proceeds

**Force Close Recovery**:
1. Validated output appears in commitment transaction
2. Vault funds and security deposit transfer to neutral party
3. Neutral party coordinates re-homing of deposits
4. Balances obtained via auditor records or channel counterparty

**Recovery Guarantee:**
Users' funds are protected through the combination of:
- Operator security deposits (covers immediate losses)
- Neutral party bonds (covers recovery process failures)
- Community auditing

**Bootstrap Strategy**: Initial neutral parties may include:
- Lightning service providers seeking ecosystem growth
- Exchanges offering value-added services
- Payment processors with existing infrastructure
- Community members with vested interest in Bitcoin adoption
- Auditors expanding their service offerings (with appropriate separation of concerns)

### 4.4 Atomic Vault Transfers

Deposits transfer between vaults using atomic Lightning payments:

```
Transfer Process:
1. Verify destination vault meets requirements:
   - Supports Bitcoin Deposits protocol
   - Has acceptable auditor approval
   - Has sufficient capacity and reserves

2. Create Lightning payment to destination vault with metadata:
   - Transfer amount
   - Deposit public key
   - Authorization signature

3. Upon payment confirmation:
   - Destination vault adds deposit with transferred balance
   - Origin vault zeros the deposit balance
   - Both vaults update their validated outputs

4. If payment fails, no state change occurs (atomicity preserved)
```

### 4.5 Operational Flexibility

The protocol's design enables flexible deployment models:

**Geographic Distribution**: Vault operators can serve users globally without geographic restrictions. The Lightning Network's onion routing naturally provides payment privacy.

**Pseudonymous Operation**: Both vault operators and auditors can build reputation using persistent cryptographic identities rather than real-world identities. This allows new entrants to establish trust through consistent good behavior.

**User Privacy**: Users need only share:
- A public key for deposit control
- Authorization signatures for transactions
- No personal information is required by the protocol

**Connection Privacy**: Standard privacy tools enhance operational security:
- Tor support for anonymous network access
- NWC (Nostr Wallet Connect) for remote wallet management
- No requirement for direct user-operator connections

## 5. Implementation Requirements

### 5.1 Lightning Node Requirements

**Message Flow Integration:**
Validated output messages integrate with existing Lightning message flow during commitment transaction updates. Both channel partners must validate all outputs before signing new commitments, ensuring consensus on protocol rules.

### 5.2 Vault Operator Requirements

- Maintain Lightning channels with sufficient capacity
- Process deposit operations and payment authorizations
- Maintain full visibility into deposit state for validation
- Maintain security reserves >= 20% of total deposits
- Work with auditor-approved operations for deposit transfers
- Publish maintenance fee schedules
- Coordinate with neutral parties for recovery scenarios during force closes

### 5.3 Auditor Requirements

- Monitor validated outputs in commitment transactions
- Provide soft enforcement through reputation reporting
- Report compliance scores and violations
- Assist in recovery by providing balance records
- May operate pseudonymously with reputation-based identity

### 5.4 Client Requirements

- Manage single ECDSA keypair that controls one or many deposits
- Sign payment authorizations
- Track deposit states across multiple vaults
- Select acceptable auditors for each deposit
- Optionally proactively redistribute deposits

**Nostr Wallet Connect Integration**:
- Enables use of any updated NWC-compatible wallet as interface
- Deposits can be accessed through web pages using browser credential management
- No app store requirements or dedicated applications needed
- Messages routed through Nostr relays for censorship resistance
- Future support for Nostr Wallet Authorization for enhanced security

## 6. Protocol Analysis

### 6.1 Target Use Cases

Bitcoin Deposits specifically targets users and scenarios poorly served by existing solutions:

**Below UTXO Line Users**:
- Holdings worth $10-$1000 when on-chain fees are $50+
- Would pay 5-500% of their wealth in fees for self-custody
- Currently forced to use fully custodial exchanges
- Benefit from cryptographic guarantees without on-chain costs

**Low Availability Users**:
- Intermittent internet connectivity
- Cannot maintain 24/7 Lightning nodes
- Need to receive payments while offline (vault handles invoice generation)
- Developing world users with basic phones

**Specific Applications**:
- Coffee shop spending wallets ($50-$200 balance)
- Remittance receivers (periodic small payments)
- Content creator tips (many small payments)
- Point-of-sale systems (simple key management)

### 6.2 Risk Management Through Diversification

Users can significantly reduce trust requirements through smart deposit management:

**Multiple Small Deposits**:
- A single wallet (private key) can control many deposits across different vaults
- Distribute $100 as 10 × $10 deposits across different operators
- If one vault fails, maximum loss is limited to $10
- Similar to not keeping all cash in one physical location

**Dynamic Rebalancing**:
- Actively move deposits to vaults with better reputations
- Shift funds away from vaults showing warning signs
- Automate transfers based on auditor scores
- Market dynamics reward good operators with more deposits

**Availability Benefits**:
- Multiple vaults mean redundancy for payments
- If one vault is offline, others remain accessible
- Geographic distribution for global availability
- No single point of failure for the wallet

This approach transforms the trust model: instead of trusting one operator with $100, trust ten operators with $10 each—significantly reducing the incentive for theft while improving availability.

### 6.3 Progressive Trust Model

The protocol enables natural progression as users' needs evolve:

```
Trust                               Complexity
Level                               Level
  |                                 |
  |   Raw Lightning                 | High
  |   (Full control)                |
  |                                 |
  |   Phoenix/Breez                 |
  |   (Self-custody)                |
  |                                 |
  |   Community Vault               |
  |   ($100s, community auditors)   |
  |                                 |
  |   Corp-Audited Vault            |
  |   ($100s, corporate auditor)    |
  |                                 |
  |   Exchange Balance              | Low
  |   (Full Custody, zero control)  |
  |                                 |
  v                                 v
```

Users can move between models based on:
- Balance size relative to on-chain fees
- Technical sophistication
- Trust requirements
- Availability requirements

### 6.4 Trust Model

The protocol operates on a **trust-minimized** model appropriate for its use cases:

**For coffee-money amounts ($5-$200)**:
- Corporate auditor reputation provides sufficient security
- Risk of loss acceptable relative to convenience gains
- Further minimized through deposit distribution across vaults
- Similar trust model to transit cards or payment apps

**Cryptographic guarantees**:
- Authorization required for balance changes
- Validation rules enforced by channel consensus
- Atomic transfers via Lightning payments
- User controls all deposits with single private key

**Economic and social guarantees**:
- Security deposits create theft disincentives
- Corporate auditors risk billion-dollar reputations
- Market competition among vault operators
- Small deposit sizes reduce theft incentive

### 6.5 Privacy Implications

Current privacy characteristics support flexible deployment:

**What operators see**:
- Deposit balances
- Authorization signatures
- Activity patterns

**What remains private**:
- User identity
- Geographic location (via Tor/VPN usage)
- Payment destinations (via onion routing)
- Connection metadata (via privacy tools)

**Deployment flexibility**:
- Operators can choose their level of identity disclosure
- Users can interact entirely pseudonymously
- Auditors can build reputation without revealing identity
- The protocol remains neutral on identity requirements

### 6.6 Comparison with Existing Solutions

**Vs. Phoenix/Breez (Self-Custodial Lightning)**:
- Lower barrier to entry (no minimum balance requirements)
- Better offline receiving capabilities
- Simpler mental model for new users
- Flexible deployment without app store dependencies
- Trade-off: trust-minimized rather than trustless

**Vs. Custodial Services**:
- User maintains cryptographic control
- Distributed risk through multiple vaults
- Transparent security deposits
- Auditor oversight and reputation systems
- No KYC requirements at protocol level

### 6.7 Deployment Strategy

Bitcoin Deposits is designed for progressive deployment, starting with low-risk use cases and expanding as the ecosystem matures:

**Phase 1: Nostr Zaps (Months 1-6)**
- Small amounts ($0.10-$10) minimize risk
- Technical users comfortable with experimental protocols
- Existing Lightning integration provides natural bridge
- Viral growth through social tipping
- Metrics-based reputation builds organically

**Phase 2: Remittances & Tips (Months 6-12)**
- Expand to $50-$200 use cases
- Focus on users with intermittent connectivity
- Corporate auditors provide trust anchors
- Multiple vault operators ensure availability
- Wallet interfaces simplify user experience

**Phase 3: General Purpose (Year 2+)**
- Broader Lightning integration
- Privacy enhancements for sensitive use cases
- Cross-implementation support (LDK with splicing)
- Regulatory clarity through operational history
- Insurance products for additional protection

**Success Metrics**:
- Number of active vaults and geographic distribution
- Total value secured across the network
- User retention and organic growth
- Successful recovery events handled
- Uptime and performance metrics from auditors

### 6.8 Open Questions and Future Work

1. **Reputation System**: Concrete implementation needed
2. **Neutral Party System**: Selection and incentive mechanisms
3. **Maintenance Fee Timing**: When and how fees are applied
4. **Economic Analysis**: Fee structures and viability
5. **Privacy Enhancements**: Potential cryptographic solutions
7. **Version Migration**: Upgrade mechanisms for the protocol

## 7. Conclusion

Bitcoin Deposits addresses a critical gap in Bitcoin's infrastructure: serving users below the UTXO line and those with limited connectivity. By providing a trust-minimized alternative to fully custodial services, the protocol enables millions of potential users to access Lightning payments without the burden of channel management or on-chain fees.

The key insight is matching security models to use cases. For coffee-money amounts, Proof of Over-Reserves with metrics-based auditing provides an acceptable risk/reward tradeoff—similar to how users trust payment apps or transit cards with small balances. This pragmatic approach enables:

- **Financial inclusion** for users priced out by on-chain fees
- **Offline receiving** for users with intermittent connectivity
- **Simple onboarding** with just a private key
- **Progressive sovereignty** as users can upgrade to self-custodial solutions when ready
- **Flexible deployment** through support for pseudonymous operation
- **Universal access** via Nostr Wallet Connect integration

The protocol succeeds not by replacing self-custodial Lightning but by providing a bridge for users who would otherwise remain on centralized exchanges. Starting with Nostr zaps demonstrates the model with minimal risk, while the flexible deployment model allows operators to serve users globally. The metrics-based trust system enables participants to establish credibility through consistent good behavior rather than regulatory compliance.

Bitcoin Deposits represents a practical step toward the Lightning Network's goal of bringing Bitcoin payments to billions of users. By acknowledging that different users have different needs, and that perfect trustlessness isn't required for every use case, we can build systems that serve real users today while maintaining Bitcoin's core values of cryptographic verifiability and user sovereignty.

Future development will focus on implementation across Lightning nodes, privacy enhancements, and ecosystem growth. The protocol is designed to evolve as the Lightning Network matures, always providing users with the best tradeoff between security and usability for their specific needs.
