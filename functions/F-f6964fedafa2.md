<!-- Trusted provenance: F-f6964fedafa2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f6964fedafa2

## Purpose

F-f6964fedafa2 coordinates a selector-dependent sequence involving a context, a recipient, and opaque dependencies. It has three principal outcomes:

- A recognized selector with a nonzero marker causes immediate delegation.
- A zero descriptor-lookup result causes an indirect recipient notification.
- A nonzero descriptor-lookup result permits the remaining object-related, placement, and registration calls, followed by a marker write for recognized selectors.

Selectors 0, 1, 2, and 4 have marker bytes. Other selector values bypass marker access but remain eligible for descriptor lookup and the complete subsequent sequence.

“Context,” “recipient,” “descriptor,” “object,” “placement,” and “registration” are descriptive role labels. They do not establish domain types or the internal behavior of dependencies. In particular, the call to F-8157a180d901 establishes neither allocation nor a byte-count argument.

## Interface

The function accepts six inputs and has no explicit return value.

| Input | Representation | Established role |
|---|---|---|
| Context | 32-bit address-bearing value | Base for marker accesses and an embedded destination; passed to dependencies |
| Recipient | Address represented in the supplied 32-bit model | Supplied to dependencies; provides a dispatch-table chain on lookup failure |
| Selector | Signed 32-bit integer | Selects a marker where defined and is passed unchanged to descriptor lookup |
| Placement selector | Signed 32-bit integer | Zero selects displacement 320; any nonzero value selects an external displacement |
| Mode selector | Signed 32-bit integer | Zero selects mode 7; any nonzero value selects mode 10 |
| Option word | Unsigned 32-bit integer | Passed to F-2698159acdb1 after XOR with 1 |

There is no explicit status return. A normal return identifies which visible sequence finished, not whether an opaque dependency achieved an intended semantic result.

### Referenced memory layout

Offsets are byte offsets from the stated base.

| Base | Offset | Access | Role |
|---|---:|---|---|
| Context | `0x98` | One-byte read; conditional one-byte write | Marker for selector 0 |
| Context | `0x99` | One-byte read; conditional one-byte write | Marker for selector 1 |
| Context | `0x9A` | One-byte read; conditional one-byte write | Marker for selector 2 |
| Context | `0x9B` | One-byte read; conditional one-byte write | Marker for selector 4 |
| Context | `0x58` | Address formed and passed onward | Embedded destination; no direct field read is shown |
| Recipient | `0x00` | Four-byte address-bearing read | Dispatch-table address on lookup failure |
| Recipient dispatch table | `0xA0` | Function-address read in the supplied 32-bit model | Indirect notification target |
| Result of F-38aa1bb9efd0 | `0xE8` | Four-byte read | First argument to F-db754b1ab3c7 |
| Descriptor | `0x34` | Four-byte read | Second placement word |
| Descriptor | `0x30` | Signed 32-bit read | Base for the first placement word |
| Result of F-2698159acdb1 | `0x04` | Four-byte read | Associated value retained for later calls |

Three external four-byte storage locations supply a creation parameter, a displacement, and a registration parameter. Their actual values and meanings are not supplied. They are read at distinct stages, not captured together at function entry.

Two temporary records are exposed by address:

- A placement record containing two consecutive four-byte words: computed placement first, captured descriptor value second.
- A four-byte slot containing the retained associated value.

No wider representation, portable address model, or alignment guarantee is established.

### Preconditions

No explicit validation is present. Memory must support the accesses reached by the selected path.

A recognized selector requires a readable marker. Its successful full path also requires that marker to be writable. Lookup requires a readable field at offset `0xE8` from the provider result. A nonzero descriptor requires readable fields at offsets `0x34` and `0x30`. The result of F-2698159acdb1 must support the read at offset `0x04`.

A zero lookup result requires a readable recipient dispatch-table chain and a callable entry at offset `0xA0`.

These are access requirements, not checks performed by the function. Requirements imposed by opaque callees remain additional and unknown.

