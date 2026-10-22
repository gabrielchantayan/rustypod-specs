<!-- Trusted provenance: F-3f6a3abf1d81; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3f6a3abf1d81

## Purpose

F-3f6a3abf1d81 selects one of three 32-bit result values by examining up to two bytes in an indirectly referenced record. It invokes F-0b4658e20c99 before making the selection and invokes F-96dd78c4ca33 afterward, passing the original input to the latter.

The selection has strict precedence:

- A zero first byte selects 0, without inspecting the second byte.
- A nonzero first byte and a zero second byte select 1.
- Two nonzero bytes select 2.

These are established control-flow properties. The meanings of the two bytes and the three results are unknown. Describing them as flags or classification states may be useful, but assigning application-specific meanings would be inferred intent rather than established behavior.

## Interface

### Input

The function accepts one argument represented as a 32-bit integer in the supplied static representation. The function uses that value as the address of an outer object and also passes it unchanged to F-96dd78c4ca33.

The accessed layout is:

| Location | Access width | Role |
|---|---:|---|
| Outer-object address plus 4 bytes | 32 bits | Address of an indirectly referenced record |
| Referenced-record address plus 22 bytes | 8 bits | First selection byte |
| Referenced-record address plus 23 bytes | 8 bits | Second selection byte, accessed conditionally |

The input is not itself the address of either selection byte. One level of address indirection through the field at offset 4 is required.

The static representation uses a 32-bit integer load for the indirect address. It does not establish a portable host-language pointer interface, a calling convention, or compatibility with wider addresses. A reimplementation must preserve the represented address model rather than silently substitute a different record layout.

### Output

On normal completion, the function returns a 32-bit value equal to 0, 1, or 2.

The output is a selected value, not a copy of a dependency’s return value. Neither dependency’s return value is used in the visible selection or return behavior.

### Preconditions

For a normally completing invocation:

- F-0b4658e20c99 must return control.
- After that call, the outer-object field at offset 4 must support a valid 32-bit read.
- The address obtained from that field must support a valid byte read at offset 22.
- If that first byte is nonzero, the outer-object field must support another 32-bit read, and the address obtained by that second read must support a valid byte read at offset 23.
- F-96dd78c4ca33 must accept the original input and return control.

These are access requirements, not validation performed by this function. The visible behavior contains no null checks, bounds checks, or error-result conversion.

## Behavior

### Entry dependency

F-0b4658e20c99 is invoked first, with no explicit arguments in the supplied representation.

No outer-object or selection-byte read appears before this call. Consequently, selection observes memory after this dependency returns, not necessarily memory as it existed at function entry.

The dependency may affect the data subsequently inspected. Its effects on globals, shared memory, object lifetime, or other state are not established here.

### First observation

After the entry dependency returns, the function reads the 32-bit address field at outer-object offset 4. It then reads the byte at offset 22 from the resulting record address.

Only equality with zero matters:

- If the byte is zero, the selected result becomes 0.
- If the byte is nonzero, the function proceeds to the second observation.

All nonzero byte patterns have the same branch effect. Signedness of a character interpretation does not distinguish them for this comparison.

### Conditional second observation

The second observation occurs only when the first byte was nonzero.

The function reads the outer-object address field again, then reads the byte at offset 23 from the record address obtained by that second read. The static representation does not retain the first record address for this access.

The second byte selects between the remaining results:

- Zero selects 1.
- Nonzero selects 2.

The distinction between the two address-field reads is observable in memory models where repeated reads may yield different values. The contract therefore does not require the two bytes to come from the same record unless an external stability guarantee supplies that condition.

There is no visible retry, consistency check, or attempt to obtain an atomic snapshot of the two bytes.

### Exit dependency and return

After selecting a result, the function invokes F-96dd78c4ca33 with the original input value.

This invocation occurs on all three normal selection paths. It is not conditional on a particular result and is not skipped when the first byte is zero.

The result is selected before this call. There is no visible rereading of the selection bytes afterward. If F-96dd78c4ca33 changes those bytes and then returns normally, the function still returns the previously selected result under ordinary call semantics.

The visible body does not use F-96dd78c4ca33’s return value to revise the selection. If that dependency does not return normally, a normal return from F-3f6a3abf1d81 is not established.

## Side effects

