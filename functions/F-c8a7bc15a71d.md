<!-- Trusted provenance: F-c8a7bc15a71d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c8a7bc15a71d

## Purpose

F-c8a7bc15a71d computes and returns a 32-bit integer derived from four fields in a caller-supplied object and the results of two dependencies.

The established behavior has two components:

- A product of two object fields, captured before either dependency is called, is divided by eight with rounding toward zero.
- A second dependency supplies an integer contribution. Its arguments are another product of object fields and a selector determined by whether the first dependency returned zero.

The return value is the sum of those components, subject to the arithmetic qualifications below.

The source does not establish the semantic meaning of the object fields or the returned quantity. Interpreting the result as a size, allocation requirement, stride, capacity, or resource count would be inferred intent, not an established contract.

## Interface

### Input

The function accepts one argument represented as a 32-bit integer. It uses that argument as the base address of an object.

The object must support the following four 32-bit signed integer reads:

| Byte offset from object base | Role in this function | Read phase |
|---|---|---|
| `0x10` | Second factor of the product passed to F-870c4fcf7295 | After F-8e65ca9c767c returns |
| `0x18` | One factor of the product subsequently divided by eight | Before F-8e65ca9c767c is called |
| `0x1c` | Saved first factor of the product passed to F-870c4fcf7295 | Before F-8e65ca9c767c is called |
| `0x20` | Other factor of the product subsequently divided by eight | Before F-8e65ca9c767c is called |

Each listed field occupies four bytes. Thus, the highest explicitly accessed field occupies offsets `0x20` through `0x23`. This establishes the accessed extent, not the complete object size.

The source does not establish an object type, field names, byte order, packing convention, or alignment tolerance. A compatible object representation and valid memory accesses are preconditions.

### Output

The function returns one 32-bit integer. There is no explicit output pointer or direct object-field write in the visible function body.

For executions whose signed arithmetic is representable, define:

- **Early product:** the value read at offset `0x18` multiplied by the value read at offset `0x20`, both read before the first dependency call.
- **Call product:** the value read at offset `0x1c` before the first dependency call multiplied by the value read at offset `0x10` after that call.
- **Selector:** `1` if F-8e65ca9c767c returns zero; otherwise `2`.
- **Dependency contribution:** the integer returned by F-870c4fcf7295 when called with the call product and selector.

The result is:

**early product divided by eight, rounded toward zero, plus the dependency contribution.**

This mathematical description is not an arbitrary-precision guarantee. Intermediate products and the final addition use 32-bit integer representations; overflow is addressed under **Uncertainties**.

### Preconditions

A well-defined ordinary execution requires:

- A base address suitable for the four field reads.
- Valid address arithmetic for the referenced offsets.
- Memory that remains readable at the relevant phases.
- Satisfied preconditions of both dependencies.
- Representable signed arithmetic, unless the execution environment supplies a separately established overflow model.

The function performs no visible null check, object-size check, field-range check, or dependency-error check.

## Behavior

### Initial capture

The function first reads and retains the signed 32-bit field at offset `0x1c`.

It then reads the signed 32-bit fields at offsets `0x18` and `0x20` and computes their product. That product is retained across the first dependency call.

The visible statement ordering establishes that the `0x1c` read precedes the early-product computation, and that all three reads precede F-8e65ca9c767c. It does not establish an observable ordering between the two operand reads within the early-product expression.

This distinction matters if the object can change during execution. The early product is not recomputed after either dependency call.

### First dependency and selector

The function calls F-8e65ca9c767c with the original object-base argument.

Its returned integer is tested only for equality with zero:

- Zero selects the 32-bit value `1`.
- Any nonzero value selects the 32-bit value `2`.

Negative and positive nonzero results are treated identically. No magnitude, sign, bit pattern, or apparent error convention changes that choice.

There is no visible early return following this call. If it returns normally, execution proceeds to the second dependency.

### Late field read and second dependency

After F-8e65ca9c767c returns, the function reads the signed 32-bit field at offset `0x10`.

It multiplies this newly read value by the previously retained value from offset `0x1c`. It then calls F-870c4fcf7295 with:

1. The resulting 32-bit integer product.
2. The selector, represented as a 32-bit value of either `1` or `2`.

The timing is part of the contract:

- A change to offset `0x10` made by F-8e65ca9c767c can affect the first argument to F-870c4fcf7295.
- A change to offset `0x1c` made by F-8e65ca9c767c does not replace the already retained factor.
- Changes to offsets `0x18` or `0x20` made by F-8e65ca9c767c do not replace the already computed early product.

These statements describe the visible data flow. They do not assume that F-8e65ca9c767c actually modifies the object.

### Division and return

The retained early product is adjusted and shifted to implement signed division by eight with rounding toward zero, under the signed-shift interpretation shown by the supplied representation:

- For a nonnegative early product, the adjustment is zero.
- For a negative early product, the adjustment is seven.
- The adjusted signed value is shifted right by three positions.

The adjustment is derived using 32-bit signed and unsigned operations. A signed right shift by 31 produces the sign-dependent value; conversion to unsigned followed by a right shift by 29 yields either zero or seven. This is not an unsigned division of the original product.

Consequently, small negative products between `−7` and `−1` contribute zero, rather than `−1`. A product of `−8` contributes `−1`; a product of `−9` also contributes `−1`.

The function adds this quotient to the integer returned by F-870c4fcf7295 and returns the sum.

The function does not validate, clamp, normalize, or otherwise interpret the second dependency’s return value. A negative return participates directly in the final addition.

### Control-flow boundaries

On the visible normal path:

- F-8e65ca9c767c is called exactly once.
- F-870c4fcf7295 is called exactly once, after the first dependency returns.
- There are no loops or retries.
- There is one final returned integer.

