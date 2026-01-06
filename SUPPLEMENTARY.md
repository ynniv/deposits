# Bitcoin Deposits Protocol - Supplementary Mechanisms

This document describes **optional coordination mechanisms** that can strengthen the core protocol. These are not required for basic security—the core protocol (WHITEPAPER.md) provides cryptographic accountability and transparency. These mechanisms add enforcement layers for operators and partners who want additional protection.

**Philosophy:** The core protocol trusts markets to enforce behavior based on transparent information. These supplementary mechanisms add protocol-level enforcement for specific attack vectors. The tradeoff is complexity for additional assurance.

---

## S1. Circular Attestation Enforcement

The core protocol displays circular attestation percentages. This supplement adds enforcement limits.

### The Problem

Circular attestation—where operators attest to each other—creates liability escape: Bob won't force-close with Alice if doing so would lose Alice's attestation and cascade failure across his channels.

### Enforcement Limits

```rust
const MAX_CIRCULAR_ATTESTATION_BPS: u16 = 4000;  // 40% max from circular
const MIN_EXTERNAL_ATTESTATION_BPS: u16 = 6000;  // 60% must be external

pub struct AttestationSource {
    pub partner: PublicKey,
    pub amount: u64,
    pub is_reciprocal: bool,  // Partner also operates ledgers we attest
}

impl VoterSet {
    pub fn circular_attestation_ratio(&self) -> u16 {
        let total: u64 = self.voters.iter().map(|v| v.attested_amount).sum();
        let circular: u64 = self.voters.iter()
            .filter(|v| self.is_reciprocal_relationship(&v.pubkey))
            .map(|v| v.attested_amount)
            .sum();

        if total == 0 { return 0; }
        ((circular * 10000) / total) as u16
    }

    pub fn validate_circular_limits(&self) -> Result<(), Error> {
        let circular_bps = self.circular_attestation_ratio();
        if circular_bps > MAX_CIRCULAR_ATTESTATION_BPS {
            return Err(Error::ExcessiveCircularAttestation {
                actual_bps: circular_bps,
                max_bps: MAX_CIRCULAR_ATTESTATION_BPS,
            });
        }
        Ok(())
    }
}
```

### Why 40%/60%?

At 40% circular, security degrades from 2x capital at risk to 1.6x:
- 60% of attestations come from partners with no conflicting incentive (external)
- 40% come from partners who might hesitate to slash (circular)
- Net slashing power: 60% + (40% × some_fraction) ≈ 80% of full strength

This is a reduction, not elimination. The attack becomes unprofitable, not impossible.

### Limitations

Hidden circular patterns cannot be fully prevented without identity verification:
- Alice → Sybil-1 → Charlie → Sybil-2 → Bob looks acyclic but is actually controlled

The enforcement provides defense against naive circular patterns. Sophisticated attackers can evade it. The core protocol's market-based approach (display percentage, let partners decide) may be equally effective with less complexity.

---

## S2. Grace Period Pre-Commitment

The core protocol has a grace period for partners to evaluate collusion evidence. This supplement adds commit-reveal to prevent "wait and see" behavior.

### The Problem

Without pre-commitment:
- Partners wait for voting certainty before acting
- Waiting too long makes them "Complicit"
- Malicious operators can drain capital via Lightning routing during the grace period

### Commit-Reveal Mechanism

**Phase 1 - Evidence Publication (Block N):**

```rust
pub struct CollusionEvidencePublication {
    pub operator: PublicKey,
    pub partner: PublicKey,
    pub channel_id: ChannelId,
    pub force_close_tx: Transaction,
    pub on_chain_ledger_hash: [u8; 32],
    pub force_close_block: u32,
    pub publisher: PublicKey,
    pub signature: [u8; 64],
}
```

**Phase 2 - Commitment (Blocks N+1 to N+144):**

Partners commit to their evaluation WITHOUT revealing it:

