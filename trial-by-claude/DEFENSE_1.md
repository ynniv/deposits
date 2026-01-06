# DEFENSE Opening Statement - Bitcoin Deposits Protocol

## The Security Model: Economic Constraints, Not Cryptographic Magic

The Bitcoin Deposits Protocol achieves trust-minimization through **economic deterrence**, **cryptographic accountability**, and **continuous transparency**. It does not eliminate trust entirely—no multi-party system can—but it fundamentally transforms the trust model from "hope the custodian is honest" to "multiple parties with capital at risk would need to coordinate against their own economic interests."

### Core Security Pillars

**1. Economic Constraints (200% Collateral)**

Operators must maintain:
- 100% reserves: Direct backing of deposits in each channel
- 100% attestations: Slashable collateral from OTHER channels

To steal X sats, an operator must have locked 2X+ capital. If caught (near-certain due to cryptographic evidence), they lose the full 2X+. The expected value of theft is strictly negative for any rational actor.

**2. Cryptographic Accountability (Hash-Chained Ledger)**

Every operation is:
- Signed by the operator (undeniable authorship)
- Hash-chained to previous state (tamper-evident)
- Broadcast to multiple auditors (observable)
- Validated against protocol rules (deterministically verifiable)

Fraud creates unforgeable evidence: ledger/state mismatches, uncredited preimages, dishonest votes. There is no plausible deniability—either the signatures match and the rules were followed, or they don't.

**3. Multi-Party Validation**

Each channel has:
- A partner who cosigns invoices and validates reserves
- Collateral partners who attest to slashable funds
- All partners receiving every ledger update in real-time

No single party controls the narrative. Evidence is distributed across independent observers who can independently verify compliance.

**4. Automatic Recovery**

If an operator is dishonest, funds don't vanish—they transfer through tiered voting:
- Ledger (record of who owns what) is portable
- Reserves transfer to a new operator
- Deposits continue to be honored
- Service continuity, not just fund recovery

## Why The Key Assumptions Are Reasonable

### Assumption 1: Partners Are Functioning Lightning Operations

**What this means:** Partners are online, monitoring channels, processing updates.

**Why it's reasonable:** Anyone running a Lightning node for routing already has:
- Always-on infrastructure with monitoring
- Channel management expertise
- Economic incentive to stay online (routing revenue)
- Watchtower capabilities

Deposits doesn't add new liveness requirements—it leverages requirements that existing Lightning operations already meet. A partner who is "offline for 2 weeks" has already failed as a Lightning business regardless of Deposits.

### Assumption 2: Collusion Requires Coordination

**What this means:** To profit from theft, multiple parties must coordinate while putting more capital at risk than they can steal.

**Why it's reasonable:**

*Single-partner collusion:* If Operator + Partner A steal from Channel A, Partners B and C slash collateral from their channels. Net: Operator loses capital.

*Multi-partner collusion:* If Operator + Partners A + B collude, each conspirator has OTHER partners who can observe the collusion evidence and slash THEIR funds through transitive liability. Capital at risk scales faster than potential theft.

