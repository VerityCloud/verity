![](../img/vera_300px_nobg.png)

# Verity
Verity is a decentralised and censorship-resistant data storage and distribution
platform on top of which fully georedundant, peer-to-peer-enabled applications
can be built. It leverages unique cube structures, each containing 1kB of data,
which are then synchronized across participating nodes. Different Cube types
offer predefined cryptographic schemes to ensure data integrity and authenticity.
Although light nodes are supported, nodes are encouraged to be operated as full
nodes, replicating all cubes.

As a demo application, this project also creates a decentralized and
censorship-resistant microblogging app application using the Verity platform,
akin to Twitter, Threads or Reddit.

# Documentation
* Application developers should follow the [Client Application Guide](clientApplicationGuide.md)

# Specifications
* [Cubes](cube.md) are the basic building blocks of Verity
* The [Common Cube Interface](cci.md) simplifies and streamlines how applications
  store and retrieve data to and from Cubes

# Reference implementation
This repository contains both the specifications and the reference implementation
of Verity. The reference client is written in TypeScript and allows developers to
easily build Progressive Web Applications (PWA) on top of Verity.

# Name and logo
Verity is another word for truth.
The dove Vera is the logo of Verity.
The dove not only signifies peace, harmony, and purity, but also symbolizes the
free spirit of a decentralized network that cannot be caged or controlled, much like the dove itself.

# Cube Store
  The cube store is part of each node and contains all the cubes known. While a full node is online it tries to keep its Node Store synchronized with the rest of the network by regularly synchronizing with all connected peers.

## Spam Prevention
  Verity has two primary defenses against Denial of Service (DoS):
### Hashcash
  The node's cube store sets a challenge level of trailing zeroes for each cube, any cubes that fail this challenge get rejected and dropped. The level initially will be hardcoded, but later may be adjusted to scale with the amount of data in the Cube Store. This is a well proven defense against spam and sufficiently effective.
### Ephemeral Cubes
  Cubes have a limited lifetime. By default they will only live for 7 days until dropped from the network. Though once switching to Merkle Patricia Trie (MPT) timestamps can be easily sorted into different subtrees, allowing flexible control over cube retention times. Nodes may regularly re-inject their posts to keep them available online, if desired.

  > Also see [Cube Lifetime Function](cube.md#cube-lifetime-function)

# Node types
- `Full Nodes` replicate all Cubes on the entire network.
  They form the backbone of the Verity network and users are encouraged to
  operate as full nodes whenever possible.
- `Light Nodes` only request cubes on demand.
  Verity applications are usually expected to operate as light nodes as users
  cannot be expected to have the bandwidth and storage to operate a full node,
  especially on mobile devices.


# Network Communication
See the current [wire format 1.0 specification](wireFormat1.md).


# Encryption
## Cryptographic primitives used
   - All hashes are SHA3, may be truncated where sensible.
   - All signatures are ED25519
   - Fingerprints of either are truncated to 64 bit

  > Note: Signatures are very computationally expensive. Signing a cube with ED25519 is roughly 20 times as expensive as hashing with SHA3, verifying the signature even 60 times. Thus whenever signatures are to be verified, the cube challenge has to be verified first, in order to avoid creating a vector for Denial of Service (DoS).

## Cryptography in the core layer
The core layer uses cryptographic hashing to ensure Cube integrity and derive
the keys for frozen Cube and PICs. It also uses signatures to ensure authenticity
and derive the keys for MUCs and PMUCs.

It does not handle encryption; this is deferred to the CCI layer.

## Cryptography in CCI
See [CCI Encryption](cciEncryption.md).

Applications are encouraged to always use CCI encryption when possible.
If in doubt, encrypt.


> Note: this project does not implement a cryptocurrency, nor does it resemble a blockchain.
