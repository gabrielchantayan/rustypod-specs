<!-- Trusted provenance: F-d4789af8fa5d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d4789af8fa5d

## Purpose

F-d4789af8fa5d conditionally updates one shared 32-bit word, obtains a masked snapshot of another shared 32-bit word, optionally invokes two dependencies, removes selected bits from the snapshot, and publishes the remaining bits through two writes.

The routine resembles an event or status handler. That interpretation is **inferred intent**, not an established hardware contract. The supplied evidence does not establish whether the shared locations are ordinary memory, device registers, or aliases with special read/write behavior.

A significant behavioral distinction is that an initially empty masked snapshot causes an early return, whereas a snapshot that becomes empty during processing still causes the final publication writes.

## Interface

### Input

The routine accepts one integer-valued argument used as an address-bearing context reference.

Its directly observed uses are:

- Addressing one byte at offset **0x2B**, or 43 bytes, from the supplied reference.
- Passing the original argument unchanged to F-fec396f49ccf.
- Passing the original argument unchanged to F-87d0f22453b5.

No complete context layout is established. The only directly accessed field occupies one byte at offset 0x2B. Its interpretation here is strictly “zero” versus “nonzero”; signedness does not affect that distinction.

The supplied representation uses an integer argument rather than a typed pointer. A 32-bit address-bearing environment is consistent with the evidence, but the target ABI and address-conversion rules are not independently established.

### Output

There is no explicit return value. Observable results consist of:

- Shared-memory or register reads and writes.
- An optional byte read from the context.
- Optional dependency calls and whatever effects those dependencies produce.

### Shared operands

This contract uses the following descriptive names:

| Name | Role |
|---|---|
| **Control word** | Shared 32-bit word at address **0x38400004**. |
| **Status word** | Shared 32-bit word at address **0x38400014**. |
| **Control mask** | Externally supplied mask used to test and update the control word. |
| **Selection mask** | Externally supplied mask applied to the initial status-word read. |
| **Removal mask** | Externally supplied mask used during later residual-bit removal. |
| **Publication base** | Externally supplied base address used to locate a 32-bit publication word at offset **0x24**, or 36 bytes. |

The numerical values of the masks and publication base are not supplied. Their storage properties, mutability, and possible aliasing are also unknown.

The bit-processing operations use an unsigned 32-bit working value. In particular:

- **0x40** selects bit 6.
- **0x00000002** selects bit 1.
- Clearing either bit leaves the other 31 bits unchanged.
- Complement operations in the bit-processing description are confined to 32 bits.

### Preconditions

On any path that accesses them, the shared words and mask operands must support their represented reads or writes.

Additional path-dependent requirements are:

- The context byte at offset 0x2B must be readable if the selected snapshot contains bit 6.
- The context must satisfy both dependencies’ requirements if that byte is zero.
- The publication destination must support a 32-bit read and write whenever the selected snapshot is initially nonzero.

The routine contains no explicit null, alignment, bounds, address-overflow, or access-permission checks. The exact consequences of violating these requirements are outside the supplied evidence.

## Behavior

### 1. Conditional control-word update

Before examining the status word, the routine tests whether the control word and control mask have any set bit in common.

If they have none, it performs no explicit control-word write.

If they have at least one, the routine performs a new read-modify-write operation on the control word: it writes the bitwise intersection of the control word and control mask.

This operation preserves bits selected by the mask and clears other bits in the value used for that write. It is **not** an operation that simply clears the tested bits.

The initial test and the conditional update are separate evaluations. The supplied representation does not cache the initially tested word or mask for the update. Consequently, if either operand changes between those evaluations, the write need not reflect the values that caused the branch to be taken.

### 2. Initial status selection and early return

After the control-word phase, the routine reads the status word and intersects it with the selection mask. The resulting 32-bit value becomes the working snapshot.

If this snapshot is zero, the routine returns immediately.

