# How the Global State Database Works

The global state database can is a key-value store that is used to implement a
merkle-radix tree. The values actually stored in the database are a simple
data structure:

    Node:
      Children: Map of address segments to keys for all child nodes
      Value: An optional value store at this node

The keys in the database are all hex-digests of hashes of serialized instances
of this data structure.

In order to make sense of the database, you need a starting key. This is what
we call the state root. Once you have a state root, you can read an "address"
from the tree.

An address is something that is stored in the database, but is not itself
a key or value of the data, nor are any of its segments. (This is something
that confused me for awhile.)

To read the value at an address starting from a given state root, you break the
address into segments (35 in this case) and work your way down from the state
root to the leaf one segment at a time.

Starting with the first segment of the address, you find the segment in the root
node's children and get the key for the next node. You then lookup the next node
with that key and move onto the next segment. And so on.

If you start traversing the tree from different state root, you will potentially
see different nodes and ultimately different values at the desired address.
However, you can also end up at the same node when traversing the tree from
different state roots, if you are accessing a portion of the tree that has not
changed between state roots.

# Implications

1. There are many instances of the node structure for every value written to state.

    Because of how the global state database works, this means the database
    contains many serialized instances of the node data structure.

    As a simple example, consider writing a single value to an empty tree with
    addresses consisting of 35 segments. In addition to writing the value to the
    database, we also read to write 35 serialized instances of the node data
    structure to the database.

    When values are stored in the database with addresses that have common
    prefixes, fewer instances of the node data structure need to be written for
    every value.

    For example, if two values have the same address except for the last
    segment, then the total number of node data structures that need to be
    written is only the number of address segments plus one. If addresses are 35
    segments long, then there would be 36 instances written. On the other hand,
    if two values are written that have different initial address segments, then
    two times the number of address segments instances have to be written. If
    addresses are 35 segments long, this means 70 instances are written.

2. All history for all blocks including forks is stored in the database.

    It is desirable that users be able to read historical values of an address
    or values of an address on different forks. This is possible because
    retrieving values from the database requires starting a specific state root.
    However, this also means that for every every block on all forks known to a
    validator, all historical values and all the nodes needed to traverse the
    tree to that value must be stored in the database.

# Optimizations for Storage Size

In order to reduce the size of the global state database, the following
optimizations can be made in the future:

1. Opportunistically collapse branches of the tree

    Instead of storing a node for every address segment in the database, we
    could collapse branches of the tree for which there is only a single value
    stored.

    Currently, the children field in the node data structure maps a single
    address segment to the database key for a child node. Collapsing branches
    would require changing this map to support referencing child nodes with
    multiple address segments.

    As a simple example, consider the case where two values with different
    initial address segments are stored. Currently, this results in 70 instances
    of the node data structure being written to the database. This is because
    each node can only contain a single address segment. If multiple segments
    could be stored instead, then the root node would have a children field that
    contains the entirety of each address for each value. Two child nodes would
    then be written, each containing the value to be stored. This reduces the
    number of node data structure instances written from 70 to 3 in this
    example.

2. State checkpointing

    Rather than store all historical blocks, state checkpointing could be
    performed periodically after which, all historical values and required nodes
    would be deleted from the database and history would only be readily
    available up to that checkpoint. Historical values beyond the checkpoint
    could still be retrieved, but only by starting from the genesis block or an
    earlier checkpoint and replaying transactions.

3. Purge state roots from old forks

    In addition to state checkpointing, state from blocks that are not on the
    current fork could be purged according to some algorithm, for example time
    or number of blocks behind.

4. Use a more efficient serialization format than CBOR

    The CBOR serialization format is not the most space-efficient serialization
    format. In fact, we use protobuf almost everywhere else in the system for
    this reason. Replacing CBOR with a more efficient serialization format would
    also reduce database size.

    Here are some stats I found comparing serialization schemes:
    https://eng.uber.com/trip-data-squeeze/ The article also recommends using
    zlib for compression, which could may further reduce the stored size.
