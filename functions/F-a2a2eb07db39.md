<!-- Trusted provenance: F-a2a2eb07db39; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a2a2eb07db39

## Purpose

F-a2a2eb07db39 samples one byte of caller-provided context and writes three outputs:

- A 32-bit value supplied by a resource-associated expression.
- A 16-bit value: 60 when the sampled byte is zero, or 80 otherwise.
- An 8-bit zero.

The resource-associated expression is evaluated after the selector is sampled and before the first output is written. Its declaration and definition are absent. It may supply an address-like constant, or it may read a stored value. The evidence therefore does not establish that the first output is constant across invocations.

The resource’s naming suggests a Helvetica-related purpose. Interpreting the outputs as a font reference, size, and attribute is inferred intent, not an established contract. The resource contents and the numeric outputs’ semantic units are unknown.

The function has no explicit return value and contains no visible function calls, allocation, or release.

## Interface

### Arguments and storage

The interface has four arguments in this order:

| Argument | Role | Established access |
|---|---|---|
| Context address representation | Locates the selector | One 8-bit read at byte offset 168, hexadecimal 0xA8 |
| First output destination | Receives the resource-expression value | One 32-bit write |
| Second output destination | Receives the selected magnitude | One 16-bit write |
| Third output destination | Receives zero | One 8-bit write |

The context argument is represented as an integer used in address formation. The excerpt does not independently establish its ABI width, the original pointer type, or the arithmetic rules of the execution environment. The intended accessed location is 168 bytes beyond the context address; this is not an index into an array of wider elements.

The output arguments designate storage. Their existing contents are not explicitly read through those arguments. However, an output region could also be storage read by the unresolved resource expression. The evidence does not justify excluding that possibility.

### Output values

With valid, disjoint output storage and ordinary completion:

- The first destination contains the 32-bit value supplied by the resource expression during this invocation.
- The second destination contains 60, hexadecimal 0x003C, if the sampled selector is zero; otherwise it contains 80, hexadecimal 0x0050.
- The third destination contains an 8-bit zero.

The first output is a value assignment, not an observed copy of the resource’s contents. Whether the assigned value is an address, handle, stored scalar, or another representation is unresolved. Its exact numeric value and any conversion into the 32-bit destination are also not established by the excerpt.

Overlapping output regions do not necessarily retain all three assigned values simultaneously. Later writes can replace bytes written earlier.

There is no explicit return value, status code, or error indicator.

### Preconditions

Ordinary execution requires:

- A readable selector byte at the effective context address plus 168.
- Four writable bytes at the first destination.
- Two writable bytes at the second destination.
- One writable byte at the third destination.
- Address formation, alignment, and access permissions suitable for the execution environment.
- Any storage read while evaluating the resource expression to be accessible.

No null checks, bounds checks, destination-size checks, or alignment checks are visible. A null output destination does not request omission of that output.

The excerpt does not establish that the context, output regions, and any resource-expression storage must be disjoint. It also does not establish that every such overlap is permitted by the original language or execution environment. Overlap examples below are conditional on those accesses being valid.

## Behavior

### Established ordering

The visible evaluation and assignment sequence is:

1. Sample the selector byte at context offset 168.
2. Evaluate the resource-associated expression to obtain the value for the first output.
3. Write that value to the first destination using a 32-bit assignment.
4. Select 60 or 80 according to whether the previously sampled byte was zero.
5. Write the selection to the second destination using a 16-bit assignment.
6. Write an 8-bit zero to the third destination.

The selector is not explicitly sampled again. All output writes follow its sample.

This sequence identifies the explicit selector read and output writes, together with the position of resource-expression evaluation. It is not an exhaustive inventory of memory reads: evaluating that expression may require a read from storage whose declaration is unavailable.

### Resource-expression evaluation

Two materially different interpretations remain possible:

- **Address-like expression:** Evaluation supplies an address-associated value without reading the referenced resource’s contents.
- **Stored-value expression:** Evaluation reads storage and supplies a value derived from that read.

