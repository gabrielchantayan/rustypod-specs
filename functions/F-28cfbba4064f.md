<!-- Trusted provenance: F-28cfbba4064f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-28cfbba4064f

## Purpose

F-28cfbba4064f performs a long, ordered sequence of external calls, address calculations, memory reads, and memory writes. Its explicit input is a single 32-bit value. Its effective interface additionally depends on incoming execution state, external-call results, globally accessible storage, and memory at computed addresses.

The established behavior is primarily a state-mutation and call-order contract. The supplied static representation does not establish a reliable application-level purpose, such as initialization, allocation, decoding, or dispatch. Those interpretations must not be used to substitute a simpler operation for the observed effects.

The visible execution path has no explicit conditional branches or loops. If every external call returns normally and every memory access succeeds, execution reaches an invocation whose target address is zero, with three zero-valued arguments. A normal return is shown after that invocation, but successful execution of the zero-address target is not established.

## Interface

### Explicit input and output

- **Input:** one uninterpreted 32-bit value, called the **request** in this contract.
- **Output:** no explicit return value.
- **Primary observable results:** external invocations, writes to computed memory locations, and updates to two global storage locations.

The request is passed as both the first and fourth arguments of the initial external invocation. It is passed again as the third argument of an early later invocation. The visible body does not establish that the request is a pointer, length, enumeration, or owned resource.

### Implicit inputs

The routine also consumes:

- An incoming context base from which a 32-bit word at offset `0x14` is read.
- An incoming selector base from which bytes at offset `1` are read.
- An incoming 32-bit value used in the first direct word store.
- Incoming execution state used to form the initial call target.
- Primary and secondary result components of external invocations.
- A byte stored at absolute address `1`.
- Two global storage locations, designated **G-A** and **G-B**.
- Other memory reached through computed addresses.

These are descriptive interface roles, not established structure types or calling-convention declarations.

Some secondary results appear as values available after calls without a complete declared output interface. Their provenance and lifetime are therefore `documented_only`. An implementation cannot safely treat them as initialized constants or reconstruct them from the explicit request alone.

### Widths and arithmetic

The visible representation distinguishes:

| Quantity | Width |
|---|---:|
| Explicit request | 32 bits |
| Ordinary arithmetic values and addresses | 32 bits |
| Byte loads and stores | 8 bits |
| Halfword loads and stores | 16 bits |
| Word loads and stores | 32 bits |
| Combined external results | 64 bits, split into two 32-bit components |

For address and bit-pattern interpretation, arithmetic is described using 32-bit truncation. This captures the displayed machine-width effects; it does not establish portable signed-overflow behavior.

Unsigned right shifts insert zeros. Byte stores retain only the low eight bits of their supplied value. In particular, masking to 30 bits and shifting right by 20 produces a value up to `1023`; storing that result as a byte discards its upper two bits.

Several left shifts by 31 retain only the original least significant bit, placing it in bit 31. They do not preserve the other source bits.

### Preconditions and ownership

Normal progress requires every reached call target to be executable and every reached memory access to be valid for its width and access mode. No explicit validation establishes those conditions.

No allocation, release, ownership transfer, or resource-lifetime guarantee can be attributed to an unknown dependency. A temporary stack address is passed four times in one invocation. Its writable extent, initialization requirements, and permitted retention are unknown.

## Behavior

### Observable ordering rules

The visible direct operations occur in the order described below, interleaved with numerous external invocations.

Those invocations are observable boundaries. They must not be assumed to preserve global storage, memory selectors, pointed-to objects, or secondary result state. Consequently:

- A repeated load is a new observation, not necessarily a duplicate of an earlier value.
- Repeated zero stores remain distinct effects.
- A locally retained value is different from a later reload of its original storage.
- A store followed by an external invocation and another store is not necessarily equivalent to a single final store.
- Overlapping computed addresses may make apparently unrelated operations affect one another.

For readability, a **selector write based on an address** means an 8-bit read at that address plus `1`, followed by an 8-bit write at the resulting unsigned byte value plus `0x11`. The selected destination therefore lies between `0x11` and `0x110`, inclusive. This notation describes an access relationship, not a helper function.

