<!-- Trusted provenance: F-e8c11dbde4a4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e8c11dbde4a4

## Purpose

F-e8c11dbde4a4 coordinates two output-producing dependencies, prepares a 16-byte scratch region through another dependency, conditionally performs byte writes, and may update a caller-supplied 32-bit word. It also invokes a dependency with no explicit arguments after selecting pointers in internal storage.

The supplied static representation establishes these operations and their local ordering. It does not establish a complete execution sequence: entry, transitions, arithmetic factors, and branch selection depend on global words whose contents are unavailable.

This contract distinguishes:

- **Established static behavior:** operations visible in the supplied representation, conditional on reaching the relevant alternative.
- **Inferred intent:** interpretations suggested by data movement, such as capacity checking or cleanup.
- **documented_only:** unresolved behavior that must not be replaced with an assumed conventional implementation.

In particular, neither a successful “prepare, transform, copy, release” sequence nor a complete error-handling sequence is established.

## Interface

The function accepts four inputs and returns an unsigned 32-bit value.

| Input | Representation | Established role |
|---|---|---|
| Caller word | Pointer to an unsigned 32-bit word | Saved at entry; copied into a separate working pointer by a setup alternative. Consumers read or write through the working pointer. |
| Scalar | Signed 32-bit integer | Saved at entry; a setup alternative multiplies it by a global factor to populate retained arithmetic state. |
| Auxiliary value | Signed 32-bit integer | Saved at entry; copied into a separate working scalar by the setup alternative. Consumers forward or interpret the working scalar as an address. |
| Caller destination | Byte pointer | Saved at entry; copied into a separate working destination pointer by the setup alternative. Later alternatives can select other destinations. |

The visible address arithmetic and conversions use a 32-bit representation. Behavior with wider pointers is not established.

### Saved inputs and working inputs

Saving caller inputs is distinct from making them available to consumer alternatives.

Initially:

- The working caller-word pointer is null.
- The working auxiliary scalar is zero.
- The working destination pointer is null.
- The retained arithmetic state later used in constructing the first helper argument is zero.

A separate setup alternative:

- Copies the saved caller-word pointer into its working pointer.
- Copies the saved auxiliary value into the working auxiliary scalar.
- Copies the saved destination into the working destination pointer.
- Computes a globally scaled version of the saved scalar and retains it for subsequent arithmetic.

The dispatch graph is unresolved. Reaching a consumer alternative does **not** prove that setup has occurred.

The working auxiliary scalar and caller-word pointer have no other visible assignments beyond initialization and setup. The working destination pointer can also be replaced with the selected source pointer during an address-order alternative. Other arithmetic alternatives can replace the retained numeric state used by the first helper.

Consequently, caller-input provenance must be conditioned on both setup and any intervening assignments.

### Output slots and scratch storage

The function initializes the following dependency-facing storage:

- First-helper 32-bit output word: zero.
- First-helper 32-bit metadata slot: zero.
- First-helper pointer output: null.
- Second-helper 32-bit count output: zero.
- Second-helper pointer output: null.

A separate alternative resets the second-helper count and pointer outputs to zero and null before testing the working caller-word pointer.

The function also provides a 16-byte scratch region. No entry-time initialization of that region is visible. Its contents depend on prior writes and dependency behavior.

### Preconditions

There is no established unconditional null-tolerant interface. Preconditions apply to the **actual working values** at each operation:

- A working caller-word pointer must designate readable or writable 32-bit storage when dereferenced.
- Selected source and destination pointers must cover every offset actually accessed.
- For the auxiliary-data preparation alternative, the working auxiliary scalar must represent an address with a readable 32-bit field at byte offset 4. The address at byte offset 8 is passed onward.
- Dependency-facing local addresses must be usable under the dependencies’ unknown contracts.

Valid caller inputs alone do not satisfy these conditions if setup has not populated the working values. Conversely, the existence of a null-test alternative does not establish that all dereferences are guarded.

Alignment requirements, supported aliases, object extents, and valid scalar ranges remain unresolved.

## Behavior

### Dispatch and arithmetic model

