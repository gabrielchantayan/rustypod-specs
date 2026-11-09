<!-- Trusted provenance: F-6feec85f6f2a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6feec85f6f2a

## Purpose

F-6feec85f6f2a propagates a per-record 32-bit value into selected fields of linked nodes and marks each visited node with a corresponding flag.

Two update modes are established:

- **Replacement:** copy the record’s current value into each visited node.
- **Adjustment:** add the record’s current-minus-reference difference to each visited node’s existing value.

The selector determines the destination field and flag bit. For selectors zero and one, bits in an enclosing object determine the update mode. Every other selector uses adjustment mode while selecting the same destination field and flag bit as selector one.

The interpretation of these fields as positions, coordinates, or another application-specific quantity is **inferred intent**, not established behavior. The observable contract is expressed in terms of byte offsets, fixed-width values, traversal, and writes.

## Interface

### Inputs and result

The function has two explicit inputs:

| Input | Established representation | Role |
|---|---|---|
| Object address | 32-bit unsigned address-like value | Base for descriptor selection and, for selectors zero and one, mode flags |
| Selector | 32-bit signed integer | Selects a descriptor, destination field, and update-mode rule |

There is no explicit return value. Results are memory modifications.

No length, validity, ownership, or error-result parameter is supplied.

### Arithmetic model

The supplied representation establishes 32-bit address and value operations and 16-bit node flags.

For this contract:

- Address additions and multiplications are evaluated as 32-bit bit-pattern arithmetic.
- Arithmetic results retain their low 32 bits.
- The record-loop comparison is an **unsigned 32-bit address comparison**.
- Node adjustment likewise retains the low 32 bits; no saturation, range check, or overflow report is present.

Some intermediate operations are displayed with signed types. That does not establish a separate checked signed-arithmetic interface. Behavior at signed-language overflow boundaries is not independently established by this static representation; the contract describes the represented fixed-width operations.

### Object and descriptor layout

Let the object address be the base and the selector be the index. The selected descriptor origin is:

**object address + selector × 28 bytes**, using 32-bit arithmetic.

The following fields are accessed relative to that origin:

| Byte offset | Width | Meaning in this function |
|---|---:|---|
| `0x3C` | 32 bits | Record-count bit pattern used to calculate the record endpoint |
| `0x44` | 32 bits | Initial record address |

For selectors zero and one, a 32-bit mode-flags field is also read at offset `0x6C` from the **original object address**, not from the selected descriptor origin.

Descriptor selection is not validated. Negative selectors and selectors above one still participate in address calculation.

### Record layout

Records are traversed with a fixed stride of `0x30` bytes.

| Byte offset | Width | Meaning in this function |
|---|---:|---|
| `0x04` | 32 bits | Reference value for adjustment mode |
| `0x08` | 32 bits | Current value; replacement source or adjustment minuend |
| `0x28` | 32 bits | Address of the initial group; also reread as the group-cycle termination anchor |

The reference field is used as a 32-bit arithmetic operand. Although the supplied representation gives it a pointer-like type, this function does not dereference that field’s value.

### Group layout

| Byte offset | Width | Meaning in this function |
|---|---:|---|
| `0x0C` | 32 bits | Successor-group address |
| `0x24` | 32 bits | First-node address |
| `0x28` | 32 bits | Inclusive last-node address |

### Node layout

| Byte offset | Width | Meaning in this function |
|---|---:|---|
| `0x00` | 16 bits | Flags |
| `0x10` | 32 bits | Destination for selector zero |
| `0x14` | 32 bits | Destination for every nonzero selector |
| `0x20` | 32 bits | Next-node address |

A node’s first and last markers are group fields, not fields in the node itself.

## Behavior

### 1. Select the record range and update mode

The function reads the initial record address from the selected descriptor.

For selector zero, it next reads the object’s mode-flags field. It then reads the count field and computes the endpoint:

**initial record address + count × 48 bytes**, with 32-bit arithmetic.

For selector one, the count and endpoint are obtained before the mode-flags field is read.

For any other selector, the mode-flags field is not read.

The mode and node destination are:

| Selector | Replacement condition | Node destination | Flag added |
|---|---|---|---|
| Zero | Object mode flags contain bit `0x01` | Offset `0x10` | `0x0040` |
| One | Object mode flags contain bit `0x02` | Offset `0x14` | `0x0080` |
| Any other value | Never | Offset `0x14` | `0x0080` |

