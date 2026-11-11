<!-- Trusted provenance: F-96dfcadd24d7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-96dfcadd24d7

## Purpose

F-96dfcadd24d7 has no established application-level purpose. The supplied static representation describes a context-dependent sequence of memory accesses, address calculations, conditional writes, a memory traversal, and up to four indirect calls. Every represented terminating path ends at an invalid-instruction boundary rather than an ordinary return.

This contract specifies the represented behavior without interpreting it as successful initialization, parsing, allocation, or another application operation.

Three evidence categories apply:

- **Established static evidence:** access widths, calculations, conditions, and ordering explicitly represented.
- **Inferred intent:** possible interpretations that are not established requirements.
- **Unknown / documented_only:** behavior requiring missing instruction, platform, calling-convention, or dependency information.

Warnings about invalid instruction data, omitted unreachable regions, and overlapping global symbols limit confidence in the representation. The contract does not establish that every represented path is executable.

## Interface

### Invocation and result

The displayed interface declares no explicit parameters or result. Its body nevertheless consumes incoming register values and preexisting stack contents. A conventional zero-argument calling convention is not established.

The required incoming context includes:

- A register holding a context address.
- A register holding an address used for eight consecutive 32-bit output stores.
- Two additional incoming register values copied into that output region.
- An incoming link value used to calculate the first indirect-call target.
- Preexisting stack words at displayed frame offsets `0x1e8` and `0x264`.
- A stack mapping that permits the represented stack-address calculations and accesses.

No ordinary return value, success status, error code, or recovery convention is shown. Execution may encounter an invalid-instruction boundary, remain in the memory traversal, or fail earlier during a memory access or indirect transfer. The actual machine response to each failure is unknown.

### Widths and layout

The representation uses 32-bit addresses and words, 16-bit stores, and 8-bit accesses. It includes signed 32-bit comparisons, arithmetic right shifts, logical right shifts, and signed-overflow detection.

Endianness, alignment enforcement, executable-address conventions, and machine treatment of questionable signed arithmetic are not established.

The incoming context is read as follows:

| Context offset | Access | Role established by use |
|---|---|---|
| `+0x08` | 32-bit read | Primary-record address |
| `+0x10` | 32-bit read | Payload later stored elsewhere |

The primary record has these represented accesses:

| Record offset | Access | Role established by use |
|---|---|---|
| `+0x00` | Conditional 32-bit read | Address of another 32-bit value |
| `+0x04` | 32-bit read | Candidate address |
| `+0x08` | 32-bit read | Selector, subsequently usable as an auxiliary word |
| `+0x0c` | 32-bit read | Associated word |
| `+0x2a` | 16-bit write | Character-sized conversion of the record address’s upper byte |

These accesses do not establish an object type or ownership model.

Stack offsets in this contract identify locations in the displayed frame. They are not a verified public frame layout. Several stack addresses are used as integers or exported pointers, not merely as locations for ordinary stack accesses.

### Address-derived regions and absolute accesses

One derived base is the signed 32-bit record address arithmetically shifted right by 24 bits. Its signed value is therefore between −128 and 127. Accesses relative to it can address low memory or the high end of a 32-bit address space.

A second derived base is the signed representation of the stack address at frame offset `0x324`, arithmetically shifted right by 20 bits.

The explicit absolute accesses are distinct:

| Address | Represented access |
|---|---|
| `0x04` | Initial 32-bit traversal read when its accumulator starts at zero |
| `0x22` | 8-bit flag read |
| `0x3c` | 32-bit store |
| `0x5a` | 16-bit store |
| `0x7e` | Conditional 32-bit store |
| `0xa9` | 32-bit read |
| `0x440000` | Post-call 8-bit store |

The representation does **not** establish direct stores to `0x22` or `0xa9`. Storing the numeric value `0x22` into the output region is not a write through address `0x22`. Possible aliasing and unknown dependency effects do not change the direction of these explicit accesses.

## Behavior

### Initial context reads and record-derived writes

The routine first reads the record address from context offset `0x08`, then the payload from context offset `0x10`.

It derives:

- A signed base by arithmetically shifting the record address right by 24 bits.
- An 8-bit character-sized value from the record address’s upper byte.

The following stores occur in order:

1. A 16-bit conversion of that character-sized value is written at record offset `0x2a`.
2. Five 32-bit values are written relative to the signed derived base:

| Offset | Stored value |
|---|---|
| `+0x00` | Derived base |
| `+0x04` | Record address |
| `+0x08` | Stack address at frame offset `0x324` |
| `+0x0c` | Signed record address arithmetically shifted right by eight bits |
| `+0x10` | Previously read context payload |

The character conversion’s signedness is unresolved. For upper-byte values `0x80` through `0xff`, the halfword may differ depending on whether the character-sized value is sign-extended or zero-extended.

Only after these stores does the routine read record offsets `+0x04`, `+0x08`, and `+0x0c`, in that order. Earlier stores can affect these reads if the addressed regions overlap.

The three resulting values serve as the candidate address, selector or auxiliary word, and associated word.

### Zero-selector adjustment

If the selector is zero:

1. The signed record-derived base is added to the candidate address.
2. The 32-bit value at record offset `+0x00` is read as an address.
3. A 32-bit value is read through that address.
4. The associated word is replaced by that indirect value minus the signed record-derived base.
5. The auxiliary word is replaced by the adjusted candidate’s unsigned 32-bit representation logically shifted right by 16 bits.

The subtraction is represented using signed 32-bit operands before its result is held as a word. Its overflow behavior is not independently established.

There is no represented null or bounds check before the indirect read. In particular, an adjusted candidate of zero does not bypass that read: the candidate-null branch occurs later.

If the selector is nonzero, this adjustment is skipped.

### Stack-derived setup and candidate replacement

The character-sized upper-byte value is written as one byte at derived-base offset `+0x0d`. This overlaps the earlier word store spanning offsets `+0x0c` through `+0x0f`. Which portion of the earlier word changes depends on endianness.

The routine then:

1. Arithmetically shifts the signed stack address at frame offset `0x324` right by 20 bits to obtain the stack-derived base.
2. Selects a secondary pointer four bytes below that base.
3. Stores the low byte of the unshifted stack address at **stack-derived-base offset `+0x09`**.
4. Selects the stack address at frame offset `0x248` as a separate address operand.

The byte store is at the stack-derived base plus nine, equivalently the initially selected secondary pointer plus thirteen. It is not at secondary-pointer offset `+0x09`.

If the current candidate is zero, the routine additionally:

- Replaces the secondary pointer with the stack address at frame offset `0x244`.
- Overwrites the byte at record-derived-base offset `+0x0d` with the low byte of a static datum.
- Multiplies the record-derived base by 16.
- Replaces the separate address operand with the arithmetic-right-shift-by-eight result of combining the incoming stack word at `0x264` with negative sixteen times the stack-derived base.
- Forms a replacement candidate by adding the scaled record-derived base to the stack address at `0x244`.
- Replaces the associated word with the replacement candidate logically shifted right by 20 bits.
- Replaces the auxiliary word with the incoming stack word at `0x1e8`.

The signed multiply, signed addition, address arithmetic, and subsequent shift require platform interpretation where their mathematical results exceed their represented widths.

### Candidate high-nibble gate

The candidate is interpreted as signed 32 bits, arithmetically shifted right by 28 bits, and compared with zero.

The gate passes exactly when the candidate’s highest four bits are not all zero. It is not merely a negative-address test: `0x10000000` through `0x7fffffff` also pass.

If the gate fails, the next represented event is an invalid-instruction boundary.

If it passes, the low 16 bits of the auxiliary word are stored at secondary-pointer offset `+0x24`. The incoming stack word at frame offset `0x264` then selects the next path.

### Zero control-word path

When that control word is zero, one zero byte is stored at an address formed from:

- The stack address at frame offset `0x254`; plus
- The signed 32-bit interpretation of the stack address at frame offset `0x210`.

This uses a stack address as an integer byte index. It is not established to address an in-frame buffer.