## Behavior

### Marker gate

The selector determines whether the function reads a marker:

| Selector | Marker offset |
|---:|---:|
| 0 | `0x98` |
| 1 | `0x99` |
| 2 | `0x9A` |
| 4 | `0x9B` |

For one of these selectors, any nonzero marker byte causes F-d4b95dc5d36c to receive the context, recipient, and integer 1. The function returns when that call returns normally.

On that path, this function does not acquire the registration handle, perform descriptor lookup, call F-8157a180d901, or perform its own marker write.

A zero marker permits processing. Every other selector bypasses these marker reads entirely. The function does not normalize or reject an unrecognized selector.

There is no second marker check. Changes made by dependencies after the initial read do not cause the current invocation to reconsider this gate.

### Handle acquisition and descriptor lookup

Once the gate permits processing, the observable sequence is:

1. Call F-7a89f2f7fd9a and retain its return value as the registration handle.
2. Call F-38aa1bb9efd0.
3. Read four bytes at offset `0xE8` from that result.
4. Call F-db754b1ab3c7 with the field value and original selector.

The registration handle is acquired even when descriptor lookup subsequently returns zero. It is not reacquired later.

No null check is shown before the provider-field read. Only the descriptor-lookup result controls the next branch.

### Zero descriptor result

If F-db754b1ab3c7 returns zero, the function reads the recipient’s dispatch-table address, reads the callable entry at table offset `0xA0`, and invokes that entry with the recipient and integer 0.

The function returns after that invocation returns normally. It performs none of the subsequent object-related, placement, value-supplier, or registration calls, and makes no explicit marker write.

The earlier handle-acquisition and provider calls have already happened. No rollback is shown.

This indirect recipient operation is not established to be F-d4b95dc5d36c or to share its semantics.

### Nonzero descriptor result

For a nonzero lookup result, the function first captures:

1. The four-byte descriptor field at `0x34`.
2. The signed 32-bit descriptor field at `0x30`.

That read order precedes F-2698159acdb1. The function then calls F-2698159acdb1 with six arguments, in this order:

1. Context.
2. Integer 0.
3. Descriptor.
4. The external four-byte creation parameter.
5. Integer 0.
6. The option word XOR 1.

The creation parameter is read for this call after the preceding acquisition and lookup calls. Its entry-time value must not be assumed to survive those calls.

The XOR toggles the least significant bit of the full 32-bit option word and preserves the other bits. It is not Boolean negation.

The return value of F-2698159acdb1 is used as an object address without a zero check or another explicit validity check. The function reads four bytes at offset `0x04` and retains that associated value.

Neither the dependency’s name here nor this use proves that it allocates a new object.

### Placement record

After capturing the associated value, the function reads the external displacement. If the placement selector is zero, it replaces that displacement with `0x140`, or 320 decimal. Otherwise, it retains the external displacement unchanged.

The external displacement read occurs even when the zero selector subsequently replaces its value.

The two-word placement record presented to F-a8c06eac92c0 contains:

- First word: the previously captured descriptor value from `0x30`, plus the selected displacement.
- Second word: the previously captured descriptor value from `0x34`.

The addition uses represented 32-bit integer values. There is no explicit overflow check, saturation, or error branch. The supplied representation does not establish a portable signed-overflow rule; cases requiring such a rule remain documented_only.

F-a8c06eac92c0 receives, in order:

1. Context.
2. Object returned by F-2698159acdb1.
3. Descriptor.
4. Address of the placement record.
5. Integer 0.

The record is passed by address, not as two independent scalar arguments. The function does not subsequently use its contents. Whether the callee modifies it or retains its address is unknown.

Changes to descriptor fields by F-2698159acdb1 do not change the captured placement inputs used by this invocation.

### Opaque value supply and mode-dependent call

After F-a8c06eac92c0 returns normally, the function calls F-8157a180d901 with the integer `0x24`, which is 36 decimal.

Only that argument value and the subsequent use of the return value are established. The supplied body does not establish:

