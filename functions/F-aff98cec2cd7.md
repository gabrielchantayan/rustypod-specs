<!-- Trusted provenance: F-aff98cec2cd7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aff98cec2cd7

## Purpose

F-aff98cec2cd7 computes a scaled difference between two adjacent stored signed integers obtained through F-a516db142ed4. The higher-addressed integer is the minuend; the lower-addressed integer is subtracted from it. A signed right shift by two positions applies to the difference, not separately to the operands.

**Established static evidence:** The function obtains access to the pair through one dependency invocation, observes the two integers, subtracts them in the stated direction, and returns the shifted difference. No local validation or ownership-management operation is explicit.

**Inferred intent:** The calculation may express a numerical difference in groups of four units. Neither the units nor the application meaning is established.

**Unresolved interpretation:** The computed integer’s meaning to callers is **documented_only**. The body alone does not determine whether callers interpret it as a quantity, status, encoded address, resource token, or another application-specific value. Any ownership convention associated with the result is also **documented_only**.

This contract is based on static evidence only. No runtime tests have been performed.

## Interface

### Inputs and output

There are no explicit parameters.

The effective inputs are:

- The result of F-a516db142ed4.
- The lower-addressed signed integer accessible through that result.
- The immediately following signed integer.

External state may influence the dependency, the storage it supplies, and the values observed.

On defined normal completion, the function returns a signed integer of the same declared integer type as the stored operands. Its numerical value is the higher-addressed value minus the lower-addressed value, followed by a signed right shift of two positions.

The function does not return the unshifted difference or separately return the discarded remainder. Whether callers assign status, address, resource, or ownership significance to the computed result remains **documented_only**. In particular, zero and negative results are possible arithmetic outcomes; this body does not establish whether they also have application-specific meanings.

### Storage layout and representation

The dependency result is converted to an address suitable for accessing signed-integer elements.

| Storage position | Numerical role |
|---|---|
| Element at the converted address | Subtrahend |
| Immediately following integer element | Minuend |

The second element is displaced by one object size of the relevant integer type. The shift count does not establish that this displacement is four bytes.

The supplied body does not establish a concrete integer width, signed range, object size, alignment, byte order, padding scheme, address width, or pointer representation. Both operands and the result use the same declared signed-integer type; no explicit widening or unsigned reinterpretation is present.

This is a typed-integer interface, not a serialized-byte interface. A region containing enough bytes is not necessarily suitable: representation, alignment, lifetime, permitted typed access, and the rules governing access to the following element must also be satisfied.

The dependency’s original return representation and the validity guarantees of its conversion to an address remain **documented_only**.

### Preconditions for defined normal completion

All of the following are required:

- F-a516db142ed4 returns normally.
- Its result supports the required address conversion.
- Both integer elements are accessible under the applicable object, lifetime, alignment, and typed-access rules.
- Each element remains valid through its observation.
- Concurrent accesses, if any, comply with the applicable memory model.
- The intermediate subtraction satisfies the applicable signed-arithmetic rules.

Under ordinary signed-overflow rules, the difference itself must be representable. It is insufficient that the mathematical difference, after division by four, would fit.

These conditions are not locally checked. Their violation does not select a specified sentinel or error return.

## Behavior

### Dependency invocation and operand observations

The function directly invokes F-a516db142ed4 once, without explicit arguments. That invocation supplies the address basis for both operand observations.

The dependency completes before either operand is observed. Its invocation is part of the behavior and cannot be omitted merely because a prospective arithmetic result appears predictable. The dependency’s external effects are not known.

Each element contributes one value observation at the expression level. The relative order of the two observations is not established. The higher-addressed element’s role as the minuend does not establish that it is read before or after the lower-addressed element.

No atomic-pair, coherent-snapshot, volatile-access, or synchronization mechanism is explicit. With valid stable storage, the calculation uses the unchanged pair. If the values change, any defined execution must still satisfy the applicable access and synchronization rules. An unsynchronized conflicting access must not be treated as a specified facility for returning arbitrary combinations.

These expression-level observations do not fix the number, size, or order of physical memory transactions performed by an implementation.

### Numerical transformation

