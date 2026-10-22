<!-- Trusted provenance: F-67b5db3dc22a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-67b5db3dc22a

## Purpose

F-67b5db3dc22a returns a 32-bit Boolean-like result based on an unsigned numeric input and two control bytes accessed relative to a supplied memory base.

For stable, readable memory, the result is **1** exactly when:

- The primary control byte is neither 0 nor 6; and
- The numeric input satisfies the condition selected by the secondary control byte:
  - Secondary byte equal to 3: the numeric input must be zero.
  - Secondary byte not equal to 3: the numeric input must be less than 91.

Every other normally completing case returns **0**.

The function establishes a predicate, not the application-level meaning of that predicate. Interpreting the result as authorization, readiness, validity, or another domain decision would be inferred intent.

## Interface

### Inputs

| Input | Representation | Behavioral role |
|---|---|---|
| Memory base | Integer-represented address-bearing value | Used to form the addresses of two control bytes |
| Numeric input | Unsigned 32-bit integer in the supplied representation | Compared with zero or the exclusive upper bound 91 |

The numeric input ranges from 0 through 4,294,967,295. Its threshold comparison is unsigned: a value with its high bit set is not treated as a negative number that could qualify as less than 91.

The memory base is not dereferenced at offset zero. The function accesses only the byte locations identified below.

The complete ABI, pointer width, and conversion rules between the integer-represented base and usable addresses are not established by the supplied evidence.

### Referenced memory layout

| Byte offset from base | Access width | Role |
|---|---:|---|
| 0x100, decimal 256 | 1 byte | Primary gate |
| 0x244, decimal 580 | 1 byte | Selects the numeric condition |

No surrounding structure layout, allocation size, object header, length field, or ownership metadata is established.

The control bytes participate only in equality or inequality comparisons against 0, 3, and 6. Their signedness does not change whether their bit patterns equal those constants. Other byte values have no separately established interpretation.

### Output

The return is a 32-bit scalar with two established normal values:

- **1:** the primary gate and selected numeric condition both pass.
- **0:** at least one required condition fails.

There is no separate status output, returned pointer, or returned allocated object. A normal return of 0 is a predicate result, not a diagnostic for inaccessible memory.

### Preconditions

Every address actually read must be valid and readable when the access occurs. Address formation must also be valid under the execution environment’s rules.

A readable region covering the base through offset 580 is sufficient for all paths, provided address formation is valid. That full contiguous extent is not itself a demonstrated requirement: the function does not access every intervening byte, and an early primary-gate rejection avoids the secondary location entirely.

The supplied function contains no explicit null check, bounds check, lifetime check, or recovery path for invalid memory. Ownership of the referenced region is unknown; it is not an established precondition that the caller owns it.

## Behavior

### Primary gate

The source-level evaluation first accesses the primary byte at offset 256 and compares it with 6.

- If that comparison finds 6, the function returns 0 without evaluating the remaining conditions.
- Otherwise, the primary location is evaluated again and compared with zero.
- If the second comparison finds zero, the function returns 0 without accessing the secondary byte.
- Only if both primary comparisons permit progress does evaluation reach the secondary location.

For stable memory, this reduces to the following gate:

| Stable primary byte | Outcome |
|---:|---|
| 0 | Return 0 |
| 6 | Return 0 |
| Any other byte value | Evaluate the secondary selector |

The order matters at the expression level: comparison against 6 precedes comparison against zero. A first comparison that rejects 6 prevents even the second primary comparison.

This establishes short-circuit evaluation, not a guaranteed number of physical memory transactions. The evidence does not establish volatile access, a memory-mapped-device contract, synchronization, or a concurrency model. A compiled implementation’s physical load count is therefore not specified here.

If an execution model permits separately observable accesses to the primary location to produce different values, each comparison concerns its own observed value. The function supplies no consistency check or snapshot mechanism. Supported behavior under concurrent mutation remains unresolved.

### Secondary selection

Once the primary gate passes, the function evaluates the byte at offset 580.

A secondary byte of **3** selects the zero-only condition:

- Numeric input 0 produces 1.
- Every nonzero numeric input produces 0.

Any secondary byte **other than 3** selects the unsigned threshold condition:

- Numeric inputs 0 through 90 produce 1.
- Numeric inputs 91 through 4,294,967,295 produce 0.

These alternatives are mutually exclusive. Selecting the zero-only condition does not permit a subsequent threshold check if the zero comparison fails. For example, numeric input 90 fails when the secondary byte is 3, despite being below 91.

### Evaluation constraints

The numeric input does not bypass the primary gate. In particular, zero still returns 0 when the stable primary byte is 0 or 6.

At the source-expression level, passing the primary gate leads to evaluation of the secondary byte before the selected numeric comparison. This remains the source-level sequence even when the numeric input makes both possible numeric conditions fail, such as an input of 91.

These statements describe the supplied expression’s evaluation. They do not promise distinct externally observable reads where the execution model permits optimization.

### Widths, arithmetic, and boundaries

The numeric input is compared without an explicit arithmetic transformation. The function does not increment, decrement, divide, or narrow it before making the decision.

The threshold is decimal **91**, expressed as 0x5B in the supplied representation. The comparison is strictly less than that threshold. Consequently:

- 90 is the largest accepted input under the broader condition.
- 91 is rejected.
- The unsigned maximum is rejected.
- Zero is accepted under either secondary selection, but only after the primary gate passes.

