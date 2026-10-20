<!-- Trusted provenance: F-41ba7d9bb9e3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-41ba7d9bb9e3

## Purpose

F-41ba7d9bb9e3 examines successive 16-bit elements from an initial position. It advances past elements whose numeric values are decimal 32 or decimal 9, stopping when its working position equals a supplied endpoint or when an examined element fails both comparisons.

**Inferred intent:** Skip a prefix of space and horizontal-tab values. The established behavior is numeric comparison, not general whitespace recognition or an encoding-aware text operation.

The interface exposes no scan result. The function returns no value, does not write an advanced position through an output parameter, and does not modify the examined elements. For valid, stable ordinary memory, its source-level activity consists of pointer comparisons, element evaluations, and advancement of a private working position.

## Interface

The function accepts three arguments in this order:

| Argument | Represented width or type | Established role |
|---|---|---|
| First argument | Opaque 32-bit value | Unused by the supplied body. |
| Initial position | Pointer to signed 16-bit elements | Starting position for examination. |
| Endpoint | Pointer to signed 16-bit elements | Position at which the function returns without examining an element. |

The return type is void. There is no returned pointer, count, status, or result buffer.

The first argument is not read, dereferenced, or used in a decision. Its interpretation beyond occupying a represented 32-bit argument is unspecified. No particular numeric or pointer interpretation is needed to describe this body.

Both pointer arguments are passed by value. The function changes its own working position, not a caller-held pointer variable. The endpoint is not advanced.

### Element representation and arithmetic

Each examined element is represented as a signed 16-bit value, with a representable range of −32768 through 32767. The two advancing values, 32 and 9, are exactly representable. Every other stable element value causes examination to stop, including zero, negative values, and positive values outside that two-value set.

An advance moves forward by one 16-bit element: a two-byte displacement in the represented layout. The body does not combine adjacent elements, decode byte sequences, or calculate a range length.

The 32-bit first argument does not constrain pointer width or advancement arithmetic. The body does not establish the target pointer width, address-space size, or any supported address-wraparound behavior.

### Preconditions

For defined execution:

- Each element actually evaluated must support a valid 16-bit read under the execution environment’s accessibility, alignment, and pointer-validity rules.
- Each advancement must be valid under that environment’s pointer-arithmetic rules.
- Pointer values must support the comparisons performed by the body.
- A stable-memory interpretation requires that the examined values not change between their source-level evaluations.

A conventional finite range consists of valid contiguous elements with an endpoint reachable by successive one-element advances. The endpoint may be a valid one-past position; it need not designate a readable element because it is not dereferenced when reached.

The body does not validate these preconditions. In particular, it does not check ordering, calculate the distance between the pointers, or reject an endpoint that is not reachable.

If the initial position equals the endpoint, the function takes a no-element-access path. This does not establish that arbitrary invalid pointer representations are acceptable to the surrounding calling interface.

## Behavior

### Endpoint precedence

Before examining an element at any position, the function compares the working position with the endpoint.

Equality causes immediate return. No element evaluation occurs at that position. Thus, for an ordinary contiguous range, the endpoint is exclusive.

Only equality matters. An endpoint that is numerically below the initial position is not treated as a lower bound, and an endpoint beyond a stopping element is not consulted through a distance calculation.

### Element recognition

When the working position differs from the endpoint, the function first evaluates the current element for comparison with decimal 32.

If that evaluation equals 32, the working position advances by one element. Processing then resumes with another endpoint comparison.

If the first evaluation does not equal 32, the function evaluates the element at the same position for comparison with decimal 9.

- If that second evaluation equals 9, the working position advances by one element.
- If it does not equal 9, the function returns without advancing past that position.

For stable ordinary memory, this is equivalent to advancing across the maximal initial sequence whose elements belong to the set {32, 9}, subject to the endpoint.

No other value receives special treatment. Zero stops the scan because it fails both comparisons, not because the body implements a separate zero-termination rule. Decimal 10 and decimal 13 also stop the scan; there is no general-purpose whitespace classification.

### Order of reads

The source-level evaluation order is established:

1. Compare the working position with the endpoint.
2. Only when unequal, evaluate the current element against 32.
3. Only when that comparison is unequal, evaluate the same element position against 9.
4. Return or advance according to those results.

Consequently, a position matching 32 on its first evaluation requires one source-level element evaluation. A position not matching 32 on its first evaluation requires a second evaluation, whether the function then advances or returns.

These are source-level evaluations, not an asserted count of physical machine loads. The pointer is not represented as volatile. Compiler reuse of a loaded value, executable instruction selection, hardware access granularity, and speculative activity are not established.

For memory that changes between evaluations, the stable-memory set-membership description is insufficient. The supplied body provides no synchronization or snapshot guarantee. Defined behavior under such changes depends on execution-environment rules not supplied here.

### Termination and invalid ranges

The function returns upon either:

- Equality between the working position and the endpoint; or
- Failure of both element comparisons at a position examined before that equality occurs.

For a valid finite contiguous range, it cannot perform a source-level element evaluation at or beyond the endpoint.

If the endpoint is not reached, advancement continues as long as the comparisons permit it. There is no independent iteration limit, length counter, pointer-order check, or recovery path for invalid advancement.

An unreachable endpoint does not itself force failure before a stopping value. If a stopping value is encountered while every preceding operation remains valid, the function returns there. Conversely, an unreachable endpoint with only advancing values provides no established safe termination guarantee.

## Side effects

The supplied body establishes the following effects:

- Element reads along the examined prefix.
- Examination of a stopping element when one is encountered before the endpoint.
- Changes to the function’s private working position.
- Return without a result value.

It contains no explicit:

- Writes through either pointer.
- Global-state reads or writes.
- Function calls.
- Allocation or deallocation.
- Ownership-transfer or other ownership-management operation.
- Retention of either pointer in persistent or externally visible storage.
- Logging, diagnostic reporting, or error-status production.

**Storage ownership is unspecified.** The supplied body does not establish who owns the examined storage, who may release it, or whether it is owned storage at all. The absence of allocation, deallocation, pointer retention, and ownership-management operations must not be interpreted as proof that the caller owns the storage.

For valid ordinary memory, the examined contents remain unchanged by this body. Device-backed memory may attach effects to reads, but such behavior is not established by the supplied representation.

Invalid access or invalid advancement may have consequences determined by the language implementation or execution environment. The function supplies no explicit handling for those violations. No particular fault, exception, diagnostic, or successful return is promised.

## Dependencies

No function dependencies appear in the supplied body. There are therefore no dependency function IDs to enumerate.

Execution depends on the surrounding environment’s rules for:

- Pointer representation and equality.
- Access to signed 16-bit elements.
- Alignment and memory accessibility.
- One-element pointer advancement.
- Evaluation and optimization of non-volatile reads.

The body does not establish calling convention, pointer width, byte order, or exact alignment requirements.

No assumptions about a callee preserving globals, retaining pointers, or managing memory are needed: no callee is invoked.

## Acceptance examples

All examples below are conditional on valid pointer operations and stable ordinary memory. Numbers denote signed 16-bit element values, not byte sequences. References to a final position describe internal behavior, not an exposed result.

### Equal initial position and endpoint

**Given:** The two pointer arguments compare equal.

**Required behavior:** Return without evaluating an element.

**Observable boundary:** No value validation or read from the designated position occurs. Changing the unused first argument does not change this path.

### Immediate stopping element

**Given:** The initial position differs from the endpoint, and the first element is 65.

**Required behavior:** Evaluate that position against 32 and then against 9, and return without advancing.

**Excluded behavior:** Examining the next element, modifying the first element, or returning a stopping-position result.

### Mixed advancing prefix followed by a stop

**Given:** The range contains 32, 9, 32, and 65, with the endpoint immediately after the fourth element.

**Required behavior:** Advance across the first three elements, examine the fourth, and return without advancing past it.

The 32-valued positions take the first-comparison matching path. The 9-valued position requires the second comparison. No position after the 65-valued element is examined.

### Advancing values fill the range

**Given:** The range contains only 32 and 9, with a reachable endpoint immediately after its last element.

**Required behavior:** Advance across the entire range, then return upon pointer equality. Do not evaluate an element at the endpoint.

**Interface consequence:** The caller receives no count or advanced pointer.

### Nonaccepted whitespace-like values

**Given:** The first element is decimal 10 or decimal 13, and the initial position differs from the endpoint.

**Required behavior:** Return at that first position without advancing.

A replacement that also skips these values would change the established behavior.

### Zero and signed boundaries

**Given:** The first element is zero, −32768, −1, or 32767, and the initial position differs from the endpoint.

**Required behavior:** Return at that element after both comparisons fail.

Neither signedness nor a zero-terminated-text convention introduces an additional advancing value.

### Unreadable storage begins at the endpoint

**Given:** All elements before a reachable endpoint are valid and advancing values. The pointer operations remain valid, but storage beginning at the endpoint is not readable.

**Required behavior:** Return when the working position equals the endpoint, without a source-level element evaluation there.

This example establishes endpoint-before-read ordering, not a guarantee about speculative hardware activity.

### Stopping before an unreachable endpoint

**Given:** The endpoint is not on the advancement path, but a stopping value is encountered before any invalid access or advancement.

**Required behavior:** Return at that stopping value.

**Limit:** This does not establish safe behavior if all subsequent values permit advancement and the endpoint remains unreachable.

### Storage without caller ownership

**Given:** The caller has valid read access to suitable storage but does not own it; all pointer operations satisfy the stated preconditions.

**Required behavior:** Perform the same examination and return behavior. No allocation, release, retention, or ownership-transfer operation is introduced.

Caller ownership is not an established precondition.

## Uncertainties

The following remain **documented_only**:

- The purpose and external meaning of the unused 32-bit argument.
- Why an internally advanced position is not exposed to callers.
- Whether a surrounding operation gives significance to these reads.
- The intended text encoding or application-level meaning of 32 and 9.
- Storage ownership, lifetime-management responsibility, and any external access permissions beyond those required for valid execution.
- Target pointer width, calling convention, byte order, and exact alignment requirements.
- Exact physical load counts and hardware-level memory-access behavior.
- Behavior under concurrent mutation, device-memory semantics, invalid pointers, or invalid pointer advancement.
- Whether the supplied representation omits relevant facts about the surrounding executable or environment.

No ownership claim, hidden output, synchronization guarantee, or error-recovery behavior is inferred to fill these gaps.

## Evidence

**Established static evidence:** The supplied body compares the working position with the endpoint before element access; compares the current signed 16-bit element against 32 and, conditionally, 9; advances by one element on matching paths; and otherwise returns. Its interface includes one unused 32-bit argument, two element pointers, and no return value. The body contains no calls, explicit global accesses, writes through supplied pointers, allocation, deallocation, pointer retention, or ownership-management operations.

**Inferred intent:** Skipping a prefix of space and horizontal-tab values. This interpretation does not broaden the recognized numeric set or establish an encoding contract.

**Verification boundary:** This contract is based solely on the supplied source representation. No runtime tests, executable inspection, or external dependency analysis have been performed. Acceptance examples specify conditional expected behavior; they are not reported test results.
