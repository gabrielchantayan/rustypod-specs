<!-- Trusted provenance: F-c6c3e63cae00; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c6c3e63cae00

## Purpose

F-c6c3e63cae00 is a globally controlled state machine that can read caller-supplied source storage and issue byte writes to calculated destination addresses. Two scalar inputs contribute to arithmetic state and emitted bytes.

**Established static evidence:** the body contains a source-byte transformation, an operation issuing eight ordered byte stores, and an operation storing the literal byte `0x4F`. Global values determine dispatch, arithmetic, addresses, and progression.

**Inferred intent:** these operations are consistent with constructing a byte-oriented record containing scalar-derived and source-derived data. They do not establish a record format, contiguous output, payload length, or encoding.

Behavior depends on the states actually selected. This is not an unconditional serialization, bounded-copy, or validated-buffer contract.

## Interface

The routine takes four positional inputs and returns no value.

| Input | Represented width | Established role |
|---|---:|---|
| Destination address value | 32-bit integer | Captured for subsequent destination-address calculations. |
| First scalar | 32-bit integer | Used in arithmetic and byte emission. |
| Source address | Pointer in the represented 32-bit address model | Used for four-byte integer reads and displaced byte reads. |
| Second scalar | 32-bit integer | Used in arithmetic and byte emission. |

The source pointer has a four-byte integer pointee in the supplied representation, but accesses are not exclusively word-sized. There is no source extent, destination capacity, output-length result, or error-status result.

Arguments are captured only when selection six executes. Before capture, the working source pointer, captured scalars, and destination-related state retain their initialized zero values unless another operation changes the relevant state.

### Preconditions and ownership

Safe execution requires every effective address actually accessed to be valid when accessed:

- Source-word accesses require four readable bytes and any target-required alignment.
- Source-byte accesses require one readable byte.
- Destination stores require one writable byte.
- Integer operations and integer-to-address conversions require applicable target semantics.

These conditions are not checked. The supplied global values and state sequence are insufficient to derive a required source or destination extent.

Null values are not explicitly rejected. Conversely, non-null arguments do not establish safety: caller-memory operations can be selected before argument capture or before destination working bases are established.

Storage must remain live throughout its accesses. No ownership transfer, allocation, or release is established.

## Behavior

### Initialization and dispatch

Working scalar, pointer, arithmetic, displacement, and progress state begins at zero. An initial-control global supplies the first control value.

Every dispatch iteration performs these actions in order:

1. Reads an arithmetic-factor global and caches its value for that iteration.
2. Reads a dispatch-offset global.
3. Selects an operation using the current control value plus the dispatch offset.

These reads also precede the return selection. Immediate return is therefore not free of global reads.

The cached arithmetic factor is distinct from fresh reads made inside operations. In particular, eight-store emission uses the cached value even after destination stores that might modify the factor’s storage.

Selections zero through eleven have explicit entries. Selection five and every selection outside that range enter eight-store emission. There is no invalid-selection error path.

Repeated states and nontermination are possible under unconstrained globals.

### Capture and simple control operations

| Selection | Established effect |
|---|---|
| One | Reads a successor-control global and redispatches. |
| Three | Resets working progress to zero, reads a successor-control global, and redispatches. |
| Four | Reads a successor-control global, then reads a separate global into working arithmetic state. |
| Six | Reads a successor-control global, then captures all four arguments. |
| Nine | Reads a successor-control global and redispatches. |
| Eleven | Returns without a value. |

Argument capture does not dereference the source or store to the destination. Capturing the destination value does not itself populate every working destination-base role; selection zero subsequently copies it into those roles.

Global reads are not an entry-time snapshot. The order of the two reads in selection four is established.

### Source-word inspection: selection zero

This operation performs two separate four-byte reads at the working source address.

The first contributes to a product involving a secondary-factor global and a fresh read of the arithmetic-factor global. That arithmetic-factor read is not the dispatch-cached sample. The relative evaluation order of operands within the product is not established.

After the first product, the operation:

