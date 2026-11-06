<!-- Trusted provenance: F-d5fa6dc21b15; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d5fa6dc21b15

## Purpose

F-d5fa6dc21b15 performs a statically recoverable sequence of indirect invocations and memory updates. It invokes two dependencies unconditionally within the represented prefix. A third invocation and five-field construction occur only when a destination derived from Argument 3 is nonzero.

The nonzero-destination check guards the entire conditional region: the second-global read, third invocation, destination `+0x30` read, and five destination writes. Earlier accesses are outside this guard. A nonzero destination is not necessarily a valid, aligned, or writable address.

The supplied representation does not establish a complete, normally returning function. Both visible paths end at invalid-instruction analysis boundaries. This contract describes the represented prefix without asserting a return, deliberate runtime trap, or behavior beyond those boundaries.

Initialization or record-linking intent is plausible but remains inference. The unusual byte-derived destination and address scaling are part of the static evidence, not operations to normalize into more conventional behavior.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| Argument 1 | 32 bits | No explicit read in the represented body. |
| Argument 2 | 32 bits | No explicit read in the represented body. |
| Argument 3 | Signed 32 bits | Added to a captured global value for a byte lookup; shifted right by two to derive the conditional destination. |
| Argument 4 | Signed 32 bits | Used as an address base for a word write and stored in the context. |

The lack of explicit reads of Arguments 1 and 2 does not establish that their incoming machine representations are irrelevant to indirectly invoked dependencies. Actual calling conventions are unresolved.

### Ambient inputs

The explicit arguments alone are insufficient to determine the effects. Additional represented inputs are:

- A context address supplied through incoming machine state.
- An incoming link-related value used to calculate the first invocation target.
- A first global 32-bit value, read after the first invocation.
- A second global 32-bit value, read only inside the nonzero-destination region.
- Secondary machine results associated with the first and third invocations.
- The address of a represented stack-associated location labeled `+0x2D0`.
- Accessible memory and executable invocation targets.

The stack-associated label identifies the represented location whose address is stored. It does not independently establish a runtime calculation from the current stack pointer, an allocated object, or a lifetime.

### Widths and layouts

The represented operations use 8-bit byte reads, 16-bit halfword writes, and 32-bit word reads and writes. Pointer-derived values and addresses are represented through 32-bit quantities. Byte order, alignment requirements, and actual machine calling conventions are unresolved.

The context has these visible accesses:

| Byte offset | Access | Role |
|---:|---|---|
| `+0x00` | 32-bit write | Receives the first invocation’s secondary result. |
| `+0x04` | 32-bit write | Receives Argument 4. |
| `+0x08` | Address formation | Interior address used in the second invocation and final conditional field. |
| `+0x24` | 32-bit read | Supplies the value truncated for both halfword writes. |

Other visible memory accesses are:

| Base | Byte offset | Access |
|---|---:|---|
| First invocation’s secondary result, interpreted as an address | `+0x36` | 16-bit write |
| Argument 4, interpreted as an address | `+0x48` | 32-bit write |
| Zero-extended lookup byte, interpreted as an address | `+0x2A` | 16-bit write |
| Shift-derived destination | `+0x00`, `+0x04`, `+0x08`, `+0x0C`, `+0x10` | Five ordered 32-bit writes |
| Shift-derived destination | `+0x30` | 32-bit read during conditional construction |

These accesses do not establish complete object definitions, object extents, or ownership.

### Outputs and completion

No explicit return value is represented. Recoverable outputs consist of memory updates and dependency invocations.

Neither visible path reaches an established normal return. Invalid-instruction analysis boundaries are not sufficient evidence of an intentional error response or runtime termination mechanism.

## Behavior

The following order is established by the supplied static representation. Reaching each operation requires preceding accesses to complete and preceding dependencies to return to the represented continuation.

### First invocation and captured values

The function invokes **F-000000000001** at a target formed by adding `0xC2A` to the incoming link-related value. No explicit arguments are shown.

A secondary result associated with this invocation is subsequently interpreted both as an address and as a 32-bit value.

After the invocation, the function reads:

1. The first global 32-bit value.
2. The context word at byte offset `+0x24`.

Both reads occur after the dependency, so neither value is necessarily a snapshot of function-entry memory.

### Initial writes

Using those captured values, the function performs these writes in order:

1. Writes the low 16 bits of the captured context word to the first secondary-result address plus `0x36`.
2. Writes the captured first-global value to Argument 4 plus `0x48`.
3. Writes the first secondary result to context offset `+0x00`.
4. Writes Argument 4 to context offset `+0x04`.

The halfword conversion retains the low 16-bit pattern. It is not a checked conversion or a selection of the upper halfword.

Both source words were captured before these writes. Aliasing can therefore change their original storage without changing the represented captured values used later.

No represented null, bounds, or alignment checks guard these accesses.

