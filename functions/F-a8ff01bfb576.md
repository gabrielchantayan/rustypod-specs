<!-- Trusted provenance: F-a8ff01bfb576; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a8ff01bfb576

## Purpose

F-a8ff01bfb576 conditionally copies two 32-bit words between fixed locations relative to an implicitly supplied memory base.

The copy decision depends on the signed interpretations of the second and fourth explicit inputs and on an incoming carry flag:

- When incoming carry is set, copy if the fourth input is strictly less than the second input.
- When incoming carry is clear, copy if the fourth input is less than or equal to the second input.

The first and third explicit inputs do not participate in the represented behavior.

This is a source-facing contract derived from static evidence. It does not establish a higher-level purpose for the copied words, the surrounding memory object, or the caller’s comparison convention. No runtime tests have been performed.

## Interface

### Explicit inputs

The function has four explicit inputs, each represented as a 32-bit quantity.

| Position | Interpretation in this function | Use |
|---|---|---|
| First | Uninterpreted 32-bit value | Not read by the represented body |
| Second | 32-bit value interpreted as signed for the decision | Compared with the fourth input, with incoming carry affecting the equality case |
| Third | Uninterpreted 32-bit value | Not read by the represented body |
| Fourth | 32-bit value interpreted as signed for the decision | Determines whether the memory copy occurs |

The signed range for the second and fourth inputs is −2,147,483,648 through 2,147,483,647. Their high bit determines the signed interpretation; an unsigned ordering is not interchangeable with the required ordering.

### Implicit inputs

Two additional inputs are present in the supplied representation:

- **Incoming carry flag:** A Boolean machine-state input. Set corresponds to no additional subtraction; clear corresponds to subtracting an additional one.
- **Memory base:** An address-bearing value obtained from incoming register state rather than from an explicit argument. It identifies the memory region used by the conditional copy.

The explicit signature alone is therefore insufficient to invoke this behavior reliably. A caller or execution harness must also establish the relevant carry and base state according to the actual calling context.

The static evidence does not identify that calling context or establish a standard calling convention for these implicit inputs.

### Memory layout

All offsets below are byte offsets from the implicit memory base.

| Offset | Width | Role |
|---|---:|---|
| +0x38, decimal 56 | 32 bits | First source word |
| +0x3C, decimal 60 | 32 bits | Second source word |
| +0x40, decimal 64 | 32 bits | First destination word |
| +0x44, decimal 68 | 32 bits | Second destination word |

The nominal source span is eight bytes beginning at +0x38. The nominal destination span is eight bytes beginning at +0x40. They are adjacent and non-overlapping in an ordinary contiguous address space.

The words are copied without interpretation. No field-specific meaning, numeric conversion, byte swapping, validation, or normalization is represented.

### Outputs

There is no explicit return value.

The intended observable result of a successful copying execution is the replacement of the two destination words. A non-copying execution performs no explicit data-memory accesses in the represented body.

No return-register value or condition-flag value is specified by this contract.

## Behavior

### Decision

The decision is made before either source word is read.

The supplied expression represents a 32-bit subtraction of the second input from the fourth input, followed by subtraction of one when incoming carry is clear. It accounts for signed overflow from both subtraction stages.

The resulting condition corresponds to the mathematical question:

> Is the signed value of the fourth input, minus the signed value of the second input, minus the carry-dependent extra subtraction, negative?

The extra subtraction is zero for set carry and one for clear carry.

This question must not be implemented or interpreted merely by inspecting the sign bit of a wrapped subtraction result. Overflow handling is material to the behavior.

The equivalent comparison rules are:

| Incoming carry | Copy condition |
|---|---|
| Set | Fourth input is signed-less-than the second input |
| Clear | Fourth input is signed-less-than-or-equal-to the second input |

Thus, for fixed explicit input values, incoming carry changes the outcome only when the second and fourth inputs are equal.

These rules also hold at the signed minimum and maximum. In particular, the clear-carry case must not be reduced to a signed comparison against a wrapped “second input plus one.” That transformation gives an incorrect result when the second input is the signed maximum.

### Copying path

When the condition is true, the represented access sequence is:

1. Read one 32-bit word at base +0x38.
2. Write that word at base +0x40.
3. Read one 32-bit word at base +0x3C.
4. Write that word at base +0x44.
5. Return without an explicit value.

The first destination write precedes the second source read in the supplied representation. The operation is not represented as reading an eight-byte snapshot and then committing an eight-byte replacement.

For ordinary, stable memory, successful completion leaves:

- The first destination word equal to the first source word read.
- The second destination word equal to the second source word read.
- Both source words unchanged by this function.

The initial destination contents do not affect the decision or the values copied. No destination read is represented.

The access sequence above describes the supplied source representation. It does not independently establish machine-instruction ordering, memory barriers, or device-visible ordering.

### Non-copying path

When the condition is false, neither source location nor destination location is accessed by the represented body. The function returns without an explicit value.

In particular, the function does not read the source words speculatively at the source level and then discard them. An invalid implicit base need not cause a data access on this path, although caller correctness, address validity requirements, and machine-level speculative behavior are not established here.

### Preconditions and boundary conditions

For an ordinary successful copying execution:

- Both source words must be readable.
- Both destination words must be writable.
- The effective addresses must be valid under the execution environment’s address model.
- The addresses must satisfy whatever alignment requirements apply to the actual word accesses.