On this early-return path:

- The control-word phase has already occurred.
- The context byte is not read.
- Neither dependency is called.
- No explicit status-word write occurs.
- No explicit publication-word read or write occurs.

An originally nonzero status word can therefore still take this path if the selection mask excludes all its set bits. The routine does not write the masked result back on this path.

### 3. Processing bit 6

If the working snapshot contains **0x40**, the routine reads the context byte at offset 0x2B.

If the byte is zero, it invokes these dependencies in order:

1. F-fec396f49ccf, with the original context argument and a second integer argument equal to **1**.
2. F-87d0f22453b5, with the original context argument.

The second call occurs after the first returns normally. There is no intervening recheck of the context byte or working snapshot.

If the byte is nonzero, both calls are skipped.

In either case, after this conditional call sequence returns normally, bit 6 is cleared from the working snapshot.

The bit is removed only from the routine’s working value at this stage. An explicit status-word write occurs later.

If bit 6 was absent from the snapshot, this entire phase—including the context-byte read—is skipped.

### 4. Processing bit 1

If the working snapshot contains **0x00000002**, the routine clears that bit.

There is no dependency call, context access, or immediate shared-word write associated with this bit.

This phase follows bit-6 handling, including any dependency calls. Therefore, any calls triggered by bit 6 occur before bit 1 is removed from the working snapshot.

### 5. Removing additional selected bits

The routine tests whether the current working snapshot intersects the removal mask.

If the intersection is empty, the working snapshot is unchanged.

If the intersection is nonempty, the routine evaluates the removal mask again and removes the intersecting bits from the working snapshot. For a stable removal mask, this means that every bit selected by that mask is cleared from the working snapshot.

The test and removal use separate evaluations of the mask. If the mask can change between them, the first value controls whether removal is attempted, while the later value determines which bits are removed.

The control mask and selection mask have already been consumed before any dependency calls. The removal mask is evaluated afterward. Thus, effects of the dependencies on later-read shared state cannot be excluded.

### 6. Publishing the residual value

If execution reached this phase, the routine performs both of the following operations in order:

1. Writes the residual working snapshot to the status word.
2. Performs a read-modify-write on the 32-bit word at publication base plus 0x24, writing the bitwise OR of that word’s read value and the residual snapshot.

These operations occur even when the residual snapshot is zero.

Consequences include:

- A nonzero initial snapshot that is entirely consumed still causes a zero write to the status word.
- The publication destination is still read and written when the residual is zero.
- The status-word write replaces the whole represented 32-bit word; it is not an explicit merge with a fresh status read.
- Bits excluded by the initial selection mask are not preserved by that final assignment.
- Status changes made by dependencies are not explicitly merged into the final status write.

If the publication destination aliases the status word, the publication operation follows the status write and must be interpreted with that aliasing in mind. No distinct-storage guarantee is provided.

The supplied expression also refers to the publication base for both the read and write destination. A stable base is required to interpret this as a read-modify-write of one fixed location; caching or atomicity is not established.

## Side effects

The routine’s explicit effects are limited to:

- Conditional modification of the control word.
- Conditional reading of one context byte.
- Conditional calls to two dependencies.
- A status-word write on every path whose initial selected snapshot is nonzero.
- A subsequent publication-word read and write on those same paths.

There is no explicit allocation, deallocation, ownership transfer, or direct context-field write.

That absence does not constrain the dependencies. The evidence does not establish whether they allocate memory, release the context, modify shared operands, alter the context byte, or affect device state.

The routine itself does not dereference the context after the dependency sequence. However, it passes the same context argument to the second dependency after the first returns. Validity of that second call depends on the dependencies’ actual contracts, which are unavailable here.

No lock, atomic read-modify-write primitive, barrier, retry loop, or explicit error-reporting path is shown. Access ordering within an expression, device-specific semantics, and compiler-level volatility are not established by this source representation.

## Dependencies

