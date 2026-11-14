<!-- Trusted provenance: F-34d9c1aebced; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-34d9c1aebced

## Purpose

F-34d9c1aebced is a stateful, four-argument routine that dispatches among arithmetic operations, four dependency calls, pointer-selection operations, and byte writes. It returns an unsigned 32-bit status word.

**Established static evidence:** The routine uses three 128-byte stack buffers, one 1,300-byte stack buffer, three dependency length fields, saved arguments, separately activated working arguments, and a shared arithmetic accumulator. External values govern dispatch, arithmetic, and indexing.

**Inferred intent:** The dependency interfaces and buffer transfers suggest staged data processing. Neither the algorithm nor a particular encoding, cryptographic operation, or protocol is established.

**Contract boundary:** Concrete external values are absent. Execution order, reachability, termination, valid buffer bounds, and status meanings therefore remain **documented_only**. The behaviors below are conditional on reaching the corresponding operations; they do not establish an unconditional processing pipeline.

## Interface

### Arguments and activation

| Position | Representation | Established use |
|---|---|---|
| 1 | Pointer to a 32-bit element | Saved at entry. A separate activation operation copies it into a working holder whose first element can later be interpreted as a byte-buffer address. |
| 2 | Opaque 32-bit word | Saved at entry. Activation copies it into a working word supplied to F-ff064c948205. |
| 3 | Signed 32-bit value | Saved at entry. Activation multiplies it by the entry-captured multiplier and writes the result into a shared accumulator. |
| 4 | Byte-buffer pointer | Saved at entry. Activation copies it into a working caller-destination field, which other operations can select for byte writes. |

Saving arguments is not activation. Initially:

- The working first-argument holder is null.
- The working second-argument word is zero.
- The working caller-destination field is null.
- The shared accumulator is zero.

The activation operation copies all four saved arguments into their working roles, with the multiplication described for argument 3. It does not itself dereference argument 1 or write through argument 4.

No dispatch ordering proves that activation precedes a call or memory access. Activation may also be revisited.

### Return value

The result is the unsigned 32-bit contents of a dedicated return-status field, initially zero. A call to F-f0c4ba604fa0 can replace that field with a transformed return value. A separate arithmetic operation can also replace it.

Other dependencies update intermediate status and arithmetic fields; their returns are not necessarily returned to the caller. No success constant, error enumeration, or unchanged propagation of dependency results is established.

The return operation does not directly write an output length to caller memory.

### Preconditions

For every reached operation, safe execution requires valid addresses, adequate accessible storage, and dependency-specific preconditions.

In particular:

- The working first-argument holder must address a readable 32-bit element before dereference.
- Every byte index must designate accessible memory relative to its selected pointer.
- Dependencies must respect the actual storage supplied, unless a broader accessible region is independently established.
- External control values must produce a terminating path if termination is required.

The routine has no explicit null checks, caller-destination capacity argument, bounds rejection, or recoverable invalid-address path. Supplying valid incoming arguments alone does not establish safety if dispatch uses still-null working fields.

## Behavior

### Entry storage

Four internal buffers are allocated:

| Buffer role | Size |
|---|---:|
| Intermediate buffer | 128 bytes |
| Workspace | 1,300 bytes |
| Result buffer | 128 bytes |
| Auxiliary buffer | 128 bytes |

Their contents are not visibly initialized at entry.

There are **three distinct dependency length fields**, all initially zero:

1. A signed 32-bit input-associated length passed by address to F-f0c4ba604fa0.
2. A signed 32-bit result-associated length passed by address to F-b70666fb72d8.
3. An unsigned 32-bit auxiliary-associated length passed by address to F-d27f4f9f5f70.

The routine also initializes counters, intermediate status fields, working pointers, and arithmetic fields. Several fields serve different arithmetic roles in different operations; they are not independent semantic input lengths or permanent copies of argument 3.

### External-value timing and dispatch

For timing descriptions, the **entry multiplier** denotes the external arithmetic value captured once near entry. The **recurrent external word** denotes another external value referenced repeatedly during execution. These are descriptive roles, not dependency identifiers.

The entry multiplier is retained and reused, including after dependency calls. A dependency changing its external source does not, by itself, change this retained value.

At entry, the routine also forms an address-derived seed from the address of an internal counter multiplied by the recurrent external word read during entry setup. Later changes to that external word do not retroactively recompute the seed.

