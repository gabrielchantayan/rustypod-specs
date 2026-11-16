<!-- Trusted provenance: F-ebb851b304ef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ebb851b304ef

## Purpose

F-ebb851b304ef executes an ordered sequence of unresolved indirect invocations, transforms dependency-produced values, performs mixed-width memory mutations, and invokes a final unresolved target.

The supplied static evidence does not establish an application-level purpose. Numeric constants resembling characters do not establish formatting or text output. Repeated offsets do not establish a container, record type, or initialization operation.

This contract distinguishes:

- **Established static evidence:** represented calls, arithmetic, memory-access widths, data dependencies, and ordering.
- **Inferred intent:** possible application-level interpretations, none adopted as acceptance requirements.
- **documented_only:** unresolved execution semantics or portions of the trace not fully specified here.

Requirements are conditional on execution reaching the relevant operation and on the supplied representation accurately describing execution. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Width | Established use |
|---|---:|---|
| First argument | 32 bits; interpretation unspecified | Forwarded unchanged to the first indirect invocation. |
| Second argument | 32 bits; represented as signed | Forwarded unchanged and also supplied after a left shift of six positions. |

Neither explicit argument is directly dereferenced by the visible body. Dependencies may nevertheless interpret either argument as an address.

The first invocation target depends on an incoming link-related execution value plus 4094. Thus, the two explicit arguments are not the entire execution context.

### Output and completion

The represented interface has no explicit return value.

A final unresolved indirect invocation receives four represented 32-bit arguments. A return follows if that target returns normally. Neither normal return nor a conventional callback interpretation is established.

No status code, output length, exception interface, or success indicator is visible.

### Preconditions

Successful execution requires:

- Every reached indirect target to be callable under the actual execution convention.
- Every reached read and write to be valid for its represented width.
- Required execution-register and global state to be present.
- Address arithmetic and shifts to have suitable target-specific semantics.

These conditions include computed addresses, loaded addresses, and absolute low addresses. Validating only the two explicit arguments would not establish memory safety.

Required mappings, alignment, argument ranges, initial memory contents, and the completeness of the represented function boundary are **documented_only**.

### Ownership

No ownership transfer is established for inputs, returned values, or loaded addresses.

No allocation or release operation is identified in the visible body. Unresolved dependencies may allocate, retain, replace, or release resources. Their ownership behavior and preservation of global state must not be assumed.

## Behavior

### Dependency phase

The first indirect invocation receives the first argument unchanged, the second argument unchanged, and the second argument shifted left by six positions, in that order.

Many further indirect invocations precede direct memory mutation. Most targets are constructed from code-related locations plus 4094. Some invocations show no explicit arguments; others receive up to four transformed values.

An invocation without represented arguments does not establish a zero-argument machine interface. Later arguments sometimes depend on secondary result-register contents whose production and calling convention are unresolved.

The dependency phase has these observable constraints:

- Invocation order and returned-value dependencies are significant.
- An unused primary result does not make an invocation removable.
- Similar call shapes do not establish identical targets.
- Repeated invocations cannot be merged on the assumption that they are pure.
- Dependencies cannot be assumed to preserve globals or memory.

The complete dependency trace is not enumerated here. Its unresolved interfaces and full trace coverage remain **documented_only**, rather than being replaced with a guessed high-level algorithm.

### Widths and arithmetic

The representation contains 8-, 16-, 32-, and 64-bit quantities. Many 64-bit results are consumed as separate low and high 32-bit portions. Other operations extract bits beyond position 31 directly from a 64-bit result.

Transformations include fixed left shifts up to 31 positions, explicitly unsigned logical right shifts, low-bit masks, scaling, and subtraction of 150 before further scaling. Byte stores retain only the low eight bits. Unsigned byte loads become values from 0 through 255 when published as words or used in address calculations.

Unlimited-precision arithmetic is not an acceptable substitute for fixed-width behavior. Under 32-bit bit-vector semantics:

- A left shift by eight followed by a logical right shift by eight retains only the original low 24 bits.
- Two successive left shifts by 28 positions yield zero.
- Multiplication and address formation discard overflowing high bits.

