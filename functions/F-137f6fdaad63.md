<!-- Trusted provenance: F-137f6fdaad63; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-137f6fdaad63

## Purpose

F-137f6fdaad63 coordinates dependency calls involving a primary resource and, conditionally, a secondary resource. It uses a two-component position and a threshold obtained from an owning object.

The established behavior is:

- Query the primary resource.
- Compare the fourth 32-bit word of the primary query result against the object’s threshold field plus 6.
- If that word is below the threshold, submit a primary operation, query the secondary resource, and submit a secondary operation.
- Otherwise, submit only the primary operation.

Coordinate arithmetic and resource/context arguments suggest rendering or composition. That is inferred intent, not an established contract. The supplied body does not establish that any dependency draws, clips, copies, stretches, or interprets its arguments as geometry.

## Interface

### Inputs

The function accepts three arguments.

| Input | Established use |
|---|---|
| Object reference | Address used to read resource fields and a signed threshold field. |
| Operation context | Opaque 32-bit value forwarded unchanged to operation dependencies. |
| Position reference | Address used to read two consecutive signed 32-bit words, designated X and Y. |

The object accesses are:

| Byte offset | Width | Interpretation within this function |
|---|---:|---|
| `0xBC` | 32 bits | Primary resource value; opaque bit pattern. |
| `0xB8` | 32 bits | Secondary resource value; opaque bit pattern. |
| `0x14C` | 32 bits | Signed threshold field. |

The position accesses are:

| Byte offset | Width | Interpretation |
|---|---:|---|
| 0 | 32 bits | Signed X coordinate-like value. |
| 4 | 32 bits | Signed Y coordinate-like value. |

The supplied representation uses a 32-bit object-address carrier and 32-bit integer arithmetic. A replacement must preserve the target’s address and field-access semantics; it must not silently substitute host-native widths.

Neither resource value nor the operation context is established to be a pointer, handle, index, or owned object.

### Output

There is no explicit return value.

Results are observable through dependency calls, their arguments, and their effects. The function does not inspect any dependency return value and provides no local success or failure indication.

### Record views and extent limits

The following names describe accesses made by this function, not complete dependency-defined types.

**Primary record view:** Four consecutive 32-bit words are used:

- A₀ at byte offset 0.
- A₁ at byte offset 4.
- A₂ at byte offset 8.
- A₃ at byte offset 12.

These accesses cover a 16-byte window beginning at the address supplied to the primary query. The complete access footprint of dependencies receiving that address is not supplied.

**Secondary record view:** The function explicitly reads only:

- B₀ at byte offset 0.
- B₂ at byte offset 8.

It also passes the secondary record’s starting address to two dependencies. These facts do **not** establish a four-word record, an exactly 16-byte extent, or the existence of a fourth field. The dependency-defined record size, intervening content, and full read/write footprint are unknown.

**Destination views:** Destination arguments use four ordered 32-bit word positions at offsets 0, 4, 8, and 12. Direct construction writes those positions. On one path, a dependency receives destination storage and four scalar inputs instead; its precise writes are unknown. A four-word view does not establish a dependency’s complete access footprint.

### Preconditions

Defined execution requires:

- Valid object storage for every reached field read.
- Valid position storage for every reached X or Y read.
- Valid dependency arguments under those dependencies’ actual contracts.
- Initialized, readable query-result words wherever the function subsequently reads them.
- Arithmetic within the defined signed 32-bit range, unless an independently established target arithmetic contract specifies otherwise.

No visible null checks, alignment checks, resource checks, bounds checks, or overflow checks enforce these requirements.

## Behavior

### Initial query and threshold decision

The function reads the primary resource field and invokes **F-c24767c70ad0**, passing the resource value and writable storage for the primary record.

After that call returns, it reads the threshold field and computes:

**T = threshold field + 6**

It saves T before making the branch decision. The comparison is signed and uses A₃ directly:

- **A₃ < T:** two-operation path.
- **A₃ ≥ T:** one-operation path.

Equality therefore selects the one-operation path. The comparison does not use A₃ − A₁.

The threshold field is explicitly read only once. Subsequent changes to that field do not cause another threshold calculation. Conversely, the initial query occurs before the threshold read, so a query-mediated change to the object can affect T.

### Two-operation path

#### First destination construction

The function invokes **F-961d28b6f6dd** with writable destination storage and four signed 32-bit scalar arguments:

| Scalar position | Value |
|---|---|
| First | X − 3 |
| Second | Y |
| Third | (A₂ − A₀) + X |
| Fourth | (A₃ − A₁) + Y |

These are the values supplied to the dependency. The body does not establish that the dependency stores them unchanged, nor that it normalizes, clips, or validates them.