Every dispatcher iteration has these established read relationships:

1. It captures the recurrent external word.
2. It forms an unsigned selector by subtracting an external dispatch base from the current dispatch value.
3. It reads an external fallback dispatch value, including when the selector identifies a recognized operation.
4. For a recognized selector, it subsequently reads the recurrent external word again and interprets it as signed for a right shift by 30 bits.

There are 37 recognized selectors, numbered 0 through 36. An out-of-range selector causes another dispatcher iteration without executing a recognized operation body. Dispatcher bookkeeping and external reads still occur.

The iteration capture and the later signed-shift read are distinct. The captured word is used after F-d27f4f9f5f70 and in one byte-generation operation. Other arithmetic occurrences read the external word anew. Equality between these observations must not be assumed.

Some operations select subsequent dispatch values using the most significant bit of an arithmetic result. Others only transfer arithmetic state or redirect dispatch. Missing external values prevent determination of the actual state sequence, dependency-call count, loop bounds, or termination.

### Working arguments and shared accumulator

Activation sets the working first-argument holder, second-argument word, and caller-destination field from the saved arguments. It also places the entry-multiplier-scaled argument 3 into the shared 32-bit accumulator.

That accumulator is subsequently reassigned by multiple operations, including byte-generation, length-related, and branch-related arithmetic. Consequently:

- Its initial value is zero.
- Immediately after activation, it has argument-3-derived provenance.
- At a later call, that provenance holds only if intervening operations have not replaced it.

The first argument to F-f0c4ba604fa0 is the accumulator’s current value, not an independently preserved argument-3 product.

### F-f0c4ba604fa0

When its call operation is reached:

1. The routine dereferences the current working first-argument holder.
2. It interprets the resulting 32-bit element as a byte-buffer pointer and selects that pointer as the working buffer.
3. It calls F-f0c4ba604fa0 with:
   - The current shared 32-bit accumulator.
   - The pointer just read.
   - The address of the signed input-associated length.
   - The 128-byte intermediate buffer.
   - Literal 128.
   - Literal 1.
4. After the call, it transforms the returned word, updates return-status state, and calculates subsequent dispatch arithmetic.

If activation has not occurred, the holder remains null under the routine’s visible assignments. The dereference then lacks an established valid-memory precondition; this is not a defined error return.

The input-associated length starts at zero, but this call operation does not reset it. Repeated calls can therefore receive a value left by an earlier dependency invocation.

Return processing combines the retained entry multiplier with external values read after the call. The routine does not assume those external values equal their pre-call values.

### F-ff064c948205

This operation supplies:

- The 1,300-byte workspace.
- The current working second-argument word.
- An external 32-bit word used in call preparation.

Before activation, the second argument is zero. After activation, it is the saved incoming argument 2. The call operation itself does not activate arguments.

The returned word is scaled using a post-call external multiplier, combined with the retained entry multiplier, and used in intermediate status and dispatch calculations. An internal counter is reset to zero after the call.

No explicit argument communicates the workspace capacity of 1,300 bytes.

### F-b70666fb72d8

Immediately before this call, the signed result-associated length is set to 128. The seven arguments are:

1. Literal zero.
2. The 128-byte result buffer.
3. The address of that signed length.
4. The 128-byte intermediate buffer.
5. The 1,300-byte workspace.
6. Literal zero.
7. The signed input-associated length multiplied by an external factor.

The call receives **three distinct stack-buffer addresses**, as well as a length-field address.

The length value 128 is guaranteed immediately before dependency entry. It does not prove that the dependency preserves the value, produces 128 bytes, or respects that value as capacity.

Afterward, the routine transforms the return using a fresh recurrent-word observation and the retained entry multiplier, updates intermediate status, and derives dispatch state. The transformed value is not necessarily the final return-status value.

### F-d27f4f9f5f70

Immediately before this call, the unsigned auxiliary-associated length is assigned an external value. The arguments are:

1. The 128-byte result buffer.
2. Literal zero.
3. A product of the signed result-associated length and three external factors, masked to its low 16 bits.
4. The 128-byte auxiliary buffer.
5. The address of the newly initialized unsigned length.

The third argument is numerically between 0 and 65,535. The mask does not constrain it to 128 or less. The external factors in this argument include a fresh recurrent-word observation, not necessarily the dispatcher’s earlier capture.