Let \(L\) denote the observed lower-addressed integer and \(H\) the observed higher-addressed integer. The intermediate mathematical difference is:

\[
D = H - L
\]

The implemented subtraction must be valid in the relevant signed-integer type. The signed right shift by two positions then applies to that intermediate result.

For a representable nonnegative difference, the result is:

\[
\left\lfloor \frac{D}{4} \right\rfloor
\]

Thus, differences from zero through three produce zero, and differences from four through seven produce one. A nonzero discarded remainder does not cause an explicit error.

For a negative difference, the exact numerical result depends on the applicable negative signed-right-shift rule. That rule remains **documented_only**.

If the target uses arithmetic right shift, the result is division by four rounded toward negative infinity. Under that condition, a difference of negative five produces negative two. Division rounded toward zero would produce negative one and is not an equivalent substitute.

The individual operands need not be nonnegative. Their signs impose no additional local restriction beyond valid representation and arithmetic. Two negative operands can produce a positive difference, and two positive operands can produce a negative difference.

Scaling each operand before subtraction is not equivalent to scaling the difference. Separate scaling can discard information at different points and change the result.

### Arithmetic boundaries

The subtraction is performed using the operands’ signed-integer type. No explicit wider intermediate, saturation, overflow detection, clamping, or modular conversion is present.

Individually representable operands can produce an unrepresentable difference. The later shift does not retroactively make that subtraction valid.

Under ordinary signed-overflow rules, an out-of-range subtraction has no defined numerical result. Wrapping to a representable signed value is not established behavior. Any target-specific alternative remains **documented_only**.

### Failure behavior

There is no explicit local error branch, retry, fallback, or recovery mechanism.

An unusable dependency result, invalid address conversion, inaccessible element, invalid typed access, expired lifetime, or invalid arithmetic operation does not require any particular return value. The body does not establish whether such an execution faults, terminates, or has another outcome.

If F-a516db142ed4 does not return normally, this function has no established independent recovery behavior.

## Side effects

Outside the dependency, the body explicitly reads the two integers and computes the return value. It contains no explicit write to the pair or other external storage, allocation, release, or retention of the obtained address for later use.

No ownership-management operation is explicit in this body. That observation does not determine the ownership of the underlying storage or any ownership convention associated with the computed return value. Both remain **documented_only**.

The absence of explicit writes does not establish purity or global-state preservation. F-a516db142ed4 may modify globals, other memory, resource state, or storage lifetime. It may allocate, replace, release, or transfer resources internally; those possibilities are neither established nor excluded by this body.

Ordinary operand reads do not assign new values to the stored integers. If the address refers to special memory whose reads have observable effects, those effects depend on the environment and remain **documented_only**.

Defined normal completion requires usable storage through both observations. This requirement does not identify who owns that storage, who maintains its lifetime, or who must eventually release it.

## Dependencies

### F-a516db142ed4

**Established interaction:** One direct invocation without explicit arguments. Its result supplies the address basis for both adjacent integer accesses.

**Required for defined normal completion:** It returns normally and supplies a result compatible with the conversion and accesses described in this contract.

| Property | Status |
|---|---|
| Original return representation | **documented_only** |
| Address-conversion guarantees | **documented_only** |
| Calling convention and ABI details | **documented_only**, beyond the absence of explicit arguments |
| Storage extent, alignment, representation, and permitted access | Required for valid access; dependency guarantees are **documented_only** |
| Storage lifetime and stability | **documented_only** |
| Allocation, release, replacement, or ownership transfer | **documented_only** |
| Effects on globals, memory, or resources | **documented_only** |
| Synchronization and concurrent-access guarantees | **documented_only** |
| Failure conditions and expression of failure | **documented_only** |
| Application meaning of supplied values | **documented_only** |

No other direct dependency is established. This does not constrain calls or effects internal to F-a516db142ed4.

## Acceptance examples

These examples are conditional mathematical deductions, not observed executions. Unless otherwise stated, they assume normal dependency completion, valid stable storage, permitted accesses, and representable subtraction.

### Nonnegative differences