- That this is an allocation operation.
- That the argument measures bytes, elements, or any other unit.
- That the return value is an address.
- That zero denotes failure.

The return value is passed unchecked as the first argument to F-3a954974c1bb.

The remaining arguments to F-3a954974c1bb are, in order:

2. Retained associated value.
3. Recipient.
4. Descriptor.
5. Selected mode.

The selected mode is 7 when the mode selector is zero, and 10 for every nonzero mode selector. The return value of F-3a954974c1bb is not used.

Calling this dependency an initializer would be an interpretation, not an established contract.

### Registration sequence

After F-3a954974c1bb returns normally, the function makes three calls in order:

1. F-4b539170a43a receives the retained registration handle and object.
2. F-aa408aa1a280 receives that same handle, the retained associated value, and the external four-byte registration parameter.
3. F-7e55b35f8edc receives the address at context offset `0x58` and the address of a temporary four-byte slot containing the retained associated value.

The external registration parameter is read for the second call, after F-4b539170a43a returns. Earlier dependencies must not be assumed to preserve that external storage.

The temporary associated-value slot is populated after F-aa408aa1a280 returns and before F-7e55b35f8edc is called.

The handle remains the value captured before provider lookup. The associated value remains the value read from the object before placement. Neither is refreshed from external state during this sequence.

### Final marker assignment

Only after all three registration calls return normally does the function write byte value 1 to the marker for selector 0, 1, 2, or 4.

There is no comparison with the marker’s current value. This write can replace a value written by an intervening dependency.

For every other selector, the function performs no marker write, including after the full sequence succeeds in returning normally.

## Side effects

### Direct writes and exposed addresses

The only explicit persistent data write shown is the final one-byte marker assignment for a recognized selector.

The function also writes its temporary placement record and temporary associated-value slot. Dependencies receive their addresses. Those addresses are established as usable during the respective calls; later use requires a lifetime contract not supplied here.

Passing the address at context offset `0x58` does not itself establish a direct read or write of that embedded destination by this function.

### Dependency-mediated effects

Opaque dependencies can have effects not described by this body. No guarantee is established that they preserve globals, descriptors, markers, or other reachable state.

Consequently:

- Absence of an explicit marker write on an early-return path does not guarantee the marker remains unchanged.
- Saved scalar values remain the previously captured values even if related external state changes.
- External parameters read later can reflect effects of earlier dependencies.
- Passing an address does not establish whether the callee copies, mutates, retains, owns, or releases the referenced data.

### Ownership and abnormal completion

No explicit release operation is shown for the descriptor, object, registration handle, associated value, or value returned by F-8157a180d901.

This does not establish a leak. It also does not establish that any of those values represent owned resources. In particular, neither allocation nor ownership of the F-8157a180d901 result is known.

There is no explicit rollback, retry, or recovery sequence. If a dependency does not return normally, the subsequent calls and final marker write are not reached by the visible control flow. Prior effects may remain.

## Dependencies

| Opaque ID | Established boundary behavior | Unresolved behavior |
|---|---|---|
| F-d4b95dc5d36c | Receives context, recipient, and 1 when a recognized marker is nonzero | Meaning, mutations, ownership effects |
| F-7a89f2f7fd9a | Returns a value retained for two later calls | Identity, validity, lifetime, ownership |
| F-38aa1bb9efd0 | Returns the base for the read at `0xE8` | Provider semantics and failure conventions |
| F-db754b1ab3c7 | Receives provider field and selector; zero selects recipient dispatch | Supported selector domain and descriptor lifetime |
| F-2698159acdb1 | Receives six stated arguments; its result is used for the `0x04` read | Creation semantics, validity, ownership, failure conventions |
| F-a8c06eac92c0 | Receives context, object, descriptor, temporary placement address, and 0 | Placement meaning and address-retention rules |
| F-8157a180d901 | Receives integer `0x24`; its return value is forwarded unchecked | Purpose, argument units, return representation, failure conventions |
| F-3a954974c1bb | Receives the forwarded value, associated value, recipient, descriptor, and mode | Initialization semantics, validation, ownership effects |
| F-4b539170a43a | Receives saved handle and object | Exact effects |
| F-aa408aa1a280 | Receives saved handle, associated value, and late-read external parameter | Exact effects |
| F-7e55b35f8edc | Receives embedded-destination address and temporary-slot address | Copying, retention, mutation, ownership |