After dependency return, the routine first scales the return word using the recurrent word captured at the beginning of the current dispatcher iteration. It combines the result with the retained entry multiplier and another external factor read after the call. This use of the earlier capture must not be replaced by a post-call reread.

The auxiliary-associated length is initialized before each reached call. Its dependency-written value can later influence length-derived arithmetic.

### Pointer roles and byte transformations

The routine maintains separate pointers for:

- The activated caller destination.
- A working buffer.
- A byte-copy source.
- A byte-store destination.

They initially contain null. Their later assignments are not interchangeable:

- Reading the caller-supplied address selects the working buffer but does not simultaneously select the byte-store destination.
- Selecting the result buffer sets both the working buffer and byte-store destination.
- Selecting the workspace changes the working buffer without necessarily changing the byte-store destination.
- Selecting the auxiliary buffer as source also selects the activated caller destination as both working buffer and byte-store destination.

Two operations read a source byte and store the following transformed value at the same selected index:

\[
\operatorname{output}=(-99\operatorname{input}-20)\bmod256.
\]

One derives the index from a counter and an external offset, then updates that counter after the store. The other uses an index calculated by preceding arithmetic and then updates a separate counter.

These are individual byte operations. A whole-buffer transformation, byte count, or traversal direction is not established.

Another operation compares the working-buffer pointer with the source pointer and branches on address order. Because the working-buffer and byte-store pointers can differ, this comparison does not alone establish the ordering of the actual write destination.

Each transformed-byte operation reads its source byte before storing the destination byte. There is no visible full-source snapshot. With overlap, earlier writes can change later source reads.

### Byte generation

Two other operations generate bytes without reading a source-buffer byte. Both:

- Update an indexing counter.
- Derive a byte index from the updated counter.
- Replace the shared accumulator using sequence arithmetic.
- Form a byte using multiplication, bitwise OR with 139, addition, and the factor −67.
- Store only the low eight bits.
- Update a sequence-related counter.

Their inputs are not identical. One uses a sequence counter in an expression involving −126; the other uses a separate arithmetic field in the corresponding byte construction.

The first stores through the working-buffer pointer and then aligns the byte-store pointer with that pointer. Its final sequence update uses the dispatcher’s captured recurrent word as one multiplier, while other recurrent-word occurrences are fresh reads.

The second stores through the byte-store pointer and uses fresh recurrent-word occurrences in its final sequence update. Both use the signed-shift-derived value calculated before the operation body.

No particular generated sequence, initialization pattern, padding convention, or security property is established.

### Length and arithmetic widths

A separate operation begins with the auxiliary-associated length’s low 16 bits, multiplies by an external factor, and masks again to 16 bits. It derives further 32-bit arithmetic through multiplication, complement, bitwise OR, and addition. This is not propagation of the original full-width length.

Unsigned 32-bit arithmetic wraps modulo \(2^{32}\). Byte stores retain the low eight bits. The affine byte mapping is therefore valid regardless of the signed interpretation of an intermediate byte.

The representation includes both logical unsigned shifts and arithmetic signed shifts. In particular, signed right shift by 30 is not equivalent to extracting only the top bit.

Some calculations involve signed 32-bit values, including argument 3 and an address-derived value. Exact target behavior for signed overflow, address-to-integer conversion, and ordering unrelated pointers remains unresolved. A portable implementation must not silently substitute different semantics.

## Side effects

### Caller memory and external state

Depending on dispatch, the routine can directly:

- Read the 32-bit element addressed by the working first-argument holder.
- Write generated or transformed bytes through selected pointers, including the activated caller destination.
- Pass caller-derived and internal addresses to dependencies.

No direct assignment replaces the incoming argument-1 element. Aliasing or dependency effects can nevertheless change memory later read through it.

No direct allocation, deallocation, file access, synchronization, or external-global assignment is visible. Such effects inside dependencies remain unknown.

External reads occur during entry, every dispatcher iteration, call preparation, post-call processing, and byte arithmetic. The routine is not established to be a pure function of its four arguments. Reentrancy and thread safety are **documented_only**.

### Storage lifetime and failure effects

All four buffers and all three dependency length fields belong to the invocation’s stack storage. Passing their addresses does not transfer ownership.

