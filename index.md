# Bitcoin Deposits: A Layer 3 Protocol for Zero UTXO Trust-Minimized Lightning Wallets

Vinny Fiano<br/>
ynniv@ynniv.com<br/>
npub12akj8hpakgzk6gygf9rzlm343nulpue3pgkx8jmvyeayh86cfrus4x6fdh

## TL;DR

* we leverage a Lightning channel's uanimous consensus to implement a ledger of deposits
* with full reserves and security deposits held in a dedicated transaction output.
* joint-signatures are used to create vault addresses and sign attestations
* invoices are created by the operator, then jointly signed
* a high-water mark security deposit is maintained until invoices time out or are paid
* when the invoice is paid, the operator credits the deposit
* if they don't, evidence of the paid invoice requires honest parties to unilaterally close the channel
* deposits are used to pay invoices when a request is signed by the depositor.
* operators should run balanced vaults across different partners
* partners that cross-audit their other vault channel updates.
* if someone wants to leave they can ask others to service their deposits
* but unilateral close sends reserves and security deposits to a multisig of channel partners.
* if the operator was honest and requests to continue, it is fully returned to them
* if they were provably dishonest, channel partners unilaterally close their vault channels.
* security deposits from all channels are used to re-create stolen reserves
* and randomly selected partners re-establish services for each of the closed vaults.
* is it trustless? not quite. but it's close, and it scales

## 1. Introduction

<span class="newthought">The Lightning Network has demonstrated</span> the viability of off-chain Bitcoin transactions, achieving sub-second settlement times with minimal fees. However, adoption remains constrained by the complexity of channel management and the requirement for on-chain transactions to establish payment channels. Each new Lightning participant must execute at least one on-chain transaction, creating a scalability bottleneck as Bitcoin's block space is limited.

Additionally, Lightning's current architecture assumes participants can maintain high availability for channel management and have reliable internet for gossip synchronization. This excludes those in developing regions with intermittent connectivity and those who need simple, phone-based payment solutions. Even wealthy participants may prefer the simplicity of delegated channel management while maintaining cryptographic control of their funds.

Existing solutions to this problem involve custodial services that sacrifice Bitcoin's core property of cryptographic verifiability. Depositors must trust operators to maintain reserves and process withdrawals honestly. Chaumian Ecash systems provide privacy but require participants to trust mints with custody of funds without transparent reserve requirements.

We propose a new layer 3 protocol that enables Lightning wallets through validated outputs: a Lightning protocol extension that describes new commitment transaction outputs which are subject to negotiated validation rules. Depositors will control funds through cryptographic keys without managing channels, while operators are constrained by protocol rules enforced at the Lightning channel level and monitored by other channel partners. Vaults are addressed using a multisig between channel partners. Security deposits and penalties when presented with pre-images for signed invoices that were never credited prevent operators from pocketing payments intended for vaults. Consensus rules allow operators to establish recurring maintenance fees, making profitability commonly achievable.

The system achieves:

- **Zero on-chain footprint** for depositors through Layer 3 abstraction
- **Offline receiving** where the vault creates invoices on depositor's behalf
- **Low-connectivity sending** without gossip sync requirements
- **Cryptographic enforcement** via validated outputs in commitment transactions
- **Multisig attestation** providing proof that both parties are involved
- **Penalties for verifiable theft** making dishonesty unprofitable
- **Organic trust** through multi-vault cross-auditing networks
- **Economic sustainability** through routing fees and configurable maintenance fees
- **Progressive trust model** matching deposit size to operator reputation
- **Trust-minimized operation** through full reserve requirements
- **Flexible deployment** through support for operators at all reputation levels
- **Universal wallet access** via Nostr Wallet Connect integration

### 1.1 Design Notes

