<!-- Trusted provenance: F-dfa866bc2daa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dfa866bc2daa

## Purpose

F-dfa866bc2daa selects one of two embedded regions in a caller-supplied object and, under specific conditions, initializes a region using metadata obtained through external dependencies.

Two selector values receive special handling: `0x63C8` and `0x63C9`. Other selector values participate only in the initial lookup path. The function returns either zero or the address of an embedded region; it does not directly return the status produced by the dependencies that populate or initialize that region.

The interpretation of the regions as cached representations, and of the calculated differences as dimensions, is inferred intent. The established behavior is the lookup, conditional dependency calls, construction of an initialization record, and selection of a return address described below.

## Interface

### Inputs

The interface has five arguments:

| Argument | Established role |
|---|---|
| Object address | A 32-bit address used as the base for fixed-offset field accesses and embedded-region addresses. |
| Selector | A 32-bit integer compared with `0x63C8`, `0x63C9`, and a value read from shared storage. |
| First forwarded value | An opaque 32-bit value forwarded unchanged to F-6f3eae7c0190 when the initialization path is taken. |
| Second forwarded value | Another opaque 32-bit value forwarded unchanged to F-6f3eae7c0190 on that path. |
| Final forwarded value | An opaque 32-bit value forwarded unchanged to F-d2832466b7bd if its prerequisite call returns nonzero. |

The following object-relative locations are relevant:

| Offset | Use |
|---|---|
| `0x30` | A 32-bit field read before invoking F-6f3eae7c0190. |
| `0x540` | Address passed to F-bb3cb5b257ba for selector `0x63C8`. |
| `0x544` | Address passed to F-bb3cb5b257ba for every other selector. |
| `0x54C` | First embedded destination region. |
| `0x604` | Second embedded destination region. |

The extent and internal layout of the two destination regions are unknown. The function supplies their addresses to dependencies; the provided evidence does not establish how many bytes those dependencies read or write.

### Output

The return value is a 32-bit integer used as either:

- zero;
- the object address plus `0x54C`; or
- the object address plus `0x604`.

There is no separate error result. A nonzero return does not establish that a dependency succeeded, because relevant dependency results are ignored or converted into zero-valued initialization inputs.

The returned nonzero value is an interior address within the supplied object, not a directly observed allocation result. No ownership transfer or lifetime extension is established.

### Preconditions

The object and any memory reachable by called dependencies must be valid for the accesses actually performed. In particular:

- The address computations must be valid in the target’s 32-bit address model.
- The slot addresses passed to F-bb3cb5b257ba must satisfy that dependency’s requirements.
- On the initialization path, the 32-bit field at offset `0x30` must be readable.
- Destination regions must meet the access requirements of F-3f4fef4002d7 and F-4225f441f48b when those calls occur.

There are no explicit null checks, bounds checks, selector-range checks, or pointer-overflow checks. Behavior for invalid addresses or invalid dependency inputs is not established as a recoverable error.

## Behavior

### Initial lookup and candidate return address

The selector determines the initial slot and destination:

| Selector | Lookup slot offset | Destination offset |
|---|---:|---:|
| `0x63C8` | `0x540` | `0x54C` |
| Any other value | `0x544` | `0x604` |

The candidate return value initially equals zero.

F-bb3cb5b257ba is called with the selected slot address. If its result is zero, the initial stage ends without calling F-3f4fef4002d7 and without selecting a nonzero candidate.

If the result is nonzero:

1. F-bb3cb5b257ba is called again with the same slot address.
2. The result of this second call is passed as the first argument to F-3f4fef4002d7.
3. The selected destination address is passed as that dependency’s second argument.
4. After the call, the candidate return value becomes the selected destination address.

The second lookup is not replaced by the first lookup’s result. No check requires the second result to be nonzero. Thus, a nonzero first result followed by a zero second result still causes F-3f4fef4002d7 to be called with zero as its first argument.

The result returned by F-3f4fef4002d7 does not determine the final return value or whether the later initialization path is taken.

### Additional lookup for the two special selectors

Only selectors `0x63C8` and `0x63C9` receive an additional lookup:

- `0x63C8` uses the slot at offset `0x540`.
- `0x63C9` uses the slot at offset `0x544`.

F-bb3cb5b257ba is called again with that slot address, regardless of whether the initial lookup was zero or nonzero.

After this additional lookup returns, the function reads a 32-bit value from shared storage. This value is the **captured selector discriminator**. The read occurs even when the additional lookup result is nonzero.

If the additional lookup result is nonzero, no initialization follows. The function returns the candidate established by the initial stage.

