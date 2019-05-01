# Ethereum Essence: Complex Consensus Operations

"Imagine all the people living life in peace. Yoo-hoo..." The soul yearns for social peace, but to get there, the hard problem of consensus needs solutions. Here we are at it again.

First about the context:
The most important piece of a decentralized system is its consensus layer. But, what does "consensus" mean? By definition, it is an opinion or position reached by a group as a whole. It can imply unanimity or a judgment arrived at by most of those concerned.

A consensus protocol can generally be thought of as having:
- a set of proponents who make proposals
- a set of tests which marks a proposal as valid
- a set of voters who vote on each proposal
- a voting mechanism that decides whether a proposal is accepted or dismissed
- a set of actions triggered when a decision is reached

The voting mechanism can:
- set rules regarding who can vote and whether the voters need to pass a knowledge test
- have algorithms to calculate the weight of each vote
- set voting score or time thresholds for triggering post-vote or during-vote actions

To formalize: we have a `Consensus` function, with input arguments: proposal, voting mechanism, voting score at `timestamp`, a voting pool. The output will be the decision snapshot at time `timestamp`: whether the proposal is considered accepted or not at a certain point in time, based on what voting mechanism is used and the voting data aggregated up to that point.

The voting pool is the total amount of voters who have the right to vote, regardless if they vote or not.

```
function Consensus(
    Proposal proposal,
    VotingMechanism mechanism,
    address[] votingPool,
    VotingScore voting_score_at_timestamp
)
    public
    returns (Decision decision);
```

## The Consensus Engine

Ethereum currently uses Proof of Work as its consensus protocol. We will take a brief look at how it works.

The first stage can be thought of as a consensus protocol in itself:
Users can propose transactions that are in fact chain state transitions. They use their Ethereum clients to add the transaction in the transaction queue and propagate it further, to its connected peers. These peers follow the same process and the transaction finally reaches Ethereum clients that can also mine transactions. At this point, each miner acts like a "voter", deciding if they want to include the transaction or not in the next block. They have their own voting mechanism, based on the associated gas price and gas cost of the transaction. They also need to pass a hard cryptographic test in order to have the right to propose blocks. The decision is the form of an array of transactions, that can make up the next block.

The second stage is about client consensus on what chain version should be further extended with blocks. Miners propose blocks and due to network propagation delays, multiple chain forks coexist at the same time. Each client node "votes" on what fork to choose as valid, based on the Greedy Heaviest Observed Subtree (GHOST) protocol, that Ethereum currently uses.

But Ethereum has Turing Complete expressiveness and this means it can become a platform for building higher-level consensus.

## Boolean Consensus

In the following parts, we will explore only boolean value decisions (similar to binary consensus). Meaning, a voter can assign either `false` or `true` to a proposal.

In order to build a general (boolean-based) consensus protocol, one should consider fine-grained voting on:

- function execution
- data type
- data type and record
- address
- a mix of the above

## Consensus Memoization

Consensus implies that a group of people agrees on something. For this to be efficient, the agreed upon result must be registered. It must be "memoized", in order to not go through the voting process time and time again.

The results can be stored on-chain as decisions, permissions, and claims. Additional information about the result can be attached - e.g. adding a Swarm hash of a file containing voting statistics and data.

We can go further and say that the voting process can be ongoing (on or off-chain) - people can change their minds if opinions or assumptions change. Therefore, permissions can be seen as memoized results at a point in time.

### Permissions on Function execution

Such permissions are not limited to a consensus on who (EOAs) and what contracts can execute individual contract functions: one can additionally have permissions that control what functions can call/execute a given function.
You can also define classes of functions that have similar types of permissions. For example, `update()` and `remove()` functions may have the same permissions, based on the executor of an `insert()`.

If you want to build an interoperable consensus engine on top of Ethereum, you might want to differentiate between EVM and non-EVM environments. Imagine that you want to use a function inside the EVM protocol: such a function needs to be `public`. However, you may not want this function to be called from non-EVM environments - e.g. from a JavaScript dapp.

### Permissions on Data Type and Record

