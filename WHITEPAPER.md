# Bitcoin Deposits Protocol

## Overview

The **Bitcoin Deposits Protocol** is a Layer 3 trust-minimized custodial wallet system built on top of LDK Node. The protocol enables operators to manage user deposits while being constrained by cryptographic rules enforced at the Lightning channel level.

### Why This Design?

Traditional custodial wallets require users to fully trust the operator. Bitcoin Deposits solves this through:

1. **Economic Constraints**: Operators must lock up 200% of deposits (100% reserves + 100% collateral), making theft economically irrational
2. **Cryptographic Accountability**: Every operation is signed and hash-chained, creating unforgeable audit trails
3. **Multi-Party Watching**: Channel partners and collateral partners independently validate operations
4. **Automatic Recovery**: Time-locked mechanisms ensure funds are never permanently stuck

The result: a wallet service where operators **can't** steal, not just **won't** steal.

---

## Core Systems

The protocol consists of 10 interconnected systems:

| # | System | Description |
|---|--------|-------------|
| 1 | **Cryptographic Ledger** | Hash-chained state updates signed by operator |
| 2 | **Multichannel Collateral** | Reserves backed by channel balance + attestations from other channels |
| 3 | **Dedicated Commitment Output** | Tapscript reserves output embedded in Lightning commitment tx |
| 4 | **Peer Multisig + VoterSet** | Reserves output spends to multisig of channel peers + collateral partners |
| 5 | **Invoice Cosigning** | Partner must cosign invoices before they're valid |
| 6 | **Enhanced Watchtower** | Watching for force-closes on channels where we're in the multisig |
| 7 | **Ledger Validation** | Conformance evaluation against protocol rules |
| 8 | **Judgement Voting** | RecoveryVote conforming/non-conforming determination |
| 9 | **Tiered Recovery** | Time-based threshold degradation (1-of-N → 3-of-N → any) |
| 10 | **Ledger Reassignment** | Claim process transferring deposits to new operator |

---

## 1. Cryptographic Ledger

Every ledger update forms a cryptographic hash chain signed by the operator and broadcast to auditors.

### Hash Chain Structure

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Update #0   │───►│  Update #1   │───►│  Update #2   │
│  genesis     │    │  prev: H0    │    │  prev: H1    │
└──────────────┘    └──────────────┘    └──────────────┘
```

### SignedLedgerUpdate (Storage Format)

```rust
pub struct SignedLedgerUpdate {
    pub message: Vec<u8>,              // Serialized protocol message
    pub message_type: u16,             // For quick filtering
    pub operator_signature: [u8; 64],  // Operator's ECDSA signature
    pub operator_pubkey: PublicKey,    // Operator's Lightning node ID
    pub partner_pubkey: PublicKey,     // Partner's node ID (ledger identifier)
    pub sequence_number: u64,          // Position in update chain
    pub previous_state_hash: [u8; 32], // Hash chain linkage
    pub current_state_hash: [u8; 32],  // Hash after this update
    pub timestamp: u64,                // When operator signed
}
```

The hash of each update is computed deterministically:
`SHA256(sequence_number || previous_state_hash || message_bytes)`

This hash becomes the `previous_state_hash` for the next update, forming the chain.

### SignedAuditUpdateMsg (Wire Format)

The wire protocol uses `SignedAuditUpdateMsg` for broadcasting:

```rust
pub struct SignedAuditUpdateMsg {
    pub message: Vec<u8>,              // Serialized original message
    pub message_type: u16,             // Message type for filtering
    pub operator_signature: [u8; 64],  // Operator's ECDSA signature
    pub operator_pubkey: PublicKey,    // Operator's Lightning node ID
    pub partner_pubkey: PublicKey,     // Partner's public key
    pub sequence_number: u64,          // Position in update chain
    pub previous_state_hash: [u8; 32], // Hash before this update
    pub current_state_hash: [u8; 32],  // Hash after this update
    pub timestamp: u64,                // When signed
}
```

### Audit Broadcasting

Every ledger update triggers broadcast to all partners and auditors:

1. **Operator applies update** - Calls `append_mut()`, captures `prev_hash` and `new_hash`
2. **Sent to partner** - Original message with TLV payload
3. **Broadcast to auditors** - `SignedAuditUpdateMsg` sent to all other partners/auditors

```
Operator                    Partner                   Other Partners
    │                          │                           │
    │─── Proposed Update ─────►│                           │
    │◄─── Ack'd Proposal ──────│                           │
    │                          │                           │
    │─── SignedAuditUpdate ───────────────────────────────►│
    │    (prev_hash, new_hash, sequence)                   │
