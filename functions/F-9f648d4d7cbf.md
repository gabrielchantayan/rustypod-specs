<!-- Trusted provenance: F-9f648d4d7cbf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9f648d4d7cbf

## Purpose

F-9f648d4d7cbf copies two 32-bit fields within a caller-supplied object, combines the copied second field with an indirectly supplied adjustment, obtains a correction from F-870c4fcf7295, and updates the two destination fields.

The established behavior is a sequence of memory accesses and arithmetic operations. A coordinate, fixed-point, or normalization interpretation is plausible but not established. In particular, the available evidence does not establish that the resulting second field lies within any particular range.

This contract distinguishes:

- **Established static evidence:** operations and ordering visible in the supplied representation.
- **Inferred intent:** possible explanations that are not acceptance requirements.
- **documented_only unknowns:** behavior that cannot be determined without dependency, global-data, or execution evidence.

No runtime tests have been performed.

## Interface

### Inputs

The function receives three arguments:

| Argument | Represented width and role |
|---|---|
| Object address | A 32-bit address-bearing value identifying storage containing fields through byte offset `0x2B`. |
| Direct adjustment | A 32-bit integer added to the first destination field after the dependency returns. |
| Indirect adjustment address | An address from which one 32-bit integer is read after both initial field copies. |

The object layout relevant to this function is:

| Byte offset | Width | Role |
|---|---:|---|
| `0x1C` | 4 bytes | First source field |
| `0x20` | 4 bytes | Second source field |
| `0x24` | 4 bytes | First destination field |
| `0x28` | 4 bytes | Second destination field |

The initial copies transfer 32-bit field contents. Subsequent arithmetic interprets destination values and adjustments as integers.

The function also accesses three global values:

| Contract label | Use |
|---|---|
| Negative-branch global | Second argument to the dependency when the intermediate sum is negative |
| Nonnegative-branch global | Second argument to the dependency otherwise |
| Scale global | Factor in the final second-field calculation |

These labels identify three distinct global locations, not known numeric constants. Their contents, signedness declarations, volatility, and broader meaning are not supplied.

### Output

There is no explicit return value. Results are observable through:

- Writes to the object’s destination fields.
- Any additional effects of F-870c4fcf7295.

### Preconditions

For ordinary completion, the supplied addresses must permit the reads and writes described below. The representation includes no null checks, bounds checks, alignment checks, or recovery path.

The object must provide accessible storage for all four listed fields. The indirect adjustment address must permit a 32-bit read when that read occurs. No non-aliasing requirement is visible: the indirect adjustment may refer to an object field, with consequences determined by access order.

The behavior of invalid addresses, unsupported alignment, inaccessible global storage, or arithmetic outside the representable signed range is not established.

## Behavior

### 1. Copy the source fields into the destinations

The first source field at `0x1C` is read and its 32-bit contents are written to `0x24`.

Next, the second source field at `0x20` is read and its 32-bit contents are written to `0x28`.

These are ordered copies, not an established atomic snapshot of both source fields. The function does not first collect both fields and then publish both destinations.

Consequently, the old destination values are overwritten before the indirect adjustment is read or the dependency is called.

### 2. Form and classify an intermediate sum

After both copies, the function reads the second destination field at `0x28` and the 32-bit value at the indirect adjustment address. It adds these values to form an intermediate signed integer, called the **sum** in this contract.

The available representation does not establish the relative evaluation order of those two reads within the addition. Both occur after the initial copies and before the dependency call.

The function selects a branch using a strict signed comparison:

- If the sum is less than zero, it takes the negative branch.
- If the sum is zero or positive, it takes the nonnegative branch.

A negative source field alone does not select the negative branch. The decision depends on the sum after incorporating the indirect adjustment.

### 3. Obtain and interpret the dependency result

Exactly one call to F-870c4fcf7295 is directly present on each ordinary execution path.

On the negative branch:

- The first argument is the negative sum itself.
- The second argument is the value obtained from the negative-branch global.
- If the dependency returns integer \(r\), the correction is \(c = -(r + 1)\).

On the nonnegative branch:

- The first argument is the sum.
- The second argument is the value obtained from the nonnegative-branch global.
- The correction is the dependency’s returned integer: \(c = r\).

The negative branch does not pass an absolute value. Its correction is not merely the negation of the result; the increment occurs before negation.

No established evidence identifies the dependency as division, remainder, rounding, or normalization. Its behavior must remain opaque.

### 4. Update the first destination field

After the dependency returns, the function reads the first destination field at `0x24`. It adds the correction and the direct adjustment, then writes the result back to `0x24`.

For representable arithmetic:

\[
\text{first destination after write}
=
\text{first destination read after the call}
+
c
+
\text{direct adjustment}
\]

The read is from the destination after the call, not necessarily from an unchanged copy of the first source field. If the dependency alters that destination, its alteration participates in this calculation.

The direct adjustment does not affect the branch decision or the dependency’s explicit arguments.

### 5. Update the second destination field

After the first destination update, the function reads the scale global and computes the final second destination value:

\[
\text{second destination after write}
=
s + ((g \times c) \times 128)
\]

Here:

- \(s\) is the previously computed sum.
- \(g\) is the scale global as read for this final calculation.
- \(c\) is the correction derived from the dependency result.

The multiplication grouping visible in the representation is scale times correction, followed by multiplication by 128. The final operation adds the saved sum.

This calculation does not reread the second destination field. If the dependency changed that field, the final write replaces its value using the previously computed sum.

The function then returns without an explicit result.

### Arithmetic width and exceptional cases

The supplied representation uses 32-bit integer quantities for the relevant local arithmetic and destination storage. The factor `0x80` is decimal 128.

Potential signed-range failures exist in:

- Forming the sum.
- Incrementing and negating the negative-branch dependency result.
- Adding the correction and direct adjustment to the first destination.
- Each multiplication in the second destination calculation.
- Adding the saved sum to the scaled correction.

The evidence does not establish overflow checking, saturation, a wider arithmetic path, or guaranteed modulo-\(2^{32}\) behavior. It also does not establish a language-level overflow contract.

Accordingly, the formulas above are exact requirements when the relevant intermediate results are representable. Overflowing cases remain **documented_only** until execution-level arithmetic semantics are established. In particular, neither signed wraparound nor a particular branch outcome following an overflowing sum should be assumed.

## Side effects

### Direct memory effects

On an ordinary completed path, the function directly performs four 32-bit writes in this order:

1. First source contents to the first destination.
2. Second source contents to the second destination.
3. Adjusted value to the first destination.
4. Final computed value to the second destination.

The source fields are read but not directly written by this function. The indirect adjustment storage is read but not directly written through that argument. The three globals are read, not directly assigned.

These statements describe direct effects only. They do not constrain dependency effects or aliasing with other storage.

### Aliasing

The indirect adjustment is read after the initial destination writes. Therefore:

- If it points exactly to `0x24`, it supplies the newly copied first source value, absent intervening external mutation.
- If it points exactly to `0x28`, it supplies the newly copied second source value, absent intervening external mutation.
- If it points to separate storage, that storage supplies the adjustment at the time of the read.

No initial snapshot of the indirect adjustment is preserved before the copies.

Partial-overlap behavior depends on address validity, alignment, and memory representation; these are not established.

### Dependency visibility and mutation

The dependency is called after both initial copies and before either final destination update. It may therefore observe the copied destinations if it has an independent means of accessing the object.

It receives neither the object address nor the indirect adjustment address as an explicit argument. That does not prove it cannot access them through global or shared state.

No preservation guarantee is available for the object or globals across the call:

- A change to the first destination may affect its subsequent update.
- A change to the second destination is not used to reconstruct the saved sum.
- A change to the scale global may affect the final calculation because that global is read after the call.
- Changes to either branch-selection global after its argument value has been obtained do not retroactively change the argument already passed.