- **One vault address per operator**: Each vault services many deposits, scaling with operators not depositors
- **Maintenance fees fundamentally change node economics**: Predictable revenue from idle deposits, not just routing fees
- **Theft of funds requires active collusion**: Both operator and channel partner must modify software and coordinate
- **Theft of payments is defended**: Proof of uncredited payment reveals operator dishonesty
- **Collusion is self-defeating**: Channel partners profit more by revealing dishonesty and claiming security deposits
- **Market adoption over protocol adoption**: A small number of well-capitalized operators can service millions of deposits

## 2. Validated Outputs Extension

### 2.1 Overview

Bitcoin Deposits uses a new Lightning Network protocol extension called Validated Outputs that manages additional channel outputs which are subject to negotiated validation rules. This mechanism provides cryptographic enforcement of Layer 3 protocols at the Lightning consensus layer.

### 2.2 Protocol Layers

```
┌─────────────────────┐
│      Layer 3        │  Bitcoin Deposits Protocol
│  (Deposit Wallets)  │  - Private key controlled deposits
│                     │  - Multisig attestation
│                     │  - Security deposits and recovery protocol
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
6. **Multisig Coordination**: Both peers must cooperate for vault payment processing

Both channel partners must validate outputs before signing commitments, ensuring consensus on protocol rules

### 2.4 Cross-Vault Auditing Network

In the Bitcoin Deposits protocol, auditing is handled through a natural network of cross-vault relationships rather than dedicated auditor entities. Operators should spread their vault operations across 4-6 different channel partners, and aim for each vault to be audited by the channel partners from the their other vaults.

**Cross-Auditing Structure:**
- Operators establish multiple vaults with different channel partners
- Each channel partner serves as an auditor for the operator's other vaults
- Creates a web of mutual accountability and monitoring
- Natural geographic and operational diversity

**Auditing Functions:**
- Monitor vault payment processing integrity across the network
- Validate proper crediting of vault payments
- Confirm reserves and operational metrics
- Report compliance issues and performance data
- Force close controled channels when invalid operations reveal operator to be dishonest

**Network Benefits:**
- Leverages existing vault operator relationships
- Creates symmetric accountability between operators
- Natural business incentives for accurate monitoring
- Scales organically with network growth

## 3. Deposits

### 3.1 Overview

**A core mechanic of Bitcoin Deposits is multisig invoice attestation** This prevents operators from acting without oversight. If payments are not credited, evidence of payment results in loss of security deposits.

### 3.2 Vault Creation and Addressing

**Vault Creation:**
A vault is created using a MuSig2 aggregated public key derived from both channel peers:
```
vault_address = MuSig2.KeyAgg(
    derive_key(operator_key, vault_id),
    derive_key(partner_key, vault_id)
)
```

## 3.3 Payment Processing

**Invoice Generation:**
The vault operator creates an invoice with metadata identifying the target deposit. The channel partner verifies there are security deposits exceeding the high-water mark of recent invoice amounts. If the invoice is well formed and properly secured, both parties use MuSig2 to attest validity of the invoice.

**Payment Receptino**
The operator chooses to place payment balances on the correct deposit. If they choose to keep the funds, it is up to the depositor to reveal proof of payment.

**Preventing Payment Theft**
When a signed invoice has been paid, the payer will know the payment secret. They can share this secret with the depositor, who can broadcast both the invoice and the secret to all nodes associated with the deposit. An honest channel partner, seeing proof of payment without a corresponding credit, is required to force close the channel. The high-water payment security funds are used to make the depositor whole.

## 4. Validation Rules

### 4.1 Core Validation Functions

The Bitcoin Deposits validator implements validation with full deposit state visibility. Validation rules are deterministic and agreed upon by all channel participants.

### 4.2 Maintenance Fee Implementation

Maintenance fees are enforced through block-based timing mechanisms similar to HTLC timeout periods.

**Fee Application Examples:**
- **Dust cleanup**: `UnitFee=1000, Period=144` → "1000 millisats per day"
- **Percentage fees**: `RateFee=27, Period=52560` → "0.27% per year"

**Timing Mechanism:**
Fees are calculated and applied during commitment transaction creation, ensuring deterministic timing. The block-based approach provides:
- Consistent timing across all implementations
- Integration with existing Lightning timeout mechanisms
- Predictable fee schedules for depositors and operators
- Support for both absolute and percentage-based fee structures

## 5. Security Model

### 5.1 Trust-Minimized Operation

This protocol is trust-minimized, not fully trustless. However, the MuSig2 addressing and multi-vault architecture significantly reduce trust requirements by preventing the most common attack vectors.

**Primary Attack Vectors:**
1. **Operator-Channel Partner Collusion**: Both parties cooperate to steal vault funds
1. **Operator Payment Pocketing**: The operator receives a payment but doesn't credit the deposit
2. **Recovery Party Failure**: Recovery party doesn't properly reintegrate deposits after force close
3. **Fake Invoice Generation**: Operator creates invalid invoices to steal individual payments

**Trust Minimization Through:**
- **Multisig addressing**: Prevents most unilateral actions
- **Payment security deposits**: Makes evidence of payment theft expensive
- **Multi-vault cross-auditing**: Creates mutual accountability between operators
- **Full-Reserves**: Operators must maintain more than 100% of deposit balances
- **Metrics-based monitoring**: Auditors track objective performance indicators
- **Economic disincentives**: Security deposit at risk for misbehavior
- **Neutral party recovery**: Dispute resolution for force-close scenarios
- **Reputation-value matching**: Deposit sizes appropriate to operator reputation

**Organic Trust:**
The protocol creates organic trust through structural incentives rather than pure reputation:
- Operators should maintain multiple vaults with different channel partners
- Channel partners should also serve as auditors for other vaults
- Balanced vault sizes ensure symmetric risk exposure
- Security deposit forfeiture exceeds potential theft gains
- Network effects punish dishonesty across all relationships

### 5.2 Multi-Vault Architecture for Collusion Prevention

**Recommended Vault Network Structure:**
Operators should maintain 4-6 vaults with different channel partners, creating a robust cross-auditing network:

```
Operator A with Channel Partners B, C, D, E, F
├── Vault A-B (audited by partners C, D, E, F)
├── Vault A-C (audited by partners B, D, E, F)
├─ [...]
```

**Security Deposit Alignment:**
Each operator's security deposit per vault equals:
```
Security Deposit = 100% of vault value ÷ number of other vaults
```

For 5 total vaults: Security deposit = 25% of vault value
For 6 total vaults: Security deposit = 20% of vault value

**Cross-Auditing Network Effect:**
This structure creates powerful economic incentives against collusion:
- Channel partners monitor each other's vault operations
- Theft from one vault triggers force closes of other vaults
- Total security deposit forfeit approximates stolen vault balance
- Recovery handled by existing channel partner network
- No net gain for colluding parties

**Economic Balance:**
- Multiple vault failures drain security deposits quickly
- Recovery operators (other channel partners) gain deposits from failures
- Reputation damage affects all vault relationships
- Long-term revenue exceeds short-term theft opportunities

### 5.3 Attack Vector Analysis

**1. Operator-Channel Partner Collusion**
- **Attack**: Both parties cooperate to steal vault funds
- **Prevention**: Multi-vault requirements with cross-auditing create mutual accountability
- **Detection**: Other channel partners serving as auditors observe the theft
- **Consequence**: Force closure of other vaults, security deposit forfeiture
- **Result**: Economic incentives strongly discourage collusion

**2. Operator Payment Thft**
- **Attack**: Operator receives payment funds but doesn't credit the deposit
- **Prevention**: Payment secret gives payer proof of payment
- **Detection**: Depositor reveals proof of payment to channel partner
- **Consequence**: Force closure of vault, security deposit forfeiture
- **Result**: Economic incentives strongly discourages payment theft

**3. Recovery Party Failure**
- **Attack**: Recovery party doesn't properly reintegrate deposits
- **Status**: Open design item requiring further specification
- **Potential Mitigations**: Recovery party bonds, multi-party recovery, or auditor-based recovery

**4. Fake Invoice Generation**
- **Attack**: Operator creates invalid invoices to redirect payments
- **Detection**: Immediately visible to validating clients
- **Scope**: Limited to single payment, not entire vault
- **Prevention**: Client-side invoice validation
- **Consequence**: Immediate reputation destruction, future deposits unlikely
- **Result**: Minimal gain for massive reputation loss

### 5.4 Reputation-Value Matching

The key insight of Bitcoin Deposits is that **deposit sizes should match operator reputation value**, enhanced by structural security:

**Anonymous Operators**:
- Can build reputation starting with minimal deposits
- Should establish multi-vault architecture early
- As reputation grows, can attract larger deposits
- Market determines appropriate deposit sizes

**Community-Known Operators**:
- Bitcoin community members with established identities
- Expected to maintain professional multi-vault setups
- Can support moderate deposit sizes based on social reputation
- Bridge between anonymous and corporate operators

**Corporate Operators**:
- Multi-billion dollar companies with legal entities
- Natural multi-vault architecture through business relationships
- Can support substantial deposits (even life savings)
- Reputation worth far more than any potential theft

Depositors naturally distribute their holdings based on this hierarchy:
- Test deposits with new operators
- Moderate balances with community operators
- Substantial holdings with corporate operators
- Preference for operators with robust multi-vault architectures
- Diversification across all levels for resilience

### 5.5 Cross-Vault Monitoring

Auditing peers report objective metrics about vaults:
- **Uptime**: Vault response rates across the network
- **Payment Integrity**: Proper payment processing
- **Reserve Compliance**: Verified reserve levels vs. deposits
- **Response Times**: Payment authorization latency
- **Cross-Vault Health**: Network connectivity and balance
- **Operational Consistency**: Performance across all vaults

**Reporting Structure:**
- Metrics stored by operator public key
- Aggregated across all channel partners in the network
- Historical trends enable early warning systems
- Cross-validation between multiple reporting partners
- Transparent scoring for depositor decision-making

**Network Transparency:**
- Depositors can assess operator vault network health
- Market rewards operators with robust cross-auditing
- Poor performers isolated through reduced partnerships
- Natural selection improves overall network quality

### 5.6 Recovery Mechanisms

**Cooperative Channel Close**:
1. All deposits must be transferred to other vaults or withdrawn
2. Validated output must be removed (zero balance)
3. Standard Lightning cooperative close proceeds

**Force Close Recovery Process**:

When a force close occurs, the validated output transfers vault funds and security deposits to a simple multisig controlled by the operator's channel partners from their other vaults. This initiates a structured recovery process that distinguishes between honest operational failures and dishonest behavior.

**Stage 1: Initial Force Close Output**
- Vault funds and security deposits go to threshold multisig of operator's other channel partners
- Simple spending condition: 3-of-N signatures from channel partner network
- Partners receive investigative custody to assess situation

**Stage 2: Protocol Compliance Verification**
Channel partners verify force close circumstances through objective audit data:

*Verification Activities:*
- Compare received channel updates against protocol rules
- Validate that deposit balances match received payments
- Check reserve requirements and security deposit compliance
- Verify payment authorizations and balance changes
- Cross-reference audit records from operator's other vaults

*Network Coordination:*
- If protocol violations detected: Partners force close operator's other vault channels
- Security deposits from all channels become available for recovery
- If all updates valid: Prepare for fund return to compliant operator

**Stage 3: Resolution Based on Protocol Compliance**

*Compliant Operator Path:*
- Partners return full vault funds and security deposits to operator
- Operator can resume operations or gracefully wind down
- Maintains vault network relationships
- Handles legitimate technical issues or external coordination problems

*Non-Compliant Operator Path:*
- Proceed to deterministic reassignment using cryptographic selection
- Security deposits from all closed vaults fund recovery process
- Channel partners who discovered violations are eligible recovery candidates
- Cryptographic evidence of violations removes subjectivity from process

**Stage 4: Deterministic Assignment (Non-Compliant Case)**

If operator violated protocol rules, a commit-reveal process selects new operators:

*Commit-Reveal Selection:*
1. **Commit Phase**: Each recovery participant specifies themselves, a substitute, or abstention
2. **Reveal Phase**: Partners reveal selections after all commits received
3. **Entropy Combination**: Use revealed selections + future block hash for final randomness
4. **Assignment**: Deterministically select recovery operator(s) from qualified candidate pool
5. **Fund Transfer**: Multisig releases funds to selected operator(s)

*Time-Based Degradation for Assignment:*
- **Week 1**: Selected recovery operator(s) can claim funds
- **Week 2**: Any 3 qualified operators can claim together
- **Week 3**: Any single qualified operator can claim

**Recovery Operator Expectations:**
- Recreate deposit accounts with balances verified from audit records
- Honor existing payment authorizations
- Broadcast updates to auditors during transition period
- Re-establish reserve requirements

**Key Properties:**
- **Just**: Honest operators aren't punished for technical failures
- **Secure**: Non-compliant operators face network-wide consequences
- **Objective**: Protocol compliance verification, not subjective judgment
- **Pseudorandom Fallback**: Deterministic assignment when coordination fails
- **Fund Safety**: Multiple degradation paths prevent permanent loss

### 5.7 Vault-Initiated Transfers

Vaults can transfer deposits to other vaults for operational efficiency:

```
Transfer Process:
1. Origin vault identifies need to transfer deposits:
   - Channel capacity constraints
   - Planned channel closure
   - Liquidity rebalancing