The chosen mode remains fixed for the entire invocation. Later changes to the object’s mode-flags field do not cause a mode change.

The function processes a record only while its current address is unsigned-less-than the computed endpoint. After a record completes, the current address advances by `0x30` bytes.

This is an address-bounded traversal, not an independently decremented count loop. A count whose bit pattern would be negative as a signed integer is not rejected. Multiplication or endpoint wraparound can change which records, if any, are visited.

### 2. Establish the record’s update information

In adjustment mode, each visited record begins with these reads, in order:

1. Its 32-bit current value at offset `0x08`.
2. Its 32-bit reference value at offset `0x04`.
3. Its initial-group address at offset `0x28`.

The adjustment is the low 32 bits of current value minus reference value. It is calculated once for that record and reused for all nodes reached through that record.

In replacement mode, the function initially reads only the record’s initial-group address. It does not cache the replacement value for the record: that value is read separately for each visited node.

Neither mode writes the record’s current or reference fields as an explicit operation.

### 3. Traverse groups and their nodes

Group traversal executes at least once for every entered record. There is no null-group or empty-group check.

For each group:

1. Read its first-node address at offset `0x24`.
2. Update that node.
3. Read the group’s last-node address at offset `0x28`.
4. If the updated node equals that last-node address, finish this group.
5. Otherwise, read the updated node’s next-node address at offset `0x20` and continue.

The last node is included in the updates. Its next-node field is not read.

The last-node address is reread after every node update; it is not cached on group entry.

After completing a group, the function:

1. Reads the completed group’s successor address at offset `0x0C`.
2. Rereads the current record’s initial-group field at offset `0x28`.
3. Finishes the record if those two addresses are equal.
4. Otherwise, treats the successor as the next group and continues.

For stable, well-formed data, this describes a circular group chain returning to the record’s initial group. The function does not validate that topology. Its actual termination condition uses the **reread record field**, not an immutable copy of the original group address.

### 4. Update each node

#### Adjustment mode

For each visited node, the function:

1. Reads the selected 32-bit destination field.
2. Adds the record’s cached adjustment.
3. Writes the low 32 bits to the same field.
4. Reads the node’s 16-bit flags.
5. Writes those flags with the selected bit set.

The current value in the record is not reread for subsequent nodes in this mode.

#### Replacement mode

For each visited node, the function:

1. Reads the current record’s 32-bit value at offset `0x08`.
2. Writes that value to the selected node destination.
3. Reads the node’s 16-bit flags.
4. Writes those flags with the selected bit set.

The replacement source is therefore observed once per node. It is not guaranteed to be identical across all nodes if the source field changes during execution or aliases a written field.

#### Shared properties

The 32-bit destination write precedes the 16-bit flag write. Existing flag bits are retained by the bitwise OR operation.

Only one of the two destination fields is explicitly written for a given selector. If a node is reached more than once, it is updated more than once. There is no visited-node tracking or duplicate suppression.

These are the access orders represented by the supplied body. Atomicity, synchronization, and hardware-level ordering guarantees are not established.

## Side effects

### Memory writes

The explicit writes are limited to each visited node’s:

- Selected 32-bit destination field.
- 16-bit flags field.

This describes writes by address and width, not an assurance that unrelated logical objects cannot change. If those addresses overlap descriptor fields, record fields, group links, other nodes, or each other, the writes can affect later reads.

The routine does not enforce disjoint storage.

### Ownership and lifetime

No allocation, deallocation, ownership transfer, reference-count operation, or external call appears in the supplied body.

The function uses caller-provided reachable memory without establishing its lifetime. Valid storage must remain accessible for every read and write actually reached.

### Errors and partial progress

No explicit validation or recoverable error path is present.

In particular, there is no check for:

- Invalid object or selected-descriptor addresses.
- Out-of-range selectors.
- Invalid record ranges.
- Null group or node addresses.
- Missing last nodes.
- Group chains that fail to return to the termination anchor.
- Arithmetic overflow.
- Overlapping structures.

Malformed data can lead to invalid memory accesses or nontermination. The resulting platform-specific fault behavior is not established.

There is no rollback mechanism. If execution stops after some node writes, those writes have already been performed.

### Practical preconditions for normal completion

A conventional, terminating use requires:

- Readable descriptor fields and any applicable mode-flags field.
- An intended record range consistent with the unsigned endpoint comparison.
- Readable record and group fields.
- Writable destination and flag fields for every reached node.
- Node chains that reach their group’s last-node address.
- Group successors that eventually satisfy the record-anchor comparison.