### Byte lookup and derived destination

The function reads one byte at the address formed by adding Argument 3 to the captured first-global value. That byte is zero-extended to a 32-bit value between `0` and `255`.

The lookup follows all four initial writes. If those writes overlap the lookup address, the lookup observes the resulting memory, subject to the execution environment’s access semantics.

Argument 3 is then shifted right by two, and the result is interpreted as the conditional destination address. For nonnegative Argument 3, this discards the two low bits and divides the numeric value by four. It does not align the original address downward.

For example, Argument 3 equal to `0x00001000` produces destination `0x00000400`. Treatment of negative Argument 3 remains documented_only because the signed expression alone does not settle the underlying execution semantics.

The function next writes the low 16 bits of the previously captured context word to the zero-extended lookup byte plus `0x2A`, treating that sum directly as an address.

This halfword write starts at an address between `0x2A` and `0x129`, inclusive. There is no represented pointer-table lookup or additional dereference to obtain its destination. This access is not guarded by the later destination check.

### Second invocation

The function invokes **F-000000000002** with two explicit 32-bit arguments:

1. The zero-extended lookup byte.
2. Eight times the integer representation of the context interior address at `+0x08`.

The multiplication applies to the address value, not to contents read from that address. It does not establish a size, count, or allocation request.

The static target calculation clears the low two bits of `0x083FE7DD`, obtaining `0x083FE7DC`, then adds `0xCF8`. The resulting target is **`0x083FF4D4`**.

This arithmetic identifies a represented target value, not a verified executable entry point.

The scaled argument uses represented 32-bit integer operations. It must not be interpreted as an unbounded mathematical product. Exact overflow and pointer-conversion behavior remain unresolved.

This invocation occurs before the destination check and is therefore part of both visible paths.

### Nonzero-destination guard

After the second invocation, the shift-derived destination is explicitly tested against zero.

If it is zero, the function skips all of the following:

- The second-global read.
- The third invocation.
- The read at destination `+0x30`.
- All five destination writes.

If it is nonzero, execution enters the conditional construction region.

This is an established null check for that region. It is not a validity check: nonzero alone establishes neither accessible storage nor sufficient extent, alignment, or writability.

### Conditional five-field construction

Inside the guarded region, the function first reads the second global 32-bit value, then invokes **F-000000000003** with no explicit arguments shown.

The third target calculation adds `0xD6E` to `0x083FE7FF`, producing **`0x083FF56D`**. No low-bit-clearing operation is represented for this invocation. The odd target’s architecture-specific meaning and executability are unresolved.

After the invocation, the function constructs five consecutive 32-bit fields:

| Destination offset | Written value |
|---:|---|
| `+0x00` | Address of the represented stack-associated location labeled `+0x2D0`. |
| `+0x04` | Secondary result associated with the third invocation. |
| `+0x08` | Second-global value captured before the third invocation. |
| `+0x0C` | Value read from destination `+0x30`. |
| `+0x10` | Context interior address at `+0x08`, without scaling. |

The writes occur in ascending offset order.

The read at destination `+0x30` occurs after the third invocation and after the first three field writes, immediately supplying the fourth field. In contrast, the second-global value was captured before the third invocation.

The first field contains an address, not the contents of the stack-associated location or the literal offset `0x2D0`.

The final field contains the unscaled context interior address. It differs from the address-derived quantity multiplied by eight for the second invocation.

### Continuation boundary

Both represented paths reach invalid-instruction analysis boundaries. No later cleanup, return, error conversion, or continuation is established.

## Side effects

### Memory access and ordering

The earlier accesses have no represented null guards. The conditional region does have an explicit nonzero-destination guard, covering its global read, invocation, source read, and five writes.

No represented bounds, accessibility, or alignment checks establish the safety of either region. A nonzero destination can still be invalid.

Observing the complete represented sequence requires readable sources, writable destinations, and dependencies that return to the shown continuation. These are execution preconditions, not guarantees provided by the function.

Aliasing and dependency effects can influence later reads:

- Initial writes can alter the later lookup byte.
- Dependencies can modify globals, context storage, or destination storage.
- Destination `+0x30` is read only after the third invocation and the first three conditional writes.

Captured values are distinguished from subsequent memory contents. The first global and context word are captured after the first invocation; the second global is captured after the guard and before the third invocation.

No transaction or rollback is represented. Earlier writes may remain observable if a later operation fails or a dependency does not return. Atomicity, synchronization, and concurrent-access guarantees are unresolved.

### Ownership and lifetime

No explicit allocation, deallocation, reference-count operation, or ownership transfer is established.

Storing an address does not prove ownership transfer. Passing an address-derived value does not establish whether a dependency dereferences, retains, or releases anything.

The lifetime of the stack-associated address stored in the first conditional field is unresolved. That field must not be described as a durable allocation.

### Errors

