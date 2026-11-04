<!-- Trusted provenance: F-1e2dde9a08d4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1e2dde9a08d4

## Purpose

F-1e2dde9a08d4 derives a four-component destination rectangle and passes it, together with a dependency-produced source rectangle, to another operation.

The established calculation centers the source rectangle’s width within the horizontal interval supplied by the caller and its height within the vertical extent obtained from a second object-associated value. The horizontal interval’s starting coordinate and the second rectangle’s vertical starting coordinate are not added to the result. Consequently, the calculated destination is relative to an implicit origin, rather than necessarily expressed in either input rectangle’s coordinate system.

A plausible intent is positioning or rendering one object relative to another, with a fixed final argument of 255. The supplied evidence does not establish rendering semantics, coordinate units, or the meaning of that constant.

## Interface

The function has three inputs and no explicit return value.

| Input | Established representation | Use |
|---|---|---|
| Context address | A 32-bit integer used as a memory address | Provides four-byte values at byte offsets 0xC0 and 0xB4 |
| Forwarded value | An uninterpreted four-byte value | Passed unchanged as the second argument to F-945301cd184f |
| Horizontal-bounds address | Address of an array of signed 32-bit integers | Elements 0 and 2 supply the horizontal interval endpoints |

The context input is address-like despite its integer representation. Its accesses use byte offsets, not integer-array indices.

The function directly consumes these context locations:

| Byte offset | Width | Role |
|---|---:|---|
| 0xC0 | Four bytes | Value supplied to the first F-c24767c70ad0 call; read again for the final operation |
| 0xB4 | Four bytes | Value supplied to the second F-c24767c70ad0 call |

For the horizontal-bounds input:

- Element 0 is read at byte offset 0.
- Element 2 is read at byte offset 8.
- Elements 1 and 3 are not directly used.
- No direct write to this input is shown.

The calculations use signed 32-bit integer values. The four-byte context fields and forwarded value have no established signed interpretation.

### Working rectangle layouts

For description, designate the first dependency-produced rectangle as **S**, the second as **T**, and the calculated destination rectangle as **D**. These are descriptive labels, not additional interface objects.

Each is interpreted as four consecutive four-byte components:

| Relative byte offset | Interpretation |
|---|---|
| 0 | First horizontal coordinate |
| 4 | First vertical coordinate |
| 8 | Second horizontal coordinate |
| 12 | Second vertical coordinate |

For S, all four components are subsequently read. For T, only the components at offsets 4 and 12 are subsequently read.

The layout is established by how the caller consumes the dependency outputs. The exact writable extent promised by F-c24767c70ad0 is not available. In particular, the supplied declaration of the second scratch region does not itself establish a complete sixteen-byte output object.

### Preconditions

Normal completion requires:

- The context address to permit both context-field reads, including the later reread at offset 0xC0.
- The horizontal-bounds address to permit reads of elements 0 and 2 after both preparatory dependency calls.
- Both F-c24767c70ad0 calls to establish the scratch components subsequently consumed.
- Dependency writes to remain within valid writable storage.
- Arithmetic results to have defined behavior under the target’s integer execution semantics.

There are no explicit checks enforcing these conditions.

## Behavior

### Ordered dependency preparation

The function first reads the four-byte context field at offset 0xC0 and invokes F-c24767c70ad0 with:

1. Writable local storage for S.
2. The field value just read.

After that call returns, it reads the four-byte context field at offset 0xB4 and invokes F-c24767c70ad0 with:

1. Separate writable local scratch storage for T.
2. The newly read field value.

These calls occur in this order. The field at offset 0xB4 is not captured before the first call.

The function does not inspect a return value from either call. Its subsequent computation depends on the scratch memory contents.

Unknown callees must not be assumed to preserve context fields, caller-owned memory, globals, or resources. Therefore, the second context-field read can observe a change caused by the first call, if such access is possible through dependency behavior.

### Geometry calculation

Let:

