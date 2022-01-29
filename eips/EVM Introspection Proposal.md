# EVM Introspection Proposal

## The Structure of EVM Data

Block:
```
    timestamp timestamp,
    **number bigint,**
    hash varchar(66),
    parent_hash varchar(66),
    nonce varchar(42),
    sha3_uncles varchar(66),
    logs_bloom text,
    transactions_root varchar(66),
    state_root varchar(66),
    receipts_root varchar(66),
    miner varchar(42),
    difficulty numeric(38),
    total_difficulty numeric(38),
    size bigint,
    extra_data text,
    gas_limit bigint,
    gas_used bigint,
    transaction_count bigint,
    base_fee_per_gas bigint
```
Transaction:
```
   **hash varchar(66),**
    nonce bigint,
    transaction_index bigint,
    from_address varchar(42),
    to_address varchar(42),
    value numeric(38),
    gas bigint,
    gas_price bigint,
    input text,
    receipt_cumulative_gas_used bigint,
    receipt_gas_used bigint,
    receipt_contract_address varchar(42),
    receipt_root varchar(66),
    receipt_status bigint,
    block_timestamp timestamp,
    block_number bigint,
    block_hash varchar(66),
    max_fee_per_gas bigint,
    max_priority_fee_per_gas bigint,
    transaction_type bigint,
    receipt_effective_gas_price bigint
```
Logs:
```
   **log_index bigint,**
    transaction_hash varchar(66),
    transaction_index bigint,
    address varchar(42),
    data text,
    topic0 varchar(66),
    topic1 varchar(66),
    topic2 varchar(66),
    topic3 varchar(66),
    block_timestamp timestamp,
    block_number bigint,
    block_hash varchar(66)
```
Contract:
```
    **address varchar(42),**
    bytecode text,
```

## Proposed Introspection

1. Choose an address for this set of precompiles: consider `0xff` (`0x00000000000000000000000000000000000000ff`)
2. Choose functions' signatures: consider the first 4 bytes as `0x000000ff` and the next 2 bytes as the function index
3. The rest of the input code is the function input and it is function-dependent.

The introspection call will look like:
```
call(gas, 0x00000000000000000000000000000000000000ff, value, inputPtr, inputSize, outputPtr, outputSize)
```
Its input data: `0x000000ff**xxxxxx..xx`

### The Functions

#### Block

- getBlock(number)
- getBlockByHash(hash)
- getFromBlock(number, field)
- getFromBlockByHash(hash, field)
- getTxsFromBlock(number)
- getTxsFromBlockByHash(hash)

#### Transaction

- getTransaction(hash)
- getLogsFromTransaction(hash)
- getFromTransaction(hash, field) (usable to get input data for rollups)

#### Log

- getLog(log_index)
- getLogByTx(tx_hash)
- getLogByAddress(address)
- getLogByTopic0(topic)
- getLogByTopic1(topic)
- getLogByTopic2(topic)
- getLogByTopic3(topic)
- getFromLog(log_index, field)

Total: 17 functions

### Cache

If instead of the normal contract at `0xff`, we design a contract with cache at `0x01ff`:
input data:  `0x000001ff**xxxxxx..xx`, then the input may be hashed with keccak and a stored value presented as output in case of a match.
In case of no match: the computation is performed normally and the output is stored at the address keccak256(input) before being returned. (As any other memoization solution)

### Gas Cost

Since nodes should have the freedom to keep only partial state, the digging into the deep past should be more costly. We propose the grading by height:

1. Blocks from present (height h) to h-255 (about 1 hour)
2. Blocks from h-255 to h-6500 (about 1 day)
3. Blocks from h-6500 to h-2400000 (about 1 year)
4. Blocks from h-2400000 to 0 (all the rest)

In the case of the cache: the differentiation does not apply when a match is found.

To this differentiated cost we add the cost of the external contract call, the computing cost of interfacing with the LevelDB persistence. TBD.