No explicit error return, status output, or recovery branch is represented. The zero-destination path skips construction; it is not established as an error path.

Invalid addresses, unsuitable alignment, arithmetic semantics, invalid invocation targets, or non-returning dependencies can prevent later operations. Their concrete fault mechanisms and externally visible error behavior remain documented_only.

## Dependencies

These opaque IDs identify invocation sites for this contract, not independently verified catalog identities.

| Dependency | Established interface | Unresolved properties |
|---|---|---|
| **F-000000000001** | Target is incoming link-related value plus `0xC2A`; no explicit arguments; secondary result is used as an address and stored. | Actual inputs, result convention, effects, ownership, preservation, completion. |
| **F-000000000002** | Target calculation yields `0x083FF4D4`; receives the lookup byte and eight times the integer representation of context address `+0x08`. | Executability, argument meanings, results, effects, ownership, completion. |
| **F-000000000003** | Target calculation yields `0x083FF56D`; invoked only under the nonzero-destination guard; secondary result is stored at destination `+0x04`. | Target interpretation, actual inputs, result meaning, effects, ownership, completion. |

No dependency is assumed to preserve globals or memory, or to own or release any referenced storage. Distinguishing captured values from later reads does not establish the machine-level preservation mechanism.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Each assumes the required accesses succeed and dependencies return to the represented continuation.

### Halfword truncation

If the captured context word is `0x1234ABCD`, both halfword writes use bit pattern `0xABCD`.

A later change to context `+0x24` does not alone change the already captured value.

### Zero destination

If Argument 3 is `3`, the derived destination is zero.

The first invocation, initial reads and writes, byte lookup, byte-derived halfword write, and second invocation still occur. The guard then skips the second-global read, third invocation, destination `+0x30` read, and all five destination writes.

No accessible destination storage is required for the skipped operations. Earlier accesses still require their own valid locations.

### Nonzero does not establish validity

If Argument 3 is `4`, the derived destination is address `1`, so the nonzero guard admits the conditional region.

This does not establish that construction succeeds. Even if the second-global read and third invocation complete, a destination write at address `1` may be invalid or misaligned.

### Nonzero construction and copy source

If Argument 3 is `0x00001000`, the destination is `0x00000400`.

The five writes start at `0x400`, `0x404`, `0x408`, `0x40C`, and `0x410`. The fourth field reads its source from `0x430`, after the third invocation and first three writes.

### Lookup-byte boundary

If the lookup yields `255`, the byte-derived halfword write starts at `0x129`, and the second invocation receives `255` as its first explicit argument.

The odd address may be unsuitable for a halfword access. Successful execution is not implied.

### Captured global versus later source

If the second global contains A when read, and the third dependency then changes that global to B and destination `+0x30` to C, the conditional fields receive A at `+0x08` and C at `+0x0C`, assuming no further interference.

### Address-valued first field

On the nonzero path, let S denote the address of the represented stack-associated location labeled `+0x2D0`.

The first field receives S, not the contents at S and not the literal value `0x2D0`. Resolving S from a runtime stack pointer requires additional evidence.

### Target calculations

The second target is `0x083FF4D4`; the third is `0x083FF56D`. The third calculation has no represented low-bit-clearing step. These are arithmetic expectations, not evidence of successful invocation.

## Uncertainties

The following remain documented_only:

- Whether the supplied representation identifies a complete executable function.
- Actual calling conventions and incoming machine-state requirements.
- Dependency identities, effects, ownership, preservation, and completion.
- Executability and architecture-specific interpretation of invocation targets.
- Whether the unusual low-address write reflects executable behavior.
- Right-shift behavior for negative Argument 3.
- Exact overflow, pointer-conversion, byte-order, and alignment semantics.
- Valid address ranges, object extents, and accessibility.
- Runtime resolution and lifetime of the stack-associated address.
- Behavior beyond either invalid-instruction analysis boundary.

The nonzero-destination guard is established, not uncertain. Its adequacy as an address-validity check is not established.

Higher-level initialization or record-linking purpose remains inference rather than an acceptance requirement.

## Evidence

Established static evidence comprises the supplied representation’s access widths, offsets, arithmetic expressions, capture points, invocation order, explicit nonzero-destination condition, and ordered field construction.

The guard precedes the second-global read, third invocation, destination `+0x30` read, and five destination writes. It does not guard earlier accesses.

The represented target arithmetic yields `0x083FF4D4` and `0x083FF56D`. The first conditional field explicitly receives the address of a stack-associated location labeled `+0x2D0`. These facts do not establish executable-target validity or a concrete runtime stack layout.

The supplied representation reports invalid instruction data and overlapping global symbols. Those limitations prevent claiming a complete executable contract but do not justify discarding unusual operations or the explicit guard.

No runtime tests, independent executable inspection, or dependency inspection have been performed. All acceptance examples are conditional static expectations, not observed execution results.
