<!-- Trusted provenance: F-fe623eeba790; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fe623eeba790

## Purpose

F-fe623eeba790 conditionally performs either a bulk structural reset or repeated processing of individual entries between two caller-supplied positions.

The established behavior is structural, not semantic: the function compares position-like values through a dependency, may invoke a bulk-operation dependency, advances one caller-supplied position, invokes an entry-operation dependency, and writes a 32-bit value to a caller-provided output location.

**Inferred intent:** the layout and reset pattern are consistent with a container having a sentinel or header, a count, and iterator-like positions. The bulk path appears intended to handle a complete range efficiently. Neither “container,” “iterator,” nor “erase” is an established dependency contract.

No runtime tests have been performed.

## Interface

### Arguments

The supplied representation exposes four arguments:

| Argument | Represented width | Role established by this function |
|---|---:|---|
| Output location | 32-bit address of a writable 32-bit slot | Receives an initial structural pointer and may receive subsequent operation results. |
| Context address | 32 bits | Base address used to access structural state at byte offsets 16 and 20. |
| Mutable position address | 32-bit address of a readable position slot | Its first 32-bit word is read during iteration. The address is also passed to comparison and advancement dependencies. |
| Limit argument | 32 bits | Passed unchanged as the first argument of one comparison and the second argument of repeated comparisons. Its pointee layout, if any, is not established here. |

The supplied representation uses 32-bit integers and addresses. Address arithmetic adds byte offsets to a represented address; it does not establish portability to a different pointer width.

### Directly accessed layouts

The context has these directly accessed fields:

| Byte offset | Width | Established use |
|---|---:|---|
| 16 | 32 bits | Address of a structural object. Reloaded repeatedly rather than retained throughout the operation. |
| 20 | 32 bits | Tested for nonzero before the bulk path; set to zero on that path. |

The structural object addressed through context offset 16 has these directly accessed fields:

| Byte offset | Width | Established use |
|---|---:|---|
| 4 | 32 bits | Supplied to the bulk dependency, then cleared on the reset path. |
| 8 | 32 bits | Initially copied into temporary storage for comparison; set to the structural object's address on the reset path. |
| 12 | 32 bits | Set to the structural object's address on the reset path. |

No other fields are directly accessed by the function.

### Return value

The represented return value is 64 bits, formed by concatenating two 32-bit words:

- The upper word is a context/result temporary.
- The lower word is an output-address/entry temporary.

This is bit concatenation, not arithmetic addition and not a sign-extending conversion. The source does not establish a higher-level type for the pair, or whether a particular calling convention treats part of it as incidental register state.

The function also communicates through the output location. That memory result must not be assumed equivalent to the entire 64-bit return.

### Preconditions

For execution to complete without an invalid direct memory access:

- The output location must support a 32-bit write.
- The context fields at offsets 16 and 20 must be accessible whenever the corresponding path reads or writes them.
- Every structural address loaded for a subsequent direct field access must designate suitably accessible memory.
- If the iterative body executes, the mutable position address must support a direct 32-bit read.
- All dependency-specific validity and lifetime requirements must hold.

There are no explicit null, bounds, alignment, range-validity, or ownership checks. The function does not reject overlapping arguments. Aliasing can change later reads because writes and dependency calls occur between them.

## Behavior

### Initial output and comparison preparation

The function first reads the 32-bit structural address from context offset 16 and writes that value to the output location.

It then reads context offset 16 again and reads the 32-bit field at offset 8 of that structural object. This field value is placed in a temporary slot.

The distinction between these two context reads is observable. For example, if the initial output write aliases context offset 16, the later read observes the state after that write. The contract must not replace these operations with one cached structural address.

Before the first dependency call, two other temporary words are initialized:

- The entry/output-address temporary contains the output location's address.
- The context/result temporary contains the original context address.

The function calls F-626327d2116a with:

1. The mutable position address.
2. The address of the temporary containing the structural object's offset-8 field.

A zero result bypasses the bulk-path eligibility checks and enters the repeated-comparison phase. Any nonzero result proceeds to a second comparison.

### Bulk-path eligibility

After a nonzero first comparison result, the function reloads context offset 16 and puts that structural address into another temporary slot.

It then calls F-626327d2116a with:

1. The unchanged limit argument.
2. The address of this structural-address temporary.

The bulk path is selected only when both of the following hold:

