<!-- Trusted provenance: F-47de6633aece; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-47de6633aece

## Purpose

F-47de6633aece performs an ordered update of two externally addressed 32-bit words. It records an unsigned request value, passes a derived configuration word to F-5c24bc78359c, writes that dependency’s result to a previously selected destination, and waits until bits 16–23 of that destination match the corresponding bits of the result.

The established behavior is a register-like update and readback sequence. Interpreting the words as hardware registers, the request as a divisor, or the polling loop as hardware synchronization is **inferred intent**, not an established interface guarantee.

There is no explicit validation, timeout, or failure return. If the function returns normally, its 32-bit result is zero.

## Interface

### Inputs

| Input | Width and interpretation | Established use |
|---|---|---|
| First argument | Opaque 32-bit value | Not inspected, dereferenced, stored, or forwarded by this function. |
| Second argument | Unsigned 32-bit integer, called the request below | Stored unchanged in the request destination and used to construct the dependency argument. |

All 32-bit request values have a defined arithmetic path in the supplied representation. Only the exact value `1` selects the special path. Other values, including zero, odd values, and values with the high bit set, select the general path.

### Ambient address references

The function obtains addresses through two external references:

- **Configuration-base reference:** supplies bases for accesses at byte offset `4`.
- **Request-base reference:** supplies the base for a 32-bit write at byte offset `4`.

These are descriptive labels, not public symbol names.

The configuration base used for the final write and polling is captured on entry. The two earlier configuration-word reads obtain their addresses through separate evaluations of the configuration-base reference. Consequently, the supplied operation sequence does not establish that every configuration access necessarily uses the same base if the reference can change during execution.

### Output

Normal return produces a 32-bit zero. This does not report which request was accepted, whether other bits match, or whether any higher-level operation succeeded.

### Preconditions

For the described execution to proceed normally:

- Every address actually accessed must support the corresponding 32-bit read or write.
- The selected address plus byte offset `4` must identify a valid accessible word.
- F-5c24bc78359c must be callable with the constructed 32-bit argument.
- F-5c24bc78359c must return normally for the final write and polling to occur.
- A polling read must eventually match the required byte for this function to terminate.

The supplied evidence does not establish address validation, alignment checks, access-fault recovery, or synchronization with concurrent users.

## Behavior

### Address selection and preliminary read

The function first captures the current configuration base for use after the dependency call.

It then reads a 32-bit configuration word at offset `4`, using the configuration-base reference again to determine that read’s base. From this first word, it extracts bits 16–20 and adds one.

That computed value has no influence on the dependency argument or any explicit write:

- For request `1`, the later special path does not use it.
- For every other request, a request-derived value replaces it before use.

The first read is nevertheless part of the established access sequence. If reading this location has device-specific side effects or can fault, its lack of contribution to the final value does not make it behaviorally irrelevant.

### Configuration-word construction

A second 32-bit configuration read occurs at offset `4`, again using the configuration-base reference to obtain the base for that access.

This second read supplies the preserved bits of the dependency argument. The function clears bits 16–23 by applying the mask `0xFF00FFFF`. Bits 0–15 and 24–31 retain their values from this second read.

The request determines what replaces the cleared byte.

**Request exactly `1`:**

- Bits 16–23 remain zero.
- No request-derived field is inserted.
- Bit 22 remains zero as part of the cleared byte.

**Any other request:**

- The request is logically shifted right by one bit.
- One is subtracted using unsigned 32-bit arithmetic.
- Only the low five bits of that result are retained.
- Those five bits occupy bits 16–20 of the dependency argument.
- Bit 22 is set.
- Bits 21 and 23 remain zero.

For the general path, the five-bit field is:

\[
n = ((r \mathbin{\mathrm{>>}} 1)-1)\bmod 32
\]

where \(r\) is the unsigned 32-bit request. The subtraction is a 32-bit unsigned operation before the five-bit reduction.