The function performs no explicit writes through the input address or the referenced-record addresses.

Its directly visible externally relevant actions, in order, are:

1. Invoke F-0b4658e20c99.
2. Read the outer-object address field.
3. Read the first selection byte.
4. Only if that byte is nonzero, reread the address field and read the second selection byte.
5. Invoke F-96dd78c4ca33 with the original input.
6. Return the selected 32-bit value.

This list describes the visible body, not every action taken by its dependencies.

No allocation, deallocation, ownership transfer, reference-count change, lock operation, or resource release is explicit in this body. None should be attributed to either dependency without additional evidence. In particular, the position of F-96dd78c4ca33 at the end does not establish that it performs cleanup.

The function does not visibly retain the input or record addresses after returning. A dependency could retain them; that possibility is outside the established body-level behavior.

## Dependencies

| Opaque ID | Invocation | Established relationship |
|---|---|---|
| F-0b4658e20c99 | Once before all selection reads, if execution reaches and completes the call | No explicit arguments; return value unused |
| F-96dd78c4ca33 | Once after a result is selected, on each normally progressing selection path | Receives the original input; return value unused |

Neither dependency’s implementation is available in the supplied evidence.

The contract does not assume that either dependency preserves globals, leaves object contents unchanged, owns memory, releases memory, succeeds, or returns normally. There is also no established relationship between their purposes beyond their invocation order.

## Acceptance examples

These examples are conditional consequences of the static behavior, not runtime observations. Unless stated otherwise, both dependencies return normally and all required reads are valid.

| First observed byte | Second observed byte | Expected result | Required selection-byte reads |
|---|---|---:|---|
| Zero | Any value or unreadable | 0 | First byte only |
| Nonzero | Zero | 1 | First byte, then second byte |
| Nonzero | Nonzero | 2 | First byte, then second byte |

Additional acceptance conditions:

- **Unneeded second-byte access:** If the first byte is zero, an inaccessible second-byte location does not itself prevent normal completion through the visible selection path. The dependencies may impose separate access requirements.
- **Noncanonical nonzero values:** A first byte with all bits set is treated like any other nonzero first byte. A nonzero second byte likewise selects 2 regardless of which bits are set.
- **Entry-time state change:** If F-0b4658e20c99 changes the first byte from nonzero to zero before returning, the subsequent valid read selects 0. Entry-time contents are not a saved snapshot.
- **Changed indirect address:** If a supported execution environment causes the first address-field read to yield one record and the second to yield another, selection uses the first record’s byte at offset 22 and the second record’s byte at offset 23. This example does not assert that unsynchronized concurrent modification is supported.
- **Postselection state change:** If selection produces 1 and F-96dd78c4ca33 then changes both bytes to zero, normal completion still returns 1. There is no postcall recomputation.
- **Invalid required access:** An invalid first-byte address has no specified result of 0, 1, or 2. The function supplies no visible recovery path for such an access.

## Uncertainties

The following are **documented_only** limitations:

- The application-level interpretation of the bytes and results is unknown.
- Record sizes, allocation provenance, alignment guarantees, byte order for the 32-bit address field, and target calling convention are not established.
- Address arithmetic validity is a precondition. Although the represented address field is 32 bits, the evidence does not establish a portable wraparound rule for addresses near the address-space boundary.
- The two one-byte accesses do not require a multi-byte integer interpretation. Byte signedness therefore does not change the established zero/nonzero outcomes.
- Synchronization, volatility, atomicity, and permitted concurrent mutation are unspecified. Repeated reads are visible in the supplied representation, but that alone does not establish a supported concurrent-access protocol.
- Dependency failures, exceptions, nonlocal exits, memory faults, and their propagation mechanisms are unspecified.
- The dependencies’ memory ownership and state effects are unknown.

The inferred intent is a three-way classification bracketed by two dependency calls. No stronger lifecycle or resource-management interpretation is justified.

## Evidence

The supplied static representation establishes the dependency order, the address field at offset 4, the byte offsets 22 and 23, the conditional second address-field read, the zero/nonzero branch precedence, the selected values, and the final dependency invocation before return.

No dependency bodies, runtime traces, independent layout declarations, or execution results were supplied. No runtime tests have been performed. The acceptance examples express the visible contract under their stated preconditions; they do not claim execution-based verification.
