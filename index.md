# Bitcoin Deposits: A Layer 3 Protocol for Trust-Minimized Lightning Wallets

Vinny Fiano<br/>
ynniv@ynniv.com<br/>
npub12akj8hpakgzk6gygf9rzlm343nulpue3pgkx8jmvyeayh86cfrus4x6fdh

## TL;DR

* given Lightning channels are practically trustless,
* and they can have any rules that both sides agree to:
* let's agree to hold tagged payments in a reserve output
* and pay invoices with them when there is a signed request.
* **NEW: vault payments are secured by 2-of-2 multisig between channel peers**
* others will monitor your channel updates in exchange for traffic.
* if you want to leave you can ask others to take your obligations,
* and if it all blows up the full reserves go to someone else in the system.

## Abstract

<span class="newthought">We present Bitcoin Deposits</span>, a Layer 3 protocol utilizing channel consensus, 2-of-2 vault multisig addressing, and full reserve requirements that enables users to maintain Lightning wallets without requiring on-chain Bitcoin transactions or channel management. This protocol introduces validated outputs—a new Lightning extension that allows commitment transactions to include outputs subject to agreed-upon validation rules, enforcing protocol compliance at the channel layer. **Vault payments are addressed using 2-of-2 multisig keys derived from both channel peers, preventing operators from selectively honoring payments.** Through cryptographic authentication, mandatory over-reserve requirements, metrics-based auditor monitoring, and deterministic validation rules, the system creates economic incentives for honest behavior while providing instant Lightning access. Integration with Nostr Wallet Connect enables users to access deposits from any compatible wallet interface. We rely on objective performance metrics and neutral party intervention for recovery scenarios, creating a trust-minimized system that enables users to match deposit sizes to operator reputation value.

## 1. Introduction

<span class="newthought">The Lightning Network has demonstrated</span> the viability of off-chain Bitcoin transactions, achieving sub-second settlement times with minimal fees. However, adoption remains constrained by the complexity of channel management and the requirement for on-chain transactions to establish payment channels. Each new Lightning user must execute at least one on-chain transaction, creating a scalability bottleneck as Bitcoin's block space is limited.

Additionally, Lightning's current architecture assumes users can maintain high availability for channel management and have reliable internet for gossip synchronization. This excludes users in developing regions with intermittent connectivity and those who need simple, phone-based payment solutions. Even wealthy users may prefer the simplicity of delegated channel management while maintaining cryptographic control of their funds.

Existing solutions to this problem involve custodial services that sacrifice Bitcoin's core property of cryptographic verifiability. Users must trust operators to maintain reserves and process withdrawals honestly. Chaumian Ecash systems like Cashu provide privacy but require users to trust mints with custody of funds without transparent reserve requirements.

We propose a new layer 3 protocol that enables Lightning wallets through validated outputs—a Lightning protocol extension that allows commitment transactions to include outputs subject to agreed-upon validation rules. Users control deposits through cryptographic keys without managing channels, while operators are constrained by protocol rules enforced at the Lightning channel level and monitored by auditors. **Critically, vault payments are addressed using 2-of-2 multisig keys derived from both channel peers, preventing operators from pocketing payments intended for vaults.** The system achieves:

- **Zero on-chain footprint** for users through Layer 3 abstraction
- **Offline receiving** where the vault creates invoices on user's behalf
- **Low-connectivity sending** without gossip sync requirements
- **Cryptographic enforcement** via validated outputs in commitment transactions
- **2-of-2 multisig addressing** preventing selective payment honoring
- **Economic sustainability** through routing fees and configurable maintenance fees
- **Progressive trust model** matching deposit size to operator reputation
- **Metrics-based reliability** through objective auditor monitoring
- **Trust-minimized operation** through full reserve requirements
- **Flexible deployment** through support for operators at all reputation levels
- **Universal wallet access** via Nostr Wallet Connect integration

## 2. Validated Outputs Extension

### 2.1 Overview

Bitcoin Deposits requires a new Lightning Network protocol extension called "validated outputs" that enables commitment transactions to include additional outputs subject to validation rules agreed upon by channel partners. This mechanism provides cryptographic enforcement of Layer 3 protocols at the Lightning consensus layer.

### 2.2 Protocol Layers

