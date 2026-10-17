<!-- Trusted provenance: F-b3601dc1fbbf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b3601dc1fbbf

## Purpose

F-b3601dc1fbbf initializes or replaces a contiguous pair of four-byte storage units using two caller-supplied values.

The established behavior is limited to two assignments: the first supplied value is stored in the first unit, followed by the second supplied value in the immediately adjacent unit. Neither assignment depends on the destination’s previous contents.

This is a raw storage operation. The supplied representation does not establish a higher-level meaning for the pair, such as coordinates, a range, a descriptor, or an ownership-bearing object. Such interpretations are not part of this contract.

## Interface

The function has three inputs and no explicit return value.

| Input | Role | Width or extent |
|---|---|---|
| Destination | Address of the first unit in a writable two-unit region | Pointer width is not established; required destination extent is eight bytes |
| First value | Value written to the first destination unit | Four bytes, or 32 bits |
| Second value | Value written to the second destination unit | Four bytes, or 32 bits |

The destination layout, expressed as byte offsets from its supplied address, is:

| Byte offsets | Contents after normal completion |
|---|---|
| 0–3 | First value in the target’s native representation |
| 4–7 | Second value in the target’s native representation |

The four-byte input types do not establish signedness, floating-point interpretation, pointer interpretation, or an application-specific encoding. The contract therefore treats each supplied value as a 32-bit payload. No numeric conversion or semantic validation is shown.

### Preconditions

For normal completion, the caller must supply a destination that permits both four-byte writes. This requires:

- A valid writable region covering the complete eight-byte extent.
- Alignment acceptable for the target’s four-byte storage operations.
- An address at which locating the second unit is valid.
- Any synchronization required when another execution context can access the same storage.

The supplied representation contains no checks enforcing these requirements. It also establishes no guarantee for null, dangling, read-only, insufficiently sized, or improperly aligned destinations.

No initialized prior contents are required. Both destination units are assigned without first reading their stored values.

## Behavior

### Established operation

The function performs two destination assignments in the order shown by the supplied static representation:

1. Assign the first 32-bit value to the four-byte unit beginning at the destination.
2. Assign the second 32-bit value to the next four-byte unit, beginning four bytes after the destination.

The function then returns without an explicit result.

The second assignment uses the supplied destination address and second value. It does not obtain its address or value from the contents written by the first assignment. In particular, storing the first value does not cause that value to be interpreted as a replacement destination pointer.

Both assignments are unconditional in the supplied representation. There are no branches, loops, retries, comparisons, or conditional early returns.

### Prior contents and repeated calls

The destination’s prior contents do not contribute to either resulting value. On normal completion, both four-byte units have been replaced.

Calling the function again with the same destination replaces the same two units again. There is no shown accumulation, merge, deduplication, or preservation of part of either prior value.

Equal input values are accepted without a special case: both units receive that value. Zero values likewise produce ordinary assignments; zero has no demonstrated sentinel meaning.

### Widths and address arithmetic

The only represented address displacement is one four-byte unit, making the second write begin at byte offset four. The two write extents are adjacent and do not overlap when the destination satisfies the preconditions.

There is no arithmetic on either 32-bit payload. Signed overflow, saturation, carry handling, and numeric range checks therefore do not arise from the shown value operations.

The destination’s address width is unknown. The function does not show an address-wrap check or a bounds check. A destination for which the second unit cannot be validly addressed is outside the established normal-completion contract.

### Ordering limits

The static representation places the first assignment before the second. That ordering is the supported description of the function’s operations.

It does not independently establish the ordering observed by another processor, a concurrent thread, a device, or an asynchronous observer. There are no shown synchronization operations or explicit device-access qualifiers. Neither a jointly atomic eight-byte update nor atomicity of each four-byte write is established.

Consequently, this function must not be treated as a publication barrier or transactional replacement of the pair.

## Side effects

### Destination mutation

The two assignments account for an eight-byte destination extent. No additional memory writes or destination-content reads are shown.

For ordinary valid memory, the established postcondition concerns the two destination units. If the supplied address instead refers to special memory, such as device registers, the consequences of writing that memory are not established by this function’s representation.

### Ownership and lifetime