Address formation uses fixed offsets of 256 and 580 from the supplied base. The evidence does not establish portable overflow, wraparound, or pointer-provenance behavior for those operations. Invalid address arithmetic must not be treated as a defined way to reach another memory region.

## Side effects

The function contains no explicit memory store, global update, allocation, release, callback, or ownership-management operation.

**Ownership of the referenced memory is unknown.** The supplied function does not establish that the caller owns the region, that the function borrows it under any particular lifetime discipline, or that a broader ownership policy exists. The narrower supported observations are:

- No explicit ownership-management operation appears.
- The supplied base is not explicitly stored for later use.
- No alias into the region is returned.
- No direct callee is invoked to perform an ownership operation.

The explicit source-level memory accesses are limited to:

1. The primary byte for comparison against 6.
2. Potentially the same primary byte for comparison against zero.
3. Potentially the secondary byte, after the primary gate passes.

No explicit write ordering exists to specify because no writes appear.

Invalid memory can prevent normal completion. The function defines no intentional diagnostic output, exception conversion, or recovery behavior for such a failure. Any fault or other exceptional outcome depends on the execution environment and is not interchangeable with returning 0.

## Dependencies

No direct function calls appear in the supplied function. There are therefore no callee dependencies requiring opaque function IDs.

The function relies on environmental support for:

- Receiving the two scalar inputs.
- Forming addresses from the supplied base and fixed offsets.
- Reading the selected byte locations.
- Returning a 32-bit scalar.

These requirements do not establish additional application-level calls, ownership relationships, or guarantees about preservation of external state.

## Acceptance examples

These examples are conditional deductions from static evidence, not executed tests. Unless otherwise stated, the referenced bytes remain stable, every performed access is valid, and address formation is valid.

| Primary byte | Secondary byte | Numeric input | Required normal result | Basis |
|---:|---:|---:|---:|---|
| 0 | Any | 0 | 0 | Primary gate rejects zero |
| 6 | Any | 90 | 0 | Primary gate rejects six |
| 1 | 3 | 0 | 1 | Zero-only condition passes |
| 1 | 3 | 1 | 0 | Zero-only condition rejects nonzero |
| 1 | 3 | 90 | 0 | Threshold alternative is not selected |
| 1 | 0 | 0 | 1 | Non-3 selector accepts zero |
| 1 | 0 | 90 | 1 | Largest value below 91 |
| 1 | 0 | 91 | 0 | Exclusive upper boundary |
| 1 | 255 | 90 | 1 | Every non-3 selector uses the threshold |
| 255 | 2 | 90 | 1 | Primary values other than 0 and 6 pass |
| 1 | 2 | 2,147,483,648 | 0 | High-bit input remains unsigned and exceeds 90 |
| 1 | 2 | 4,294,967,295 | 0 | Unsigned maximum exceeds the threshold |

Additional path-sensitive acceptance conditions:

- **Primary byte 6:** source-level evaluation rejects the first comparison. Neither the second primary comparison nor the secondary access is evaluated.
- **Stable primary byte 0:** the first comparison permits progress, the second rejects zero, and the secondary access is not evaluated.
- **Passing primary byte and numeric input 91:** the source-level expression evaluates the secondary byte and returns 0 under either selected condition.
- **Unreadable secondary location with an excluding primary byte:** secondary unreadability alone does not prevent normal completion, because the expression does not evaluate that location on the rejecting path. All performed accesses and address formation must still be valid.
- **Unreadable secondary location with a passing primary byte:** no normal return value is promised, because the selected path requires that location to be readable.
- **Any normally completing execution:** the return is 0 or 1; no third normal result is established.

None of these examples requires that the caller own the memory. They require access validity on the evaluated path, while leaving ownership policy unspecified.

## Uncertainties

The following remain **documented_only**:

- **Domain meaning:** neither control byte nor either return value has an established application-level interpretation.
- **Ownership policy:** the evidence does not identify the owner of the region or establish acquisition, borrowing, transfer, or release rules.
- **Lifetime and provenance:** the origin, permitted lifetime, allocation extent, and valid base values are not supplied.
- **Complete ABI:** the exact calling convention, pointer width, and integer-to-address rules are not established.
- **Address overflow:** no portable wraparound or overflow policy is established for forming the two addresses.
- **Physical read count:** repeated source-level evaluation does not prove repeated machine-level loads.
- **Concurrent mutation:** no supported synchronization, atomicity, or interleaving contract is supplied.
- **Special memory:** behavior for volatile or memory-mapped locations is not established.
- **Exceptional execution:** the function does not define the fault type, recovery mechanism, or diagnostic behavior for invalid accesses.

These unknowns do not alter the stable-memory predicate under the stated validity conditions. They do limit claims about ownership, physical accesses, unusual address values, and exceptional execution.

## Evidence

**Established static evidence:** the supplied function representation contains two short-circuited primary-byte comparisons, a conditional secondary-byte comparison, an unsigned numeric comparison against 91, a zero comparison, and literal normal returns of 0 and 1. It contains no direct calls or explicit stores.

**Inferred intent:** describing the function as a predicate is consistent with its two-valued output. Any more specific domain purpose is unsupported.

**Unresolved behavior:** ownership, target-specific address semantics, physical memory-access details, concurrent mutation, and invalid-memory outcomes remain documented rather than assumed.

The acceptance examples are deductions from the supplied representation. No runtime tests, implementation execution, or machine-code verification have been performed. Trusted provenance is left to the publishing system.
