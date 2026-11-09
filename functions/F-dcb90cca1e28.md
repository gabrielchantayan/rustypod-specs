<!-- Trusted provenance: F-dcb90cca1e28; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dcb90cca1e28

## Purpose

F-dcb90cca1e28 applies a mutable configuration record to an existing destination object. Its established direct behavior comprises:

- Copying one-byte and four-byte fields.
- Normalizing six signed 32-bit inputs to nonnegative values.
- Raising two other signed 32-bit inputs to destination-derived lower bounds.
- Resolving a paired default by conditionally modifying two fields in the caller’s record.
- Conditionally writing a zero or one through a globally supplied address.
- Writing destination state bytes.
- Calling two dependencies before and after the direct updates.

“Configuration update” is inferred intent. Application-level field meanings are not established. The dependency calls do not establish locking, transactionality, ownership transfer, or rollback.

This contract describes static evidence from the supplied representation. No runtime tests have been performed.

## Interface

### Arguments and result

| Input | Role | Directly accessed extent |
|---|---|---|
| Destination base | Address of an existing mutable object | Through byte offset 0x1C7 |
| Record base | Address of a readable, conditionally writable record | Through byte offset 0x3F |

All offsets below are byte offsets relative to the specified base.

The represented scalar accesses are one byte or four bytes. Comparisons involving four-byte fields use signed 32-bit values. Uncompared four-byte copies preserve the raw field value; their application-level interpretation is unknown.

Normal completion returns a four-byte zero. No alternate return value or explicit error-reporting branch is visible.

### Destination layout

| Offset | Width | Direct use |
|---|---:|---|
| 0x08 | 1 byte | Conditionally read as a shared-update gate |
| 0x09 | 1 byte | Read first as a shared-update gate |
| 0x0A | 1 byte | Receives record 0x00 |
| 0x0B | 1 byte | Receives record 0x01 |
| 0x0C | 4 bytes | Receives normalized record 0x04 |
| 0x10 | 4 bytes | Receives normalized record 0x08 |
| 0x14 | 4 bytes | Receives normalized record 0x28 |
| 0x18 | 4 bytes | Receives normalized record 0x24 |
| 0x1C | 4 bytes | Receives normalized record 0x0C; later supplies a bound and default |
| 0x20 | 4 bytes | Receives normalized record 0x10; later supplies a bound and default |
| 0x24 | 4 bytes | Receives record 0x1C after default resolution |
| 0x28 | 4 bytes | Receives record 0x20 after default resolution |
| 0x2C | 4 bytes | Receives record 0x18, subject to the destination 0x20 bound |
| 0x30 | 4 bytes | Receives record 0x14, subject to the destination 0x1C bound |
| 0x34 | 1 byte | Receives record 0x2C; later selects the shared-store value |
| 0x3C | 4 bytes | Receives record 0x30 unchanged |
| 0x40 | 1 byte | Written with 1 near completion |
| 0x44 | 1 byte | Written with 1 near entry |
| 0x46 | 1 byte | Receives record 0x2D |
| 0x49 | 1 byte | Conditionally written with 2 |
| 0x78 | Unknown extent | Address passed to both dependencies |
| 0x1BC | 1 byte | Written with 1 near completion |
| 0x1BE | 1 byte | Receives record 0x34 |
| 0x1C0 | 4 bytes | Receives record 0x38 unchanged |
| 0x1C4 | 4 bytes | Receives record 0x3C unchanged |

Passing the address at 0x78 establishes neither the embedded region’s size nor the dependencies’ access limits.

### Preconditions and representation limits

Both arguments must support the accesses reached during execution. The record must permit four-byte writes at 0x1C and 0x20 when paired default resolution occurs.

On the enabled shared-update branch, the global address-bearing location must be readable and its resolved target must support the selected store. The supplied representation does not establish that store’s width.

There are no visible checks for null addresses, bounds, alignment, lifetime, or overlap. The direct access extents do not establish complete allocation-size requirements because the dependencies may access additional storage.