```
┌─────────────────────┐
│      Layer 3        │  Bitcoin Deposits Protocol
│  (Deposit Wallets)  │  - Vault management
│                     │  - Payment authorization
│                     │  - 2-of-2 multisig addressing
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
6. **Multisig Coordination**: Both peers must cooperate for vault payment processing

Both channel partners must validate all outputs before signing commitments, ensuring consensus on protocol rules.

### 2.4 Auditors

In the Bitcoin Deposits protocol, "auditors" serve a dual purpose:
1. **Watchtower role**: Monitor for outdated commitment transactions
2. **Monitoring role**: Validate and monitor vault operator behavior

Auditors are peers that validate commitment transactions but cannot reject them. Instead, they:
- Collect metrics on vault availability and correctness
- Report compliance scores for reputation systems
- Provide objective, verifiable performance data

Critically, auditors can be operated by various entities:
- **Corporate auditors**: Large companies with established reputations
- **Community auditors**: Bitcoin organizations and trusted community members
- **Independent auditors**: Anonymous operators building reputation over time
- **Commercial auditors**: Paid services competing on reliability and accuracy

Each deposit specifies a set of acceptable auditors, while the channel state tracks which auditors are currently participating. This creates a two-layer system: deposits express auditor preferences, and channels enforce auditor participation.

## 3. Vault Multisig Addressing

### 3.1 Overview

**The core innovation of Bitcoin Deposits is 2-of-2 multisig addressing for vault payments.** This prevents operators from selectively honoring payments by requiring cooperation from both channel peers to claim any payment intended for a vault.

### 3.2 Vault Creation and Addressing

**Vault Creation:**
A vault is created within a specific channel and identified by:
- `vault_id`: A unique identifier within the channel
- `multisig_pubkey`: A 2-of-2 multisig public key derived from both channel peers' keys
- `channel_id`: The specific channel containing the vault

**Multisig Key Derivation:**
```
vault_multisig_key = multisig(
    derive_key(node_a_key, channel_id, vault_id),
    derive_key(node_b_key, channel_id, vault_id)
)
```

**Vault Addressing:**
Vaults are addressed using the format:
```
vault_id@multisig_pubkey
```

Where:
- `vault_id` is unique within the channel
- `multisig_pubkey` is the 2-of-2 multisig key for this vault (globally unique)

### 3.3 Payment Processing

**Invoice Generation:**
When creating an invoice for vault funding:
1. Generate a standard Lightning invoice addressed to one of the channel peers
2. Include vault metadata in the invoice description or TLV extension:
   - `vault_id`
   - `multisig_pubkey`
   - `channel_id`
   - `peer_signature` (signature from the other channel peer approving this vault)

**Payment Reception and Escrow:**
When a payment is received for a vault:
1. The receiving node identifies it as a vault payment via the metadata
2. The payment is held in escrow and cannot be claimed unilaterally
3. Both channel peers must provide signatures to credit the vault
4. The vault balance is updated only after both signatures are verified

**Timeout Recovery:**
To prevent HTLCs from being stuck forever if one peer loses their key:
```
vault_claim = multisig(peer_a_key, peer_b_key) OR 
              (vault_output_address + timelock)
