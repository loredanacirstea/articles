# Almost All Solidity Contracts Have This Security Issue

![](https://raw.githubusercontent.com/loredanacirstea/articles/6ca0ef92e871e71c040d3f21d644ccadd8e922f2/articles/assets/collisions-m.png)

Almost all Solidity contracts have this security issue, and some have it more gravely than others. This problem is inherent to the Solidity compiler.
Consider the implementation of mapping in Solidity:
```
mapping(uint256 => bytes32) mapping1;
```
The resulting storage address is always in the range `0` -- `2**256-1`. There is no mathematical guarantee of lack of collision. In fact, it is easy to demonstrate the existence of collisions in that range.

## Pseudo-Demonstration

Consider the `keccak` function `k` defined on the range `N1`: natural numbers from `0` to `2**256-1`. Its output in EVM is also in N1. Moreover: the existence of output is guaranteed for any element in `N1` and the output is not trivial (otherwise it would not meet the cryptographic requirements); therefore `k` is not the identity function. Also: the output is as close to being injective as possible. Therefore it is as close to being bijective as possible (but demonstrating bijectivity should be impossible without breaking cryptographic properties). (I would call it pseudo-bijective.)
    
Such pseudo-bijective functions that are almost all the time different than the identity function are guaranteed to have at least one collision per argument value (exception being the cases when the function gives the same output as the identity function). And, for any other cases for which they have no collision, they have to have the same number of cases with more than one collision).
In other words: pseudo-bijective functions that are not permutations have an average of 1 collision per input value with high chances for more than 1. The `k` function has a chance of `1/(2**256-1)` collisions with another input.

## Gravity

Some contracts are less exposed than others. 

For example, arrays have a much smaller chance of collisions, than mappings. In mappings, the input value has a wider range of possible values, often given by or determined by a user. While the array's indexes, used in the `keccak` function usually have smaller range and are less controlled by the user - they are more deterministic.

Those contracts that use mapping and `keccak` have a non-zero exposure. And those are by far the most common.
But even those are not equally exposed.

```
mapping(uint256 => bytes32) mapping1;
mapping(address => bytes32) mapping2;
mapping(uint128 => bytes32) mapping3;
```

The chance of collision raises with:
- user control of the input value
- the number of variables defined
- the range of the input of mappings
- the amount of data that you have in such a mapping (the contract use)


## Mitigation

Most would argue that the chance for a collision to happen is low. And that is true, but since we have solutions to make it 0, we should make the additional effort (if no additional gas is needed).
And all exploits are unimportant and unlikely if it is not your assets that get stolen or your contracts that become compromized. So rather than praying for bad things to not happen to your contracts, we could enact some solutions for this particular issue:

- use salt as part of the storage key to offset the input range `NI` to obtain an output range `NO` (this may lead to other collisions, but the author may decide on the colliding sets)
- make asset management independent of the collision-ridden variables
- use assembly rather than the standard Solidity solution

## The Secure Solution

Use the identity function `i` instead of the `keccak` function `k`. Since the identity guarantees a lack of collisions on any range. It is the trivial permutation: the identity permutation.
In the next article, we will explore the EVM solution to close this security hole.