2. Verify destination vault compatibility:
   - Supports Bitcoin Deposits protocol
   - Has acceptable reputation
   - Has sufficient capacity and reserves

3. Create keysend payment to destination vault with metadata:
   - Deposit details (public key, current fee structures, connection details)

4. Upon payment confirmation:
   - Destination vault adds deposit with transferred balance
   - Origin vault zeros the deposit balance
   - Both vaults update their validated outputs

5. If payment fails, no state change occurs
```

This enables vaults to:
- Maintain optimal channel liquidity
- Gracefully shut down operations
- Balance load across the network
- Respond to capacity constraints

Depositors discover transfers through:
- Auditor reports of balance movements
- Wallet interfaces showing vault changes
- Continued ability to authorize payments from new vault

**Depositor-initiated rebalancing**:
1. Open new deposit at preferred vault
2. Authorize payment from old vault to themselves
3. Receive payment at new vault deposit

### 5.8 Operational Flexibility

The protocol's design enables flexible deployment models:

**Geographic Distribution**: Vault operators can serve depositors globally without geographic restrictions. The Lightning Network's onion routing naturally provides payment privacy.

**Progressive Reputation Building**: New operators can enter the market and build reputation over time:
1. Start with small deposits from risk-tolerant depositors
2. Establish multi-vault architecture early
3. Build metrics through consistent operation
4. Attract larger deposits as reputation solidifies
5. Eventually compete with established operators

**Infrastructure Specialization**: Different entities can focus on their strengths:
- Channel partners can focus on Lightning infrastructure
- Vault operators can focus on depositor experience
- Recovery agents can specialize in dispute resolution
- Natural separation allows each party to optimize their role

**Depositor Privacy**: Depositors need only share:
- A public key for deposit control
- Authorization signatures for transactions
- No personal information is required by the protocol

**Connection Privacy**: Standard privacy tools enhance operational security:
- Tor support for anonymous network access
- NWC (Nostr Wallet Connect) for remote wallet management
- No requirement for direct depositor-operator connections

## 6. Implementation Requirements

### 6.1 Lightning Node Requirements

**Message Flow Integration:**
Validated output messages integrate with existing Lightning message flow during commitment transaction updates. Both channel partners must validate all outputs before signing new commitments, ensuring consensus on protocol rules.

**MuSig2 Support:**
- Key derivation for vault address
- Cooperative attestation

### 6.2 Vault Operator Expectations

**Multi-Vault Network Management:**
- Ideally maintain 4-6 Lightning channels with different partners
- Ensure channel partners audit other vaults
- Maintain balanced vault sizes for symmetric security exposure
- Choose security deposits aligned with number of vaults (~100% ÷ count)

**Operational Expectations:**
- Process deposit operations and payment authorizations
- Maintain full reserve requirements plus security deposits across all vaults
- Publish consistent fee schedules
- Coordinate with channel partners for recovery scenarios
- Build reputation through connections and consistent cross-vault performance

### 6.3 Channel Partner Expectations

**Vault Infrastructure Support:**
- Participate in MuSig2 vault creation
- Validate and co-sign vault payment processing
- Monitor commitment transactions for validated outputs
- Provide infrastructure support for vault operations

**Cross-Auditing Responsibilities:**
- Monitor channel partners' other vault operations
- Report performance metrics to reputation systems
- Force close channels upon detecting dishonesty in channel partners
- Participate in recovery processes for force closes
- Maintain audited vault balance records

**Network Participation:**
- Maintain relationships with multiple vault operators
- Coordinate with other channel partners for monitoring
- Participate in recovery processes as needed
- Compete on infrastructure quality and reliability

### 6.4 Other-channel Partner Expectations

- Cross-monitor commitment updates
- Monitor vault payment processing integrity
- Provide hard enforcement through unilateral close
- Directly participate in recovery process

### 6.4 Independent Auditor Expectations

- Monitor commitment transaction updates
- Provide soft enforcement through reputation reporting
- Report compliance scores and violations
- Monitor vault payment processing integrity
- Track multi-vault architectures and cross-relationships
- Report on operator collusion risks
- Assist in recovery by providing balance records
- Maintain independence from operators they audit
- Compete on accuracy and reliability of monitoring

### 6.5 Client Expectations

- Manage Schnorr keypairs that each control one or many deposits
- Sign payment authorizations
- Validate invoice signatures to prevent fake invoice attacks
- Track deposit states across multiple vaults
- Prefer operators with robust multi-vault architectures
- Implement deposit distribution strategies

**Nostr Wallet Connect Integration**:
- Enables use of any updated NWC-compatible wallet as interface
- Deposits can be accessed through web pages using browser credential management
- No app store requirements or dedicated applications needed
- Messages routed through Nostr relays for censorship resistance

## 7. Protocol Analysis

### 7.1 Universal Applicability

Bitcoin Deposits serves depositors across the entire economic spectrum:

**High-Value Depositors**:
- Simplified Lightning access without channel management
- Ability to distribute holdings across multiple operators
- Corporate operators provide institutional-grade security
- Maintain cryptographic control of funds

**Medium-Value Depositors**:
- Access to Lightning without on-chain transactions
- Choice of operators based on reputation/trust preferences
- Easy migration between operators as needs change
- Community operators provide good balance of trust and accessibility

**Low-Value Depositors**:
- Entry point to Lightning Network without minimum balances
- Progressive upgrade path as holdings grow
- Anonymous operators enable permissionless access
- Distributed deposits minimize risk

### 7.2 Risk Management Through Network Effects

**Vault Network Distribution:**
The protocol naturally encourages risk management through network effects:

**Operator Selection Criteria:**
- Prefer operators with 4-6 vault robust networks
- Assess quality and reputation of channel partners
- Monitor cross-vault performance metrics
- Evaluate geographic and operational diversity
- Track security deposit levels across vaults

**Network Health Indicators:**
- Balanced vault sizes indicate professional operation
- Diverse channel partner relationships reduce systemic risk
- Consistent performance across all vaults builds confidence
- Strong cross-auditing relationships provide transparency
- Security deposit ratios align with network size

**Dynamic Rebalancing:**
- Monitor network health changes over time
- Move deposits based on cross-vault performance
- Reward operators who maintain robust networks
- Early warning systems detect network degradation
- Market forces improve network architectures

**Systemic Resilience:**
- Multiple vault failures required to affect depositor funds
- Channel partner network provides natural recovery
- Cross-auditing prevents undetected issues
- Network effects punish bad actors across all relationships
- Organic trust through visible network relationships

### 7.3 Economic Incentive Alignment

The protocol creates sustainable economics for all participants:

**For Operators**:
- Revenue from maintenance fees and low connectivity routing
- Reputation as a valuable long-term asset
- Market differentiation through service quality
- Multi-vault web of operations builds reputation over time

**For Channel Partners**:
- Revenue sharing through liquidity
- Economic incentive to properly validate vault payments
- Reduced risk through shared responsibility
- Natural business model for LSPs
- Natural potential benefit from recovery process

**For Auditors**:
- Potential fee revenue from monitoring services
- Reputation for accurate reporting
- Network effects from wide adoption
- Competition drives monitoring innovation
- Natural role for channel partners

**For Depositors**:
- Access to Lightning without technical complexity
- Choice of trust/convenience/cost tradeoffs
- Competitive fee pressure from operator competition
- Increasing security as ecosystem matures
- Cryptographic protection against payment theft
- Organic trust through visible architectures

### 7.4 Trust Model

The protocol operates on a **progressive trust model** that adapts to depositor needs:

**Cryptographic guarantees**:
- Authorization required for all balance decreases
- Validation rules enforced by channel consensus
- Atomic transfers via Lightning network
- Depositor maintains exclusive control via private key
- Multisig attestation prevents deceptive invoices
- Evidence of non-credited payment reveals dishonest operators

**Web of Trust guarantees**:
- Multi-vault architectures create mutual accountability
- Cross-auditing provides continuous monitoring
- Balanced vault sizes ensure symmetric exposure
- Force-close risks deter dishonest behavior
- Security deposits align long-term incentives

**Economic guarantees**:
- Over-reserve requirements provide skin in the game and replacement of stolen funds
- Security deposit forfeiture exceeds theft gains
- Reputation value exceeds potential theft gains
- Competition ensures good behavior

**Social guarantees**:
- Transparent metrics enable informed decisions
- Community monitoring provides oversight
- Market dynamics punish bad actors
- Success stories build ecosystem trust

### 7.6 Why Not Existing Solutions?

**Why not Cashu/Fedimint (Ecash)?**
- Requires mint-specific tokens that fragment liquidity
- No cryptographic proof of reserves - just promises
- No direct Lightning integration - mints must manage Lightning liquidity separately
- Federated mints add coordination complexity without solving trust

**Why not Ark?**
- Requires users to come online every few weeks or lose funds
- Introduces novel cryptographic assumptions and complexity
- Requires users to understand and manage timeout risk
- Requires service providers who could censor

**Why not Spark/Statechains?**
- Limited to fixed denominations reducing payment flexibility
- Requires users to find counterparties/SSPs for transfers
- Not integrated with Lightning Network routing
- Assumes that operators will not leak and securely delete keys
- Simple transactions can grief using their unilateral exit
- Early transactions are less expensive to grief
- Later transactions are more expensive to defend
- On-chain activity defeats scaling goals

**Why not Liquid?**
- Federated peg introduces trust in federation members
- Limited Lightning integration requires atomic swaps
- Permissioned network where federation can censor transactions

**Why not Channel Factories?**
- Requires on-chain transactions to join factory
- Complex coordination between factory participants
- Doesn't solve offline receiving or channel management
- Uncooperative member can force everyone on-chain

**Why not Hosted Channels?**
- No consensus enforcement of channel rules
- Provider can arbitrarily modify balances
- No standardization across implementations
- Requires trust without cryptographic guarantees

**Why not Lightning Addresses with LSPs?**
- Requires each user to have their own Lightning node
- LSPs still require on-chain transactions for channel creation
- Users must manage channel liquidity and backups
- Complexity of node management remains

**Why not Simple Custody?**
- No cryptographic proof of reserves
- No protection against selective payment processing
- Single point of failure with no recovery mechanism
- No competitive pressure for good behavior

### 7.6 Deployment Strategy

Bitcoin Deposits enables organic growth through market dynamics:

**Bootstrap Phase**:
- Anonymous operators start with minimal deposits
- Early adopters test with small amounts
- Multi-vault architectures emerge naturally
- Reputation metrics establish baselines
- Community discusses experiences
- Multisig addressing provides security from day one

**Growth Phase**:
- Successful operators attract larger deposits
- Multi-vault networks become standard
- Corporate entities enter with instant credibility
- Wallet integration improves depositor experience
- Network effects drive adoption
- Proven security model builds confidence

**Maturity Phase**:
- Full spectrum of operators serves all depositors
- Robust cross-auditing networks established
- Insurance products develop for additional protection
- Regulatory frameworks adapt to reality
- Protocol becomes standard Lightning infrastructure
- Multi-vault architectures become expected standard

### 7.7 Future Considerations

1. **Privacy Enhancements**: Balance transparency vs. operator needs while maintaining multisig security
2. **Cross-Implementation Support**: Ensure broad compatibility of MuSig2 coordination
3. **Protocol Governance**: Upgrade mechanisms as ecosystem evolves
4. **Tokenization options**: Guaranteed full reserve ecash. Recovery is an issue

## 8. Conclusion

Bitcoin Deposits provides a trust-minimized path to Lightning Network access for all depositors, regardless of their technical capability, connectivity, or wealth. The introduction of MuSig2 keys for vault payments significantly strengthens the security model by preventing operators from selectively honoring payments. The addition of multi-vault architectures with cross-auditing creates organic trust through mutual accountability, making collusion economically irrational. By acknowledging that different depositors have different needs and risk tolerances, the protocol enables a market-based approach where deposit sizes naturally match operator reputation values and architectural robustness.

The key innovation is not in preventing all possible theft but in creating transparent mechanisms for reputation building, structural accountability, and risk assessment, while cryptographically preventing the most common attack vectors and economically disincentivizing collusion. Depositors can make informed decisions about which operators to trust with what amounts, while market dynamics ensure that good operators thrive and bad operators fail.

This pragmatic approach enables:
- **Universal access** to Lightning payments without on-chain transactions
- **Depositor sovereignty** through cryptographic control of deposits
- **Cryptographic payment integrity** via MuSig2 addressing
- **Organic trust building** through visible multi-vault architectures
- **Market efficiency** through operator competition
- **Progressive adoption** as deposits can start small and grow
- **Systemic resilience** through diversification and choice

The protocol succeeds by providing infrastructure that meets people where they are. Whether someone has $10 or $10 million, intermittent connectivity or fiber internet, technical expertise or none at all—Bitcoin Deposits provides an appropriate solution that maintains the core principle of cryptographic verifiability while acknowledging the realities of human trust networks.

The combination of MuSig2 addressing and organic multi-vault architectures transforms Bitcoin Deposits from a purely reputation-based system to one with strong cryptographic guarantees and structural incentives for honest behavior. This makes the protocol suitable for peoples across the entire economic spectrum, from small-amount testers to large-scale adopters.

Bitcoin Deposits is evolution, not revolution. It builds on Lightning's success while addressing its adoption barriers. By creating space for operators at all reputation levels to serve peoples with all risk tolerances, by providing cryptographic protection against the most common attack vectors, and by fostering organic trust through cross-vault accountability, we enable the Lightning Network to truly become the payment layer for the world.
