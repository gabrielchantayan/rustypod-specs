<!-- Trusted provenance: F-aca4578c6cd8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aca4578c6cd8

## Purpose

The supplied static representation establishes a condition-dependent operation with one normal-return path and several paths whose recovered control flow terminates at unrecognized instruction data.

On the normal-return path, the operation reads one 32-bit memory word relative to its second explicit input and returns a 64-bit value containing that word and a related address-sized arithmetic result. On another path, it writes a register-like record below an implicit base address, then requests a coprocessor operation before reaching the unresolved instruction region.

**Inferred intent:** the record resembles a saved execution context. Neither that interpretation nor a conventional callable-function interface is established. The incoming condition flags and otherwise unspecified register values are essential inputs.

This contract describes the recovered behavior up to the limits of the supplied evidence. It does not establish what the unrecognized instructions actually do.

## Interface

### Explicit inputs

The recovered interface exposes four inputs:

| Input | Recovered width and interpretation | Observable use |
|---|---|---|
| First | Opaque 32-bit word | Written into the lowest-addressed record slot on the record-writing path. |
| Second | Signed 32-bit integer, also used in address expressions | Supplies the normal-return address calculation and memory-read address; also written into the record. |
| Third | Opaque 32-bit word | Written into the record and supplied to the coprocessor operation. |
| Fourth | Opaque 32-bit word | Written into the record. |

These recovered types do not establish a public ABI. In particular, the signed interpretation of the second input coexists with its use as a memory-address basis.

### Implicit inputs

Behavior also depends on:

- Incoming carry, zero, negative, and overflow flags.
- An implicit base value used to address the record-writing destination.
- Four independent 32-bit saved words.
- An opaque register value assigned to a pointer-typed destination in the recovered representation.
- A coprocessor register value.
- Memory contents and access permissions at the selected addresses.

For this contract, the four saved words are named **H1**, **H2**, **H3**, and **H4**, in the order in which they are written. These labels do not imply a calling-convention role.

The flags are inputs to the recovered control flow. There is no visible comparison or arithmetic instruction establishing them within this operation. An interface accepting only the four explicit inputs therefore omits behaviorally necessary state.

### Output

Only the first branch has a recovered normal return. Its result is a 64-bit concatenation:

- Bits 63–32 contain the 32-bit representation of the second input minus 64.
- Bits 31–0 contain the 32-bit word read from the address represented by the second input minus 60.

This is a bit-field concatenation, not a numerically signed extension of the loaded word. The loaded word’s top bit does not alter the upper half of the result.

No normal-return value is established for the remaining paths.

### Preconditions

For the normal-return path to complete:

- The address derived from the second input minus 60 must support the represented 32-bit read.
- The arithmetic and address conversion must have a defined interpretation on the actual target.

For the record-writing path to progress through its visible writes:

- The implicit base and every destination must be valid under the target’s memory-access rules.
- The opaque pointer-typed assignment must be supported at its destination.
- Execution must be permitted to perform the requested coprocessor operation.

Alignment requirements, memory-fault behavior, and arithmetic behavior at representation boundaries are not established. No null check, bounds check, or address validation is visible.

## Behavior

### 1. Normal-return branch takes precedence

The operation first tests whether carry is set and zero is clear.

When both conditions hold:

1. It reads one 32-bit word at the address represented by the second input minus 60.
2. It returns that word in the low half of a 64-bit result, with the 32-bit representation of the second input minus 64 in the high half.

This branch takes precedence over every later condition. Negative and overflow do not change its selection.

No record writes, coprocessor operation, or unresolved-instruction termination are visible on this branch. The ordering between the memory read and evaluation of the independent subtraction is not established; only the read’s contribution to the return value is established.

### 2. Conditional record writes

If the first branch is not taken, the operation tests whether:

- Zero is clear; and
- Negative and overflow have equal Boolean values.

If both conditions hold, the following assignments occur in the listed order. Offsets are byte displacements from the implicit base.

