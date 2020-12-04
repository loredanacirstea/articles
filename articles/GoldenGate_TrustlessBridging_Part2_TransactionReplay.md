---
tags: bridging
---

# Golden Gate - Trustless-Bridging Ethereum (EVM) Blockchains - Part 2: Transaction Replay

In [Trustless-Bridging Ethereum (EVM) Blockchains - Part 1: Basics](https://loredanacirstea.medium.com/golden-gate-trustless-bridging-ethereum-evm-blockchains-part-1-basics-d016300ea0dd) you saw that we have the tools to prove any momentary state change that happened on chain A, on another chain B, assuming that chain B has access to trustworthy chain A block hashes.

Today we will explore a mechanism for keeping smart contract state in sync between two EVM-based chains, as demoed in the following video:

https://youtu.be/WD_2tuX9jeg

Protocols usually implement replay protections to make sure malicious actors cannot execute state-changing transaction crafted on another chain. But purposefully implementing a mechanism for replaying state changes or syncing state snapshots directly, can be a useful pattern for keeping assets on multiple chains or migrating from one chain to another.

What does it mean for a state change to be replayed? Using the same signed transaction on chain B, that a user sent on chain A.

```solidity
struct Transaction {
    uint256 nonce;
    uint256 gasPrice;
    uint256 gasLimit;
    address to;
    uint256 value;
    bytes data;
    uint8 v;
    bytes32 r;
    bytes32 s;
}
```

**A successful replay requires the same setup on both chains:**
1) same contract address (transaction's `to` field)
2) same account address (transaction's `from` field)
3) same account nonce (number of transactions sent for EOAs / number of contracts created for contracts), checked by miners
4) with [EIP-155](https://eips.ethereum.org/EIPS/eip-155), same chain id, which is included in the signed transaction data

Therefore, to create a protocol that allows replaying transactions for the purpose of having correctly synced state between chains, we need a Proxy contract that fulfills the role of the miner: validates the transaction and runs it.

The above rules, allow us to reuse the exact same `calldata` on all chains where we want to have synced state changes. The Proxy contract will keep track of the account nonce per each chain ID that it supports and will forward the call to the appropriate contract address (`to` field), with some caveats for this receiving contract:
- it will need to know how to interface with the Proxy
- `msg.sender` will now be the Proxy address, so the original `msg.sender` must be additionally provided by the Proxy
- the Proxy will be given administrative rights to change state and it will be the only mechanism through which state is changed

```
participant relayer
participant user

relayer -> ChainA: subscribe('newBlockHeaders')
user -> ChainA: CounterA.incrementCount()
Note over ChainA: transaction mined
ChainA -> relayer: new block header
relayer -> LightClientB: addBlock()
user -> ChainA: get proof data
ChainA -> user: header, tx proof, receipt proof
user -> ProverChainB: header, tx proof, receipt proof
ProverChainB -> ProverChainB: verifyHeader()
ProverChainB -> LightClientB: getConfirmedBlockHash()
LightClientB -> ProverChainB: hashIsValid (bool)
ProverChainB -> ProverChainB: verifyTransaction()
ProverChainB -> ProverChainB: verifyReceipt()
ProverChainB -> ProverChainB: recoverOriginAddress()
ProverChainB -> ProverChainB: verifyOriginNonce()
ProverChainB -> ProverChainB: forwardTransaction()
ProverChainB -> ChainB: CounterB.incrementCount()
```


What are the steps for replaying a transaction on another chain?
* **get the transaction and receipt proofs, along with the header of the block in which the transaction was mined.**
* **send the proofs to the Proxy-Prover smart contract**. You can see the entire logic of the `forwardAndVerify` function [here](https://github.com/loredanacirstea/goldengate/blob/master/contracts/contracts/ProverStateSync.sol#L11-L55).
```solidity
function forwardAndVerify(
    EthereumDecoder.BlockHeader memory header,
    MPT.MerkleProof memory accountdata,
    MPT.MerkleProof memory txdata,
    MPT.MerkleProof memory receiptdata,
    uint256 chainId
)
    public returns (bytes memory);
```
* **check the block header & hash validity as shown in Part 1**
```solidity
(bool valid, string memory reason) = verifyHeader(header);
if (!valid) revert(reason);
```
* **check the [transaction proof](https://github.com/loredanacirstea/statebridge/blob/495abc40596e7b4cad519131d16874fbc844bd79/contracts/contracts/Prover.sol#L39-L51)**, as shown in Part1
```solidity
(valid, reason) = verifyTransaction(header, txdata);
if (!valid) revert(reason);
```
* **check the [receipt proof](https://github.com/loredanacirstea/statebridge/blob/495abc40596e7b4cad519131d16874fbc844bd79/contracts/contracts/Prover.sol#L53-L65)**, as shown in Part 1
```solidity
(valid, reason) = verifyReceipt(header, receiptdata);
if (!valid) revert(reason);
```
* **check the [account proof](https://github.com/loredanacirstea/statebridge/blob/495abc40596e7b4cad519131d16874fbc844bd79/contracts/contracts/Prover.sol#L67-L79)**, as shown in Part 1
```solidity
(valid, reason) = verifyAccount(header, accountdata);
if (!valid) revert(reason);
```
* **check transaction and receipt have the same index (the key used for each of the proofs) and therefore are directly correlated**
```solidity
require(
    keccak256(txdata.key) == keccak256(receiptdata.key),
    "Transaction & receipt index must be the same."
);
```
* **Decode RLP-encoded transaction, receipt, and account data**. The transaction data is needed for retrieving the `to` field - the address of the contract to which we forward the transaction. The receipt contains the `status` field, which tells us if the transaction must be successful or not. The account data contains the account `nonce`.
```solidity
Account memory account = toAccount(
    accountdata.expectedValue
);
Transaction memory transaction = toTransaction(
    txdata.expectedValue
);
TransactionReceipt memory receipt = toReceipt(
    receiptdata.expectedValue
);
```
* **recover the sender address and account nonce from the signed transaction data and verify that the nonce is what we expect**. We are effectively synchronizing the account nonce from chain A, with chain B in order to protect from replaying the same transaction multiple times. This `nonce` can be kept per each account, per each chain we are synchronizing with. It is updated regardless of success/fail status of the transaction.
```solidity
address sender = getTransactionSender(txdata, chainId);

if (accountNonces[sender] > 0) {
    require(
        account.nonce == accountNonces[sender] + 1,
        "Account nonce out of sync"
    );
}
accountNonces[sender] = account.nonce;
```
* **We can even check that the receiving contract has the same code on chain B as it has on chain A, by comparing the account's `codeHash` with the target contract `codeHash`**
```solidity
bytes32 codeHash;
address target = transaction.to;
assembly {
    codeHash := extcodehash(target)
}
require(account.codeHash == codeHash);
```
* **execute the call and check that it has the same status as the receipt**
```solidity
(bool success, bytes memory data) = transaction.to.call{
    value: transaction.value,
    gas: transaction.gasLimit
}(transaction.data);
uint8 _success = success ? uint8(1) : uint8(0);
require(
    _success == receipt.status,
    "Diverged transaction status"
);
return data;
```


## Transaction hygiene

For this system to work properly, it requires transaction hygiene. Nonces need to be kept in sync across chains and this means that cross-chain accounts need to use an account manager to send transactions and make sure they are not sent out of order.

Cross-chain synced contracts need to be deployed with the same address, which can easily be achieved by deploying them from the same account address, at the same nonce.

The EVM does not give access to transaction data (e.g. transaction hash) or the value of the account's nonce for that transaction. If there was such access, the need for storing the nonce on-chain (yet again) would not exist.

## Shadow payments

This transaction replay pattern can be used for shadow payments.

* **lower value chain payment precedes higher value chain payment.** One can pay on a testnet as a promise to buy a product.
* **lower value chain payment postcedes higher value chain payment.** One can send a payment on the mainnet, use the proof for that payment on a testnet, and process further logic.
    - Projects can maintain their own rich-interaction chains, only for their users, shadowing the contracts on the high-value chains that handle more important payments or transactions.
    - For a bridge between something that does not have an EVM-like execution environment and something that has, this could add additional behavior to a purchase/transaction.
    - This bridge may be extended to Bitcoin, so this becomes especially relevant - you could do something on Ethereum dependent upon buying something with Bitcoin.


## Other mechanisms

We can do a mechanism for syncing state changes without these restrictions of using the same sender account. But it does not get easier, we would just need other rules to ensure the protocol cannot be gamed: **a pattern of executing correlated actions**, which we will discuss in the next articles. You might be familiar with this pattern because it is widely used for locking tokens on one chain and minting them on another chain.


We can also **keep the storage state in sync** - entirely or sparsely, depending on what storage keys we need for each contract. Then, we do not replay state changes, but sync snapshots of data at certain points in time. We will talk about the rules of this mechanism in the next articles.
