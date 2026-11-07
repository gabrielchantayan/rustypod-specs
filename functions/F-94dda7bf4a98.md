<!-- Trusted provenance: F-94dda7bf4a98; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-94dda7bf4a98

## Purpose

F-94dda7bf4a98 updates selected fields in a caller-supplied state object according to a selector and a signed value. It supports:

- Setting a nonnegative scalar.
- Setting either endpoint of a pair and, when the endpoints differ, updating a derived value and its scale marker.
- Selecting one of three byte-sized modes.
- Recording a rejection in a first-error field without overwriting an existing nonzero error.

The observable contract is primarily state mutation. The function has no explicit return value.

The endpoint calculations suggest a scaled numerical representation, but the meaning of the endpoints, derived value, and mode is not established. They are described by their observable storage and transitions rather than assigned domain-specific names.

## Interface

### Inputs

The function receives three inputs:

1. **State address:** a pointer to mutable storage.
2. **Selector:** an integer identifying the requested operation.
3. **Value:** an integer interpreted according to the selector.

The supplied representation uses integer-word accesses and signed arithmetic. This contract describes those words as 32-bit and byte accesses as 8-bit. The target’s complete type and ABI declarations are unavailable; the width interpretation is addressed under **Uncertainties**.

### State layout

Offsets below are byte offsets from the state address.

| Offset | Access width | Contract role |
|---|---:|---|
| `0x000` | 32 bits | First-error field |
| `0x8B8` | 8 bits | Mode |
| `0x8BC` | 32 bits | First endpoint |
| `0x8C0` | 32 bits | Nonnegative scalar |
| `0x8C4` | 32 bits | Second endpoint |
| `0x8C8` | 32 bits | Derived value |
| `0x8CC` | 8 bits | Derived-value scale marker |

Byte writes affect only the addressed byte. In particular, updating the mode or scale marker does not, by this function’s explicit operations, replace the other bytes of the surrounding word.

No other state fields are explicitly accessed in the supplied body. Dependencies may have additional effects that are not shown.

### Outputs and ownership

There is no explicit returned status. Results are communicated through state writes and possible dependency side effects.

The function does not explicitly allocate, free, retain, or transfer ownership of the state storage. Its dependency calls cannot be assumed to preserve memory or ownership merely because no state pointer is explicitly shown in those calls.

### Preconditions

For every path that accesses the state, the caller must provide suitably aligned, readable or writable storage covering the fields used on that path. A state object covering all listed fields requires accessible storage through offset `0x8CC`, inclusive.

There is no explicit pointer validation, object-size validation, locking, or recovery from an invalid memory access. Some paths return without dereferencing the state, but that does not establish a general null-pointer guarantee.

## Behavior

### Selector `0xB62`: nonnegative scalar

The function first reads a global integer used as a possible rejection code. This read occurs even when the supplied value is nonnegative.

If the value is zero or positive, the function:

- Writes the value to the 32-bit field at `0x8C0`.
- Returns without explicitly reading or modifying the first-error field.

If the value is negative, the scalar is not updated. The previously read global integer determines the remaining behavior:

- If that integer is zero, the function returns without accessing the first-error field.
- If it is nonzero, the function reads the first-error field.
- If the first-error field is zero, the function writes the previously read global integer there.
- If the first-error field is already nonzero, it is preserved.

The global integer is therefore a candidate error value, not an unconditional error assignment. Its sign is not restricted by this body.

### Selectors `0xB63` and `0xB64`: endpoint updates

For selector `0xB63`, the supplied value is first written to the first endpoint at `0x8BC`.

For selector `0xB64`, the supplied value is first written to the second endpoint at `0x8C4`.

There is no visible range check on either endpoint value. After that initial write, both selectors use the same derivation path.

#### Equal endpoints

The function compares the endpoint fields. If they are equal, it returns immediately.

The endpoint write has already happened. The derived value and scale marker are left unchanged by this path. In particular, endpoint equality does **not** reset the derived value, clear the scale marker, or produce an error.

Consequently, an equal-endpoint state can retain derived data from an earlier unequal-endpoint state.

#### Difference calculation

If the comparison reports unequal endpoints, the function reads the endpoint values for a subtraction:

**difference = second endpoint − first endpoint**

The comparison and subtraction are separate evaluations. The supplied representation does not establish the relative order of the two operand reads within either evaluation.