```rust
pub struct EvidenceCommitment {
    pub evaluator: PublicKey,
    pub evidence_hash: [u8; 32],      // Hash of evidence publication
    pub commitment_hash: [u8; 32],    // SHA256(evaluation || nonce)
    pub bond_amount: u64,             // Capital bonded
    pub bond_expiry: u32,             // Block height
    pub signature: [u8; 64],
}

pub struct SealedEvaluation {
    pub is_dishonest: bool,
    pub will_force_close: bool,
    pub nonce: [u8; 32],
}
```

**Phase 3 - Reveal and Execute (Block N+144 to N+150):**

```rust
pub struct EvidenceReveal {
    pub evaluator: PublicKey,
    pub commitment_hash: [u8; 32],
    pub evaluation: SealedEvaluation,
    pub signature: [u8; 64],
}

// Verification
assert!(SHA256(evaluation || nonce) == commitment_hash);
```

**Enforcement:**
- Partners who committed `will_force_close = true` MUST execute within 6 blocks or lose bond
- Partners who committed `is_dishonest = false` when evidence clearly shows fraud lose bond

**Phase 4 - Complicit Status Determination:**

```rust
pub fn determine_complicit_status(
    partner: PublicKey,
    evidence: &CollusionEvidencePublication,
    commitment: Option<&EvidenceCommitment>,
    reveal: Option<&EvidenceReveal>,
    force_close_executed: bool,
) -> NodeStatus {
    if commitment.is_none() {
        return NodeStatus::Complicit {
            reason: "Failed to commit evaluation within window"
        };
    }

    if !verify_reveal(commitment, reveal) {
        return NodeStatus::Complicit {
            reason: "Reveal inconsistent with commitment"
        };
    }

    let evidence_valid = validate_evidence(evidence);
    if evidence_valid && !reveal.evaluation.is_dishonest {
        return NodeStatus::Complicit {
            reason: "Evaluated provable fraud as honest"
        };
    }

    if reveal.evaluation.will_force_close && !force_close_executed {
        return NodeStatus::Complicit {
            reason: "Failed to execute committed force-close"
        };
    }

    NodeStatus::Honest
}
```

### Economic Analysis

```
Without pre-commitment:
- Partners wait for voting certainty → exceed grace period → become Complicit
- Operator drains 1M+ during grace period via Lightning routing
- Slashing recovers only remaining reserves (~100k)
- Net profit: ~900k

With pre-commitment:
- Partners independently evaluate and commit by Block N+144
- Partners reveal and execute force-closes
- Operator's excess (180k) is slashed before drainage is safe
- Net: 100k stolen - 180k slashed = -80k LOSS
```

### Tradeoff

Pre-commitment adds significant complexity (a full sub-protocol) to solve the coordination problem. The core protocol relies on partners acting in their own interest without coordination guarantees. Pre-commitment provides stronger guarantees at the cost of implementation complexity.

---

## S3. Four-Layer Sustained Coverage Defense

The core protocol makes collateral reductions visible via time-locks and broadcasts. This supplement adds four enforcement layers.

### The Attack

Without sustained coverage requirements, an operator could gradually withdraw excess collateral over weeks, then execute theft with minimal slashable capital remaining.

### Layer 1: Objective Objection Criteria

Partners can automatically object to reductions that violate thresholds:

```rust
pub enum ObjectionReason {
    CoverageWouldDropBelowMinimum {
        current_coverage_bps: u16,
        post_reduction_bps: u16,
        minimum_allowed_bps: u16,  // ABSOLUTE_MINIMUM_COVERAGE_BPS = 15000
    },

    ExcessiveReductionRate {
        reduction_amount: u64,
        window_days: u32,
        actual_bps_per_day: u16,
        max_allowed_bps_per_day: u16,  // 30 = 0.3%/day
    },

    FrequentReductionPattern {
        notices_in_window: u32,
        window_blocks: u32,
        max_allowed: u32,  // 2 per 2016 blocks
    },

    PartnerChurnDetected {
        new_partners_added: u32,
        partners_removed: u32,
        churn_rate_bps: u16,
        max_allowed_churn_bps: u16,  // 3000 = 30% per 90 days
    },
}
```

### Layer 2: Cross-Channel Aggregation