```

**SignedLedgerUpdateLog** buffers out-of-order updates in `pending_updates` and detects duplicates by `current_state_hash` match.

Auditors can verify:
- Operator signature validity
- Hash chain integrity
- Sequential ordering

---

## Bidirectional Ledgers

Each Lightning channel supports **two independent operator ledgers** - one in each direction.

### Key Insight: Asymmetric Validation

The system maintains **two copies** of each ledger with different validation behaviors:

| Copy | Location | Behavior | Why |
|------|----------|----------|-----|
| **Operator Copy** | `channel_ledgers` | Strict validation - rejects out-of-order updates | Operator creates updates, must maintain perfect chain |
| **Partner Copy** | `partner_ledgers` | Relaxed validation - buffers out-of-order updates | Partner receives updates, may arrive out of order due to network |

This asymmetry allows the system to handle network delays without losing integrity:

```
┌──────────────────────────────────────────────────┐
│                    Lightning Channel             │
│                     Alice ◄──► Bob               │
├──────────────────────────────────────────────────┤
│                                                  │
│  ┌──────────────────┐      ┌──────────────────┐  │
│  │ Alice→Bob Ledger │      │ Bob→Alice Ledger │  │
│  │ Operator: Alice  │      │ Operator: Bob    │  │
│  │ Partner: Bob     │      │ Partner: Alice   │  │
│  │ Hash: 8205a6bb   │      │ Hash: cb5ad07f   │  │
│  └──────────────────┘      └──────────────────┘  │
│                                                  │
│  Commitment TX has up to two reserves outputs:   │
│  ├─ local_reserves  (our operator ledger)        │
│  └─ remote_reserves (peer's operator ledger)     │
└──────────────────────────────────────────────────┘
```

### Why Bidirectional?

- Each operator manages their own deposits independently
- Both sides hold synced copies of BOTH ledgers (for validation)
- Commitment transactions must include reserves for both ledgers
- Partners validate they agree on the state of both ledgers before signing

### Ledger Sync

Both parties maintain copies of both ledgers via `LedgerUpdate` broadcasts:
- **Operator copy**: Authoritative, includes signed updates
- **Partner copy**: Mirror for validation, tracks operator's state

When building commitment transactions, both parties must agree on the hash of each ledger to compute matching signatures.

---

## 2. Multichannel Collateral

The 100%+100% collateral model ensures deposits are backed by both direct reserves AND slashable collateral from other channels.

### The Two-Layer Protection

```
Layer 1 - RESERVES:      reserves >= deposits (direct backing in this channel)
Layer 2 - ATTESTATIONS:  attestations >= deposits (slashable funds in other channels)
```

Both requirements must be met before any operation that increases deposit liability.

### Example: Alice Operates Three Channels

```
Alice→Bob:     deposits=60k, reserves=80k (20k as collateral)
Alice→Charlie: deposits=50k, reserves=80k (30k as collateral)
Alice→David:   deposits=40k, reserves=70k (30k as collateral)

Total deposits: 150k
Total reserves: 230k (80k as slashable collateral)
```

For the Alice→Bob ledger:
- **Reserves**: 60k ≥ 60k deposits ✓
- **Attestations**: Charlie attests 30k + David attests 30k = 60k ≥ 60k deposits ✓

### Collateral Attestation Message

Partners send `COLLATERAL_ATTESTATION` to attest how much of the operator's funds are collateral at risk:

```rust
pub struct CollateralAttestationMsg {
    pub reserves_output_amount: u64,  // Operator's reserves in our channel
    pub reserves_required: u64,       // Required for operator's deposits with us
    pub block_height: u32,            // Freshness check
    pub signature: [u8; 64],          // Partner's signature
}

// Available collateral = reserves_output_amount - reserves_required
// This is what recovery can slash if the operator cheats
```

### Game Theory: Why Theft is Unprofitable

If Alice colludes with Bob to steal the 60k reserves from Alice→Bob:
1. The reserves are gone from Alice→Bob
2. BUT Charlie can slash 30k of Alice's excess in Alice→Charlie
3. AND David can slash 30k of Alice's excess in Alice→David
4. Users recover 60k from slashed collateral

**Alice cannot profit from colluding with any single partner** — her collateral in other channels covers the theft.

### The Economics of Attack

| Attack | Capital Required | Losses if Caught | Success Probability | Expected Value |
|--------|-----------------|------------------|---------------------|----------------|
| Steal 10k sats | 20k+ reserves | 20k+ slashing | ~0% (cryptographically proven) | **NEGATIVE** |
| Steal 100k sats | 200k+ reserves | 200k+ slashing | ~0% | **NEGATIVE** |
| Any theft | 2x theft amount | 2x+ theft amount | ~0% | **NEGATIVE** |

**Key insight**: To steal X, operator must have locked 2X+ capital. Getting caught means losing all 2X+. Getting caught is near-certain due to:
- Signed ledger updates prove every operation
- Hash chain detects any modification
- Multiple independent validators watching
- Cryptographic proofs are irrefutable

### Capital Efficiency

The same attestation backs multiple ledgers:
- Charlie's 30k excess backs BOTH Alice→Bob AND Alice→David
- David's 30k excess backs BOTH Alice→Bob AND Alice→Charlie

This allows 60k of excess collateral to provide security for 150k of deposits across 3 ledgers, because theft of any ledger triggers slashing that fully covers it, and stealing two at once is hard.

### Validation Before Operations

```rust
pub fn validate_collateral_for_liability(
    &self,
    deposit_liability: u64,
    current_block: u32,
    max_attestation_age_blocks: u32,
) -> Result<(), BitcoinDepositsError> {
    // Requirement 1: reserves >= deposit_liability
    if self.reserves_amount < deposit_liability {
        return Err(BitcoinDepositsError::InsufficientReserves { ... });
    }

    // Requirement 2: attestations >= deposit_liability
    if !self.collateral_partners.is_empty() {
        let total_collateral = self.total_available_collateral(current_block, max_attestation_age_blocks);
        if total_collateral < deposit_liability {
            return Err(BitcoinDepositsError::InsufficientCollateral { ... });
        }
    }
    Ok(())
}
```

---

## 3. Dedicated Commitment Output

The protocol embeds a reserves output in every Lightning commitment transaction. This output uses a tapscript structure that encodes the ledger hash and recovery spending paths.

```rust
// In commitment transaction
reserves_output: TxOut {
    value: reserves_amount,
    script_pubkey: tapscript_with_ledger_hash(
        operator_key,
        partner_key,
        ledger_hash,
        voter_set,
    ),
}
```

When the channel force-closes, this output goes on-chain with the final ledger hash, enabling recovery.

---

## 4. Peer Multisig + VoterSet

The reserves output spends to a multisig controlled by the channel peers and optional collateral partners. This ensures neither party can unilaterally claim funds.

### VoterSet

The VoterSet defines who can participate in recovery voting. The operator does not get a vote - they are being judged.

```rust
pub struct VoterSet {
    voters: Vec<Voter>,  // All voters (tie-breaker + other voters)
}

impl VoterSet {
    /// Create a new voter set with a tie-breaker and additional voters
    pub fn new(tie_breaker: PublicKey, other_voters: Vec<PublicKey>) -> Self;
}
```

In a dangerous 2-party setup:
- Tie-breaker: channel partner
- Other voters: none (partner is sole voter)

In a multi-party setup with collateral partners:
- Tie-breaker: channel partner
- Other voters: collateral partners from other channels

Note: Cooperative spending (operator + partner collude) is handled separately from recovery voting.

### Dynamic Voter Registration

Collateral partners can be added after ledger creation via `ADD_COLLATERAL_PARTNER`:

```rust
pub struct AddCollateralPartnerMsg {
    pub partner_id: PublicKey,           // Channel partner receiving this
    pub collateral_partner: PublicKey,   // New voter to add
    pub operator_signature: [u8; 64],    // Operator authorization
}
```

The partner validates and ACKs, then the new collateral partner is included in future VoterSet constructions and receives ledger update broadcasts.

### Tapscript Structure

The reserves output has multiple spending paths:
- **Cooperative**: Operator + Partner can spend together (normal case)
- **Recovery tiers**: Various threshold requirements after timelock (see Tiered Recovery)

### UpdateReserves Message

The `UpdateReserves` message includes both ledger hashes for bidirectional verification:

```rust
pub struct UpdateReserves {
    pub channel_id: ChannelId,
    pub reserves_sats: u64,           // Amount for sender's reserves output
    pub custodian_pubkey: PublicKey,  // Operator's custodian key
    pub ledger_hash: [u8; 32],        // Sender's operator ledger hash
    pub remote_ledger_hash: [u8; 32], // Sender's view of peer's ledger hash
}
```

The `remote_ledger_hash` field enables declarative reserves: both parties can verify they have synchronized copies of BOTH ledgers before building the commitment transaction. This prevents signature mismatches when both sides have different views of channel state.

### UpdateReserves Flow

```
Operator                                    Partner
    │                                          │
    │─────── UpdateReserves ──────────────────►│
    │        (reserves_sats,                   │
    │         ledger_hash,                     │
    │         remote_ledger_hash)              │
    │                                          │
    │        Partner verifies:                 │
    │        - ledger_hash matches their copy  │
    │        - remote_ledger_hash matches      │
    │          their operator ledger           │
    │                                          │
    │◄────── AcceptReserves ───────────────────│
    │                                          │
    │─────── commitment_signed ───────────────►│
    │                                          │
    │◄────── revoke_and_ack ───────────────────│
```

**Important**: Only operators send `UpdateReserves`. Partners validate and respond with `AcceptReserves`. This asymmetry prevents race conditions where both sides send conflicting reserve amounts simultaneously.

---

## 5. Invoice Cosigning

Before an invoice is trustworthy, the partner must cosign it. This prevents the operator from creating invoices that would exceed reserve coverage, or silently pocketing the proceeds.

### Flow

1. Operator creates invoice for deposit
2. Operator sends `RECEIVING_COSIGN_INVOICE` to partner
3. Partner validates: adequate reserves exist for this invoice exposure
4. Partner returns cosignature
5. Invoice is now valid and can be shared with wallet and payers

### Validation Rules

- Invoice amount + existing invoices ≤ available reserves
- Deposit exists and is in good standing
- Invoice expiry is reasonable

### Uncredited Payment Accusation

When a payment is settled (preimage revealed) but the operator has not already issued a `RECEIVING_CREDIT_PAYMENT` to credit the wallet, the partner can broadcast an `UNCREDITED_PAYMENT` accusation.

**This is a fraud proof** - the preimage cryptographically proves the payment was settled.

**Accusation Flow:**

1. Wallet receives incoming Lightning payment (preimage revealed to settle HTLC)
2. Partner expects operator to send `RECEIVING_CREDIT_PAYMENT` to credit the deposit
3. If no credit arrives within reasonable time, partner detects fraud
4. Partner calls `broadcast_uncredited_payment_accusation()`:
   - **Force-closes the channel** with operator to protect funds
   - Constructs accusation with cryptographic proof
   - Broadcasts to operator + all collateral partners
   - Emits `UncreditedPaymentAccusation` event

**API:**

```rust
/// Initiate an uncredited payment accusation (partner-side)
fn broadcast_uncredited_payment_accusation(
    &self,
    operator: PublicKey,
    payment_hash: [u8; 32],
    preimage: [u8; 32],
    deposit_pubkey: PublicKey,
    amount_msat: u64,
    invoice_cosignature: [u8; 64],
) -> Result<(), String>
```

**UncreditedPaymentMsg Structure:**

```rust
pub struct UncreditedPaymentMsg {
    pub operator: PublicKey,           // Operator who failed to credit
    pub partner: PublicKey,            // Partner making the accusation
    pub payment_hash: [u8; 32],        // Payment hash that was settled
    pub preimage: [u8; 32],            // Preimage proving settlement
    pub deposit_pubkey: PublicKey,     // Deposit that should have been credited
    pub amount_msat: u64,              // Amount that should have been credited
    pub invoice_cosignature: [u8; 64], // Partner's cosignature (proves valid invoice)
    pub settlement_sequence: u64,      // Ledger sequence at settlement
    pub settlement_ledger_hash: [u8; 32], // Ledger hash at settlement
    pub settlement_block_height: u32,  // Block when settlement occurred
    pub accuser_signature: [u8; 64],   // Signature over accusation
}
```

**Verification (receiver-side):**

1. Verify preimage: `SHA256(preimage) == payment_hash`
2. Verify sender is the claimed partner
3. Check ledger for existing credit via `ledger.has_credit_for_payment(&payment_hash)`
4. If credit exists, accusation is invalid
5. If no credit, record violation and emit event

**Consequences:**

- Channel is force-closed immediately (funds protected)
- Accusation broadcast to all collateral partners (auditors)
- Recorded as `ConformanceViolation::UncreditedPayment`
- Counts toward non-conformance determination in recovery voting
- Partner can initiate recovery claim if sufficient violations exist

---

## 6. Enhanced Watchtower

Partners watch for force-closes on channels where they are part of the reserves multisig. This enables timely recovery initiation.

### Responsibilities

1. Monitor blockchain for commitment transactions containing reserves outputs
2. Detect force-close by parsing reserves output script
3. Extract ledger hash from on-chain data
4. Initiate recovery evaluation process
5. Participate in voting and claims

### Detection

When a channel force-closes:
- The reserves output appears on-chain
- The embedded ledger hash reveals the final state
- Watchtowers sync their audit records to this hash

---

## 7. Ledger Validation

After force-close, partners evaluate the operator's signed update chain against protocol rules to determine conformance.

### Evaluation Process

1. Retrieve signed update log from storage
2. Verify hash chain integrity (each hash links to previous)
3. Verify operator signature on each update
4. Replay updates and validate each against protocol rules
5. Compare final computed hash to on-chain ledger hash

### Protocol Rules Checked

- Deposits only receive funds from valid Lightning payments
- Payments only debit from deposits with sufficient balance
- Reserves adjusted correctly after each balance change
- No unauthorized operations

---

## 8. Judgement Voting

Partners vote on whether the operator was conforming or non-conforming based on their ledger validation.

### RecoveryVote Message

```rust
pub struct RecoveryVoteMsg {
    pub operator: PublicKey,
    pub partner: PublicKey,
    pub voter: PublicKey,
    pub is_conforming: bool,
    pub validated_hash: [u8; 32],
    pub validated_sequence: u64,
    pub substitute_nomination: Option<PublicKey>,
    pub discovered_violation: bool,
    pub signature: [u8; 64],
}
```

### Voting Flow

```
Force Close Detected
        │
        ▼
┌─────────────────────┐
│ Each voter          │
│ evaluates ledger    │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ Voters broadcast    │
│ RecoveryVote        │
└─────────┬───────────┘
          │
          ▼
┌──────────────────────────────┐
│ Threshold reached            │
│ → RecoveryNonCompliant event │
└──────────────────────────────┘
```

---

## 9. Tiered Recovery

When a majority of voters determine an operator is non-conforming, funds are awarded based on time-degrading eligibility tiers.

### Why Time-Based Degradation?

The system balances two competing concerns:
1. **Prevent monopolistic seizure**: Don't let one partner grab everything immediately
2. **Guarantee eventual access**: Don't let funds get stuck forever if partners are unresponsive

Time-based tiers solve both: early claims require coordination, later claims get progressively easier.

### Recovery Phases

After force-close, recovery proceeds through phases:

```
Force Close (Block N)
        │
        ▼ (wait 6 blocks for entropy)
┌──────────────────────────┐
│ WaitingForEntropy        │ Block N → N+6
│ (prevents frontrunning)  │
└───────────┬──────────────┘
            │
            ▼
┌──────────────────────────┐
│ Evaluating               │ All voters independently
│ (conformance check)      │ validate ledger
└───────────┬──────────────┘
            │
    ┌───────┴───────┐
    ▼               ▼
Conforming     NonCompliant
(funds return) (tiered claims)
```

### Award Tiers

| Tier | Block Window | Recipient | Threshold | Description |
|------|-------------|-----------|-----------|-------------|
| 0 | 0-143 (~day 1) | Selected partner | 1-of-1 | Pseudorandomly selected from voters |
| 1 | 144-1007 (~days 1-7) | Any 3 partners | 3-of-N | Requires consensus of multiple nodes |
| 2 | 1008-2015 (~weeks 1-2) | Any partner | 1-of-N | Any single voter can claim |
| 3 | 2016+ (~after 2 weeks) | Community | variable | Fallback mechanism |

### Anti-Frontrunning

The pseudorandom selection for Tier 0 uses entropy from block N+6 (where N = force close block), making the recipient unpredictable when force close is triggered.

**Selection algorithm:**
```
selected_partner = voters[SHA256(entropy_block_hash || ledger_id) mod len(voters)]
```

Since the entropy block doesn't exist when force-close is triggered, no one can predict who will be selected.

### ClaimEligibility

```rust
pub enum ClaimEligibility {
    SelectedPartnerOnly { partner: PublicKey },  // Tier 0: awarded to specific partner
    AnyThreePartners,                             // Tier 1: any 3 voters together
    AnySinglePartner,                             // Tier 2: any single voter
    CommunityFallback,                            // Tier 3: fallback
}
```

---

## 10. Ledger Reassignment

The claim process transfers deposits from a non-compliant operator to a new operator.

### Claim Flow

```
┌───────────────────────┐
│ Eligible claimant     │
│ initiates claim       │
└───────────┬───────────┘
            │
            ▼ (RecoveryClaimRequest)
┌───────────────────────┐
│ Request signatures    │
│ from voters           │
└───────────┬───────────┘
            │
            ▼ (RecoveryClaimSignature)
┌───────────────────────┐
│ Collect signatures    │
│ until threshold met   │
└───────────┬───────────┘
            │
            ▼ (threshold met → RecoveryClaimReady event)
┌───────────────────────┐
│ Broadcast claim tx    │
│ to Bitcoin network    │
└───────────┬───────────┘
            │
            ▼ (RecoveryClaimComplete)
┌───────────────────────┐
│ Deposits reassigned   │
│ to new operator       │
└───────────────────────┘
```

### Outcome

- Non-compliant operator loses reserves
- Deposits are transferred to claiming partner
- Depositors' funds are protected
