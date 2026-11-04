<!-- Trusted provenance: F-3fcc73b1ebe8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3fcc73b1ebe8

## Purpose

F-3fcc73b1ebe8 conditionally handles a record using state stored in a supplied object. It recognizes one record tag, attempts a primary match, then attempts a secondary match if the primary match fails.

A successful primary match invokes an object-associated indirect operation. A successful secondary match invokes F-5710a8628cba, obtains an 8-bit result, and forwards that result to a different object-associated indirect operation.

The function returns the 32-bit value `1` after either successful dispatch, provided the invoked dependencies return normally. It returns the 32-bit value `0` when neither path is selected.

**Inferred intent:** this resembles a record or event filter with two handling routes. The meanings of the tag, matching fields, global values, and dispatched operations are not established.

## Interface

### Inputs

The function receives two address-bearing arguments:

- **Object:** an address used to read object state and resolve indirect operations.
- **Record:** an address used to read a tag and two matching fields.

The supplied representation uses 32-bit integer fields and byte-based address offsets. Its address-bearing values are represented in a 32-bit form. This contract describes that representation; it does not establish portability to a different pointer width or object ABI.

### Record layout

| Byte offset | Width | Role in this function |
|---|---:|---|
| `+0x04` | 32 bits | Record tag; must equal `0x00000013` to proceed |
| `+0x08` | 32 bits | First matching field |
| `+0x0C` | 32 bits | Second matching field |

No record field at offset zero is explicitly read. No record field is explicitly written.

### Object layout

| Byte offset | Width | Role in this function |
|---|---:|---|
| `+0x00` | Address-sized, represented as 32 bits | Address of an indirect-operation table |
| `+0x40` | 32 bits | Primary expected first field |
| `+0x44` | 32 bits | Primary expected second field |
| `+0x98` | 32 bits | Secondary expected second field |
| `+0x9C` | 32 bits | Secondary enable field; any nonzero value enables consideration |

The indirect-operation table is accessed at these byte offsets:

| Table offset | Invocation |
|---|---|
| `+0xD4` | Receives the object address |
| `+0xA0` | Receives the object address and an 8-bit result from F-5710a8628cba |

The target function identities and full calling conventions are unresolved.

### Global inputs

Two distinct 32-bit global values participate in matching:

- **Primary alternate value:** an alternative to the object’s primary expected first field.
- **Secondary required value:** the required first record field for secondary handling.

These are descriptive names for unresolved data locations, not recovered semantic names. Their values, initialization, mutability, and relationship to one another are unknown.

### Output

On normal return, the result is a 32-bit value:

- `1`: a matching path was selected and its final indirect operation returned normally.
- `0`: no matching path was selected.

The result is not copied from a dependency’s return value. It does not establish that a dispatched operation achieved any higher-level effect.

### Preconditions

Every memory access actually reached must be valid for the represented layout. A selected indirect target must be callable with the arguments supplied.

The function performs no explicit null checks, bounds checks, pointer validation, table validation, or recovery from invalid memory. Preconditions are path-dependent: a nonmatching tag does not require access to the remaining record fields, object state, globals, or operation table.

## Behavior

### 1. Tag gate

The function first reads the 32-bit record tag at `+0x04`.

If it is not `0x00000013`, the function returns `0` immediately. No explicit object read, global read, matching-field read, or dependency invocation occurs on this path.

### 2. Primary match

For a matching tag, the function:

1. Reads the record’s second field at `+0x0C`.
2. Reads the object’s primary expected second field at `+0x44`.
3. Compares those 32-bit values for equality.

If these values differ, the primary path fails without reading the object’s primary expected first field or the primary alternate global value.

If they are equal, the function reads:

1. The record’s first field at `+0x08`.
2. The object’s primary expected first field at `+0x40`.

An exact match between these first-field values is sufficient for primary handling. In that case, the primary alternate global value is not read.

If the first-field values differ, the function reads the primary alternate global value. Primary handling is then selected only if the previously read record first field equals that global value.

Thus, primary handling requires both:

- The record second field equals the object’s primary expected second field.
- The record first field equals either the object’s primary expected first field or the primary alternate value.

On primary success, the function reads the object’s operation-table address, resolves the target at table offset `+0xD4`, and invokes it with the object address. If that invocation returns normally, the function returns `1`.

The secondary path is not considered after primary success.

### 3. Secondary match

The secondary path is considered only after the primary path fails.

The function first reads the object’s secondary enable field at `+0x9C`. Zero disables this path and causes a return of `0`.

For a nonzero enable field, the function reads the object’s secondary expected second field at `+0x98`. It compares that value with the record second field already read during the primary attempt.

If those values differ, the function returns `0`. It does not read the secondary required global value or invoke a dependency.

If they are equal, the function:

1. Reads the record’s first field at `+0x08`.
2. Reads the secondary required global value.
3. Compares these 32-bit values.

This is a fresh read of the record first field. If the primary path previously read that field, the secondary comparison does not simply reuse that earlier value.

If the values differ, the function returns `0`.

If they are equal, secondary handling proceeds in this order:

1. Invoke F-5710a8628cba with no explicit arguments.
2. Retain an 8-bit result from that invocation.
3. Read the object’s operation-table address.
4. Resolve the target at table offset `+0xA0`.
5. Invoke that target with the object address and the retained 8-bit result.
6. Return `1` if the indirect invocation returns normally.

The operation-table address for this path is read **after** F-5710a8628cba returns. It is not captured before that dependency runs.

