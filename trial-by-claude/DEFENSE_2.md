# DEFENSE - Round 2 Response

## Addressing The Sustained Attacks

The JUDGE sustained two attacks and requested specific responses. I provide both explanations of existing protections AND proposed protocol enhancements to eliminate any ambiguity.

---

## Question 1: Ledger Equivocation Prevention

**JUDGE's Question:** How does the protocol prevent an operator from maintaining parallel valid ledger chains and selectively committing one during force-close?

### Current Protocol Defense: Partner Ledger Validation

The protocol ALREADY contains mechanisms that prevent ledger equivocation, though they may not have been sufficiently emphasized in the whitepaper.

**Key Mechanism: UpdateReserves Message with Bidirectional Hash Verification**

From WHITEPAPER.md lines 328-340:

```rust
pub struct UpdateReserves {
    pub channel_id: ChannelId,
    pub reserves_sats: u64,
    pub custodian_pubkey: PublicKey,
    pub ledger_hash: [u8; 32],        // Sender's operator ledger hash
    pub remote_ledger_hash: [u8; 32], // Sender's view of peer's ledger hash
}
```

**How This Prevents Equivocation:**

1. **Every commitment transaction update includes the ledger hash**
   - Operator sends `UpdateReserves` with current `ledger_hash`
   - Partner validates: "does this hash match MY copy of the operator's ledger?"
   - Partner responds with `AcceptReserves` ONLY if hashes match
   - New commitment transaction is signed with this agreed-upon hash

2. **Partner must sign the commitment transaction**
   - Commitment tx includes reserves output with embedded `ledger_hash`
   - Partner's signature on commitment tx = attestation that this hash is correct
   - If operator tries to force-close with Chain B, they need a commitment tx with Chain B's hash
   - But partner never signed a commitment with Chain B's hash

3. **Equivocation requires forged partner signature**
   - To force-close with a parallel chain, operator needs a valid commitment tx
   - Valid commitment tx requires partner's signature
   - Partner only signs commitments with ledger hashes they've validated
   - Operator cannot forge partner's signature (requires partner's private key)

**The Attack Fails:**

- Operator maintains Chain A (shown to partner) with hash `0xAAAA...`
- Operator maintains Chain B (hidden) with hash `0xBBBB...`
- Partner validates Chain A, signs commitments with `ledger_hash = 0xAAAA...`
- Operator wants to force-close with Chain B
- But operator only has valid commitment txs with `ledger_hash = 0xAAAA...`
- Broadcasting commitment with Chain A's hash while claiming Chain B is invalid is immediately detectable
- On-chain hash = `0xAAAA...`, but operator has no way to claim Chain B is canonical

**Where The Confusion Arose:**

The prosecution's attack assumed the operator could unilaterally commit ANY hash on-chain during force-close. But this misunderstands Lightning's commitment transaction structure:

- Commitment txs are pre-signed by BOTH parties
- The reserves output script is part of what's being signed
- Partner won't sign a commitment with a ledger hash they haven't validated
- Operator cannot create a valid commitment tx without partner cooperation

**Existing Protocol Protects Against This:**

