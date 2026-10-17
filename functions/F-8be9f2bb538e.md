<!-- Trusted provenance: F-8be9f2bb538e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8be9f2bb538e

## Purpose

F-8be9f2bb538e returns either zero or the second byte of an input sequence, selected by the most significant bit of its first byte.

- When bit 7 of the first byte is clear, the result is zero.
- When bit 7 is set, the result is the entire second byte, unchanged.

The lower seven bits of the first byte do not affect selection. No bit of the second byte is discarded.

This is a byte-selection operation, not a Boolean query: a selected second byte may itself be zero, and a nonzero result need not equal the selection mask.

These properties are established by static inspection. Interpreting the first byte as a format header or the second byte as an optional field is inferred intent, not an established requirement.

## Interface

**Input:** One pointer identifying the start of a byte sequence.

**Output:** One unsigned 8-bit value, from 0 through 255 inclusive.

There is no length parameter, separate status output, or returned pointer. The function does not establish who owns the storage addressed by its input.

### Relevant memory layout

| Byte offset from the input pointer | Meaning for this operation | Read requirement |
|---|---|---|
| 0 | Selection byte; bit 7 determines the result source | Read on every normal execution |
| 1 | Candidate result byte | Read only when bit 7 at offset 0 is set |

Offsets are measured in bytes. No offset beyond 1 is accessed by the supplied body.

For normal execution, offset 0 must support a valid byte read. If its selection bit is set, offset 1 must also support a valid byte read. When the selection bit is clear, the supplied body does not require readable storage at offset 1.

The function performs no explicit validation of these requirements. Merely passing a pointer does not establish that the required bytes exist or are accessible.

### Widths and representation

Both the selection byte and result byte are 8-bit quantities. The selection mask is hexadecimal `80`, corresponding to bit 7. The returned result preserves all eight bits of the selected byte, including its most significant bit.

No multi-byte value is assembled, so byte order does not affect the result. There is no variable index, length calculation, multiplication, or application-level arithmetic whose overflow changes the selection rule. The only nonzero byte displacement is the fixed offset 1; the corresponding read must still be valid in the execution environment.

The evidence establishes a byte-valued result, not the width of a machine pointer or the contents of unused bits in a wider return register. Integer promotions used to evaluate the bit test do not change its required outcome.

## Behavior

### Selection and result

The function first reads the byte at offset 0 and tests its most significant bit.

For selection-byte values from `00` through `7F`, it returns `00`. The second byte does not contribute to this outcome and is not read by the supplied body.

For selection-byte values from `80` through `FF`, it reads offset 1 and returns that byte unchanged. The lower seven selection-byte bits have no further role.

The function does not return the selection byte, the mask, or a normalized true-or-false result. For example, a selection byte of `80` followed by `A5` produces `A5`, not `80` or `01`.

### Observable access order

The established source-level order is:

1. Read the byte at offset 0.
2. Determine whether its high bit is set.
3. If selected, read the byte at offset 1.
4. Return zero or the selected byte.

There is one explicit read of the selection byte. There is no later reread that revalidates the decision, no loop, and no scan for additional bytes.

Conditional access matters independently of the returned value. A selected second byte containing zero must still be read. Conversely, an unselected second byte is not read even if its contents would also be zero.

These are source-level access properties. They do not establish generated instruction widths, speculative-access behavior, cache effects, or suitability for device memory.

### Result interpretation

A zero result is ambiguous between two ordinary cases:

- The selection bit was clear.
- The selection bit was set and the second byte contained zero.

There is no separate indication of which case occurred. Zero is not an established error code.

Under valid, stable input-memory conditions, a nonzero result means that the selection bit was set and the second byte contained the returned value when read.

### Stability and invalid inputs

For deterministic ordinary use, the relevant storage must remain valid and its contents stable during the reads. The body provides no synchronization, explicit atomic access, or coherent two-byte snapshot.

If the selection byte changes after its read, the body contains no revalidation step. Broader behavior under concurrent modification depends on rules not supplied here.

There is no explicit null-pointer check, bounds check, or inaccessible-memory recovery. An unreadable first byte violates the normal-execution preconditions. An unreadable second byte does so only when the selection bit is set.