Thus the byte at bits 16–23 is `0x00` for request `1`, and `0x40 | n` otherwise. The general-path byte ranges from `0x40` through `0x5F`.

Important arithmetic boundaries:

- Request `0` produces field value `31`; unsigned subtraction wraps rather than producing an error.
- Requests `2` and `3` both produce field value `0`.
- Requests `64` and `65` both produce field value `31`.
- Requests `66` and `67` both produce field value `0`.
- Request `0xFFFFFFFF` produces field value `30`.
- High request bits can affect the full recorded request without changing the five-bit field.

The field does not provide range validation. The transformation discards information and deliberately maps many distinct requests to the same dependency argument.

### Request recording and dependency call

After both configuration reads and argument construction, the function obtains the request base and writes the complete, unchanged 32-bit request at byte offset `4`.

That write occurs before invoking F-5c24bc78359c.

The function then calls F-5c24bc78359c with the constructed configuration word as its sole visible argument. The dependency’s returned 32-bit value, rather than the pre-call configuration argument, determines both the final full-word write and the polling target.

The function does not re-read either configuration word to reconstruct the dependency argument after recording the request. Therefore, if the request destination aliases a configuration location, the recorded request does not replace the already constructed argument.

### Final write and polling

After F-5c24bc78359c returns, the function writes its complete 32-bit result at offset `4` from the configuration base captured on entry.

It then reads that same destination at least once. It continues reading until:

\[
(\text{observed word} \mathbin{\&} \texttt{0x00FF0000})
=
(\text{dependency result} \mathbin{\&} \texttt{0x00FF0000})
\]

The comparison covers all eight bits from 16 through 23. It is not limited to the five-bit field or to bit 22.

Differences in bits 0–15 or 24–31 do not prevent completion. Conversely, a difference in any bit from 16 through 23 prevents completion, even if every other bit matches.

There is no explicit delay, retry limit, cancellation check, or secondary dependency call inside this loop. The function does not perform another explicit write while waiting.

After a matching read, the function returns zero.

## Side effects

### Ordered explicit accesses

The supplied operation sequence establishes this order:

1. Capture the configuration base for the later write and polling.
2. Read the preliminary configuration word at offset `4`.
3. Read the configuration word used to construct the dependency argument.
4. Write the unchanged request at offset `4` from the request base.
5. Invoke F-5c24bc78359c.
6. Write the dependency result at offset `4` from the captured configuration base.
7. Read that destination one or more times until the selected byte matches.
8. Return zero.

This is the visible operation order in the supplied representation. It does not establish processor barriers, bus completion rules, atomic transactions, or inter-thread ordering beyond that representation.

### Partial progress and aliasing

There is no explicit rollback:

- If the dependency does not return, the request write has already occurred.
- If polling never matches, both explicit writes have already occurred.
- If a later access faults, the supplied evidence provides no recovery behavior.

No non-aliasing guarantee is shown. If both write destinations identify the same word, the request is written first and the dependency result overwrites it later, assuming normal execution reaches the second write.

The dependency may perform additional accesses or change shared state. Its effects must not be limited to those explicitly visible in this function.

### Ownership

The function contains no explicit allocation, release, ownership transfer, or returned address. The unused first argument is not visibly consumed.

No ownership or lifetime guarantee can be assigned to F-5c24bc78359c from this evidence. In particular, it must not be assumed to preserve global references or keep all referenced storage valid.

A dependency-side change to the configuration-base reference does not redirect the final write: that write uses the base captured before the call. If the dependency invalidates the captured destination, subsequent behavior is not defined by the supplied evidence.

## Dependencies

### F-5c24bc78359c

**Visible interface:** receives one 32-bit configuration word and returns a 32-bit value.

**Established interaction:**

- Called once on the normal path reaching it.
- Called after the request has been written.
- Receives the word derived from the second configuration read.
- Its full result is written to the captured configuration destination.
- Bits 16–23 of its result define the polling target.