The bidirectional ledger sync (System 1) + commitment transaction co-signing (Lightning's base security) + UpdateReserves validation = equivocation prevention.

### Proposed Enhancement: Explicit Sequence Number Co-Signatures

While the existing protocol prevents equivocation, we can make this protection MORE explicit and robust:

**Addition to Protocol:**

Each `SignedLedgerUpdate` includes BOTH operator and partner signatures:

```rust
pub struct SignedLedgerUpdate {
    pub message: Vec<u8>,
    pub message_type: u16,
    pub operator_signature: [u8; 64],
    pub operator_pubkey: PublicKey,
    pub partner_pubkey: PublicKey,
    pub sequence_number: u64,
    pub previous_state_hash: [u8; 32],
    pub current_state_hash: [u8; 32],
    pub timestamp: u64,

    // NEW: Partner attestation
    pub partner_signature: Option<[u8; 64]>,  // Partner signs (sequence_number || current_state_hash)
}
```

**How This Works:**

1. **Operator creates ledger update** (deposit, payment, etc.)
   - Computes new hash: `SHA256(sequence_number || previous_state_hash || message_bytes)`
   - Signs update with operator key

2. **Partner validates and co-signs**
   - Verifies update follows protocol rules
   - Verifies hash chain integrity
   - Signs attestation: `partner_sign(sequence_number || current_state_hash)`
   - Returns signature to operator

3. **Update becomes canonical**
   - Both signatures required for update to be considered valid
   - Operator stores signed update
   - Partner stores signed update
   - Both copies have identical signatures

4. **Equivocation becomes impossible**
   - To create parallel chains at sequence N, operator would need partner to sign BOTH hashes
   - Partner won't sign conflicting updates at the same sequence number
   - Attempting to present unsigned updates during recovery = obvious fraud

**Why This Is Better:**

- Makes partner's validation role explicit in the data structure
- Creates signed proof that partner acknowledged each update
- During recovery, voters can verify BOTH signatures on each update
- Any update lacking partner signature is automatically suspect

**Backward Compatibility:**

`partner_signature: Option<[u8; 64]>` allows gradual rollout:
- Old channels: continue with operator-only signatures + commitment validation
- New channels: require partner co-signatures for all updates
- Migration path: require partner signatures after a grace period

### Alternative Enhancement: Merkle Checkpoints

For channels with high update frequency, requiring partner signatures on EVERY update might create latency. Alternative approach:

**Periodic Merkle Commitments:**

Every N updates (e.g., every 144 updates = ~daily for active channels), partner signs a Merkle root covering the last N updates:

```rust
pub struct LedgerCheckpoint {
    pub operator: PublicKey,
    pub partner: PublicKey,
    pub sequence_range: (u64, u64),  // e.g., (0, 144)
    pub merkle_root: [u8; 32],       // Root of updates 0..144
    pub partner_signature: [u8; 64], // Partner attests to this root
    pub checkpoint_block: u32,       // Block height when created
}
```

**How This Prevents Equivocation:**

- Operator cannot equivocate on any update before the last checkpoint without breaking partner's signed Merkle root
- Equivocation window limited to updates since last checkpoint
- Checkpoints can be more frequent for higher security (every 10 updates, every update for maximum security)

### Summary: Equivocation Prevention

**Existing Protocol:** Already prevents equivocation through commitment transaction co-signing + `UpdateReserves` hash validation.

**Proposed Enhancement:** Add explicit partner co-signatures on updates OR Merkle checkpoint attestations to make this protection cryptographically explicit in the ledger structure itself.

**The attack as described DOES NOT WORK against the current protocol**, but the enhancement makes the protection more obvious and verifiable.

---

## Question 2: Sustained Coverage Withdrawal Detection

**JUDGE's Question:** What specific mechanisms detect coordinated gradual withdrawal through Sybil partner rotation over extended periods?

The JUDGE is correct that the existing specification is insufficient. I propose specific enhancements to System 11.

### Enhancement 1: Objective Objection Criteria

**Problem:** Partners can "object" to reductions but spec doesn't define when objections are valid.

**Solution:** Define objective, cryptographically verifiable objection criteria:

```rust
pub enum ObjectionReason {
    // OBJECTIVE CRITERIA (automatically enforceable)
    CoverageWouldDropBelowMinimum {
        current_coverage: u16,   // basis points
        post_reduction: u16,     // basis points
        minimum_allowed: u16,    // e.g., 15000 = 150%
    },

    ExcessiveReductionRate {
        reduction_amount: u64,
        window_days: u32,
        max_allowed_bps_per_day: u16,  // e.g., 30 = 0.3%/day
    },

    FrequentReductionPattern {
        notices_in_window: u32,   // e.g., 5 notices
        window_blocks: u32,       // in last 2016 blocks
        max_allowed: u32,         // e.g., 2 per 2016 blocks
    },

    PartnerChurnDetected {
        new_partners_added: u32,  // in last 90 days
        partners_removed: u32,    // in last 90 days
        churn_rate: u16,          // basis points
        max_allowed_churn: u16,   // e.g., 3000 = 30%
    },

    InsufficientJustification {
        reason_claimed: ReductionReason,
        evidence_provided: Vec<u8>,
        validation_failure: String,
    },
}
```

**How This Works:**

1. **Operator sends `ReduceNotice`** with reason (e.g., "Rebalancing")

2. **Partner evaluates against objective criteria:**
   - Query: Would this reduction drop coverage below 150%?
   - Query: Have there been >2 reductions in last 2016 blocks?
   - Query: Is partner churn rate >30% in last 90 days?
   - Query: Does reduction rate exceed 0.3%/day over any 30-day window?

3. **If ANY criterion triggers, objection is automatically valid:**
   - Partner broadcasts `ReductionObjection` with specific reason
   - Other partners can verify the objection using same criteria
   - Operator must either:
     - Withdraw the reduction notice, OR
     - Provide additional collateral attestations, OR
     - Accept that partner will force-close

4. **Dispute resolution is deterministic:**
   - "Coverage would drop to 145%" → Valid objection (below 150% threshold)
   - "I think this seems suspicious" → Invalid objection (subjective)
   - All partners compute same result from same data

### Enhancement 2: Cross-Channel Reduction Aggregation

**Problem:** Each partner sees local reductions but not global pattern.

**Solution:** Partners share reduction notices amongst themselves to detect aggregate patterns.

**Protocol Addition:**

```rust
pub struct AggregateReductionView {
    pub operator: PublicKey,
    pub reporting_partner: PublicKey,
    pub observation_window: (u32, u32),  // block range
    pub total_reduction_notices: u32,
    pub total_reduction_amount: u64,
    pub affected_channels: Vec<ChannelId>,
    pub partner_churn_events: Vec<PartnerChurnEvent>,
    pub signature: [u8; 64],
}

pub struct PartnerChurnEvent {
    pub event_type: ChurnType,  // Added or Removed
    pub partner_id: PublicKey,
    pub block_height: u32,
    pub channel_id: Option<ChannelId>,
}
```

**How This Works:**

1. **Each partner maintains a view of operator's global state:**
   - Subscribes to reduction notices across ALL of operator's channels
   - Subscribes to partner addition/removal events
   - Computes aggregate metrics over rolling windows

2. **Partner broadcasts aggregate view to other collateral partners:**
   - Every 1008 blocks (~weekly), each partner broadcasts their view
   - Other partners can cross-check: do all partners see the same pattern?
   - Discrepancies = evidence of selective disclosure

3. **Triggers on aggregate thresholds:**
   - If 5+ reduction notices across all channels in 30 days → Alert
   - If 30%+ partner churn in 90 days → Alert
   - If total reduction exceeds 20% of total coverage in 60 days → Alert

4. **Coordinated response:**
   - If multiple partners see same concerning pattern, they can coordinate exit
   - Reduces "everyone waits for someone else to act" problem

**Example Detection:**

Operator has 10 channels. Attack plan:
- Week 1-4: Sybil-1 sends reduction notice on Channel 1
- Week 5-8: Sybil-2 sends reduction notice on Channel 2
- Week 9-12: Sybil-3 sends reduction notice on Channel 3

**What Partners See:**

Partner A (Channel 1):
- Local: 1 reduction notice (seems fine)
- Aggregate view from Partner B: Shows 3 reduction notices across channels in 12 weeks
- Aggregate view from Partner C: Confirms 3 reduction notices
- **Trigger: FrequentReductionPattern (3 notices > 2 allowed per 2016 blocks)**
- Partner A objects to the reduction on Channel 1

**Attack fails because local decisions use global information.**

### Enhancement 3: Minimum Coverage Lock

**Problem:** Operator can reduce coverage to minimal levels before attack.

**Solution:** Enforce absolute minimum coverage that cannot be reduced without channel closure.

```rust
pub const ABSOLUTE_MINIMUM_COVERAGE_BPS: u16 = 15000;  // 150%
pub const OPERATIONAL_TARGET_COVERAGE_BPS: u16 = 20000; // 200%

pub struct CoverageLock {
    pub locked_until: u32,  // Block height
    pub locked_amount: u64, // Sats that cannot be withdrawn
    pub lock_reason: LockReason,
}

pub enum LockReason {
    InitialDeposit,      // Locked for min 12 weeks after deposit
    ReputationBuild,     // New operators locked longer
    SlashingReserve,     // Always maintain minimum buffer
}
```

**Policy:**

1. **Coverage can NEVER drop below 150%**
   - Reduction notices that would violate this are automatically rejected
   - No exceptions, no matter the reason
   - To reduce below 150%, operator must close channel (reducing deposits to zero)

2. **Reductions from 200% to 150% require extended notice:**
   - Standard reduction: 2016 blocks (~2 weeks)
   - Reduction below 175%: 4032 blocks (~4 weeks)
   - Reduction below 150%: Impossible (must close channel)

3. **New operators face stricter locks:**
   - First 12 weeks: Coverage locked at initial ratio (e.g., 250%)
   - Weeks 12-24: Can reduce to 200%
   - Weeks 24+: Can reduce to 150% (absolute minimum)

4. **Reputation-based policy:**
   - Operators with 0 slashing events for 1 year: More flexible reduction policy
   - Operators with ANY slashing event: Stricter locks for next 6 months
   - Market can choose operators based on track record

**Why This Prevents The Attack:**

The prosecution's attack requires reducing collateral to near-zero before theft. But:

- Cannot reduce below 150% without closing channels
- Closing channels = deposits must go to zero (users withdraw)
- Cannot execute "steal with minimal collateral" attack if 150% coverage is enforced

**Attack Modification:**

Attacker might try: "Steal 100k with 150k collateral"

But:
- 150k collateral comes from OTHER channels (attestations)
- Those channels also have deposits with 150% minimum
- To steal 100k, attacker needs 150k excess across N other channels
- If N = 3 channels, each has 50k excess, each probably has 33k deposits (150% coverage)
- Total capital needed: 100k (attack channel reserves) + 150k (collateral) + 99k (other channel deposits) = 349k
- Stolen: 100k
- Slashed if caught: 150k minimum (likely more with transitive liability)
- **Still economically irrational**

### Enhancement 4: Partner Stability Metrics

**Problem:** Sybil partner rotation is not tracked or penalized.

**Solution:** Make partner stability a visible metric that affects operator reputation.

```rust
pub struct PartnerStabilityScore {
    pub operator: PublicKey,
    pub measurement_window_blocks: u32,  // e.g., 26280 = ~6 months
    pub partners_at_start: Vec<PublicKey>,
    pub partners_at_end: Vec<PublicKey>,
    pub partners_added: Vec<(PublicKey, u32)>,    // pubkey, block
    pub partners_removed: Vec<(PublicKey, u32)>,  // pubkey, block
    pub churn_rate_bps: u16,             // basis points
    pub average_partner_tenure_blocks: u32,
    pub signature: [u8; 64],
}

pub fn calculate_partner_stability(
    events: &[PartnerChurnEvent],
    window_blocks: u32,
) -> PartnerStabilityScore {
    let added = events.iter().filter(|e| e.event_type == Added).count();
    let removed = events.iter().filter(|e| e.event_type == Removed).count();
    let average_partners = (start_count + end_count) / 2;
    let churn_rate_bps = ((added + removed) * 10000) / (average_partners * 2);
    // ... compute average tenure, etc.
}
```

**Policy:**

1. **Operators broadcast partner stability metrics monthly**
   - Included in `HealthBroadcast` (System 11)
   - Shows partner churn rate, average tenure

2. **Wallets display stability as a security indicator:**
   - Green: <10% partner churn per 6 months, avg tenure >6 months
   - Yellow: 10-30% churn, avg tenure 3-6 months
   - Red: >30% churn, avg tenure <3 months

3. **High churn triggers risk alerts:**
   - Partners see "Operator has 40% partner churn" → Increases monitoring
   - Users see "High partner turnover" → May choose different operator
   - Market pressure against Sybil rotation

**Why This Prevents The Attack:**

The prosecution's attack involves:
- Closing Sybil-1, Sybil-2, Sybil-3
- Opening Sybil-4, Sybil-5, Sybil-6
- Net: 6 partner changes over 12 weeks

**Partner churn rate:** (6 changes) / (avg 10 partners) = 60% over 12 weeks = 260% annualized

This is:
- **Extremely visible** in stability metrics
- **Automatic red flag** for honest partners
- **Reputation damage** visible to users

Honest partners force-close before attack completes. Users withdraw deposits. Attack fails.

### Summary: Sustained Coverage Withdrawal Prevention

**Four-Layer Defense:**

1. **Objective Objection Criteria**: Partners can cryptographically prove reduction is excessive
2. **Cross-Channel Aggregation**: Local decisions use global information about operator behavior
3. **Minimum Coverage Lock**: Absolute floor (150%) that cannot be breached
4. **Partner Stability Metrics**: High churn is visible and penalized by market

**Attack Requires:**

- Violating objective thresholds (blocked by layer 1)
- Evading aggregate pattern detection (blocked by layer 2)
- Reducing below 150% (blocked by layer 3)
- Rotating partners without detection (blocked by layer 4)

**With these enhancements, the attack as described cannot succeed.**

---

## Question 3: Capital Requirements Documentation

**JUDGE's Question:** Should the documentation explicitly state the relationship between security level, collateral ratio, and assumed honest partner percentage?

**Answer: YES, absolutely.**

The whitepaper's claim "to steal X, operator must lock 2X+" is oversimplified. I propose the following clarification:

### Recommended Documentation Addition

**Section: "Understanding Collateral Requirements and Security Levels"**

The protocol's security against theft depends on the relationship between:
1. **Collateral coverage ratio** (reserves + attestations) / deposits
2. **Number of channel partners** (N)
3. **Assumed honest partner majority** (H)
4. **Transitive liability depth** (how many hops)

#### Single-Channel Security (Basic Model)

For an operator with ONE channel:
- Coverage ratio 200% (100% reserves + 100% attestations)
- Theft requires collusion with the partner
- Capital at risk: 200% of deposits
- **Security claim:** Theft unprofitable for rational operator

#### Multi-Channel Security (Attestation Recycling Model)

For an operator with N channels using the same collateral attestations across channels:

**To protect against K colluding partners:**

Minimum coverage ratio = 100% + (K / (N - K)) × 100%

**Examples:**

| Total Partners (N) | Colluding Partners (K) | Required Coverage | Effective Security |
|-------------------|----------------------|------------------|-------------------|
| 3 | 1 | 200% | Theft unprofitable |
| 3 | 2 | 300% | Theft unprofitable |
| 10 | 2 | 125% | Protected against 20% collusion |
| 10 | 5 | 200% | Protected against 50% collusion |
| 10 | 9 | 1000% | Protected against 90% collusion (impractical) |

**Key Insight:** Capital efficiency (using same collateral for multiple channels) trades off against protection level.

**User Decision:** Choose operators based on threat model:
- **High security:** 300%+ coverage, 10+ partners → Protected against majority collusion
- **Balanced:** 200% coverage, 5+ partners → Protected against minority collusion
- **Cost-optimized:** 150% coverage, 3 partners → Basic protection

#### Transitive Liability (Second-Order Model)

The above model assumes colluding partners have ZERO capital at risk in their own channels. But System 10 (Transitive Liability) extends slashing to colluding partners' OTHER channels.

**Extended Calculation:**

For each colluding partner, add their at-risk capital from OTHER channels:

Total capital at risk = Direct collateral + Σ(colluding_partner[i].other_channel_reserves)

**Example:**

Operator Alice with 10 channels (100k deposits each):
- Alice's coverage: 200% (200k excess total)
- Alice colludes with Bob and Charlie

Bob operates 5 other channels with 50k at risk.
Charlie operates 3 other channels with 30k at risk.

**Capital at risk if caught:**
- Alice's excess: 200k (direct)
- Bob's other channels: 50k (transitive)
- Charlie's other channels: 30k (transitive)
- **Total: 280k**

**Theft potential:** 200k (if Bob and Charlie's channels)

**Net expected value:** (Theft - Slashing) × P(caught) = (200k - 280k) × ~100% = **-80k**

**Still unprofitable.**

#### Recommended Operator Disclosures

Every operator should disclose in machine-readable format:

```rust
pub struct OperatorSecurityParams {
    pub operator_id: PublicKey,
    pub total_deposits: u64,
    pub total_reserves: u64,
    pub total_attested_collateral: u64,
    pub coverage_ratio_bps: u16,           // basis points (20000 = 200%)
    pub channel_count: u32,
    pub partner_count: u32,
    pub collateral_partner_count: u32,

    // NEW: Security level disclosures
    pub protected_against_k_colluding: u32, // e.g., 5 of 10 partners
    pub transitive_liability_depth: u32,    // e.g., 2 hops
    pub assumed_honest_majority: u16,       // basis points (e.g., 7000 = 70%)

    // Partner quality metrics
    pub partner_avg_coverage_ratio: u16,    // avg of partners' own ratios
    pub partner_avg_tenure_blocks: u32,     // avg time as partner
    pub operator_track_record_blocks: u32,  // time operating without slashing

    pub signature: [u8; 64],
}
```

#### User Interface Guidelines

Wallets should display security levels in user-friendly terms:

**Tier 1 (Maximum Security):**
- Coverage: 300%+
- Partners: 10+
- Protected against: Majority collusion (6+ of 10)
- Transitive liability: 2+ hops
- Track record: 1+ years without incidents

**Tier 2 (High Security):**
- Coverage: 200-300%
- Partners: 5-10
- Protected against: Minority collusion (2-4 of 5-10)
- Transitive liability: 1+ hops
- Track record: 6+ months

**Tier 3 (Standard Security):**
- Coverage: 150-200%
- Partners: 3-5
- Protected against: Single partner collusion
- Transitive liability: 1 hop
- Track record: 3+ months

**Tier 4 (Basic Security):**
- Coverage: 100-150%
- Partners: 1-3
- Protected against: None (trust-based)
- New operator or recovery situation

**User choice:**
- "I want maximum security, I'll pay higher fees" → Tier 1 operator
- "I want good security at reasonable cost" → Tier 2-3 operator
- "I trust this operator, I want low fees" → Tier 4 operator

### Math Should Be Transparent, Not Hidden

**Current whitepaper issue:** Claims "2X capital locks" without explaining it assumes honest majority.

**Better approach:**

1. **State the security model explicitly:**
   - "Protection level depends on coverage ratio, partner count, and collusion threshold"

2. **Provide the formula:**
   - `Required_Coverage = 100% + (K/(N-K)) × 100%`

3. **Give concrete examples:**
   - Show calculations for different N, K combinations

4. **Let users choose:**
   - Display operator's actual parameters
   - Let market determine acceptable risk/cost tradeoffs

5. **Don't oversimplify:**
   - "Economically irrational for rational actors given these assumptions"
   - NOT "cryptographically impossible"
   - NOT "guaranteed secure"

### Proposed Whitepaper Addition

I recommend adding a new section (Section 12):

**"Security Model and Capital Requirements"**

This section would include:
- The formulas above
- Example calculations
- Operator disclosure requirements
- User decision framework
- Explicit statement of assumptions:
  - At least (N-K) partners are honest
  - At least one partner online during recovery
  - Cryptographic primitives secure (SHA256, ECDSA)
  - Transitive liability propagates through partner graph
  - Market disciplines operators with poor track records

**This makes security properties transparent rather than implied.**

---

## Summary: Addressing the Sustained Attacks

### Ledger Equivocation (Attack 2)

**Status:** Protocol ALREADY prevents this through commitment transaction co-signing + UpdateReserves validation.

**Enhancement:** Add explicit partner co-signatures on updates OR Merkle checkpoint attestations for cryptographic clarity.

**Attack viability:** Does not work against current protocol. Enhancement makes protection more obvious.

### Sustained Coverage Withdrawal (Attack 5)

**Status:** Current specification is insufficient against coordinated gradual withdrawal.

**Enhancements Required:**

1. Objective objection criteria (coverage floors, rate limits, pattern detection)
2. Cross-channel reduction aggregation (global view for local decisions)
3. Minimum coverage lock (absolute 150% floor)
4. Partner stability metrics (churn detection and reputation impact)

**Attack viability:** With these four enhancements, attack cannot succeed.

### Capital Requirements Clarification (Question 3)

**Status:** Current documentation oversimplifies.

**Required Change:** Add explicit formulas, examples, and operator disclosure requirements showing relationship between coverage ratio, partner count, and security level.

**User Impact:** Enables informed choice based on transparent metrics.

---

## Conclusion

The protocol's core mechanisms are sound. The two sustained attacks reveal:

1. **Documentation gaps** (ledger equivocation prevention exists but wasn't clear)
2. **Enforcement gaps** (sustained coverage withdrawal needs stronger detection)

Both are addressable without fundamental protocol redesign:

- **Equivocation:** Already prevented; enhancement makes it explicit
- **Withdrawal:** Add objective criteria + aggregate monitoring + minimum locks + stability metrics

**The protocol, with these clarifications and enhancements, provides the claimed security properties.**

Users who need custodial Lightning get:
- Cryptographic accountability (unforgeable evidence)
- Economic deterrence (theft unprofitable given honest majority)
- Continuous transparency (visible metrics)
- Automatic recovery (funds never permanently lost)

**This remains a dramatic improvement over naive custodial Lightning, and with the proposed enhancements, addresses the sustained vulnerabilities.**