Dependency retention of these addresses, memory release behavior, and unrelated memory management are unknown. Safe conventional stack use requires that retained internal addresses not be accessed after the invocation ends.

There is no visible final buffer erasure, caller-output rollback, or universal error cleanup. A dependency return that influences an error-like branch does not establish that buffers remained untouched.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-f0c4ba604fa0 | Receives the current shared accumulator, an address read through the working holder, a signed length address, the intermediate buffer, 128, and 1. | Accumulator meaning, formats, capacities, return meanings, mutation, ownership. |
| F-ff064c948205 | Receives the workspace, the working second-argument word—initially zero—and an external word. | Workspace requirements, argument meanings, initialization, side effects. |
| F-b70666fb72d8 | Receives result, intermediate, and workspace buffers; a signed length address initialized to 128; two zero controls; and a computed final argument. | Actual sizes, length interpretation, failure effects, ownership. |
| F-d27f4f9f5f70 | Receives result and auxiliary buffers, zero, a low-16-bit input length, and an unsigned length address initialized externally. | Accepted lengths, capacity interpretation, output representation, failure effects. |

No dependency is assumed to preserve globals, preserve lengths, initialize complete buffers, avoid retaining addresses, or leave memory unchanged on failure.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Activation and the second argument.** If F-ff064c948205 is reached before activation, its second argument is zero. If activation has occurred, its second argument is the saved incoming argument 2.

2. **Caller-address dereference.** If activation has occurred and argument 1 addresses a readable 32-bit element containing a valid buffer address, the relevant call operation passes that contained address to F-f0c4ba604fa0. Without activation, a valid incoming argument 1 does not itself make the working-holder dereference valid.

3. **Accumulator provenance.** If activation is immediately followed by the F-f0c4ba604fa0 operation without an accumulator replacement, the dependency receives the entry-multiplier-scaled argument 3. If an intervening operation replaces the accumulator, it receives that replacement instead.

4. **Three independent lengths.** Changing the input-associated length does not directly change the result-associated or auxiliary-associated length. Each occupies separate storage.

5. **Per-call initialization.** Every reached F-b70666fb72d8 call receives a result-associated length initially set to 128. Every reached F-d27f4f9f5f70 call receives an auxiliary-associated length freshly set from an external value.

6. **Affine mapping.** At a valid selected index, source bytes 0, 1, and 255 produce destination bytes 236, 137, and 79 respectively.

7. **Masked call length.** If the product supplying F-d27f4f9f5f70 has low 16 bits equal to 2, its third argument is 2. Higher bits are discarded. A result greater than 128 encounters no explicit bounds rejection here.

8. **Captured versus changed external state.** If F-d27f4f9f5f70 changes the recurrent external word, its return is still initially scaled by the earlier dispatcher capture. Later fresh reads may observe the changed value.

9. **Fallback observation.** Even an iteration selecting a recognized operation reads the fallback dispatch value before executing that operation.

10. **Overlap and return.** Overlapping byte operations observe memory after preceding writes, not an original snapshot. Independently, if return is reached before any return-status replacement, the returned word is zero. Reachability of such a path is not established.

## Uncertainties

The following remain **documented_only**:

- Concrete external values, mutability, and equality between separate reads.
- Reachable operation order, activation timing, call counts, and termination.
- The accumulator value at each dependency call.
- Generated-byte sequences, complete index sequences, and caller capacity requirements.
- Dependency directions, supported lengths, ownership, retention, and failure behavior.
- Intended formats and algorithm.
- Success and failure encodings.
- Permitted overlap and whether pointer ordering implements an intended overlap strategy.
- Exact signed-overflow, pointer-conversion, and pointer-ordering semantics.
- Whether address-dependent arithmetic is intentional functional behavior.
- Defined behavior for invalid or null addresses beyond the absence of explicit checks.

These gaps prevent an unconditional end-to-end data-processing contract.

## Evidence

**Established static evidence:** The supplied representation supports the argument snapshots and separate activation, initially zero shared accumulator, three length fields, four buffer allocations, dependency call shapes, pointer assignments, affine byte mapping, masked lengths, dispatcher reads, captured multipliers, and return-status access described above.

**Inferred intent:** Staged processing and possible address-order-sensitive copying are interpretations only.

**Unknowns:** External constants, dependency implementations, runtime state histories, and target-specific arithmetic details were not supplied.

No runtime tests have been performed.
