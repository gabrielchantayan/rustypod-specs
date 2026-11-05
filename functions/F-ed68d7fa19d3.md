<!-- Trusted provenance: F-ed68d7fa19d3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ed68d7fa19d3

## Purpose

F-ed68d7fa19d3 accepts an object address, two signed coordinates, and two signed dimensions. When both dimensions are nonnegative, it stores the four geometry inputs in the object, prepares two derived records, submits them to F-bbb623bce77b, and invokes F-f385d619c451.

The derived records are consistent with a center and half-extents scaled by 65,536. That interpretation is inferred intent; the established behavior is the arithmetic, memory access, and call ordering described below.

A negative dimension selects a separate path: the function reads an external datum from a fixed location and passes its value, together with the object argument, to F-375c8199e7c2. The location is fixed in the supplied representation; its contents are not established as constant.

## Interface

### Arguments and result

| Argument | Visible representation | Established use |
|---|---|---|
| Object | 32-bit address-bearing integer | Base for object-relative addresses and argument to dependencies |
| X | Signed 32-bit integer | First stored coordinate and input to the first derived record |
| Y | Signed 32-bit integer | Second stored coordinate and input to the first derived record |
| Width | Signed 32-bit integer | First stored dimension; negative values select the error path |
| Height | Signed 32-bit integer | Second stored dimension; negative values select the error path |

There is no explicit return value. Observable results are memory accesses, direct writes, dependency invocations, and effects produced by those dependencies.

Zero dimensions are accepted by the visible branch condition. Negative coordinates are also accepted; only Width and Height participate in dimension rejection.

### Object-relative layout

The accepted-dimension path uses these byte offsets:

| Offset | Direct use |
|---|---|
| 0x108 | Store X in a four-byte word |
| 0x10C | Store Y in a four-byte word |
| 0x110 | Store Width in a four-byte word |
| 0x114 | Store Height in a four-byte word |
| 0x134 | Pass this address as the first conversion target |
| 0x13C | Read one opaque four-byte word before the first conversion |
| 0x140 | Pass this address as the second conversion target |
| 0x148 | Read one opaque four-byte word after the first conversion returns |

The four geometry fields are adjacent, nonoverlapping words. Each opaque-word location is eight bytes beyond its associated conversion target. This relationship does not establish the size, type, or complete layout of either target.

The function performs no visible validation of the object's address, allocation size, alignment, or access permissions. Correct execution requires a memory environment that permits the accesses actually performed. Dependency access requirements may be broader and are unknown.

### Temporary record

The supplied temporary-storage layout supports a seven-word record beginning at the address passed to F-bbb623bce77b:

| Byte offset | Value prepared for the first invocation |
|---|---|
| 0 | Scaled horizontal center |
| 4 | Scaled vertical center |
| 8 | Opaque word read from object offset 0x13C |
| 12 | X |
| 16 | Y |
| 20 | Width |
| 24 | Height |

Each entry occupies four bytes. This layout is supported by the supplied representation; it does not establish how many bytes the dependency reads or writes.

Height at temporary offset 24 is initialized **before** the four object geometry stores. The other six words are initialized later. Before the second conversion, the caller explicitly replaces only offsets 0, 4, and 8.

### External error datum

The negative-dimension path evaluates a datum at a fixed external location and passes the resulting value as the second argument to F-375c8199e7c2.

The supplied representation does not establish:

- The datum's contents or semantic type.
- Its storage width.
- Whether other code can modify it.
- Whether its value denotes an address, an identifier, or another kind of argument.

The contract therefore distinguishes the fixed location from the value obtained when it is read. It does not treat that value as an immutable literal.

## Behavior

### Path selection

If Width is negative or Height is negative, the function takes the negative-dimension path. Otherwise, it takes the accepted-dimension path.

There is no additional visible rejection for zero dimensions, negative coordinates, large positive dimensions, or arithmetic overflow.

### Negative-dimension path

The established sequence is:

1. Obtain the value of the external datum from its fixed location while preparing the error call.
2. Invoke F-375c8199e7c2 with the object argument and that obtained value.
3. Return if the dependency returns normally.

This path performs none of the accepted-path temporary-record initialization, none of the four direct geometry stores, neither object-relative opaque-word read, neither conversion call, and no call to F-f385d619c451.

It does perform an external-data access. The absence of object-relative writes by this function does not mean that the object remains unchanged: F-375c8199e7c2 may mutate it or have other effects. Its reporting, exception, termination, and return behavior are unavailable.

The function itself does not enter the accepted path after a normal return from F-375c8199e7c2.

### Accepted-dimension path: ordered operations

For nonnegative Width and Height, the supplied representation establishes this order:

1. Initialize temporary offset 24 with Height.
2. Store X at object offset 0x108.
3. Store Y at object offset 0x10C.
4. Store Width at object offset 0x110.
5. Store Height at object offset 0x114.
6. Read the opaque four-byte word at object offset 0x13C and place it at temporary offset 8.
7. Compute and store the scaled vertical center at temporary offset 4.
8. Compute and store the scaled horizontal center at temporary offset 0.
9. Initialize temporary offset 12 with X.
10. Initialize temporary offset 16 with Y.
11. Initialize temporary offset 20 with Width.
12. Invoke F-bbb623bce77b with the target address at object offset 0x134 and the temporary-record address.
13. After that invocation returns, read the opaque four-byte word at object offset 0x148 and replace temporary offset 8 with it.
14. Replace temporary offset 4 with the scaled vertical half-extent.
15. Replace temporary offset 0 with the scaled horizontal half-extent.
16. Invoke F-bbb623bce77b with the target address at object offset 0x140 and the same temporary-record address.
17. After that invocation returns, invoke F-f385d619c451 with the object argument.
18. Return if the final dependency returns normally.

The first temporary Height write therefore precedes all direct object stores, rather than belonging to preparation performed after the first opaque-word read.

All four geometry stores precede both conversions. The read at object offset 0x148 occurs after the first conversion, not before it. Any change to that location during the first dependency invocation can affect the second record.

### Derived values

For accepted dimensions, define the integer half-extents as:

- Horizontal half-extent: Width divided by 2, discarding the remainder.
- Vertical half-extent: Height divided by 2, discarding the remainder.

Because these operands are nonnegative, an odd dimension is rounded down. A dimension of 5 contributes a half-extent of 2.

The first record contains:

- At offset 0: X plus the horizontal half-extent, multiplied by 65,536.
- At offset 4: Y plus the vertical half-extent, multiplied by 65,536.
- At offset 8: the opaque word read from object offset 0x13C.
- At offsets 12 through 24: X, Y, Width, and Height.

The second preparation replaces:

- Offset 0 with the horizontal half-extent shifted left by 16 bits.
- Offset 4 with the vertical half-extent shifted left by 16 bits.
- Offset 8 with the opaque word read from object offset 0x148.

For representable nonnegative results, the shifts correspond to multiplication by 65,536.

The function uses the supplied geometry arguments for these calculations; it does not reload them from the object fields. Dependency changes to the stored geometry therefore do not, by themselves, replace the inputs to the second calculations.

The caller does not restore temporary offsets 12 through 24 after the first conversion. If that dependency modifies those words, the second invocation can receive the modified contents. Preservation of the original trailing words requires an independent dependency guarantee.

### Integer widths and boundaries

Geometry inputs, arithmetic intermediates, and stored geometry words are represented as signed 32-bit values. No widening, saturation, explicit overflow check, or overflow-rejection branch is visible.

Arithmetic safety requires separate consideration of:

- Addition of a half-extent to a coordinate.
- Multiplication of the resulting center by 65,536.
- Left-shifting a nonnegative half-extent by 16 bits.

A scaled center is mathematically representable as a signed 32-bit integer exactly when the unscaled center is between −32,768 and 32,767 inclusive. A nonnegative scaled half-extent is representable when the half-extent is at most 32,767. Dimensions from 0 through 65,535 meet the latter condition, but do not independently guarantee safe center arithmetic.

The function does not enforce these bounds. Exact outcomes for unrepresentable arithmetic are **documented_only**: the supplied representation does not establish wrapping, saturation, trapping, or another result. Object-address arithmetic at invalid or overflowing addresses is likewise not assigned an invented behavior.

## Side effects

### Direct effects

On the accepted path, the function:

- Initializes and reuses invocation-local temporary storage.
- Overwrites four geometry words in the object.
- Reads two opaque object words at distinct points in the call sequence.
- Invokes F-bbb623bce77b twice with different object-relative targets.
- Invokes F-f385d619c451 after both conversions return.

On the negative path, the function:

- Reads the datum at a fixed external location.
- Invokes F-375c8199e7c2 with the object and the obtained value.

No allocation, deallocation, or explicit ownership-transfer operation is visible in the function itself. This observation does not restrict dependency behavior.

### Partial progress and failure

The update is not visibly transactional. All geometry stores occur before the first conversion, and there is no visible rollback.

If a dependency does not return normally, later operations need not occur. Earlier stores and dependency effects are not visibly reversed. For example:

- Failure during the first conversion can occur after all geometry stores but before the second opaque-word read.
- Failure during the second conversion can prevent F-f385d619c451 from being reached.
- A nonreturning error dependency prevents the function's normal return.

No status result from either conversion or the final dependency is visibly examined.

### Ownership, lifetime, and aliasing

The same temporary-record address is used for both conversions. Its visible storage duration is limited to this invocation; the function supplies no persistent copy for retention after return.