If the additional lookup result is zero, the initialization path follows. This decision uses the additional lookup result, not either earlier lookup result. Consequently:

- An initially available value can be followed by initialization if the additional lookup returns zero.
- An initially absent value can lead to a zero return without initialization if the additional lookup returns nonzero.

For selectors other than `0x63C8` and `0x63C9`, this additional lookup and discriminator read do not occur. The function returns the initial candidate.

### Metadata acquisition on the initialization path

Two calculated differences begin at zero.

The function chooses a request value by comparing the original selector with the captured selector discriminator:

- Equal: request value `8`.
- Unequal: request value `9`.

F-cb9fa74c0a71 receives that request value and returns a 64-bit value. Its halves have distinct uses:

- The upper 32 bits are passed to F-6f3eae7c0190.
- The lower 32 bits are reserved for a possible subsequent call to F-d2832466b7bd.

After F-cb9fa74c0a71 returns, the function reads the 32-bit field at object offset `0x30`. It then calls F-6f3eae7c0190 with these four arguments, in order:

1. The value just read from object offset `0x30`.
2. The upper 32 bits of the 64-bit result.
3. The first forwarded input value.
4. The second forwarded input value.

If F-6f3eae7c0190 returns zero, both calculated differences remain zero, and F-d2832466b7bd and F-53994e63b9d8 are skipped.

If it returns nonzero, F-d2832466b7bd receives:

1. That nonzero result.
2. The lower 32 bits of the 64-bit result.
3. The final forwarded input value.

If F-d2832466b7bd returns zero, the calculated differences remain zero, and F-53994e63b9d8 is skipped.

If both calls return nonzero, two consecutive 32-bit temporary words are initialized to zero. F-53994e63b9d8 receives the second call’s result and a pointer to the first temporary word.

After that dependency returns, the function derives two differences:

- **Low-half difference:** the signed 16-bit low half of the second word minus the signed 16-bit low half of the first word.
- **High-half difference:** the unsigned 16-bit high half of the second word minus the unsigned 16-bit high half of the first word.

Each operand is widened before subtraction. Both resulting ranges fit within a signed 32-bit integer: `−65535` through `65535`. There is no 16-bit wraparound in these differences and no clamp to zero.

The asymmetric signedness is significant. A low half containing `0xFFFF` contributes `−1`; a high half containing `0xFFFF` contributes `65535`.

No result from F-53994e63b9d8 is tested. The contents of the temporary words after the call determine the differences, including any portions left unchanged from their initial zero values.

### Initialization record and destination

The function constructs a temporary record with the following visible layout:

| Byte offset | Width | Content |
|---|---:|---|
| `0x00` | 16 bits | Low 16 bits of a value read from a second shared storage location. |
| `0x02` | 16 bits | No explicit initialization is visible. |
| `0x04` | 16 bits | Zero. |
| `0x06` | 16 bits | `0x0010`. |
| `0x08` | 32 bits | Zero. |
| `0x0C` | 32 bits | Zero. |
| `0x10` | 32 bits | Low-half difference. |
| `0x14` | 32 bits | High-half difference. |
| `0x18` | 32 bits | Zero. |

The shared value supplying the first field is read after the metadata acquisition calls. It is not established to have remained unchanged during those calls.

The two bytes at offset `0x02` must not be specified as zero, padding ignored by the callee, or a meaningful field with a known value. Their interpretation and whether the dependency reads them are unknown.

Destination selection uses the previously captured selector discriminator, not a fresh read:

- Equality selects object offset `0x54C`.
- Inequality selects object offset `0x604`.

Thus the discriminator controls both the earlier choice between request values `8` and `9` and this final destination choice.

F-4225f441f48b is called with:

1. The selected destination address.
2. A pointer to the temporary record.
3. Zero.
4. Zero.

Its return value is not used. After the call returns, the function returns the selected destination address, even if preceding metadata calls returned zero.

## Side effects

No direct write into the caller’s object is visible in the function body. Object or global mutations may nevertheless occur through dependencies.

Observable interactions include:

- Repeated calls to F-bb3cb5b257ba, whose results are not assumed stable.
- A possible F-3f4fef4002d7 call before the additional special-selector lookup.
- A discriminator read after that additional lookup.
- A read of object offset `0x30` after F-cb9fa74c0a71.
- A later shared-value read used to populate the initialization record.
- Calls receiving addresses of embedded regions and temporary storage.

No dependency may be assumed to preserve object fields, shared storage, or other global state. In particular, the discriminator is captured once, so subsequent mutation of its shared source does not change the request or destination decisions already based on that captured value.