| Order | Offset | Assigned value | Established access width |
|---:|---:|---|---|
| 1 | −4 | Fixed word `0x08362d48` | 32 bits |
| 2 | −8 | Opaque register value | Pointer-typed; exact width unresolved |
| 3 | −12 | Implicit base value itself | 32 bits |
| 4 | −16 | H1 | 32 bits |
| 5 | −20 | H2 | 32 bits |
| 6 | −24 | H3 | 32 bits |
| 7 | −28 | H4 | 32 bits |
| 8 | −32 | Fourth explicit input | 32 bits |
| 9 | −36 | Third explicit input | 32 bits |
| 10 | −40 | Second explicit input | 32 bits |
| 11 | −44 | First explicit input | 32 bits |

These are ordered writes, not an established atomic record update. No rollback is visible if execution fails after some writes have completed.

The repeated four-byte spacing suggests a 44-byte record. However, the assignment at offset −8 is pointer-typed in an anomalous address-space representation. Its exact width is not independently established. Consequently, neither a wholly nonoverlapping layout nor a definitive total write footprint can be guaranteed from this evidence alone. With a four-byte interpretation for that assignment, the record occupies offsets −44 through −1 inclusive.

The stored implicit base does not establish an update to the live base. No such update is visible.

### 3. Coprocessor operation follows the writes

After the last listed write, the operation requests a coprocessor-store primitive with:

- Coprocessor selector 5.
- The incoming coprocessor register value.
- The third explicit input as the remaining operand.

The primitive’s name suggests a store, but the supplied evidence does not define its addressing rules, transfer width, privilege requirements, completion behavior, or effects on other state. The third input must not be treated as a proven ordinary-memory destination merely because of that name.

There is no visible normal return immediately after this primitive. If it completes and execution continues as represented, control reaches the unresolved instruction region.

### 4. Remaining paths reach unresolved instruction data

If the normal-return branch is not selected, every recovered continuation ends at an invalid-instruction termination marker, either directly or after the record writes and coprocessor operation.

Later tests distinguish overflow and, within one case, negative. All their visible endpoints are unresolved instruction data. These tests do not establish distinct return values or distinct recoverable error codes.

The termination markers are limits of the static representation. They do not prove that the actual program intentionally aborts, raises a particular signal, traps in a specific way, or cannot execute valid instructions under some other interpretation.

### Flag combinations

The following table captures the visible branch selection before considering memory faults or coprocessor behavior:

| Incoming flags | Visible path |
|---|---|
| Carry set, zero clear | Normal return, regardless of negative and overflow |
| Zero set, any other flags | No record writes; unresolved instruction region |
| Carry clear, zero clear, negative clear, overflow clear | Record writes, coprocessor request, unresolved instruction region |
| Carry clear, zero clear, negative set, overflow set | Record writes, coprocessor request, unresolved instruction region |
| Carry clear, zero clear, negative and overflow unequal | No record writes; unresolved instruction region |

### Arithmetic limits

The constants 60 and 64 are byte displacements in the recovered address and return expressions. They are not element counts.

For values where both subtractions are representable as signed 32-bit arithmetic and produce valid target addresses, the normal-return relation is unambiguous. Near the signed lower bound, subtraction may leave that range. The supplied evidence does not settle whether the actual target wraps, faults, or requires another interpretation.

A specification consumer must not silently choose checked arithmetic, saturation, or modular wrapping as though that choice were established.

## Side effects

### Memory

The normal-return branch performs one visible 32-bit memory read and no visible memory write.

The record-writing branch performs the ordered assignments listed above before requesting the coprocessor operation. No ordinary-memory read is explicitly represented for constructing those assignments; their values come from explicit or implicit inputs.

This does not prove that the accesses are harmless. If an address refers to device-backed memory, a read or write may itself have external effects. The evidence does not classify any addressed region.

### Other state

The record-writing path invokes an incompletely specified coprocessor operation. Its effects on memory, registers, flags, global state, or execution status are unknown.

No visible allocation, deallocation, lock operation, ownership transfer, or release occurs before that dependency. That observation does not establish that the dependency preserves state or avoids memory-management effects.

### Errors and partial completion

No explicit error result or recovery mechanism is visible.

