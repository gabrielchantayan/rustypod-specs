<!-- Trusted provenance: F-a60c6ec2c98a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a60c6ec2c98a

## Purpose

F-a60c6ec2c98a initializes eight consecutive 32-bit words in caller-supplied storage. It writes a fixed value of 4, four zero values, one externally supplied 32-bit value, and two 32-bit values obtained through an external reference.

The function performs these operations in a specific order. It does not first acquire all external values and then write the destination. That distinction matters if destination storage aliases external data or if another agent can change that data during execution.

**Established static evidence:** the supplied representation contains only assignments, external reads, and a normal return. It contains no explicit branches, loops, function calls, allocation, deallocation, or error reporting.

**Inferred intent:** the routine appears to initialize a fixed-layout record. The meanings of the record, the constant 4, and the externally supplied values are not established.

## Interface

### Input

The sole explicit input is a pointer to writable storage containing at least eight consecutive 32-bit words.

For this contract, destination words are numbered 0 through 7. Their byte offsets are:

| Destination word | Byte offset | Width |
|---|---:|---:|
| 0 | 0 | 32 bits |
| 1 | 4 | 32 bits |
| 2 | 8 | 32 bits |
| 3 | 12 | 32 bits |
| 4 | 16 | 32 bits |
| 5 | 20 | 32 bits |
| 6 | 24 | 32 bits |
| 7 | 28 | 32 bits |

The described layout occupies 32 bytes, using the four-byte word width shown by the supplied representation. Required alignment is not established independently of the target environment.

The routine also uses two implicit inputs:

- An externally stored 32-bit scalar.
- An external reference through which a first and a second consecutive 32-bit word are read.

These descriptions identify roles, not additional public parameters or established semantic types.

### Output

There is no explicit return value. The output is the mutation of destination storage.

On ordinary completion with stable, non-overlapping external inputs, the destination contains:

| Destination word | Result |
|---|---|
| 0 | 32-bit value 4 |
| 1 | Value obtained from the external scalar |
| 2 | 32-bit zero |
| 3 | 32-bit zero |
| 4 | Value obtained from the first externally referenced word |
| 5 | Value obtained from the second externally referenced word |
| 6 | 32-bit zero |
| 7 | 32-bit zero |

All copied values retain their 32-bit bit patterns. The representation does not establish that they are signed integers, unsigned integers, addresses, flags, or another semantic type.

### Preconditions

For the ordinary completion contract:

- The destination must support all eight indicated writes.
- The external scalar must be readable when accessed.
- Each use of the external reference must designate storage supporting the corresponding read.
- Destination and external accesses must satisfy the execution environment’s alignment and memory-access requirements.

The routine does not check these conditions. Null pointers, insufficient storage, inaccessible memory, and invalid external references have no defined recovery behavior in the supplied evidence.

## Behavior

### Ordered accesses

The following is the source-level access order established by the supplied representation:

| Order | Access or effect |
|---:|---|
| 1 | Write zero to destination word 2. |
| 2 | Write 4 to destination word 0. |
| 3 | Read the external scalar and retain its 32-bit value. |
| 4 | Write zero to destination word 7. |
| 5 | Write zero to destination word 6. |
| 6 | Write the retained external scalar value to destination word 1. |
| 7 | Write zero to destination word 3. |
| 8 | Read the second externally referenced word and retain its 32-bit value. |
| 9 | Read the first externally referenced word and write that value to destination word 4. |
| 10 | Write the retained second-word value to destination word 5. |
| 11 | Return normally. |

This is an access-order description, not a claim about instruction boundaries, memory barriers, or visibility to concurrent observers. In particular, reading the first externally referenced word precedes the corresponding destination write, but those actions are not established as one atomic operation.

### Value capture

The external scalar is captured after destination words 2 and 0 have been written. Its captured value is subsequently written to word 1 after words 7 and 6 have been cleared.

The externally referenced second word is captured after destination word 3 has been cleared. Its captured value is retained while the first externally referenced word is read and destination word 4 is written.

Consequently:

- A later change to the scalar’s source does not, by itself, change the scalar value already captured for word 1.
- A change to the second word’s source caused by the word-4 write does not replace the second-word value already captured for word 5.
- The two externally referenced words are not acquired as an established atomic pair.
- The first word is read after the second word, despite being stored at a lower destination offset.

### Aliasing and changing external state

No non-aliasing guarantee appears in the supplied representation.

If destination storage overlaps external source storage, earlier destination writes can affect later source reads. The routine does not preserve the destination’s original contents before beginning initialization.

For example, if the external scalar occupies the same storage as destination word 2, the initial zero write occurs before the scalar read. Under ordinary sequential memory behavior, word 1 therefore receives zero rather than the original contents of word 2.

If the external pair directly occupies destination words 4 and 5, and its reference remains valid and unchanged, both values survive: the second word is captured first, the first word is read before its destination write, and the captured second word is then stored.