| Opaque ID | Invocation condition | Arguments | Established ordering |
|---|---|---|---|
| **F-fec396f49ccf** | Initial selected snapshot contains bit 6, and the context byte is zero | Original context argument; integer **1** | First dependency call |
| **F-87d0f22453b5** | Same condition, provided the preceding call returns normally | Original context argument | After F-fec396f49ccf |

No returned value from either dependency is consumed by this routine.

Their purpose, success criteria, failure behavior, memory ownership rules, and preservation of globals are **documented_only** unknowns. Normal return from a call is a condition for the subsequent behavior described here, not a guarantee supplied by the dependency map.

## Acceptance examples

These examples are conditional acceptance criteria, not reported test results. Unless noted otherwise, they assume stable masks and base addresses, ordinary non-aliased memory behavior, valid accesses, and normal dependency returns.

### Empty initial selection

Suppose the status word contains 0x80 and the selection mask is 0x40.

The selected snapshot is zero. After completing the control-word phase, the routine returns without reading the context byte, calling either dependency, or accessing the publication destination. The routine does not explicitly rewrite the status word.

### Control-word intersection

Suppose the control word is 0xF0 and the control mask is 0x30.

Their intersection is nonzero, so the conditional write occurs. With stable operands, the control word becomes 0x30, not 0xC0.

### Bit 6 with a zero context byte

Suppose the selected snapshot is exactly 0x40 and the context byte is zero.

The routine calls F-fec396f49ccf with the context and integer 1, then F-87d0f22453b5 with the context. After normal returns, the residual is zero. The routine writes zero to the status word and still performs the publication read and write.

### Bit 6 with a nonzero context byte

Suppose the selected snapshot is 0x44, the context byte is nonzero, and the removal mask is zero.

Neither dependency is called. Bit 6 is removed, leaving 0x04. The status word receives 0x04. A publication word initially containing 0x10 receives 0x14.

### Multiple removal mechanisms

Suppose the selected snapshot is 0x4F and the removal mask is 0x0C.

Clearing bit 6 leaves 0x0F. Clearing bit 1 leaves 0x0D. Removing the mask-selected bits leaves 0x01. Subject to completion of any required calls, the final status write is 0x01.

### Dependency mutation of status

Suppose bit 6 triggers both calls, and a dependency changes the status word before returning.

The routine does not reread that status word to refresh its snapshot. Its later status assignment uses the residual derived from the original selected snapshot. For ordinary memory, that assignment can overwrite the dependency’s status change.

### Changing removal mask

Suppose the working snapshot is 0x0C. The first removal-mask evaluation yields 0x04, and the evaluation used for removal yields 0x08.

The removal phase is entered because the first intersection is nonzero. It then removes 0x08, leaving 0x04.

## Uncertainties

The following remain **documented_only**:

- Actual values and storage widths of the externally supplied masks.
- The target ABI, pointer conversion rules, and address-arithmetic overflow behavior.
- Whether shared words are hardware registers and whether reads or writes have special effects.
- Whether accesses are volatile, atomic, ordered, or concurrent with other execution.
- Whether the publication base is stable and whether any locations alias.
- Exact operand-read ordering within individual expressions.
- Dependency ownership, preservation, failure, and nonreturning behavior.
- Required context layout beyond the single observed byte.
- Whether replacing the status word is intentional acknowledgment behavior or ordinary state replacement.

## Evidence

**Established static evidence:** the supplied function representation supports the branch conditions, 32-bit bit manipulations, byte offset, shared-word addresses, dependency arguments and ordering, early return, and final write sequence described above.

**Inferred intent:** the shared-word pattern resembles masked event processing followed by residual-state publication. No specific peripheral, protocol, or event meaning is established.

**Verification limit:** no implementation execution, runtime tests, device observations, or dependency-body inspection has been performed. The acceptance examples specify conditional behavior; they are not execution evidence.