**documented_only unknowns:**

- Whether it transforms, validates, or merely returns its argument.
- Whether it reads or writes either addressed word.
- Whether it changes external address references.
- Whether it allocates, releases, or otherwise changes storage lifetime.
- Whether it returns for every argument.
- Whether it signals failures through side effects or its return value.

This function does not visibly interpret a dependency result as an error. Any returned 32-bit pattern is used as the write value and polling target.

## Acceptance examples

These examples are conditional behavioral expectations derived from static evidence. They are not runtime test results.

Unless stated otherwise, assume accessible stable destinations, normal dependency return, and no unrelated changes to the addressed words.

### Special request clears the selected byte

Given request `1` and a second configuration read of `0xA5D3BEEF`:

- The request destination receives `0x00000001`.
- F-5c24bc78359c receives `0xA500BEEF`.
- The final configuration write uses the dependency result, not necessarily `0xA500BEEF`.

The preliminary configuration read does not affect this argument.

### Small general requests share an encoding

Given a second configuration read of `0xA5D3BEEF`, either request `2` or request `3` causes F-5c24bc78359c to receive `0xA540BEEF`.

The recorded requests remain distinct: one stores `0x00000002`, the other stores `0x00000003`.

### Zero wraps to the maximum field

Given request `0` and a second configuration read of `0xA5D3BEEF`:

- The request destination receives zero.
- The five-bit field is `31`.
- The dependency receives `0xA55FBEEF`.

No explicit rejection or error return is associated with this boundary.

### Field wrap does not truncate request recording

Given request `66` and the same second configuration word:

- The request destination receives `0x00000042`.
- The dependency receives `0xA540BEEF`.

This shares an encoding with request `2`, while preserving the full request in the earlier write.

### Polling accepts differences outside the compared byte

Suppose the dependency returns `0x12345678`.

The final full-word write is `0x12345678`. A subsequent read of `0xAB34CDEF` satisfies the polling condition because bits 16–23 equal `0x34`. The function may then return zero despite differences elsewhere.

### Polling rejects a selected-byte mismatch

With dependency result `0x12345678`, a read of `0x12355678` does not satisfy the condition. If every subsequent read retains that mismatch, the function never returns through its normal return path.

### A changed global reference does not redirect the final write

Suppose the entry capture selects destination A, but the configuration-base reference later selects destination B.

The later preliminary reads may use B if the reference has changed by then. Regardless, after the dependency returns, the final write and polling use A. A dependency-side reference change alone does not select a new final destination.

## Uncertainties

The following are **documented_only**:

- Whether the addressed words are ordinary memory, device registers, or another storage class.
- The mechanism and timing of any read or write side effects.
- Whether concurrent actors can change references or destination contents.
- Target alignment requirements, address width, and address-overflow behavior.
- Whether repeated reads have language-level volatile guarantees or hardware-ordering guarantees beyond the supplied representation.
- Whether the dependency preserves any globals, memory contents, or storage lifetimes.
- Why the preliminary read and unused extracted value exist.
- The semantic units or valid operating range of the request.
- Whether readback equality means completion of any underlying operation.
- Fault behavior and any externally imposed interruption or timeout.

None of these unknowns justifies removing the preliminary read, assuming dependency transparency, narrowing the polling mask, or adding validation to the established contract.

## Evidence

**Established static evidence:** the supplied function representation shows two configuration-word reads before a request write; unsigned request-dependent bit construction; one call to F-5c24bc78359c; a full-word result write through an entry-captured base; repeated readback comparison using `0x00FF0000`; and a zero return after a match.

**Inferred intent:** the address-plus-offset layout, selected-byte update, and polling pattern suggest register configuration with readback synchronization. No particular device, protocol, clock, or divisor meaning is established.

**Unknown dependency behavior:** no implementation or behavioral contract for F-5c24bc78359c was supplied.

**Verification status:** static analysis only. No runtime tests have been performed.
