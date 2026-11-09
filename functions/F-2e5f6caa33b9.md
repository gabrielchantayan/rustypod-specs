<!-- Trusted provenance: F-2e5f6caa33b9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2e5f6caa33b9

## Purpose

F-2e5f6caa33b9 performs an ordered sequence of raw memory reads and writes, then invokes an unresolved indirect destination with four arguments. Its visible work includes byte-field updates, full-word transfers, temporary stores that are later overwritten, and two stores to addresses obtained by doubling loaded values.

The supplied static evidence does not establish an application-level purpose. In particular, it does not justify interpreting the touched storage as a list, object hierarchy, allocator structure, or validated message.

The contract is therefore an ordered memory-effect contract. Intermediate stores matter: later reads may observe them through aliasing, and a memory fault may leave only a prefix of the effects completed.

## Interface

### Explicit inputs

The recovered interface exposes four arguments:

| Position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No visible use |
| Second | 8 bits | Written to the global byte at address `0x3A` |
| Third | 32 bits | No visible use |
| Fourth | 8 bits | Written to two byte locations during the memory-update sequence |

The unused arguments are not validated or dereferenced by the visible body. Their broader calling-convention significance is unknown.

### Additional entry state

The function also consumes three machine-state values that are not represented as explicit parameters. This document calls them:

- **Primary base**: a value represented as an unsigned 32-bit quantity, used as a memory base, as stored word data, as a low-byte value, and as the source of a shift count.
- **Secondary base**: an integer-valued memory base, also used as stored word data and low-byte data.
- **Context base**: an integer-valued memory base, also stored as a word.

These names describe access roles only. They do not establish types, object boundaries, ownership, or distinctness.

For normal completion, the executed reads and writes must be valid in the target environment. The function performs no visible null, bounds, alignment, permission, or address-overflow checks.

### Memory widths and notation

The visible operations distinguish:

- **Byte accesses:** 8 bits.
- **Halfword accesses:** 16 bits.
- **Word accesses:** 32 bits.

Offsets below are byte offsets and are written in hexadecimal. “Low byte” means the least significant eight bits of a value, not necessarily the byte at its lowest memory address.

The global locations relevant to this body are:

| Global address | Visible access width | Use |
|---|---:|---|
| `0xA9` | 32 bits | Initial captured value |
| `0x3A` | 8 bits | Second explicit argument |
| `0x67` | 8 bits | Low byte of the captured global word; subsequently reread |
| `0x21` | 8 bits | Copy of the byte read from `0x67` |

The evidence warns that global symbols overlap smaller symbols. No non-aliasing guarantee exists between these globals and addresses derived from the three bases.

### Output and return

There is no explicit result value in the recovered interface.

The final indirect invocation receives four represented 32-bit arguments. Its return value, if any, is not visibly consumed. If that invocation returns normally, the enclosing function returns. Whether it actually returns, throws, traps, or transfers control elsewhere is unknown.

## Behavior

### Entry snapshots and initial updates

Before making any visible store, the function reads, in order:

1. The word at global address `0xA9`.
2. The word at context base plus `0x14`.
3. The word at primary base plus `0x04`.

These are captured values. Later changes to their original storage do not, by themselves, change the captured values.

The function then writes the second explicit argument to global byte `0x3A`. It writes the low byte of the primary base to context offset `0x0D`, byte value `0x6C` to primary offset `0x08`, and the low byte of the secondary base to primary offsets `0x0D` and `0x09`, in that order.

Next, it writes the context base as a word to context offset `0x54`, then writes the low byte of the captured global word to global byte `0x67`.

These writes are not deferred initialization. They precede the following reads and can affect them if the addressed storage overlaps.

### Initial context-word propagation

The function reads a word from context offset `0x44`. It then reads global byte `0x67` and writes that byte to global address `0x21`.

Using the captured context-offset-`0x44` word, it writes:

- Its low byte to primary offset `0x10`.
- The complete word to secondary offset `0x54`.
- A byte derived by right-shifting that word to context offset `0x01`.

The shift count is the low eight bits of the primary base. The shifted value is represented as a signed integer in the supplied reconstruction. For counts and operand values whose meaning depends on the target’s shift instruction or signed-shift rules, the exact result is not established; see **Uncertainties**.

After the byte shift-result store, the function writes the primary base as a word to secondary offset `0x54`. Thus, on non-overlapping storage, the preceding word at that same location is replaced before the next read. Both writes remain part of the observable sequence.

### Primary and secondary field transfers

The function next reads a word from primary offset `0x40`.