### Ownership and failure

No direct allocation, deallocation, ownership transfer, or resource-release operation is visible. This is not a guarantee that the dependency performs none.

There is no explicit status return, rollback, or error-handling branch. If the dependency does not return normally, the initial copies have already occurred, while the two final updates have not yet occurred. Any additional effects of such a failure are unknown.

The operation is not established as atomic, synchronized, or safe against concurrent mutation.

## Dependencies

### F-870c4fcf7295

This is the only directly visible function dependency.

Its observed interface in this context is:

- First argument: the 32-bit signed sum.
- Second argument: a value read from one of two globals, selected by the sum’s sign.
- Return: a value consumed as a 32-bit integer.

Its algorithm, accepted argument domain, failure modes, global effects, and ownership behavior are **documented_only** unknowns.

The caller’s required treatment of the result is established independently of those unknowns: use it directly on the nonnegative branch; increment and negate it on the negative branch.

## Acceptance examples

These are conditional static acceptance examples, not reports of execution. Unless stated otherwise, assume valid storage, representable arithmetic, a normally returning dependency, and no dependency or concurrent mutation of the object or globals.

### Nonnegative sum

Given:

- First source: 100.
- Second source: 5.
- Indirect adjustment: 3.
- Direct adjustment: 7.
- Scale global: 2.
- Dependency result for this call: 4.

Required observations:

- Initial destination copies produce 100 and 5.
- The sum is 8.
- The dependency receives 8 and the nonnegative-branch global’s value.
- The correction is 4.
- The final first destination is \(100 + 4 + 7 = 111\).
- The final second destination is \(8 + 2 \times 4 \times 128 = 1032\).

### Negative sum

Given:

- First source: 10.
- Second source: −6.
- Indirect adjustment: 1.
- Direct adjustment: 2.
- Scale global: 1.
- Dependency result: 2.

The sum is −5, so the dependency receives −5 and the negative-branch global’s value. The correction is −3. Final destinations are 9 and −389.

Passing positive 5 to the dependency, or using correction −2, would not satisfy the observed behavior.

### Zero boundary

If the second source is −3 and the indirect adjustment is 3, the sum is zero. The nonnegative branch is required, regardless of the source field’s negative sign.

### Adjustment aliases the second destination

Given second source 6 and an indirect adjustment address equal to the second destination’s address, the initial copy makes that destination 6 before the adjustment read. The sum is therefore 12, not 6 plus the destination’s old value.

### Dependency changes post-call inputs

Suppose the dependency returns 2, changes the first destination to 50, and changes the scale global to 3. With nonnegative saved sum 8 and direct adjustment 1:

- The final first destination is 53.
- The final second destination is 776.

These results require the stated mutations and no subsequent interference. They illustrate why preservation of object fields and globals cannot be assumed.

## Uncertainties

The following remain **documented_only**:

- The semantic domain of the object fields and adjustments.
- The numeric contents and declared types of the three globals.
- The dependency’s actual computation and side effects.
- Overflow behavior and exact target arithmetic semantics.
- Endianness, alignment rules, and validity of unusual aliasing arrangements.
- Volatile, atomic, or memory-mapped properties of the accessed storage.
- Behavior on invalid memory access or abnormal dependency termination.
- Any caller-enforced range, lifetime, synchronization, or ownership requirements.

An inferred normalization interpretation is consistent with the correction and scale factor, but no quotient, remainder, carry, or bounded-output property is established.

## Evidence

The supplied static representation establishes the four field offsets, 32-bit transfers, ordered initial copies, post-copy indirect read, signed negative test, branch-dependent dependency arguments, negative-result transformation, post-call first-destination read, and final scale-based write.

It provides no dependency body, global initializers, caller constraints, runtime observations, or tests. The acceptance examples derive only from the visible operations under their stated conditions; they do not claim observed execution.