Execution repeatedly selects an alternative using an arithmetic combination of retained state and a global word. The initial state is also global-derived. Most alternatives compute another state and redispatch.

The textual ordering of alternatives is not execution ordering. Neither reachability nor termination can be reconstructed without the global values. The default alternative performs arithmetic and redispatches; it is not an explicit invalid-state error.

Unsigned 32-bit results retain their low 32 bits. Byte stores retain their low eight bits. Some displayed expressions involve signed integers, character conversions, signed shifts, or address-to-integer conversions. Their full portability and overflow semantics are not established by this representation.

For the sampling discussion below, **factor A** and **factor B** denote two distinct recurring global scaling words. These labels identify global storage roles, not fixed values or local source identifiers.

### First dependency: argument provenance and global sampling

One alternative invokes F-d84d90ccd501 with five arguments:

1. A 32-bit value formed by multiplying the current retained numeric state by a fresh read of factor B.
2. The address of the first-helper word output.
3. The current working auxiliary scalar.
4. The address of the metadata slot.
5. The address of the first-helper pointer output.

The third argument is not an unconditional forwarding of the caller’s auxiliary input. Without setup, its visible initial value is zero. With setup, it has the saved auxiliary value.

The first argument is likewise not unconditionally a fixed function of the caller’s scalar. Setup supplies a globally scaled caller scalar to the relevant retained state, but other alternatives can subsequently replace that state.

The global-sampling distinction around this call is established:

1. At dispatch entry, the function captures factor B.
2. In the helper-call alternative, it reads factor B again when constructing the first argument.
3. It invokes F-d84d90ccd501.
4. After the call, it reads factor A and multiplies that value by the helper’s return.
5. It multiplies this product by the **dispatch-entry capture** of factor B, not by a new post-call read of B.
6. Further status arithmetic reads factor A again. The branch predicate also combines the captured B with another read of A shifted right by 30 bits.

The captured B is therefore distinct from both the argument-construction read and any later value in B’s global storage. The supplied evidence does not permit treating these samples as one stable constant. Nor may all status factors be described as post-call reads.

For example, if a dependency changes factor B, the already-constructed call argument and the pre-call capture are unaffected by that later change. If it changes factor A, the subsequent reads of A can observe the changed value.

The resulting arithmetic feeds retained status values and a high-bit predicate. No particular helper return, including zero, is established as success or failure.

### Metadata-dependent branch

Another alternative examines only the low byte of the metadata slot. It multiplies character-sized interpretations by positive and negative 21, combines the products bitwise, converts the result through a character-sized value, and extracts a sign-related bit after widening.

Upper metadata bytes do not participate in this predicate. Character signedness matters to its interpretation; the representation does not establish a portable Boolean meaning such as “metadata is nonzero.”

### Scratch preparation

F-51bc3d1d9a13 has two visible invocation forms. Both receive a retained 32-bit value as their first argument and the scratch-region address as their fourth argument.

The zero-valued form supplies zero for both middle arguments.

The auxiliary-data form:

- Uses the **working auxiliary scalar** as an address.
- Reads a 32-bit field at byte offset 4.
- Multiplies that field by global factors for the second argument.
- Supplies the working auxiliary address plus 8 as the third argument.

The field read occurs before the call. No layout validation or bounds check is visible in that alternative.

If setup has not executed, this alternative uses the initial working value zero, not a valid caller-supplied auxiliary address. The representation consequently exposes an attempted field read at address 4 in that case; it does not establish that such a read is valid or recoverable.

The field’s meaning, payload length, first argument’s meaning, and scratch initialization coverage are unknown.

### Second dependency

Another alternative invokes F-dc10f5d75f95 with:

1. The current first-helper word output.
2. The address of the second-helper count output.
3. The scratch-region address.
4. The address of the second-helper pointer output.
5. The current first-helper pointer output.

Its return is combined with global factors in post-call arithmetic and a high-bit branch predicate.

Reaching this alternative does not establish that the first helper or scratch preparation has run. In their absence, the output slots retain their initial values unless another visible reset or an unknown dependency effect has changed them. The scratch region has no corresponding zero-initialization guarantee.