It writes the earlier captured primary-offset-`0x04` word to secondary offset `0x74`, then writes the newly read primary-offset-`0x40` word to primary offset `0x10`. The latter is a full-word write, not another byte update. On ordinary non-overlapping storage, it replaces the earlier low-byte store at that address and also writes the following three bytes.

It then writes the context base as a word to secondary offset `0x44`.

After those word stores, the function:

- Reads global byte `0x67` and writes it to primary offset `0x09`.
- Writes the low byte of the primary base to context offset `0x09`.
- Writes the initially captured global word to primary offset `0x44`.

The global byte is read at the time of use. It must not be assumed still to contain its earlier assigned value if intervening writes can alias it.

### Linked-base selection and updates

The function reads a word from primary offset `0x54` and uses that captured value as a further address base, called the **linked base** here.

It then rereads global byte `0x67` and writes that byte to primary offset `0x09`, even though the same destination was updated from the same global earlier. This second read and write occur after the linked-base read and must not be collapsed into the earlier pair.

Next, in order, it:

- Writes the fourth explicit argument to linked offset `0x0D`.
- Restores the initially captured context-offset-`0x14` word to context offset `0x54`.
- Reads a word from primary offset `0x14`.
- Writes the fourth explicit argument to context offset `0x11`.
- Writes the low byte of the linked base to primary offset `0x09`.
- Writes the linked base as a word to primary offset `0x44`.

“Restores” here describes the destination receiving a previously captured value; it does not establish that the destination originally contained that value.

The linked base is a snapshot of the earlier primary-offset-`0x54` read. Subsequent changes to that field do not replace the captured base.

### Doubled-address stores and dispatch preparation

The function reads a word from context offset `0x04`. This captured value becomes the **dispatch base**.

Its next operation is a 16-bit store of numeric value `0x0065` to the address obtained by doubling the dispatch base. This is an absolute derived address, not dispatch base plus a field offset. Only two bytes are written.

After that store, the function reads a word from dispatch offset `0x64`. This is the **dispatch value**. Because the read follows the halfword write, overlap between the doubled address and this field can affect the value read.

The function then writes the low byte of the previously captured primary-offset-`0x14` word to linked offset `0x11`, followed by the 32-bit numeric value `100` (`0x00000064`) to linked offset `0x54`.

Next it reads:

1. A word from dispatch offset `0x74`, retained as the fourth indirect-call argument.
2. A word from context offset `0x64`, used as the **relay base**.

These reads occur after the linked-field writes, so those writes may influence either read if storage overlaps.

Finally, before invoking the indirect destination, the function:

- Writes the low byte of the dispatch base to context offset `0x09`.
- Writes the dispatch value as a word to relay offset `0x54`.
- Writes the relay base as a word to the address obtained by doubling the dispatch value.

The second doubled-address store is 32 bits wide, unlike the earlier 16-bit store.

### Final indirect invocation

The indirect destination is invoked after all preceding visible memory operations, with these arguments in order:

| Position | Value |
|---|---|
| First | Numeric value `0x72` |
| Second | Captured dispatch value from dispatch offset `0x64` |
| Third | Initially captured context-offset-`0x14` word |
| Fourth | Captured dispatch-offset-`0x74` word |

The dispatch value remains the captured value even if a later store changes its original memory location.

No target-selection expression is recovered. The evidence supports neither a target-address calculation nor a finite set of possible destinations.

## Side effects

### Memory changes

All identified writes are direct effects of this function. No transaction, rollback, synchronization, or atomicity mechanism is visible.

Several destinations are written more than once, with different widths or values:

- Primary offset `0x09` receives the secondary-base low byte, two separately read global bytes, and later the linked-base low byte.
- Primary offset `0x10` receives a byte and later a word.
- Primary offset `0x44` receives the captured global word and later the linked base.
- Secondary offset `0x54` receives the captured context-field word and then the primary base.
- Context offset `0x54` receives the context base and later the captured context-offset-`0x14` word.
- Context offset `0x09` receives the primary-base low byte and later the dispatch-base low byte.

These are execution-order facts, not unconditional final-state guarantees. Cross-base aliasing, overlapping accesses, and effects of the indirect destination can change the eventual contents.

Byte order is not established. Numeric halfword and word writes therefore do not determine a portable byte-by-byte layout.

### Ownership and lifetime

The body contains no visible allocation, deallocation, reference-count operation, or explicit ownership transfer.

Storing an address-like value does not establish that ownership moves with it. The indirect destination’s ownership and lifetime effects are unknown, including whether it releases storage, retains arguments, or invalidates previously used regions.

