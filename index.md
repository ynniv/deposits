Abstract
===
A peer-to-peer system for custodial Bitcoin payments would allow users without the capacity to manage on-chain UTXOs to transact with reduced trust in any single party. We propose a solution using Lightning channels between custodians who post collateral exceeding their reserves. Custodians maintain transparent ledgers, and partners can verify that ledger updates match on-chain commitments. Misbehavior is provable and results in collateral forfeiture. Critically, seized collateral replaces stolen reserves and partners who take custody of the ledger also receive funds sufficient to honor it. Wallets need not be aware that recovery has occurred. Liability is transitive: partners who fail to act against provable misbehavior are themselves subject to slashing by their partners. The result is a network where theft requires collusion of a majority of bonded participants, and such collusion is publicly visible. Users cannot exit to chain unilaterally, but need not take any action to recover from custodian failure. The security model parallels the base layer: cryptographic authorization with economic enforcement against majority collusion, while serving users for whom on-chain settlement is impractical.

Introduction
===
Bitcoin's security relies on users holding their own keys and validating their own transactions. Layer 2 protocols like Lightning extend this model by allowing off-chain payments with on-chain enforcement. However, both require users to manage UTXOs, a resource that cannot scale to billions of users given block space constraints.

In practice, many users rely on custodians. Current custodial solutions offer convenience but require complete trust. A dishonest custodian can steal funds with no on-chain recourse. Ecash improves privacy but does not address this fundamental trust problem.

We propose a protocol where custodians form a network of Lightning channels with collateral requirements. Each custodian maintains a transparent ledger of user balances, with state commitments published on-chain. Partners independently verify ledger integrity. Misbehavior triggers collateral seizure, with seized funds transferred alongside the ledger to a new custodian. Because collateral exceeds reserves, the new custodian can fully honor existing balances. Users experience only a change in counterparty; their balances remain intact and usable without any action on their part.

Because partners who ignore misbehavior risk their own collateral, accountability propagates through the network. A partner who fails to challenge a provably dishonest neighbor can themselves be challenged by their own partners. Collusion must therefore grow to encompass the network or be isolated and penalized at its boundary.

The system explicitly does not provide unilateral exit. Users who cannot afford on-chain fees would not benefit from such a mechanism. Instead, recovery occurs automatically through ledger reassignment, keeping users within a layer they can operate in.

FAQ
===
q: was any of this written by an llm?

a: yes. all of it.