An inaccessible read can prevent the normal return. An inaccessible record destination can interrupt the write sequence after earlier assignments. A coprocessor failure can occur after the record writes have been attempted.

The particular exception, signal, trap, or continuation associated with any such event is outside the established contract.

## Dependencies

No separately identified application dependency is available in the supplied dependency map.

Two opaque dependency roles are used here solely to distinguish the unresolved external mechanisms:

- **F-unknown-01 — coprocessor-store primitive.** Invoked only after the record-writing sequence. Its operand selection is established; its implementation, completion behavior, and broader effects are not.
- **F-unknown-02 — invalid-instruction termination marker.** Represents the endpoints where recovered control flow stops at unrecognized instruction data. It is not established to be an actual callable routine in the original execution environment.

These labels are local contract references, not provenance-backed identities of additional functions.

Neither dependency is assumed to preserve globals, preserve incoming registers, own memory, or release memory.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

### Normal return with concrete addresses

Given:

- Carry is set and zero is clear.
- The second input is `0x00001000`.
- A supported 32-bit read at `0x00000FC4` yields `0x89ABCDEF`.

The expected normal return is `0x00000FC089ABCDEF`.

The upper half is `0x00000FC0`, corresponding to subtraction by 64. The low half is the word read at the distinct address obtained by subtraction by 60. No record writes or coprocessor request should precede this return.

### Early-return precedence

Given carry set, zero clear, negative set, and overflow set, the normal-return branch still wins.

Although negative equals overflow, the record-writing condition is never reached. An implementation that writes the record before returning would violate the visible branch order.

### Record-writing path

Given carry clear, zero clear, negative clear, overflow clear, and an implicit base of `0x00002000`:

- The first write places `0x08362d48` at `0x00001FFC`.
- The base-value write places `0x00002000` at `0x00001FF4`.
- The last write places the first explicit input at `0x00001FD4`.
- The intervening assignments occur in the specified order.
- The coprocessor request follows the last assignment.

No successful normal return is established after that request.

The access at `0x00001FF8` remains subject to the unresolved width of the pointer-typed assignment.

### Zero suppresses both visible productive paths

Given zero set, neither normal return nor record writing is selected, regardless of the other flags.

The visible continuation reaches unresolved instruction data without the listed writes or coprocessor request. This does not establish a particular runtime failure mode.

### Unequal negative and overflow flags

Given carry clear, zero clear, negative set, and overflow clear, the record-writing condition fails.

The operation must not perform that record-writing sequence merely because zero is clear.

### Partial write failure

If the record-writing path reaches its fifth assignment and that access fails, the first four assignments have already been attempted in order. There is no visible rollback.

Whether each earlier assignment completed, and how the failure is reported, depends on the target’s memory semantics.

## Uncertainties

The following remain **documented_only**:

- Whether the recovered entry is a conventional callable function, an interior entry point, or a misinterpreted instruction sequence.
- The actual ABI and the origins of implicit register and flag inputs.
- Address-space width, pointer representation, alignment requirements, and byte order.
- The exact width and footprint of the pointer-typed assignment at offset −8.
- Arithmetic behavior when subtraction exceeds the signed 32-bit range or crosses address-space boundaries.
- Whether addressed memory is ordinary storage, device memory, or invalid.
- The meaning of the fixed word `0x08362d48`.
- The full semantics and side effects of F-unknown-01.
- The actual instructions and runtime outcomes represented by F-unknown-02.

The context-saving interpretation is inferred intent only. No ownership model, record consumer, restoration mechanism, or later successful continuation is established.

## Evidence

This contract is based solely on the supplied static representation: its recovered input widths, condition tests, 64-bit concatenation, single explicit read, ordered assignments, coprocessor primitive, and unresolved control-flow endpoints.

The evidence explicitly reports unrecognized instruction data. Accordingly, statements about execution beyond those endpoints remain unresolved rather than being treated as ordinary function behavior.

No implementation access, runtime execution, memory inspection, or tests were performed. The acceptance examples are deductions from the visible operations under their stated preconditions, not observations of a running program.