| Lower-addressed value | Higher-addressed value | Difference | Required result |
|---:|---:|---:|---:|
| 20 | 20 | 0 | 0 |
| 20 | 23 | 3 | 0 |
| 20 | 24 | 4 | 1 |
| 20 | 31 | 11 | 2 |
| −12 | −4 | 8 | 2 |
| −7 | 4 | 11 | 2 |
| 3 | 4 | 1 | 0 |

These cases distinguish the subtraction direction, establish discarded-remainder behavior for nonnegative differences, and demonstrate that negative individual operands are not excluded.

The final row also distinguishes the required operation from separately shifting the operands: separately scaling 3 and 4 would yield a difference of one rather than the required zero.

### Negative differences

The following exact results additionally require arithmetic right shift:

| Lower-addressed value | Higher-addressed value | Difference | Conditional required result |
|---:|---:|---:|---:|
| 9 | 5 | −4 | −1 |
| 9 | 8 | −1 | −1 |
| 9 | 4 | −5 | −2 |

The last two cases distinguish arithmetic right shift from division rounded toward zero. Without an established negative-shift rule, their exact results remain **documented_only**.

### Conditional 32-bit boundaries

These examples assume a signed range of −2,147,483,648 through 2,147,483,647. They do not establish the actual target range.

| Lower-addressed value | Higher-addressed value | Conditional outcome |
|---:|---:|---|
| 0 | 2,147,483,647 | 536,870,911 |
| 0 | −2,147,483,648 | −536,870,912 if the shift is arithmetic |
| −2,147,483,648 | 2,147,483,647 | Difference exceeds the signed range |
| 2,147,483,647 | −2,147,483,648 | Difference falls below the signed range |

Under ordinary signed-overflow rules, the last two rows have no defined numerical result. A mathematically representable scaled result does not repair the invalid intermediate subtraction.

### Storage, effects, and interpretation

- One accessible integer is insufficient; both elements must be valid for their observations.
- A null or otherwise unusable converted address does not require a zero result.
- A dependency that changes external state and then supplies valid stable storage is not excluded.
- An acceptance check must not require a particular relative order for the operand observations.
- A changing pair does not acquire a coherent-snapshot guarantee from this function.
- Abnormal dependency completion does not trigger an established retry or fallback.
- No explicit ownership-management operation should be attributed to this body. Allocation or release inside the dependency remains unresolved.
- A caller’s interpretation of the computed integer as a status, encoded address, or resource token cannot be accepted or rejected from this body alone. Any associated ownership convention requires an external contract.

## Uncertainties

The following remain **documented_only**:

- Concrete integer width, signed range, object size, alignment, byte order, and padding.
- Dependency return representation, ABI, and address-conversion semantics.
- Exact behavior of negative signed right shift.
- Target-specific departures from ordinary signed-overflow rules.
- Storage ownership, lifetime guarantees, and release responsibility.
- Application interpretation of the returned integer, including any ownership it may convey.
- Dependency effects on globals, memory, resource state, and storage lifetime.
- Synchronization guarantees and admissible concurrent access.
- Physical memory transactions and special-memory read effects.
- Outcomes after failed access or arithmetic preconditions.
- Units, domain restrictions, and application meaning of the stored values.

The inferred four-unit interpretation does not establish byte units, pointer-distance semantics, operand ordering by value, or divisibility by four.

These unknowns do not justify adding validation, clamping, retries, separate operand scaling, ownership operations, or a prescribed error sentinel.

## Evidence

**Established static evidence:** The supplied body has no explicit parameters and returns a signed integer computed from two adjacent signed-integer observations. One invocation of F-a516db142ed4 provides their address basis. The higher-addressed value is reduced by the lower-addressed value, and a signed right shift by two positions applies after subtraction. No local validation, explicit external write, allocation, release, retry, or recovery operation appears.

**Inferred intent:** Expressing a difference in four-unit groups is plausible, but neither units nor application semantics are established.

**Not established:** Concrete representation widths, dependency internals, global preservation, storage ownership, ownership conveyed by the return value, caller interpretation, and the target’s negative-shift rule cannot be resolved from this body.

All acceptance examples are conditional deductions. No runtime tests have been performed.
