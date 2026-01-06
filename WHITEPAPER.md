# Bitcoin Deposits Protocol

## Overview

The **Bitcoin Deposits Protocol** is a Layer 3 trust-minimized custodial wallet system built on top of LDK Node. The protocol enables operators to manage user deposits while being constrained by cryptographic rules enforced at the Lightning channel level.

### Why This Design?

Traditional custodial wallets require users to fully trust the operator. Bitcoin Deposits solves this through:

1. **Economic Constraints**: Operators lock collateral (150-300%+ of deposits) making theft economically irrational – partners are armed with slashing power and won't coordinate because they can't trust each other
2. **Cryptographic Accountability**: Every operation is signed and hash-chained, creating unforgeable audit trails
3. **Multi-Party Watching**: Channel partners and collateral partners independently validate operations
4. **Automatic Recovery**: Time-locked mechanisms ensure funds are never permanently stuck

The result: a wallet service where theft is **economically irrational**, not merely discouraged. Partners form a Mexican standoff – each armed with slashing power, none willing to move first because they can't trust the others to coordinate. Security level depends on coverage ratio and partner count (see Section 12).

---

## Core Systems

The protocol consists of 12 interconnected systems:

| # | System | Description |
|---|--------|-------------|
| 1 | **Cryptographic Ledger** | Hash-chained state updates signed by operator |
| 2 | **Multichannel Collateral** | Reserves backed by channel balance + attestations from other channels |
| 3 | **Dedicated Commitment Output** | Tapscript reserves output embedded in Lightning commitment tx |
| 4 | **Peer Multisig + VoterSet** | Hard 40% circular limit preserves Mexican standoff game theory |
| 5 | **Invoice Cosigning** | Invoice-deposit binding with partner cosignature prevents payment misdirection |
| 6 | **Enhanced Watchtower** | Watching for force-closes on channels where we're in the multisig |
| 7 | **Ledger Validation** | Conformance evaluation against protocol rules |
| 8 | **Recovery Voting** | Conformance determination with degrading thresholds |
| 9 | **Recovery Claims** | Sequential random selection for fund distribution |
| 10 | **Transitive Liability** | Two-stage capital locks + evidence propagation through partner graph |
| 11 | **Collateral Transparency** | 5% rate limits, 26-week lookback, churn tracking enable market response |
| 12 | **Security Model** | Explicit capital requirements, collusion protection formulas, and user tiers |

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

### Bidirectional Ledgers

Each Lightning channel supports **two independent operator ledgers** - one in each direction.

#### Asymmetric Validation

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

#### Why Bidirectional?

- Each operator manages their own deposits independently
- Both sides hold synced copies of BOTH ledgers (for validation)
- Commitment transactions must include reserves for both ledgers
- Partners validate they agree on the state of both ledgers before signing

#### Ledger Sync

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

**Alice cannot profit from colluding with any single partner**  –  her collateral in other channels covers the theft.

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

### Capital Efficiency vs. Collusion Protection

**Key design decision:** The same attestation can back multiple ledgers by the same operator.

```
Charlie's 30k attestation backs:
  └── Alice→Bob ledger (30k deposits)
  └── Alice→David ledger (30k deposits)
  └── Alice→Eve ledger (30k deposits)
```

This enables capital efficiency: 30k of Charlie's excess secures 90k of deposits across 3 channels. An operator with 10 channels doesn't need 10x the collateral – the same attestations provide coverage across all ledgers.

**The tradeoff:** Capital efficiency creates vulnerability to multi-partner collusion.

| Scenario | Outcome |
|----------|---------|
| Single partner colludes | Slashing from other channels covers theft ✓ |
| Majority of partners collude | Shared collateral may be insufficient ✗ |

If Alice colludes with Bob AND David (2 of 3 partners), she can steal from both channels while only Charlie's attestation remains to slash. The same collateral that provided efficiency now provides insufficient coverage.

**The formula (see Section 12):**

To protect against K colluding partners out of N total:
```
Required_Coverage = 100% + (K / (N - K)) × 100%
```

| Partners | Collusion Threat | Required Coverage |
|----------|-----------------|-------------------|
| 3 partners | 1 colluding | 150% |
| 3 partners | 2 colluding | 300% |
| 10 partners | 2 colluding | 125% |
| 10 partners | 5 colluding | 200% |

**Operator choice:** More partners at lower per-partner coverage, OR fewer partners at higher coverage. Both can achieve the same security level. The protocol makes this tradeoff visible; operators and users choose their position on the curve.

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
pub struct Voter {
    pub pubkey: PublicKey,
    pub attested_amount: u64,      // Capital attested to this channel
    pub is_tie_breaker: bool,
}

pub struct VoterSet {
    voters: Vec<Voter>,
}

impl VoterSet {
    /// Average attestation (capped contributions)
    pub fn average_attestation(&self) -> u64 {
        if self.voters.is_empty() {
            return 0;
        }
        self.capped_total() / self.voters.len() as u64
    }

    /// Total with each contribution capped at 2x average
    /// (iteratively computed since cap depends on average)
    fn capped_total(&self) -> u64 {
        let raw_total: u64 = self.voters.iter()
            .map(|v| v.attested_amount)
            .sum();
        let raw_avg = raw_total / self.voters.len() as u64;
        let cap = raw_avg * 2;

        self.voters.iter()
            .map(|v| v.attested_amount.min(cap))
            .sum()
    }

    /// Minimum = half the average attestation
    pub fn minimum_attestation(&self) -> u64 {
        self.average_attestation() / 2
    }

    /// Maximum contribution that counts = twice the average
    pub fn maximum_attestation(&self) -> u64 {
        self.average_attestation() * 2
    }

    /// Add a voter
    /// - Must attest >= half the average
    /// - Contribution capped at twice the average
    pub fn add_voter(
        &mut self,
        pubkey: PublicKey,
        attested_amount: u64,
        is_tie_breaker: bool,
    ) -> Result<(), Error> {
        let minimum = self.minimum_attestation();

        // Tie-breaker (channel partner) exempted from minimum
        if !is_tie_breaker && attested_amount < minimum {
            return Err(Error::InsufficientAttestation {
                required: minimum,
                provided: attested_amount,
            });
        }

        self.voters.push(Voter {
            pubkey,
            attested_amount,
            is_tie_breaker,
        });
        Ok(())
    }

    /// Get voter's effective contribution (capped)
    pub fn effective_attestation(&self, voter: &Voter) -> u64 {
        voter.attested_amount.min(self.maximum_attestation())
    }
}
```

**Why This Band?**

Attestation requirements are relative to existing voters, with floor and ceiling:
- **Minimum to vote**: 0.5x average (accessible, but meaningful)
- **Maximum contribution**: 2x average (no whale dominance)

```
Starting state:
- Charlie attests 50k, Dave attests 50k
- Average: 50k
- Minimum to join: 25k (0.5x)
- Max contribution: 100k (2x)

