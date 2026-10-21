<!-- Trusted provenance: F-801a4d58dfbe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-801a4d58dfbe

## Purpose

F-801a4d58dfbe performs an initial bit-setting assignment, invokes three dependencies in a fixed order, and then performs a bit-clearing assignment using a base retained at entry.

Under a stable-global assumption, the two assignments affect the same word at byte offset 12 from the entry base. Without that assumption, the initial assignment is not established as a same-word read-modify-write: its source address and destination address contain separate references to the global base.

The final assignment uses the retained entry base for both its read and its write. A dependency changing the global base therefore does not, by itself, redirect that final assignment.

This is not restoration of the entry word or its original least significant bit. On normal completion, the final assignment clears bit 0 of the word selected through the retained base. In the represented 32-bit word model, it preserves bits 1 through 31 from the final read, not from the entry state.

An active-state marker surrounding a two-phase operation is a possible interpretation. That is inferred intent, not an established meaning of the field or dependencies.

## Interface

**Explicit inputs:** None.

**Implicit inputs:**

- A global slot supplying a base value.
- The unsigned word read at byte offset 12 through a reference to that global base during the initial assignment.
- The unsigned word read at byte offset 12 through the retained entry base during the final assignment.
- Any state consulted or modified by the dependencies.

**Explicit output:** None. The function returns without a result.

**Observable effects:** Two word assignments and three ordered dependency invocations, plus any effects produced by those dependencies.

### Widths and layout

The target field is accessed as an unsigned word. The initial operation sets the least significant value bit using the numerical mask 1. The final operation applies the numerical mask `0xFFFFFFFE`.

For a 32-bit unsigned word:

- The accessed field occupies bytes 12 through 15 relative to the selected base.
- The initial transformation preserves bits 1 through 31 and sets bit 0.
- The final transformation preserves bits 1 through 31 and clears bit 0.

The supplied representation does not independently establish the definition of the unsigned-word type, the target ABI, or the width of the integer used to retain the base. The 32-bit descriptions and examples are conditional on that word model. If the accessed unsigned word were wider than 32 bits, the final mask would not preserve value bits above bit 31.

The offset is 12 bytes, not 12 words. Byte order is unspecified; bit 0 identifies a value bit without identifying its byte address.

The retained base uses an integer representation. Pointer conversion, address width, and the rules governing addition of 12 are not independently specified. The function contains no explicit check for address overflow, null values, alignment, bounds, accessibility, or object lifetime.

The dependency arguments have the numerical values 82, 1, 0, and 50 in the combinations specified below. Their formal parameter types and ABI-level widths are unavailable.

## Behavior

### Entry capture and the three global-base references

There are three distinct source-level references to the global base:

1. The reference used to retain the entry base.
2. The reference in the address expression for the initial word read.
3. The reference in the destination-address expression for the initial word write.

The entry capture precedes the initial assignment in the represented execution sequence. It supplies the retained value used by the final assignment.

The initial assignment does **not** explicitly reuse the retained entry base. Nor does it explicitly retain a separate initial-assignment base for both its read and its write.

No evaluation order between the initial assignment’s destination-address expression and its right-hand-side expression is established here. In particular, the contract does not require the global reference associated with the word read to be evaluated before the global reference associated with the write destination.

These three source-level references do not establish three physical memory loads. Compiler behavior, access qualifiers, and platform rules needed to make such a claim are not supplied.

### Initial assignment

The initial assignment obtains an unsigned-word value from offset 12 using its read-side reference to the global base. It sets bit 0 in that value and stores the result at offset 12 using its write-side reference to the global base.

For the 32-bit model, the stored value is the read value combined with `0x00000001` by bitwise OR:

- Bit 0 is set in the stored result.
- Bits 1 through 31 come from the word that was read.

If the global base is stable across the relevant references, the read and write address the same word. Only under that condition is this established as a same-word read-modify-write.

If platform-defined behavior permits the two address expressions to obtain different base values, the assignment can instead read one word and write the transformed value to another. In that conditional case, the stored non-bit-0 portion comes from the read-side word, not necessarily from the previous contents of the destination word.

The source does not establish that asynchronous changes to the global are permitted, visible, or well-defined. Therefore, differing-address behavior is a conditional consequence, not a guaranteed concurrency feature or a specified outcome of an unsynchronized race.

The initial write precedes the first dependency invocation. No atomic read-modify-write operation, lock, or memory barrier is established.

### Dependency sequence

After the initial assignment, the normal execution path invokes:

1. **F-17d6434f9fa3**, with positional arguments **82, 1, 0**.
2. **F-233fe84cf225**, with argument **50**.
3. **F-17d6434f9fa3**, with positional arguments **82, 1, 1**.

Each invocation must return normally before execution reaches the next listed operation.

The caller does not inspect a dependency result or branch on a reported status. No retry, error conversion, or compensating action appears.

The two invocations of F-17d6434f9fa3 differ only in their third argument. Their meanings are unknown. The argument 50 supplied to F-233fe84cf225 does not establish a duration, time unit, wait operation, or blocking guarantee.

No preservation of the global base, target contents, accessibility, or object lifetime is assumed across these calls.

### Final assignment and return

After the third dependency invocation returns, the final assignment uses the retained entry base for both its source address and its destination address, each at byte offset 12.

It reads the word at that address, applies the mask `0xFFFFFFFE`, and writes the result back to that address.

Under the 32-bit word model:

- Bit 0 becomes clear, regardless of its entry value or its value immediately before the final read.
- Bits 1 through 31 retain the values obtained by the final read.