- Copies the captured destination value into a working destination base.
- Forms the first product minus twenty.
- Performs the second source-word read in a later statement.
- Combines that second sample with another secondary-factor read and an alternative-factor read, then adds twenty.
- Bitwise-ORs the two arithmetic expressions, complements the result, and extracts its unsigned top bit as a predicate.
- Reads a successor-control global to form a candidate one less than that sample.
- Multiplies the first product by a further secondary-factor read.
- Copies the captured destination value into another working destination base.
- Reads the same successor-control global again to form the other candidate.

The predicate selects between those candidates under the represented arithmetic: zero selects the first candidate; one selects the second.

The two source reads and two successor reads must not be collapsed into single samples. The secondary-factor read between the successor reads is also separately sequenced.

This operation does not write destination memory. The first source word is not established as a length or element count.

### Source-byte transformation: selection two

Before accessing destination memory, the operation updates working progress using subtraction of a step-global sample and multiplication by a fresh arithmetic-factor sample. Both reads precede the store; their relative operand-evaluation order within the expression is not established.

It reads one byte from:

\[
\text{working source address}+4+\text{working source displacement}
\]

and stores a transformed byte at:

\[
\text{working destination base}+\text{working destination displacement}.
\]

The two displacements are separately maintained and need not be equal.

For an input byte with unsigned value \(b\), the stored low byte is:

\[
(157b+236)\bmod256.
\]

This is the eight-bit result of multiplying by negative ninety-nine and subtracting twenty. Whether the loaded byte is initially treated as signed or unsigned does not change that low-byte result.

After the store, the operation reads the secondary-factor global, multiplies progress by that sample, reads a successor-control global, and redispatches.

A destination alias of the secondary-factor or successor global can therefore affect the post-store reads. It cannot retroactively affect the completed pre-store progress update.

There is no intrinsic source bound, destination bound, fixed increment, or exhaustion check.

### Eight-store emission: selection five and default

This operation issues eight byte stores in a fixed order.

Before the first store, it captures the low byte of a byte-multiplier global. All applicable stores use that captured byte, even if an earlier store modifies the global’s storage.

For describing byte values, define:

\[
E(q)=(9q-20)\bmod256.
\]

This notation is not a dependency or an additional operation.

The byte sources are:

- A first working 32-bit quantity.
- A previously held working 32-bit quantity whose low byte supplies store four.
- A new product of the dispatch-cached arithmetic factor and the captured first scalar.
- A separate previously calculated quantity whose low byte supplies store eight.

**The previously held quantity is not necessarily a product.** Selection eight can populate it with the second scalar multiplied by 139, but selection seven assigns that same working role a bitwise-OR result. The supplied globals do not exclude a transition from selection seven into emission. Other paths may also leave initialized or earlier state in that role.

| Store | Stored low-byte value |
|---:|---|
| 1 | \(E\) applied to the captured byte multiplier times bits 31–24 of the first working quantity |
| 2 | Corresponding result using bits 23–16 |
| 3 | Corresponding result using bits 15–8 |
| 4 | \(E\) applied to the pre-replacement low byte of the previously held working quantity |
| 5 | \(E\) applied to the captured byte multiplier times bits 31–24 of the new scalar product |
| 6 | Corresponding result using bits 23–16 |
| 7 | Corresponding result using bits 15–8 |
| 8 | \(E\) applied to the low byte of the separate previously calculated quantity |

Stores four and eight do not apply the captured byte multiplier. Store four does not necessarily use the low byte of the first working quantity or of any product.

The fourth byte is preserved before the held working quantity is replaced by the new scalar product. Stores five through seven use that replacement product.

#### Addresses and step observations

All eight stores use the operation’s working destination base.

- Store one adds an already calculated displacement.
- Store two adds the cached factor multiplied by an existing offset.
- Store three adds the cached factor multiplied by a second existing offset.

After store three and before store four, the operation:

1. Reads the step global and saves that sample.
2. Captures the pre-replacement low byte for store four.
3. Reads the step global twice in an expression deriving a twice-reduced offset from the offset used for store three.
4. Replaces the held working quantity with the new scalar product.
5. Reads the step global again for store four’s address.