Adding voter with 30k:
- 30k >= 25k minimum ✓
- Joins VoterSet
- New average recalculates

Adding voter with 10k:
- 10k < 25k minimum ✗
- Rejected

Whale with 500k wants to join:
- 500k >= 25k minimum ✓
- Joins, but only 100k counts (2x cap)
- Can't jack up the minimum for future voters
```

**Sybil attack blocked:**
```
Charlie (50k) + Dave (50k), average = 50k, minimum = 25k

Operator wants 5 Sybils:
- Each Sybil needs 25k to join
- 5 Sybils = 125k real capital
- All 125k is slashable via transitive liability
- Attack is expensive, not cheap
```

**Whale dominance blocked:**
```
Charlie (50k) + Dave (50k), average = 50k

Eve joins with 500k:
- Only 100k counts toward average (2x cap)
- New average = (50 + 50 + 100) / 3 = 66k
- NOT (50 + 50 + 500) / 3 = 200k
- Future minimum = 33k, not 100k
- Eve can't price out smaller voters
```

**What minimum attestation does NOT do:**

It doesn't weight votes. Each voter gets one vote. The on-chain tapscript uses standard threshold multisig (K-of-N), not weighted signing.

**What it DOES do:**

1. **Raises Sybil cost** - Adding voters requires real capital
2. **Enables transitive liability** - Dishonest voters have channels to slash
3. **Aligns incentives** - Everyone in VoterSet has something to lose

The actual protection against collusive voting comes from:
- **Tiered recovery**: Degrading thresholds mean any single non-colluding voter eventually wins
- **Transitive liability**: Collusive votes are evidence; other partners slash the colluding voter to protect their own capital

In a dangerous 2-party setup:
- Tie-breaker: channel partner (exempted from minimum)
- Other voters: none (partner is sole voter)

In a multi-party setup with collateral partners:
- Tie-breaker: channel partner
- Other voters: collateral partners (must meet minimum attestation)

### Dynamic Voter Registration

Collateral partners can be added after ledger creation via `ADD_COLLATERAL_PARTNER`:

```rust
pub struct AddCollateralPartnerMsg {
    pub partner_id: PublicKey,           // Channel partner receiving this
    pub collateral_partner: PublicKey,   // New voter to add
    pub attested_amount: u64,            // Amount they will attest
    pub operator_signature: [u8; 64],    // Operator authorization
}

pub struct CollateralPartnerResponse {
    pub collateral_partner: PublicKey,
    pub approved: bool,
    pub rejection_reason: Option<RejectionReason>,
    pub partner_signature: [u8; 64],
}

pub enum RejectionReason {
    InsufficientAttestation { required: u64, provided: u64 },
    SuspectedSybil { evidence: String },
    ExcessiveVoterInflation { current_voters: u32, proposed: u32 },
    CircularAttestationDetected,
    UnknownIdentity,
}
```

**Partner Approval Required:**

The channel partner MUST approve voter additions. This prevents operators from unilaterally inflating the VoterSet with Sybil voters:

```rust
pub fn evaluate_voter_addition(
    msg: &AddCollateralPartnerMsg,
    current_voter_set: &VoterSet,
    channel_deposits: u64,
) -> CollateralPartnerResponse {
    let minimum = (channel_deposits * MINIMUM_VOTER_ATTESTATION_BPS as u64) / 10000;

    // Check minimum attestation
    if msg.attested_amount < minimum {
        return CollateralPartnerResponse {
            approved: false,
            rejection_reason: Some(RejectionReason::InsufficientAttestation {
                required: minimum,
                provided: msg.attested_amount,
            }),
            ..
        };
    }

    // Check for circular attestation patterns
    if detect_circular_attestation(&msg.collateral_partner) {
        return CollateralPartnerResponse {
            approved: false,
            rejection_reason: Some(RejectionReason::CircularAttestationDetected),
            ..
        };
    }

    // Check for excessive voter inflation
    let new_voter_count = current_voter_set.voters.len() + 1;
    if new_voter_count > MAX_VOTERS_PER_CHANNEL {
        return CollateralPartnerResponse {
            approved: false,
            rejection_reason: Some(RejectionReason::ExcessiveVoterInflation { .. }),
            ..
        };
    }

    CollateralPartnerResponse { approved: true, .. }
}
```

**Circular Attestation Limit:**

Circular attestation—where operators attest to each other—breaks the Mexican standoff game theory. At high circular ratios (>60%), partners become interdependent: slashing cascades failures, making coordination to NOT slash rational. This is a structural failure, not a risk preference.

The protocol enforces a **hard 40% limit**:

```rust
const MAX_CIRCULAR_ATTESTATION_BPS: u16 = 4000;  // 40% maximum

impl VoterSet {
    pub fn circular_attestation_ratio(&self, network: &AttestationNetwork) -> u16 {
        let total: u64 = self.voters.iter().map(|v| v.attested_amount).sum();
        let mut circular: u64 = 0;

        // Circular = path exists from voter back to operator (any length, max depth 5)
        for voter in &self.voters {
            if network.has_path_to_operator(&voter.pubkey, max_depth: 5) {
                circular += voter.attested_amount;
            }
        }

        if total == 0 { return 0; }
        ((circular * 10000) / total) as u16
    }