An invalid-instruction boundary follows. The output-array writes, flag read, traversal, and indirect calls are not reached on this represented path.

### Nonzero control-word path

When the control word is nonzero, the routine combines, by bitwise OR:

- The current record-derived quantity shifted left by eight bits.
- That quantity multiplied by 16.

“Current” includes the multiplication already performed on the candidate-null path.

The routine next tests whether the constructed stack address at frame offset `0x210` equals zero. If it does, an invalid-instruction boundary occurs before the output-array stores or the store at `0x5a`. Whether this condition can occur is unknown.

If the combined word is nonnegative as a signed 32-bit value, eight 32-bit stores occur through the incoming output address, in increasing word-index order:

| Index | Value |
|---|---|
| 0 | Stack address at frame offset `0x248` |
| 1 | `0x22` |
| 2 | Stack address at frame offset `0x254` |
| 3 | Associated word bitwise-ORed with the selected separate address operand |
| 4 | First additional incoming register value |
| 5 | Output address itself |
| 6 | Second additional incoming register value |
| 7 | `0x0860b3b4` |

No capacity check is represented. If the combined word is negative, these eight stores are skipped.

In either case, the routine then:

1. Stores the combined word’s low 16 bits at absolute address `0x5a`.
2. Assigns the stack address at frame offset `0x160` to a displayed pointer slot at `0x70`.
3. Reads the flag byte at absolute address `0x22`.

A nonzero flag leads to an invalid-instruction boundary. It does not trigger an explicit flag write.

If the flag is zero, the traversal accumulator is initialized to zero. The displayed pointer slot is tested against zero. If that test succeeds, the routine:

- Stores the signed stack address at frame offset `0x2c2`, arithmetically shifted right by eight bits, as a 32-bit value at `0x7e`.
- Assigns zero to the displayed pointer slot.
- Sets the accumulator to `0x280`.
- Assigns zero to the displayed stack slot at `0x74`.

The reachability of this stack-address-null condition is unresolved.

### Traversal and dependency chain

The traversal executes at least once. Each iteration reads a 32-bit word from the current accumulator plus four and replaces the accumulator with that word.

Iteration continues exactly when adding `0x7e` to the newly read signed 32-bit word would overflow signed 32-bit arithmetic. The tested addition does not update the accumulator.

Thus:

- With accumulator zero, the first read is at `0x04`.
- With accumulator `0x280`, the first read is at `0x284`.
- Continuation occurs for newly read bit patterns `0x7fffff82` through `0x7fffffff`.
- No iteration bound or cycle detection is represented.

After traversal terminates:

1. A 32-bit value is read at `0xa9`.
2. That value is stored as 32 bits at `0x3c`.
3. A static 32-bit word is read and shifted left by four bits.
4. Four indirect dependencies are called in sequence.

The first call receives the shifted word twice, followed by `0x220000` and `0x88`. Each of the first three call results is passed to the next call. The fourth result is unused.

If all calls return to the represented continuation, three byte stores follow:

1. One at `0x440000`.
2. One at `0x220000` plus the shifted word arithmetically shifted right by eight bits.
3. Another at that same calculated address.

The stored byte is obtained by logically shifting the shifted word right by eight bits and retaining its low byte.

The duplicate destination stores are separately represented. They must not be described as a single write; device-backed memory could distinguish them.

An invalid-instruction boundary follows.

## Side effects

The routine can modify the record, derived-address regions, displayed stack state, the incoming output region, and the explicitly identified absolute destinations. No rollback is represented.

The direct low-address stores are to `0x5a`, conditionally `0x7e`, and `0x3c`. The direct accesses to `0x22` and `0xa9` are reads. Other computed destinations may alias these locations, but that possibility is not evidence of an additional explicit store to either address.

The word read at `0xa9` and conditional word store at `0x7e` are not conventionally four-byte aligned. Their success or fault behavior depends on the platform.

Earlier writes may alter later memory reads through aliasing. The representation also uses displayed stack storage for the auxiliary word and pointer state. Its physical correspondence to machine accesses, and the consequences of overlapping writes to such storage, remain unverified.