Some operations are represented on signed values. Portable signed-overflow behavior is not established. Arithmetic examples below explicitly identify when they assume truncating 32-bit machine semantics.

### Address roles at the mutation boundary

The following labels describe dataflow, not object types:

| Label | Derivation |
|---|---|
| **Packed base** | A retained dependency result scaled by 33,554,432. |
| **Byte-address base** | Another dependency-derived value shifted left by 25 positions. |
| **Small base** | Twice a value formed by shifting a result left by eight, retaining its low 15 bits, and logically shifting right by six. |
| **Retained word** | A dependency result shifted left by eight. |
| **Final pair** | Low and high 32-bit portions of the last represented 64-bit dependency result before direct stores. |

The opening mutation sequence publishes the byte-address base as a 32-bit word at the final pair’s high portion plus 20. It then writes the low byte of that high portion at byte-address-base offset 9.

A word store of the packed base at 256 times the final pair’s low portion plus 84 intervenes before another byte store to byte-address-base offset 9. That second byte store uses the low byte of the byte-address base.

Both byte stores are observable events. A final-state description that retains only the second is insufficient.

### Early zero field: overlap does not invalidate the result

The function stores a zero word at packed-base offset 20, reads an unsigned byte at packed-base offset 5, stores a zero word at small-base offset 20, and publishes the loaded byte as a word at packed-base offset 52. It then reads the word at packed-base offset 20.

Under ordinary valid 32-bit-addressed memory semantics, without external mutation, this read returns zero. Non-overlap with the small-base store is **not** required:

- An overlapping zero-word store cannot introduce a nonzero byte into the already zeroed field.
- The four-byte fields at packed-base offsets 20 and 52 are disjoint, including under modulo-\(2^{32}\) address arithmetic.
- The intervening byte read does not mutate ordinary memory.

Device-memory read effects, nonordinary write behavior, or external mutation can invalidate this conclusion and remain **documented_only**.

After this read, another zero word is stored at small-base offset 20, followed by two byte writes at small-base offset 21 using the loaded zero’s low byte.

The retained word is subsequently stored through the loaded zero plus offset 84. Under the ordinary-memory assumptions above, this is a word store to absolute address 84. There is no null-address guard.

### Early named-global ordering

The byte at absolute address 5 is read only after the preceding computed stores, including the repeated byte writes at small-base offset 21. Its value is zero-extended and retained for subsequent addressing. It is not an entry-state snapshot.

Following that read:

- A byte is written at absolute address 1 using the low byte of the small base.
- Computed byte and word accesses use the retained address-5 byte as a base.
- A word at that byte value plus 84 is read before that same word destination is overwritten with the earlier loaded zero.
- Later, a 32-bit write to absolute address 84 publishes twice the word obtained from that computed read.
- After further computed writes, a byte at absolute address 21 receives the low byte of the small base.

Computed writes may alias these absolute locations. The global labels do not establish independent storage.

### Subsequent loaded-address chains

The following mutation region uses word fields as addresses and byte fields as small address bases. Important access shapes include:

- Loading a word at offset 116 and storing through that value at offset 84.
- Reading an unsigned byte at offset 5, then loading a word at that byte value plus 116.
- Loading a word at offset 100 and mutating the referenced storage.
- Re-reading byte or word fields after earlier computed writes.
- Repeating byte stores and word stores even when their source values appear unchanged.

These later reads are genuinely alias-sensitive. The early guaranteed-zero argument does not extend to them: intervening writes can contain nonzero values and can overlap the fields being read.

The visible offsets include:

| Width | Representative offsets |
|---|---|
| 8 bits | 1, 5, 8, 9, 13, 16, 17, 21, 25 |
| 16 bits | Computed scaled addresses |
| 32 bits | 4, 20, 32, 36, 52, 64, 68, 84, 100, 116 |

This is an access-shape inventory, not a complete record layout. No shared object type, object size, or non-aliasing property is established.

### Literal writes and global-base acquisition

A later loaded word determines computed destinations for numeric 58, written using 8-, 16-, and 32-bit accesses.

