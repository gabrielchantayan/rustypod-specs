<!-- Trusted provenance: F-0abf9b5ffbbf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0abf9b5ffbbf

## Purpose

F-0abf9b5ffbbf conditionally handles an event for a receiver.

Two event kinds have handling paths:

- **15:** accepted only when an event-carried address equals the address of an embedded receiver location. The accepted path conditionally clears one byte of receiver state, invokes a receiver callback, and then invokes **F-42382d1a1b3b** with a newly evaluated state value.
- **19:** obtains an auxiliary address through an event callback, compares an auxiliary field with a receiver field, and conditionally invokes another receiver callback. Both comparison outcomes return a handled result.

Unsupported kinds and kind 15 events with nonmatching addresses return an unhandled result.

“Event,” “receiver,” “state,” and “auxiliary object” describe their structural roles. Their application-level meanings are not established.

This contract distinguishes source-visible behavior from representation-dependent interpretation. In particular, conversion of a four-byte integer to plain character type must not be treated unconditionally as extracting its low byte.

## Interface

### Inputs and representation

The function takes two pointer-valued inputs:

1. **Receiver:** used for address comparison, state access, and indirect dispatch.
2. **Event:** used to select a handling path and provide event-specific data.

The supplied representation uses four-byte integer elements and a four-byte return value. Pointer-bearing integer fields and dispatch operations are consistent with a 32-bit address model. The exact ABI and applicability to another address width are not established.

Relevant storage locations are:

| Object | Byte offset | Use |
|---|---:|---|
| Receiver | `0x000` | Indirect-call-table address |
| Receiver | `0x044` | Four-byte comparison field for kind 19 |
| Receiver | `0x468` | Embedded location whose address is compared for kind 15 |
| Receiver | `0x494` | Four-byte integer converted to plain character type; first addressed byte can be cleared |
| Event | `0x000` | Indirect-call-table address for kind 19 |
| Event | `0x004` | Four-byte event kind |
| Event | `0x008` | Address-valued field for kind 15 |
| Auxiliary object | `0x00c` | Four-byte comparison field |

Indirect dispatch uses these byte offsets within the respective tables:

- Receiver table: `0x124` on the accepted kind 15 path.
- Event table: `0x014` on the kind 19 path.
- Receiver table: `0x0d4` when the kind 19 comparison fields are equal.

These offsets identify dispatch entries, not code offsets within the receiver or event.

### Output

Normal completion returns a four-byte value:

| Condition | Result |
|---|---:|
| Unsupported event kind | 0 |
| Kind 15, nonmatching address | 0 |
| Kind 15, matching address | 1 |
| Kind 19, either comparison outcome | 1 |

The result indicates path handling, not dependency success. Receiver-callback results and the result of F-42382d1a1b3b are not used to select the returned value.

A memory fault, conversion signal, nonlocal exit, or dependency that does not return normally has no established ordinary return result.

### Preconditions

The event must support the initial four-byte read at offset `0x004`. A null event is therefore not a supported input, despite a later explicit null comparison.

Further requirements are path-dependent:

- An unsupported kind needs no visible receiver dereference.
- Kind 15 needs a readable event field at `0x008` and a valid basis for computing the receiver’s embedded address. A matching address additionally requires readable receiver state, writable storage if the conditional byte write occurs, and valid dependencies.
- Kind 19 needs valid event dispatch storage, a callable event dependency, a usable returned auxiliary address, and a readable receiver comparison field. Equality additionally requires valid receiver dispatch storage.

No visible check establishes object extent, alignment, lifetime, table validity, or permission to write. Address computations are not validation operations. No wrapping-address behavior or out-of-bounds pointer semantics are established.

## Behavior

### Dispatch

The function first reads the event kind at offset `0x004` and compares it with 15.

If that comparison does not select kind 15, the supplied body evaluates the kind field again to determine whether it is 19. For a stable event field, this implements the two-kind dispatch described above. The source does not establish an atomic snapshot of the kind field or a concurrency guarantee.

If neither branch is selected, the function returns 0 without a visible receiver dereference or dependency call.

### Kind 15: address selection

The event field at offset `0x008` is interpreted as an address and compared with the address **receiver + `0x468`**.

This is an address comparison. The contents of the embedded location are not read.

If the addresses differ, the function returns 0. It does not inspect receiver state at `0x494`, invoke either receiver callback, or invoke F-42382d1a1b3b.

### Kind 15: accepted path

A matching address selects this ordered sequence:

1. Read the four-byte receiver integer at `0x494` and convert it to plain character type.
2. If the converted value equals exactly 1, write zero to the single byte beginning at receiver offset `0x494`.
3. Invoke the receiver dependency selected through table offset `0x124`, passing the receiver.
4. After that dependency returns, read the four-byte receiver integer at `0x494` again and convert it to plain character type again.
5. Invoke F-42382d1a1b3b with the receiver and the second converted value.
6. Return 1 if execution reaches normal completion.