Aliasing involving storage that holds the external reference itself requires additional care. Earlier writes may change that reference before a later use. The supplied representation does not establish that the reference is captured once for the entire routine. A contract must not replace the observed accesses with an assumed single, stable source snapshot.

### Arithmetic and boundaries

The function performs no explicit arithmetic on copied values. There is no value addition, multiplication, comparison, sign extension, saturation, or overflow handling.

The destination extent follows from eight four-byte words: the final word starts at byte offset 28 and occupies bytes through offset 31. No explicit access beyond that extent appears.

The externally referenced second word is one four-byte word beyond the first. Address validity and address-calculation failure behavior remain properties of the target environment, not checked outcomes of this routine.

## Side effects

The explicit writes are confined to the eight destination words. Each destination word is assigned once in the supplied representation.

The routine does not read the destination merely to preserve its previous contents. Nevertheless, destination bytes may be read indirectly when they overlap an external source.

There are no explicit writes to the external scalar or externally referenced pair. Such storage can still change if it aliases the destination.

No explicit allocation, release, ownership transfer, reference-count adjustment, callback, synchronization, or I/O operation appears. The routine neither establishes ownership of the destination nor establishes that any copied 32-bit value grants ownership of another resource.

Initialization is incremental rather than transactional. If execution cannot complete an access, earlier writes have already been specified; the source shows no rollback mechanism. Whether an invalid access traps, terminates execution, or produces another environment-specific result is not established.

Repeated calls repeat the writes and external reads. Identical final results require sufficiently stable external values and access conditions; repetition alone does not establish idempotence under aliasing or concurrent modification.

## Dependencies

No called-function dependencies appear. There are therefore no additional opaque function IDs to list.

The routine depends on external data:

| Dependency | Required role | Unestablished properties |
|---|---|---|
| External scalar | Supplies one 32-bit value | Meaning, mutability, ownership, synchronization |
| External reference | Permits reads of two consecutive 32-bit words | Lifetime, stability across uses, alignment, ownership |
| Execution environment | Executes the specified memory accesses | Fault behavior, atomicity, memory ordering |

No unknown callee behavior is needed to explain the supplied routine. Conversely, the absence of calls does not establish that external storage is immutable or exclusively owned.

## Acceptance examples

These are conditional acceptance cases derived from static evidence. They are not runtime test results.

### Stable, separate sources

Assume valid, non-overlapping storage, an external scalar of `0x12345678`, and external pair values `0x89ABCDEF` followed by `0x76543210`.

After normal completion, destination words 0 through 7 must be:

| Word | Value |
|---|---|
| 0 | `0x00000004` |
| 1 | `0x12345678` |
| 2 | `0x00000000` |
| 3 | `0x00000000` |
| 4 | `0x89ABCDEF` |
| 5 | `0x76543210` |
| 6 | `0x00000000` |
| 7 | `0x00000000` |

Prior destination contents do not affect this result under those assumptions.

### Scalar overlaps an early-cleared word

Assume the external scalar is exactly destination word 2, other sources are valid and separate, and accesses behave sequentially.

Destination word 1 must receive zero because word 2 is cleared before the scalar is read.

### External pair overlaps destination words 3 and 4

Assume the external reference remains stable and points to destination word 3. Let destination word 4 initially contain `0xAABBCCDD`.

The first source word becomes zero when destination word 3 is cleared. The second source word is then captured as `0xAABBCCDD`. Completion must leave word 4 zero and word 5 equal to `0xAABBCCDD`.

This case distinguishes the established read order from an implementation that reads both original source words before clearing word 3, or reads the second source word only after overwriting word 4.

### Invalid destination

A null destination or one lacking writable space for all eight words does not satisfy the preconditions. Acceptance must not require a particular error code, exception, partial result, or safe return.

## Uncertainties

The following remain **documented_only**:

- Semantic names and meanings of all fields.
- Whether the constant 4 identifies a type, state, size, or another property.
- The target’s byte order, pointer width, and alignment requirements.
- Whether the external scalar or pair values represent pointers.
- Whether external-reference evaluations access identical storage throughout execution.
- Concurrency guarantees, access atomicity, and device-memory behavior.
- Exact machine-level access instructions or fault boundaries.
- Caller ownership obligations beyond supplying valid storage.
- Behavior when any precondition fails.

None of these unknowns should be resolved by assuming conventional constructor semantics or immutable global data.

## Evidence

The sole evidence is the supplied source representation for F-a60c6ec2c98a. It establishes eight destination assignments, three external data-word reads, their source-level ordering, and a normal return without an explicit result.

No surrounding declarations, callers, machine instructions, or runtime observations were provided. No runtime tests have been performed. The acceptance examples are conditional consequences of the stated operations, not observed executions.