A **scaled word slot** means a 32-bit location computed from a supplied value using the stated scale and offset. It does not imply an allocated table.

### 1. Entry and early publication

Before the first external invocation, the routine reads a 32-bit word at context-base offset `0x14`, shifts it right by 19, and supplies the result as the second argument. The other visible arguments are the request, zero, and the request, in that order around it.

After several invocations, a returned value determines a word-store destination: four times that value plus `0x14`. The incoming implicit 32-bit value is stored there. The corresponding four-times-scaled value is then passed to another dependency.

A later returned 32-bit word supplies three distinct derived values:

- Four times the word, with 32-bit truncation.
- Its low 30 bits shifted right by 17.
- Its low 30 bits shifted right by 7.

The low byte of the four-times-scaled value is written using the incoming selector base. After intervening calls, the selector byte is read again and used for another write. This second value is the original returned word’s low 30 bits shifted right by 20, then truncated to eight bits.

The intervening call receives the four-times-scaled value twice and the request as its third argument. A subsequent call receives a later result four times, followed by the two retained masked-and-shifted values and a control-derived value.

A combined 64-bit result is then split into low and high 32-bit components. The routine retains both and performs a selector write of zero based on the earlier right-by-7 derivation. The high result component is subsequently passed twice to a dependency, alongside an earlier retained result.

### 2. Repeated clearing and scalar reads

Further calls and selector writes use the retained low component and later call-produced state.

Established effects include:

- A second selector write of zero based on the same earlier right-by-7 derivation.
- Passing the retained low component shifted right by 28 to a dependency.
- Two separate selector writes of zero based on that component shifted right by 9.
- An 8-bit read at twice that component, followed by passing the zero-extended byte four times to a dependency.
- A selector write of zero using the byte at absolute address `1`.
- Passing a later secondary result shifted right by 18 to another dependency.

A left-shift-by-31 argument in this portion is applied to a value already multiplied by four. Under the displayed 32-bit arithmetic, that argument is zero. Another such argument is derived from the loaded byte and therefore depends only on that byte’s bit 0.

A later combined result provides a new retained 32-bit base. The routine derives its right-by-19 value, performs a selector clear based on the unshifted base, and reads a 32-bit word at:

- The base itself when bit 19 of that base is zero.
- The base plus `0x80000000`, modulo 32 bits, when bit 19 is one.

The loaded word is passed four times to a dependency. A selector clear based on the base shifted right by 9 follows intervening calls.

Next, another secondary-result base is used for a selector clear, an 8-bit read at offset `0x0e`, and a later selector clear based on that base shifted right by 9. The byte read is supplied four times to a dependency.

Another combined result supplies a retained value used for a selector clear based on itself and, later, for a clear based on itself multiplied by `0x1000`.

### 3. Temporary address and halfword consumption

A secondary result is passed in three argument positions, while its right-by-19 derivation occupies another position. It also supplies the base for a selector clear.

The routine then passes the same temporary stack address in four argument positions. A subsequent combined result supplies another retained base. Before consuming it further, the routine clears a selector whose source address is either `1` or `0x80000001`, according to the low bit of an earlier right-by-19 derivation.

The next scalar observations are:

1. An 8-bit read at the retained base plus a call-produced offset.
2. A 16-bit read at the sum of that zero-extended byte and a later secondary result shifted right by 19.

Each loaded scalar is passed four times to an external invocation.

After the halfword consumption, a combined result supplies a base-like low component and a high component used as a word-slot index. The routine stores the retained right-by-19 value at four times the high component plus `0x14`. Following more calls, it stores the same retained value in another four-times-scaled word slot.

### 4. Field writes and scaled-word propagation

The retained low component receives two byte writes at offset `0x0d`:

1. The low byte of twice that component.
2. The constant `4`, after intervening calls.

Between those writes, a word read at four times a newly returned value plus `0x44` is passed to a dependency. After the second write, another dependency receives a word read at the returned value’s low 30 bits shifted right by 20, plus `0x44`. The second address is not scaled by four.