The comparison is equality with 1, not a general nonzero test. A successfully converted value of 0, 2, or any other value does not trigger the direct write.

The direct write is one byte wide. It is not a four-byte assignment of zero.

Both dependencies are invoked regardless of whether the conditional write occurs, provided execution continues normally. The second conversion uses a fresh state evaluation after the receiver callback; it is not a saved pre-callback value.

The receiver callback can therefore affect the value passed to F-42382d1a1b3b. No preservation of state or object lifetime is assumed. Safe continuation requires the receiver storage to remain usable after the callback.

### Character conversion and byte order

The source establishes two distinct operations:

- Conversion of a four-byte integer value to plain character type.
- A zero write to the first addressed byte of that integer’s storage.

These operations must not be conflated.

If the integer value is representable in the character type, conversion preserves that value. In particular, integer values 1 and 2 convert to 1 and 2 respectively.

For an out-of-range value, the supplied material does not establish the conversion result. With signed plain character type, the conversion can have an implementation-defined result or raise an implementation-defined signal. Little-endian storage and eight-bit characters alone do not establish low-eight-bit retention.

Consequently, treating the conversion as low-byte extraction requires an additional implementation assumption: **integer-to-character conversion retains the low eight bits and completes normally for the value in question**.

Separately, on a little-endian target with eight-bit bytes, the first addressed byte is the integer’s least significant byte. That storage fact does not establish how an out-of-range integer converts to character type.

These qualifications apply to both conversions:

- A signal or other non-normal outcome from the first conversion can prevent the write and both subsequent calls.
- A non-normal outcome from the second conversion can occur after the receiver callback and prevent the call to F-42382d1a1b3b.

### Kind 19: auxiliary lookup

The kind 19 path initially establishes a zero auxiliary-address value. It then checks whether the event is non-null.

For a non-null event, it invokes the dependency selected through event table offset `0x014`, passing:

- The event.
- The integer selector `0x0c00`, equal to 3072.

The returned value replaces the auxiliary-address value.

The null check does not protect the earlier event-kind access. It must not be interpreted as making a null event valid.

### Kind 19: comparison and callback

After auxiliary lookup, the function compares:

- The four-byte integer at auxiliary address + `0x00c`.
- The four-byte receiver integer at `0x044`.

The event dependency completes before these comparison accesses. Its mutations may therefore affect either field or the storage used to reach them.

The supplied expression does not establish a portable relative evaluation order between the two comparison operands. This contract does not require the auxiliary field to be read before the receiver field, or vice versa.

If the fields differ, the function returns 1 without invoking the receiver dependency at table offset `0x0d4`.

If the fields are equal, that receiver dependency is invoked with the receiver. The function returns 1 after it returns normally.

The comparison uses the full four-byte integer values. No character conversion, masking, range test, or ordered comparison is visible.

The auxiliary address is not checked before field access. A zero callback result is used as the base for an attempted access at offset `0x00c`; it does not produce a defined unhandled result or a visible error code.

## Side effects

### Direct effects

The only explicit object-state write is a one-byte zero write at receiver offset `0x494`. It occurs only on the accepted kind 15 path when the first character conversion completes and equals 1.

There is no visible direct write to:

- The event.
- The auxiliary object.
- The receiver comparison field at `0x044`.
- The embedded location at `0x468`.
- The remaining bytes of the receiver state field.

### Dependency effects and ordering

| Path | Dependency calls in order |
|---|---|
| Unsupported kind | None |
| Kind 15, address mismatch | None |
| Kind 15, address match | Receiver slot `0x124`, then F-42382d1a1b3b |
| Kind 19, unequal fields | Event slot `0x014` |
| Kind 19, equal fields | Event slot `0x014`, then receiver slot `0x0d4` |

These sequences assume all preceding operations complete normally.

No explicit allocation, release, ownership transfer, synchronization, rollback, or recovery mechanism is visible. This does not constrain the dependencies: they may mutate globals, change objects, allocate or release storage, reenter other behavior, or fail to return normally.

There is no visible lifetime protection around calls. In particular:

- Kind 15 reads receiver state after the first receiver callback.
- Kind 19 reads auxiliary and receiver storage after the event callback.

Passing a pointer does not establish whether the dependency borrows, retains, transfers, or releases it.

## Dependencies

The following opaque labels identify unresolved indirect dependencies within this contract only. They do not assert resolved implementation identities.