```

After a timeout period (typically 24-72 hours), unclaimed HTLCs automatically resolve to a predetermined vault output address controlled by a neutral party, ensuring funds still reach the intended vault system.

### 3.4 Attack Prevention

**The Selective Payment Attack:**
Without multisig addressing, an operator could:
1. Receive a payment intended for a vault
2. Route it through a non-vault-aware channel
3. Pocket the payment instead of crediting the vault
4. The vault balance doesn't increase, but the operator keeps the funds

**Multisig Solution:**
With 2-of-2 multisig addressing:
1. Payments to vaults require both channel peers to cooperate
2. The operator cannot unilaterally claim vault payments
3. Both peers must verify the payment is properly credited to the vault
4. This creates cryptographic enforcement of vault integrity

### 3.5 Message Types

#### `vault_create` (type TBD)
- `vault_id`: Unique identifier for the vault
- `multisig_pubkey`: 2-of-2 multisig public key
- `initial_balance`: Starting balance (may be 0)
- `terms`: Vault terms and conditions

#### `vault_fund_request` (type TBD)
- `vault_id`: Target vault identifier
- `amount`: Amount to fund
- `invoice`: Lightning invoice for the funding

#### `vault_fund_approve` (type TBD)
- `vault_id`: Vault being funded
- `signature`: Signature approving the vault credit

#### `vault_balance_update` (type TBD)
- `vault_id`: Vault identifier
- `new_balance`: Updated balance after funding
- `signatures`: Both peers' signatures

## 4. Validation Rules

### 4.1 Core Validation Functions

The Bitcoin Deposits validator implements validation with full deposit state visibility. Validation rules are deterministic and agreed upon by all channel participants. **Critically, both channel peers must validate that vault payments are properly credited to prevent the selective payment attack.**

### 4.2 Maintenance Fee Implementation

Maintenance fees are enforced through block-based timing mechanisms similar to HTLC timeout periods.

**Fee Application Examples:**
- **Dust cleanup**: `UnitFee=1000, Period=144` → "1000 millisats per day"
- **Percentage fees**: `RateFee=27, Period=52560` → "0.27% per year"

**Timing Mechanism:**
Fees are calculated and applied during commitment transaction creation, ensuring deterministic timing. The block-based approach provides:
- Consistent timing across all implementations
- Integration with existing Lightning timeout mechanisms
- Predictable fee schedules for users and operators
- Support for both absolute and percentage-based fee structures

## 5. Security Model

### 5.1 Trust-Minimized Operation

**Important**: This protocol is trust-minimized, not fully trustless. However, **the 2-of-2 multisig addressing significantly reduces trust requirements** by preventing the most common attack vector: selective payment honoring.

**Remaining Trust Requirements:**
- Force closing with invalid state
- Refusing to process legitimate withdrawals  
- Coordinating with malicious channel partners

**Trust Minimization Through:**
- **2-of-2 multisig addressing**: Prevents selective payment honoring
- **Full-Reserves**: Operators must maintain more than 100% of deposit balances
- **Metrics-based monitoring**: Auditors track objective performance indicators
- **Economic disincentives**: Security deposit at risk for misbehavior
- **Neutral party recovery**: Dispute resolution for force-close scenarios
- **Reputation-value matching**: Deposit sizes appropriate to operator reputation

### 5.2 Reputation-Value Matching

The key insight of Bitcoin Deposits is that **deposit sizes should match operator reputation value**:

**Anonymous Operators**:
- Can build reputation starting with minimal deposits
- As reputation grows, can attract larger deposits
- Market determines appropriate deposit sizes

**Community-Known Operators**:
- Bitcoin community members with established identities
- Can support moderate deposit sizes based on social reputation
- Bridge between anonymous and corporate operators

**Corporate Operators**:
- Multi-billion dollar companies with legal entities
- Can support substantial deposits (even life savings)
- Reputation worth far more than any potential theft

Users naturally distribute their holdings based on this hierarchy:
- Test deposits with new operators
- Moderate balances with community operators
- Substantial holdings with corporate operators
- Diversification across all levels for resilience

### 5.3 Auditor Monitoring

**Reputation System Framework:**

Auditors report objective metrics that create transparent reputation scores:
- **Uptime**: Percentage of time operator responds to requests
- **Correctness**: Valid signature rate on payment authorizations
- **Timeliness**: Response time for payment processing
- **Reserve Ratio**: Validated reserve levels vs. deposits
- **Vault Integrity**: Proper crediting of multisig vault payments

**Storage and Query:**
- Reports stored by operator public key in distributed systems
- Scores aggregated from last 90 days of activity
- Query interface allows filtering by auditor, timeframe, and metric type
- Historical data enables trend analysis and early warning systems

**Key Insight**: This is not a social reputation system—auditors report objective metrics that can be independently verified. Multiple auditors reporting similar metrics provide confidence in accuracy.

### 5.4 Recovery Mechanisms

**Cooperative Channel Close**:
1. All deposits must be transferred to other vaults or withdrawn
2. Validated output must be removed (zero balance)
3. Standard Lightning cooperative close proceeds

**Force Close Recovery**:
1. Validated output appears in commitment transaction
2. Vault funds and security deposit transfer to neutral party
3. Neutral party coordinates re-homing of deposits
4. Balances obtained via auditor records or channel counterparty

**Timeout Recovery for Stuck HTLCs:**
1. HTLCs to vault addresses have automatic timeout fallback
2. After timeout period, funds go to vault output address
3. Neutral party ensures funds reach intended vault system
4. Prevents permanent loss due to key loss or peer unavailability

**Recovery Guarantee:**
Users' funds are protected through the combination of:
- Validated output reserves (covers immediate losses)
- Multiple auditor records for balance verification
- Economic incentives for proper recovery handling
- Configuration-appropriate accountability
- **Multisig timeout recovery prevents permanent HTLC loss**

### 5.5 Configuration Flexibility

The protocol supports multiple operational configurations, each with different trust and efficiency tradeoffs:

**Standard Configuration (A+B|C)**:
- A: Vault operator manages deposits
- B: Channel partner provides infrastructure
- C: Neutral party handles recovery
- Maximum separation of concerns
- Suitable for new operators building reputation

**Consolidated Recovery (A+B|B)**:
- Channel partner serves dual role as recovery agent
- Natural synergy for Lightning Service Providers
- Existing infrastructure monetization
- Efficient for established channel relationships

**Operator Recovery (A+B|A)**:
- Operator controls recovery process
- Requires strong existing reputation
- Channel partner provides accountability
- Suitable for well-known operators

**Configuration Progression**:
New operators typically start with full separation (A+B|C) to build trust. As reputation solidifies, they may transition to more efficient configurations. The market determines which configurations users accept based on operator history and deposit sizes.

### 5.6 Vault-Initiated Transfers

Vaults can transfer deposits to other vaults for operational efficiency:

```
Transfer Process:
1. Origin vault identifies need to transfer deposits:
   - Channel capacity constraints
   - Planned channel closure
   - Liquidity rebalancing