A separate address derives from an earlier scaled value minus 150, then multiplied by four. At that derived address plus 32, the function performs four word publications in order:

1. Numeric 32.
2. Numeric 47.
3. The zero-extended byte read from the derived address plus 4.
4. Numeric 32 again.

The byte read occurs after the first two word stores. It is not taken before the literal-write sequence.

Only after those computed literal writes does the function load a separate global word used as a mutation base. That load precedes the 32-bit write of 32 at absolute address 96.

The global-base load must not be moved to function entry or treated as immune to preceding computed writes. The word write at address 96 can likewise overlap other storage.

### Repeated publication region

A long contiguous region repeats a four-store publication pattern:

- A word store at a scaled computed base plus 84.
- A byte store at the separately loaded global base plus 25.
- A byte store at another computed base plus 17.
- A word store at the global base plus 52.

The initial group uses the earlier scale-derived value for its first destination and offset-52 publication. Subsequent groups use the value adjusted by subtracting 150 before scaling.

The global base is retained from its earlier load; the repeated groups do not establish repeated loads of its source global word. Nevertheless, their destination ranges may overlap one another.

The repetition is fixed in the supplied straight-line representation. Its numerical count is not specified by this contract and remains **documented_only** coverage. Replacing the region with one final-state publication is not justified for write-observing memory or alias-sensitive execution.

### Late absolute reads and byte publication

After the repeated region, the following ordering is established:

- A **32-bit word is read at absolute address 169**.
- A computed word at offset 116 is read.
- A **byte is written at absolute address 58**, using the low byte of eight times the adjusted value plus 48.
- A computed word at offset 20 is read and retained for final dispatch.
- A computed word at offset 4 is read and retained for later publication.

Thus, the address-169 read precedes the address-58 write and the subsequent retained-word reads. Aliasing with prior computed writes can affect the address-169 value; aliasing from the address-58 write can affect later reads.

Further computed byte and word writes precede publication of the low byte of the retained address-169 word at **absolute address 103**.

### Address 103 is reread, not merely reused

The byte publication at address 103 is followed by three distinct byte reads from that address:

| Read | Destination of the read value | Relevant intervening activity |
|---|---|---|
| First | Byte write at absolute address 33 | A computed 32-bit read at offset 68 occurs after the address-103 publication and before this read. |
| Second | First later byte write at a computed base plus 9 | Computed byte and word writes, a variable-shift byte write, and computed word reads intervene after the address-33 publication. |
| Third | Second byte write at that same represented computed base plus 9 | A computed byte write, a word publication of the retained address-169 value, and a computed word read at offset 84 intervene after the second read. |

All three reads are 8-bit accesses. They are not guaranteed reuse of the byte originally published at address 103.

In particular, intervening computed stores may overlap address 103 and change what the second or third read observes. The address-33 write can also overlap later computed accesses. Equality of the three loaded bytes requires suitable memory and aliasing conditions; it is not an unconditional invariant.

### Final mutations and dispatch

The late computed mutation region also includes:

- An 8-bit store of 108.
- A word publication that can cover a preceding byte publication at offset 16.
- A signed right shift whose count is restricted to the low eight bits of an address-derived value.
- A 16-bit store of 101 at twice a loaded word value.
- A 32-bit store of 100 at a loaded address plus 84.

Large variable-shift counts and negative shifted operands have target-dependent semantics that remain **documented_only**.

Near completion, a word loaded through an offset-4 field supplies the base for separate offset-100 and offset-116 reads. Another offset-100 read supplies a destination base for a word store at offset 84.

The last direct memory operation stores that destination-base value as a 32-bit word at twice the value obtained from the late offset-100 read through the offset-4-derived base.

The final unresolved target then receives:

1. The 32-bit value 114.
2. The late offset-100 value obtained through the offset-4-derived base.
3. The word retained from the post-repetition offset-20 read.
4. The late offset-116 value obtained through the offset-4-derived base.

These arguments use retained read results, not implied reloads at dispatch time. No post-dispatch cleanup is visible.

## Side effects

Direct effects include mixed-width writes, loaded-address traversal, low-address reads and writes, repeated publications, and final indirect dispatch.