| Opaque function ID | Selection and visible interface | Unknown contract |
|---|---|---|
| **F-indirect-01** | Receiver table offset `0x124`; receives the receiver on accepted kind 15 events | Concrete identity, return type, mutations, ownership, global effects, failure behavior |
| **F-indirect-02** | Event table offset `0x014`; receives the event and 3072; result supplies the auxiliary address | Concrete identity, selector meaning, result validity, lifetime, ownership, mutations |
| **F-indirect-03** | Receiver table offset `0x0d4`; receives the receiver when kind 19 fields are equal | Concrete identity, return type, mutations, ownership, global effects, failure behavior |
| **F-42382d1a1b3b** | Receives the receiver and the state value converted after F-indirect-01 returns | Semantic purpose, argument ABI details, mutations, ownership, global effects, failure behavior |

Indirect targets are selected from the applicable object’s dispatch table when the call is made. No table or field preservation across earlier calls is assumed.

## Acceptance examples

These are conditional expectations derived from static evidence, not runtime test results. Each normal-result example assumes valid required storage, valid calls, and normal completion of conversions and dependencies.

1. **Unsupported kind.**  
   If the kind field remains 7 throughout dispatch, the result is 0. No dependency is called and no receiver state is directly accessed.

2. **Kind 15, wrong address.**  
   If the event carries an address different from receiver + `0x468`, the result is 0. State at `0x494` is neither read nor written by this path.

3. **Kind 15, integer state 1.**  
   With a matching address and initial four-byte state value 1, the first conversion yields 1. The first addressed byte is cleared before F-indirect-01 is invoked. F-42382d1a1b3b follows with a newly converted state value. Normal completion returns 1. The full integer value after the byte write depends on representation and subsequent callback effects.

4. **Kind 15, integer state 2.**  
   With a matching address and initial state value 2, the conditional write is skipped. F-indirect-01 and F-42382d1a1b3b still execute in that order.

5. **Callback changes the state.**  
   If F-indirect-01 leaves valid receiver storage containing integer value 2, F-42382d1a1b3b receives the converted value 2, not a cached pre-callback value.

6. **Conditional low-byte preservation example.**  
   Assume eight-bit bytes, little-endian integer storage, and an implementation whose integer-to-character conversion retains the low eight bits and completes normally for `0x12340101`. With that initial state, the first conversion yields 1. The direct byte write changes the stored integer to `0x12340100` before callback effects. Without the conversion assumption, neither reaching that write nor obtaining that intermediate value is established.

7. **Out-of-range signed-character conversion.**  
   If the initial state is outside the signed character type’s range, the source alone does not determine whether the first conversion yields 1, yields another value, or raises an implementation-defined signal. No unconditional callback sequence or normal result is specified for that case.

8. **Kind 19, unequal fields.**  
   If F-indirect-02 returns usable auxiliary storage and the compared fields differ after it returns, the result is 1. F-indirect-03 is not called.

9. **Kind 19, equal fields.**  
   If the fields are equal after F-indirect-02 returns, F-indirect-03 is invoked once. Its return value does not select the function’s normal result, which is 1.

10. **Zero auxiliary result.**  
    If F-indirect-02 returns zero, the next comparison attempts to use auxiliary storage at address offset `0x00c`. No safe error return is established.

## Uncertainties

The following remain **documented_only**:

- Exact ABI, confirmed pointer width, byte order, alignment, and character signedness.
- Out-of-range integer-to-character conversion results and possible implementation-defined signaling.
- Argument extension or promotion details for the character-derived value passed to F-42382d1a1b3b.
- Address arithmetic behavior outside valid object and address ranges.
- Complete object layouts and application meanings.
- Concrete identities and full contracts of indirect dependencies.
- Auxiliary-storage validity, ownership, and lifetime.
- Dependency effects on globals, dispatch tables, object lifetime, reentrancy, and synchronization.
- Surrounding runtime behavior for faults, signals, exceptions, or nonlocal exits.
- Concurrency semantics and whether event-kind reads observe a stable value.

An event-handling interpretation is inferred intent. The stated branches, field accesses, conditional byte write, dependency ordering, and explicit normal return values are established static observations, subject to the representation qualifications above.

## Evidence

This contract uses only the supplied function body and opaque dependency mapping. Prior prose and reviewer feedback are not treated as independent evidence of behavior.

The four-byte element representation supports the listed byte offsets. The supplied body establishes:

- Recognition of constants 15 and 19.
- An address comparison against an embedded receiver location.
- Two separate state conversions surrounding a receiver callback.
- One conditional single-byte zero write.
- Three indirect dispatch locations.
- One direct dependency call to F-42382d1a1b3b.
- Normal results of 0 or 1.

No dependency implementation, architecture metadata, full type definition, or runtime observation was supplied. Low-eight-bit retention for out-of-range character conversion is therefore an explicit conditional assumption, not an established property.

No runtime tests have been performed.