Passing target and temporary addresses does not establish ownership transfer, permission to retain them, or a release obligation. Dependency-side memory management is unknown.

Both conversion targets lie within the supplied object address range, but their actual access extents are unavailable. No assumption is made that dependencies preserve temporary data, object fields, global state, or object validity.

## Dependencies

| Opaque ID | Established invocation | Unavailable behavior |
|---|---|---|
| F-375c8199e7c2 | Negative-dimension path; receives the object and the value read from the fixed external location | Error interpretation, return behavior, mutation, global effects, ownership and lifetime effects |
| F-bbb623bce77b | Twice on the accepted path; receives an object-relative target and the shared temporary-record address | Record interpretation, read/write extent, source mutation, target mutation, retention, failure and ownership behavior |
| F-f385d619c451 | Receives the object after both conversions return | Update or finalization semantics, mutation, global effects, failure and ownership behavior |

No dependency is assumed to preserve globals, preserve the object, preserve the temporary record, or release memory.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Accepted-path examples assume accessible storage, representable arithmetic, and dependencies that return normally.

### Even dimensions

For X = 10, Y = 20, Width = 6, Height = 8:

- Temporary offset 24 first receives 8, before the object geometry stores.
- Object offsets 0x108 through 0x114 receive 10, 20, 6, and 8 in order.
- Half-extents are 3 and 4.
- The first record's first two words are 851,968 and 1,572,864.
- The second preparation replaces those words with 196,608 and 262,144.
- Conversion targets are object offsets 0x134 and 0x140, in that order.
- F-f385d619c451 follows the second conversion.

The third word of each record comes from its specified object read, not from geometry arithmetic.

### Odd dimensions

For X = 0, Y = 0, Width = 5, Height = 3:

- Half-extents are 2 and 1.
- The first record's scaled centers are 131,072 and 65,536.
- The second preparation writes those same two values.

Preserving fractional halves would not match the supplied arithmetic.

### Zero dimensions and negative coordinates

For X = −2, Y = 3, Width = 0, Height = 0:

- The accepted path executes.
- The first record's first two words are −131,072 and 196,608.
- The second preparation writes zero to both corresponding words.
- Both conversions and the final dependency are reached if preceding calls return.

There is no visible zero-size early return.

### Negative dimension and external value

For Width = −1 and Height = 4:

- The external datum is evaluated before F-375c8199e7c2 is invoked.
- The dependency receives the object and the value obtained from that access.
- No accepted-path temporary initialization, geometry store, conversion, or final dependency call occurs.

If two invocations observe different contents at the fixed external location, their second error arguments may differ. The fixed location does not require a fixed value.

The same path applies when Height alone is negative or both dimensions are negative.

### Mutation during the first conversion

If the first conversion changes object offset 0x148 and returns, the second preparation uses the word read from that location afterward.

If the first conversion changes temporary offset 12 or 24, the caller does not restore that word before the second invocation. Acceptance must not require restoration without an additional dependency contract.

### Arithmetic beyond the safe range

For Width = 65,536, the horizontal half-extent is 32,768. Its mathematical scaled value is 2,147,483,648, outside the positive signed 32-bit range.

Width is nevertheless nonnegative and does not select the error path. The exact arithmetic outcome is documented_only; no particular resulting word or successful completion is promised.

## Uncertainties

- **documented_only — Semantic interpretation:** Rectangle, center, half-extents, and fixed-point terminology describe inferred intent, not established consumer types.
- **documented_only — External datum:** Its value, width, interpretation, mutability, and access properties are not established. Its fixed location is established.
- **documented_only — Arithmetic overflow:** Exact outcomes of unrepresentable signed addition, multiplication, shifts, or address calculations are unavailable.
- **documented_only — Dependency contracts:** Access bounds, mutation, ownership, retention, global effects, and exceptional control flow are unknown.
- **documented_only — Temporary-record consumption:** Seven-word preparation is supported by the layout; dependency consumption of all seven words is not established.
- **documented_only — Memory environment:** Byte order, synchronization, concurrent modification, and special access semantics are unspecified.
- **documented_only — Final object state:** Direct stores are established, but subsequent dependencies may alter or invalidate the object.

## Evidence

The evidence is the supplied static function representation: signed arguments, the dimension branch, the external-datum expression on the negative path, object-relative accesses, temporary-storage layout, arithmetic, and ordered dependency calls.

In particular, it establishes both the external-datum access before the error call and the temporary Height initialization before the accepted-path object stores. Neither an absence of global-data access nor a wholly post-read temporary initialization sequence is supported.

Geometric meaning is inferred from halving dimensions, adding coordinates, and scaling by 65,536. Callee semantics and unresolved arithmetic outcomes are not inferred.

No runtime tests have been performed. The acceptance examples are derived behavioral criteria, not observed execution results.