Four-byte fields are represented as native fields, not as an explicitly decoded external byte format. Byte order, alignment requirements, and behavior for invalid addresses are unresolved. Address validity is a precondition; no portable behavior for address overflow or out-of-object address calculation is established.

## Behavior

The following order is established by the supplied representation. It is not a verified machine-instruction trace and does not establish external observability of every intermediate store.

### Entry and initial byte updates

F-338d16f9ce72 is called with destination base plus 0x78 before any direct record read or destination write.

After it returns:

1. Destination 0x44 receives the byte value 1.
2. Record byte 0x00 is read and copied to destination 0x0A.
3. If that captured byte equals 3, destination 0x49 receives 2. Otherwise, this step does not write destination 0x49.
4. Record byte 0x01 is copied to destination 0x0B.

The comparison with 3 uses the captured record byte, not a subsequent read of destination 0x0A. A nonmatching value does not reset destination 0x49.

The record is first read after the initial dependency returns. Its contents are not captured at function entry.

### Six nonnegative normalizations

The following mappings occur in order:

| Order | Record offset | Destination offset |
|---:|---|---|
| 1 | 0x04 | 0x0C |
| 2 | 0x08 | 0x10 |
| 3 | 0x24 | 0x18 |
| 4 | 0x28 | 0x14 |
| 5 | 0x0C | 0x1C |
| 6 | 0x10 | 0x20 |

For each mapping, a signed 32-bit source value is captured and written unchanged to the destination. If the captured value is negative, the same four destination bytes are then overwritten with zero.

For stable storage, each completed mapping leaves the greater of the captured input and zero. Zero and 2,147,483,647 remain unchanged; all negative values, including −2,147,483,648, become zero. This operation uses comparison and replacement, not negation, so it requires no potentially overflowing signed arithmetic.

The represented negative case contains two stores. However, no memory read intervenes between the initial store and its zero replacement, and the decision uses the already captured input. For ordinary memory without intervening external mutation, overlap alone cannot make later reads distinguish those two stores from a single store of the normalized result. The replacement overwrites exactly the bytes written by the initial store.

An additional observable effect of the transient negative value would require unestablished conditions, such as external observation during that interval or special memory semantics. This contract does not assume such conditions.

Completed normalization writes can still affect subsequent source reads when the arguments overlap. That effect follows from the final bytes written by each mapping, not from an assumed observation of the transient negative value.

### Lower-bound updates

Record 0x18 is read as a signed 32-bit value and written to destination 0x2C. The captured input is compared with a signed read of destination 0x20. If the input is smaller, destination 0x20 is read again and copied to destination 0x2C.

The bound’s comparison read and replacement read are distinct in the representation. Equality does not cause a replacement store.

Next, record 0x14 is captured as a signed 32-bit value and written to destination 0x30. Before adjusting that destination against its bound, the following copies occur:

| Order | Record offset | Destination offset | Width |
|---:|---|---|---:|
| 1 | 0x2C | 0x34 | 1 byte |
| 2 | 0x2D | 0x46 | 1 byte |
| 3 | 0x30 | 0x3C | 4 bytes |

Destination 0x1C is then captured as a signed 32-bit bound. If the previously captured record 0x14 value is smaller, the captured bound replaces destination 0x30. Equality leaves the initial store unchanged.

This captured destination 0x1C value is also retained for the following default-resolution step.

With stable, non-overlapping storage, destination 0x2C becomes the greater of record 0x18 and normalized record 0x10; destination 0x30 becomes the greater of record 0x14 and normalized record 0x0C. Both results are then nonnegative. These simplified final-value statements require the stated storage assumptions.

### Paired default resolution

Record 0x1C is read as a signed 32-bit value and compared with exactly −1.

If it equals −1:

- Record 0x1C receives the destination 0x1C value captured during the preceding lower-bound step.
- Destination 0x20 is then read, and its raw four-byte value is written to record 0x20.