X and Y occur in separate argument expressions. The supplied body does not establish a relative evaluation order for those expressions or guarantee a single position snapshot.

#### Primary operation

After the construction dependency returns, the function reads the primary resource field again and invokes **F-945301cd184f** with:

1. The newly read primary resource value.
2. The operation context.
3. The primary record address.
4. The destination address supplied to F-961d28b6f6dd.
5. The constant 255, represented as `0xFF`.

The queried resource and the submitted resource come from separate object-field reads. They need not be equal if intervening dependency behavior changes the object.

No result is checked.

#### Secondary query

After the primary operation returns, the function reads the secondary resource field and invokes **F-c24767c70ad0** with that value and writable secondary-record storage.

Only the subsequent reads of B₀ and B₂ establish which secondary-result words this function consumes directly. They do not establish the query’s complete output layout.

#### Second destination construction

After the secondary query returns, the function directly writes three words of another destination view. Its fourth word occupies the storage where T was saved before the branch.

The caller-established destination values are:

| Word position | Value |
|---|---|
| First | X − 3 |
| Second | Y + current A₃ |
| Third | (B₂ − B₀) + X |
| Fourth | Saved T |

The explicit post-query write order is:

1. Second word.
2. First word.
3. Third word.

The fourth word is not recomputed or explicitly rewritten at this stage.

The A₃ read for the second word occurs after both the primary operation and the secondary query. Because the primary record was passed by address, its contents cannot be assumed preserved without a dependency contract. This later A₃ is not necessarily the value used in the branch comparison.

Position values are also read again after those calls. Dependencies that can reach the position storage through external aliases may change the values used here.

The subtraction B₂ − B₀ precedes addition of X. No other secondary-result word participates in this function’s explicit destination arithmetic. Nevertheless, the secondary operation receives the record address, not merely the two consumed words; its accesses remain unknown.

#### Secondary operation

The function reads the secondary resource field again and invokes **F-168345203dbe** with:

1. The newly read secondary resource value.
2. The operation context.
3. The secondary record address.
4. The directly constructed destination address.
5. The constant 1.
6. The constant 0.

The queried and submitted secondary resources are separate reads. No result is checked.

### One-operation path

The function directly constructs a destination view with:

| Word position | Value |
|---|---|
| First | X − 3 |
| Second | Y |
| Third | (A₂ − A₀) + X |
| Fourth | T |

The explicit write order is second, first, third, then fourth.

The fourth value is T itself, not Y + T.

The function then reads the primary resource field again and invokes **F-945301cd184f** with the primary resource, operation context, primary record address, destination address, and 255.

This path does not invoke F-961d28b6f6dd, query the secondary resource, or invoke F-168345203dbe.

### Arithmetic and read ordering

All visible coordinate and threshold arithmetic uses signed 32-bit values.

Operations requiring representable results include:

- Threshold field + 6.
- X − 3.
- A₂ − A₀ and A₃ − A₁.
- Addition of X or Y to those differences.
- Y + current A₃.
- B₂ − B₀, followed by addition of X.

Intermediate results matter. A final mathematical result within range does not establish defined execution if an earlier subtraction overflows.

No explicit saturation, clamping, overflow rejection, or modular conversion is present. Behavior outside the non-overflowing domain is **documented_only**; modulo 2³² behavior is not established.

Negative positions, negative thresholds, and reversed endpoints are not explicitly rejected.

Sequencing between separate statements and dependency calls is established. Relative evaluation order within individual multi-argument calls or arithmetic expressions is not established beyond the arithmetic grouping shown above.

## Side effects

### Direct effects

The function directly:

- Reads the stated object fields.
- Reads position words at branch-dependent stages.
- Writes invocation-local storage.
- Invokes dependencies in the specified order.

There is no explicit write to the object or position storage. There is no explicit allocation, release, ownership transfer, retry, or rollback.

### Dependency-mediated effects and ownership

Dependencies receive addresses into invocation-local storage. Those addresses are available for the respective calls; the supplied body establishes neither permission to retain them after return nor a durable lifetime beyond the invocation.

The function body does not establish that dependencies:

- Preserve records passed by address.
- Preserve globals, object fields, or externally aliased position storage.
- Limit all accesses to the caller-visible record words.
- Allocate or release memory.
- Acquire or relinquish ownership of resources.
- Complete an external operation successfully.
- Return normally.

In particular, the secondary record’s full storage requirement and access footprint remain unknown. Its two explicit reads must not be used as a substitute for a dependency memory contract.

The saved-threshold and destination descriptions identify the caller’s writes. They do not grant an unknown dependency permission to overwrite unrelated storage, nor prove that every dependency respects an otherwise unstated memory boundary.