There are four distinct step-global read occurrences in this interval. The two within the twice-reduced expression have no established evaluation order relative to each other; both occur after the saved-step read and before the fourth-address read.

Store four uses the cached factor multiplied by the store-three offset reduced by the final fresh step sample. Store five uses the cached factor multiplied by the twice-reduced offset.

The saved step sample is then reused:

- Store six uses the twice-reduced offset minus one saved step.
- Store seven uses that offset minus two saved steps.
- Store eight uses that offset minus three saved steps.

These descriptions remain subject to the integer conversions and overflow qualifications below; they are not unbounded mathematical address calculations.

Between stores, the operation also updates working displacements used by later states. After store eight, it derives further progression state from the saved step, its signed right shift by thirty bits, the cached factor, and the reduced offset. It then reads a successor-control global, preserves the final displacement for later use, and redispatches.

No emission store causes the cached arithmetic factor to be refreshed. A store aliasing that global can affect a later dispatch, not the remaining cached-factor calculations in the current operation. Stores four through eight likewise do not change the already saved step sample.

Addresses need not be adjacent, increasing, distinct, or contained in one object. Coincident addresses are written in store order.

### Arithmetic and predicate preparation: selections seven and eight

These operations update working arithmetic, addresses, and control without directly reading or writing caller memory.

Selection seven derives state from working progression and several globals. Its calculations include:

- Offset preparation and a displacement reduced by a step-global sample.
- Bitwise combinations and a signed-shift-derived predicate.
- Assignment of a bitwise-OR result to the working quantity later replaced during eight-store emission.
- Preparation of the separate quantity supplying emission’s eighth byte.
- Two separately sequenced reads of a successor-control global.

Under arithmetic sign propagation, its predicate is zero or one. Those values select, respectively, a candidate one less than the first successor sample or the second successor sample.

Selection eight combines working progress with another arithmetic quantity, computes offsets, and prepares:

- The second scalar multiplied by a fresh arithmetic-factor sample.
- The second scalar multiplied by 139.
- The first scalar multiplied by 139.

The second of these replaces the same working quantity that selection seven can fill with a bitwise-OR result. It is not a permanent type or provenance guarantee for that state.

Selection eight also forms a signed-shift-derived predicate and reads its successor global twice. Under arithmetic sign propagation, predicate zero selects the first sample plus three, while predicate one selects the second sample.

Both operations repeatedly reference globals. Their calculations must not be interpreted as using coherent snapshots. Operand order within individual expressions is not established where the supplied representation does not sequence it.

Neither operation establishes a conventional source-length check, destination-capacity check, or validated payload-loop condition.

### Literal-byte operation: selection ten

This operation:

1. Multiplies a working displacement by a fresh arithmetic-factor sample.
2. Stores `0x4F` at the working destination base plus an existing displacement.
3. Reads the secondary-factor global and multiplies the preceding result by that sample.
4. Reads a successor-control global and redispatches.

The pre-store arithmetic-factor read and post-store secondary-factor read are distinct. A store aliasing the latter global can affect the post-store multiplication.

The literal is not established as a terminator, delimiter, or final output byte.

### Widths and arithmetic

The represented model uses 32-bit integer and address quantities, four-byte source-word accesses, and eight-bit destination stores.

- Unsigned 32-bit arithmetic wraps modulo \(2^{32}\).
- Explicit unsigned right shifts are logical.
- Byte formulas describe retained low-eight-bit results.
- Some multiplication, addition, and subtraction occur in signed arithmetic before assignment to unsigned state. Such assignment does not establish wrapping semantics for earlier signed overflow.
- Some unsigned results are assigned to signed working quantities; out-of-range conversion behavior is target-dependent.
- Signed right shifts affect predicates and progression. Arithmetic sign propagation fits their apparent roles but is not established as portable behavior.
- No address-range or overflow checks accompany address formation.
- Source-word byte order is not supplied.

The low-byte formulas do not resolve every wider arithmetic, narrowing-conversion, or address-conversion question.

## Side effects

Possible externally relevant effects are repeated global reads, two sequenced source-word reads, displaced source-byte reads, and individual destination-byte writes.