    pub fn validate_circular_limit(&self, network: &AttestationNetwork) -> Result<(), Error> {
        let ratio = self.circular_attestation_ratio(network);
        if ratio > MAX_CIRCULAR_ATTESTATION_BPS {
            return Err(Error::CircularAttestationExceedsLimit {
                actual_bps: ratio,
                max_bps: MAX_CIRCULAR_ATTESTATION_BPS,
            });
        }
        Ok(())
    }
}
```

**Enforcement:**
- Partners MUST reject collateral partner additions that push circular >40%
- VoterSet construction fails if circular exceeds limit
- This is protocol enforcement, not partner discretion

**Defense in Depth:**

| Layer | Mechanism |
|-------|-----------|
| Hard limit | 40% ceiling enforced at protocol level |
| Graph analysis | Detect chains >3 hops, new nodes <12 weeks, centrality patterns |
| Tier penalties | Tier 1 requires <20%, Tier 2 requires <40% |
| User education | Tools to verify attestor independence |

**Honest Caveat:**

Hidden circular patterns via Sybil intermediaries (Alice → Sybil-1 → Bob → Sybil-2 → Charlie) can evade the 40% limit. The protocol acknowledges this limitation. The hard limit prevents OPEN high-circular configurations; determined attackers with significant capital can create hidden patterns. Users should:
- Prefer operators with <20% circular
- Verify attestors are independent entities
- Monitor for suspicious graph patterns

The partner validates and explicitly approves, then the new collateral partner is included in future VoterSet constructions and receives ledger update broadcasts.

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

### UpdateReserves Validation Requirements

Partner MUST validate UpdateReserves messages to prevent stale state commits:

```rust
pub fn validate_update_reserves(
    msg: &UpdateReserves,
    partner_operator_ledger: &Ledger,
    partner_own_ledger: &Ledger,
) -> Result<AcceptReserves, RejectReason> {
    // 1. Verify ledger_hash matches partner's copy of operator ledger
    if msg.ledger_hash != partner_operator_ledger.current_hash() {
        // Operator referencing different state than partner has
        return Err(RejectReason::LedgerHashMismatch {
            expected: partner_operator_ledger.current_hash(),
            received: msg.ledger_hash,
        });
    }

    // 2. Verify ledger_hash sequence is not older than partner's copy
    let msg_sequence = extract_sequence_from_hash(msg.ledger_hash);
    if msg_sequence < partner_operator_ledger.sequence() {
        // Operator trying to commit stale state
        return Err(RejectReason::StaleState {
            partner_sequence: partner_operator_ledger.sequence(),
            msg_sequence,
        });
    }

    // 3. Verify remote_ledger_hash matches partner's own ledger
    if msg.remote_ledger_hash != partner_own_ledger.current_hash() {
        return Err(RejectReason::RemoteLedgerHashMismatch);
    }

    Ok(AcceptReserves { .. })
}
```

**If validation fails:**

| Failure | Action |
|---------|--------|
| Hash mismatch (newer) | Request missing updates, buffer UpdateReserves |
| Hash mismatch (older) | REJECT UpdateReserves, operator may be malicious |
| Sequence < partner's | Immediately force-close, evidence of attempted stale commit |
| Remote hash mismatch | Reject, request ledger synchronization |

**Post-signing fraud detection:**

If partner signs commitment but later discovers fraud (e.g., receives ledger update proving operator committed stale state):

```rust
pub fn handle_post_signing_fraud(
    signed_commitment: &Transaction,
    fraud_evidence: &LedgerUpdate,
) {
    // 1. Force-close immediately
    broadcast_force_close(signed_commitment);

    // 2. Construct fraud accusation
    let accusation = StaleCommitAccusation {
        committed_hash: extract_ledger_hash(signed_commitment),
        actual_hash: fraud_evidence.state_hash,
        proof: fraud_evidence.clone(),
    };

    // 3. Broadcast to all collateral partners
    broadcast_accusation(accusation);
}
```

Partner who detects fraud post-signing MUST force-close immediately. The race between operator's malicious commitment and partner's honest commitment is 50/50, but fraud is always detectable and slashable.

---

## 5. Invoice Cosigning

Before an invoice is trustworthy, the partner must cosign it. This prevents the operator from creating invoices that would exceed reserve coverage, or silently pocketing the proceeds.

### Flow

1. Operator creates invoice for deposit
2. Operator creates `InvoiceDepositBinding` linking payment_hash to deposit
3. Operator sends `RECEIVING_COSIGN_INVOICE` with binding to partner
4. Partner validates: adequate reserves, valid binding, deposit exists
5. Partner returns cosignature over invoice AND binding
6. Invoice + binding + cosignature sent to user
7. User validates binding before paying

### Invoice-Deposit Binding

Every invoice must include a cryptographically signed binding linking the payment to a specific deposit:

```rust
pub struct InvoiceDepositBinding {
    pub invoice_payment_hash: [u8; 32],
    pub deposit_pubkey: PublicKey,
    pub amount_msat: u64,
    pub operator: PublicKey,
    pub expires_at: u64,
    pub operator_signature: [u8; 64],
}

impl InvoiceDepositBinding {
    pub fn verify(&self) -> Result<(), Error> {
        let message = self.signing_message();
        verify_signature(&message, &self.operator_signature, &self.operator)
    }

    fn signing_message(&self) -> Vec<u8> {
        [
            &self.invoice_payment_hash[..],
            &self.deposit_pubkey.serialize()[..],
            &self.amount_msat.to_le_bytes()[..],
            &self.operator.serialize()[..],
            &self.expires_at.to_le_bytes()[..],
        ].concat()
    }
}
```

**Partner Cosigning with Binding:**

```rust
pub struct ReceivingCosignInvoiceMsg {
    pub invoice: Invoice,
    pub binding: InvoiceDepositBinding,
    pub operator_signature: [u8; 64],
}

pub struct ReceivingCosignResponseMsg {
    pub invoice_payment_hash: [u8; 32],
    pub binding_hash: [u8; 32],            // Hash of binding
    pub partner_cosignature: [u8; 64],     // Signs invoice + binding
    pub partner_pubkey: PublicKey,
}
```

Partner cosigns BOTH the invoice AND the binding hash, creating a cryptographic chain: `partner → binding → invoice → payment`.

**User Validation Before Payment:**

```rust
pub fn validate_invoice_before_payment(
    invoice: &Invoice,
    binding: &InvoiceDepositBinding,
    partner_cosig: &[u8; 64],
    expected_deposit: &PublicKey,
) -> Result<(), Error> {
    // 1. Verify operator signature on binding
    binding.verify()?;

    // 2. Verify binding links to expected deposit
    if binding.deposit_pubkey != *expected_deposit {
        return Err(Error::WrongDeposit);
    }

    // 3. Verify invoice matches binding
    if invoice.payment_hash() != &binding.invoice_payment_hash {
        return Err(Error::InvoiceMismatch);
    }

    // 4. Verify partner cosigned the binding
    let cosign_msg = [
        invoice.payment_hash(),
        &sha256(&serialize(binding))[..],
    ].concat();
    verify_signature(&cosign_msg, partner_cosig, &partner_pubkey)?;

    // 5. Verify not expired
    if binding.expires_at < current_time() {
        return Err(Error::BindingExpired);
    }

    Ok(())
}
```

**Why This Matters:**

Without binding, operators could create invoice I1 for deposit X (get cosignature), then give users different invoice I2. User pays I2, operator pockets funds, and partner cannot prove fraud because I1 was never paid.

With binding:
- User's wallet rejects invoices without valid binding
- User's wallet rejects bindings without partner cosignature
- Partner can prove fraud: operator signed binding, payment settled, deposit not credited

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

## 8. Recovery Voting

After a force-close, partners vote on whether the operator was conforming or non-conforming. Voting threshold degrades over time to ensure a decision is always reached.

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

### Threshold Degradation

Voting threshold decreases over time to ensure a decision is always reached:

| Window | Threshold | Example (9 voters) |
|--------|-----------|-------------------|
| Day 1-3 | Majority | 5-of-9 |
| Day 3-7 | 40% | 4-of-9 |
| Day 7-14 | 30% | 3-of-9 |
| Day 14-21 | 20% | 2-of-9 |
| Day 21+ | Any single voter | 1-of-9 |

This ensures:
- **Fast resolution** if peers are online and agree
- **Slow but guaranteed resolution** if peers are split or offline
- **Decision always reached** - eventually one voter is enough

### Recovery Coordination

```
Force Close (Block N)
        │
        ▼