### Errors and interruption

There is no local error-handling branch.

If a dependency returns a failure status, the function does not visibly inspect it. If a dependency fails to return normally, later calls are not reached through the visible normal sequence.

Exception behavior, nonlocal exits, resource cleanup, and partial-operation effects are unknown.

## Dependencies

| Opaque ID | Established use | Unresolved contract |
|---|---|---|
| **F-c24767c70ad0** | Receives writable record storage and a 32-bit resource value; called for the primary resource and conditionally for the secondary resource. | Output layout, full extent, initialization guarantees, validation, ownership, errors, other effects. |
| **F-961d28b6f6dd** | Receives writable destination storage and four computed scalar values on the two-operation path. | Exact writes, transformations, footprint, return meaning, other effects. |
| **F-945301cd184f** | Receives primary resource, context, primary record address, destination address, and 255 on both paths. | Operation semantics, record mutation, ownership, status, external effects. |
| **F-168345203dbe** | Receives secondary resource, context, secondary record address, destination address, 1, and 0 on the two-operation path. | Record footprint, operation semantics, constant meanings, ownership, status, external effects. |

The two operation dependencies are distinct interfaces. Their semantics are not established as interchangeable.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

Unless an example states otherwise, assume valid dependency-specific storage, initialized consumed words, non-overflowing arithmetic, normal returns, stable object and position fields, and no dependency mutation affecting later observations. No example defines the secondary record’s complete layout.

### Below-threshold sequence

Assume:

- Primary words are (0, 0, 20, 8).
- The threshold field is 6, giving T = 12.
- Position is (100, 50).
- The secondary query makes B₀ = 2 and B₂ = 12.

Expected observations:

1. The two-operation path is selected because 8 < 12.
2. F-961d28b6f6dd receives scalar values (97, 50, 120, 58).
3. F-945301cd184f receives the resulting destination storage and constant 255.
4. The secondary query occurs after that primary operation.
5. F-168345203dbe receives destination words (97, 58, 110, 12), followed by constants 1 and 0.

The final fourth destination word is 12, not 62. No value or existence is asserted for any unused secondary-result field.

### Equality

Assume primary words (0, 0, 20, 12), threshold field 6, and position (100, 50).

The one-operation path constructs (97, 50, 120, 12) and invokes F-945301cd184f with 255. None of the two-operation-only calls occurs.

### Fourth-word comparison, not height comparison

Assume primary words (5, 100, 25, 108) and T = 12.

The one-operation path is selected because 108 ≥ 12. Substituting the difference 108 − 100 would select the wrong path.

### Later primary-record mutation

Assume initial A₃ = 8 and T = 12. The two-operation path is selected.

If the primary operation changes A₃ to 10, the secondary query leaves that value unchanged, and the later Y read yields 50, the second destination’s second word is 60. The branch is not reconsidered.

This example is conditional on such mutation being permitted by the actual dependency contracts.

### Repeated resource and threshold reads

If the initial query changes the primary resource field, the primary operation receives the subsequently read value, not necessarily the queried value.

If that query changes the threshold field to 14 before returning, the subsequent threshold calculation uses 14 and produces T = 20. A later change to the threshold field does not itself trigger recalculation.

## Uncertainties

The following remain **documented_only**:

- Resource types and record semantics.
- The secondary record’s complete size, fields, and dependency access footprint.
- Complete dependency access footprints for other passed storage.
- Destination interpretation as corners, extents, or another format.
- Meanings of 255, 1, and 0.
- Dependency return types, failure conventions, and nonlocal exits.
- Mutation of records, globals, object fields, or aliased position storage.
- Allocation, release, retention, and ownership inside dependencies.
- Arithmetic behavior outside the defined signed 32-bit domain.
- Alignment and validity requirements beyond the visible accesses.
- Unspecified expression-evaluation ordering.
- Concurrency and input-stability requirements.

Apparent rendering intent resolves none of these unknowns.

## Evidence

**Established static evidence:** The supplied body supports the field offsets, 32-bit word accesses, signed comparison, arithmetic grouping, explicit destination-write order, dependency order, constants, repeated resource reads, and absence of local status handling described above.

For the secondary result specifically, the body establishes reads at offsets 0 and 8 and address passing to dependencies. It does not establish a four-word or exactly 16-byte secondary record.

**Inferred intent:** Coordinate-like arithmetic and resource/context argument patterns suggest rendering or composition. This inference does not define dependency behavior.

**Unavailable evidence:** Dependency implementations, complete record definitions, ownership contracts, target overflow guarantees, and execution traces were not supplied.

No runtime tests have been performed.