The expression’s name alone does not distinguish these interpretations. In particular, it does not prove that the first output is immutable, that it has the same value on every invocation, or that no global read occurs.

If evaluation reads storage, that read belongs after the selector sample and before the first output write in the visible sequence. No output write in this invocation precedes that evaluation.

The excerpt does not establish the read’s width, the storage layout, or the relationship between the expression’s own type and the 32-bit destination. Only the destination assignment width is established.

### Selector interpretation and arithmetic

The selector is tested solely for equality with zero.

Every nonzero byte selects 80, including values with the high bit set. Whether the sampled character representation is signed does not affect this zero-versus-nonzero distinction.

Both selected constants fit within 16 bits. There is no visible scaling, multiplication, saturation, or arithmetic adjustment of either value.

The context displacement is 168 bytes. The excerpt does not specify what happens if computing or using the resulting address exceeds the valid range of the underlying address or integer representation. No wrapping or checked-overflow behavior should be promised.

### Overlap and retained values

If an output write overlaps the selector location, it can change the context byte after sampling. That change does not alter the current invocation’s selected magnitude.

If output regions overlap each other:

- The second output can replace bytes from the first output.
- The final zero byte can replace a byte from either preceding output.
- The final contents must be interpreted according to the target’s byte order and valid-access rules.

If the resource expression reads storage that overlaps an output region, its current value is obtained before any output write. A subsequent write can nevertheless change the underlying resource storage. That change may affect later invocations.

The excerpt does not establish whether such storage is writable or whether a particular overlap is valid. These consequences describe permitted overlap, not permission to modify an otherwise protected resource.

### Completion and invalid accesses

After the final byte assignment, the function returns without an explicit value.

There is no visible error branch, validation failure result, retry, logging, fallback, or rollback.

Invalid memory or invalid address arithmetic has no specified application-level outcome here. The excerpt does not justify promising a particular exception, termination behavior, recovery mechanism, or partial-output state.

The visible ordering does establish that the function contains no restoration step. If the execution environment exposes effects from operations completed before a later fault, no explicit code in this function undoes those effects. This is not a guarantee about executions outside valid memory-access conditions.

## Side effects

### Explicit writes

The explicit write footprint comprises:

- Four bytes at the first output destination.
- Two bytes at the second output destination.
- One byte at the third output destination.

For disjoint regions, this is seven distinct byte locations. Overlap can reduce that count. No additional buffer clearing or write beyond those widths is visible.

There is no explicit write through the context argument itself. Context storage can still change if an output points into it.

Likewise, there is no explicit assignment to named resource storage. Such storage can still change if an output aliases it and the write is valid.

### Reads and external state

The selector is explicitly read once. The resource expression may introduce a further read, potentially from global storage. Its declaration is needed to resolve that possibility.

Consequently, the function cannot be described unconditionally as reading only caller context, depending on no global state, or producing an invariant first output.

No function calls, synchronization operations, allocation operations, or release operations are visible. Access atomicity and concurrency guarantees are not established.

### Ownership and lifetime

The function visibly neither allocates nor releases storage. It does not explicitly retain the destination addresses.

Writing the first output does not establish an ownership transfer. The evidence does not determine:

- Whether the value refers to another object.
- Whether such an object may be dereferenced or modified.
- Its lifetime.
- Whether a recipient must release it.
- Whether it is shared with other callers.

Those obligations cannot be inferred from an assignment or a resource-related name.

## Dependencies

No called-function dependencies are visible, so there are no additional opaque function IDs to list.

The resource-associated expression is a data dependency, not an established function dependency.

**Established static evidence:**

- Its value is used for the first 32-bit output assignment.
- It is evaluated after the selector sample and before that assignment.
- It is not visibly chosen by the zero-versus-nonzero branch.

**Inferred intent:**

