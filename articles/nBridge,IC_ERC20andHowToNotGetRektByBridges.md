---
tags: evmos
---


# nBridge, Inter-Chain ERC20 and How To Not Get Rekt By Bridges (unique assets & shared-security) [ELI5]

![](https://raw.githubusercontent.com/loredanacirstea/articles/master/articles/assets/bridge.png)

I am writing this while still in Seoul, after HackATOM, having presented as an independent volunteer researcher, the first ever multi-chain EVM transactions for Cosmos-EVM chains (yes, it can be extended to CosmWASM too), using Ethermint/Evmos. We call this technology nBridge.

https://youtu.be/j5CVliOb-Ck

This morning, the Nomad bridge just got hacked and drained and users have started to panic sell, thinking that their madUSDC or madWETH will now be worthless.

https://twitter.com/nomadxyz_/status/1554246853348036608
https://twitter.com/EvmosOrg/status/1554275898156670976
https://twitter.com/samczsun/status/1554252024723546112?s=21&t=2MhnaDExgg5oMP6Qys01Dg

Having presented the Inter-Chain Singular Object pattern as a hacker submission using nBridge (for which I won 3rd prize), where you can have the same asset replicated on multiple chains:

https://youtu.be/Khqv-1TbsMo

I realize that our nBridge technology can help users to not get rekt by choosing and trusting one bridge.

First, let's see where the problem is now:

## 1. Today: fragmented assets, fragmented security, fight for trust and monopoly

This is how bridging looks like now:

![](https://raw.githubusercontent.com/loredanacirstea/articles/master/articles/assets/bridge_fragmentation.png)

Each bridge exposes a different wrapped version of the native coin they bridge to another chain. `Br1_WrappedCoin1` is used by Br1 (a bridge) and `Br2_WrappedCoin1` is used by Br2 (another bridge), to represent `Coin1` on Chain2.
 
Now, you have multiple wrapped assets representing `Coin1` (Chain1's native token): `Br1_WrappedCoin1` and `Br2_WrappedCoin1`. Which one represents `Coin1`? Are they the same? What asset should you trust?

Right now, these assets are not the same and neither one fully represents `Coin1`. They represent the promise (IOU) of `Coin1`, on Chain2. The value of `Br1_WrappedCoin1` is tightly linked to what Bridge1 can offer as security and liquidity. If Bridge1 does not have enough liquidity to allow you to exchange your `Br1_WrappedCoin1` back to `Coin1` on Chain1, it is worthless. And `Br1_WrappedCoin1` cannot be swapped for `Br2_WrappedCoin1`.

Now, you have an ecosystem of bridges that produce a fragmentation of assets with lower security and liquidity than the native coin they represent.

Of course, if one bridge gets hacked, this does not affect the other bridges. But it affects the entire ecosystem, by users losing trust. If a popular bridge gets hacked, who else they can trust?


Let's take the simple example of bridging ETH, from Ethereum to another Evmos. What the bridge does is create a wrapped ERC20 asset representing ETH, on Evmos.

![](https://raw.githubusercontent.com/loredanacirstea/articles/master/articles/assets/bridge_fragmentation_ETH.png)

As in the general example above, we are left with `madWETH`, `multiWETH`, `ceWETH`, which are not interchangeable directly.

## 2. Evmos and the ERC20 Cosmos SDK module

Evmos has already taken steps to avoid the fragmentation of assets that we talked about above, by introducing the [ERC20 module](https://docs.evmos.org/modules/erc20/). This module makes a unique representation of native coins from other Cosmos SDK modules, in Evmos. Then, this unique representation can be used by anyone (DEXes, bridges, etc.). And it exposes Evmos-deployed ERC20 tokens to other Cosmos chains, through IBC.

But the ERC20 module is limited to the ERC20 standard and a Cosmos SDK environment for the collaborating chains. And the ERC20 standard was designed for single-chain opperations.

When you support a virtual environment like EVM and CosmWASM, flexibility is the name of the game. You want to allow devs to create new standards without needing to change the underlying chain protocol in order to support them. Of course, flexibility comes with a security risk, but allows for rapid and decentralized innovation that can later be integrated even in the base protocol. 


## 3. nBridge and Inter-Chain Contracts

What we call nBridge is the solution for allowing smart contracts to take advantage of the InterBlockchain Communication protocol (IBC). So, you can send cross-chain, dependent or multi-chain transactions from a smart contract (EVM or CosmWasm).

nBridge is a layer1 bridge, embedded in the protocol, available as a Cosmos SDK module. It is a Cosmos SDK module that any Cosmos chain can use, with a focus on Cosmos chains with a virtual execution environment.

As I showed in my hackathon submission, you can have multi-chain deployed smart contracts that represent the same thing. And multi-chain synced/replayed data (especially useful for immutable data in the current implementation). I demonstrated this for the user's identity (nickname).

And we modified the ERC20 standard to become multi-chain. We call it Inter-Chain ERC20 standard: IC_ERC20. An early example of such a standard was shown: https://github.com/the-laurel/demos/blob/main/IcERC20/IcERC20.sol, but a truly multi-chain demonstration will follow in future demos.

You can have the same smart contract (e.g. ERC20, NFT, other standard) represented uniquely across Cosmos chains (with or without execution engine). Without any fragmentation. If the chain has an execution engine, the flexibility of what types of standards can become cross-chain without further protocol changes, becomes maximum.

**This solves fragmentation across any Cosmos chains and allows protocols to fully control their tokens on all chains.**

To further solve for non-Cosmos chains, like in the case of Ethereum and Nomad, I propose the following:


## 3'. nBridge, Inter-Chain Standards and the BridgeGuild

Cosmos itself is proposing a shared security model across chains. And I am proposing the same across bridges.

**For the sake of protecting our users for getting rekt or panic selling, or getting panic-scammed, I urge bridges to think about a shared security model with shared responsibility.**

![](https://raw.githubusercontent.com/loredanacirstea/articles/master/articles/assets/bridge_guild_wrapper.png)

I propose a **Bridge Guild** where:
- a `BridgeGuildWrapper` smart contract converts bridge-specific wrapped assets (e.g. `madWETH`, `multiWETH`, `ceWETH` ...) to a unique instance, inter-chain compatible (e.g. `IC_WETH`).
- each bridge that joins must pass a 3rd party audit and cross-audits from other bridge teams. Only in a guild do you have the right incentive to do such an audit with care. And who better to review your code if not other bridge specialists?
- bridges declare how much they are open to guarantee like any financial institution does to pass as a financial institution; a well-thought slashing mechanism
- or, a way to pay back the other bridges in time, with additional interest if your bridge gets hacked

Then, as a user, instead of holding `madWETH`, you can swap for `IC_WETH` on a 1:1 ratio and you `IC_WETH` is backed by the entire **Bridge Guild**'s liquidity.

As a user, your assets as safer.
As a bridge, you get some leeway in how you recover your lost funds, over time.
As a guild, bridge teams can join legal efforts against a hacker, when needed.

With inter-chain standards (IC_ERC20 & others), you then have a protocol-secured way to move tokens to other cosmos chains.

Shared-security can also mean shared risk when things go wrong. To limit the risk, we can create multiple gateways where an attack can be stopped:
- bridge-specific contract can be halted by the bridge team (as it happens now)
- inter-chain swap smart contract and inter-chain Cosmos SDK modules can be halted by chain governance, in an urgent proposal



**The future is inter-chain, with shared responsiblity, with collaboration and constructive criticism rather than monopolies. Users should no longer fear getting rekt.**