A one-byte readable region can therefore satisfy the body’s access requirements when that byte is below `80`. The same region does not satisfy them when its byte is `80` or greater.

No particular exception, fault, termination behavior, or fallback value is specified for invalid memory.

## Side effects

The supplied body reads the selected bytes without writing to the sequence or to other caller-visible storage. It contains no explicit global-state access, allocation, deallocation, callback, logging, or I/O operation.

**Ownership of the underlying storage is not established.** Receiving the pointer does not establish that the caller owns that storage. The body establishes only that this function reads the selected bytes without writing, retaining, freeing, or returning the pointer.

The input pointer is not advanced in caller-visible storage. The result is a copied byte value, not an alias into the input sequence. There is no pointer escape through a dependency call or persistent store in the supplied body.

Reads may have externally defined effects when an address refers to device memory or another special memory region. Such effects are **documented_only**: the body does not establish that arbitrary readable addresses are side-effect-free or supported.

The ownership and read-effect distinctions are independent. Absence of a release operation neither identifies the owner nor determines the storage’s lifetime before or after this call.

## Dependencies

No callable dependencies appear in the supplied body. There are no additional opaque function IDs to enumerate.

Normal execution relies on the surrounding environment providing valid byte-addressable storage for the required reads. The supplied evidence does not establish hidden length validation, memory management, synchronization, or error recovery.

Because there are no dependency calls, no assumption about an unknown callee’s treatment of globals or ownership is needed.

## Acceptance examples

These are conditional acceptance requirements derived from static evidence, not observed test results. Unless a case states otherwise, the required bytes are readable and remain stable throughout execution. All table values are hexadecimal.

| Byte at offset 0 | Byte at offset 1 | Required result | Required source-level reads |
|---|---|---|---|
| `00` | `A5` | `00` | Offset 0 only |
| `7F` | `FF` | `00` | Offset 0 only |
| `80` | `00` | `00` | Offset 0, then offset 1 |
| `80` | `01` | `01` | Offset 0, then offset 1 |
| `80` | `80` | `80` | Offset 0, then offset 1 |
| `FF` | `A5` | `A5` | Offset 0, then offset 1 |
| `81` | `FF` | `FF` | Offset 0, then offset 1 |

Additional acceptance boundaries:

- **Only offset 0 is readable and contains `7F`:** the read preconditions are satisfied. The required result is zero without a source-level read at offset 1.
- **Only offset 0 is readable and contains `80`:** a required read is unavailable. No normal result is specified.
- **Separate stable calls use selection bytes `80` and `FF`, both followed by `A5`:** both must return `A5`. Changing the lower seven selection bits does not change the result.
- **The second byte changes between separate stable calls while the selection byte remains `01`:** both must return zero without reading offset 1.
- **Bytes beyond offset 1 differ:** those differences cannot affect the result through any access present in this body.
- **The pointer addresses valid storage not owned by the caller:** lack of caller ownership alone does not change this function’s selection rule or read requirements. Whether the caller is otherwise authorized to use that storage is not established here.
- **The pointer is null or otherwise invalid:** no successful outcome or specific failure mechanism is required by this contract.

## Uncertainties

The following remain **documented_only**:

- Who owns the underlying storage and what governs its lifetime.
- The meaning of either byte in an enclosing data format.
- Whether callers validate accessibility or sequence length.
- Pointer width, machine calling convention, and unused return-register bits.
- Behavior for null, dangling, inaccessible, or otherwise invalid addresses.
- Concurrency, atomicity, memory ordering, and device-memory rules.
- Generated instruction selection and microarchitectural effects.

The apparent purpose may be to retrieve a conditionally present byte. That is inferred intent only. The established behavior requires the selection rule and conditional read, not a particular format interpretation.

No unresolved ownership question is converted into a caller-ownership guarantee.

## Evidence

Static inspection establishes an 8-bit zero default, a high-bit test of the first input byte, a conditional read of the adjacent byte, and return of the resulting byte value.

The supplied body contains no explicit memory writes, callable dependencies, bounds checks, pointer-retention operations, allocation or release operations, or error-handling branches. Those absences support the narrow behavioral claims above; they do not identify the owner of the input storage or define external memory behavior.

No runtime tests have been performed. The acceptance examples specify conditional outcomes and access requirements; they are not execution evidence.