2. Verify destination vault compatibility:
   - Supports Bitcoin Deposits protocol
   - Has acceptable auditor approval
   - Has sufficient capacity and reserves

3. Create Lightning payment to destination vault with metadata:
   - Transfer amount
   - Deposit details (eg, public key, fee structures)

4. Upon payment confirmation:
   - Destination vault adds deposit with transferred balance
   - Origin vault zeros the deposit balance
   - Both vaults update their validated outputs

5. If payment fails, no state change occurs (atomicity preserved)
```

This enables vaults to:
- Maintain optimal channel liquidity
- Gracefully shut down operations
- Balance load across the network
- Respond to capacity constraints

Users discover transfers through:
- Auditor reports of balance movements
- Wallet interfaces showing vault changes
- Continued ability to authorize payments from new vault

**User-initiated rebalancing**:
1. Open new deposit at preferred vault
2. Authorize payment from old vault to themselves
3. Receive payment at new vault deposit

### 5.7 Operational Flexibility

The protocol's design enables flexible deployment models:

**Geographic Distribution**: Vault operators can serve users globally without geographic restrictions. The Lightning Network's onion routing naturally provides payment privacy.

**Progressive Reputation Building**: New operators can enter the market and build reputation over time:
1. Start with small deposits from risk-tolerant users
2. Build metrics through consistent operation
3. Attract larger deposits as reputation solidifies
4. Eventually compete with established operators

**Infrastructure Specialization**: Different entities can focus on their strengths:
- Channel partners can focus on Lightning infrastructure
- Vault operators can focus on user experience
- Recovery agents can specialize in dispute resolution
- Natural separation allows each party to optimize their role

**User Privacy**: Users need only share:
- A public key for deposit control
- Authorization signatures for transactions
- No personal information is required by the protocol

**Connection Privacy**: Standard privacy tools enhance operational security:
- Tor support for anonymous network access
- NWC (Nostr Wallet Connect) for remote wallet management
- No requirement for direct user-operator connections

## 6. Implementation Requirements

### 6.1 Lightning Node Requirements

**Message Flow Integration:**
Validated output messages integrate with existing Lightning message flow during commitment transaction updates. Both channel partners must validate all outputs before signing new commitments, ensuring consensus on protocol rules.

**Multisig Support:**
- Key derivation for vault multisig addresses
- Cooperative signing for vault payment processing
- Timeout handling for stuck HTLCs
- Integration with existing HTLC processing logic

### 6.2 Vault Operator Requirements

- Maintain Lightning channels with sufficient capacity
- Process deposit operations and payment authorizations
- **Coordinate with channel partner for vault payment processing**
- Maintain full visibility into deposit state for validation
- Maintain reserve output with balance greater than total deposits
- Work with auditor-approved operations for deposit transfers
- Publish maintenance fee schedules
- Coordinate with neutral parties for recovery scenarios during force closes
- Build and maintain reputation through consistent operation

### 6.3 Channel Partner Requirements

- **Participate in 2-of-2 multisig vault addressing**
- **Validate and co-sign vault payment processing**
- Monitor commitment transactions for validated outputs
- Provide infrastructure support for vault operations
- Assist in recovery scenarios when configured as recovery agent
- Maintain auditor-approved operations

### 6.4 Auditor Requirements

- Monitor validated outputs in commitment transactions
- Provide soft enforcement through reputation reporting
- Report compliance scores and violations
- **Monitor vault payment processing integrity**
- Assist in recovery by providing balance records
- Maintain independence from operators they audit
- Compete on accuracy and reliability of monitoring

### 6.5 Client Requirements

- Manage single ECDSA keypair that controls one or many deposits
- Sign payment authorizations
- Track deposit states across multiple vaults
- Select acceptable auditors for each deposit
- Monitor operator reputation scores
- Implement deposit distribution strategies

**Nostr Wallet Connect Integration**:
- Enables use of any updated NWC-compatible wallet as interface
- Deposits can be accessed through web pages using browser credential management
- No app store requirements or dedicated applications needed
- Messages routed through Nostr relays for censorship resistance

## 7. Protocol Analysis

### 7.1 Universal Applicability

Bitcoin Deposits serves users across the entire economic spectrum:

**High-Value Users**:
- Simplified Lightning access without channel management
- Ability to distribute holdings across multiple operators
- Corporate operators provide institutional-grade security
- Maintain cryptographic control of funds
- **2-of-2 multisig prevents payment theft**

**Medium-Value Users**:
- Access to Lightning without on-chain transactions
- Choice of operators based on reputation/trust preferences
- Easy migration between operators as needs change
- Community operators provide good balance of trust and accessibility
- **Multisig addressing ensures payment integrity**

**Low-Value Users**:
- Entry point to Lightning Network without minimum balances
- Progressive upgrade path as holdings grow
- Anonymous operators enable permissionless access
- Distributed deposits minimize risk
- **Cryptographic guarantees even for small amounts**

### 7.2 Risk Management Through Diversification

The protocol's design naturally encourages intelligent risk management:

**Reputation-Based Distribution**:
- Allocate largest deposits to highest-reputation operators
- Use medium-reputation operators for moderate balances
- Test new operators with minimal amounts
- Maintain diversity across operator types and configurations

**Configuration Awareness**:
- Standard configurations (independent operator, channel partner, and neutral party) for maximum security
- Consolidated configurations for established relationships
- Understand the trust model of each vault
- Match deposit sizes to configuration risk

**Dynamic Rebalancing**:
- Actively move deposits based on reputation changes
- Automated strategies based on auditor metrics
- Market-driven allocation rewards good operators
- Early warning systems prevent losses

**Systemic Resilience**:
- No single point of failure for user funds
- Operator failures affect only portion of holdings
- Competition ensures alternative operators exist
- Natural selection improves operator quality over time
- **Multisig addressing prevents most common attack vectors**

### 7.3 Economic Incentive Alignment

The protocol creates sustainable economics for all participants:

**For Operators**:
- Revenue from maintenance fees and LSP services
- Reputation as valuable long-term asset
- Growing deposit base from good performance
- Market differentiation through service quality
- **Cannot selectively pocket vault payments**

**For Channel Partners**:
- Revenue sharing for infrastructure provision
- **Economic incentive to properly validate vault payments**
- Reduced risk through shared responsibility
- Natural business model for LSPs

**For Auditors**:
- Fee revenue from monitoring services
- Reputation for accurate reporting
- Network effects from wide adoption
- Competition drives monitoring innovation

**For Users**:
- Access to Lightning without technical complexity
- Choice of trust/convenience/cost tradeoffs
- Competitive fee pressure from operator competition
- Increasing security as ecosystem matures
- **Cryptographic protection against payment theft**

### 7.4 Trust Model

The protocol operates on a **progressive trust model** that adapts to user needs:

**Cryptographic guarantees**:
- Authorization required for all balance decreases
- Validation rules enforced by channel consensus
- Atomic transfers via Lightning payments
- User maintains exclusive control via private key
- **2-of-2 multisig prevents selective payment honoring**

**Economic guarantees**:
- Over-reserve requirements provide skin in the game
- Security deposits align operator incentives
- Reputation value exceeds potential theft gains
- Competition ensures good behavior

**Social guarantees**:
- Transparent metrics enable informed decisions
- Community monitoring provides oversight
- Market dynamics punish bad actors
- Success stories build ecosystem trust

### 7.5 Comparison with Existing Solutions

**Vs. Self-Custodial Lightning (Phoenix/Breez)**:
- Simpler user experience (no channel management)
- Better offline receiving capabilities
- Works with any wallet via NWC
- **Cryptographic protection via multisig addressing**
- Trade-off: requires trusting vault operators

**Vs. Custodial Services**:
- User maintains cryptographic control
- Transparent reserve requirements
- Open protocol enables competition
- Objective performance monitoring
- **Cannot selectively honor payments**

**Vs. Ecash Systems (Cashu/Fedimint)**:
- Direct Lightning integration
- No mint-specific tokens
- Simpler trust model
- Better audit capabilities
- **Stronger payment integrity guarantees**
- Trade-off: auditors and channel operators have transactional visibility

### 7.6 Deployment Strategy

Bitcoin Deposits enables organic growth through market dynamics:

**Bootstrap Phase**:
- Anonymous operators start with minimal deposits
- Early adopters test with small amounts
- Reputation metrics establish baselines
- Community discusses experiences
- **Multisig addressing provides security from day one**

**Growth Phase**:
- Successful operators attract larger deposits
- Corporate entities enter with instant credibility
- Wallet integration improves user experience
- Network effects drive adoption
- **Proven security model builds confidence**

**Maturity Phase**:
- Full spectrum of operators serves all users
- Insurance products develop for additional protection
- Regulatory frameworks adapt to reality
- Protocol becomes standard Lightning infrastructure
- **Multisig addressing becomes expected security standard**

### 7.7 Future Considerations

1. **Privacy Enhancements**: Balance transparency vs. operator needs while maintaining multisig security
2. **Cross-Implementation Support**: Ensure broad compatibility of multisig coordination
3. **Insurance Integration**: Third-party protection options for remaining trust requirements
4. **Regulatory Adaptation**: Maintain flexibility for compliance while preserving cryptographic guarantees
5. **Protocol Governance**: Upgrade mechanisms as ecosystem evolves

## 8. Conclusion

Bitcoin Deposits provides a trust-minimized path to Lightning Network access for all users, regardless of their technical capability, connectivity, or wealth. **The introduction of 2-of-2 multisig addressing for vault payments significantly strengthens the security model by preventing operators from selectively honoring payments.** By acknowledging that different users have different needs and risk tolerances, the protocol enables a market-based approach where deposit sizes naturally match operator reputation values.

The key innovation is not in preventing all possible theft but in creating transparent mechanisms for reputation building and risk assessment, **while cryptographically preventing the most common attack vector**. Users can make informed decisions about which operators to trust with what amounts, while market dynamics ensure that good operators thrive and bad operators fail.

This pragmatic approach enables:
- **Universal access** to Lightning payments without on-chain transactions
- **User sovereignty** through cryptographic control of deposits
- **Cryptographic payment integrity** via 2-of-2 multisig addressing
- **Market efficiency** through operator competition
- **Progressive adoption** as users can start small and grow
- **Systemic resilience** through diversification and choice

The protocol succeeds by providing infrastructure that meets users where they are. Whether someone has $10 or $10 million, intermittent connectivity or fiber internet, technical expertise or none at all—Bitcoin Deposits provides an appropriate solution that maintains the core principle of cryptographic verifiability while acknowledging the realities of human trust networks.

**The addition of 2-of-2 multisig addressing transforms Bitcoin Deposits from a purely reputation-based system to one with strong cryptographic guarantees for payment integrity.** This makes the protocol suitable for users across the entire economic spectrum, from small-amount testers to large-scale adopters.

Bitcoin Deposits is evolution, not revolution. It builds on Lightning's success while addressing its adoption barriers. By creating space for operators at all reputation levels to serve users with all risk tolerances, **and by providing cryptographic protection against the most common attack vectors**, we enable the Lightning Network to truly become the payment layer for the world.