Subsequent direct word writes include:

- At `256` times a returned value plus `0x14`: four times an earlier returned word.
- At four times another result plus `0x14`: the earlier result’s low 26 bits shifted right by 13.
- At `16` times a result plus `0x14`: four times that same result.

The last pattern recurs repeatedly. Some of its four-times-scaled values are retained and later written to ordinary four-times-scaled slots. Other retained values are shifted left by 24 or 22 and passed to dependencies.

After several such publications, the routine writes zero at `16` times a retained index plus `0x24`, then reads a word at that same scaled base plus `0x74`. The read word becomes a new base used later.

A subsequent returned word is shifted right by 20. That derivation is stored at new-base offset `0x24`. A word at new-base offset `0x74` is read and passed to a dependency.

The routine then performs two selector writes based on the new base: first a truncated right-by-20 derivation of an earlier retained word, then zero, with calls between them.

### 5. Three byte observations at offset `0x15`

Three separate 8-bit reads occur at offset `0x15` from three call-produced bases. They are separate observations and need not refer to the same object.

Between these reads, selector clears repeatedly use a source address derived from `1024` times the retained right-by-20 value. The third loaded byte is eventually written as a zero-extended 32-bit value into a four-times-scaled slot at offset `0x14`.

Additional word publications retain an index, its four-times-scaled form, and an 11-bit extraction obtained by masking a secondary result to 29 bits and shifting right by 18. That extraction shifted left by 13 is passed to a dependency and stored in a word slot.

Later operations include selector writes of scaled low bytes, repeated clears, and word publications of dependency results. A separately retained secondary result undergoes successive left shifts by 3, then 2, then 2; its final published form corresponds to a left shift by 7 under 32-bit truncation.

### 6. Global clearing and zero-derived accesses

The routine explicitly writes zero to G-A. Numerous external calls occur before G-A is subsequently read. Its later value is therefore not established as zero.

A separate locally retained word is assigned zero and remains zero through the following calls. Several later calculations consequently reduce to fixed values:

- A right shift of that local word is zero.
- Its four-times-scaled value is zero.
- Selector reads based on either calculation occur at absolute address `1`.
- Later extractions derived exclusively from those locally retained zeros are also zero.

This region includes a returned value passed twice to a separately identified dependency, an 8-bit read at a returned base plus `1`, and an 8-bit read at the zero-extended first byte plus `1`.

A later byte load uses a base derived entirely from the retained zero calculations, so its address is `1`. The loaded byte is subsequently published as a 32-bit value in a scaled word slot.

Other selector writes in this region receive masked right-shifted values that are zero by their established bounds. Such writes must still occur; their zero value does not remove their observable effect.

### 7. Late object-like fields and dependent word reads

A byte result from a dependency is multiplied by four and stored through a selector. Later operations include:

- A selector clear based on another retained shifted value.
- An 8-bit zero store at a scaled address plus `9`.
- A byte store at a call-produced base plus `9`, using that base’s low byte.
- A 32-bit read at that base plus `0x34`, supplied to a dependency together with both components of a combined result.

A secondary-result value then receives an associated publication at four times itself plus `0x14`. Separately, a word at the unscaled secondary-result value plus `0x14` is read. These addresses are distinct unless the actual values make them coincide.

Further dependent reads use:

- A previously read word shifted right by 22, plus `0x14`.
- Another retained word plus `0x14`.
- A secondary result masked to 29 bits, shifted right by 19, plus `0x14`.

G-A is read again in this later sequence. One retained value is shifted right by 23 and then shifted right by 23 again. The second result is necessarily zero for any 32-bit original value.

### 8. Final publication and terminal invocation

A later returned word is written to G-B and passed to a dependency. A word is then read at a retained base plus `0x74`; its low byte is stored at that base plus `1`.

Similar word-to-byte updates occur at other four-times-scaled bases. At one base, the word at offset `0x74` is read twice across intervening external calls. The two observations are not required to match.

A later `0x74` word is retained and published into additional scaled word slots. Another direct store writes a scaled result at an earlier retained value, with its bottom two bits cleared, plus `0x14`.