- This second comparison returns a nonzero value.
- The 32-bit field at context offset 20, read after that comparison, is nonzero.

The context offset-20 field is not read for this condition when the second comparison returns zero. Its interpretation as a count is inferred; the actual condition is simply nonzero, not positive. A word with its high bit set also satisfies the condition.

If either condition fails, execution proceeds to the repeated-comparison phase. In particular, a zero context offset-20 field does not cause an immediate return.

### Bulk operation and reset

When eligible, the function calls F-c490d9fb041d with:

1. The original context address.
2. The 32-bit value read from offset 4 of the object currently addressed by context offset 16.

The dependency's return value is not used.

After that call returns, the function performs these full-expression effects in order:

1. Stores the current structural address into the offset-8 field of the structural object.
2. Stores zero into the offset-4 field of the structural object.
3. Stores the current structural address into the offset-12 field of the structural object.
4. Stores zero into context offset 20.
5. Reads context offset 16 into the context/result value used for this return.
6. Writes that value to the output location.
7. Returns the packed 64-bit result.

The structural address is reloaded from the context during these operations. The function does not preserve the pre-call address across F-c490d9fb041d. Consequently, mutations made by that dependency to context offset 16 affect the subsequent reset.

The supplied source establishes the ordering between the listed effects. It does not establish an exact machine-level ordering among multiple address loads within one assignment expression.

On this path, the lower return word remains the original output location's address; the upper return word is the final structural address read before the output write.

No iterative entry operation is performed after this reset.

### Repeated comparison and entry processing

All paths not taking the bulk return reach a comparison-controlled loop.

At the beginning of every iteration attempt, including the first, the function calls F-626327d2116a with:

1. The mutable position address.
2. The unchanged limit argument.

The loop terminates **only when the returned 32-bit value is exactly 1**. Zero, 2, negative represented values, and every other value continue into the body.

For each body execution, the function:

1. Reads and saves the first 32-bit word at the mutable position address.
2. Calls F-e6902067ce91 with the mutable position address.
3. Places the saved, pre-advancement word into the entry/output-address temporary.
4. Calls F-8d7ff2efc6b8 with:
   - The address of the context/result temporary.
   - The original context address.
   - The address of the entry/output-address temporary.
5. Writes the context/result temporary's current value to the output location.
6. Repeats the comparison.

Advancement therefore occurs before the entry-operation call, but the entry-operation temporary begins with the position word captured before advancement.

F-8d7ff2efc6b8 receives writable addresses of both temporaries. This function neither restores the context/result temporary to the original context address on each iteration nor preserves the entry temporary against dependency modification. The context/result temporary carries its current value into later calls. The entry temporary, by contrast, is replaced with a newly saved position word before each entry-operation call.

When the loop comparison returns exactly 1, the function returns the packed current temporary values. There is no final output write merely because the loop terminates.

### Zero-body return

If the first repeated comparison returns exactly 1:

- No advancement dependency is called.
- No entry-operation dependency is called.
- The function itself has performed only the initial output write.
- The upper return word is the original context address.
- The lower return word is the original output location's address.

Dependencies already called may have their own side effects; these statements do not imply that all surrounding memory is unchanged.

## Side effects

### Direct writes

The function directly writes:

- The initial structural address to the output location.
- On the bulk path, structural fields at offsets 8, 4, and 12, followed by zero at context offset 20 and a final output value.
- After each iterative entry operation, the current context/result temporary to the output location.

It also modifies temporary storage supplied by address to dependencies.

### Indirect mutation and ownership

The position and context structures may be modified by dependencies. The function does not establish that comparison dependencies are pure or that advancement affects only the position slot.

There is no explicit allocation or release operation in the supplied body. Whether F-c490d9fb041d or F-8d7ff2efc6b8 destroys entries, frees storage, changes ownership, or retains any supplied address is unknown. No such behavior may be assumed from their position in the call sequence.

The reset following F-c490d9fb041d requires the subsequently accessed context and structural storage to remain valid. A dependency that invalidates that storage without replacing it with accessible storage would make those later accesses invalid.

### Errors and completion

There is no explicit error return, exception handler, rollback, or recovery branch.

Comparison results are used as branch values rather than distinguished error statuses. In particular, a comparison result other than 1 in the loop causes another body execution.

