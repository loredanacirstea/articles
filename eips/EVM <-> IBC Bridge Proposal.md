# EVM <-> IBC Bridge Proposal

This is a proposed technique of information transfer between IBC and EVM. Both ways.

## The EVM point-of-view

This bridge has to be compatible with the EVM in its current form, therefore we propose the use of precompiled behavior that will assume the form of a normal smart contract.

## Proposed Implementation

1. Choose an address for this set of precompiles: consider `0xff` (`0x00000000000000000000000000000000000000fe`)
2. Choose functions' signatures: consider the first 4 bytes as `0x000000fe` and the next 2 bytes as the function index
3. The rest of the input code is the function input and it is function-dependent.

The bridge call will look like:
```
call(gas, 0x00000000000000000000000000000000000000fe, value, inputPtr, inputSize, outputPtr, outputSize)
```
Its input data: `0x000000fe**xxxxxx..xx`


## The Functions

TDB

## Gas Cost

All obvious gas costs of an external contract call and return, plus the gas pertinent to IBC opperations, plus the computing cost of the interfacing.

TDB