Faults, exceptions, nonlocal exits, or nonreturning behavior inside a dependency can prevent subsequent steps. The source supplies no recovery behavior for those cases.

## Side effects

The visible body directly reads object memory but performs no direct memory writes.

That observation does not establish that the complete operation is read-only. Both dependencies may have side effects not described by the supplied material.

In particular:

- F-8e65ca9c767c receives the object-base argument and may affect object memory, globals, resources, or other external state; its actual behavior is unknown.
- F-870c4fcf7295 receives two integer values. Its effects on globals, resources, or memory reachable by other means are unknown.
- Neither dependency may be assumed to preserve global state.
- No visible operation establishes ownership transfer, allocation, retention, or release.
- The absence of a direct release in this function does not establish that a dependency releases nothing.

The late read at offset `0x10` requires that location to remain valid after F-8e65ca9c767c returns. If that dependency invalidates the storage, the wrapper provides no visible protection.

The source provides no synchronization, atomic-access guarantee, volatile-access guarantee, or coherent-snapshot guarantee. Concurrent modification is therefore outside the established behavioral guarantees.

## Dependencies

| Opaque function ID | Arguments supplied | Return handling | Established ordering |
|---|---|---|---|
| F-8e65ca9c767c | Original object-base argument | Compared with zero to select `1` or `2` | After the three early field reads; before the `0x10` read |
| F-870c4fcf7295 | Call product; 32-bit selector | Added directly to the early product’s truncated quotient | After F-8e65ca9c767c returns and the late field is read |

Neither dependency’s implementation or independent contract is supplied. Their argument meanings, valid ranges, failure conventions, ownership behavior, and side effects remain unknown.

In particular, a zero return from F-8e65ca9c767c must not be labeled success or failure on this evidence. Likewise, a negative return from F-870c4fcf7295 must not be labeled an error without a separate dependency contract.

## Acceptance examples

These are conditional acceptance examples derived from static evidence, not reports of executed tests. Each assumes valid memory, normal dependency returns, the stated field values at their respective read times, and representable arithmetic.

### Zero result selects one

Suppose the early fields are:

- Offset `0x18`: `8`
- Offset `0x20`: `3`
- Offset `0x1c`: `5`

If F-8e65ca9c767c returns zero and the later `0x10` read yields `4`, F-870c4fcf7295 must receive `20` and `1`.

If that dependency returns `10`, the function returns `13`: the early product is `24`, its quotient is `3`, and the dependency contribution is `10`.

### Every nonzero result selects two

Using the same field values, a first-dependency return of `−1`, `1`, or another nonzero integer must select `2`.

If the late field remains `4`, the second dependency receives `20` and `2`. The first dependency’s nonzero magnitude does not otherwise enter the calculation.

### Negative division truncates toward zero

Suppose the early factors are `−3` and `3`, and the second dependency returns `20`.

The early product is `−9`; its quotient is `−1`. The final result is `19`, not `18`.

If the early factors instead produce `−7`, the quotient is zero and the final result is `20`.

### Saved and late-read fields differ

Suppose the initial values are:

- Offset `0x18`: `8`
- Offset `0x20`: `2`
- Offset `0x1c`: `3`
- Offset `0x10`: `4`

Suppose F-8e65ca9c767c returns nonzero and, before returning, changes:

- Offset `0x18` to `80`
- Offset `0x20` to `20`
- Offset `0x1c` to `30`
- Offset `0x10` to `5`

F-870c4fcf7295 must receive `15` and `2`, not `150` and `2`. The retained factor is `3`, while the late-read factor is `5`.

The early quotient remains `2`, from the previously computed product `16`. If the second dependency returns `7`, the final result is `9`.

This example is conditional on the first dependency actually making those changes and leaving the late-read location valid.

### Negative dependency contribution remains arithmetic input

If the early product is `16` and F-870c4fcf7295 returns `−5`, the result is `−3`.

There is no visible branch that treats `−5` as a failure code.

## Uncertainties

The following items are **documented_only**; the supplied evidence does not resolve them.

- **Signed overflow:** Both multiplications and the final addition can exceed the signed 32-bit range, `−2,147,483,648` through `2,147,483,647`. The source representation alone does not establish whether such executions wrap, trap, or lack defined language-level behavior. No overflow-result guarantee is made.
- **Signed right shifts:** The quotient characterization relies on sign-propagating signed right shifts, as indicated by the supplied arithmetic representation. Portability to an environment with different signed-shift semantics is not established.
- **Address model:** The argument is represented as a 32-bit integer used for addressing. Pointer provenance, address-space rules, and address-overflow behavior are not supplied.
- **Object representation:** Endianness, alignment requirements, complete extent, field meanings, and construction rules are unknown.
- **Dependency effects:** Neither dependency’s global-state behavior, memory ownership behavior, resource effects, or failure convention is established.
- **Invalid inputs:** There is no specified recovery result for null, unreadable, misaligned, stale, or otherwise invalid object addresses.
- **Concurrent access:** No safe behavior is established for unsynchronized modification of the object during execution.
- **Semantic intent:** A calculation involving products and division by eight may suggest a size conversion, but no such interpretation is established.

## Evidence

This contract is based exclusively on the supplied static function representation and the supplied mapping to opaque dependency IDs.

Established static evidence includes four 32-bit field reads at the listed offsets, retention of three early-read values through their saved factor or product, two ordered dependency calls, a zero/nonzero selector, sign-dependent rounding before division by eight, and a final integer addition. The visible body contains no direct memory write, input validation, retry, or explicit error-handling path.

The mathematical quotient description is derived from the displayed 32-bit adjustment and shift operations under the stated signed-shift assumption. Dependency semantics and overflow behavior are not inferred beyond the documented limits.

No runtime tests have been performed.
