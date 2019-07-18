# Flexible Alias or Why ENS is Obsolete


tl;dr We are proposing Alias - a homogeneous standard for identifying resources by human-readable qualifiers: [EIP-2193 dType Alias Extension - Decentralized Type System](https://github.com/ethereum/EIPs/pull/2193), that supersedes ENS.

## Semantic Analysis

This article is meant to analyze current standards used for identifying resources, through a semantic and intuitive lens.


### Root->Leaf

From a semantic standpoint, it is intuitive to traverse a path to a leaf/resource starting from its root origin: `root > component > subcomponent > leaf`. There are multiple separators used to intuitively define such a path: `.`, `>`, `/`, `#`, `:`, `\`.

For example, in object access, in most object oriented languages, the `.` means refining. It provides access to a property or method of the parent object, respecting the root->leaf order or writing the path to the property.

The same root->leaf order is present in operating system file browsers, menus and sub-menus in applications, breadcrumb UI components. Historically, even when you reference a passage in the Bible, you first mention the Old/New Testament, then the chapter and then the verse number.

### Leaf->Root

Characters that symbolize the leaf->root path are: `@`, `<`, `/`. `@` means "at" and is widely used in email addresses.`/` is used to represent mathematical fractions `2/3`.


### Leaf->Leaf

Characters that symbolize the leaf->leaf relations in natural and programming languages are: `-`, `,`, `&`, `|`, `=`, `/`, `+`.

### Ambiguities

As seen above, there are symbols with ambiguous meaning, depending on context. An example is `/`, which can convey a root->leaf path (e.g. `/Users/alice/Documents/Books`) or a leaf->root rule (e.g. `2/3` fraction), or a leaf->leaf relation (e.g. in natural language: "he/she").


## Existing Standards

We will now take a look at the existing standards used for providing human-readable addressability for resources and how they use the semantic rules mentioned above.

## DNS

The current DNS format is `leafsubdomain.subdomain.domain.tld`. Users can set their own subdomains and domains, buying them from a decentralized network of DNS providers and registrars. They can choose the TLD only from the available options, controlled by [ICANN](https://www.icann.org), in associations with world governments.

The hierarchical order of the standard is: `leafsubdomain < subdomain < domain < tld`. We notice that this order makes the current URL standard heterogeneous and not intuitive: `protocol://leafsubdomain.subdomain.domain.tld?query#hash`, where `protocol > leafsubdomain`, but `leafsubdomain < subdomain < domain < tld`, and `tld > query > hash`. Therefore, the order changes from meaning `>`, to meaning `<` and back.

We also notice that the DNS leaf->root format replaces the IP address (e.g. IPv4: `192.0.2.235`, IPv6: `FE80::0202:B3FF:FE1E:8329`), which has a root->leaf ordering.

This leads to confusion and unpredictability, making it harder for the internet to be machine-readable and intuitive.


## DNS for Ethereum/Blockchain

Some of the existing solutions for providing DNS-like functionality for blockchain-based systems are [ENS](https://ens.domains), [Handshake](https://handshake.org/), [Unstoppable Domains](https://unstoppabledomains.com).

These follow the same principles as DNS, while striving to be more decentralized. However, without a central authority such as ICANN, they still need to set a collaboration process, in order to not assign the same domain to different users.

Here, the character `.` is used to mean a leaf->root rule, which is unintuitive: `leafsubdomain.subdomain.domain.tld`, where `leafsubdomain < subdomain < domain < tld`.

For ENS, the hostname can now resolve to an Ethereum address, [Swarm](https://swarm.ethereum.org/) or [IPFS](https://ipfs.io) content, or to IP addresses.

## Libra/Move Comparison

Libra has two ways of addressing resources. In Move, developers use `address.modulename.resource` to reference a resource - e.g. `0x56.Currency.TCoin` or even `0x56.Currency.deposit()` for resource methods.

However, resources are stored under user accounts as `<useraddress>/resources/<moduleaddress>.<modulename>.<resourcename>`, e.g. `0x12/resources/0x56.Currency.TCoin`.

The `.` separator respects the root->leaf hierarchical discovery path: from the more general, to the more particular, in the direction of writing. A Libra account can contain multiple modules, with multiple resources.

The `/` separator also respects the root->leaf rule, as an account can have multiple resources.

## Alias is Based on Intuition

Alias is based on semantic ordering, following the meaning of the separators that are used. It currently allows the following separators:
 - `.`: general domain separation, e.g. `domain.subdomain.leafsubdomain` (root->leaf)
 - `@`: identifying actor-related data, such as user profiles, e.g. `alice@domain.subdomain` (leaf->root)
 - `#`: identifying concepts, e.g. `somain.subdomain.topicY#postX` (root->leaf)
 - `/`: general resource path definition, e.g. `resourceRoot/resource` (root->leaf)

dType, through [EIP-1900](http://eips.ethereum.org/EIPS/eip-1900) and [EIP-2157](http://eips.ethereum.org/EIPS/eip-2157), enables Alias to resolve any type of data - from Ethereum addresses, Swarm & IPFS pointers, to any user defined data located inside a smart contract.

For more information, check [EIP-2193 dType Alias Extension - Decentralized Type System](https://github.com/ethereum/EIPs/pull/2193).

dType is akin to Move resources, because they are similar to dType `structs`. With Alias, they will gain human-readable accessibility, akin to DNS. However, DNS/ENS do not have data modeling and are less flexible and Move resources are (and probably will be) controlled by a centralized entity.

In the new decentralized world, maybe it's time to break with tradition and inflexibility and retie the knot with intuition and clear semantics.
