<!-- Trusted provenance: F-b4ddeaf855f2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b4ddeaf855f2

## Purpose

F-b4ddeaf855f2 dispatches on a signed selector and a 32-bit position/value, uses context fields and dependencies to obtain or construct a result, and writes one 32-bit output on successful completion.

Most result paths operate through a globally referenced mutable object and a final accessor. One conditional path writes a dependency result directly, bypassing shared assignment and finalization.

**Established behavior:** dispatch precedence, memory-access widths, dependency arguments and ordering, selected shared-state writes, and explicit status returns.

**Inferred intent:** the operation resembles indexed retrieval and formatting. Terms such as “count,” “lookup,” “assignment,” and “formatting” describe observed roles, not established dependency semantics.

Returning 1 means that the function reached an output-writing path. It does not establish that the output is nonzero, valid text, newly allocated, or caller-owned. Explicit rejection returns 0 without a direct output-slot write, but initialization may already have produced effects.

## Interface

### Arguments and return value

| Argument | Representation | Observed use |
|---|---|---|
| Context | Address represented in a 32-bit argument | Supplies fields and an embedded-object address. |
| Selector | Signed 32-bit integer | Controls dispatch through equality tests, signed ordering, and signed arithmetic. |
| Position/value | Unsigned 32-bit integer | Used as an index, arithmetic operand, or formatting input; bounded paths also interpret its bits as signed. |
| Output location | Address of a four-byte slot | Receives one 32-bit result on each normally completed result-writing path. |

The return value is unsigned 32-bit. All explicit returns are 0 or 1.

### Context layout

| Byte offset | Established access |
|---|---|
| `+0xBC` | Read as a four-byte value passed to F-5fca8e42aa3d. |
| `+0xC0` | Address passed to F-5836102a55c4. |
| `+0xC4` | Read as a four-byte count-like value; some comparisons interpret it as signed. |

The object beginning at `+0xC0` has no established extent or internal layout. The field at `+0xC4` may belong to that object.

### Shared state and payload terminology

- **Shared reference:** a global reference to a mutable object.
- **Cached object:** the shared-reference value saved at entry.
- **Initialization record:** a globally referenced record whose four-byte flag word is read at offset `+0x10`.
- **Boundary:** a global signed 32-bit selector value.
- **Adjacent selector:** the represented signed calculation boundary plus 2.
- **Header payload:** a global four-byte value written into the cached object’s first word during initialization.
- **Early initialization payload:** a global four-byte value captured immediately after F-b255b26284e6 returns.
- **Late initialization payload:** a global four-byte value read after the initialization object writes.
- **Other payloads:** distinct global values or addresses used by particular branches.

These are descriptive labels, not semantic names or assertions about ownership.

### Preconditions

Every location actually accessed must support the relevant read or write. The output slot must be writable on a result-writing path. Dependency-specific validity and lifetime requirements must also hold.

The indirect path requires a valid first word in the cached object and a valid callable entry at byte offset `+0x0C` in the table referenced by that word.

There is no general pointer validation. An argument unused directly on a particular path is not thereby guaranteed safe when invalid: dependencies and aliasing may still make it relevant.

## Behavior

### Initialization and payload-read ordering

Before dispatch, the function saves the shared-reference value as the cached object, then reads the initialization flag word.

If flag bit 0 is set, the initialization sequence is skipped.

If bit 0 is clear, the following order is established:

1. F-b255b26284e6 is called with an address 24 bytes before the shared object obtained from the global reference for that call.
2. After the call returns, the early initialization payload is read and cached.
3. The dependency’s return value determines whether initialization continues.

The early payload read occurs **even when F-b255b26284e6 returns zero**. That read is not conditional on initialization proceeding.

For a zero return, there are no further visible initialization object writes or initialization calls.

For a nonzero return:

1. Read the header payload and write it into the cached object’s first four-byte word.
2. Clear the cached object’s second four-byte word.
3. Read the late initialization payload.
4. Call F-804adc29bbfb with the cached object, the late payload as its second argument, and the previously captured early payload as its third argument.
5. Call F-d08d3ca17b81 with the address 24 bytes before the cached object.