### Comparison and arithmetic rules

All matching decisions use equality or inequality of 32-bit values. There are no ordered comparisons, range checks, increments, counters, or arithmetic transformations of matching values.

Signedness therefore does not change the stated matching decisions: equality depends on the 32-bit representation. The secondary enable field accepts every nonzero 32-bit representation, not only `1`.

Layout offsets are byte offsets. The supplied evidence does not establish checked address arithmetic, alignment handling, or behavior for addresses whose offset calculations are invalid.

## Side effects

### Direct effects

The function explicitly reads record memory, object memory, selected global values, and selected indirect-operation-table entries.

It contains no explicit writes to:

- The record.
- Object fields.
- Either matching global.
- The indirect-operation table.

It performs no explicit allocation, deallocation, ownership transfer, or reference-count operation.

### Dependency effects

Dependency side effects are unknown. A selected dependency may change object state, globals, other memory, or ownership relationships. This function provides no visible restoration or rollback.

In particular, F-5710a8628cba is not known to preserve the object, its operation-table address, or any global. Because table resolution follows this call, a valid change to the object’s table address may affect which indirect target is selected.

If a dependency releases or invalidates memory needed by a subsequent access, safe continuation is not established. The absence of an explicit release in this function is not evidence that dependencies preserve ownership.

### Read-order significance

The ordering above describes explicit accesses in the supplied static representation. It is not a concurrency or memory-ordering guarantee.

The function retains the record second field across both matching stages but may read the record first field twice. Consequently, it does not implement an atomic snapshot of the record. No synchronization, atomic access, or mutation-safety protocol is established.

### Exceptional behavior

No explicit error reporting, exception handling, or failure translation is present.

If a dependency does not return normally, the corresponding `1` return is not reached. Invalid memory or invalid indirect targets have no specified recoverable outcome in the supplied evidence.

## Dependencies

### F-5710a8628cba

Established behavior at this call site:

- Invoked only after all secondary matching conditions succeed.
- Receives no explicit arguments.
- Supplies a value retained as 8 bits.
- Runs before the secondary operation-table address and target are read.

Its purpose, full return convention, side effects, ownership behavior, and failure modes are **documented_only** unknowns.

### Unresolved indirect dependencies

Two indirect call sites are established:

- Operation-table offset `+0xD4`, supplied with the object address.
- Operation-table offset `+0xA0`, supplied with the object address and an 8-bit value.

Opaque function IDs for their actual targets are **documented_only** unknowns; none are invented here. Distinct table offsets do not prove distinct target functions.

The function does not use a return value from either indirect operation.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. No runtime tests have been performed. Each example assumes valid memory and normally returning dependencies wherever execution reaches them.

1. **Unsupported tag**

   Given a record tag other than `0x00000013`, the result is `0`. Only the tag is explicitly read; neither matching path is entered.

2. **Primary exact match**

   Given the accepted tag, matching primary second fields, and matching primary first fields, the target at table offset `+0xD4` is invoked once with the object. The primary alternate global is not read. The result is `1` after the invocation returns.

3. **Primary alternate match**

   Given the accepted tag and matching primary second fields, suppose the record first field differs from the object’s primary expected first field but equals the primary alternate global value. The primary target is invoked, then the function returns `1`.

4. **Primary second-field mismatch**

   Given the accepted tag but unequal primary second fields, the primary first-field comparison is skipped. Secondary eligibility is still evaluated.

5. **Primary precedence**

   If primary matching succeeds and the object’s secondary state would also permit a match, only the primary target is invoked. F-5710a8628cba and the secondary target are not invoked.

6. **Secondary disabled**

   If primary matching fails and the secondary enable field is zero, the result is `0`. The secondary expected second field and secondary required global are not read.

7. **Noncanonical enable value**

   If primary matching fails and the secondary enable field is `0x80000000`, secondary matching is enabled. Selection still depends on both secondary field comparisons.

8. **Secondary second-field mismatch**

   If secondary matching is enabled but its expected second field differs from the retained record second field, the result is `0`. The secondary first-field read, secondary global read, and dependency calls do not occur.

9. **Secondary success with zero byte**

   If all secondary conditions succeed and F-5710a8628cba supplies an 8-bit zero, that zero is forwarded to the secondary target. It is not treated as failure. The function returns `1` after the target returns.

10. **Post-helper target selection**

    If F-5710a8628cba changes the object’s operation-table address while leaving the object valid, the subsequent secondary target resolution uses the address read after that call.

## Uncertainties

The following remain **documented_only**:

- The semantic meanings of the tag, record fields, object fields, and global values.
- Concrete global values and initialization rules.
- Exact source-language types, endianness, alignment requirements, and ABI details.
- Full argument-extension rules for forwarding the 8-bit value; only its eight retained bits are established.
- Identities and contracts of indirect targets.
- Dependency side effects, reentrancy, memory ownership, and nonlocal control flow.
- Behavior under concurrent mutation or invalid memory.

The record-filter interpretation is inferred intent, not an additional behavioral requirement.

## Evidence

Established static evidence consists of the supplied representation’s branch conditions, 32-bit field accesses, byte offsets, conditional global reads, indirect table accesses, dependency invocation order, 8-bit intermediate result, and explicit returns of `0` or `1`.

No runtime execution, tests, dependency implementations, global definitions, type declarations, or ABI documentation were supplied. Acceptance examples are conditional specifications, not observed results.