Using the 32-bit signed interpretation, the function examines the signed result shifted right by 24 bits. It chooses the scaled path when that top signed portion is neither zero nor negative one.

Equivalently, when subtraction produces a representable signed difference:

| Signed difference | Path |
|---|---|
| `−16,777,216` through `16,777,215`, inclusive | Unscaled path |
| Less than `−16,777,216` | Scaled path |
| Greater than `16,777,215` | Scaled path |

This classification is asymmetric around zero at the boundary: `−16,777,216` is unscaled, while `16,777,216` is scaled.

Endpoint pairs whose mathematical difference exceeds the signed 32-bit range require the overflow qualification in **Uncertainties**.

#### Scaled path

The function invokes F-dc13818ac82a with one explicitly shown argument: the signed difference shifted right by eight bits.

For negative differences, the signed arithmetic-shift interpretation rounds toward negative infinity rather than toward zero. For example, `−16,777,217` shifted right by eight yields `−65,537`.

After the dependency returns, the function:

1. Writes its returned integer to the derived-value field at `0x8C8`.
2. Writes the byte value `8` to the scale marker at `0x8CC`.
3. Returns.

The returned integer is stored without a visible validity check.

#### Unscaled path

If the computed difference is zero, the function selects `0x7FFFFFFF` as the derived value and does not invoke either dependency.

Otherwise, it invokes F-829ee7540d02. No explicit arguments are shown for that call. The dependency’s returned integer becomes the derived value.

The function then:

1. Writes the selected or returned integer to `0x8C8`.
2. Writes the byte value `0` to `0x8CC`.
3. Returns.

With stable endpoint storage, the zero-difference case is unreachable because endpoint equality already returned. It remains a visible branch in the supplied representation. The body does not establish what circumstances, if any, make it reachable in the real execution environment.

### Selector `0xB65`: mode selection

This selector accepts exactly three values:

| Supplied value | Byte written at `0x8B8` |
|---|---:|
| `0x0800` | `1` |
| `0x0801` | `2` |
| `0x2601` | `0` |

For an accepted value, the function writes the mapped byte and returns. It does not explicitly consult or clear the first-error field, modify either endpoint, or refresh derived data.

Every other value follows the rejection behavior below.

### Rejected requests

Every selector other than `0xB62`, `0xB63`, `0xB64`, and `0xB65` is rejected. Selector `0xB65` with an unsupported value is also rejected.

For these rejections, the candidate error value is the fixed integer `0x0500`:

- The function reads the first-error field.
- If it is zero, the function writes `0x0500`.
- If it is nonzero, the function leaves it unchanged.

No mode, scalar, endpoint, derived-value, or scale-marker update is explicitly made by this rejection path.

The negative-value handling for selector `0xB62` is different: it uses the global candidate error value, which may be zero, rather than the fixed value `0x0500`.

## Side effects

The explicit write ordering is:

- **Accepted scalar:** global candidate read, then scalar write.
- **Endpoint operation:** selected endpoint write, endpoint comparison, subtraction reads if needed, optional dependency call, derived-value write, scale-marker write.
- **Accepted mode:** one byte write.
- **Rejection:** candidate selection, conditional first-error read, and conditional first-error write.

The first-error field is never explicitly cleared. Successful operations do not erase an earlier error, and an existing error does not prevent a supported operation from updating its selected state fields.

The derived-value write occurs before the scale-marker write. They are not presented as an atomic pair. No synchronization or guarantee for concurrent observation is established.

The function explicitly reads a global integer only on selector `0xB62`. It contains no explicit global write. That distinction does not constrain dependency effects: either dependency may access globals or aliased state through mechanisms absent from the supplied body.

If a dependency does not return normally, the selected endpoint has already been written, but the subsequent derived-value and scale-marker writes have not occurred through this function’s normal continuation.

## Dependencies

| Opaque function ID | Invocation condition | Explicit inputs | Result use |
|---|---|---|---|
| F-dc13818ac82a | Endpoint difference selects the scaled path | Difference shifted right by eight bits | Returned integer is written to the derived-value field |
| F-829ee7540d02 | Endpoint difference selects the unscaled path and is nonzero | No explicit arguments shown | Returned integer is written to the derived-value field |

Neither dependency’s numerical operation is established. In particular, neither may be specified as a reciprocal, division, normalization, or conversion routine solely from this caller.