┌──────────────────────────┐
│ Peer Notification        │ Force-closing peer broadcasts
│                          │ to all partners/voters
└───────────┬──────────────┘
            │
            ▼
┌──────────────────────────┐
│ Ledger Synchronization   │ Peers share final signed
│                          │ ledger state, resolve any gaps
└───────────┬──────────────┘
            │
            ▼
┌──────────────────────────┐
│ Independent Validation   │ Each peer validates ledger
│                          │ against protocol rules
└───────────┬──────────────┘
            │
            ▼
┌──────────────────────────┐
│ Voting                   │ Peers broadcast RecoveryVote
│                          │ Threshold degrades over time
└───────────┬──────────────┘
            │
    ┌───────┴───────┐
    ▼               ▼
Conforming     Non-Conforming
```

### Threshold Implementation

```rust
impl RecoveryState {
    /// Check if conformance threshold is met given current block
    pub fn check_conformance_threshold(&self, current_block: u32) -> Option<bool> {
        let blocks_elapsed = current_block - self.force_close_block;
        let threshold = self.threshold_for_block(blocks_elapsed);

        let conforming_votes = self.votes.values().filter(|v| v.is_conforming).count();
        let non_conforming_votes = self.votes.values().filter(|v| !v.is_conforming).count();

        if conforming_votes >= threshold {
            Some(true)
        } else if non_conforming_votes >= threshold {
            Some(false)
        } else {
            None // No decision yet
        }
    }

    /// Get voting threshold for elapsed blocks
    fn threshold_for_block(&self, blocks_elapsed: u32) -> usize {
        let n = self.voter_set.len();
        match blocks_elapsed {
            0..=431 => (n / 2) + 1,      // Day 1-3: majority
            432..=1007 => (n * 2 / 5),   // Day 3-7: 40%
            1008..=2015 => (n * 3 / 10), // Day 7-14: 30%
            2016..=3023 => (n / 5),      // Day 14-21: 20%
            _ => 1,                       // Day 21+: any single voter
        }.max(1)
    }
}
```

### Evaluation Guidelines

Voters evaluate conformance based on cryptographic evidence. These guidelines establish norms for consistent evaluation:

**Primary Evidence (Dispositive):**

| Evidence Type | Meaning | Vote |
|---------------|---------|------|
| Ledger hash mismatch | On-chain hash ≠ signed ledger hash | Non-Conforming |
| Missing updates | Gap in sequence numbers with valid signatures | Non-Conforming |
| Invalid signatures | Updates signed by wrong key or corrupted | Non-Conforming |
| Balance violation | Deposit balance went negative | Non-Conforming |
| Uncredited payment | Preimage exists, no credit in ledger | Non-Conforming |
| Clean ledger | All hashes match, all signatures valid | Conforming |

Primary evidence is cryptographically verifiable. A hash mismatch is proof – not an accusation requiring interpretation.

**Contextual Evidence (Not Dispositive):**

| Evidence Type | Interpretation |
|---------------|----------------|
| Capital injection during investigation | Suspicious timing – increases fraud probability |
| Large withdrawal before force-close | May indicate preparation, or normal operations |
| Partner churn in preceding weeks | May indicate Sybil rotation, or business changes |
| Operator silence during grace period | May indicate guilt, or operational issues |

Contextual evidence informs but does not determine votes. An operator who injects 500k during the commitment window looks like they're trying to influence voting – but the vote still depends on whether primary evidence shows conformance.

**Social Engineering Resistance:**

Operators facing non-conformance determination may attempt to sway voters:

```
Attack patterns:
- "I'm fixing an honest mistake" (capital injection during investigation)
- "The accuser is lying" (without addressing cryptographic proof)
- "I'll make you whole privately" (bribery)
- "You'll lose your attestation income" (economic pressure)
```

**Defense:** Primary evidence is dispositive. If the hash doesn't match, the operator is non-conforming regardless of explanations. Voters who ignore cryptographic proof to vote "Conforming" become liable via transitive liability (System 10).

**Voting Checklist:**

```
□ Verify ledger hash on-chain matches signed ledger updates
□ Verify all signatures are valid and from correct keys
□ Verify sequence numbers are continuous with no gaps
□ Verify all balance changes have corresponding operations
□ Check for uncredited payment accusations (preimage proof)
□ Ignore capital movements and timing as primary factors
□ Vote based on cryptographic evidence, not explanations
```

---

## 9. Recovery Claims

Once voting determines conformance (System 8), funds are distributed based on the outcome.

**If Conforming**: Funds return to operator via cooperative spend.

**If Non-Conforming**: Funds go to a randomly selected peer who becomes the new operator. The claim process uses sequential random selection with fallback if selected peers don't claim.

### Sequential Random Selection

```
Round 1 (blocks N+6 to N+150):
  - Entropy: SHA256(block[N+6].hash || ledger_id)
  - Select peer A from voter set
  - Peer A has exclusive window to claim

Round 2 (blocks N+150 to N+294):
  - If A didn't claim, exclude A from set
  - Entropy: SHA256(block[N+150].hash || ledger_id)
  - Select peer B from remaining voters
  - Peer B has exclusive window to claim

Round 3, 4, ... N:
  - Continue until a peer claims or all peers exhausted

Final Fallback (after all peers exhausted):
  - Any original voter, the channel partner, or the operator can claim
  - Funds never burn
