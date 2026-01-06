# PROSECUTION Opening Statement

## Fatal Flaws in the Bitcoin Deposits Protocol

The Bitcoin Deposits Protocol is an intricate construction attempting to create trust-minimized custody through cryptographic accountability and economic incentives. However, beneath its sophisticated mechanisms lie fundamental vulnerabilities that cannot be patched away. These attacks exploit the protocol's core assumptions, creating conditions where rational adversaries can profit from theft despite the collateral requirements.

---

## Attack 1: The Timelock Exploitation Attack

**Severity: FATAL FLAW**

### The Mechanism

The protocol relies on time-degrading voting thresholds (System 8) to ensure recovery decisions are "always reached." However, this creates a predictable attack window that sophisticated adversaries can exploit.

The threshold degradation schedule:
- Days 1-3: Majority (5-of-9)
- Days 3-7: 40% (4-of-9)
- Days 7-14: 30% (3-of-9)
- Days 14-21: 20% (2-of-9)
- Day 21+: Any single voter (1-of-9)

### The Attack

An operator with 9 collateral partners can:

1. **Control 2 partners directly** through Sybil identities or explicit collusion
2. **Delay 3 additional partners** using network attacks (DoS, partition attacks, targeted delays)
3. **Wait for Day 14-21** when threshold drops to 20% (2-of-9)
4. **Cast 2 "conforming" votes** using controlled partners
5. **Claim funds back** despite being non-conforming