Partners share their views to enable global pattern detection:

```rust
pub struct AggregateReductionView {
    pub operator: PublicKey,
    pub reporting_partner: PublicKey,
    pub observation_window: (u32, u32),
    pub total_reduction_notices: u32,
    pub total_reduction_amount: u64,
    pub affected_channels: Vec<ChannelId>,
    pub partner_churn_events: Vec<PartnerChurnEvent>,
    pub signature: [u8; 64],
}
```

**Aggregate Sharing Protocol:**
1. Each partner maintains a view of operator's global state
2. Weekly broadcasts (every 1008 blocks): Partners share `AggregateReductionView`
3. Cross-verification: Discrepancies = evidence of selective disclosure
4. Aggregate triggers:
   - 5+ reduction notices across all channels in 30 days → Alert
   - 30%+ partner churn in 90 days → Alert
   - Total reduction exceeds 20% of coverage in 60 days → Alert

### Layer 3: Minimum Coverage Lock

Coverage can never drop below 150%:

```rust
pub struct CoverageLock {
    pub locked_until: u32,
    pub locked_amount: u64,
    pub lock_reason: LockReason,
}

pub enum LockReason {
    InitialDeposit,    // Locked for min 12 weeks
    ReputationBuild,   // New operators locked longer
    SlashingReserve,   // Always maintain minimum buffer
}
```

**Coverage Lock Policy:**

| Coverage Threshold | Lock Period |
|-------------------|-------------|
| Any → below 150% | **Impossible** (must close channel) |
| 200% → 150-175% | 4032 blocks (~4 weeks) |
| Above 200% → 175-200% | 2016 blocks (~2 weeks) |

**New Operator Restrictions:**

| Operator Age | Minimum Coverage |
|--------------|------------------|
| First 12 weeks | Initial ratio (locked) |
| Weeks 12-24 | Can reduce to 200% |
| Week 24+ | Can reduce to 150% |

### Layer 4: Partner Stability Metrics

```rust
pub struct PartnerStabilityScore {
    pub operator: PublicKey,
    pub measurement_window_blocks: u32,
    pub partners_at_start: Vec<PublicKey>,
    pub partners_at_end: Vec<PublicKey>,
    pub partners_added: Vec<(PublicKey, u32)>,
    pub partners_removed: Vec<(PublicKey, u32)>,
    pub churn_rate_bps: u16,
    pub average_partner_tenure_blocks: u32,
    pub signature: [u8; 64],
}
```

**Stability Display:**

| Stability Level | Churn Rate | Avg Tenure | Indicator |
|-----------------|-----------|------------|-----------|
| Excellent | <10% / 6mo | >6 months | Green |
| Acceptable | 10-30% / 6mo | 3-6 months | Yellow |
| Concerning | >30% / 6mo | <3 months | Red |

### Summary

| Layer | Attack Vector | Defense |
|-------|---------------|---------|
| 1. Objective Objection | Gradual reductions | Cryptographically verifiable thresholds |
| 2. Cross-Channel Aggregation | Pattern hiding | Global view from local observations |
| 3. Minimum Coverage Lock | Capital optimization | 150% absolute floor |
| 4. Partner Stability | Sybil rotation | Churn visibility |

### Tradeoff

Four layers provide comprehensive defense but add substantial complexity:
- Objective criteria require consensus on thresholds
- Aggregation requires coordination protocol between partners
- Coverage locks reduce operator flexibility
- Stability scoring requires tracking and broadcasting metrics

The core protocol achieves similar outcomes through transparency + market pressure, with less implementation burden. These layers are for high-security deployments that want protocol-level enforcement.

---

## When to Use These Supplements

| Supplement | Use When |
|------------|----------|
| S1. Circular Enforcement | Partners want hard limits, not just visibility |
| S2. Pre-Commitment | Grace period drainage is a significant concern |
| S3. Four-Layer Defense | Operating high-value channels with sophisticated adversaries |

**Default recommendation:** Start with the core protocol. Add supplements if specific attack vectors become problems in practice. Complexity has costs; add it only when benefits are proven.