The third argument to F-804adc29bbfb is **not a fresh payload read**. Its value predates both object writes. Conversely, the late payload read follows both writes. Aliasing between payload storage and the cached object can therefore affect the two arguments differently.

F-b255b26284e6 may change global references or payloads. The writes and subsequent initialization calls nevertheless use the entry-cached object. The function does not visibly set initialization flag bit 0 through a dedicated flag assignment; dependencies or aliasing may change it.

Initialization precedes even eventual rejection. Boundary reads and dispatch follow this sequence, so initialization dependency mutations can affect dispatch.

### Dispatch precedence and arithmetic qualification

The selector is first tested for equality with the boundary. Equality selects the bounded-index family.

Otherwise:

- A signed selector smaller than the boundary enters the lower-selector family.
- A larger selector is tested against the adjacent selector, then `0x36E0`, then `0x6097`.

Boundary equality takes precedence over numeric special cases. The adjacent-selector match takes precedence over the two larger-family numeric cases.

The representation also evaluates boundary plus 2 as signed arithmetic. Its overflow behavior is not established. The branch descriptions below require that calculation to have a defined interpretation; no universal behavior is claimed for an overflowing boundary value.

Within the lower family, there is another signed operation: selectors other than `0x368F` and `0x3695` undergo subtraction of `0x3600` before a comparison with `0x9A`. This is not safely replaceable by an unconditional equality dispatch for every signed input. Its effect and uncertainty are specified below.

### Bounded-index family

This family handles boundary equality and a reachable adjacent-selector match.

First, the position’s signed 32-bit interpretation is checked. If negative, return 0 without reading the context count.

Otherwise, read the four-byte count at context offset `+0xC4`. Reject if the count equals the position or if the signed count is smaller than the signed position.

Thus acceptance requires:

- Position in the range 0 through `0x7FFFFFFF`; and
- Signed count strictly greater than position.

Zero is not independently rejected. Count zero or a negative signed count rejects every nonnegative position.

After acceptance, call F-5836102a55c4 with the context address at `+0xC0` and the position.

For the boundary selector:

- A nonzero lookup result is followed by F-3c6b67b2a91d and shared assignment of its result.
- A zero lookup result is followed by F-46f0b3cb316e with the then-current shared reference and a fallback payload, then shared finalization without shared assignment.

For the adjacent selector:

- A nonzero lookup result is followed by F-95d4c0417f99 and shared assignment.
- A zero lookup result causes another F-5836102a55c4 call with position minus 1. Its return value is ignored. F-709a25ead270 then supplies the shared-assignment input.

There is no second bounds check. An accepted zero position can therefore produce a fallback lookup at `0xFFFFFFFF`.

The boundary and adjacent values used for post-lookup selection are retained values, not fresh reads after the lookup dependency.

### Lower-selector family

This family requires the selector to be signed-less-than the boundary.

Selector `0x368F` proceeds directly to the position-dependent construction family below. Selector `0x3695` proceeds directly to the count-gated lookup family.

For every other lower-family selector, the representation first performs signed selector minus `0x3600`:

- If that result equals `0x9A`, use the count-gated lookup family.
- Otherwise, selector `0x36A5` uses position-dependent construction.
- Otherwise, return 0 without a direct output write.

When subtraction is representable, its equality test identifies selector `0x369A`. For sufficiently negative selectors, subtraction can overflow; behavior for those inputs is **documented_only**, not an established unsupported-selector return.

#### Count-gated lookup

This covers reachable `0x3695` and, with defined subtraction, `0x369A`.

Read the four-byte field at context offset `+0xC4`:

- If zero, proceed directly to shared finalization.
- If nonzero, call F-5836102a55c4 with the context subobject address and supplied position, ignore its return, then call F-3c6b67b2a91d and perform shared assignment.

There is no bounded-family index validation. A high-bit-set count is nonzero and permits the lookup; a high-bit-set position is passed unchanged.

#### Position-dependent construction

This covers reachable `0x368F` and, after the intervening subtraction test, `0x36A5`.

At position zero:

- Call F-9d0ecc5f6adf.
- After that call returns, read the zero-position payload.
- Pass the dependency result and payload to F-c0b020cd3882.

At a nonzero position:

1. Read the context field at `+0xBC` and call F-5fca8e42aa3d.
2. If its result plus 1 equals the position, call F-9d0ecc5f6adf, then read the first trailing-position payload and call F-c0b020cd3882.
3. Otherwise, reread the context field and call F-5fca8e42aa3d again.
4. If this second result plus 2 equals the position, call F-9d0ecc5f6adf, then read the second trailing-position payload and call F-c0b020cd3882.
5. Otherwise, call F-3f052ce6f06c with the context and position minus 1.

The selected result becomes the shared-assignment input.

The two context reads and two possible F-5fca8e42aa3d calls are distinct observations. Neither their inputs nor their results are guaranteed equal. Payload reads after F-9d0ecc5f6adf observe any changes that dependency makes to their storage.

### Larger-selector cases

These require signed selector greater than boundary and failure of the adjacent-selector match.

#### Selector `0x36E0`

The function calls F-a062a6eb1219 with the position/value and temporary storage represented as three bytes.

It subsequently reads a one-byte temporary immediately following that storage and passes the byte to F-e8df72c5dacc, together with:

- A separate four-byte temporary destination;
- The integer 3;
- The address of an unknown constant.

The destination address becomes the shared-assignment input.

The function does not visibly initialize the one-byte temporary before reading it. Whether F-a062a6eb1219 writes that byte is unknown. Neither the meaning of 3 nor destination initialization, bounds, and termination guarantees are established.

#### Selector `0x6097`

Read context offset `+0xBC`, call F-5fca8e42aa3d, and compare its result plus 2 with the position.

On equality:

1. Call F-23e8d37d5f69.
2. Pass its result, a global payload read for the subsequent call, and `0x6097` to F-5710a8628cba.
3. Write the returned four-byte value directly to the output slot.
4. Return 1.

This path bypasses shared assignment and finalization.

On inequality, obtain an indirect callable from table byte offset `+0x0C`, using the cached object’s first word as the table reference. Invoke it with the then-current global shared reference, then perform shared finalization.

The object supplying the callable and the object passed as its argument need not be the same. No relative evaluation order between independent callable-resolution and argument reads is established here.

Other larger-family selectors return 0 without a direct output write.

### Shared assignment, finalization, and integer widths

Shared assignment calls F-151ab4bfe553 with the then-current shared reference and the selected result or temporary-buffer address.

Shared finalization calls F-ecd1af0519f9 with the then-current shared reference, writes its four-byte result to the output slot, and returns 1. The shared reference is read again for finalization; assignment may have changed it.

Position arithmetic uses a 32-bit unsigned domain:

- `0xFFFFFFFF` plus 1 becomes zero.
- `0xFFFFFFFF` plus 2 becomes one.
- Zero minus 1 becomes `0xFFFFFFFF`.

The explicit position-zero case precedes the lower-family plus-1 comparison.

Signed arithmetic has separate unresolved cases:

- Boundary plus 2 exceeds signed range when boundary is greater than `0x7FFFFFFD`.
- Lower-family selector minus `0x3600` is below signed range when selector is less than \(-2^{31} + \texttt{0x3600}\).

No wraparound rule is established for those signed operations.

## Side effects

Direct writes consist of initialization’s two cached-object word writes and one four-byte output write on a normally completed result path. Aliasing can make those writes affect context fields, payload storage, flags, or references.

Dependencies may mutate memory and globals, allocate or release resources, or fail by mechanisms not shown. No dependency is assumed to preserve globals.

There is no visible direct allocation or release. Pointer-shaped results have no established ownership transfer. Passing temporary stack storage to F-151ab4bfe553 does not establish that it copies the data; retaining that address beyond this invocation would require an unavailable lifetime contract.

No exception handling, rollback, error object, or dependency-failure translation is visible. Status 1 does not validate dependency results. Status 0 does not imply absence of dependency effects or indirect writes to aliased output storage.

## Dependencies

