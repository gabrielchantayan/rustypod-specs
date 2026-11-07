<!-- Trusted provenance: F-bf0cdff5c162; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bf0cdff5c162

## Purpose

F-bf0cdff5c162 coordinates a mode-dependent transition on an object and, when an object or related-object flag permits, performs a further update followed by notification and flag propagation.

The established behavior consists of:

- Optional dispatch through two object callbacks for a nonzero mode.
- A separate predicate-controlled callback sequence for mode zero.
- Conditional notification of an associated object.
- A gate controlled by bit `0x10` in the object or a previously captured related object.
- Setting bit `0x20` on the object before invoking an update dependency.
- A conditional callback through the object’s current related object.
- Setting bit `0x40` on an object returned by a final callback.

The business meaning of this transition is not established. Terms such as “enable,” “disable,” “visibility,” or “invalidation” would add semantics not supported by the supplied evidence.

## Interface

### Inputs

The visible interface has four inputs:

1. **Object address.** A pointer to mutable, word-addressed object storage.
2. **Mode.** An unsigned 32-bit value. Zero and nonzero select different early behavior. For one dependency argument, zero is distinguished from every nonzero value, including values greater than one.
3. **First forwarded value.** An opaque 32-bit value.
4. **Second forwarded value.** An opaque 32-bit value.

The update dependency also receives two additional opaque machine values appearing in the supplied call representation. Their provenance and interface status are unknown. They must not be specified as constants, omitted arguments, or values derived from the four visible inputs.

### Output

The function has no established return value. Its observable results are dependency calls, callback dispatches, and memory effects.

### Relevant storage

Offsets below are byte offsets. The supplied representation uses 32-bit words and 32-bit addresses.

| Base | Offset | Established use |
|---|---:|---|
| Object | `0x00` | Address of a dispatch table |
| Object | `0x34` | Related-object address; initially captured, later read again |
| Object | `0x3C` | Associated-object address |
| Object | `0x48` | 32-bit flags containing bits `0x10` and `0x20` |
| Object | `0x80` | Address passed to the update dependency as an interior pointer |
| Object | `0xA0` | Word whose low eight bits control a later callback |
| Associated object | `0x78` | Back-reference compared with the input object address |
| Related object | `0x00` | Address of a dispatch table |
| Captured related object | `0x48` | 32-bit flags tested for bit `0x10` |
| Final callback result | `0x48` | 32-bit flags updated with bit `0x40` |

The object dispatch table is accessed at byte offsets `0x1C`, `0x5C`, `0x114`, and `0x11C`. The related-object dispatch table is accessed at byte offset `0x128`.

These offsets describe accessed storage, not a complete type definition. The size and interpretation of other fields are unknown.

### Preconditions

The input object must be valid for every access reached on the selected path. There is no initial null check.

Every reached dispatch-table pointer, callback entry, related-object access, and associated-object access must be valid. The final callback must return an address valid for a 32-bit read and write at offset `0x48`; its result is not checked for null.

The supplied evidence establishes neither synchronization nor reentrancy protection. Callers must not assume that this operation is atomic.

## Behavior

### 1. Nonzero-mode preliminary callbacks

If the mode is nonzero, the function reads the object’s flags at `0x48` and tests bit `0x10`.

If the bit is set, it invokes two callbacks in this order:

1. Object dispatch-table entry `0x11C`, with the object as the visible argument.
2. Object dispatch-table entry `0x114`, with the object as the visible argument.

The second callback remains scheduled even if the first clears bit `0x10`: the branch decision has already been made. Conversely, if the initial bit is clear, neither callback runs merely because a later operation sets it.

The dispatch table used for the second callback is obtained through the object again. The contract does not require that the first callback preserve the object’s dispatch-table pointer.

For mode zero, this initial flag test and both callbacks are skipped.

### 2. Zero-mode predicate and callback sequence

For mode zero only, the function invokes F-a1a48bd3fbde with the object.

If its result is nonzero:

1. The object callback at dispatch-table offset `0x1C` is invoked with the object.
2. F-20460d1f31d3 is invoked with no visible arguments.

If the predicate returns zero, both calls are skipped.

This sequence is independent of the later flags gate. It can run even when the function subsequently returns before the update dependency.

The predicate’s result is used only as a zero/nonzero condition. No particular nonzero value is required.

### 3. Capture association state and optionally notify