*Sybil resistance:* Creating fake partners requires real capital (can't fake channel balances). To create a closed collusion ring with N partners requires 2N × deposit_amount in capital. For N ≥ 2, the attacker locks more than they can steal.

The game theory is simple: theft requires more capital than honest operation, with higher risk and lower returns.

### Assumption 3: Evidence Is Cryptographically Deterministic

**What this means:** Disputes are resolved by checking signatures and hashes, not subjective interpretation.

**Why it's reasonable:**

Evidence types are binary:
- Ledger/state mismatch: On-chain hash ≠ signed update chain hash? Yes/No
- Uncredited payment: SHA256(preimage) = payment_hash AND no credit in ledger? Yes/No
- Dishonest vote: Voter claims conforming but their own ledger copy shows violations? Yes/No
- Continued operations: New ledger updates after collusion evidence published? Yes/No

There's no "operator claims they were censored" defense—either they signed the updates or they didn't. Either the hashes match or they don't. This is what "cryptographic accountability" means.

### Assumption 4: The Market Determines Acceptable Risk

**What this means:** The protocol doesn't mandate specific collateral ratios—it makes all parameters transparent and lets users choose.

**Why it's reasonable:**

Different users have different risk tolerances:
- Maximum security: Choose operators with 300% collateral, 10+ partners, known identity
- Balanced: Choose 200% collateral, 3+ partners, track record
- Cost-optimized: Choose lower collateral, accept more risk for lower fees

This is how markets work. The protocol provides transparency (coverage ratios, partner counts, health trends) and enforces the rules (slashing when violations are proven), but doesn't mandate one-size-fits-all parameters.

New operators can enter by offering higher collateral or partnering with established operators. Users vote with deposits. Bad operators lose funds to slashing and market discipline.

## Pre-Emptive Defense Against Likely Attacks

### Attack 1: "Gradual Capital Withdrawal Before Coordinated Theft"

**The Attack:** Operator slowly reduces excess collateral over weeks, then executes theft when slashable capital is minimal.

**Why It Fails:**

System 11 (Sustained Coverage) prevents this through:

1. **Time-locked excess:** Reductions require `ReduceNotice` broadcast 2 weeks before taking effect
2. **Cross-channel visibility:** All partners see reduction notices, not just affected channel
3. **Health broadcasting:** Regular signed metrics expose coverage trends
4. **Partner objections:** Partners can object to suspicious patterns or force-close

Fast withdrawal is visible (multiple reduction notices = obvious preparation). Slow withdrawal is detectable (health trend analysis shows degradation). Partners exit before coverage reaches critical levels.

**The attacker's dilemma:** Withdraw quickly → partners see notices and exit. Withdraw slowly → trend analysis detects degradation, partners exit. Attack with full collateral → transitive liability makes theft unprofitable.

### Attack 2: "Collusion Ring with Sybil Partners"

**The Attack:** Operator creates multiple Sybil identities as "partners" to provide fake attestations and collude without real consequences.

**Why It Fails:**

Each Sybil requires REAL capital:
- Can't fake channel balances (on-chain verification)
- Can't fake attestations (cryptographically signed by channel partners)
- Each Sybil needs channels with OTHER partners (who aren't Sybils)

To create a closed ring of N Sybil partners with D deposits:
- Capital required: 2D × N (each Sybil needs 200% collateral)
- Capital at risk: All of it (transitive liability)
- Profit potential: D (the deposits)

For N ≥ 2, the attacker must lock MORE capital than they can steal. It's capital-inefficient. Honest operation earns better returns on the same capital.

### Attack 3: "Manufacture False Collusion Evidence to Trigger Contagion"

**The Attack:** Adversary creates fake evidence that Operator is dishonest, triggering force-closes across the network and causing systemic disruption.

**Why It Fails:**

Evidence is cryptographically verifiable:
- Can't forge operator signatures (requires private key)
- Can't fake hash chain (would need to break SHA256)
- Can't fake on-chain commitment transactions (blockchain consensus)
- Can't fake preimages (would need to break SHA256 again)

If someone broadcasts "evidence" that doesn't verify cryptographically, it's ignored and the broadcaster may be penalized for spam.

If genuine evidence exists (operator actually violated rules), then partners SHOULD force-close—this is the protocol working correctly, not a vulnerability.

The concern about "cascading failures" confuses designed behavior (honest partners exiting from dishonest operators) with systemic contagion.

### Attack 4: "Recovery Coordination Failure Locks Funds"

**The Attack:** Partners can't coordinate during recovery, voting deadlocks, funds remain locked indefinitely.

**Why It's Mitigated:**

1. **Degrading thresholds:** Voting threshold decreases over time (majority → 40% → 30% → 20% → any single voter). Even if most voters are offline, one honest voter eventually suffices.

2. **Common entropy:** Recovery tiers use on-chain data (force-close txid, block hashes) as coordination substrate. No off-chain coordination needed—everyone computes the same result from blockchain data.

3. **Economic incentive:** The selected recovery operator inherits deposit revenue. They have strong incentive to complete the recovery process.

4. **Multiple tiers:** If Tier 0 (random partner) doesn't claim, Tier 1 activates. If that fails, Tier 2. Eventually Tier 4 allows any willing operator with majority partner attestation.

**Worst case:** Recovery is slow (weeks instead of days). But funds are NEVER burned—the protocol ensures someone eventually has standing to claim.

### Attack 5: "Capital Efficiency Creates Oligopoly"

**The Attack:** Established operators leverage network effects and economies of scale to dominate, preventing new operators from competing.

**Why It's Acceptable:**

First, this isn't a protocol failure—it's market dynamics. If users prefer established operators because they offer better security (more partners, longer track record), that's rational user choice, not a bug.

Second, new operators have multiple paths to entry:
- Offer higher collateral ratios (compete on security)
- Partner with established operators (leverage their reputation)
- Target underserved niches (users who prioritize low fees over reputation)
- Build track record gradually (start small, prove reliability)

Third, concentration isn't permanent:
- Established operators who become complacent lose deposits to competitors
- Protocol transparency makes switching operators frictionless
- Users can withdraw to different operators if service degrades

The protocol provides tools (transparency, verifiability, portability) but doesn't guarantee equal outcomes. That's a feature, not a bug—market competition drives operators to compete on security, uptime, fees, and service quality.

## What The Protocol Explicitly Does NOT Guarantee

### 1. Service Quality After Recovery

**Guarantee:** Funds remain available and balances are honored.

**NOT Guaranteed:** The new operator has identical routing capabilities, channel topology, or service quality as the original.

**Why that's okay:** Users chose the original operator for many reasons (routing, geography, uptime). Recovery can't preserve all of those. But it CAN preserve the most critical guarantee: fund availability.

Users who care about specific routing properties can withdraw to a different operator. The protocol ensures they CAN withdraw, not that they won't WANT to.

### 2. Instant Recovery

**Guarantee:** Funds will eventually be recovered through tiered voting.

**NOT Guaranteed:** Recovery completes within a specific timeframe.

**Why that's okay:** Recovery involves coordination among multiple parties. If all voters are online and agree, recovery can be fast (days). If voters are offline or disagree, degrading thresholds ensure recovery eventually succeeds (weeks).

This is slower than "click a button and unilaterally exit to on-chain," but it's infinitely faster than current custodial systems where theft means permanent loss.

### 3. Zero Operator Centralization

**Guarantee:** Protocol makes concentration visible and users can switch operators.

**NOT Guaranteed:** Network effects won't cause some operators to be more popular.

**Why that's okay:** Some concentration may emerge from legitimate advantages (better uptime, more partners, established reputation). The protocol makes this transparent so users can make informed choices.

If concentration becomes problematic, users can coordinate to support new operators. The protocol doesn't prevent concentration—it makes the parameters visible and switching frictionless.

### 4. Protection Against All Coordination Failures

**Guarantee:** Cryptographic evidence makes fraud provable. Economic constraints make theft unprofitable. Recovery ensures funds remain available.

**NOT Guaranteed:** Protection against scenarios where all partners simultaneously fail (datacenter destruction, nation-state attack on Lightning infrastructure, etc.).

**Why that's okay:** These scenarios also break base Lightning Network assumptions. Deposits is a Layer 3 protocol built on Lightning—if Lightning fails catastrophically, L3 protocols fail too. This isn't a Deposits-specific vulnerability.

### 5. No Trust Required

**Guarantee:** Trust is minimized through economics, cryptography, and transparency.

**NOT Guaranteed:** Zero trust (that's impossible in multi-party systems).

**Why that's okay:** Users trust that:
- Multiple independent parties won't ALL collude against their economic interests
- Cryptographic evidence is unforgeable (SHA256, ECDSA security)
- At least ONE partner will be honest and online during recovery

These are weaker trust assumptions than "trust this single custodian completely." That's the measure of success—meaningful improvement, not perfection.

## Conclusion: The Right Comparison

The relevant question isn't "Is Deposits perfect?" but "Is it better than the alternatives for users who need custodial Lightning?"

**Current custodial Lightning:**
- Trust operator completely
- Zero recourse if operator steals
- No transparency into solvency
- Funds disappear if operator exits

**Deposits Protocol:**
- Trust distributed across multiple parties with capital at risk
- Cryptographic evidence makes theft provable
- Continuous transparency into collateral coverage
- Funds transfer to new operator if original fails

For users who can self-custody Lightning, Deposits offers nothing. For users who can't or won't (which is most users), Deposits transforms the security model from "hope for the best" to "economic and cryptographic constraints make theft irrational."

The protocol has limitations, yes. Recovery can be slow. Service quality may degrade temporarily. Some operators may become dominant. These are real tradeoffs. But they're VASTLY better than the current situation where custodial users have zero protection.

The protocol is sound. It achieves its security goals for its intended use case. The attacks that seem devastating on paper fail against the economic and cryptographic constraints. The assumptions are reasonable for anyone running Lightning infrastructure.

Is it perfect? No. Is it a meaningful improvement that serves a real, underserved user base? Absolutely.