The function does not visibly allocate or release storage. It does not visibly retain the destination address in separate persistent state.

No ownership transfer is established. The caller must provide storage whose lifetime covers both writes and remains responsible for managing that storage under the surrounding program’s rules.

The 32-bit values are copied as values. Even if a caller interprets one as an encoded address or resource identifier, the function does not visibly dereference it, acquire the referenced resource, or release anything previously represented in the destination.

### Other state

No global-variable accesses, external calls, I/O operations, or synchronization operations are present in the supplied representation.

This is a bounded static observation, not a claim about every environmental effect of executing memory stores. Invalid memory, instrumented environments, and special address mappings can have consequences not specified here.

### Errors and partial effects

There is no explicit status result, error code, exception construction, or recovery path.

Behavior after an invalid write is not established. The representation provides no rollback mechanism: if the first write takes effect and the second cannot complete, restoration of the first unit is not promised. The exact fault or termination behavior depends on the execution environment.

## Dependencies

No calls to other functions are shown. There are therefore no function dependencies requiring opaque dependency IDs for this behavior.

Execution relies on environmental facilities rather than visible callees:

- A calling convention capable of supplying the destination and two four-byte values.
- A memory model supporting the represented stores.
- A target representation for four-byte values.
- Valid storage supplied by the caller.

The destination pointer width, byte order, precise alignment requirement, and synchronization rules are not supplied. No assumptions about unknown callees are needed because none appear in the representation.

## Acceptance examples

These examples are conditional contract examples, not reports of executed tests.

### Two distinct values

Given a valid writable eight-byte destination, a first value with bit pattern `0x11223344`, and a second value with bit pattern `0xAABBCCDD`, normal completion must leave:

- The first four-byte unit containing `0x11223344`.
- The second four-byte unit containing `0xAABBCCDD`.

The expected byte sequence depends on target byte order. This example does not select little-endian or big-endian storage.

### Replacing existing contents

Given valid destination units initially containing arbitrary values, supplying `0x00000000` and `0xFFFFFFFF` must replace the first unit with all zero bits and the second with all one bits.

Neither prior value is retained or combined with the supplied values.

### Equal values

Given a valid destination and two inputs both containing `0x76543210`, both destination units must contain that pattern after normal completion. Equality does not suppress either represented assignment.

### Destination within a larger object

Given a suitably aligned writable subregion inside a larger allocation, the two assignments target the eight bytes beginning at the supplied destination. No write to surrounding ordinary storage is shown.

Acceptance requires that this subregion be valid for the represented accesses; being inside a larger allocation alone does not establish all language-level access requirements.

### Repeated replacement

Given two successful calls on the same valid destination, the destination after the second call must contain the second call’s pair of values, irrespective of the pair supplied by the first call.

### Invalid destination

A null destination, a destination with only four writable bytes available, or an address unsuitable for the target’s four-byte accesses does not have a specified successful outcome.

Acceptance must not require graceful rejection, a particular error message, or preservation of the original contents in these cases. None is established.

## Uncertainties

The following are **documented_only**, not inferred guarantees:

- **Target details:** Pointer width, byte order, calling convention, and exact alignment constraints.
- **Machine-level stores:** Whether each represented assignment becomes one machine instruction or multiple accesses.
- **Concurrency:** Atomicity, inter-thread visibility, and ordering observable outside this execution context.
- **Fault handling:** The precise response to inaccessible, invalid, or specially mapped memory.
- **Object-model restrictions:** Any additional effective-type, aliasing, or storage-lifetime requirements imposed by the original language and runtime.
- **Application meaning:** What the two values represent and whether surrounding code imposes additional invariants.

An initialization or pair-replacement role is a reasonable inferred intent because the function unconditionally overwrites two adjacent units. The established behavior does not depend on that interpretation.

## Evidence

The supplied static representation establishes a three-input, no-explicit-result function containing two unconditional four-byte assignments and a return. The destination positions are the initial unit and its immediate successor. No other reads, writes, calls, checks, or branches are shown.

The layout, overwrite behavior, and absence of explicit error handling follow from that evidence. Runtime fault behavior, external visibility, and higher-level semantics do not.

No runtime tests have been performed. The acceptance examples specify conditional expected behavior; they are not execution evidence.
