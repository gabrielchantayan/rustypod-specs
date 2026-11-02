<!-- Trusted provenance: F-a8474adba0ce; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a8474adba0ce

## Purpose

F-a8474adba0ce combines two two-word records into a caller-supplied destination. It adds the first components and the second components separately. If the resulting signed second-component sum is at least 86,400, it subtracts 86,400 from that component and adds one to the first-component sum.

The adjustment happens **at most once**. The function does not repeatedly reduce a large second component, and it does not borrow from the first component when the second-component sum is negative.

The constant 86,400 suggests a time-related representation, such as a coarse unit paired with seconds. That interpretation is inferred intent, not an established interface requirement. The observable contract concerns two signed integer components; it does not establish units, an epoch, or a calendar interpretation.

This contract distinguishes behavior established by the supplied static evidence from arithmetic and execution details that remain undocumented. No runtime tests have been performed.

## Interface

### Arguments and layout

The function accepts three addresses:

| Argument | Role | Required accessible storage |
|---|---|---|
| First input | Supplies a first component and a second component | Two consecutive readable integer words |
| Second input | Supplies a first component and a second component | Two consecutive readable integer words |
| Destination | Receives the combined components | Two consecutive writable integer words |

The exposed arithmetic model uses signed 32-bit integer words. Under that model, each record occupies eight bytes:

- Offset 0: first component.
- Offset 4: second component.

The signed 32-bit range is −2,147,483,648 through 2,147,483,647. The four-byte signed-subtraction overflow operation in the supplied evidence supports this width interpretation. Byte order is not established by the function itself.

The arguments are addresses to existing storage, not values passed by ownership transfer. There is no returned data value; the result is delivered through destination writes.

### Preconditions

For the defined, non-overflowing behavior described below:

- Each input must permit reading both component words at the time those reads occur.
- The destination must permit writing both component words.
- The addresses must satisfy the execution environment’s alignment and object-access requirements.
- The first-component addition and second-component addition must be representable as signed 32-bit integers.
- If a carry adjustment occurs, the first-component sum plus one must also be representable as a signed 32-bit integer.
- No concurrent or asynchronous mutation may invalidate the read/write sequence being described.

The function performs no visible checks for null addresses, inaccessible storage, arithmetic overflow, alignment, or semantic validity of the components.

Input and destination storage may overlap in ways that are meaningful at the execution-model level, but overlap can change which values are read. An implementation or language binding may impose additional validity constraints on such overlap; those constraints are not established here.

## Behavior

### Ordinary, non-overlapping result

When the destination does not alter either input before its components are read, define:

- **A** as the mathematical sum of the two first components.
- **B** as the mathematical sum of the two second components.

Provided the applicable additions are representable:

| Condition | Destination first component | Destination second component |
|---|---:|---:|
| B < 86,400 | A | B |
| B ≥ 86,400 | A + 1 | B − 86,400 |

The comparison is signed. Negative values do not satisfy the carry condition.

This rule does not require either input second component to be individually normalized. It depends on their sum. For example, a negative second component can cancel part of a positive second component.

The result is not necessarily normalized:

- A second-component sum of 172,800 produces a second component of 86,400, with only one increment to the first component.
- A negative second-component sum remains negative, without decrementing the first component.

### Observable access and write order

The sequence of accesses matters independently of the final arithmetic result.

1. The first input’s first component is read.
2. The second input’s first component is read.
3. Their sum is written to the destination’s first component.
4. Both inputs’ second components are read and added.
5. The unadjusted second-component sum is written to the destination’s second component.
6. If the carry condition is satisfied, the destination’s second component is overwritten with that sum minus 86,400.
7. On that same carry path, the destination’s first component is overwritten with the sum of the previously read first components plus one.

The supplied expression does not establish an ordering between the two second-component reads. Both occur after the initial destination first-component write.

The initial first-component values are retained for the final carry calculation. The carry path does not reread the destination’s first component to increment whatever happens to be there.

Likewise, the carry decision and adjusted second-component value use the computed second-component sum. They do not require rereading the destination’s second component after its initial write.

These are the accesses and writes expressed by the supplied static representation. Whether a compiled execution exposes each intermediate write to an external observer depends on memory properties, compilation, and the execution environment, none of which are supplied.

### Threshold calculation

The static evidence uses intermediate constants 81,920 and 4,480, whose sum is 86,400.

For a representable signed second-component sum:

- Sums below 81,920 do not carry.
- Sums from 81,920 through 86,399 also do not carry.
- Sums of 86,400 or greater carry once.

The intermediate split at 81,920 does not introduce an additional adjustment or a different output format. Its observable carry boundary is 86,400.

The signed-subtraction overflow indication is part of evaluating the threshold condition. It is not an error report and does not constitute a general check for overflow in the preceding additions.

### Overlapping storage

The function does not snapshot both complete inputs before writing the destination.

If the destination is exactly the first input record, the initial destination write replaces that record’s first component only after it has been read. Under the ordinary two-word layout, it does not replace that record’s second component before the second-component read. The same reasoning applies when the destination is exactly the second input record.

Partial overlap is different. If the destination’s first word occupies an input’s second-word location, the initial first-component write changes the value later read as that input’s second component. In that case:

- The first-component sum still uses the original first-component reads.
- The second-component sum uses the values present after the initial destination write.
- A formula based on a simultaneous snapshot of both original input records is incorrect.

The later second-component writes occur after both second-component reads. Their overlap with input locations therefore does not change the already-computed sum.

## Side effects

### Destination mutation

On the non-carry path, the function expresses two destination writes:

1. First-component sum.
2. Second-component sum.

On the carry path, it expresses four destination writes:

1. Unadjusted first-component sum.
2. Unadjusted second-component sum.
3. Adjusted second component.
4. Adjusted first component.

The two-word result is not published as one atomic update. The supplied evidence provides no synchronization or transaction mechanism.

### Input mutation through aliases

The function contains no direct writes through the input arguments. Nevertheless, an input’s backing storage is modified if it overlaps the destination.

Consequently, “input” describes how an argument is accessed, not a guarantee that its underlying storage remains unchanged.

### Ownership and resources

No allocation, release, ownership transfer, retained address, or resource-management operation is visible. The caller must keep the referenced storage valid throughout the accesses.

No direct global-state access, external I/O, or dependency call is visible.

### Errors and exceptional execution

There is no status result and no explicit error path.

Invalid memory access, arithmetic overflow, or an execution-environment fault has no specified recovery behavior in the supplied evidence. In particular:

- No rollback is visible.
- An invalid later access may be reached after the initial destination write.
- No guarantee of an unchanged destination on failure is established.
- Signed-addition overflow must not be silently assigned wrapping behavior by this contract.

## Dependencies

No calls to other functions are visible in F-a8474adba0ce. There are therefore no opaque callee IDs to enumerate.

The operation depends on the execution environment for:

- Signed 32-bit word interpretation and storage access.
- The semantics of arithmetic outside the representable signed range.
- Address validity, alignment, and permitted overlap.
- The visibility of intermediate stores.
- Fault and concurrency behavior.

The signed-subtraction overflow operation appearing in the static evidence is an arithmetic primitive, not an identified external callee. It does not imply an allocator, callback, or other hidden dependency.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Except where overlap is explicitly described, assume disjoint valid storage, signed 32-bit values, no concurrent mutation, and representable arithmetic.

### Below the carry threshold

First input: (7, 40,000).  
Second input: (11, 46,399).

Expected destination: **(18, 86,399)**.

The second-component sum is one below the carry threshold. Neither component receives an adjustment.

### Exactly at the carry threshold

First input: (7, 40,000).  
Second input: (11, 46,400).

Expected destination: **(19, 0)**.

The expressed destination writes are first component 18, second component 86,400, second component 0, then first component 19.

### Intermediate split is not a carry boundary

First input: (2, 81,920).  
Second input: (3, 0).

Expected destination: **(5, 81,920)**.

Reaching 81,920 alone does not cause the 86,400 adjustment.

### Adjustment occurs only once

First input: (2, 172,800).  
Second input: (3, 0).

Expected destination: **(6, 86,400)**.

A result of (7, 0) would incorrectly impose repeated normalization.

### Negative sum does not borrow

First input: (10, −1).  
Second input: (20, 0).

Expected destination: **(30, −1)**.

A result of (29, 86,399) would add a borrow behavior absent from the evidence.

### Individually out-of-range components can cancel

First input: (1, 90,000).  
Second input: (2, −4,000).

Expected destination: **(3, 86,000)**.

The function does not first normalize either input record separately.

### Exact in-place destination

First input initially: (7, 40,000).  
Second input: (11, 46,400).  
Destination: exactly the first input record.

Expected final first input and destination: **(19, 0)**.

This is conditional on valid overlapping access. The initial first-component store does not overwrite the second component that is still needed.

### Partial overlap changes a later read

Three consecutive words initially contain 1, 10, and an arbitrary writable value. The first input starts at the first word; the destination starts at the second word. A separate second input contains (2, 0).

The initial destination first-component write stores 3 into the location that is also the first input’s second component. The subsequent second-component sum is therefore 3 + 0.

Expected destination: **(3, 3)**, not (3, 10).

This example requires an environment in which the described overlapping accesses are valid.

### Overflow boundary is not assigned a wrapped answer

First input: (2,147,483,647, 86,400).  
Second input: (0, 0).

The initial component sums are representable, but the carry would require a first component of 2,147,483,648.

The exact resulting behavior is **documented_only**. Acceptance must not assume wraparound, saturation, an exception, or an unchanged destination without additional execution-model evidence.

## Uncertainties

- **Arithmetic overflow — documented_only.** The supplied representation does not establish a complete executable semantics for signed-addition overflow. The mathematical result contract applies when the relevant additions are representable.
- **Intermediate write visibility — documented_only.** Static write order is established at the supplied representation level. Compiler transformations and externally observable memory behavior are not established.
- **Invalid-address behavior — documented_only.** No validation is visible, but the exact fault mechanism and resulting memory state are unknown.
- **ABI details — documented_only.** Calling convention, pointer width, byte order, and a usable return-register value are not established.
- **Second-component read ordering — documented_only.** Both follow the initial destination first-component write; their relative order is not specified by the supplied expression.
- **Concurrency — documented_only.** No coherent snapshot, atomicity, race handling, or synchronization guarantee is established.
- **Domain meaning — inferred intent only.** A time-like interpretation is plausible because of 86,400, but it is not required to explain the behavior.
- **Caller invariants — unknown.** The evidence does not establish that callers restrict second components to a particular range or prohibit overlap.

## Evidence

**Established static evidence:** two first-component reads precede the initial destination write; second-component reads follow that write; the unadjusted second-component sum is stored before any carry correction; the correction subtracts 86,400 and increments the retained first-component sum once. No dependency call, loop, explicit validation, or error return is visible.

**Derived behavior:** within representable signed arithmetic, the threshold logic is equivalent to carrying exactly when the second-component sum is at least 86,400. The access sequence also establishes that partial overlap can change a later input read.

**Inferred intent:** the two components may encode a coarse time unit and seconds. No domain-specific interpretation is necessary for acceptance.

**Verification status:** static analysis only. The acceptance examples are derived expectations; no runtime tests or execution observations have been performed.