Termination depends on dependency behavior. The function contains no iteration bound and no direct progress check. Invalid dependency outputs or failure to reach a comparison result of exactly 1 can cause repeated processing or invalid accesses.

## Dependencies

| Opaque ID | Established call contract at this call site |
|---|---|
| F-626327d2116a | Returns a 32-bit branch value. Receives two position-like arguments. The first two call sites use zero versus nonzero; the loop uses equality to exactly 1. Temporary slots supplied by address have no established preservation guarantee. |
| F-c490d9fb041d | Receives the original context address and the current structural offset-4 word. Called once on the bulk path before explicit structural reset writes. Return value unused. |
| F-e6902067ce91 | Receives the mutable position address once per loop-body execution, after its current first word has been saved. Return value unused. Advancement is inferred intent. |
| F-8d7ff2efc6b8 | Receives writable temporary addresses around the original context argument. Called after F-e6902067ce91 on each body execution. Its resulting context/result temporary is copied to the output location. Return value unused. |

Dependency implementations are unavailable. Their names here are identifiers, not semantic guarantees.

## Acceptance examples

These examples are conditional contracts, not observed test results. Except where mutation is stated explicitly, assume dependencies leave the example's referenced memory unchanged.

### Bulk reset selected

Given an accessible context whose structural field points to object H:

- The first and second eligibility comparisons return nonzero.
- Context offset 20 is nonzero.
- F-c490d9fb041d returns with the context still pointing to H.

Then:

- F-c490d9fb041d receives the original context and H's offset-4 value.
- H's fields at offsets 8 and 12 become H's address.
- H's field at offset 4 and context offset 20 become zero.
- The output contains H's address.
- No loop comparison, advancement, or entry-operation call occurs.
- The return has H's address in its upper word and the output address in its lower word.

### Bulk dependency replaces the structural object

Under the same eligibility conditions, suppose F-c490d9fb041d changes context offset 16 to an accessible object J.

The subsequent reset operates through the reloaded structural address. With no further interference, J receives the reset fields and the final output is J's address. Resetting only the previously observed H would violate this behavior.

### Immediate loop termination

Suppose the first eligibility comparison returns zero and the first loop comparison returns exactly 1.

The initial structural address is written to output. There is no second eligibility comparison, no offset-20 eligibility read, and no loop-body call. The returned words are the original context address above the original output address.

### Exact-one distinction

Suppose a loop comparison returns 2.

The body executes; 2 is not treated as loop termination. By contrast, 2 from either eligibility comparison satisfies that comparison's nonzero condition.

### One processed entry

Suppose the bulk path is bypassed, the first loop comparison returns zero, and the mutable position's first word is P.

F-e6902067ce91 runs before F-8d7ff2efc6b8, but the latter receives an entry temporary initialized to P. If it changes the context/result temporary to R and the entry temporary to Q, the function writes R to output. If the next comparison returns 1, the returned words are R above Q.

### Zero structural-state field

Suppose both eligibility comparisons return nonzero but context offset 20 is zero.

The bulk dependency and explicit reset writes do not occur. Execution still performs a loop comparison; any subsequent body behavior follows that result.

## Uncertainties

The following are **documented_only**:

- The semantic types of the context, positions, structural fields, and packed return.
- Whether comparisons represent equality, and whether successful comparisons always return exactly 1.
- Dependency allocation, deallocation, ownership-transfer, address-retention, and global-state behavior.
- Whether callers impose non-aliasing, synchronization, or stronger lifetime requirements.
- Whether the 64-bit return is a deliberate logical pair or an artifact of the represented calling convention.
- Behavior under invalid addresses, concurrent mutation, or dependency failure beyond the explicit control flow.

The sentinel/container interpretation is inferred intent. It must not replace the established offset-level behavior or justify assuming dependency effects.

## Evidence

Established static evidence consists solely of the supplied function representation:

- Four represented arguments and a concatenated 64-bit return.
- Direct 32-bit accesses at context offsets 16 and 20 and structural offsets 4, 8, and 12.
- An unconditional initial output write.
- Two nonzero-based eligibility comparisons.
- A conditional bulk call followed by ordered reset effects.
- An exact-1 loop termination condition.
- Capture-before-advancement ordering and writable temporary arguments to the entry operation.
- Distinct packed-return values for bulk, zero-body, and processed-entry paths.

No dependency implementation, caller contract, runtime trace, or test result was supplied. No runtime tests have been performed.