The routine subsequently publishes zero and components of combined results, reloads G-A, and reads a word at the reloaded value plus `0x14`. Portions of that word are written through two different selectors.

The final direct effects are:

1. Invoke another external dependency and obtain an 8-bit result.
2. Store zero through a call-produced byte pointer.
3. Update G-B using that byte as the low eight bits and a separately referenced three-byte quantity as the upper 24 bits.
4. Store a zero halfword at an address obtained by masking a retained word to 30 bits and shifting right by 7.
5. Invoke address zero with three zero arguments.

The earlier G-B publication and final G-B update are distinct. Neither implies a supervisor operation merely because of a source-level label.

## Side effects

The established direct side effects comprise 8-, 16-, and 32-bit memory accesses; updates to G-A and G-B; and external invocations.

No atomicity, synchronization, rollback, or transactional completion is visible. If a call fails to return or an access faults, earlier effects may already have occurred. No explicit error result or recovery path is established.

The warning about overlapping global storage prevents treating all global and low-address locations as disjoint. Access width and order matter where overlap occurs.

## Dependencies

Only the identity of F-28cfbba4064f is supplied. External identities are unresolved. The following opaque IDs are contract-local dependency references, not claims of recovered function identity:

| Opaque ID | Established dependency role |
|---|---|
| F-000000000001 | Initial execution-state-relative invocation. |
| F-000000000002 | Ordered family of unresolved indirect invocations, including scalar consumers and primary, secondary, or combined-result producers. This is a family label, not one shared callee. |
| F-000000000003 | Separately identified invocation receiving the same returned word twice. |
| F-000000000004 | Late invocation producing the final byte used in G-B. |
| F-000000000005 | Terminal invocation at address zero with three zero arguments. |

None has an established preservation, ownership, allocation, release, error, or termination contract. Exact identities, calling conventions, and complete per-call traces remain `documented_only`.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

- **Entry extraction:** If the context word is `0xFFF80000`, the second argument of F-000000000001 is `0x1FFF`. The request appears unchanged in its first and fourth positions.
- **Byte truncation:** If an early returned word is `0x3FF00000`, its masked right-by-20 value is `1023`; the corresponding byte store writes `0xFF`.
- **Selector reload:** If the selector byte changes from `2` to `7` during an intervening dependency, successive selector writes target `0x13` and `0x18`, not one cached destination.
- **High-bit address selection:** For a retained base with bit 19 set, the described word read adds `0x80000000` modulo 32 bits. It does not add the whole right-by-19 value.
- **Global mutation:** If a dependency changes G-A after its explicit clearing, a later reload observes the changed value. Replacing that reload with zero violates the contract.
- **Repeated field read:** If a dependency changes a word at offset `0x74`, the later byte update uses the new word’s low byte.
- **Repeated shift bound:** A 32-bit word shifted right by 23 twice produces zero, regardless of whether its original storage changes later.
- **Terminal behavior:** If execution reaches the end, the final invocation targets address zero. Silently omitting it to obtain a normal return is not equivalent.

## Uncertainties

The following remain `documented_only`:

- Application-level purpose and structure meanings.
- Reliable entry calling convention and secondary-result interfaces.
- Exact identities and behavior of unresolved dependencies.
- Target byte order, alignment rules, and fault behavior.
- Executability of address zero.
- Full storage width and overlap relationships of globals.
- Memory ownership, allocation boundaries, and stack-address retention.
- Whether the supplied static representation accurately captures every call boundary and operand.

No successful execution or normal completion is claimed.

## Evidence

**Established static evidence:** the supplied body shows one explicit 32-bit input, no explicit return value, a linear visible path, mixed-width memory operations, repeated selector reloads, scaled-address word publications, global accesses separated by external calls, combined-result splitting, and a final zero-address invocation.

**Inferred intent:** the repeated publications may coordinate state across external operations. No specific protocol or object model is established.

**Verification status:** no runtime tests have been performed. This contract describes static evidence and explicitly bounded consequences, not experimentally confirmed behavior.
