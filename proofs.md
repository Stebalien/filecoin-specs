# Proofs

## Overview

The Filecoin protocol uses cryptographic proofs to ensure the two following guarantees:

- **Storage Based Consensus**: Miners' power in the consensus is proportional to their amount of storage. Miner increase their power by proving that they are dedicating unique storage to the network.
- **Verifiable Storage Market**: Miners must be proving that they are dedicating unique physical space for each copy of the clients data through a period of time.

We briefly describe the three main Filecoin proofs:

- __*Proof-of-Replication (PoRep)*__ proves that a Storage Miner is dedicating unique dedicated storage for each ***sector***. Filecoin Storage Miners collect new clients' data in a sector, run a slow encoding process (called `Seal`) and generate a proof (`SealProof`) that the encoding was generated correctly.

  In Filecoin, PoRep provides two guarantees: (1) *space-hardness*: Storage Miners cannot lie about the amount of space they are dedicating to Filecoin in order to gain more power in the consensus; (2) *replication*: Storage Miners are dedicating unique storage for each copy of their clients data.

- __*Proof of Spacetime*__ proves that an arbitrary number of __*sealed sectors*__ existed over a specified period of time in their own dedicated storage — as opposed to being generated on-the-fly at proof generation time.

- __*Piece-Inclusion-Proof*__ proves that a given __*piece*__ is contained within a specified __*sealed sector*__.

## Glossary

Throughout this document, the following definitions are used:

- __*sector:*__ a fixed-size block of data of `SECTOR_SIZE` bytes which generally contains clients' data.
- __*piece:*__ a block of data of at most `SECTOR_SIZE` bytes which is generally is a client's file or part of.
- __*original data:*__ the concatenation of a __*sector's*__ constitutent pieces, all __*piece padding*__, and any __*terminal padding*__.
- __*unsealed sector:*__ a concrete representation (on disk or in memory) of a sector's __*original data*__.
- __*sealed sector:*__ a concrete representation (on disk or in memory) of the unique replica generated by `Seal` from an __*unsealed sector*__.
- __*piece padding:*__ a block of zero or more 'zero bytes' inserted between __*pieces*__ to ensure they are positioned within the containing __*sector*__ in a way compatible with the __*Piece Inclusion Proof*__.
- __*terminal padding:*__ a block of zero or more 'zero bytes' inserted after a __*sector's*__ final piece, ensuring that the length of the __*original data*__ is `SECTOR_BYTES`.
- __*preprocessing:*__ a transformation applied to an __*unsealed sector*__ as the first stage of sealing and which may increase the size of the data.
- __*preprocessed data:*__ the result of __*preprocessing*__ the __*original data*__.
- __*preprocessed sector:*__ a concrete representation (on disk or in memory) of a __*sector's*__ __*preprocessed data*__.
- __*SNARK proof:*__ a block of bytes which proves that the creator is in possession of a satisfying assignment to a quadratic arithmetic program (__*circuit*__).
- __*Multi-SNARK proof:*__ a block of one or more __*SNARK proofs*__, each proving partition of the total set of challenges to be proved.
- __*Merkle inclusion proof:*__ a proof (whose format is unspecified here) that a given leaf block is contained within a merkle tree whose root is the __*commitment*__ associated with the proof.
- __*commitment:*__ an opaque block of data to which a prover 'commits', enabling subsequent proofs which cannot be validly constructed unless the __*commitment*__ itself was validly constructed. For example: the output of a suitable pseudorandom collision-resistant hash function may serve as a __*commitment*__ to the data which is the preimage of that hash. Publication of the __*commitment*__ proves that the creator was in possession of the preimage at the time the __*commitment*__ was generated.
- __*prover:*__ the party who generates a proof, in Filecoin it's always the Storage Miner.
- __*verifier:*__ the party who verifies a proof generated by a __*prover*__, in Filecoin it's a full node.



## Proof-of-Replication Algorithms

__*Filecoin Proof of Replication*__ generates a unique copy (__*sealed sector*__) of a __*sector's*__ __*original data*__, a __*Multi-SNARK proof*__, and a set of __*commitments*__ identifying the __*sealed sector*__ and linking it to the corresponding __*unsealed sector*__.

### Seal

`Seal` has the side effect of generating a __*sealed sector*__ from an __*unsealed sector*__,  and returns identifying __*commitments*__ and a __*Multi-SNARK proof*__. The proof returned is a __*Multi-SNARK proof*__.

The commitments are used to verify that the correct __*original data*__ was sealed, and that the correct __*sealed data*__ is the subject of later __*Proofs of Spacetime*__, proving that this data is being stored continuously.

`Seal` operates by performing a slow encoding of the __*unsealed sector*__ — such that it is infeasible for a dishonest prover to computationally regenerate the __*sealed sector*__ quickly enough to satisfy subsequent required __*Proofs of Spacetime*__ — thus ensuring that the __*sealed sector*__ remains manifest as a unique, concrete representation of the __*original data*__.