The temporary words and initialization record are local storage. Their addresses are supplied to dependencies, but the evidence does not establish whether those dependencies retain the addresses. Any required synchronous-consumption or non-retention rule is an unresolved dependency contract.

No explicit allocation or release operation is visible. Whether intermediate nonzero results denote owned resources, borrowed handles, or ordinary values is unknown. The absence of a visible release call does not establish ownership or a leak.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-bb3cb5b257ba | Accepts a slot address and supplies a value used for presence decisions or as an argument to another dependency. May be called repeatedly for the same slot. |
| F-3f4fef4002d7 | Receives the second initial lookup result and an embedded destination address. Its result does not control subsequent behavior. |
| F-cb9fa74c0a71 | Receives request `8` or `9`; supplies a 64-bit result split into upper and lower 32-bit arguments. |
| F-6f3eae7c0190 | Receives the object field, upper result half, and two forwarded values. Zero prevents the next two metadata calls. |
| F-d2832466b7bd | Receives the preceding nonzero result, lower result half, and final forwarded value. Zero prevents the temporary-word output call. |
| F-53994e63b9d8 | Receives a nonzero intermediate result and the address of two zero-initialized 32-bit words subsequently interpreted as halfword pairs. |
| F-4225f441f48b | Receives the final destination, temporary record, and two zero arguments. Its result is ignored. |

These roles describe caller-visible usage, not independent guarantees about the dependencies’ implementations.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Each assumes all indicated calls return normally and all required memory accesses are valid.

1. **Ordinary selector with an absent initial value.**  
   For a selector other than `0x63C8` or `0x63C9`, if the first lookup returns zero, there is one lookup using offset `0x544`. Neither destination-processing dependency is called. The return value is zero.

2. **Ordinary selector with changing lookup results.**  
   If its first lookup is nonzero and its second lookup is zero, F-3f4fef4002d7 still receives zero and the address at offset `0x604`. The return value is that address, irrespective of the dependency’s return value.

3. **Special selector with no initialization.**  
   For `0x63C8`, if the first lookup is nonzero, the second supplies a value for F-3f4fef4002d7, and the additional lookup is nonzero, the function returns the address at offset `0x54C`. The discriminator is read, but the metadata acquisition path is skipped.

4. **Initially absent but subsequently present.**  
   For `0x63C9`, suppose the first lookup is zero and the additional lookup is nonzero. No F-3f4fef4002d7 call occurs, no initialization occurs, and the return value remains zero.

5. **Initialization after an earlier populated result.**  
   For either special selector, a nonzero initial lookup does not prevent initialization. If the additional lookup is zero, the initialization path runs after the earlier F-3f4fef4002d7 call.

6. **Zero intermediate metadata result.**  
   If the initialization path is entered and F-6f3eae7c0190 returns zero, F-d2832466b7bd and F-53994e63b9d8 are skipped. F-4225f441f48b still receives a record with both difference fields zero, and the selected destination is returned.

7. **Signedness boundary.**  
   If the output words are `0xFFFF8000` followed by `0x00007FFF`, the low-half difference is `65535`, while the high-half difference is `−65535`. These full 32-bit results populate record offsets `0x10` and `0x14`.

8. **Discriminator overriding the initial destination choice.**  
   If selector `0x63C8` enters initialization but differs from the captured discriminator, request `9` is used and the final destination is offset `0x604`, despite the initial stage having selected offset `0x54C`.

## Uncertainties

The following remain **documented_only**:

- The actual values and broader meanings of the two shared storage locations.
- The semantic names, sizes, and ownership rules of the object slots and destination regions.
- The extent and format of memory accessed by dependencies.
- Whether any dependency mutates shared state, retains passed pointers, allocates resources, or releases them.
- The meaning and consumption of the uninitialized two-byte portion of the record.
- Target byte order and any additional ABI constraints beyond the visible 32-bit values and address arithmetic.
- Behavior on invalid addresses, arithmetic outside valid address ranges, abnormal dependency termination, or concurrent mutation.

The coordinate-like output interpretation and cache-like role of the slots are inferred intent, not required semantics.

## Evidence

This contract is based solely on static inspection of the supplied function representation and opaque dependency mapping. Established evidence includes the branch conditions, repeated dependency calls, fixed object offsets, argument order, shared-read placement, record assignments, halfword signedness, and return-value selection.

Dependency internals were not supplied. Their names and apparent roles do not establish additional behavior. No runtime tests, execution traces, or memory-layout validation have been performed.