No explicit allocation, release, reference-count operation, or ownership transfer is shown. Pointer stores do not establish ownership transfer. Exported stack addresses have no established lifetime beyond the current execution context.

Dependencies may modify globals or other memory, retain arguments, or affect ownership. None of those behaviors is constrained by the available evidence.

## Dependencies

The following opaque IDs identify call sites, not independently verified implementations:

| Dependency | Represented target | Represented interface |
|---|---|---|
| F-unknown-01 | Selected separate address operand plus incoming link value plus `0xffe` | Four word-like arguments; word-like result |
| F-unknown-02 | `0x0860c814` | Receives the first result; produces the second |
| F-unknown-03 | `0x0860c8e7` | Receives the second result; produces the third |
| F-unknown-04 | `0x0860c937` | Receives the third result; result unused |

The fixed targets are the results of the represented target calculations. Instruction-set selection, alignment normalization, and executable mapping are unknown.

No dependency is established to return normally, preserve globals, avoid memory writes, or acquire or release memory. The post-call stores are conditional on normal continuation through all four calls.

## Acceptance examples

These are conditional consequences of the static representation, not executed tests. They assume preceding accesses succeed and relevant values are not changed by aliasing.

1. **Nonzero selector and low candidate.**  
   With a nonzero selector and candidate `0x0fffffff`, the selector adjustment is skipped. Initial writes and stack-derived setup still occur. The high-nibble gate fails, leading to an invalid-instruction boundary without the auxiliary halfword store.

2. **Positive candidate passing the gate.**  
   Candidate `0x10000000` passes the high-nibble gate despite being nonnegative. The auxiliary halfword is stored before control-word branching.

3. **Zero selector with a null adjusted candidate.**  
   The indirect read through the address obtained from record offset `+0x00` still precedes candidate-null handling. An invalid indirect address can prevent the fallback path from being reached.

4. **Stack-derived byte placement.**  
   If the stack-derived base is `0x100`, the secondary pointer initially equals `0xfc`, but the byte destination is `0x109`, not `0x105`.

5. **Zero control word after passing the gate.**  
   The auxiliary halfword store precedes the stack-address-indexed zero-byte store. The indirect-call chain is not reached.

6. **Nonzero flag at `0x22`.**  
   On the nonzero control-word path, the store at `0x5a` precedes the flag read. A nonzero flag prevents traversal and calls. The flag read itself does not write `0x22`; the output table may merely contain that numeric address value.

7. **Traversal threshold.**  
   A newly read word of `0x7fffff81` ends traversal because adding `0x7e` reaches `0x7fffffff` without overflow. A word of `0x7fffff82` causes another iteration, whose next read address is `0x7fffff86`.

8. **Copy direction and completed calls.**  
   After traversal, the word at `0xa9` is read and copied to `0x3c`; no reciprocal write to `0xa9` is shown. If all dependencies return and the final stores succeed, both consecutive stores to the repeated calculated destination occur.

## Uncertainties

The following remain **documented_only**:

- Authentic entry point, calling convention, frame layout, and executable path recovery.
- Signedness of the character-sized conversion and resulting halfword extension.
- Machine semantics for signed overflow, negative left shifts, and address wraparound.
- Endianness, alignment handling, permissions, and device-memory behavior.
- Static datum values and the access width used to obtain the datum converted to a byte on the candidate-null path.
- Reachability of conditions comparing constructed stack addresses with zero.
- Exact physical realization of displayed stack temporaries and effects of overlapping storage.
- Dependency signatures, preservation rules, memory effects, ownership behavior, and return behavior.
- Whether invalid-instruction boundaries represent faults, intentional traps, or representation artifacts.
- Any application-level purpose or success convention.

## Evidence

This contract is based solely on the supplied static representation. It distinguishes explicit access directions, widths, ordering, conditions, and call-result flow from unresolved platform behavior.

The supplied warnings identify invalid instruction data, omitted unreachable regions, and overlapping global symbols. These prevent treating the representation as a verified executable specification.

No implementation access, dependency inspection, instruction-level validation, or runtime tests have been performed. All acceptance examples are conditional static-contract examples.
