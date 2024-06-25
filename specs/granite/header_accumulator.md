# Granite - Header Accumulator

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Overview](#overview)
- [Timestamp Activation](#timestamp-activation)
- [Rationale](#rationale)
- [Constants & Definitions](#constants--definitions)
- [Accumulator Tree Construction](#accumulator-tree-construction)
  - [Interior Commitment Truncation Rules](#interior-commitment-truncation-rules)
- [Block Execution Changes](#block-execution-changes)
  - [Block Validity Changes](#block-validity-changes)
  - [Header Changes](#header-changes)
    - [`extraData` format](#extradata-format)
    - [Header RLP Size Considerations](#header-rlp-size-considerations)
  - [Accumulator Tree Functions](#accumulator-tree-functions)
    - [Zero Hashes](#zero-hashes)
    - [Tree Root](#tree-root)
    - [Insertion](#insertion)
    - [Verification](#verification)
- [Eventual Migration to EIP-7685](#eventual-migration-to-eip-7685)
- [Forwards Compatibility Considerations](#forwards-compatibility-considerations)
- [Client Implementation Considerations](#client-implementation-considerations)
- [Infrastructure Considerations](#infrastructure-considerations)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

The Granite hardfork introduces a new phase of block execution, header hash accumulation. After fork activation,
every `HEADER_BATCH_SIZE` blocks, the execution layer must append a binary merkle tree of depth
`HEADER_BATCH_TREE_DEPTH` containing the previous `HEADER_BATCH_SIZE` block hashes (exclusive of the current block) to
the header accumulation merkle tree of depth `ACCUMULATOR_TREE_DEPTH` in the previous header accumulation block.

## Timestamp Activation

Granite, like other OP Stack network upgrades, is activated at a timestamp. Changes to the L2 Block execution rules
outlined in this specification are applied when the `L2 Timestamp >= activation time`.

## Rationale

After the activation of the [interop hardfork](../interop/overview.md), the computational complexity of proving the
OP Stack's state transition will increase. The current plan for proving the inclusion of remote logs within chains
that exist in the active dependency set is to include a final "consolidation step" to resolve the dependencies of
each chain, which currently could require deriving and executing many L2 blocks in the context of the verifiable
environment that the proving system is executing within.

This feature seeks to reduce the complexity of the consolidation step significantly, through allowing the consolidation
program to produce only a single L2 block at a height that includes _all receipts_ on a single chain that must be
validated to resolve its dependents. With this, the computational complexity of resolving a dependency is drastically
reduced, only requiring two merkle proofs as well as `n` merkle patricia trie proofs to verify inclusion within the
receipt roots within the headers commited to in the accumulator tree.

This feature also allows for very efficient historical state lookups for other situations that are periphery to this
optimization for interop. For example, in the fault proof program, this feature removes the need for a commit-reveal
walkback when retrieving data within the historical chain. Instead, we would only need to provide small inclusion
proofs for a constant-time lookup of any data in the historical state accumulator.

## Constants & Definitions

| Term                      | Description                                                                                                                                |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `ACCUMULATOR_TREE_DEPTH`  | `27`                                                                                                                                       |
| `HEADER_BATCH_TREE_DEPTH` | `5`                                                                                                                                        |
| `TOTAL_TREE_DEPTH`        | `ACCUMULATOR_TREE_DEPTH + HEADER_BATCH_TREE_DEPTH`                                                                                         |
| `TRUNCATED_COMMITMENT`    | `20` bytes. Must be sufficiently long to protect against hash collision.                                                                   |
| `MERKLE_STACK_SIZE`       | `TRUNCATED_COMMITMENT * ACCUMULATOR_TREE_DEPTH`                                                                                            |
| `HEADER_BATCH_SIZE`       | `2 ** HEADER_BATCH_TREE_DEPTH`                                                                                                             |
| Accumulator Tree          | A partial merkle tree of depth `ACCUMULATOR_TREE_DEPTH`, storing the root nodes of header batch trees as leaves.                           |
| Header Batch Tree         | A binary merkle tree of depth `HEADER_BATCH_TREE_DEPTH`, storing the header hashes of `HEADER_BATCH_SIZE` block headers as leaves.         |
| Header Hash               | `keccak256(rlp(header))`                                                                                                                   |
| Merkle Stack              | The path to the latest node added to the Accumulator Tree. This is an array of `MERKLE_STACK_SIZE` bytes in length.                        |
| Partial Merkle Tree       | As specified in [_End-to-End Formal Verification of Ethereum 2.0 Deposit Smart Contract_][imt] by Daejun Park, Yi Zhang, and Grigore Rosu. |
| Zero Hashes               | As specified in [_End-to-End Formal Verification of Ethereum 2.0 Deposit Smart Contract_][imt] by Daejun Park, Yi Zhang, and Grigore Rosu. |

## Accumulator Tree Construction

The header accumulator tree is constructed as a partial merkle tree of depth `ACCUMULATOR_TREE_DEPTH`, with leaves
representing the merkle roots of binary header batch trees, containing `HEADER_BATCH_SIZE` blocks.

As an example, below is an accumulator tree with `ACCUMULATOR_TREE_DEPTH = 1` and `HEADER_BATCH_TREE_DEPTH = 2`,
holding a total of `8` blocks' header hashes across two batches:

```mermaid
flowchart TD
  A[Header Accumulator Root] --> B[Header Batch #1]

  A --> C[Header Batch #2]

  B --> B1[Intermediate]
  B --> B2[Intermediate]
  B1 --> B11[Header Hash #1]
  B1 --> B12[Header Hash #2]
  B2 --> B21[Header Hash #3]
  B2 --> B22[Header Hash #4]

  C --> C1[Intermediate]
  C --> C2[Intermediate]
  C1 --> C11[Header Hash #5]
  C1 --> C12[Header Hash #6]
  C2 --> C21[Header Hash #7]
  C2 --> C22[Header Hash #8]
```

Both the global accumulator tree as well as the header batch tree use `keccak256` as their hashing function. For all
nodes within the header accumulator tree, all commitments are shortened to `TRUNCATED_COMMITMENT` bytes in length to
save space in the Header's `extraData` field by virtue of shortening the merkle stack's encoded length. Commitments
within the header batch tree are not to be truncated.

### Interior Commitment Truncation Rules

All tree commitments that are to be truncated must save their high-order 20 bytes, when interpreting `keccak256`'s
produced digest as a big-endian 32-byte unsigned integer.

| Byte Range | Description      |
| ---------- | ---------------- |
| `[0, 20)`  | Truncated Digest |
| `[20, 32)` | Discarded        |

## Block Execution Changes

After Granite activation, every time `block.number % HEADER_BATCH_SIZE == 0`, the execution layer will:

1. Craft a binary merkle tree of depth `HEADER_BATCH_TREE_DEPTH` with the block hashes of
   `[block.number - HEADER_BATCH_TREE_DEPTH, block)` and compute its root, using the `keccak256` hash function.
1. Fetch the previous merkle stack from `block.number - HEADER_BATCH_TREE_DEPTH + 1`'s Header.
   - If the current block is the first header accumulation block, the execution client will begin with an empty
     merkle stack consisting of zero hashes from the leaf node to the root node, set the `header_batch_num` to `0`,
     and continue to the next step.
   - If the current block is not the first header accumulation block, and the block header at
     `block.number - HEADER_BATCH_TREE_DEPTH` does not contain a valid header accumulator `extraData` field,
     block execution should _always_ fail immediately.
   - If the current block is not the first header accumulation block, and the block header at
     `block.number - HEADER_BATCH_TREE_DEPTH` does contain a valid header accumulator `extraData` field, continue to
     the next step.
1. Insert the root of the header batch tree computed in step #1 into the merkle stack retrieved from the previous
   header accumulation block, using the insertion function defined below.
1. Compute the new accumulator tree root with the modified stack.
1. Encode the merkle root and modified merkle stack as defined below, and insert into the executed block's header
   prior to sealing it.

### Block Validity Changes

1. If `block.number % HEADER_BATCH_SIZE == 0`, the accumulator tree merkle root in the `extraData` must
   contain all previous header batches from the previous accumulator tree, in-order, with the current header
   batch appended in the next available leaf.
   - If `block.number` is the first header accumulation block, the accumulator merkle root in `extraData` must contain
     only the current header batch at the `0`th index.
1. If `block.number % HEADER_BATCH_SIZE == 0`, the `header_batch_num` in the header must equal the
   previous header accumulation block's `header_batch_num` plus `1`.
   - If `block.number` is the first header accumulation block, `header_batch_num` must equal `1`.
1. If `block.number % HEADER_BATCH_SIZE == 0`, the `extraData` size must be
   `8 + MERKLE_STACK_SIZE`.
1. If `block.number % HEADER_BATCH_SIZE > 0`, the `extraData` size must be `0`.

### Header Changes

#### `extraData` format

For header accumulation blocks (`block.number % HEADER_BATCH_SIZE == 0`), the accumulator tree root
as well as the merkle stack should be encoded as follows:

```txt
merkle_stack = intermediate_0 ++ ... ++ intermediate_n
extra_data = u64(header_batch_num) ++ merkle_stack
```

| Byte Range                   | Field                                                   |
| ---------------------------- | ------------------------------------------------------- |
| `[0, 8)`                     | `header_batch_num` (big-endian 64-bit unsigned integer) |
| `[8, 8 + MERKLE_STACK_SIZE)` | `merkle_stack`                                          |

where `++` denotes concatenation.

#### Header RLP Size Considerations

By extending the `extraData` field to occasionally include the merkle root of the accumulator tree as well as the
merkle stack of the previously added leaf, we do introduce a risk for bloating historical state. Mitigations for
this state expansion in this proposal include:

1. Batching additions to the global accumulator root every `HEADER_BATCH_SIZE` blocks to remove the
   requirement to store the accumulator tree root & merkle stack in every block header.
1. Truncating the intermediate commitments within the global accumulator tree to reduce the `extraData` field's
   size by `(32 - TRUNCATED_COMMITMENT) * ACCUMULATOR_TREE_DEPTH` bytes per accumulator block.

The total size of the `extraData` field in header accumulation blocks' headers will be
`8 + MERKLE_STACK_SIZE` bytes in length.

With the parameters of `TRUNCATED_COMMITMENT = 20` & `ACCUMULATOR_TREE_DEPTH = 27`, this would imply an extra `548`
bytes per `HEADER_BATCH_SIZE` blocks. At a block time of `2` seconds, this implies an extra `0.7398 MB` of
data added to historical state per day.

### Accumulator Tree Functions

#### Zero Hashes

_Modified from the
[Beacon Chain Specification](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#is_valid_merkle_branch),
to account for the truncated digests._

```python
def zero_hashes(depth: uint64) -> Sequence[Bytes20]:
  """
  Computes the zero hashes for an incremental tree of ``depth``
  """
  zero_hashes = Sequence[Bytes20.ZERO; depth]
  for height in range(depth - 1):
    zero_hashes[height + 1] = keccak256(zero_hashes[height] + zero_hashes[height])[0:20]
  return zero_hashes
```

#### Tree Root

_Modified from the
[Beacon Chain Specification](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#is_valid_merkle_branch),
to account for the truncated digests._

```python
def tree_root(branch: Sequence[Bytes20], depth: uint64) -> Bytes20:
    """
    Computes the root node of the incremental merkle tree, based off of the previous ``branch``
    """
    node = Bytes20.ZERO
    for height in range(depth):
      if ((size & 1) == 1):
        node = keccak256(branch[height] + node)
      else:
        node = keccak256(node, zero_hashes[height])
    return node
```

#### Insertion

_Modified from the
[Beacon Chain Specification](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#is_valid_merkle_branch),
to account for the truncated digests._

```python
def insert_leaf(node: Bytes32, branch: Sequence[Bytes20], header_batch_num: uint64, depth: uint64) -> bool:
    """
    Inserts the ``node`` into the merkle branch ``branch`` at ``header_batch_num + 1``. Returns false
    if the loop was exhausted without returning, which should be considered a critical failure.
    """
    header_batch_num += 1
    size = header_batch_num
    node = node[0:20]
    for height in range(depth):
      if ((size & 1) == 1):
        branch[height] = node
        return True
      node = keccak256(branch[height] + node)[0:20]
      size /= 2
    return False
```

#### Verification

_Modified from the
[Beacon Chain Specification](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#is_valid_merkle_branch),
to account for the truncated digests._

```python
def is_valid_merkle_branch(leaf: Bytes32, branch: Sequence[Bytes20], depth: uint64, index: uint64, root: Root) -> bool:
    """
    Check if ``leaf`` at ``index`` verifies against the Merkle ``root`` and ``branch``.
    """
    value = leaf[0:20]
    for i in range(depth):
        if index // (2**i) % 2:
            value = hash(branch[i] + value)[0:20]
        else:
            value = hash(value + branch[i])[0:20]
    return value == root
```

## Eventual Migration to [EIP-7685](https://eips.ethereum.org/EIPS/eip-7685)

With EIP-7685, which is being considered for inclusion in Pectra, we will have the option to move the data that this
specification assigns to `extraData` into the `requestsRoot` within the Header. This will eventually be a much more
forwards compatible method of committing to this information within historical state.

## Forwards Compatibility Considerations

It is possible that L1 chooses to implement a feature which extends the `extraData` field, which could potentially
cause us issues. However, this proposal seeks to temporarily include the information within `extraData`, until
[EIP-7685](https://eips.ethereum.org/EIPS/eip-7685) is included on L1.

## Client Implementation Considerations

For most execution clients, this should have a very minimal impact on block execution performance. Because the state
of the previous `256` blocks is commonly kept in-memory for EVM opcodes such as `BLOCKHASH`, fetching the `extraData`
field from the previous header accumulation block and recomputing the header batch + accumulator trie should be
very fast, assuming `HEADER_BATCH_TREE_DEPTH` is not too large.

## Infrastructure Considerations

Most infrastructure components support `extraData > 32 bytes`, since Clique consensus formerly used this field. However,
we do run the risk that some services such as block explorers have removed this capability.

[imt]: https://daejunpark.github.io/papers/deposit.pdf