```

Each round uses fresh entropy from **three consecutive block hashes**, keeping selection unpredictable until all three blocks are mined while adding defensive depth against miner influence.

### Why Sequential Selection?

1. **Prevents frontrunning**: Can't predict who's selected until entropy blocks exist
2. **Fair opportunity**: Each peer gets exclusive window in random order
3. **Handles unavailability**: Offline peers are skipped, not blocking
4. **Never burns**: Eventually opens to anyone with standing

### Multi-Block Entropy

Using three consecutive block hashes instead of one adds defensive depth:

```
entropy = SHA256(block_hash[N] || block_hash[N+1] || block_hash[N+2] || ledger_hash)
```

**Why this matters:** While single-block entropy is already secure (grinding costs exceed claim values), multi-block entropy makes theoretical manipulation exponentially harder. To influence the combined hash of three blocks, a miner would need to:
- Mine all three blocks (probability ~1/3^3 = 1/27 for a miner with 33% hashrate)
- Sacrifice block rewards for each orphaned attempt (~6.25 BTC × 3)
- Complete grinding before honest blocks advance

This is defense in depth: the single-block mechanism already works economically, but multi-block removes even theoretical concerns.

### Claim Implementation

```rust
const ENTROPY_BLOCKS: u32 = 3;  // Use 3 consecutive blocks

pub struct ClaimRound {
    pub round_number: u32,
    pub start_block: u32,
    pub entropy_blocks: [[u8; 32]; 3],  // Three consecutive block hashes
    pub selected_peer: PublicKey,
    pub excluded_peers: Vec<PublicKey>,
    pub claimed: bool,
}

impl RecoveryState {
    /// Select peer for current claim round using multi-block entropy
    pub fn select_peer_for_round(
        &self,
        round: u32,
        entropy_blocks: [[u8; 32]; 3],
    ) -> PublicKey {
        let available: Vec<_> = self.voter_set.iter()
            .filter(|p| !self.is_excluded(p, round))
            .collect();

        // Combine three block hashes + ledger hash for entropy
        let combined = [
            &entropy_blocks[0][..],
            &entropy_blocks[1][..],
            &entropy_blocks[2][..],
            &self.ledger_hash[..],
        ].concat();
        let entropy = sha256(&combined);
        let index = u64::from_le_bytes(entropy[0..8].try_into().unwrap()) as usize;

        *available[index % available.len()]
    }
}

pub enum ClaimEligibility {
    /// Specific peer's exclusive window
    SelectedPeer { peer: PublicKey, window_ends: u32 },
    /// All peers exhausted - anyone with standing can claim
    OpenClaim,
}
```

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

---

## 10. Transitive Liability

Collusion between an operator and their partners is visible and provable. The protocol leverages this transparency to create **transitive liability**: evidence of collusion puts at risk not only the direct participants, but also their partners, and partners of those partners.

### Why Transitive Liability?

The basic collateral model (System 2) prevents profitable collusion with a *single* partner. But what if an operator coordinates with *multiple* partners simultaneously? Without transitive liability, an operator colluding with 2 of 3 partners could profit by stealing from both colluding channels while only losing collateral in the non-colluding channel.

Transitive liability closes this gap by making collusion evidence *contagious* through the partner graph.

### Evidence Types

The following constitute provable collusion evidence:

| Evidence | Source | Proves |
|----------|--------|--------|
| **Ledger/State Mismatch** | Force-close reserves vs. ledger hash | Operator and partner agreed to invalid state |
| **Uncredited Preimage** | Payer/wallet holds preimage for uncredited payment | Operator pocketed payment, partner failed to detect/report |
| **Dishonest Recovery Vote** | Vote contradicts provable ledger state | Voter protecting dishonest operator |
| **Continued Operations** | New ledger updates after evidence published | Partner knowingly operating with proven cheater |

### Liability Propagation

When collusion evidence emerges against an operator-partner pair:

```
Phase 1: Direct Liability
┌─────────────────────────────────────────────────────┐
│  Alice (operator) ←──collusion──→ Bob (partner)    │
│                                                     │
│  Evidence: Ledger hash on-chain doesn't match      │
│            signed ledger updates                    │
│                                                     │
│  Result: Both Alice and Bob are marked DISHONEST   │
│          All their reserves in all channels at risk│
└─────────────────────────────────────────────────────┘

Phase 2: Partner Obligation
┌─────────────────────────────────────────────────────┐
│  Eve ←────channel────→ Bob (DISHONEST)             │
│                                                     │
│  Eve MUST within N blocks:                         │
│    1. Force-close channel with Bob                 │
│    2. Initiate recovery proceedings                │
│    3. Broadcast acknowledgment of Bob's dishonesty │
│                                                     │
│  If Eve continues operations with Bob:             │
│    → Eve becomes COMPLICIT                         │
│    → Eve's other partners may close with Eve       │
└─────────────────────────────────────────────────────┘

Phase 3: Contagion Boundary
┌─────────────────────────────────────────────────────┐
│  Frank ←────channel────→ Eve                       │
│                                                     │
│  If Eve is marked COMPLICIT:                       │
│    Frank MUST close with Eve or become complicit   │
│                                                     │
│  Contagion stops when:                             │
│    - Partner closes channel (acted honestly)       │
│    - Grace period expires without new evidence     │
│    - Liability claim is adjudicated                │
└─────────────────────────────────────────────────────┘
```

### Collusion Attack Revisited

**Without transitive liability:**
```
Alice operates: Alice→Bob (60k), Alice→Charlie (50k), Alice→David (40k)
Attack: Alice + Bob + Charlie collude

