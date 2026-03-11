# bitcoin deposits
## abstract

an ideal peer-to-peer version of electronic cash would allow online payments to be sent directly from one party to another quickly and with minimal setup. the lightning network provides part of the solution, but the essential benefits are lost if a trusted third party is required to manage state on your behalf. we propose a solution to this problem using cryptographic ledgers secured by a web of collateral. operators broadcast ledger updates to their peers, creating an auditable record of accounts. wallets broadcast evidence of dishonesty to those peers, who ensure that the ledger maintains an honest operator. unilateral exit is replaced by the guarantee that funds remain available so long as the network does. we arrive at a network that delegates liquidity maintenance, avoids setup fees, is capable of receiving payments offline, and scales independently of the base layer

## introduction

bitcoin deposits aims to provide fast, scalable, key controlled funds, trustlessly, mostly off-chain. on-chain activity scales with the number of ledgers and frequency of reserves rotation. throughput scales slightly above linearly with the number of ledgers in the network, making millions of transactions per second across trillions of wallets plausible

there are explicit tradeoffs:
- no unilateral exit – when operators fail funds stay in the network
- no privacy – verification requires transparency
- intermittent availability – a deposit is only as available as the operator. wallets should spread out funds to increase availability

we expect the wallet experience to be similar to a fast base layer, having payment economics similar to the lightning network. communication is channel agnostic, but will likely be based on nostr relays

## ledgers

a ledger is an immutable chain of updates, containing the hash of the previous update and signed by the ledger's operator. different types of update have different rules governing when and how they can be used. ledgers are self descriptive, their updates publicly available and non-repudiable, allowing anyone to evaluate conformance

ledgers have a single active operator, but are cooperatively maintained by the mesh. any operator can create one, but should they disappear or become dishonest a different operator will be assigned, along with reserves. the currently active operator is identified by the pubkey that was used to sign to most recent co-signed update

## deposits

a deposit is a stable account that can send and receive funds, controlled by miniscript. at opening a fee schedule is established, as well as whether receiving funds requires a wallet signed request. an operator must allow transfers between deposits on the same ledger as well as on-chain exits. they should allow deposits to pay lightning invoices. 

it is in the operator's discretion to create on-chain funding offers or lightning invoices on behalf of a deposit. if they do, these should be co-signed by a quorum member, and the wallet should verify this signature. offers and invoices are not part of the ledger, so it is the wallet's responsibility to verify signatures and retain them as evidence.

## fees

transfers between deposits, on-chain, and through lightning have fees paid to the ledger's operator. there are also fees periodically applied to balances with a specified period. all are negotiated when a new deposit is opened, and can be changed (within limits that have not yet been established) after a specified number of blocks, given a specified block notice. the quorum may refuse to co-sign updates that create unprofitable circumstances that they could ultimately be responsible for

## transfers

the basic form of transfer is a two phased operation between two deposits on the same ledger: a deposit issues a request to send funds. if there are sufficient funds available, a lock on the funds with a spending condition is appended to the ledger. if the spending condition is fulfilled before a timeout, funds move from the sender to recipient minus the operator's fee. if the timeout is reached, the lock is released, minus a smaller operator fee. with miniscript spending conditions, this is sufficient to allow any deposit to provide bridges and liquidity services to other deposits on the same ledger

## reserves and collateral

reserves are held in a utxo with an amount greater than or equal to the sum of a ledger's obligations, spendable by a majority of the quorum with fallback to the operator after a lengthy period. collateral is kept on other operators' ledgers, locked with specific terms. it may be attached to multiple ledgers to improve capital efficiency. wallets should prefer higher collateral ratios and non-overlapping quorum members

## quorum

operators request nodes where they store collateral to join a ledger's quorum. once membership is established, reserves are rotated into a new multisig utxo. each participant must operate their own a ledger, maintaining at least half the collateral as the one being joined. members co-sign valid updates and participate in recovery if the operator signs non-conforming ones. larger quorums increase communication overhead but reduce operator risk, increase availability, and make collusion more difficult and expensive. wallets should prefer larger quorums

## time

absolute time is measured against the base layer. tolerances should not exceed a reasonable number of confirmations in order to maintain stability during chain reorganizations. 

when higher tolerances are required, we instead rely on causal ordering. a cryptographic ledger is a merkle chain. each update proves it was created after all updates before it, but provides no guarantees about information outside the chain. in order to construct a distributed ordering, we require that co-signatures include the latest update hash from the co-signer's ledger. that hash then becomes incorporated into the ledger's chain, as well as part of all other chains that the ledger operator co-signs for, creating a web of causality. this is unable to prove time explicitly, but is able to prove that certain pieces of information were created in a specific order

## fraud proofs and bridging

we can then prove various types of fraud by exposing information which has been created in the wrong order. when the information is not included by normal network operations, it can smuggled in by creating activity that includes a hash of the evidence. once incorporated into an update signed by the operator, the evidence is revealed as having been created at a non-conforming place in the ordering:

- an operator, having offered to credit a deposit with funds sent on-chain to a specific address, signs a ledger update that does not contain the appropriate credit, but does contain a chain revealing some block hash exceeding the number of confirmations allowed before credit

- an operator, having created a lightning invoice on a deposit's behalf, signs a ledger update that has not credited the deposit despite the pre-image being revealed in the chain

- a co-signature that declares the current ledger hash to be one that precedes their own later hash in the chain

- a member of the quorum of a contested ledger who was active but did not act in accordance with proof of fraud within a number of blocks

- signing or co-signing non-conforming ledger updates

## recovery

once a ledger has become unavailable or non-conforming, quorum members may create their own continuation of the ledger from the last conforming update. they must establish a new quorum and provide collateral attestations. members must then coordinate to spend the previous reserves output to a lottery of the potential next chains. the winner of this lottery appends an acquisition update to their chain, and the others append a yield. wallets continue to address the same ledger, accepting only replies co-signed by the quorum. periodically, and when no replies have the expected co-signature, the wallet should query the network and replay ledger updates to identify changes in custody

when a ledger has become unavailable for a certain number of blocks, a change in custody must be respectful: only the amount of reserves required to cover the ledger's obligations is sent to the lottery, and change sent back to the operator's pubkey. control of collateral is unaffected. when proof of non-conformance is provided, the amount in excess of necessary reserves is split equally among members of the quorum. collateral is allowed to be confiscated by operators where it is held

## network health

one straightforward attack is to form islands of colluding operators. after building substantial obligations across their ledgers, they coordinate to exit, stealing funds that exceed the collateral lost. the network can defend against this, except in regions where the internal value exceeds the collateral connecting it to the non-colluding network. higher collateral ratios and larger, more diverse quorums reduce the likelihood of these pockets forming, but they can form on purpose and we can't expect every wallet to evaluate the entire network. instead discovery markets should publish metrics of operator accountability based on graph analyses such as prize-collecting algorithms

## conclusion

we propose a collateral network that requires collusion to steal, but collusion increases the collateral at risk faster than it increases the value to be stolen. we use this network to secure cryptographic ledgers backed by full reserves. these ledgers service accounts on behalf of offline wallets in exchange for pre-negotiated fees. ledger primitives support miniscript spending conditions sufficient for basic smart contracts. the network scales close to linearly, allowing a large network to provide billions of wallets and transaction volume in excess of traditional payment networks