Meaningful absolute accesses include byte reads at address 5; word writes at addresses 84 and 96; byte writes at addresses 1, 21, 33, 58, and 103; a word read at address 169; and the three later byte reads at address 103. Their ordering is specified above rather than inferred from a list of final values.

Endianness is unresolved. Consequently, the byte-level effects of overlapping word and halfword stores remain conditional. Global storage views may overlap; no global-independence assumption is permitted.

No locking, rollback, memory barrier, transaction boundary, or error recovery is visible. Atomicity, concurrency safety, reentrancy, and interrupt safety are **documented_only**.

A faulting access or nonreturning dependency may leave earlier effects committed. No compensating action is established.

## Dependencies

F-ebb851b304ef is the only resolved opaque function ID supplied.

No opaque IDs are resolved for its indirect dependencies. This contract therefore does not invent dependency identities or expose private source identifiers.

Visible consumer requirements include primary 32-bit results, represented 64-bit results, secondary result-register values, and the final four-argument dispatch. Actual target signatures, resource ownership, global preservation, exceptions, and nonreturning behavior remain **documented_only**.

## Acceptance examples

These are conditional static cases, not executed tests.

### Initial forwarding and truncation

For second argument 3, the first invocation receives 3 and 192 in its second and third positions. The first argument is unchanged.

For second argument 0x04000000, the third position is zero under truncating 32-bit shift semantics.

### Early zero survives overlapping zero stores

Assume ordinary valid 32-bit-addressed memory and no external mutation. Even if small-base offset 20 overlaps packed-base offset 20, the early word read from packed-base offset 20 returns zero.

The overlapping store contains only zero bytes, and the packed-base offset-52 word store is disjoint. A nonzero result is not explained by this overlap alone. Device effects or external mutation require separate qualification.

### Small-base bounds

Under 32-bit bit-vector semantics, the small base is a multiple of eight from 0 through 1016. Its offset-20 word destination starts between addresses 20 and 1036 inclusive.

### Unsigned byte publication

A loaded byte of 255 is published as word value 255, not as a sign-extended negative value.

### Global reads observe preceding writes

If preceding computed writes alter address 5, the later byte read there observes the resulting memory state. Likewise, the separate global mutation base is acquired after the computed literal-write sequence, not from an assumed entry snapshot.

### Address-103 rereads may differ

Suppose address 103 initially receives byte 0x12. If an intervening computed write changes it to 0x34 before the second read, the first later computed offset-9 byte store uses 0x34.

If another intervening write changes address 103 to 0x56 before the third read, the second such byte store uses 0x56. Reusing 0x12 for both stores would violate the represented read behavior.

### Repeated writes remain observable

A write-observing memory model must retain the ordered word publications 32, 47, the loaded byte value, and 32. The final value alone is insufficient.

### Dispatch boundary

If all earlier operations succeed, the final direct word store precedes dispatch with arguments 114 and the three retained values described above. Normal completion additionally requires dispatch to return.

## Uncertainties

The following remain **documented_only**:

- Application purpose and input meanings.
- Dependency identities, complete interfaces, and full call-trace coverage.
- Secondary result-register production.
- Signed overflow, alignment, endianness, and variable-shift behavior.
- Required address mappings and initial global contents.
- Complete alias relationships and device-memory behavior.
- Numerical repetition-count coverage.
- Concurrent or external mutation.
- Ownership and lifetime effects of dependencies.
- Fault propagation and final-target return behavior.

The early ordinary-memory zero result is not unresolved merely because zero stores can overlap it. Later nonzero publications and reloaded globals remain genuinely alias-sensitive.

## Evidence

This contract is based solely on the supplied static representation and opaque-ID map. Prior prose and reviewer feedback are not independent evidence.

Established evidence includes argument forwarding, ordered indirect invocations, represented widths, mixed-width memory accesses, the early zero-preserving sequence, named-global access ordering, distinct address-103 rereads, retained dispatch values, and the represented return.

No dependency bodies, platform memory model, executable trace, or independent implementation access were available. No runtime tests have been performed.