```
Seal
 (
  // request represents a request to seal a sector.
  partitions     uint64,      // influences the size of the output proof; using less partitions requires more hardware but produces shorter proofs
  sectorSize     uint64,      // the number of bytes in the sealed sector
  unsealedPath   string,      // path of unsealed sector (regular file, ramdisk, etc.) from which a unique replica will be created
  sealedPath     string,      // path to which sealed sector will be written
  proverID       [31]byte,    // uniquely identifies miner
  ticket         [32]byte,    // ticket to which miner commits when sealing begins
  sectorID       uint64,    // uniquely identifies sector
 ) err Error | (
  // response contains the commitments resulting from a successful Seal().
  commD          [32]byte,    // data commitment: merkle root of original data
  commR          [32]byte,    // replica commitment: merkle root of replicated data [will be removed in future iteration]
  commRStar      [32]byte,    // a hash of intermediate layers
  proof          []byte,
 )

```

### VerifySeal

`VerifySeal` is the functional counterpart to `Seal`'s proof component. It takes all of `Seal's` outputs, along with those of Seal's inputs which are required to uniquely identify the created __*sealed sector*__. This allows a __*verifier*__ to determine whether a given proof is valid.  All inputs are required because verification requires sufficient context to determine not only that a proof is *valid* but also that the proof indeed corresponds to what it purports to prove.

```
VerifySeal
 (
  // request represents a request to verify the output of a Seal() operation.
  commD       [32]byte, // returned from Seal
  commR       [32]byte, // returned from Seal [will be removed in future iteration]
  commRStar   [32]byte, // returned from Seal
  proof       []byte,   // returned from Seal
  proverID    [31]byte, // uniquely identifies miner
  ticket      [32]byte, // ticket to which miner committed when sealing began
  sectorID    uint64, // uniquely identifies sector
) err Error |
  IsValid bool          // true iff the provided proof-of-replication os valid

```

### Unseal

`Unseal` is the counterpart to `Seal`'s encoding side-effect. It reverses the 'slow encoding' and creates an __*unsealed sector*__ from a __*sealed sector*__ as a special case of its more general function. In general, it allows extraction of a range of bytes (specified in terms of the layout of the __*original data*__).

```
Unseal
 (
  // request represents a request to unseal a sector.
  sectorSize    uint64,   // the number of bytes in the sealed sector
  sealedPath    string,   // path from which sealed bytes will be read
  outputPath    string,   // path to which unsealed bytes will be written (regular file, ramdisk, etc.)
  proverID      [31]byte, // uniquely identifies miner
  sectorID      uint64, // uniquely identifies sector
  ticket        [32]byte, // ticket to which miner committed when sealing began
  startOffset   uint64,   // zero-based byte offset in original, unsealed sector-file
  numBytes      uint64,   // number of bytes to unseal (corresponds to contents of unsealed sector-file)
 ) err Error |
  NumBytesWritten uint64  // the number of bytes unsealed (and written) by Unseal()
```

### Security Notes

#### Guaranteeing sector uniqueness

Every sealed sector is unique, even if the unsealed data is identical. This prevents a malicious miner from storing the same sector twice without dedicating twice the amount of storage, or two malicious miners pretending to store the same sector, but only storing one copy. Sector uniqueness is guaranteed by having unique `proverId` and `sectorId`. Each miner has a unique `proverID`, and each sector has a unique `sectorID`within that miner's sectors. Taken together, `proverID` and `sectorID` are globally unique . Both the `proverId` and the `sectorId` are used to encode the sealed data.

The Filecoin node verifies that the correct `proverId` and `sectorId` is used when verifying the proof.

------

## Piece Inclusion Proof

A `PieceInclusionProof` contains a potentially complex Merkle inclusion proof that all leaves included in `commP` (the piece commitment) are also included in `commD` (the sector data commitment).

```
struct PieceInclusionProof {
    Position uint,
    ProofElements [32]byte
}
```

### GeneratePieceInclusionProofs

`GeneratePieceInclusionProofs` takes a merkle tree and a slice of piece start positions and lengths (in nodes), and returns
a vector of `PieceInclusionProofs` corresponding to the pieces. For this method to work, the piece data used to validate pieces will need to be padded as necessary, and pieces will need to be aligned (to 128-byte chunks, after padding, due to details of __*preprocessing*__) when written. This assumes that pieces have been packed and padded according to the assumptions of the algorithm. For this reason, practical implementations should also provide a function to assist in correct packing of pieces. All pieces will be zero-padded such that the total length of the piece is a multiple of 127 bytes before preprocessing and a multiple of 128 bytes after pre-processing.

```
GeneratePieceInclusionProofs
 (
  Tree MerkleTree,
  PieceStarts []uint
  PieceLengths []uint,
 ) []PieceInclusionProof
```

`GeneratePieceInclusionProofs` takes a merkle tree, an array of the index positions of the first nodes of the pieces, and an array of the corresponding piece lengths. It returns an array of `PieceInclusionProof`s corresponding to the supplied start/length pairs — each of which specifies a piece as a sequence of leaves (`[start..start+length]`) of `Tree`.

```
GeneratePieceInclusionProof
 (
  tree          MerkleTree,
  firstNode     uint,
  pieceLength   uint,
 ) err Error |  proof PieceInclusionProof