After the preceding mode-dependent calls have completed, the function captures:

- The associated-object address from object offset `0x3C`.
- The related-object address from object offset `0x34`.

Consequently, changes made by earlier callbacks to either field affect these captured values.

If the captured associated-object address is nonzero, its back-reference at `0x78` is read. F-df6da5bd408b is invoked only if that back-reference equals the input object address.

Its visible arguments are:

- The captured associated-object address.
- A 32-bit value equal to one for mode zero and zero for every nonzero mode.

For mode one, the second argument is zero. For modes greater than one, it is explicitly zero rather than a wrapped subtraction result. In particular, a mode of `0xFFFFFFFF` produces zero.

A null associated-object address or a nonmatching back-reference skips this dependency. Neither condition directly prevents the later update stage.

### 4. Evaluate the update gate

After the optional associated-object notification, the function reads the object’s current flags at `0x48`.

- If bit `0x10` is set, the update stage proceeds without consulting the captured related object’s flags.
- Otherwise, the function uses the related-object address captured in the previous stage:
  - A null captured address causes an immediate return.
  - A nonnull captured address permits a read of its flags at `0x48`.
  - If bit `0x10` is clear there, the function returns.
  - If bit `0x10` is set there, the update stage proceeds.

This gate combines a **current object-flags read** with a **previously captured related-object address**.

If F-df6da5bd408b changes the object’s related-object field, that new address does not replace the captured address for this gate. If that dependency changes the flags in the originally captured related object, those changed flags can affect the gate.

An early return here does not undo preceding callbacks or dependency effects.

### 5. Mark the object and invoke the update dependency

Once the gate passes, the function updates the object’s 32-bit flags at `0x48` by setting bit `0x20`. Other bits in the value used for this read-modify-write are preserved.

It then invokes F-6d16652f2d03 with:

1. The object address.
2. The address at object offset `0x80`.
3. The first forwarded 32-bit value.
4. The second forwarded 32-bit value.
5. The first additional opaque machine value.
6. The second additional opaque machine value.

The second argument is an interior address, not a separately established allocation. No contents, length, ownership transfer, or lifetime extension for this region are established.

Bit `0x20` is set before this dependency is called. The function does not later reassert that bit, so the dependency or later callbacks may change it.

### 6. Conditionally dispatch through the current related object

After F-6d16652f2d03 returns, the function reads the object’s related-object field at `0x34` again.

If that field is null, this related-object callback is skipped, and execution continues to the final object callback.

If it is nonnull, the function reads the word at object offset `0xA0` and tests its low eight bits:

- Low byte zero: dispatch through the current related object’s table entry at `0x128`.
- Low byte nonzero: skip that callback.

The upper 24 bits of the word do not affect this decision. Signed interpretation of a nonzero low byte does not affect the zero comparison.

The supplied call representation shows no explicit argument to this related-object callback. It does not establish an explicit receiver argument or a fuller calling convention.

This stage uses the related-object field after the update dependency, not necessarily the address captured for the earlier gate.

### 7. Obtain a destination and set its flag

The function invokes the input object’s current dispatch-table entry at `0x5C`, passing the object.

The returned value is used as an address without a null check. At offset `0x48` from that address, the function reads a 32-bit value and writes it back with bit `0x40` set. Other bits from that read are preserved.

This callback and flag update occur whenever execution reaches this stage, regardless of whether the optional related-object callback ran.

The destination may alias the input object or another object; the function does not enforce distinctness. After the flag write, it returns.

## Side effects

### Direct writes

The function contains two established direct writes:

- Set bit `0x20` at input-object offset `0x48`, only after the update gate passes.
- Set bit `0x40` at offset `0x48` of the final callback’s returned address.

Both are 32-bit read-modify-write operations. They are not established as atomic and can lose concurrent changes without external synchronization.

There is no direct clearing of flags and no direct write to the association fields in the supplied evidence.

### Indirect effects and ordering

Dependencies and callbacks may have additional effects. Their behavior, including mutation of fields, global state, allocation, release, or nonlocal control transfer, is not supplied.

Observable ordering is material:

- Preliminary callbacks precede the association captures.
- Associated-object notification precedes the update gate.
- The object’s `0x20` write precedes F-6d16652f2d03.
- The related-object callback, if any, follows that dependency.
- The final object callback precedes the returned destination’s `0x40` write.

