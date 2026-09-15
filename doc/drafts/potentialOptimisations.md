# Potential optimisations

This document collects potential future optimisations to the Verity network
which we don't currently plan to implement, but could be invoked once the need
arises.

## Evolving full nodes to trackers
On a cube request, full nodes could, instead of returning the Cube itself,
return a list of nodes they previously sent this Cube to.

## Credit scoring
(Full) Nodes could keep a credit score of connected nodes:
Requesting a cube costs a credit, proving further dissemination of that cube
could earn credits.

## Bloom filters
To further improve efficiency light nodes receive bloom filters of connected nodes in order to determine which peers to request cubes from.

## DHT Nodes
Another extension could be DHT nodes. Instead of aiming for full replication, these nodes just replicate a portion of the cube-space within their DHT address range. They have a bit less utility to the network, but also can operate with less resources.

## Merkle Patricia Trie
To improve efficiency during synchronization we can store all hashes in an MPT. This would allow efficient comparison and synchronization with a complexity of O(log(n)). Likely advisable to scale beyond a million cubes per day.

When two nodes synchronize, they compare the root hash of their MPTs. If the hashes are different, this indicates that the nodes have a different set of cubes. The nodes can then recursively compare the hashes of their children, continuing down the tree, until they identify the cubes that differ. This allows the nodes to find the specific cubes that need to be updated, without having to compare every single cube hash.

This is actually implemented but currently disabled.