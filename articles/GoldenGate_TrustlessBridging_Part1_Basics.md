---
tags: bridging
---

# Golden Gate - Trustless-Bridging Ethereum (EVM) Blockchains - Part 1: Basics


With the emerging proposals for integrating the EVM in Ethereum 2.0 (https://ethresear.ch/t/executable-beacon-chain/8271), this EVM to EVM trustless two-directional bridge series comes at just the right moment. It can bridge Ethereum 1 to Ethereum 2. And the other way around.

## A Symmetrical Light Client

A [Light Client](https://ethereum.org/en/developers/docs/nodes-and-clients/#light-node) is a tool for validating chain data while storing the minimum amount of information.

The purpose of an on-chain light client is simple: storing the block hashes. These are enough to prove that "something" has happened on a chain of interest. We can prove transactions, receipts, balances, contract code, and even storage slots.

We will show in later articles a demonstration of a minimal (yet effective), two-directional (symmetrical) client and a proposal of decentralized bridge economics.

## A Symmetrical Prover

We are now focusing on what can we prove on-chain (**chain A**) about another **chain B**, given a smart contract on chain A, which stores trustworthy block hashes from chain B.

This process is symmetric for proving chain A data on chain B.

The core mechanism is:
* **get the Merkle proof data from a chain B full node (archive node if a storage proof is needed)**
* **send the proof data as `calldata` to the Prover contract on chain A, along with the header corresponding to the chain B block containing the state change we want to verify**
* **the Prover contract computes the block hash from the header data and checks whether it is a valid hash, by querying the Light Client smart contract on chain A, which keeps track of chain B block hashes**
* **proof data is verified against the bytes32 Patricia-Merkle trie root found in the block header**

For an in-depth understanding of how the Patricia-Merkle trie works, check out [the official docs](https://eth.wiki/en/fundamentals/patricia-tree), and [this awesome article](https://easythereentropy.wordpress.com/2014/06/04/understanding-the-ethereum-trie/).

Compute proofs in JavaScript with https://github.com/zmitton/eth-proof or directly with your web3 library of choice - see the [getProof method from web3.js](https://web3js.readthedocs.io/en/v1.3.0/web3-eth.html?highlight=proof#getproof).

**The code examples below can be found at https://github.com/loredanacirstea/goldengate. This library will improve along with the series.**

**The bridge specification is worked on at https://github.com/loredanacirstea/blockchain-bridging-api. PRs, suggestions, issues are welcome.**


Now let's see exactly what the header data contains:

```solidity
struct BlockHeader {
    bytes32 parentHash;
    bytes32 sha3Uncles;
    address miner;
    bytes32 stateRoot;
    bytes32 transactionsRoot;
    bytes32 receiptsRoot;
    bytes logsBloom;
    uint256 difficulty;
    uint256 number;
    uint256 gasLimit;
    uint256 gasUsed;
    uint256 timestamp;
    bytes extraData;
    bytes32 mixHash;
    uint64 nonce;
    uint256 totalDifficulty;
}
```

The above names correspond exactly to what the `web3.eth.getBlock` method from https://github.com/ethereum/web3.js returns. Other libraries might have different names - e.g. `ommersHash` instead of `sha3Uncles`, `beneficiary` or `coinbase` instead of `miner`.

The block hash is `keccak256` of the [RLP-encoded](https://eth.wiki/fundamentals/rlp) header data.

Also, notice the `stateRoot`, `transactionsRoot`, `receiptsRoot`. These are the tree roots of interest for today.


**The first step is to prove that the header data received by the Prover contract can be RLP serialized and hashed into the expected block hash.**

**The next step is to prove that this block hash is indeed part of the chain of interest** - **chain B** (this is where the Light Client comes in).

```solidity
function verifyHeader(
    EthereumDecoder.BlockHeader memory header
)
    view public returns (bool valid, string memory reason)
{
    bytes32 blockHash = keccak256(getBlockRlpData(header));
    if (blockHash != header.hash) return (false, "Header data or hash invalid");

    // Check block hash was registered in light client
    bytes32 blockHashClient = client.getConfirmedBlockHash(header.number);
    if (blockHashClient != header.hash) return (false, "Unregistered block hash");

    return (true, "");
}
```
*https://github.com/loredanacirstea/goldengate/blob/495abc40596e7b4cad519131d16874fbc844bd79/contracts/contracts/Prover.sol#L24-L37*


Next, the same algorithm will be used for checking Merkle proof data against the trie root. **Check out *https://github.com/loredanacirstea/goldengate/blob/495abc40596e7b4cad519131d16874fbc844bd79/contracts/contracts/lib/MPT.sol.***

If we want to prove that a transaction receipt is part of the chain, our proof must contain the entire path of Merkle tree nodes from our transaction to the root node.

A simple proof example:

```javascript
{
    receiptIndex: 0,
    headerData: '0xf90211a0460c84f2877fd351416cc9207bbb140eda2a59f0501bda5d4e814e017024536fa01dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347948595dd9e0438640b5e1254f9df579ac12a86865fa0c2377ee7587692810e987464064b2247c8d7d6ad0f9fce327bd27ad0c92a62afa0a4041a7e77fc216726e5015e6712d6434a1a4991781ed9478f5451ad5ac1e362a077ee012c7ec8a7a5cc6f8c7899e4176f8716ce11c637820d84a09e2b548191d3b90100000000000000001000000000000000000000000000100000000000000000000000000040000000000000000000400100000000000000000000000000800000000000000000000000000020080000040000000000000000000000000000100000000000000000400000080000000000000000000000000000000000100000000000000000000000000000000000004000000000000000000000020000001000000000000000000000000000800000000000000000000000000000000004000000000000020000280000000000000800000500000100000000400000200000000000800000000000000000000000000000020000000000000000000000040000009032323137373831393836303031333938839a4aa383979f3e83024c39845ec6f2f487657a696c2e6d65a0f0a64ca3c939e2506d05505449b4caaa06eba7c86062997e6632b0ee3db977df88e284f1c803fa6f5a',
    receiptData: '0xf901a60182574ab9010000000000000000100000000000000000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000080000000000000000000000000002008000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000000000000000000000002000000000000000000000000010000000000000000000020000000000080000000000000000000000000000000000000000000000000000004000000f89df89b94d26114cd6ee289accf82350c8d8487fedb8a0c07f863a0ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3efa00000000000000000000000002c7116a63ab91084a7a5d6fef2e4eda0c84487afa00000000000000000000000007d3cd5685188c6aa498697db91ca548a1249863ea0000000000000000000000000000000000000000000000001158e460913d00000',
    logEntry: '0xf89b94d26114cd6ee289accf82350c8d8487fedb8a0c07f863a0ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3efa00000000000000000000000002c7116a63ab91084a7a5d6fef2e4eda0c84487afa00000000000000000000000007d3cd5685188c6aa498697db91ca548a1249863ea0000000000000000000000000000000000000000000000001158e460913d00000',
    proof: [
        [
            "0x6adc4881ae9f2b2bbbf70a60e5b05f0734c02d731e80ac1503231d851b24ffe6",
            "0x",
            "0x",
            "0x",
            "0x",
            "0x",
            "0x",
            "0x",
            "0x103165b38cd8ad3ffa4b1de70e7391ac2c321ffe265bc77f2316ba33288c3717",
            "0x",
            "0x",
            "0x",
            "0x",
            "0x",
            "0x",
            "0x",
            "0x",
        ],
        [
            "0x30",
            "0xf901a60182574ab9010000000000000000100000000000000000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000080000000000000000000000000002008000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000000000000000000000002000000000000000000000000010000000000000000000020000000000080000000000000000000000000000000000000000000000000000004000000f89df89b94d26114cd6ee289accf82350c8d8487fedb8a0c07f863a0ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3efa00000000000000000000000002c7116a63ab91084a7a5d6fef2e4eda0c84487afa00000000000000000000000007d3cd5685188c6aa498697db91ca548a1249863ea0000000000000000000000000000000000000000000000001158e460913d00000",
        ]
    ]
}
```

The path from the root node of the receipts tree to the receipt of interest is given by the receipt index. In the above case, the index is 0, so the verification algorithm with take the first of the 17 values of the branch node as the new expected root value and check it against the last odd-length leaf node with a key of `0x30` and a value representing the RLP-encoded receipt data. For other examples, [check this out](https://github.com/loredanacirstea/statebridge/blob/495abc40596e7b4cad519131d16874fbc844bd79/contracts/test/data.js).

The RLP-encoded receipt data has this form:
```solidity
struct TransactionReceipt {
    uint8 status;
    uint256 gasUsed;
    bytes logsBloom;
    Log[] logs;
}

struct Log {
    address contractAddress;
    bytes32[] topics;
    bytes data;
}
```

The same type of proof is used to prove that a transaction is included in a block - the Patricia tree path is the transaction index inside the block. Or that an account is part of the state trie - the path is `keccak256(abi.encode(account_address))`. And if we want to prove a storage slot is part of the storage trie (inside the account trie), the path is the storage key.

### Proving logs happened

What are the steps for proving a log happened on another chain?
* **get the receipt proof for the receipt containing the log and the header for the block in which it was mined.**
* **check the header as shown above**
* **check the [receipt proof](https://github.com/loredanacirstea/statebridge/blob/495abc40596e7b4cad519131d16874fbc844bd79/contracts/contracts/Prover.sol#L53-L65)**
```solidity
function verifyReceipt(
    EthereumDecoder.BlockHeader memory header,
    MPT.MerkleProof memory receiptdata
)
    pure public override returns (bool valid, string memory reason)
{
    if (header.receiptsRoot != receiptdata.expectedRoot) return (false, "verifyReceipt - different trie roots");

    valid = receiptdata.verifyTrieProof();
    if (!valid) return (false, "verifyReceipt - invalid proof");

    return (true, "");
}
```
* **check that the log [is part of the receipt](https://github.com/loredanacirstea/statebridge/blob/495abc40596e7b4cad519131d16874fbc844bd79/contracts/contracts/Prover.sol#L81-L94)**
```solidity
function verifyLog(
    MPT.MerkleProof memory receiptdata,
    bytes memory logdata,
    uint256 logIndex
)
    pure public override returns (bool valid, string memory reason)
{
    EthereumDecoder.TransactionReceiptTrie memory receipt = EthereumDecoder.toReceipt(receiptdata.expectedValue);

    if (keccak256(logdata) == keccak256(EthereumDecoder.getLog(receipt.logs[logIndex]))) {
        return (true, "");
    }
    return (false, "Log not found");
}
```

#### Log-based proving systems

If you trust the logs and the contract that logged them, this is the most efficient mechanism to prove "something" happened on another chain.

The most frequent use case for log-based systems has been transferring tokens between chains through a pair of Token smart contracts that can interface with a Prover contract.

##### Contract-first approach

Where the user interacts with the contract of interest first (e.g. token contract), which calls the Prover contract in order to confirm the log.

```
user -> TokenChainA: lockTokens()
TokenChainA -> TokenChainA: TokenLocked event

user -> TokenChainB: header, receipt proof, expected log
TokenChainB -> ProverChainB: verifyHeaderReceiptAndLog()
ProverChainB -> LightClient: getConfirmedBlockHash()
LightClient -> ProverChainB: hashIsValid (bool)
ProverChainB -> TokenChainB: proofIsValid (bool)
TokenChainB -> TokenChainB: mintTokens()
TokenChainB -> TokenChainB: TokenMinted event
```

##### Proxy-first approach

We can have a general proxy that gets called first, verifies the given proof and forwards the call to the token contract if the proof is valid.

The proxy contract should be able to figure out what contract it needs to call from the proof data - it can be encoded in the log data. E.g. `TokenChainA` can emit a log with a `bytes32` hash of the calldata that will be used on chain B.

```
user -> TokenChainA: lockTokens()
TokenChainA -> TokenChainA: TokenLocked event

user -> ProverChainB: header, receipt proof, expected log
ProverChainB -> ProverChainB: verifyHeader()
ProverChainB -> LightClient: getConfirmedBlockHash()
LightClient -> ProverChainB: hashIsValid (bool)
ProverChainB -> ProverChainB: verifyReceipt()
ProverChainB -> ProverChainB: verifyLog()
ProverChainB -> TokenChainB: mintTokens()
TokenChainB -> TokenChainB: TokenMinted event
```

This is a simplified flow - in reality, we need to also take some precautions against reusing the same proof data multiple times.
One solution for the log-based prover is using **idempotent execution** (e.g. a log can contain values for the final state instead of the state delta, therefore submitting it multiple times results in the exact same final state), along with **limiting the time period in which the proof can be submitted**, to avoid reusing old logs, or a **monotonically increasing value** that makes the proof submissions unique. Or a combination of these kinds of techniques.


### Proving transaction existence and outcome

For a more general system, we may need to prove that a transaction was successful on a chain and maybe check the transaction fields themselves.

The RLP-encoded transaction data contains:

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

The transaction hash is a hash of the above data.

What are the steps for proving a transaction happened on another chain?
* **get the transaction proof and the header for the block in which it was mined.**
* **check the block header as shown in the first part**
* **check the [transaction proof](https://github.com/loredanacirstea/statebridge/blob/495abc40596e7b4cad519131d16874fbc844bd79/contracts/contracts/Prover.sol#L39-L51)**
```solidity
function verifyTransaction(
    EthereumDecoder.BlockHeader memory header,
    MPT.MerkleProof memory txdata
)
    pure public override returns (bool valid, string memory reason)
{
    if (header.transactionsRoot != txdata.expectedRoot) return (false, "verifyTransaction - different trie roots");

    valid = txdata.verifyTrieProof();
    if (!valid) return (false, "verifyTransaction - invalid proof");

    return (true, "");
}
```

After we prove that the transaction data is indeed part of the transaction trie, we check the transaction's **status** from the corresponding receipt and see if it succeeded or failed.
* **get receipt proof for the receipt with the same index as the transaction we want to prove**
* **verify receipt against the header's `receiptsRoot`**
* **RLP decode receipt data and check the value of `receipt.status`**


We might need to prove that the transaction was sent from a certain address. We can do this with `ecverify`, by determining the address from the `r`, `s`, `v` signature values in the transaction data and the hash data - the message that was signed.

### Proving state after a block is processed

We can go even further, to prove chain state at a certain blockchain height. For proving storage state, we need to get the proofs from an Ethereum node synced on archive mode.

The block header contains the `stateRoot` - the root node for the accounts tree.

An account node contains the following data:
```solidity
struct Account {
    uint256 nonce;
    uint256 balance;
    bytes32 storageRoot;
    bytes32 codeHash;
}
```

This allows proving how many transactions did an EOA (externally owned account) perform or how many contract creations a contract account performed (`nonce`). It allows for proving a balance that an account had after a certain block was mined. Or that a contract had a certain code deployed at a certain block.

The `storageRoot` enables us to prove even more - that a contract's storage slot contained value `x` at block `y`.

What are the steps for proving on `chain A` that a contract from chain `B` had a storage value of `x` at block `y`?
* **get the account proof and the header for the block of interest.**
* **check the block header validity as shown in the first part**
* **check the [account proof](https://github.com/loredanacirstea/statebridge/blob/495abc40596e7b4cad519131d16874fbc844bd79/contracts/contracts/Prover.sol#L67-L79)**
```solidity
function verifyAccount(
    EthereumDecoder.BlockHeader memory header,
    MPT.MerkleProof memory accountdata
)
    pure public override returns (bool valid, string memory reason)
{
    if (header.stateRoot != accountdata.expectedRoot) return (false, "verifyAccount - different trie roots");

    valid = accountdata.verifyTrieProof();
    if (!valid) return (false, "verifyAccount - invalid proof");

    return (true, "");
}
```
* **decode the RLP-encoded account data and extract the `storageRoot`**
* **check the storage proof against the `storageRoot`**
```solidity
function verifyStorage(
    MPT.MerkleProof memory accountProof,
    MPT.MerkleProof memory storageProof
) pure public override returns (bool valid, string memory reason)
{
    EthereumDecoder.Account memory account = EthereumDecoder.toAccount(accountProof.expectedValue);

    if (account.storageRoot != storageProof.expectedRoot) return (false, "verifyStorage - different trie roots");

    valid = storageProof.verifyTrieProof();
    if (!valid) return (false, "verifyStorage - invalid proof");

    return (true, "");
}
```
* **the storage proof will contain the storage key and value on the last leaf node.**

You can find an example of such proof [here](https://github.com/loredanacirstea/statebridge/blob/495abc40596e7b4cad519131d16874fbc844bd79/contracts/test/data.js#L269-L315).


### Basics done. Now what?

These are the tools for our toolbelt. The next step is to figure out what interesting things can be done with them.

**In the next article, I will get into details about an efficient way of keeping data in sync between the same type of contracts deployed on two or more EVM compatible chains, a demo that you can see here: https://youtu.be/WD_2tuX9jeg.**


### This is not all!

The series of articles on bridging will continue with various new (and maybe surprising) features and the finale will be the release of the full product on testnets. Maybe more.