This fresh read allows changes to other bits made during the dependency sequence to survive the final assignment. The function does not reconstruct those bits from an entry snapshot.

The final assignment does not consult the then-current global base. Its addressing input remains the retained entry value even if a dependency changes the global slot.

The function then returns without an explicit result. No subsequent cleanup action is visible.

## Side effects

The function directly performs:

- An entry capture of the global base.
- An initial unsigned-word read and write whose address expressions independently reference the global base.
- Three dependency invocations.
- A final unsigned-word read and write addressed through the retained entry base.

With a stable global during the entry capture and initial assignment, and valid memory throughout execution, both assignments affect the same word. If stability is absent, that identity is not established.

The function does not explicitly assign to the global base slot or restore its previous value. A dependency’s change to that slot may remain after return.

There is no explicit allocation, deallocation, ownership transfer, reference acquisition, or reference release. No base address is passed as an explicit dependency argument. These facts do not establish that dependencies cannot reach, modify, invalidate, or release the affected storage through other state.

The initial bit set is not guaranteed to remain in effect throughout the calls. Dependencies may change that bit or overwrite the word.

The represented ordering does not establish hardware visibility, atomicity, protection from lost updates, or device-register semantics. A read followed by a write can overwrite an intervening modification unless an external concurrency contract prevents it.

The final clear is an ordinary operation following the calls, not an established exception-safe cleanup mechanism. If execution does not reach it, no direct undo of the initial assignment is established.

For normal completion, every reached memory access must remain valid under the platform’s rules. In particular, retaining a base value does not retain ownership or extend the target’s lifetime.

## Dependencies

| Opaque ID | Established invocation contract | Unestablished behavior |
|---|---|---|
| F-17d6434f9fa3 | Invoked first with `(82, 1, 0)` and third with `(82, 1, 1)`, after the initial assignment and before the final assignment. | Parameter meanings and widths, result convention, failure behavior, termination, global-state effects, ownership effects, and any relationship between the two invocations. |
| F-233fe84cf225 | Invoked once with `(50)`, between the two invocations of F-17d6434f9fa3. | Units, timing, blocking, result convention, failure behavior, termination, global-state effects, and ownership effects. |

All three invocations are unconditional along the visible normal path. No dependency bodies or independent dependency contracts were supplied.

## Acceptance examples

These are conditional deductions from static evidence, not runtime observations. Numerical word examples assume 32-bit unsigned accesses.

### Stable base, initially clear bit

Given a valid global base that remains stable throughout execution, an initial word of `0x000000A4`, and dependencies that leave the word unchanged:

- The initial assignment writes `0x000000A5` to that same word.
- The calls occur with arguments `(82, 1, 0)`, `(50)`, and `(82, 1, 1)`, in that order.
- The final assignment writes `0x000000A4`.

### Stable base, initially set bit

Given the same stability and validity conditions, an initial word of `0x000000A5`, and dependencies that leave it unchanged:

- The initial assignment writes `0x000000A5`.
- The final assignment writes `0x000000A4`.

Restoring the entry bit to 1 would not match the established final transformation.

### Changes to other bits survive

Given that the retained-base word contains `0x80000007` immediately before the final read:

- The final assignment writes `0x80000006`.

This result uses the final read, regardless of the word’s entry value.

### Global redirection during a dependency

Given that the global is stable at base A through the entry capture and initial assignment, and a dependency subsequently changes it to base B:

- The initial read and write address A plus 12.
- The final read and write also address A plus 12.
- No direct final clear at B plus 12 is established.

A must remain valid. Dependencies may independently affect either object.

### Different initial read and write bases

This example applies only if platform and concurrency rules permit defined, observable differences between the initial assignment’s global references.

Suppose the entry capture retains A, the initial read-side expression selects B, and the initial write-side expression selects C. If the word read from B plus 12 is `0x000000A4`:

- The initial assignment writes `0x000000A5` at C plus 12.
- It does not merely set bit 0 while preserving C’s prior other bits.
- The final assignment still targets A plus 12, assuming that access remains valid.

This example imposes no temporal order between the read-side and write-side global evaluations. It also does not assert that an ordinary unsynchronized execution can validly produce these selections.

### A dependency does not return normally

If the first invocation does not return normally, the later two invocations and final assignment are not reached through the visible path. No guarantee that the initial assignment is undone follows from this function.

## Uncertainties

The following remain **documented_only**:

- The identity and meaning of the global slot and offset-12 field.
- Independent confirmation of the unsigned-word width, retained-base width, pointer representation, byte order, alignment, and address-arithmetic rules.
- Whether the storage is ordinary memory, shared memory, or memory-mapped hardware.
- Access qualifiers and compiler rules affecting actual load count or generated access order.
- Whether external changes between global references are permitted and defined.
- Outcomes for invalid, overflowed, misaligned, inaccessible, or expired addresses.
- Dependency semantics, errors, termination, ownership effects, and global-state effects.
- Any synchronization, lifetime, or hardware-ordering contract supplied elsewhere.

The active-marker interpretation remains inferred intent only. None of these unknowns is resolved by assuming benign dependency behavior.

## Evidence

The supplied static body establishes the separate entry capture, the two additional global-base references within the initial assignment, the offset of 12 bytes, the numerical masks, the dependency argument values and call sequence, and the final use of the retained base.

It also establishes no visible conditional branch, explicit memory-management operation, result inspection, or caller-side error handling.

The body does not establish an evaluation order between the initial destination-address expression and right-hand-side expression, nor does it establish three physical global loads. The initial same-word interpretation therefore requires a stable-global assumption.

No runtime tests have been performed. The acceptance examples are conditional verification criteria, not observed execution results.