Stolen: 60k + 50k = 110k
Slashed: 30k (David's channel only)
Net profit: 80k ← Attack is profitable
```

**With transitive liability:**
```
Alice operates: Alice→Bob (60k), Alice→Charlie (50k), Alice→David (40k)
Bob operates: Bob→Eve (40k), Bob→Frank (30k)
Charlie operates: Charlie→Grace (35k), Charlie→Henry (25k)

Attack: Alice + Bob + Charlie collude

Phase 1 - Direct:
  Stolen: 110k
  Alice slashed by David: 30k

Phase 2 - Partner obligation:
  Eve sees Bob is DISHONEST → closes Bob→Eve → slashes Bob's excess
  Frank sees Bob is DISHONEST → closes Bob→Frank → slashes Bob's excess
  Grace sees Charlie is DISHONEST → closes Charlie→Grace → slashes Charlie's excess
  Henry sees Charlie is DISHONEST → closes Charlie→Henry → slashes Charlie's excess

Total slashed: 30k (David) + Bob's excess in Eve+Frank + Charlie's excess in Grace+Henry

If Bob and Charlie each had 50k excess across their other channels:
  Total slashed: 30k + 50k + 50k = 130k
  Net: 110k stolen - 130k slashed = -20k ← Attack is unprofitable
```

### Sybil Resistance

An attacker might try to avoid transitive liability by using Sybil identities for all partners. But each Sybil partner requires:

1. **Real capital** locked as reserves (can't fake channel balance)
2. **Real collateral** attested by other partners (or more Sybils, requiring more capital)

**The Sybil limit**: An attacker can create unlimited identities but not unlimited capital. To create a collusion ring of N partners with total deposits D:

- Minimum capital required: `D × 2 × N` (each Sybil needs 200% collateral)
- Capital at risk if caught: All of it
- Profit from attack: D (the deposits)

For N ≥ 2, the attack is capital-inefficient. The attacker must lock more than they can steal.

### Implementation

```rust
pub struct CollusionEvidence {
    pub evidence_type: EvidenceType,
    pub operator: PublicKey,
    pub partner: PublicKey,
    pub proof: Vec<u8>,              // Type-specific proof data
    pub block_height: u32,           // When evidence was created
    pub signatures: Vec<[u8; 64]>,   // Witnesses who verified
}

pub enum EvidenceType {
    LedgerStateMismatch {
        on_chain_hash: [u8; 32],
        signed_ledger_hash: [u8; 32],
        ledger_updates: Vec<SignedLedgerUpdate>,
    },
    UncreditedPayment {
        payment_hash: [u8; 32],
        preimage: [u8; 32],
        invoice_cosignature: [u8; 64],
    },
    DishonestVote {
        vote: RecoveryVoteMsg,
        contradicting_evidence: Box<CollusionEvidence>,
    },
    ContinuedOperations {
        dishonesty_evidence: Box<CollusionEvidence>,
        subsequent_updates: Vec<SignedLedgerUpdate>,
    },
}

pub struct LiabilityStatus {
    pub node: PublicKey,
    pub status: NodeStatus,
    pub evidence: Option<CollusionEvidence>,
    pub grace_period_expires: Option<u32>,  // Block height
}

pub enum NodeStatus {
    Honest,
    Dishonest { evidence: CollusionEvidence },
    Complicit { failed_to_act_on: CollusionEvidence },
    Cleared { was: Box<NodeStatus>, cleared_at: u32 },
}
```

### Grace Period and Appeals

Partners have a grace period (e.g., 144 blocks / ~1 day) to:
1. Detect published collusion evidence
2. Close channels with dishonest partners
3. Broadcast their own acknowledgment

This prevents instant contagion from catching slow-to-react partners. It also allows time for evidence to be contested if it's fraudulent.

### Two-Stage Capital Locks

The grace period creates a timing vulnerability: sophisticated operators can drain capital between knowing fraud will be detected and evidence being published. Two-stage capital locks close this gap.

**Stage 1: Soft Lock (Monitoring Mode)**

```rust
pub enum SoftLockTrigger {
    RejectMessage { from: PublicKey, reason: String },
    PartnerForceClose { partner: PublicKey },
    RoutingSpike { volume_ratio: f64 },  // >2x 7-day average for >6 hours
}

pub struct SoftLock {
    pub operator: PublicKey,
    pub trigger: SoftLockTrigger,
    pub triggered_at: u32,
    pub justification_deadline: u32,  // triggered_at + 144
    pub justification: Option<SignedJustification>,
}
```

**Soft lock effects:**
- No hard freeze (operator can still move funds)
- Operator MUST broadcast signed justification within 144 blocks
- All partners notified of soft lock state
- If no justification or suspicious activity continues → escalates to hard lock

**Stage 2: Hard Lock (Freeze Mode)**

```rust
pub enum HardLockTrigger {
    EvidencePublished { evidence: CollusionEvidence },
    SoftLockEscalation { soft_lock: SoftLock },
    MultipleForceCloses { count: u32, window_blocks: u32 },  // >1 in 144 blocks
}

pub struct HardLock {
    pub operator: PublicKey,
    pub trigger: HardLockTrigger,
    pub triggered_at: u32,
    pub expires_at: u32,  // triggered_at + 2016
    pub locked_channels: Vec<ChannelId>,
    pub locked_amount: u64,  // ALL reserves, not just excess
}
```

**Hard lock effects:**
- ALL reserves frozen in operator's channels (not just excess above 150%)
- Duration: 2016 blocks (~2 weeks)
- Can only be unlocked by:
  - Conformance proven via recovery voting
  - 2016 blocks expire with no recovery initiated
  - Operator proves evidence was false

**Why ALL reserves, not just excess?**

If only excess locks, operators game the definition: drain "required" reserves (150%) while "excess" is frozen. Locking all reserves prevents this arbitrage during investigation.

### Collusion Evidence Broadcasting

When evidence is created, it's broadcast to:
1. All channel partners of the accused
2. All collateral partners (voters)
3. Known auditors and watchtowers

Recipients validate the evidence independently and decide whether to act. Invalid evidence (bad signatures, incorrect proofs) is ignored and the broadcaster may be penalized for spam.

---

## 11. Collateral Transparency

Transitive liability (System 10) ensures that collusion puts capital at risk across the partner graph. But this assumes excess collateral exists to slash. Without visibility into collateral changes, an operator could gradually withdraw excess over weeks, then execute theft with minimal slashable capital remaining.

System 11 makes collateral changes **visible**, enabling markets to respond.

### Rate-Limited Collateral Reduction

Collateral reductions are rate-limited to prevent "boiling frog" attacks where operators gradually drain coverage faster than markets can respond.

```rust
const LOCK_PERIOD: u32 = 2016;                    // ~2 weeks per reduction
const MAX_REDUCTION_BPS: u16 = 500;               // 5% max per period
const LOOKBACK_WINDOW: u32 = 2016 * 13;           // 26 weeks
const MAX_CUMULATIVE_REDUCTION_BPS: u16 = 3000;   // 30% max in lookback window
const ABSOLUTE_MINIMUM_COVERAGE_BPS: u16 = 15000; // 150% floor

pub struct ReduceNotice {
    pub operator: PublicKey,
    pub channel_id: ChannelId,
    pub reduction_amount: u64,
    pub reduction_bps: u16,        // Percentage of current coverage
    pub effective_block: u32,      // current_block + LOCK_PERIOD
    pub reason: ReductionReason,
    pub signature: [u8; 64],
}

pub fn validate_reduction(
    notice: &ReduceNotice,
    operator_history: &ReductionHistory,
    current_coverage_bps: u16,
) -> Result<(), ReductionError> {
    // Check rate limit: max 5% per 2016 blocks
    if notice.reduction_bps > MAX_REDUCTION_BPS {
        return Err(ReductionError::ExceedsRateLimit {
            proposed_bps: notice.reduction_bps,
            max_bps: MAX_REDUCTION_BPS,
        });
    }

    // Check absolute floor: cannot go below 150%
    let post_reduction_bps = current_coverage_bps - notice.reduction_bps;
    if post_reduction_bps < ABSOLUTE_MINIMUM_COVERAGE_BPS {
        return Err(ReductionError::BelowAbsoluteMinimum {
            proposed_bps: post_reduction_bps,
            minimum_bps: ABSOLUTE_MINIMUM_COVERAGE_BPS,
        });
    }

    // Check lookback window: max 30% cumulative in 26 weeks
    let cumulative = operator_history.cumulative_reduction_bps(LOOKBACK_WINDOW);
    if cumulative + notice.reduction_bps > MAX_CUMULATIVE_REDUCTION_BPS {
        return Err(ReductionError::ExceedsLookbackLimit {
            cumulative_bps: cumulative,
            proposed_bps: notice.reduction_bps,
            max_cumulative_bps: MAX_CUMULATIVE_REDUCTION_BPS,
        });
    }

    Ok(())
}
```

**Graduated Schedule for New Operators:**

| Operator Age | Max Reduction per 2016 blocks |
|--------------|-------------------------------|
| Weeks 1-12 | 0% (locked at initial coverage) |
| Weeks 13-26 | 2% |
| Week 26+ | 5% |

This prevents bait-and-switch: new operators cannot attract deposits at 300% coverage then immediately reduce to 150%.

**Emergency Override:**

Operators can request reductions exceeding the 5% limit:
- Requires explicit approval from 2/3 of partners
- Partners must sign acknowledgment of risk increase
- Emergency reductions count double toward lookback window

**Why 5%?**

| Rate | Time 300%→150% | Market Response |
|------|----------------|-----------------|
| 10%/2016 | 14 weeks | Insufficient |
| 5%/2016 | 30 weeks | Sufficient |
| 2%/2016 | 50 weeks | Excessive |

5% provides 7+ months for market discipline to respond while allowing legitimate operational flexibility.

### Partner Response to Reduction Notices

Partners receiving `ReduceNotice` have `LOCK_PERIOD` blocks to decide:

1. **Accept**: Take no action; reduction proceeds after timeout
2. **Exit**: Force-close channel before reduction takes effect

Partners evaluate: Does this reduction, combined with recent history and overall trends, indicate attack preparation or legitimate operations?

### Health Broadcasting

Operators broadcast signed health metrics at regular intervals:

```rust
pub struct HealthBroadcast {
    pub operator: PublicKey,
    pub block_height: u32,
    pub total_deposits: u64,
    pub total_reserves: u64,
    pub coverage_ratio: u16,      // basis points (10000 = 100%)
    pub attestation_count: u32,
    pub total_attested: u64,
    pub channel_count: u32,
    pub circular_attestation_ratio: u16,
    pub signature: [u8; 64],
}

const HEALTH_INTERVAL: u32 = 144;  // ~daily
```

Partners and users monitor these broadcasts to detect concerning patterns:
- Declining coverage ratio over time
- Increasing circular attestation
- Stale broadcasts (operator may be offline)

### Cross-Channel Visibility

`ReduceNotice` and `HealthBroadcast` messages are shared across all of an operator's partners, not just the affected channel:

```
Operator reduction notice on Channel A
        │
        ├──► Partner A (directly affected)
        ├──► Partner B (other channel - sees pattern)
        ├──► Partner C (other channel - sees pattern)
        └──► Collateral partners (track global health)
```

If an operator issues `ReduceNotice` on 9 of 10 channels while maintaining the 10th (colluding) channel, all partners see this pattern and can respond.

### Partner Churn Tracking

Sybil partner rotation—where an operator controls multiple partners through Sybil identities and rotates them to hide the pattern—is detected through churn tracking.

```rust
pub struct ChurnMetrics {
    pub operator: PublicKey,
    pub measurement_block: u32,

    // Churn rates (basis points)
    pub churn_30d_bps: u16,
    pub churn_90d_bps: u16,
    pub churn_180d_bps: u16,

    // Partner tenure
    pub partners: Vec<PartnerTenure>,
    pub average_tenure_blocks: u32,

    // Stability score (0-100)
    pub stability_score: u8,

    pub signature: [u8; 64],
}

pub struct PartnerTenure {
    pub partner: PublicKey,
    pub joined_block: u32,
    pub tenure_blocks: u32,
}

impl ChurnMetrics {
    pub fn calculate_churn_rate(&self, window_blocks: u32) -> u16 {
        let partners_changed = self.count_partners_changed(window_blocks);
        let average_partners = self.average_partner_count(window_blocks);

        if average_partners == 0 { return 0; }
        ((partners_changed * 10000) / average_partners) as u16
    }

    pub fn stability_score(&self) -> u8 {
        // 100 = excellent stability, 0 = high churn
        let base = 100u8;
        let churn_penalty = (self.churn_90d_bps / 100) as u8;  // -1 per 1% churn
        let tenure_bonus = (self.average_tenure_blocks / 1008) as u8;  // +1 per week avg tenure

        base.saturating_sub(churn_penalty).saturating_add(tenure_bonus.min(20))
    }
}
```

**Churn Indicators:**

| Churn Level | 90-Day Rate | Indicator | Interpretation |
|-------------|-------------|-----------|----------------|
| Excellent | <10% | Green | Stable partner relationships |
| Acceptable | 10-30% | Yellow | Normal business changes |
| Concerning | 30-50% | Orange | Monitor closely |
| Dangerous | >50% | Red | Possible Sybil rotation |

**Why This Detects Sybil Rotation:**

An attacker rotating 6 Sybil partners over 9 months produces ~60% churn—an obvious red flag. Each individual rotation looks innocent, but the pattern is visible through aggregate metrics.

### Market-Determined Collateralization

The protocol does not mandate specific collateral ratios. Instead, it makes parameters transparent:

- **Coverage ratio**: Total attestations / total deposit liability
- **Partner count**: Number of active channel partners
- **Circular attestation**: Percentage from reciprocal relationships
- **Health trend**: Historical coverage over time
- **Reduction history**: Recent `ReduceNotice` broadcasts

Wallets display this information, enabling users to choose operators matching their risk preferences:

| User Priority | Operator Choice |
|---------------|-----------------|
| Maximum security | High coverage (300%+), many partners, low circular |
| Balance | Standard coverage (200%), established track record |
| Low fees | Lower coverage, accepts more risk |

This market approach allows:
- **New operators** to enter by offering higher collateral
- **Established operators** to compete on reputation
- **Users** to make informed decisions based on transparent metrics
- **The market** to discover equilibrium collateralization levels

### Why Minimums + Visibility Works

**The attacker's dilemma:**
- Withdraw quickly → Rate limits block (max 5%/2016 blocks)
- Withdraw slowly → 26-week lookback caps cumulative reduction at 30%
- Drain during grace period → Two-stage capital locks freeze reserves
- Build circular attestation → Hard 40% limit blocks
- Rotate Sybil partners → Churn tracking exposes pattern
- Attack with full collateral → Transitive liability makes theft unprofitable

**The synthesis:** Enforce minimums that protect core game theory. Display everything else transparently. Trust markets within constrained bounds.

---

## 12. Security Model and Capital Requirements

The protocol's security properties depend on measurable parameters. This section makes the relationship between capital, partners, and security levels explicit.

### Capital Requirements Formula

The claim "to steal X you need 2X locked" is a simplification that assumes partners don't coordinate theft. The general formula for protection against K colluding partners out of N total:

```
Required_Coverage = 100% + (K / (N - K)) × 100%
```

### Protection Level Examples

| Total Partners (N) | Colluding Partners (K) | Required Coverage | Protection Level |
|-------------------|----------------------|------------------|-------------------|
| 3 | 1 | 150% | 1-of-3 collusion |
| 3 | 2 | 300% | 2-of-3 collusion |
| 5 | 1 | 125% | 1-of-5 collusion |
| 5 | 2 | 167% | 2-of-5 collusion |
| 10 | 2 | 125% | 20% collusion |
| 10 | 5 | 200% | 50% collusion |
| 10 | 8 | 500% | 80% collusion |

**Key insight:** Capital efficiency (using same collateral for multiple channels) trades off against protection level. Higher partner counts enable lower per-channel requirements for the same security.

### Transitive Liability Amplification

System 10 extends slashing beyond direct collateral. When collusion is detected:

```
Total_Capital_At_Risk = Direct_Collateral + Σ(colluding_partner[i].other_channel_reserves)
```

**Example:**

Operator Alice (10 channels, 100k deposits each, 200% coverage = 200k excess):
- Colludes with Bob (5 other channels, 50k at risk)
- Colludes with Charlie (3 other channels, 30k at risk)

**Capital at risk if caught:** 200k + 50k + 30k = **280k**
**Maximum theft:** 200k (Bob and Charlie's channels)
**Net if caught:** -80k

Transitive liability makes sophisticated multi-party collusion increasingly unprofitable.

### Operator Security Disclosure

Operators must publish machine-readable security parameters:

```rust
pub struct OperatorSecurityParams {
    pub operator_id: PublicKey,
    pub total_deposits: u64,
    pub total_reserves: u64,
    pub total_attested_collateral: u64,
    pub coverage_ratio_bps: u16,

    // Security level disclosures
    pub channel_count: u32,
    pub partner_count: u32,
    pub collateral_partner_count: u32,
    pub protected_against_k_colluding: u32,
    pub transitive_liability_depth: u32,

    // Quality metrics
    pub partner_avg_coverage_ratio_bps: u16,
    pub partner_avg_tenure_blocks: u32,
    pub operator_track_record_blocks: u32,  // Time without slashing
    pub partner_churn_rate_bps: u16,

    pub signature: [u8; 64],
}
```

### User Security Tiers

Wallets display operator security in user-friendly tiers:

**Tier 1 - Maximum Security:**
- Coverage: 300%+
- Partners: 10+
- Protected against: Majority collusion
- Track record: 1+ years without incidents
- Partner churn: <10%

**Tier 2 - High Security:**
- Coverage: 200-300%
- Partners: 5-10
- Protected against: Minority collusion (2-4 partners)
- Track record: 6+ months
- Partner churn: <20%

**Tier 3 - Standard Security:**
- Coverage: 150-200%
- Partners: 3-5
- Protected against: Single partner collusion
- Track record: 3+ months
- Partner churn: <30%

**Tier 4 - Basic Security:**
- Coverage: 100-150%
- Partners: 1-3
- Protected against: None (trust-based)
- New operator or recovery situation

### Explicit Security Assumptions

The protocol's security properties hold given:

1. **Partners are independent, not coordinated** - The Mexican standoff works because each partner fears the others will defect. Collusion requires trust among thieves; the protocol assumes they can't achieve it at scale.
2. **Partners are self-interested** - They slash because it protects their capital, not because they're altruistic. Greed and fear, not ethics.
3. **At least one partner remains available** during recovery
4. **Cryptographic primitives are secure** (SHA256, ECDSA, Schnorr)
5. **Transitive liability propagates** through the partner graph
6. **Market disciplines operators** with poor track records or high churn
7. **Bitcoin consensus remains secure** (51% honest mining)

**Why partners don't collude:**

Collusion requires coordinated defection. But each potential colluder faces the prisoner's dilemma:
- If I collude and others defect, I get slashed
- If I defect and others collude, I get to slash them
- Defection is the dominant strategy

The protocol doesn't need partners to be good. It needs them to be rational, independent, and unable to trust each other enough to coordinate theft.

### What This Means For Users

| If you want... | Choose... |
|----------------|-----------|
| Maximum security, willing to pay higher fees | Tier 1 operator (300%+, 10+ partners) |
| Good security at reasonable cost | Tier 2-3 operator (200%, 5+ partners) |
| Low fees, trust specific operator | Tier 4 operator (150%, known entity) |
| To evaluate yourself | Check operator's `OperatorSecurityParams` directly |

**The protocol makes tradeoffs visible rather than hidden.** Users choose their risk level based on transparent metrics, not promises.

### Comparison to Alternatives

| Property | Pure Custody | Deposits Protocol | Self-Custody |
|----------|-------------|-------------------|--------------|
| Theft possible? | Yes (trust operator) | Economically irrational | No |
| Fund recovery | Trust operator | Cryptographic guarantee | User responsibility |
| Transparency | None | Full parameters visible | N/A |
| Complexity | Low | Medium | High |
| Who it's for | Trust-based | Trust-minimized | Self-sovereign |

The protocol occupies the middle ground: meaningful security improvements over pure custody for users who can't or won't run their own Lightning infrastructure.

---

## Supplementary Mechanisms

Several mechanisms from **SUPPLEMENTARY.md** have been incorporated into the core protocol in modified form:

| Original Supplement | Core Protocol Version |
|--------------------|-----------------------|
| S1. Circular Enforcement (40%/60%) | **Hard 40% limit** (System 4) |
| S2. Pre-Commitment | **Two-stage capital locks** (System 10) - simpler than commit-reveal |
| S3. Four-Layer Defense | **5% rate limits + 26-week lookback + churn tracking** (System 11) |

The SUPPLEMENTARY.md file retains the original, more complex versions for reference. The core protocol implements targeted minimums that protect game theory without the full complexity of the supplementary mechanisms.

**Key insight from adversarial review:** Pure visibility was insufficient; markets need infrastructure (timing guarantees, rate limits, structural constraints) to enforce effectively.