The recipient’s indirect callable is an additional dependency without a supplied opaque ID. Its established arguments are the recipient and integer 0.

## Acceptance examples

These are conditional acceptance examples derived from static evidence, not executed tests. They assume valid reached memory accesses and normally returning dependencies unless stated otherwise.

1. **Nonzero recognized marker.** Selector 2 with marker `0x9A` equal to 7 invokes F-d4b95dc5d36c with context, recipient, and 1, then returns. The marker need not equal exactly 1.

2. **Missing descriptor.** Selector 1 with a zero marker at `0x99` reaches handle acquisition and provider lookup. If descriptor lookup returns zero, the recipient’s indirect operation receives integer 0. No later object-related or registration sequence occurs.

3. **Zero placement selector.** A captured descriptor base of 100 and second word of 45 produce placement words 420 and 45. The external displacement is read but does not determine the first word.

4. **Nonzero placement selector.** A captured base of 100, external displacement of 24, and placement selector −1 produce first word 124. Every nonzero placement selector selects the external value.

5. **Mode selection.** A zero mode selector passes 7 to F-3a954974c1bb. Both 1 and −1 pass 10.

6. **Full-width option transformation.** Option words 0, 1, 2, and `0xFFFFFFFF` become 1, 0, 3, and `0xFFFFFFFE`, respectively.

7. **Unrecognized selector.** Selector 3 reads none of the four marker bytes. If lookup returns a usable descriptor and all subsequent calls return normally, the full sequence executes without a final marker write.

8. **Captured descriptor values.** If F-2698159acdb1 modifies descriptor fields `0x30` and `0x34`, placement still uses their earlier captured values.

9. **Late registration-parameter read.** If F-4b539170a43a changes the external registration parameter before returning, F-aa408aa1a280 receives the value read afterward, not an entry-time snapshot.

10. **Unchecked zero forwarding.** If F-8157a180d901 returns zero, F-3a954974c1bb receives zero as its first argument. Zero is not thereby classified as an allocation failure. The downstream outcome is documented_only.

11. **Marker timing.** If F-7e55b35f8edc does not return normally, this function has not performed its final marker assignment. Earlier effects are not undone.

## Uncertainties

### Inferred intent

The markers suggest duplicate-operation handling. The object-related calls and later sequence suggest association or registration. The two-word record suggests placement data.

An allocation-and-initialization interpretation of F-8157a180d901 followed by F-3a954974c1bb is unverified. It must not supply requirements concerning byte counts, memory ownership, null handling, or resource release.

### documented_only

The following remain unresolved:

- Meanings and values of the three external parameters.
- Selector validity beyond the four marker mappings.
- Opaque dependency success and failure conventions.
- The purpose, argument units, and return-value meaning of F-8157a180d901.
- Resource ownership, reference retention, and release responsibilities.
- Whether temporary addresses are copied or retained.
- Reentrancy, synchronization, and thread safety.
- Outcomes for invalid addresses or arithmetic overflow.
- Semantic success beyond completion of the visible call sequence.

The delayed marker assignment does not establish atomic duplicate prevention. A reentrant invocation before that assignment could observe a zero marker unless dependency behavior prevents it.

## Evidence

Established claims derive from static inspection of the supplied body: branch conditions, access widths and offsets, captured values, argument relationships, call ordering, and marker assignments.

Dependency internals are not supplied. Their purposes and lifecycle behavior are not inferred from argument constants or suggestive call sequences. In particular, integer `0x24` establishes a numeric argument, not a 36-byte allocation request.

No runtime tests have been performed.