The first record write precedes the destination 0x20 read. An overlap between those locations can therefore affect the second replacement. The previous value of record 0x20 does not control whether either replacement occurs.

If record 0x1C is not −1, neither record field is directly modified by this step. Other negative values are not default sentinels.

After the optional replacements, record 0x1C is read and copied to destination 0x24. Record 0x20 is then read and copied to destination 0x28. These copies do not normalize their values.

### Conditional shared-target update

Destination 0x09 is read first. Destination 0x08 is read for this decision only if 0x09 was zero.

| Destination 0x09 | Destination 0x08 | Direct shared-update behavior |
|---|---|---|
| Nonzero | Not read | No target resolution or store |
| Zero | Any value except 5 | No target resolution or store |
| Zero | 5 | Resolve the global target and store |

On the enabled branch, destination byte 0x34 is read before the global address-bearing location supplies the store target:

| Destination 0x34 | Stored value |
|---|---:|
| Zero | 1 |
| Any nonzero byte | 0 |

This is zero/nonzero inversion. The control byte need not be limited to 0 or 1.

The target is resolved at this stage, after the preceding direct updates and before the late record copies. There is no established entry-time capture of the global address. Changes made by the initial dependency or earlier overlapping writes can therefore affect the selected target.

The shared store’s width, target identity, and target ownership are unresolved. No direct write to the global address-bearing location itself is shown, and no target validation is visible.

### Late copies and completion

After the optional shared store:

| Order | Record offset | Destination offset | Width |
|---:|---|---|---:|
| 1 | 0x34 | 0x1BE | 1 byte |
| 2 | 0x38 | 0x1C0 | 4 bytes |
| 3 | 0x3C | 0x1C4 | 4 bytes |

Destination 0x40 then receives 1, followed by destination 0x1BC receiving 1.

F-1333331b2da1 is called with destination base plus 0x78. If it returns normally, the function returns a four-byte zero. Neither dependency’s result is inspected.

The completion-byte writes occur before the final dependency, not after it. Their persistence at return requires that dependency and other actors to leave them intact.

### Aliasing and observation

The record is consumed incrementally rather than snapshotted. Completed destination writes, default-resolution writes, and the shared store may change later source reads if storage overlaps.

Relevant ordering distinctions include:

- A retained destination 0x1C value supplies the first default, while a later destination 0x20 read supplies the second.
- The global target is resolved after earlier direct writes.
- Late record reads follow the shared store and may observe its effects if their storage overlaps its target.
- Within each negative normalization, no intervening read exposes the initial negative store to a later read before its zero replacement.

No overlap validation or explicit disjointness requirement appears. Support for intentional overlapping arguments is not established. Concurrent mutation, atomicity, and synchronization guarantees are also unresolved.

## Side effects

Direct side effects are destination writes, conditional mutation of two record fields, and a conditional store through the globally resolved address.

The direct global-address read occurs only on the enabled shared-update branch. This does not constrain either dependency’s accesses.

No direct allocation, deallocation, ownership transfer, external I/O, or explicit error report is visible. These absences do not establish corresponding restrictions on the dependencies.

Neither argument address is visibly retained by a direct address-specific assignment. Raw copied fields could nevertheless represent addresses, and dependency retention is unknown.

There is no visible rollback. Invalid memory or a dependency that fails to return can leave earlier updates in place. Recovery, exception propagation, and failure-state semantics are not established.

## Dependencies

| Opaque ID | Call position | Argument | Result handling |
|---|---|---|---|
| F-338d16f9ce72 | Before direct record reads and destination writes | Destination base plus 0x78 | Not inspected |
| F-1333331b2da1 | After late copies and completion-byte writes | Destination base plus 0x78 | Not inspected |

Each call is unconditional if execution reaches its position.

Their placement suggests bracketed work around an embedded component. That is inferred intent, not a locking or lifecycle contract.

Neither dependency is assumed to preserve the destination, record, global address, pointed-to storage, or other global state. Neither is assumed to allocate, release, retain, or own memory, nor to return normally.