| Opaque ID | Established role |
|---|---|
| F-b255b26284e6 | Gates continuation of initialization. |
| F-804adc29bbfb | Receives cached object, late initialization payload, and cached early payload. |
| F-d08d3ca17b81 | Follows initialization writes and initialization call. |
| F-5836102a55c4 | Receives context subobject address and index; its result sometimes controls branching. |
| F-3c6b67b2a91d | Supplies an assignment input after selected lookup paths. |
| F-95d4c0417f99 | Supplies an input after nonzero adjacent-selector lookup. |
| F-709a25ead270 | Supplies an input after adjacent-selector fallback lookup. |
| F-46f0b3cb316e | Receives shared reference and boundary-fallback payload. |
| F-5fca8e42aa3d | Supplies values for position comparisons. |
| F-9d0ecc5f6adf | Supplies an argument for position-dependent construction. |
| F-c0b020cd3882 | Receives that argument and a selected payload. |
| F-3f052ce6f06c | Receives context and position minus 1. |
| F-a062a6eb1219 | Receives numeric input and temporary storage. |
| F-e8df72c5dacc | Receives temporary destination, 3, constant address, and one-byte value. |
| F-23e8d37d5f69 | Supplies an argument for direct-result construction. |
| F-5710a8628cba | Supplies the direct output value. |
| F-151ab4bfe553 | Receives shared reference and assignment input. |
| F-ecd1af0519f9 | Supplies the shared-finalization output. |

The indirect dependency has no supplied opaque ID. Its table offset, argument source, and invocation position are established; its semantics are not.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless an example specifically addresses uncertainty, assume defined signed selector arithmetic, valid accessed storage, and normally returning dependencies.

1. **Initialization gate returns zero:** With flag bit 0 clear, F-b255b26284e6 returns zero. The early initialization payload is still read afterward. No initialization object writes or subsequent initialization calls occur.

2. **Initialization payload aliasing:** The gate returns nonzero. Capture early payload value A, then perform the header write and second-word clear. If those writes change early-payload storage to B, F-804adc29bbfb still receives A as its third argument. Its second argument comes from the later payload read.

3. **Accepted zero index:** Boundary 100, selector 100, position zero, count 3: call F-5836102a55c4 at zero.

4. **Bounded rejection:** With the same boundary, count 3 and position 3 return 0 without lookup. Position `0x80000000` returns 0 without even reading the count.

5. **Adjacent underflow:** Boundary 100, selector 102, count 1, position zero, and zero first lookup result: perform the second lookup at `0xFFFFFFFF`, then call F-709a25ead270 and shared assignment.

6. **Unchecked lower index:** Boundary `0x4000`, selector `0x3695`, nonzero count, position `0xFFFFFFFF`: pass that position unchanged to F-5836102a55c4.

7. **Repeated query:** For reachable `0x368F` at nonzero position, failure of the first plus-1 comparison requires a second field read and query before choosing the plus-2 or ordinary path.

8. **Defined unsupported selector:** Boundary `0x4000`, selector `0x3601`: subtraction yields 1, no lower-family case matches, and the function returns 0 after any initialization effects.

9. **Overflowing lower subtraction:** Boundary zero and selector \(-2^{31}\): lower-family dispatch reaches an overflowing subtraction. A return-0 guarantee is not established.

10. **Direct output and zero results:** Reachable `0x6097` with a matching plus-2 comparison writes F-5710a8628cba’s result without shared finalization. Independently, shared finalization returning zero writes zero and still returns 1.

## Uncertainties

The following remain **documented_only**:

- Actual global values, payload contents, selector meanings, and resulting branch reachability.
- Signed-overflow behavior for both boundary addition and lower-family subtraction.
- Dependency validity requirements, failures, synchronization, ownership, copying, allocation, release, and global mutations.
- Whether initialization helpers update the flag or provide one-time initialization.
- Whether temporary-storage preparation initializes the subsequently read byte.
- Formatting destination bounds, termination, and lifetime handling.
- Meanings of returned values and of the indirect operation.
- Behavior under concurrent mutation; no concurrency guarantee is established.

## Evidence

This contract derives from static inspection of the supplied function representation and opaque dependency mapping. Established observations are access widths and offsets, explicit data dependencies, branch and call ordering, visible writes, argument sources, and explicit returns.

No dependency implementations, constant contents, external contracts, or runtime traces were supplied. No runtime tests have been performed. Acceptance examples express conditional consequences of the supplied representation, not empirical results.