- Its name suggests a Helvetica-related resource.
- The values 60 and 80 may represent a size or configuration magnitude.

**Unresolved dependency properties:**

- Address expression versus stored-value read.
- Exact value, type, representation, and any relocation behavior.
- Any underlying read width or global-state dependency.
- Resource contents, lifetime, mutability, and ownership.
- Any relationship between its storage and caller-provided regions.

No assumptions about unknown callees preserving globals or managing memory are necessary: no callee is visible.

## Acceptance examples

These examples are conditional consequences of static evidence, not runtime test results.

### Zero selector and separate outputs

Given valid, disjoint output regions and a selector initially equal to zero:

- The resource expression is evaluated after the selector sample.
- Its supplied 32-bit output value is written to the first region.
- The second region receives 60.
- The third region receives zero.

The prior output contents do not affect the selector branch. If the resource expression reads storage, that storage can still affect the first output.

### Nonzero selector

Given a selector byte of 1, 2, 128, or 255 and otherwise valid accesses:

- The second output receives 80.
- The third output receives zero.
- The first output receives the resource expression’s value for that invocation.

Equality of first outputs across these cases is required only if the resource expression supplies the same value in each case.

### Selector overwritten after sampling

Given an initially nonzero selector, with the third destination exactly at the selector’s address and the other accesses valid:

- The selected magnitude is 80.
- The final byte write changes the selector to zero.

A later invocation samples zero if that byte remains unchanged. The current invocation does not revise its selection.

### Second and third outputs share an address

Given valid overlap in which the second and third destinations have the same address:

- The selected 16-bit value is written first.
- The byte at that address is then replaced with zero.

Conditioned on little-endian representation, the resulting two-byte value is zero for either selection. Conditioned on big-endian representation, the selected value remains 60 or 80 because its first byte was already zero. The excerpt does not select either byte order.

### Resource expression reads mutable storage

Suppose additional declaration evidence establishes that the resource expression reads writable storage, and an output legally overlaps that storage:

- The current invocation obtains the expression’s value before any output write.
- A later output write can alter that storage.
- A subsequent invocation may therefore produce a different first output.

The present excerpt neither establishes nor excludes this scenario.

### Unchanged resource expression across invocations

Given two valid invocations for which the resource expression supplies the same 32-bit value, and disjoint output destinations, the first outputs are equal regardless of the selector values.

This is a conditional equality, not an unconditional invariant of the function.

### Inaccessible storage

If the selector, a destination, or storage required by resource-expression evaluation is inaccessible, acceptance must not require graceful rejection, rollback, a status result, or completion of later operations.

## Uncertainties

The following remain **documented_only**:

- Calling convention, argument ABI widths, and original declared types.
- Address-arithmetic width and invalid-arithmetic behavior.
- Whether the resource expression supplies an address or reads stored data.
- Any associated global read, its width, and its storage layout.
- Exact first-output value and conversion into its 32-bit destination.
- Resource contents, semantic type, mutability, lifetime, and ownership.
- Semantic names and units of the numeric outputs.
- Target byte order, alignment requirements, and access atomicity.
- Applicable aliasing restrictions.
- Fault handling and effects of invalid accesses.
- Concurrency behavior and externally imposed synchronization requirements.

These uncertainties do not erase the established selector rule, immediate numeric values, output widths, or visible ordering under valid ordinary execution.

## Evidence

The supplied static body establishes:

- One explicit selector-byte read at context offset 168.
- Resource-expression evaluation after that sample and before the first output write.
- A 32-bit assignment of the supplied resource-expression value.
- A zero-versus-nonzero selection of 60 or 80.
- A subsequent 16-bit assignment.
- A final 8-bit zero assignment.
- No visible function calls, allocation, release, validation, or explicit return value.

The resource declaration and definition are not supplied. Its naming is insufficient evidence for a constant address, an invariant first output, or the absence of an additional memory read.

No runtime tests have been performed. All examples are static, conditional expectations rather than observed execution results.