The representation contains no checks for nullness, region size, alignment, access permissions, or address overflow. It does not establish whether unaligned accesses are supported.

No wider object layout can be inferred from the highest accessed offset. The function requires access to the listed locations on the copying path; it does not prove allocation size or ownership of a surrounding object.

## Side effects

The only explicit data-memory writes are the two destination-word writes, and they occur only on the copying path.

There is no represented allocation, deallocation, ownership transfer, reference-count update, logging, callback, or explicit error-reporting operation. The memory base is used to address existing storage.

No ownership relationship is established. In particular, this function does not demonstrate that it owns the source words, owns the destination words, or is responsible for releasing their containing storage.

### Atomicity and concurrency

The two assignments do not establish a transactional update. There is no represented lock, fence, rollback mechanism, or combined eight-byte atomic operation.

If execution fails after the first destination write but before the second completes, the supplied representation provides no mechanism to restore the previous first destination value. Whether such a failure is possible, observable, or recoverable depends on the execution environment.

Concurrent modification is outside the established contract. Because the second source read follows the first destination write in the representation, the copied words are not guaranteed to form a mutually consistent snapshot in the presence of another actor.

The evidence also does not establish that each individual 32-bit access is atomic.

### Machine state

The incoming carry flag affects the result, but its post-return value is unspecified here. The representation does not justify assuming that carry, other condition flags, or unrelated registers are preserved.

## Dependencies

No function calls are present in the supplied body. There are therefore no evidenced opaque callee IDs or callee ownership contracts to enumerate.

The behavior depends on execution facilities rather than represented callees:

- Access to the incoming carry flag.
- Access to the implicit memory base.
- 32-bit arithmetic with the represented signed-overflow interpretation.
- Readable and writable memory at the conditional access locations.

No initialization routine, allocation routine, or caller protocol is identified. Those dependencies remain undocumented rather than being assigned invented function IDs.

## Acceptance examples

These examples are conditional acceptance criteria derived from static evidence, not reported test results.

### Comparison outcomes

Assume valid memory and no interference.

| Second input, signed | Fourth input, signed | Incoming carry | Required result |
|---:|---:|---|---|
| 12 | 11 | Set | Copy |
| 12 | 12 | Set | Do not copy |
| 12 | 12 | Clear | Copy |
| 12 | 13 | Clear | Do not copy |
| 0 | −1 | Set | Copy |
| −1 | 0 | Clear | Do not copy |
| −2,147,483,648 | −2,147,483,648 | Set | Do not copy |
| −2,147,483,648 | −2,147,483,648 | Clear | Copy |
| 2,147,483,647 | 2,147,483,647 | Clear | Copy |
| 2,147,483,647 | −2,147,483,648 | Set | Copy |
| −2,147,483,648 | 2,147,483,647 | Clear | Do not copy |

The final two cases distinguish the specified signed-overflow-aware decision from a simple wrapped-result sign check.

### Successful data replacement

Suppose the source words are 0x11223344 and 0xAABBCCDD, and the destination words initially contain 0xDEADBEEF and 0x01020304.

If the decision selects copying and all accesses succeed:

- Base +0x40 contains the word value 0x11223344.
- Base +0x44 contains the word value 0xAABBCCDD.
- The function itself has not written either source location.

These are word-value statements; they do not specify byte order in memory.

### Suppressed access

With equal second and fourth inputs and incoming carry set, no explicit source read or destination write is required by the represented body. Destination contents remain unchanged by this function.

### Unused explicit inputs

Holding the second input, fourth input, incoming carry, implicit base, and relevant memory state fixed, changing only the first or third explicit input does not change the represented decision or memory effects.

### Access trace

If the execution environment can observe the represented accesses without changing their semantics, a selected copy has the logical trace: first source read, first destination write, second source read, second destination write.

A trace consisting of both source reads followed by both destination writes would not match that source-level ordering. Actual emitted instruction ordering requires separate evidence.

## Uncertainties

The following items are **documented_only**:

- How callers establish the implicit base and incoming carry.
- Whether this body is independently callable or relies on surrounding control flow.
- The meaning and lifetime of the containing memory object.
- The concrete address-space width and address-wrap behavior.
- Required alignment, target byte order, and memory-region attributes.
- Fault handling and recovery for invalid reads or writes.
- Machine-level access ordering, speculation, and atomicity.
- Preservation of registers and condition flags.
- Concurrency guarantees or caller synchronization requirements.

An inferred intent is that the function conditionally replaces a stored pair of words after a comparison. The evidence does not establish whether that pair represents a candidate result, coordinates, a multiword number, or something else. Those interpretations impose no additional requirements.

## Evidence

The supplied static body establishes:

- Four explicit 32-bit inputs, with only the second and fourth used.
- An incoming carry-dependent decision using two signed-overflow-aware subtraction stages.
- An implicit incoming memory base.
- Two conditional 32-bit assignments at the documented offsets.
- The represented read/write sequence.
- No explicit return value and no represented function calls.

The strict-versus-inclusive signed comparison is a mathematical consequence of the represented condition, not a runtime observation.

No implementation was executed, no runtime tests were performed, and no external caller, binary, or memory-layout evidence was available. Claims about the function’s broader role, concrete calling convention, and execution environment remain outside the established contract.