- \(S_x^0, S_y^0, S_x^1, S_y^1\) be S’s four components.
- \(T_y^0, T_y^1\) be T’s components at offsets 4 and 12.
- \(H^0, H^1\) be elements 0 and 2 of the horizontal-bounds input.

Define the extents:

\[
W = S_x^1 - S_x^0
\]

\[
H = S_y^1 - S_y^0
\]

\[
A = H^1 - H^0
\]

\[
B = T_y^1 - T_y^0
\]

For arithmetic within the defined signed range, let \(q(n)\) mean signed integer division of \(n\) by 2, truncating toward zero.

The destination components are:

\[
D_x^0 = q(A) - q(W)
\]

\[
D_y^0 = q(B) - q(H)
\]

\[
D_x^1 = W + D_x^0
\]

\[
D_y^1 = H + D_y^0
\]

The vertical starting component is calculated before the horizontal starting component. The horizontal ending component is then calculated, followed by the vertical ending component.

Important consequences:

- Division applies separately to the containing extent and source extent.
- The expression is not generally equivalent to dividing their difference by two.
- The caller’s horizontal starting coordinate contributes only through the interval’s extent.
- T’s vertical starting coordinate likewise contributes only through its extent.
- S’s original coordinates are retained in the source rectangle passed to the final dependency; the function does not normalize S to the origin.
- No clipping, coordinate ordering, or minimum-size adjustment is performed.
- A source extent larger than its containing extent can produce a negative destination starting coordinate.
- Reversed endpoints produce negative extents rather than being exchanged.

Repeated source-extent expressions occur in the supplied calculation. No intervening dependency call is shown during this arithmetic. The evidence does not establish a precise machine-level operand-read order within individual arithmetic expressions.

### Final operation

After constructing D, the function rereads the four-byte context field at offset 0xC0 and invokes F-945301cd184f with five arguments, in this order:

1. The newly read context-field value.
2. The original forwarded four-byte value.
3. The address of S.
4. The address of D.
5. The numeric constant 255.

The final context value is not necessarily identical to the value used to obtain S. Either preparatory call could have changed it through behavior not visible here.

The function then returns without an explicit result or status.

## Side effects

### Directly established effects

The function:

- Reads the context field at offset 0xC0 twice.
- Reads the context field at offset 0xB4 once, between those two reads.
- Reads horizontal-bounds elements 0 and 2 after both preparatory calls.
- Supplies local writable storage to F-c24767c70ad0 twice.
- Writes the four components of D into local storage.
- Supplies addresses of S and D to F-945301cd184f.

No direct assignment to context memory, horizontal-bounds memory, or a named global is shown. This does not imply that the dependencies leave such state unchanged.

### Ownership and lifetime

No direct allocation, release, ownership transfer, or reference-count operation is visible.

S and D occupy function-local storage. Their addresses are available to the final dependency during its invocation. The evidence does not establish whether that dependency copies the values, modifies them, retains their addresses, or performs resource-management operations.

Local storage does not provide a valid post-return lifetime. Any dependency behavior requiring later access would need an independently valid mechanism, such as copying the data; no such mechanism is established here.

The four-byte context values may be handles, addresses, or other tokens. Their ownership and validity rules are unknown.

### Errors and exceptional outcomes

There is no explicit:

- Null-address handling.
- Bounds validation.
- Dependency-failure check.
- Overflow check.
- Recovery branch.
- Error return.

If a dependency fails to establish required scratch values, later geometry is not defined by this evidence. If an invalid access, abnormal dependency termination, or arithmetic fault prevents completion, the function provides no visible recovery path.

## Dependencies

### F-c24767c70ad0

Called twice, sequentially.

The observed call shape is:

- First argument: address of local writable storage.
- Second argument: an uninterpreted four-byte context-field value.

The caller expects the first invocation to establish all four S components. It expects the second to establish at least T’s second and fourth components.

Unknowns include:

- Actual parameter types and complete output layout.
- Whether all expected components are always initialized.
- Failure signaling.
- Global or externally reachable state changes.
- Resource ownership effects.
- Whether either invocation can affect memory used by the other.

### F-945301cd184f

Called once after destination calculation.

The observed argument sequence is:

1. Four-byte context value reread at offset 0xC0.
2. Forwarded four-byte input.
3. Address of S.
4. Address of D.
5. Constant 255.

The caller neither interprets a result nor performs explicit cleanup afterward. The dependency’s operational meaning, mutation behavior, ownership effects, and error semantics are undocumented here.

## Acceptance examples

These examples are conditional specifications, not observed executions. Unless stated otherwise, assume valid memory, completed dependency calls, stable relevant values during arithmetic, and no signed overflow.

### Even extents

Suppose S is \((10,20,30,30)\), the caller’s horizontal endpoints are 100 and 200, and T’s vertical endpoints are 7 and 67.

The source width is 20, its height is 10, the available horizontal extent is 100, and the available vertical extent is 60.

The required destination is:

\[
D=(40,25,60,35)
\]

The final call receives the original S, not an origin-normalized rectangle. Neither 100 nor 7 is added to D.

### Separate truncation matters

Suppose the available horizontal extent is 5 and the source width is 2.

The horizontal start is:

\[
q(5)-q(2)=2-1=1
\]

The horizontal end is 3.

If the available extent is instead 4 and the source width is 3, the start is:

\[
q(4)-q(3)=2-1=1
\]

Replacing the established calculation with \(q(4-3)\), which yields 0, would change behavior.

### Oversized source

Suppose the available horizontal extent is 4 and the source width is 8.

The destination horizontal endpoints are −2 and 6. The function does not clamp them to the available interval.

### Negative odd extent

Suppose the caller’s horizontal endpoints are 10 and 5, and the source width is 2.

The available extent is −5. With truncation toward zero:

\[
D_x^0=-2-1=-3,\qquad D_x^1=-1
\]

Using floor division would produce a different result and would not match the supplied signed-division expression.

### Context mutation between calls

Suppose the initial offset-0xC0 field contains token A. The preparatory calls establish S but leave that field containing token B.

The first preparation receives A. The final dependency receives B. An implementation that captures A once and reuses it for the final call would not preserve the established read sequence.

Likewise, a mutation of the offset-0xB4 field during the first preparatory call affects the value supplied to the second call.

### Boundary arithmetic

If a coordinate subtraction exceeds the signed 32-bit range, this contract does not prescribe a wrapped result. There is no visible overflow handling, but unchecked source expressions alone do not establish portable wrapping semantics.

## Uncertainties

The following remain **documented_only**:

- The meanings of the two context fields and forwarded value.
- The semantic meaning of the final constant 255.
- Whether the rectangle components represent pixels, another coordinate unit, or non-geometric values.
- Whether second coordinates are inclusive or exclusive boundaries.
- The complete storage contract of F-c24767c70ad0, especially for the second scratch region.
- Dependency failure modes, memory retention, resource management, and global mutations.
- Exact behavior for signed overflow and address-arithmetic overflow.
- Machine-level operand-read ordering within individual expressions.
- Whether external mutation or concurrency is permitted by the surrounding system.

The centering interpretation is inferred intent. The extent differences, separate divisions, argument order, and context reread are established static behavior.

## Evidence

This contract is based solely on the supplied function representation and dependency-ID mapping.

Static evidence establishes:

- Three inputs and no explicit return value.
- Four-byte context reads at offsets 0xC0 and 0xB4.
- Two ordered calls to F-c24767c70ad0.
- Signed integer extent calculations with separate division by two.
- A four-component destination layout.
- A later reread at offset 0xC0.
- One call to F-945301cd184f with S, D, and constant 255.

No dependency implementation, runtime trace, or runtime test was provided or performed. Acceptance examples are arithmetic consequences under their stated conditions, not execution evidence.
