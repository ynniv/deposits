# Why Stealing Deposit Funds is Hard: A Security Analysis

## The Challenge for Would-Be Thieves

The Bitcoin Deposits protocol makes theft surprisingly difficult through a combination of cryptographic enforcement, economic incentives, and forced transparency. Here's why a malicious vault operator can't easily steal user funds.

## The Three-Party Security Model

To steal funds, you need three things to align:

1. **A Vault Node** - Holds the deposits
2. **A Channel Partner** - Must sign any changes to deposit balances
3. **Auditor Approval** - Without it, users won't trust your vault

This creates a fundamental security property: **you can't steal alone**.

## Why Attacks Fail

### Attack 1: Just Stop Processing Withdrawals
**What happens:**
- Auditors detect 0% withdrawal success rate
- Your reputation immediately tanks
- Channel partner sees the metrics and force-closes
- Validated outputs ensure 120% of deposits go to neutral party
- You lose your 20% security deposit
- **Result: You lose money, steal nothing**

### Attack 2: Broadcast an Old Channel State
**What happens:**
- Standard Lightning penalty mechanism triggers
- Channel partner broadcasts penalty transaction
- ALL your channel funds go to partner
- 120% of deposits go to neutral party for distribution
- **Result: You lose everything, steal nothing**

### Attack 3: Forge Deposit Payments
**What happens:**
- Channel partner's validator rejects the invalid update
- They refuse to sign the new commitment
- You can't update the channel state without their signature
- If you try to force-close, see Attack 2
- **Result: Attack impossible to execute**

### Attack 4: Be Your Own Partner
**What happens:**
- No reputable auditor will recommend two unproven nodes
- You now need to run vault + partner + fake auditor
- This is obvious to everyone (self-channeling, self-auditing)
- Users see only your fake auditor approves you
- **Result: No users trust your obvious scam setup**

## The Overcollateralization Protection

The 120% collateral requirement creates unique protections:

### Guaranteed Liquidity
- $50,000 in deposits requires $60,000 in channel
- Even if all users exit at once, first-hop liquidity exists
- You can't claim "routing failures" to steal funds
- Downstream routing issues aren't your problem

### Mathematical Bounds on Theft
- To steal $X, you must risk losing 0.2×$X minimum
- Channel partner defection means losing everything
- No gradual theft possible - it's all or nothing

## Why "Honest Operation" Is Easier

Unlike traditional custodial services where operators can gradually become dishonest through:
- Fractional reserve practices
- "Borrowing" user funds temporarily
- Creative accounting
- Claiming technical issues

The Bitcoin Deposits protocol is **binary**:
- You operate 100% honestly (enforced by partners/auditors)
- You run a complete scam stack (obvious to everyone)

There's no middle ground for "light fraud."

## The Reputation Network Effect

In a mature ecosystem:
- **Established vaults** form a web of trust
- **Reputable auditors** have proven track records
- **Users** learn to check for multiple reputable auditor approvals
- **New scammers** can't break into this trust network

Creating a competing "scam ecosystem" requires:
- Running multiple infrastructure pieces
- Fooling users into ignoring reputation signals
- Competing against established, trusted operators
- Significant time and capital investment

## The Bottom Line

Stealing deposit funds requires either:

1. **Corrupting an entire trust network** (extremely difficult)
2. **Running an obvious scam stack** (no users will come)
3. **Buying established reputation** (expensive, sellers know the value)
4. **Long-term deception** (years of honest operation for one payday)

The protocol makes theft:
- **Technically complex** (need multiple coordinating nodes)
- **Economically punitive** (lose collateral on attempt)
- **Socially transparent** (obvious to auditors and users)
- **Practically difficult** (easier to just collect fees honestly)

While not perfectly trustless, the system creates enough friction, transparency, and economic disincentives that honest operation becomes the rational choice for most operators.