No allocation policy, output validity rule, or failure convention is available.

### Buffer selection and byte transformation

A buffer-selection alternative takes the second-helper pointer output as the selected source and the current working destination pointer as the selected destination. It also derives several retained values from the second-helper count and global factors.

Another alternative selects the second-helper pointer output itself as a destination. An address-order alternative compares the selected destination against the selected source and then replaces the working destination pointer with the selected source.

Thus, the caller destination is only one possible origin of the eventual write target.

Two alternatives perform a source-byte read followed by a destination-byte write at a computed offset. For an unsigned source byte \(x\), the stored byte is:

\[
T(x)=(181x+36)\bmod 256.
\]

This low-byte result follows from the visible multiply-add expression. It does not require choosing signed or unsigned interpretation for the final stored byte.

The offsets, traversal bounds, and progression depend on global values and retained arithmetic state. An address-order comparison suggests overlap-sensitive processing, but does not establish safe forward or reverse traversal, whole-buffer in-place support, or a particular number of processed bytes.

### Generated-byte writes

Two other alternatives write bytes without reading a source-buffer byte in those alternatives.

Their values depend on retained arithmetic state, low-byte conversions, global factors, and an address-derived seed. The seed is initialized using the address of a local 32-bit word multiplied by a global word.

One setup alternative selects an internal pointer as the byte destination and sets a counter to 16. This does not establish exactly 16 stores: subsequent updates, bounds, and transitions remain global-dependent.

The generated data cannot be specified as a known hash, key schedule, deterministic encoding of explicit inputs, or cryptographic primitive.

### Caller-word access and return

One alternative performs a 32-bit store through the working caller-word pointer. The value is the second-helper count multiplied by global factors.

A different alternative reads through that working pointer twice in a larger size-related arithmetic expression. It also tests whether the current working destination pointer is non-null. The expression uses masks, complements, and sign extraction; it is not established as an ordinary capacity comparison.

The terminal alternative returns the currently retained unsigned 32-bit value. Visible assignments can derive that value from global constants, helper returns, counts, and byte-processing arithmetic. No exhaustive status-code mapping is established.

## Side effects

### Observable memory operations

Conditional on reachability, the function can:

- Repeatedly read global words.
- Read and write a 32-bit word through the working caller-word pointer.
- Read a 32-bit field at working auxiliary address plus 4.
- Pass working auxiliary address plus 8 to a dependency.
- Read individual bytes through a selected source pointer.
- Write individual bytes through a selected destination pointer.
- Expose addresses of local output slots and the scratch region to dependencies.

Established local orderings include:

- Dispatch-entry capture of factor B precedes its separate helper-argument read and the F-d84d90ccd501 call.
- First-helper status arithmetic follows that call while retaining the earlier B capture.
- The auxiliary field read precedes its F-51bc3d1d9a13 invocation.
- Each transformed source-byte read precedes its corresponding destination-byte store.
- Internal pointer selection precedes each visible F-5e3309c9d083 invocation.

A total ordering across all categories is unresolved. No dependency is assumed to preserve globals or leave pointer-reachable state unchanged.

### Ownership and cleanup-like operations

F-5e3309c9d083 is called without explicit arguments. Immediately before its visible calls, an internal pointer slot is assigned either the first-helper pointer output or the second-helper pointer output.

Other alternatives assign the scratch-region address or another selected pointer to that slot without immediately invoking the dependency.

This suggests an implicit-input mechanism, possibly cleanup, but does not establish one. The evidence does not show that F-5e3309c9d083 reads the slot, releases memory, accepts null, preserves state, or can operate on stack-backed storage.

No allocation is directly visible in this function. No ownership transfer, release guarantee, or caller-storage preservation guarantee follows from the body alone. Dependency retention of local addresses is also unresolved; no valid lifetime beyond this invocation is established.

There is no explicit I/O or exception-handling mechanism in the supplied body. Dependencies may have additional effects.

## Dependencies