Imagine a consensus protocol for identity: users might decide to give CRUD permissions over one identity data type to some trusted 3rd party. One such a 3rd party is able to verify only one part of the user's identity and manage it on-chain, on behalf of the user. A second one can manage another identity data type. Think about Twitter attaching a user handle proof to a user's identity contract or a government entity attaching an identification hash.

You can, therefore, have fine-grained permissions on data types and on the data records themselves.

### Permissions on Address

Such permission can be thought of as a role-based access control mechanism.
An Ethereum address (EOAs or contracts) can be granted permissions by another address with higher permissions, on giving further permissions down the hierarchy.

### Permissions Mix

The above types of permissions can be combined. For example, you can give an Ethereum address only the permission to update the value of a data record attribute of a certain data type, without allowing it to update other attributes or removing that record.

### Claims

Some example of claims are:
- a user is granted the right to be a technical reviewer for protocol changes
- a user claims to be older than 18 years and therefore legally permitted to use a service

A user can receive a certain status, role or even ownership over a digital good as a result of a voting process. The claim is a permanent recording of this result.

## Consensus Boolean Operations

Let us imagine that we have some established permissions. We can define additional permissions by consensus on Boolean Operations on extant permissions.

For example, Bob wants to make a change in a public filesystem folder. He wants to move a file from that folder into another one. The system needs to check Bob's permissions on: the file, the file's update function for changing parents, the current parent folder and its update function for removing a child file, the new folder and its update function for adding a child file.

Bob will be allowed to move that file only if the end result of the aggregated permissions is true.

The sequence of boolean operations needed to calculate whether Bob can move the file or not can be done by proposing and voting. Once the sequence is accepted, it can be used by anyone who would want to do the same.

## Meta-consensus: Consensus Chaining

We have talked about some of the puzzle pieces. The puzzle itself is a general consensus engine. A protocol that allows you to build other consensus protocols by using itself to reach a consensus on the new protocol's design.

The flow would look like this: Alice proposes a new voting mechanism and submits it on-chain (the mechanism itself can even contain pointers to Swarm-stored script files). Alice does not have permission to add the voting mechanism to the protocol directly. The system, therefore, announces the submission and creates a new voting resource. People can now discuss and vote on it (on or off-chain). Developers can run tests on this new mechanism, in the form that was submitted.

If the vote is successful, Alice's voting mechanism can be fully registered into the protocol and Alice can automatically be granted some new permissions - e.g. "reviewer of mechanism updates".

And so on. This is a consensus on a future consensus.

## Consensus Lifecycle

There is a difference between consensus on facts and consensus on opinions. The consensus on opinions suffers from mutability and erosion.

While consensus on facts can also be (at least partially) reached programmatically, consensus on opinions may require a flexible allow/disallow protocol that runs continuously. Therefore, the lifecycle can be:
- creation
- mutation
- erosion

I have already mentioned permissions as having a limited life span, in the sense that they can be changed if the ongoing vote result changes.

From a moral perspective, anything that needs consensus should be implemented as a continuous voting process. People should be able to withdraw their support for an idea at any moment. A system can only be democratic and moral if:
- anyone can propose a change and start a voting process for it
- any previously approved change can be reverted by voting (context has changed, an influx of new voters, old voters disappear, etc.)

Implementing continuous voting on protocols:
- people can submit a protocol change that contains all source code and information needed to test and audit it
- voting on the protocol change is enabled
- if the positive threshold vote score is reached, the change is automatically accepted
- voting can continue and if the negative vote score is reached, the change is automatically disabled again
- at this point, voting can be left to continue or a proposal for the removal of the proposed change can be done

One could have a set of automated tests that can be run over the proposed change, and, if the tests do not pass, the proposal can be flagged and voting paused. This can be done with decentralized protocols for trustless computation.

## Conclusions

Today we have only explored ways to use binary consensus on Ethereum. There are more types of consensus that can be explored. But this, for another time.

From this respect, the Ethereum consensus engine has a lot of room for growth and evolution. We are in the incipient stages of exploration of what different types of consensus can achieve. Complex consensus operations will expand and open a new horizon of discovery.

Therefore, I say to the Blockchain naysayers: You ain't seen nothing yet!