There is no established rollback, error translation, retry, or cleanup path. If a call does not return normally, later operations are not established to occur.

### Ownership

No direct allocation or release appears in this function. That does not establish that callees preserve ownership or object lifetime.

Correct execution requires relevant storage to remain valid until its last reached use. In particular, the captured related-object address can be used after the associated-object dependency returns.

## Dependencies

| Identifier or dispatch entry | Invocation condition | Visible contract |
|---|---|---|
| F-a1a48bd3fbde | Mode zero | Receives the object; nonzero result selects two subsequent calls |
| Object entry `0x1C` | Zero-mode predicate returns nonzero | Receives the object; precedes F-20460d1f31d3 |
| F-20460d1f31d3 | Immediately after entry `0x1C` | No visible arguments; result unused |
| Object entries `0x11C`, `0x114` | Nonzero mode and initial object bit `0x10` set | Receive the object; run in that order |
| F-df6da5bd408b | Nonnull captured association with matching back-reference | Receives association and a zero-mode indicator |
| F-6d16652f2d03 | Update gate passes | Receives object, interior address, two forwarded values, and two opaque machine values |
| Related-object entry `0x128` | Current related object nonnull and object low byte at `0xA0` zero | No explicit arguments shown |
| Object entry `0x5C` | Update stage reaches its final dispatch | Receives object; result supplies destination address |

No opaque identifier is available for the indirect callbacks; their dispatch-table offsets distinguish them without assigning invented identities.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Unless stated otherwise, all reached addresses are valid, all calls return normally, and callees preserve fields relevant to the example.

1. **Zero mode, no update eligibility.**  
   The predicate returns zero, the association is null, object bit `0x10` is clear, and the captured related object is null. Only F-a1a48bd3fbde is invoked. Neither direct flag write occurs.

2. **Predicate success still permits an early return.**  
   Under the same gate conditions, a nonzero predicate result causes entry `0x1C` and then F-20460d1f31d3 to run. The function subsequently returns without invoking F-6d16652f2d03.

3. **Nonzero mode with object eligibility.**  
   Mode is two and object bit `0x10` is initially set. Entries `0x11C` and `0x114` run in order. The zero-mode predicate is not called. If an associated object has a matching back-reference, its notification argument is zero. The update stage runs while eligibility remains present.

4. **Related-object eligibility.**  
   Object bit `0x10` is clear, but the captured related object is nonnull and has bit `0x10` set. The gate passes, and the object receives bit `0x20` before the update dependency runs.

5. **Captured versus current relationship.**  
   The association dependency changes the object’s related-object field from address A to address B. With object bit `0x10` clear, the gate consults A’s flags. If the gate passes, the later related-object callback consults the field again and can dispatch through B.

6. **Low-byte condition.**  
   After the update dependency, a nonnull related object and a word value of `0x00000100` at object offset `0xA0` permit its callback. A value of `0x00000001` suppresses it.

7. **Final flag preservation.**  
   If the final callback returns a destination whose flags are `0x00000005`, the direct final write produces `0x00000045`. A destination already containing bit `0x40` still receives the read-modify-write.

8. **Initial branch is not rechecked between callbacks.**  
   The first nonzero-mode callback clears object bit `0x10`. Entry `0x114` still runs. If no eligible captured related object remains, the later update gate can nevertheless fail.

## Uncertainties

The following are **documented_only**:

- The transition’s application-level purpose and meanings of the three flag bits.
- Complete object layouts, allocation sizes, alignment guarantees, and field names.
- The provenance of the two additional machine values passed to F-6d16652f2d03.
- Full calling-convention details for the related-object callback.
- Dependency ownership, memory-release behavior, global-state effects, and error behavior.
- Reentrancy, concurrency guarantees, and whether callbacks may safely replace associations or dispatch tables.
- Whether malformed pointers trap, fault, or have another platform-specific outcome.
- Whether the final destination is normally the input object or a distinct object.

The grouping of fields as “related” and “associated” is descriptive. Their domain roles are inferred, not established.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. Established details include branch conditions, 32-bit field operations, byte offsets, visible arguments, callback order, early-return conditions, and the two direct flag writes.

No dependency bodies, runtime traces, ownership declarations, or external type definitions were supplied. No runtime tests have been performed. Acceptance examples describe required outcomes under their stated conditions; they are not verification results.