**Why it works:**
- Only 4 honest voters can reach the network during the attack window (2 are controlled, 3 are DoS'd)
- Honest voters need 3-of-9 until day 14, but only 4 are reachable
- By day 14-21, threshold drops to 2-of-9
- Attacker's controlled voters reach threshold first, voting "conforming"
- Protocol awards funds back to the dishonest operator

**Capital calculation:**
```
Steal: 100k sats from deposits
Control: 2 of 9 partners (22% of collateral graph)
Attack cost: DoS infrastructure for 14 days = negligible
Profit: 100k - (slashing from 4 honest voters) = 100k - 44k = 56k profit
```

**Why this is fatal:**
The protocol's own liveness guarantee (degrading thresholds) becomes the attack vector. You cannot simultaneously guarantee "a decision is always reached" AND "the decision will be correct" when attackers can influence which voters participate.

---

## Attack 2: The Ledger Equivocation Attack

**Severity: FATAL FLAW**

### The Mechanism

The protocol maintains bidirectional ledgers with asymmetric validation (System 1):
- **Operator copy**: Strict validation, rejects out-of-order updates
- **Partner copy**: Relaxed validation, buffers out-of-order updates

This asymmetry, combined with hash chain commitment, creates an equivocation opportunity.

### The Attack

1. **Operator maintains two parallel ledger chains** for the same channel:
   - **Chain A (shown to partner)**: Legitimate operations, all deposits properly credited
   - **Chain B (signed but hidden)**: Missing credits, operator pockets payments

2. **Partner sees only Chain A** and updates their copy, believing all is well

3. **During force-close, operator commits Chain B hash on-chain** in the reserves output

4. **Partner's validation fails** because they have Chain A, which doesn't match the on-chain hash

5. **Operator claims**: "Partner's ledger is corrupted, my on-chain hash proves the true state"

**Why current defenses fail:**

The protocol requires partners to validate that ledger updates form a proper hash chain. But the partner only sees the updates sent to them. They cannot know whether OTHER valid chains exist with different operations but valid signatures.

The `SignedAuditUpdate` broadcast only proves "this update exists and is properly signed" - it doesn't prove "this is the ONLY valid update at this sequence number."

**The cryptographic problem:**

Hash chains prevent modification of history, but they don't prevent parallel histories. An operator can sign multiple valid sequence_number=100 updates with different operations, each forming a valid chain from genesis. Partners cannot detect this until force-close when only ONE hash hits the chain.

**Why this is fatal:**

The entire protocol security depends on "signed ledger updates prove every operation." But if operators can maintain parallel valid ledgers and selectively commit one during force-close, the cryptographic accountability breaks. Voters cannot determine which chain is "correct" because both have valid signatures and hash linkage.

---

## Attack 3: The Attestation Recycling Attack

**Severity: HIGH (Economic incentive failure)

### The Mechanism

System 2's multichannel collateral allows the same excess collateral to back multiple ledgers for capital efficiency. The whitepaper states: "Charlie's 30k excess backs BOTH Alice→Bob AND Alice→David."

But this creates a reuse limit that attackers can exploit through strategic topology.

### The Attack Setup

Alice operates 10 channels with 100k deposits each (1M total):
```
Alice→Bob:     deposits=100k, reserves=120k (20k excess)
Alice→Charlie: deposits=100k, reserves=120k (20k excess)
Alice→Dave:    deposits=100k, reserves=120k (20k excess)
Alice→Eve:     deposits=100k, reserves=120k (20k excess)
Alice→Frank:   deposits=100k, reserves=120k (20k excess)
Alice→Grace:   deposits=100k, reserves=120k (20k excess)
Alice→Heidi:   deposits=100k, reserves=120k (20k excess)
Alice→Ivan:    deposits=100k, reserves=120k (20k excess)
Alice→Judy:    deposits=100k, reserves=120k (20k excess)
Alice→Mike:    deposits=100k, reserves=120k (20k excess)

Total excess: 200k sats
```

**The protocol assumes**: This 200k excess provides 100% collateral coverage across all ledgers.

**The reality**: If Alice colludes with 6 partners simultaneously:

1. **Steal from 6 channels** at once: 600k sats
2. **Remaining honest partners**: 4 (Bob, Charlie, Dave, Eve)
3. **Available slashing**: 4 × 20k = 80k sats
4. **Net profit**: 600k stolen - 80k slashed = **520k profit**

**Why this breaks the economic model:**

The whitepaper claims: "To steal X, operator must have locked 2X+ capital. Getting caught means losing all 2X+."

But with attestation recycling, to steal 600k, Alice only needs 200k at risk. She violates the 2X rule by coordinating multiple simultaneous thefts that exhaust the reusable collateral.

**The mathematical flaw:**

Let:
- N = number of channels
- D = deposits per channel
- E = excess per channel

Total excess = N × E
Total potential theft = N × D

For security, we need: N × E ≥ N × D, so E ≥ D

But if k channels collude:
- Theft = k × D
- Slashing = (N - k) × E

For attack to fail: (N - k) × E ≥ k × D

Rearranging: E ≥ (k/(N-k)) × D

When k = N-1 (all but one partner colludes):
E ≥ (N-1)/1 × D = (N-1) × D

**This means**: To prevent coordinated theft of (N-1) channels, each channel needs excess equal to (N-1) times its deposits. For a 10-channel operator, that's 900% collateral per channel, not 100%.

The protocol's capital efficiency assumption is incompatible with its security claims against coordinated multi-channel collusion.

**Severity assessment**: HIGH not FATAL because it requires coordinating 60% of partners. But it proves the 2X capital requirement is misleading.

---

## Attack 4: The Recovery Entropy Manipulation Attack

**Severity: MEDIUM (Realistic griefing attack)

### The Mechanism

System 9's sequential random selection uses on-chain entropy: `SHA256(block[N+6].hash || ledger_id)` to select which partner can claim in each round.

But miners can influence block hashes through strategic transaction ordering and nonce selection. While full preimage attacks on SHA256 are infeasible, miners can grind partial collisions to influence selection probabilities.

### The Attack

A large mining pool operating Deposits can:

1. **Force-close a channel** they want to claim at block N
2. **At block N+6**, grind nonces to influence the block hash
3. **Target selection** toward a partner identity they control
4. **Claim funds** in Round 1 with increased probability

**Economics of the attack:**

Grinding cost vs. value:
- Expected value of random selection: claim worth C with probability 1/V (V=voter count)
- Mining a block with favorable hash increases probability to 1/V'
- If mining costs M per block and claim value C > M × V, profitable

**Example numbers:**
- Voter set size: 10 partners
- Claim value: 1 BTC (100M sats)
- Normal probability: 10% chance
- Grinding 100 blocks: increases to ~20% chance
- Grinding cost: 100 × (fee revenue + opportunity cost) ≈ 0.1 BTC
- Expected value: 0.20 × 100M = 20M sats
- Cost: 0.1 × 100M = 10M sats
- **Profit: 10M sats**

**Why this matters:**

The protocol assumes entropy is unbiasable. But large mining pools (which may also run Lightning infrastructure) can profitably manipulate recovery selection. This doesn't break the protocol but undermines the fairness of recovery and creates centralization pressure toward miner-operated nodes.

**Severity assessment**: MEDIUM because it requires mining power and only increases probability, doesn't guarantee success. But it's realistic for large pools and creates bad incentives.

---

## Attack 5: The Sustained Coverage Withdrawal Attack

**Severity: HIGH (Defeats System 11's core protection)

### The Mechanism

System 11 (Sustained Coverage) prevents "pre-attack capital optimization" by requiring operators to broadcast `ReduceNotice` with a 2016-block (~2-week) delay before collateral reductions take effect.

But the protocol allows reductions for legitimate reasons including "ChannelClosure" and "PartnerRequest." An operator can exploit these exception paths.

### The Attack

Operator coordinates with Sybil partners to create a withdrawal schedule:

**Week 1-4: Sybil rotation**
1. Alice opens channels with Sybil partners (Sybil-1, Sybil-2, Sybil-3)
2. Sybils provide attestations, building apparent collateral
3. Alice broadcasts health showing strong coverage
4. Users see high collateral ratio, deposit funds

**Week 5-8: Coordinated "partner requests"**
1. Sybil-1 sends `PartnerRequest` reduction notice
2. 2016 blocks later, Sybil-1 closes channel
3. Immediately, Sybil-4 opens new channel with minimal attestation
4. Net collateral drops but gradual enough to avoid "Degrading" status

**Week 9-12: Channel closure cascade**
1. Repeat Sybil rotation for Sybil-2, Sybil-3
2. Each closure is "justified" by partner request
3. Remaining honest partners see notices but each individual reduction appears reasonable
4. Aggregate trend shows slow decline over 12 weeks

**Week 13: Execute theft**
1. Collateral now minimal (90% reduction over 3 months)
2. Force-close channels, steal deposits
3. Slashable collateral = nearly zero
4. Users lose funds

**Why defenses fail:**

The protocol states partners can "object" to reductions, but objections require proving the reduction is unjustified. When the reduction reason is "PartnerRequest" and the partner confirms the request, what grounds exist to object?

Health broadcasts show declining coverage, but over 12 weeks the slope might be -0.3%/day, below the -0.5%/day "Degrading" threshold mentioned in the spec.

Partners face a coordination problem: each individual reduction looks fine, but the aggregate pattern is malicious. Without global coordination, local validators cannot distinguish slow legitimate wind-down from strategic withdrawal.

**Why this is HIGH severity:**

It defeats System 11's core purpose (preventing pre-attack optimization) using legitimate protocol mechanisms (partner-requested reductions, gradual withdrawal). The 2-week delay is ineffective when withdrawals are staged over months.

---

## Conclusion: Systemic Vulnerabilities

These attacks share a common pattern: they exploit the protocol's complexity and coordination requirements to create exploitable gaps between theoretical guarantees and practical security.

### Summary Table

| Attack | Severity | Root Cause | Can It Be Patched? |
|--------|----------|------------|-------------------|
| Timelock Exploitation | FATAL | Degrading thresholds + network attacks | No - fundamental tradeoff |
| Ledger Equivocation | FATAL | Hash chains don't prevent parallel histories | No - cryptographic limitation |
| Attestation Recycling | HIGH | Capital efficiency vs security math | Possibly - require higher ratios |
| Recovery Entropy | MEDIUM | Miner influence on block hashes | Possibly - use VRFs or longer commitment |
| Sustained Coverage Withdrawal | HIGH | Legitimate withdrawal paths exploitable | Difficult - needs global coordination |

### The Core Problem

The protocol attempts to provide cryptographic certainty in a system that fundamentally requires social consensus (recovery voting), economic rationality (collusion unprofitability), and coordination (multi-party actions).

The attacks above show that:
1. **Liveness vs. safety tradeoffs are exploitable** (Timelock)
2. **Cryptographic commitments don't prevent equivocation** (Ledger)
3. **Capital efficiency math doesn't hold under coordination** (Attestation)
4. **Economic incentives can be gamed by sophisticated actors** (Entropy, Withdrawal)

These aren't bugs to be fixed - they're inherent tensions in a trust-minimized system that relies on:
- Honest majority voting
- Timely coordination
- Perfect information propagation
- Rational economic behavior
- Cryptographic accountability

**The verdict**: While the protocol dramatically improves upon naive custodial Lightning, it cannot achieve its stated goal of making operator theft "economically impossible." Sophisticated, well-capitalized adversaries can profit from coordinated attacks that exploit the protocol's complexity.

Bitcoin users deserve better than systems where security depends on "probably too complex to attack profitably." The protocol should either:
1. Simplify dramatically to reduce attack surface
2. Add formal verification to prove security properties
3. Acknowledge it's trust-reduced, not trust-minimized
4. Admit that at scale, custody requires trust

I argue the protocol, as specified, **cannot work** at the scale and adversarial intensity required for "the next billion Bitcoiners."