The absence of explicit arguments for F-829ee7540d02 does not prove independence from the endpoint difference or other machine state. Its calling convention, hidden inputs, global accesses, memory effects, ownership effects, and failure behavior are documented_only unknowns.

## Acceptance examples

These are conditional contract examples derived from static evidence, not executed tests. Unless stated otherwise, state storage is valid and stable during execution.

1. **Accepted zero scalar.**  
   With selector `0xB62` and value `0`, the global candidate is read and the scalar becomes zero. An existing first-error value remains unchanged.

2. **Negative scalar with an enabled candidate.**  
   With selector `0xB62`, value `−1`, global candidate `0x1234`, and a zero first-error field, the scalar is unchanged and the first-error field becomes `0x1234`.

3. **Negative scalar with a disabled candidate.**  
   With selector `0xB62`, a negative value, and global candidate zero, the function returns without reading the first-error field or changing the scalar.

4. **First error survives rejection.**  
   With an unsupported selector and first-error value `7`, that value remains `7`. If the initial first-error value is zero instead, it becomes `0x0500`.

5. **Mode mapping is exact.**  
   Selector `0xB65` with value `0x0801` writes mode byte `2`. Value `0x0802` does not select a nearby mode; it takes the fixed-error rejection path.

6. **Endpoint equality preserves stale derived data.**  
   If the second endpoint is `42`, selector `0xB63` with value `42` writes the first endpoint and returns. Existing derived-value and scale-marker bytes remain as they were.

7. **Small nonzero difference uses the unscaled dependency.**  
   If an endpoint update leaves the first endpoint at `10` and the second at `11`, F-829ee7540d02 is invoked with no explicit arguments shown. If it returns integer `R` normally, the function writes `R` as the derived value, then writes scale marker zero.

8. **Positive boundary selects scaling.**  
   If an endpoint update leaves the endpoints at `0` and `16,777,216`, respectively, F-dc13818ac82a receives `65,536`. Its returned integer is stored, followed by scale marker `8`.

9. **Negative boundary remains unscaled.**  
   Endpoints of `16,777,216` and `0`, respectively, produce difference `−16,777,216` and select F-829ee7540d02.

10. **Negative scaled argument uses signed shifting.**  
    Endpoints of `16,777,217` and `0`, respectively, produce difference `−16,777,217`. Under the signed arithmetic-shift interpretation, F-dc13818ac82a receives `−65,537`.

Dependency examples specify this function’s subsequent writes; they do not assert that the dependency leaves all other fields unchanged.

## Uncertainties

- **Integer representation — documented_only.** The supplied representation strongly supports 32-bit integer words and signed arithmetic, but no independent target type declaration is provided.
- **Subtraction overflow — documented_only.** There is no visible overflow check, widening, or saturation. The representation alone does not conclusively establish wrapping machine semantics for endpoint differences outside signed 32-bit range. No mathematical-range guarantee is assigned to those pairs.
- **Signed shifts — interpretation stated.** The range analysis and negative scaled examples use arithmetic right shifts of signed 32-bit values. Independent instruction-level confirmation is unavailable.
- **Dependency semantics — documented_only.** Numerical meaning, hidden inputs, global preservation, memory effects, and ownership behavior are unknown.
- **Global candidate — documented_only.** Its initialization, allowed values, mutability, and semantic meaning are unknown. The function uses the value read before testing the scalar’s sign.
- **Concurrency and reentrancy — documented_only.** No synchronization is visible. Separate endpoint evaluations do not establish a coherent snapshot under concurrent mutation.
- **Zero-difference derivation branch — documented_only reachability.** Its sentinel assignment is explicit, but stable endpoint storage makes the preceding equality return decisive.
- **Field meanings — inferred intent only.** “Endpoint,” “derived value,” and “scale marker” describe storage relationships, not a proven application-level interpretation.

## Evidence

The established evidence is limited to the supplied static function representation. It shows selector constants, field access positions, byte versus word writes, signed comparisons and shifts, dependency call sites, conditional error latching, and the ordering of explicit state updates.

The numerical scaling interpretation and field labels are explanatory inferences. Dependency implementations, target declarations, and runtime observations were not supplied.

No runtime tests have been performed. The acceptance examples are conditional consequences of the static behavior, not reports of observed execution.