Stable, nonoverlapping structures make the traversal predictable. They are caller obligations for such use, not checks performed by the function.

## Dependencies

No function calls are visible in F-6feec85f6f2a. There are therefore no identified opaque-function dependencies to enumerate.

The function depends directly on the memory structures described above and on the execution environment’s ability to perform the represented 16-bit and 32-bit accesses.

No claim about an unknown callee preserving globals or managing memory is necessary: no callee is present in the supplied body.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless stated otherwise, assume valid, stable, nonoverlapping storage, no concurrent mutation, one entered record, and terminating group and node chains.

### Zero records

If the count is zero, the computed endpoint equals the initial record address. No record is entered and no node is written.

Descriptor reads still occur. For selector zero or one, the applicable mode-flags read also occurs even though no record is processed.

### Selector-zero replacement

Suppose selector zero is used, mode bit `0x01` is set, the record’s current value is 300, and a visited node initially has:

- Destination at `0x10`: 12.
- Destination at `0x14`: 90.
- Flags: `0x0005`.

After that visit:

- Destination at `0x10` is 300.
- Destination at `0x14` remains 90.
- Flags are `0x0045`.

The reference field does not contribute to this result.

### Selector-one adjustment

Suppose selector one is used, mode bit `0x02` is clear, and the record has current value 90 and reference value 100.

The adjustment is the 32-bit representation of −10. A node whose destination at `0x14` initially contains 25 receives 15. Flags initially equal to `0x0001` become `0x0081`.

The destination at `0x10` is not explicitly written.

### Other selectors do not enable replacement

Suppose selector two is used and the selected descriptor is valid. Even if the object’s mode-flags storage contains both low bits set, this invocation uses adjustment mode, destination offset `0x14`, and flag `0x0080`.

The function does not read the mode-flags field for selector two. This example does not imply that descriptor two exists in every valid object.

### Fixed-width addition wraps

In adjustment mode, suppose the cached adjustment is one and a visited destination contains `0xFFFFFFFF`.

The destination becomes `0x00000000`. No overflow error or saturation is produced.

### Single-node, single-group traversal

Suppose a group’s first-node and last-node fields both identify the same valid node, its successor points to itself, and the record’s group anchor remains unchanged.

The node is updated once. Its next-node field is not read. The group successor and reread record anchor compare equal, ending that record.

### Repeated node visits

Suppose two groups each designate the same node as their sole node.

In adjustment mode with adjustment seven, a destination initially equal to ten becomes 24 after both visits. The flag is set on both visits.

In replacement mode with a stable source value, both visits write that source value.

### Wrapped endpoint skips processing

Suppose the initial record address is `0xFFFFFFE0` and the count is one. The computed endpoint is `0x00000010`.

The unsigned initial-address comparison is false. No record is processed, despite the count being one.

## Uncertainties

### Established static evidence

The supplied body establishes:

- Access widths and byte offsets.
- Selector-dependent destinations, flag bits, and update modes.
- Record stride and endpoint-based unsigned traversal.
- Per-record caching of the adjustment.
- Per-node reading of the replacement source.
- Inclusive node traversal.
- Post-group successor and anchor reads.
- The absence of explicit validation, calls, allocation, and rollback.

### Inferred intent

The two destinations and two flag bits suggest paired components of a larger state. Replacement versus current-minus-reference adjustment suggests propagating an absolute setting or a change.

Those interpretations are not required by the observed operations and are not normative field names.

### documented_only unknowns

The supplied material does not establish:

- Application-level meanings or units.
- The supported selector domain.
- Allocation sizes, alignment guarantees, or object lifetime rules.
- Whether other code guarantees the required graph topology.
- Whether overlapping structures are intentionally supported.
- Any concurrency or synchronization contract.
- Target-specific handling of invalid or misaligned accesses.
- Source-language semantics beyond the represented fixed-width operations.
- Whether a count causing endpoint wraparound is possible in intended use.

These unknowns do not justify adding validation, clamping, alternate traversal, or recovery behavior to the contract.

## Evidence

This contract is based solely on static inspection of the supplied body associated with F-6feec85f6f2a.

The evidence supports the memory layouts, branch conditions, access ordering, arithmetic widths, traversal rules, and writes described above. No caller, structure declaration, external implementation, or runtime observation was supplied.

No runtime tests have been performed. The acceptance examples are consequences of the stated static behavior under their explicit preconditions, not reported execution results.
