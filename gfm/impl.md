# Persistence

For implementing the persistence layer of the AVM, we assume that we have access to an updatable zero-knowledge
elementary database (ZK-EDB) with the following properties:

  - The ZK-EDB contains a mapping from a set of keys to a set of values.

  - Every state of the database has a commitment ![equation](https://latex.codecogs.com/gif.latex?\inline&space;C).

  - The ZK-EDB has a method ![equation](https://latex.codecogs.com/gif.latex?\inline&space;(D,&space;p)&space;=&space;get(x)), where ![equation](https://latex.codecogs.com/gif.latex?\inline&space;x) is a key and ![equation](https://latex.codecogs.com/gif.latex?\inline&space;D) is the associated value with ![equation](https://latex.codecogs.com/gif.latex?\inline&space;x) and
    ![equation](https://latex.codecogs.com/gif.latex?\inline&space;p) is a proof.

  - A user can use ![equation](https://latex.codecogs.com/gif.latex?\inline&space;C) and ![equation](https://latex.codecogs.com/gif.latex?\inline&space;p) to verify that ![equation](https://latex.codecogs.com/gif.latex?\inline&space;D) is really associated with ![equation](https://latex.codecogs.com/gif.latex?\inline&space;x), and ![equation](https://latex.codecogs.com/gif.latex?\inline&space;D) is not altered.
    Consequently, a user who can obtain ![equation](https://latex.codecogs.com/gif.latex?\inline&space;C) from a trusted source does not need to trust the ZK-EDB.

  - Having ![equation](https://latex.codecogs.com/gif.latex?\inline&space;p) and ![equation](https://latex.codecogs.com/gif.latex?\inline&space;C) a user can compute the commitment ![equation](https://latex.codecogs.com/gif.latex?\inline&space;C') for the database in which ![equation](https://latex.codecogs.com/gif.latex?\inline&space;D') is associated with
    ![equation](https://latex.codecogs.com/gif.latex?\inline&space;x) instead of ![equation](https://latex.codecogs.com/gif.latex?\inline&space;D).

We use a ZK-EDB for storing the AVM heap. We include the commitment of the current state of this DB in every block of
the Algorand blockchain, so ZK-EDB servers need not be trusted servers.

Every page of AVM heap will be stored with a key of the form: `applicationID|pageIndex` (the `|` operator concatenates
two numbers). Nodes do not keep a full copy of the AVM heap and for validating block certificates or emulating the AVM (
i.e. validating transactions) they need to connect to a ZK-EDB and retrieve the required pages of AVM heap. For better
performance, nodes keep a cache of heap pages to reduce the amount of ZK-EDB access.

We also use a ZK-EDB for storing the code area of each segment, and we include the commitment of this DB in every block.
Every code area will be divided into blocks and every block will be stored in the DB with `applicationID|blockID` as its
key. Like heap pages, nodes keep a cache of code area blocks.

*Unlike heap pages, the AVM is not aware of different blocks of the code area.*

# Transactions

Algorand has four types of transaction:

  - `avmCall` essentially is an `invoke_external` instruction that invokes a method from an AVM smart contract. Users
    interact with AVM smart contracts using these transactions. Transferring all assets, including ALGOs, is done by
    these transactions.

  - `installApp` installs an AVM smart contract and determines the update policy of the smart contract: if the contract
    is updatable or not, the accounts that can update or uninstall the contract, and so on.

  - `unInstallApp` removes an AVM smart contract.

  - `updateApp` updates an AVM smart contract.

All types of Algorand transactions contain an `invoke_external` instruction which calls a special method from ALGO smart
contract that transfers the proposed fee of the transaction in ALGOs from a sender account to the fee sink accounts.

Every transaction is required to exactly specify what heap pages or code area blocks it will access. This enables
validators to start retrieving the required memory blocks from available ZK-EDB servers as soon as they see a
transaction, and they won’t need to wait for receiving the new proposed block. A transaction that tries to access a
memory block that is not included in its access lists, will be rejected. Users could use smart contract oracles to
predict the list of memory blocks their transactions need. See Section [7](#sec:smart-contract-oracle) for more details.

# Blockchain

Every block of the Algorand blockchain corresponds to a set of transactions. We store the commitment of this transaction
set in every block, but we don’t keep the set itself. To be able to detect replay attacks, we require every signature
that a user creates to have a nonce. This nonce consists of the issuance round of the signature and a sequence number:
`(issuance, sequence)`. When a user creates more than one signature in a round, he must sequence his signatures starting
from 0 (i.e. the sequence number restarts from 0 in every round). We define a maximum lifetime for signatures, so a
signature is invalid if `currentRound - issuance > maxLifeTime` or if a signature of the same user with a bigger or
equal nonce is already used (i.e . is recorded in the blockchain). A nonce is bigger than another nonce if it has an
older issuance. If two nonces have an equal issuance, the nonce with the bigger sequence number will be considered
bigger.

To be able to detect invalid signatures, we keep the maximum nonce of used digital signatures per user. When the
difference between `issuance` component of this nonce and the current round becomes bigger than the maximum allowed
lifetime of a signature, this information can be safely deleted. **As a result, we will not have the problem of "
unremovable empty accounts" like Ethereum.**

The only information that Algorand nodes are required to store is **the most recent block** of the Algorand blockchain.
Every block of the Algorand blockchain contains the following information:

|                    Block                    |
| :-----------------------------------------: |
| commitment to the ZK-EDB storing heap pages |
| commitment to the ZK-EDB storing code areas |
|    commitment to the set of transactions    |
|             previous block hash             |
|                 random seed                 |

For confirming a new block, nodes that are not validators only need to verify the block certificate. For verifying a
block certificate, a node needs to know the ALGO balances of validators, but it doesn’t need to emulate the AVM
execution.

On the other hand, nodes that are chosen to be validators, for validating a new block, need to emulate the execution of
the Algorand virtual machine. To do so, first they retrieve all heap pages and code area blocks they need from available
ZK-EDBs. Then, they emulate the execution of AVM instructions and validate all the transactions included in the new
block. This will modify some pages of the AVM memory, so they update the ZK-EDB commitments based on the modified pages
and verify the commitments included in the new block. Validators also calculate and verify the commitment to the new
block’s transaction set.

*Validators do not need to write the modified pages back to ZK-EDB servers. ZK-EDBs will receive the new block, and they
will update their database by emulating the AVM execution.*

# Incentive mechanism

## Transaction Fee

Every transaction in the Algorand blockchain starts with an `invoke_external` instruction which calls a special method
from ALGO smart contract. This method will transfer the proposed fee of the transaction in ALGOs from a sender account
to the fee sink accounts. Algorand has two fee sink accounts: `execFeeSink` collects execution fees and `dbFeeSink`
collects fees for ZK-EDBs. The Protocol decides how to distribute the transaction fee between these two fee sink
accounts.

When a block is added to the blockchain, the proposer of that block will receive a share of the block fees.
Consequently, a block proposer is always incentivized to include more transactions in his block. However, if he puts too
many transactions in his block and the validation of the block becomes too difficult, some validators may not be able to
validate all transactions on time. If a validator can not validate a block in the required time, he will consider the
block invalid. So, when a proposed block contains too many transactions, the network may reach consensus on another
block, and the proposer of that block will not receive any fees. As a result, a proposer is incentivized to use network
transaction capacity optimally.

On the other hand, we believe that the proposer does not have enough incentives for optimizing the storage size of the
transaction set. Therefore, we require that **the size of the transaction set of every block in bytes be lower than a
certain threshold.**

Validators need to spend resources for validating transactions. When a validator starts the emulation of the AVM to
validate a transaction, solely from the code he can’t predict the time the execution will finish. This will give an
adversary an opportunity to attack the network by broadcasting transactions that never ends. Since, validators can not
finish the execution of these transactions, the network will not be able to charge the attacker any fees, and he would
be able to waste validators resources for free.

To mitigate this problem, we require that every transaction specify a cap for all the resources it needs. This will
include memory, network and processor related resources. Also, the protocol defines an execution cost for every AVM
instruction reflecting the amount of resources its emulation needs. This will define a standard way for measuring the
execution cost of any `avmCall` transaction. Every `avmCall` transaction is required to specify a maximum execution
cost. If during emulation it reaches this maximum cost, the transaction will be considered failed and the network can
receive the proposed fee of that transaction.

Every `avmCall` transaction is required to provide the following information as an upper bound for the resources it
needs:

  - Execution cost

  - A list of heap/code-area pages for reading

  - A list of heap pages for writing

  - A list of heap pages it will deallocate

  - Number and size of heap pages it will allocate

If a transaction tries to violate any of these predefined limitations, for example, if it tries to read a memory
location that is not included in its reading list, it will be considered failed and the network can receive the proposed
fee of that transaction.

*A transaction always pays all of its proposed fee, no matter how much of its predefined resources were not used in the
final emulation.*

## ZK-EDB Servers

The incentive mechanism for ZK-EDB servers should have the following properties:

  - It incentivizes storing all memory blocks, whether a heap page or a code area block, and not only those that are
    used more frequently.

  - It incentivizes the ZK-EDB servers to actively provide the required memory blocks for validators.

  - Making more accounts will not provide any advantages for a ZK-EDB server.

For our incentive mechanism, we require that every time a validator receives a memory block from a ZK-EDB, after
validating the data, he give a receipt to the ZK-EDB. In this receipt the validator signs the following information:

  - `ownerAddr` the ALGO address of the ZK-EDB.

  - `receivedBlockID` the ID of the received memory block.

  - `round` the current round number.

*In a round, an honest validator never gives a receipt for an identical memory block to two different ZK-EDBs.*

To incentivize ZK-EDB servers, every round a lottery will be held and a predefined amount of ALGOs from `dbFeeSink`
account will be distributed between winners as a prize. This prize will be divided equally between all the *winning
tickets* of the lottery.

*One ZK-EDB server could own multiple winning tickets in a round.*

To run this lottery, In every round, based on the current block seed, a collection of *valid* receipts will be selected
randomly as the *winning* receipts. A receipt is *valid* in the round ![equation](https://latex.codecogs.com/gif.latex?\inline&space;r) if:

  - The signer was a validator in the round ![equation](https://latex.codecogs.com/gif.latex?\inline&space;r&space;-&space;1) and voted for the agreed-upon block.

  - The data block in the receipt was needed for validating the **previous** block.

  - The receipt round number is ![equation](https://latex.codecogs.com/gif.latex?\inline&space;r&space;-&space;1).

  - The signer did not sign a receipt for the same data block for two different ZK-EDBs in the previous round.

For selecting the winning receipts we could use a random generator:

``` 
        if random(seed|validatorPK|receivedBlockID) < winProbability
            the receipt issued by validatorPK for receivedBlockID is a winner
```

  - `random()` produces uniform random numbers between 0 and 1, using its input argument as a seed.

  - `validatorPK` is the public key of the signer of the receipt.

  - `receivedBlockID` is the ID of the memory block that the receipt was issued for.

  - `winProbability` is the probability of winning in every round.

  - `seed` is the current block seed.

  - `|` is a concatenation operator.

*The winners of the lottery were validators one round before the lottery round.*

Also, based on the current block seed, a random memory block, whether a heap page or a code area block, is selected as
the challenge of the round. A ZK-EDB that owns a winning receipt needs to broadcast a *winning ticket* to claim his
prize. The winning ticket consists of a winning receipt and a *solution* to the round challenge. Solving a round
challenge requires the content of the memory block which was selected as the round challenge. This will encourage
ZK-EDBs to store all memory blocks.

A possible choice for the challenge solution could be the cryptographic hash of the content of the challenge memory
block combined with the ZK-EDB ALGO address: `hash(challenge.content|ownerAddr)`

The winning tickets of the lottery of the round ![equation](https://latex.codecogs.com/gif.latex?\inline&space;r) need to be included in the block of the round ![equation](https://latex.codecogs.com/gif.latex?\inline&space;r), otherwise
they will be considered expired. Validation and prize distribution for the winning tickets of the round ![equation](https://latex.codecogs.com/gif.latex?\inline&space;r) will be
done in the round ![equation](https://latex.codecogs.com/gif.latex?\inline&space;r&space;+&space;1). This way, **the content of the challenge memory block could be kept secret during the
lottery round.** Every winning ticket will get an equal share of the lottery prize.

## Memory Allocation and De-allocation

Every k round the protocol chooses a price per byte for the AVM memory. When a smart contract executes a heap allocation
instruction, the protocol will automatically deduce the cost of the allocated memory from the ALGO address of the smart
contract.

To determine the price of AVM memory, Every k round, the protocol calculates `dbFee` and `memTraffic` values. `dbFee` is
the aggregate amount of collected database fees and `memTraffic` is the total memory traffic of the system. For
calculating the memory traffic of the system the protocol considers the total size of all memory pages that were
accessed for either reading or writing during some time. These two values will be calculated for the last k rounds and
the price per byte of the AVM memory will be a linear function of `dbFee/memTraffic`

When a smart contract executes a heap de-allocation instruction, the protocol will refund the cost of de-allocated
memory to the smart contract. Here, the current price of AVM memory does not matter and the protocol calculates the
refunded amount based on the average price the smart had paid for that allocated memory. This will prevent smart
contracts from profit taking by trading memory with the protocol.

# Concurrency

Every block of the Algorand blockchain contains a list of transactions. This list is an ordered list and the effect of
its contained transactions must be applied to the AVM state sequentially as they appear in the ordered list. The
ordering of transactions in a block is solely chosen by the block proposer.

*Users should not have any assumption about the ordering of transactions in a block.*

The fact that block transactions constitute a sequential list, does not mean they can not be validated concurrently.
Many transactions are actually independent and the order of their execution does not matter. These transactions can be
safely validated in parallel by validators.

A transaction can change the AVM state by modifying either the code area or the AVM heap. In Algorand, all transactions
declare the list of memory blocks they want to read or write. This will enable us to determine the independent sets of
transactions which can be validated in parallel. To do so, we define the *memory dependency graph* as follows:

  - ![equation](https://latex.codecogs.com/gif.latex?\inline&space;G) is an undirected graph.

  - Every vertex in ![equation](https://latex.codecogs.com/gif.latex?\inline&space;G) corresponds to a transaction.

  - Vertices ![equation](https://latex.codecogs.com/gif.latex?\inline&space;u) and ![equation](https://latex.codecogs.com/gif.latex?\inline&space;v) are adjacent in ![equation](https://latex.codecogs.com/gif.latex?\inline&space;G) if and only if u has a memory block ![equation](https://latex.codecogs.com/gif.latex?\inline&space;B) in its writing list and
    ![equation](https://latex.codecogs.com/gif.latex?\inline&space;v) has ![equation](https://latex.codecogs.com/gif.latex?\inline&space;B) in either its writing list or reading list.

If we consider a proper vertex coloring of ![equation](https://latex.codecogs.com/gif.latex?\inline&space;G), every color class will give us an independent set of transactions that
can be validated concurrently. To achieve the highest parallelization, we need to color ![equation](https://latex.codecogs.com/gif.latex?\inline&space;G) with minimum number of
colors. Thus, the chromatic number of the memory dependency graph shows how good a transaction set could be run
concurrently.

Graph coloring is computationally NP-hard. However, in our use case we don’t need to necessarily find an optimal
solution. An approximate greedy algorithm will perform well enough in most circumstances. The block proposer is
responsible for solving the graph coloring problem anda proposed block must determine the independent sets of
transactions which can be run in parallel safely. Since with better parallelization a block can contain more
transactions, a proposer is incentivized enough to find a good graph coloring.

# Consensus

## Estimating A User’s Stake

In a proof of stake system the influence of a user in the consensus protocol should be proportional to the amount of
stake the user has in the system. Conventionally in these systems, for estimating a user’s stake, we use the amount of
native system tokens the user is holding. Unfortunately, one problem with this approach is that a strong attacker may be
able to obtain a considerable amount of system tokens, for example by borrowing from a DEFI application, and use this
stake to attack the system.

To mitigate this problem, for calculating a user’s stake, instead of using the raw ALGO balance, we use the minimum of a
*trust value* that the system has calculated for the user and the user’s ALGO balance:


![equation](https://latex.codecogs.com/png.latex?\dpi{120}&space;Stake_{user}&space;=&space;\min&space;(Balance_{user},Trust_{user}))

 For estimating the value of ![equation](https://latex.codecogs.com/gif.latex?\inline&space;Trust_{user}) we use the
exponential moving average of the user’s ALGO balance. Therefore, in our system a user who held ALGOs and participated
in the consensus for a long time is more trusted than a new user with a higher balance. An attacker who has obtained a
large amount of ALGOs, also needs to hold them for a long period of time before being able to attack our system.

For calculating the exponential moving average of a time series at the time step ![equation](https://latex.codecogs.com/gif.latex?\inline&space;t), we can use the following
recursive formula: 

![equation](https://latex.codecogs.com/png.latex?\dpi{120}&space;M_t&space;=&space;(1&space;-&space;\alpha)&space;M_{t-1}&space;+&space;\alpha&space;X_t&space;=&space;M_{t-1}&space;+&space;\alpha&space;(X_t&space;-&space;M_{t-1}))

 Where:

  - The coefficient ![equation](https://latex.codecogs.com/gif.latex?\inline&space;\alpha) is a constant smoothing factor between 0 and 1 which represents the degree of weighting
    decrease, A higher ![equation](https://latex.codecogs.com/gif.latex?\inline&space;\alpha) discounts older observations faster.

  - ![equation](https://latex.codecogs.com/gif.latex?\inline&space;X_t) is the value of the time series at the time step ![equation](https://latex.codecogs.com/gif.latex?\inline&space;t).

  - ![equation](https://latex.codecogs.com/gif.latex?\inline&space;M_t) is the value of the EMA at the time step ![equation](https://latex.codecogs.com/gif.latex?\inline&space;t).

Usually an account balance will not change in every time step, and we can use older values of EMA for calculating
![equation](https://latex.codecogs.com/gif.latex?\inline&space;M_t): 

![equation](https://latex.codecogs.com/png.latex?\dpi{120}&space;M_t&space;=&space;(1&space;-&space;\alpha)^{t-k}M_k&space;+&space;[1&space;-&space;(1&space;-&space;\alpha)^{t&space;-&space;k}]X)

 Where: 

![equation](https://latex.codecogs.com/png.latex?\dpi{120}&space;X&space;=&space;X_{k+1}&space;=&space;X_{k+2}&space;=&space;\dots&space;=&space;X_t)


When ![equation](https://latex.codecogs.com/gif.latex?\inline&space;|nx|&space;\ll&space;1) we can use the binomial approximation ![equation](https://latex.codecogs.com/gif.latex?\inline&space;(1&space;+&space;x)^n&space;\approx&space;1&space;+&space;nx) to further simplify this formula:


![equation](https://latex.codecogs.com/png.latex?\dpi{120}&space;M_t&space;=&space;M_k&space;+&space;(t&space;-&space;k)&space;\alpha&space;(X&space;-&space;M_k))



For choosing the value of ![equation](https://latex.codecogs.com/gif.latex?\inline&space;\alpha) we can consider the number of time steps that the trust value of a user needs for
reaching a specified fraction of his account balance. We know that for large ![equation](https://latex.codecogs.com/gif.latex?\inline&space;n) and ![equation](https://latex.codecogs.com/gif.latex?\inline&space;|x|&space;<&space;1) we have
![equation](https://latex.codecogs.com/gif.latex?\inline&space;(1&space;+&space;x)^n&space;\approx&space;e^{nx}), so by letting ![equation](https://latex.codecogs.com/gif.latex?\inline&space;M_k&space;=&space;0) and ![equation](https://latex.codecogs.com/gif.latex?\inline&space;n&space;=&space;t&space;-&space;k) we can write:


![equation](https://latex.codecogs.com/png.latex?\dpi{120}&space;\alpha&space;=-&space;\frac{\ln\left(1&space;-&space;\frac{M_{n+k}}{X}\right)}{n})

 The value of ![equation](https://latex.codecogs.com/gif.latex?\inline&space;\alpha) for a desired configuration can be
calculated by this equation. For instance, we could calculate the ![equation](https://latex.codecogs.com/gif.latex?\inline&space;\alpha) for a relatively good configuration in
which ![equation](https://latex.codecogs.com/gif.latex?\inline&space;M_{n+k}&space;=&space;0.8X) and ![equation](https://latex.codecogs.com/gif.latex?\inline&space;n) equals to the number of time steps of 10 years.

# Smart Contract Oracle

A smart contract oracle is a full AVM emulator that keeps a full local copy of AVM memory and can emulate AVM execution
without accessing aZK-EDB. Smart contract oracles can be used for reporting useful information about `avmCall`
transactions such as accessed AVM heap pages or code area blocks, exact amount of execution cost, and soon.