```


The structure of a `PieceInclusionProof` is determined by the start position and length of the piece to be proved. (Note that if `start + length` is greater than the number of leaves in `Tree`, and no `PieceInclusionProof` can be generated since these parameters do not specify a valid piece.)

The form of a `PieceInclusionProof` is as follows:
 - The piece's position within the tree must be specified. This, combined with the length provided during verification completely determines the shape of the proof.
 - The remainder of the proof is a series of `ProofElements`, whose order is interpreted by the proof algorithm and is not (yet: TODO) specified here. The significance of the provided `ProofElements` is described by their role in the verification algorithm below.


`VerifyPieceInclusionProof` takes a sector data commitment (`commD`), piece commitment (`commP`), sector size, and piece size.
Iff it returns true, then `PieceInclusionProof` indeed proves that all of piece's bytes were included in the merkle tree corresponding
to `commD` of a sector of `sectorSize`. The size inputs are necessary to prevent malicious provers from claiming to store the entire
piece but actually storing only the piece commitment.

```
VerifyPieceInclusionProof
 (
  proof PieceInclusionProof,
  commD  [32]byte,
  commP [32]byte,
  sectorSize uint,
  pieceSize uint,
 ) err Error | IsValid bool // true iff the provided PieceInclusionProof is valid.
```

The abstract verification algorithm is described below. Only the algorithm for an __*aligned `PieceInclusionProof`*__ is fully specified here. [TODO: provide details fully specifying the ordering of `ProofElements` in the general case.]

A `PieceInclusionProof` includes a start position and a sequence of `ProofElements`. Verification of a `PieceInclusionProof` is with respect to a given `commP`, `commD`, sector size, and piece size.

Piece size and start position are used, as in proof generation, to determine the shape of the proof. This shape fully determines the inputs to and order of applications of `RepCompress` which constitute the proof.

Proof verification is as follows:
 - A `ProofElement` is a 32-byte value which will be supplied as either the left or right input to `RepCompress` (along with an appropriate height) to combine with a complementary (right or left) `ProofElement` already known to the verifier.
  - Each time `RepCompress` is called on a pair of known `ProofElements`, a new `ProofElement` is considered to be known to the verifier.
  - Initially, the verifier knows only of two `ProofElements`, `commP` (the Piece Commitment) and `commD` (the Data Commitment).
  - The proof proceeds in two stages:
    1. Zero or more __*candidate `ProofElements`*__ are proved to hash to `commP` through subsequent applications of `RepCompress`. Only after `commP` has been constructed from a set of __*candidate `ProofElements`*__ do those `ProofElements` become __*eligible*__ for use in the second phase of the proof. (Because `RepCompress` takes height as an input, only `ProofElements` from the same height in either the piece or data tree's can be combined. The output of `RepCompress` is always a `ProofElement` with height one greater than that of its two inputs.)
      - `commP` itself is by definition always __*eligible*__ for use in the second proof phase.
      - An __*aligned `PieceInclusionProof`*__ is one whose `start` index is a power of 2, and for which *either* its piece's length is a power of 2 *or* the piece was zero-padded with **Piece Padding** when packed in a sector. In these cases, `commP` exists as a node in the data tree, and a minimal-size `PieceInclusionProof` can be generated.
      - In the case of an __*aligned `PieceInclusionProof`*__, zero candidate `ProofElements` are required. (This means that `commP` is the *only* __*eligible `ProofElement`*__.)
    2. Provided `ProofElements` are added to the set of __*eligible `ProofElements`*__ by successive application of `RepCompress`, as in the first phase.
      - When `commD` is produced by application of an __*eligible `ProofElement`*__ and a `ProofElement` provided by the proof, the proof is considered complete.
      - If all __*eligible `ProofElement`s*__ have been used as inputs to `RepCompress` and are dependencies of the final construction of `commD`, then the proof is considered to be valid.

 NOTE: in the case of an __*aligned `PieceInclusionProof`*__ the `ProofElements` take the form of a standard Merkle inclusion proof proving that `commP` is contained in a sub-tree of the data tree whose root is `commD`. Because `commP`'s position within the data tree is fully specified by the tree's height and the piece's start position and length, the verifier can deterministically combine each successive `ProofElement` with the result of the previous `RepCompress` operation as either a right or left input to the next `RepCompress` operation. In this sense, an __*aligned `PieceInclusionProof`*__ is simply a Merkle inclusion proof that `commP` is a constituent of `commD`'s merkle tree *and* that it is located at the appropriate height for it to also be the root of a (piece) tree with `length` leaves.

In the non-aligned case, the principle is similar. However, in this case, `commP` does *not* occur as a node in `commD`'s Merkle tree. This is because the original piece has been packed out-of-order to minimize alignment padding in the sector (at the cost of a larger `PieceInclusionProof`). Because `commP` does not exist as the root of a data sub-tree, it is necessary first to prove that the root of every sub-tree into which the original piece has been decomposed (when reordering) *is* indeed present in the data tree. Once each __*candidate `ProofElement`*__ has been proved to be an actual constituent of `commP`, it must also be shown that this __*eligible `ProofElement`*__ is *also* a constituent of `commD`.