The body contains no direct global assignments, direct function calls, allocation, deallocation, or explicit error reporting.

Destination writes may nevertheless modify globals through aliasing. Consequently, read-before-store and read-after-store distinctions matter even without concurrent mutation.

Source and destination overlap is neither rejected nor specially handled. Earlier destination writes can change later source observations. The first three emission stores can also change step-global storage before the four subsequent step reads.

There is no rollback. An interruption after partial output has no established restoration mechanism. Invalid memory access has no specified graceful outcome.

## Dependencies

No direct function dependencies appear; no additional opaque function IDs are required.

Execution depends on:

- Global storage, values, and aliasing relationships.
- Accessible source and destination storage.
- Target integer, pointer, alignment, and memory-access semantics.

No unknown callee behavior, global-preservation guarantee, or ownership convention is assumed.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Immediate return.** If the first dispatch selects eleven, there are no source reads or destination stores. The initial-control read and factor-then-offset dispatch reads still occur.

2. **Capture then return.** If selection six is followed by eleven, argument capture does not dereference either caller address. Both dispatches perform their mandatory global reads.

3. **Distinct source samples.** If the source word changes between selection zero’s reads, the first product uses the earlier sample and the complemented combination uses the later sample.

4. **Transformation boundaries.** With valid effective addresses, source bytes `0x00`, `0x01`, and `0xFF` produce transformation-store bytes `0xEC`, `0x89`, and `0x4F`, respectively. Later writes may overwrite them.

5. **Transformation alias.** If the transformation store changes secondary-factor storage, the subsequent multiplication uses the changed sample; the pre-store progress update is already complete.

6. **Held value, not guaranteed product.** If emission begins with held low byte `0x12`, store four writes `0x8E`, regardless of whether that held value came from multiplication or selection seven’s bitwise OR. Replacement by a new product before the fourth store does not change the captured byte.

7. **Replacement-product distinction.** Changing only the pre-replacement held value can change store four without changing stores five through seven. Those three use the newly calculated product.

8. **Cached emission factor.** If an early emission store changes arithmetic-factor storage, the remaining emission calculations retain the dispatch-cached sample. A later dispatch reads the global again.

9. **Repeated step samples.** Changes to step storage caused by stores one through three can affect the subsequent four reads. Stores six through eight use the saved sample where specified, not new step reads.

10. **Coincident addresses.** Without interference or interruption, the last emission store targeting a particular byte determines that byte’s resulting value.

11. **Literal-store alias.** If selection ten modifies secondary-factor storage, its following multiplication observes the modified storage. Its arithmetic-factor multiplication preceded the store.

12. **Unexpected selection.** A selection outside zero through eleven enters eight-store emission rather than rejection. Valid effective addresses remain a separate obligation.

13. **Invalid access.** An inaccessible effective address provides no contractual status result, rollback, or graceful recovery.

## Uncertainties

The following remain **documented_only**:

- Actual globals, intended state sequence, normal reachability, and termination.
- Whether capture and destination-base preparation precede every memory operation.
- The meaning of the initial source word.
- Required source and destination extents.
- Whether output addresses form any contiguous layout.
- Semantic meanings of scalar-derived bytes and `0x4F`.
- Target byte order, signed-overflow behavior, signed shifts, and out-of-range conversions.
- Permitted concurrent mutation and exact memory-observation semantics.
- Exact fault behavior and caller-level recovery.

No encryption, compression, checksum, string-termination, or validated-serialization contract is established.

## Evidence

This contract uses static inspection of the supplied body only. No runtime tests have been performed.

Established evidence includes deferred argument capture, factor-then-offset dispatch reads, two sequenced source-word reads, eight ordered emission stores, pre-store and post-store global observations, repeated and saved step samples, and the literal-byte store.

The fourth emission byte comes from a pre-replacement working quantity. Its provenance is path-dependent: selection seven assigns a bitwise-OR result to that role, selection eight assigns a scalar product, and emission subsequently replaces it with a new scalar product after saving its low byte.

Record construction remains inferred intent. No global-value snapshot, execution trace, dependency body, or caller contract was supplied.