| Opaque ID | Established interaction | documented_only |
|---|---|---|
| F-d84d90ccd501 | Five arguments; receives three local output addresses and the working auxiliary scalar. Return feeds status arithmetic with mixed pre-call captures and post-call global reads. | Input meanings, success convention, output validity, ownership, global effects. |
| F-51bc3d1d9a13 | Four arguments; receives scratch storage and either zero-valued middle arguments or a scaled field and derived payload address. | Payload extent, scratch coverage, argument meanings, retention, other effects. |
| F-dc10f5d75f95 | Consumes current first-helper outputs and scratch storage; receives count and pointer output addresses. | Format, size semantics, allocation, failure behavior, ownership. |
| F-5e3309c9d083 | No explicit arguments; visible calls follow internal pointer-slot assignments. | Implicit inputs, cleanup semantics, null handling, state preservation. |

## Acceptance examples

These are conditional static criteria, not executed tests.

### Distinct working inputs

If the first-helper alternative is reached before setup, its third argument is the initial working auxiliary value zero, regardless of the saved caller auxiliary input.

If setup executes and the first-helper alternative is later reached, its third argument is the saved auxiliary value. Its first argument still depends on the current retained numeric state; intervening alternatives may have changed that state.

A valid saved caller-word pointer does not make a pre-setup working-pointer dereference valid. The working pointer initially remains null.

### Sampling across the first helper

Suppose factor B is captured as \(b_0\) at dispatch entry and read as \(b_1\) for argument construction. The helper’s first argument uses \(b_1\). Post-call multiplication by B uses \(b_0\).

If the helper changes B to \(b_2\), that change does not replace either earlier sample. Post-call reads of factor A remain separate observations and may reflect helper changes. Acceptance must not collapse these accesses into one immutable factor.

### Auxiliary-data preparation

If setup has populated the working auxiliary scalar with a valid address, the auxiliary-data alternative reads the 32-bit field at that address plus 4 and passes the address plus 8 onward.

Without setup, the same alternative uses the zero working scalar. It must not silently substitute the saved caller address.

The zero-valued preparation form supplies zero for its two middle arguments and contains no auxiliary-field read.

### Transformed bytes

For valid selected addresses and a reached transformation alternative:

| Source byte | Stored byte |
|---|---|
| 0 | 36 |
| 1 | 217 |
| 128 | 164 |
| 255 | 111 |

For an exact same-address read and write, that individual operation transforms the pre-store byte once. This does not prove safe transformation of an overlapping range.

### Word updates and scratch contents

If the caller-word update alternative is reached with valid working storage, it performs one visible 32-bit store of the globally scaled second-helper count, not necessarily the count itself.

Uninitialized scratch storage must not be treated as zero-filled. A zero helper return must not automatically be classified as success.

Metadata values with identical low bytes produce identical inputs to the visible metadata-byte predicate; upper 24-bit differences alone do not affect it.

## Uncertainties

The following remain **documented_only**:

- Global values, stability, dispatch reachability, entry alternative, and termination.
- Whether setup precedes any consumer.
- Complete success and error conventions.
- Actual buffer lengths, capacity requirements, and processed-byte counts.
- Whether the caller word represents capacity, length, or another quantity.
- Null handling and auxiliary-layout validity.
- Endianness, alignment, signed-overflow interpretation, and character signedness where sign extraction matters.
- Address-comparison validity across unrelated objects.
- Overlap safety and traversal direction.
- Generated-byte determinism and the significance of the counter value 16.
- Dependency effects on globals, exposed storage, ownership, and lifetime.
- Behavior after invalid memory access or arithmetic outside supported representations.

## Evidence

This contract is based on the supplied static function representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent evidence.

Direct evidence establishes initialization, the separate working-input setup, dependency argument shapes, the distinct factor-B samples around F-d84d90ccd501, subsequent reuse of the pre-call capture, individual memory operations, scratch extent, and the byte-transform equation.

Preparation, capacity checking, overlap handling, and cleanup are inferred roles only. Missing global contents and dependency contracts prevent promoting them into unconditional guarantees.

No runtime tests have been performed.