Consequently, direct-write outcomes become return-time postconditions only under suitable preservation assumptions for the final dependency and other actors.

The global address-bearing location is a separate data dependency. Its representation width, storage identity, and pointee contract are unresolved.

## Acceptance examples

These are conditional static examples, not executed tests. Unless stated otherwise, assume valid native-layout storage, disjoint relevant regions, no concurrent mutation, normally returning dependencies, and no dependency changes to relevant state.

1. **Normalization and bounds.**  
   With record 0x0C equal to −7, 0x10 equal to 12, 0x14 equal to −3, and 0x18 equal to 5, destination 0x1C becomes 0 and 0x20 becomes 12. Destination 0x30 becomes 0 and 0x2C becomes 12.

2. **Signed boundaries.**  
   A normalized source equal to −2,147,483,648 produces zero. A source equal to 2,147,483,647 is preserved. Neither outcome requires negating the input.

3. **Two represented stores, no overlap-only distinction.**  
   For a negative normalization input, the representation writes the negative value and then zero to the same four bytes. Even if those bytes overlap a later source field, an ordinary-memory read occurring after replacement sees the completed zero bytes, not a transient negative value. Overlap alone supplies no observation between the two stores.

4. **Paired replacement.**  
   With record 0x0C equal to 6, 0x10 equal to −9, 0x1C equal to −1, and 0x20 equal to 99, record 0x1C and 0x20 are replaced with 6 and 0. Destination 0x24 and 0x28 receive those values.

5. **Other negative values remain explicit.**  
   Record 0x1C equal to −2 does not trigger defaults. If record 0x20 is −17, destination 0x24 and 0x28 receive −2 and −17 unchanged.

6. **Conditional assignment is not reset.**  
   Record byte 0x00 equal to 3 causes destination 0x49 to receive 2. A value of 4 causes no direct write to that destination byte.

7. **Shared-store selection.**  
   With destination 0x09 equal to zero and 0x08 equal to 5, a control byte at destination 0x34 of zero selects a store of 1; a value of 255 selects a store of 0. Store width remains unresolved.

8. **Target changes before resolution.**  
   If F-338d16f9ce72 changes the global address to another valid target, and it remains unchanged thereafter, an enabled shared update uses that new target rather than the entry-time target.

9. **Disabled shared update.**  
   A nonzero destination 0x09 prevents the direct path from reading destination 0x08 or resolving the global target. Either dependency may still access those locations.

10. **Final dependency can affect final state.**  
    The late copies and completion-byte writes have occurred when F-1333331b2da1 is entered. Their presence at function return is not guaranteed without preservation assumptions.

## Uncertainties

The following remain **documented_only**:

- Application meanings, units, complete layouts, and valid ranges beyond visible comparisons.
- Native byte order, alignment rules, invalid-address behavior, and address-arithmetic limits.
- Global address representation width and storage identity.
- Shared-store width and target identity, lifetime, ownership, and synchronization requirements.
- Dependency effects, preservation guarantees, retention, memory management, and failure behavior.
- Whether intentional overlap is supported.
- Whether intermediate stores are externally observable.
- Whether special memory semantics make a transient normalization store significant.
- Meanings of the destination state bytes.

Overlap alone does not establish observability of the transient negative normalization store. Conversely, the absence of an intervening read does not establish that special memory or an external observer could never observe it.

## Evidence

Established static evidence includes access offsets and represented widths, signed comparisons, captured-value decisions, ordered direct writes, conditional record mutation, dependency placement, shared-update gating, late target resolution, and the constant normal return.

The negative normalization path explicitly represents an initial store followed by a zero replacement to the same four bytes, with no intervening memory read. This supports the two-store description but not an overlap-only claim of an additional observable effect.

Inferred intent is limited to configuration-style updating, normalization, paired default resolution, and dependency-bracketed work.

No dependency implementations, caller contracts, platform layout specification, or runtime observations were supplied. No runtime tests have been performed.