### Errors and interruption

No explicit status code, error branch, or recovery path appears.

An invalid or unsupported access can prevent later operations from occurring under the target environment’s fault rules. Earlier stores may already have taken effect. The exact exception mechanism, alignment behavior, and fault atomicity are not established.

## Dependencies

| Opaque dependency ID | Established interaction | Unknown behavior |
|---|---|---|
| F-INDIRECT-UNKNOWN-01 | One final indirect invocation with the four arguments specified above | Actual identity, target selection, calling convention details, memory effects, global preservation, ownership effects, failure behavior, and return behavior |

F-INDIRECT-UNKNOWN-01 is a documentation identifier for the unresolved destination, not an identified implementation.

No other call is visible. Direct global and memory accesses are not evidence of additional callable dependencies.

## Acceptance examples

These examples are conditional consequences of the static evidence, not executed tests.

### Captured values survive later source changes

Assume the initial word at context offset `0x14` is `0x11223344`, accesses complete, and execution reaches the indirect invocation. The third invocation argument is `0x11223344`, even if intervening stores change that original field.

The same captured word is written to context offset `0x54` before the later dispatch-related reads.

### Word replacement of an earlier byte write

Assume little-endian memory, no relevant aliasing, and primary offset `0x40` contains `0xA1B2C3D4`. After the full-word write to primary offset `0x10`, the four bytes beginning there are `D4 C3 B2 A1`.

The earlier byte written at primary offset `0x10` does not survive that word write. No equivalent byte-order claim is established without the little-endian condition.

### Distinct widths at doubled addresses

Assume doubling the dispatch base produces valid address `0x2000`, with no arithmetic ambiguity or interfering overlap. The first doubled-address operation stores a 16-bit numeric `0x0065` at that address.

Assume the captured dispatch value is `0x3000`. The later doubled-address operation stores the 32-bit relay-base value at address `0x6000`.

Neither operation is a store at an offset of two bytes from its originating value.

### Shift with an unambiguous count

Assume the captured context-offset-`0x44` word is `0x12345678`, the primary-base low byte is `4`, and the relevant memory operations succeed. The shift-derived byte written to context offset `0x01` is `0x67`.

This example uses a nonnegative operand and a count below the word width. It does not establish behavior for oversized counts or negative signed operands.

### Repeated global-byte reads remain distinct

Assume an intervening store aliases global address `0x67` and changes it between the two reads used for primary offset `0x09`. The two stores to primary offset `0x09` use their respective observed byte values; the earlier byte must not be substituted for the later read.

### Faults do not imply rollback

Assume the first doubled-address halfword store faults. No later dispatch-value read, linked-offset-`0x11` write, relay-field write, or indirect invocation is established to occur. Earlier completed writes are not rolled back by any visible mechanism.

## Uncertainties

The following are **documented_only**:

- **Actual ABI:** how the three implicit bases arrive, register preservation obligations, and whether the recovered explicit signature is complete.
- **Pointer model:** address width, signed address interpretation, conversion rules, and wrapping behavior.
- **Arithmetic overflow:** the two doubled-address expressions use integer-represented values. The evidence does not establish whether overflowing multiplication wraps, traps, or has another target-specific interpretation.
- **Shift semantics:** arithmetic versus logical behavior for negative operands and handling of counts from `32` through `255`.
- **Memory layout:** object boundaries, intended field types, packing, alignment, byte order, and legal alias relationships.
- **Access mechanics:** whether reconstructed loads and stores correspond to single machine accesses, and whether memory is ordinary, volatile, shared, or device-backed.
- **Indirect control flow:** destination identity, selection mechanism, and all behavior inside the destination.
- **Concurrency and external mutation:** no synchronization or stable-global guarantee is established.
- **Semantic purpose:** any interpretation as initialization, relinking, dispatch setup, or deliberate encoding remains inferred intent rather than an established contract.

These unknowns must not be resolved by assuming that unfamiliar operations are harmless or that the indirect destination preserves globals or memory ownership.

## Evidence

**Established static evidence:** the supplied reconstructed body supports the access widths, relative offsets, captured-value lifetimes, ordering, explicit argument usage, numeric constants, doubled-address stores, and final invocation arguments described above.

**Inferred intent:** the body appears to propagate and replace address-like values among several memory regions before transferring control. This does not establish a data structure or protocol.

**Evidence limits:** the reconstruction does not recover the final destination or fully specify machine semantics. No implementation access, runtime execution, or runtime tests were used. The acceptance examples are conditional static expectations, not observed outcomes.
