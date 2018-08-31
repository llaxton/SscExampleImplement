# MultisignatureEscrowAccount
Implement a stellar smart contract use scenario:2-Party Multisignature Escrow Account with Time Lock & Recovery 

### Use Case Scenario

Ben Bitdiddle sells 50 CODE tokens to Alyssa P. Hacker, under the condition that Alyssa won’t resell the tokens until one year has passed. Ben doesn’t completely trust Alyssa so he suggests that he holds the tokens for Alyssa for the year.

Alyssa protests. How will she know that Ben will still have the tokens after one year? How can she trust him to eventually deliver them?

Additionally, Alyssa is sometimes forgetful. There’s a chance she won’t remember to claim her tokens at the end of the year long waiting period. Ben would like to build in a recovery mechanism so that if Alyssa doesn’t claim the tokens, they can be recovered. This way, the tokens won’t be lost forever.

### Implementation

An escrow agreement is created between two entities: the origin - the entity funding the agreement, and the target - the entity receiving the funds at the end of the contract.

Three accounts are required to execute a time-locked escrow contract between the two parties: a source account, a destination account, and an escrow account. The source account is the account of the origin that is initializing and funding the escrow agreement. The destination account is the account of the target that will eventually gain control of the escrowed funds. The escrow account is created by the origin and holds the escrowed funds during the lock up period.

Two periods of time must be established and agreed upon for this escrow agreement: a lock-up period, during which neither party may manipulate (transfer, utilize) the assets, and a recovery period, after which the origin has the ability to recover the escrowed funds from the escrow account.

Five transactions are used to create an escrow contract - they are explained below in the order of creation. The following variables will be used in the explanation:

- **N**, **M** - sequence number of escrow account and source account, respectively; N+1 means the next sequential transaction number, and so on
- **T** - the lock-up period
- **D** - the date upon which the lock-up period starts
- **R** - the recovery period

For the design pattern described below, the asset being exchanged is the native asset. The order of submission of transactions to the Stellar network is different from the order of creation. The following shows the submission order, in respect to time:

![Diagram Transaction Submission Order for Escrow Agreements](https://www.stellar.org/developers/guides/walkthroughs/assets/ssc-escrow.png)

#### Transaction 1: Creating the Escrow Account

**Account**: source account
**Sequence Number**: M
**Operations**:

- Create Account

  : create escrow account in system

  - starting balance: [minimum balance](https://www.stellar.org/developers/guides/concepts/fees.html#minimum-account-balance) + [transaction fee](https://www.stellar.org/developers/guides/concepts/fees.html#transaction-fee)

**Signers**: source account

Transaction 1 is submitted to the network by the origin via the source account. This creates the escrow account, funds the account with the current minimum reserve, and gives the origin access to the public and private key of the escrow account. The escrow account is funded with the minimum balance so it is a valid account on the network. It is given additional money to handle the transfer fee of transferring the assets at the end of the escrow agreement. It is recommended that when creating new accounts to fund the account with a balance larger than the calculated starting balance.

#### Transaction 2: Enabling Multi-sig

**Account**: escrow account
**Sequence Number**: N
**Operations**:

- Set Option - Signer

  : Add the destination account as a signer with weight on transactions for the escrow account

  - weight: 1

- Set Option - Thresholds & Weights

  : set weight of master key and change thresholds weights to require all signatures (2 of 2 signers)

  - master weight: 1
  - low threshold: 2
  - medium threshold: 2
  - high threshold: 2

**Signers**: escrow account

Transaction 2 is created and submitted to the network. It is done by the origin using the escrow account, as origin has control of the escrow account at this time. The first operation adds the destination account as a second signer with a signing weight of 1 to the escrow account.

By default, the thresholds are uneven. The second operation sets the weight of the master key to 1, leveling out its weight with that of the destination account. In the same operation, the thresholds are set to 2. This makes is so that all and any type of transactions originating from the escrow account now require all signatures to have a total weight of 2. At this point, the weight of signing with both the escrow account and the destination account adds up to 2. This ensures that from this point on, both the escrow account and the destination account (the origin and the target) must sign all transactions that regard the escrow account. This gives partial control of the escrow account to the target.

#### Transaction 3: Unlock

**Account**: escrow account
**Sequence Number**: N+1
**Operations**:

- Set Option - Thresholds & Weights

  : set weight of master key and change thresholds weights to require only 1 signature

  - master weight: 0
  - low threshold: 1
  - medium threshold: 1
  - high threshold: 1

**Time Bounds**:

- minimum time: unlock date
- maximum time: 0

**Immediate Signer**: escrow account
**Eventual Signer**: destination account

#### Transaction 4: Recovery

**Account**: escrow account
**Sequence Number**: N+1
**Operations**:

- Set Option - Signer

  : remove the destination account as a signer

  - weight: 0

- Set Option - Thresholds & Weights

  : set weight of master key and change thresholds weights to require only 1 signature

  - low threshold: 1
  - medium threshold: 1
  - high threshold: 1

**Time Bounds**:

- minimum time: recovery date
- maximum time: 0

**Immediate Signer**: escrow account
**Eventual Signer**: destination account

Transaction 3 and Transaction 4 are created and signed by the escrow account by the origin. The origin then gives Transaction 3 and Transaction 4, in [XDR form](https://www.stellar.org/developers/horizon/reference/xdr.html), to the target to sign using the destination account. The target then publishes them for the origin to [review](https://www.stellar.org/laboratory/#xdr-viewer?type=TransactionEnvelope&network=test) and save in a safe location. Once signed by both parties, these transactions cannot be modified. Both the origin and target must retain a copy of these signed transactions in their XDR form, and the transactions can be stored in a publicly accessible location without concerns of tampering.

Transaction 3 and Transaction 4 are created and signed before the escrow account is funded, and have the same transaction number. This is done to ensure that the two parties are in agreement. If circumstances were to change before one of these two transactions are submitted, both the origin and the target need to authorize transactions utilizing the escrow account. This is represented by the requirement of the signatures of both the destination account and the escrow account.

Transaction 3 removes the escrow account as a signer for transactions generated from itself. This transaction transfers complete control of the escrow account to target. After the end of the lock-up time period, the only account that is needed to sign for transactions from the escrow account is the destination account. The unlock date (D+T) is the first date that the unlock transaction can be submitted. If Transaction 3 is submitted before the unlock date, the transaction will not be valid. The maximum time is set to 0, to denote that the transaction does not have an expiration date.

Transaction 4 is for account recovery in the event that target does not submit the unlock transaction. It removes the destination account as a signer, and resets the weights for signing transactions to only require its own signature. This returns complete control of the escrow account to the origin. Transaction 4 can only be submitted after the recovery date (D+T+R), and has no expiration date.

Transaction 3 can be submitted at any time during the recovery period, R. If the target does not submit Transaction 3 to enable access to the funds in the escrow account, the origin can submit Transaction 4 after the recovery date. The origin can reclaim the locked up assets if desired as Transaction 4 makes it so the target is no longer required to sign transactions for escrow account. After the recovery date, both Transaction 3 and Transaction 4 are valid and able to be submitted to the network but only one transaction will be accepted by the network. This is ensured by the feature that both transactions have the same sequence number.

To summarize: if Transaction 3 is not submitted by the target, then Transaction 4 is submitted by the origin after the recovery period.

#### Transaction 5: Funding

**Account**: source account
**Sequence Number**: M+1
**Operations**:

- [Payment](https://www.stellar.org/developers/guides/concepts/list-of-operations.html#payment): Pay the escrow account the appropriate asset amount

**Signer**: source account

Transaction 5 is the transaction that deposits the appropriate amount of assets into the escrow account from the source account. It should be submitted once Transaction 3 and Transaction 4 have been signed by the destination account and published back to the source account.
