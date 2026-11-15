<!-- Trusted provenance: F-d28e9df0cab5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d28e9df0cab5

## Purpose

F-d28e9df0cab5 performs an ordered sequence of indirect calls, computed-address memory transfers, bit-field extraction, and shared-state writes. Its directly represented accesses use 8-bit, 16-bit, and 32-bit widths. Some dependencies supply results represented as 64 bits.

No higher-level domain purpose is established. Repeated scaling and offsets suggest encoded references or structured storage, but do not establish object types, collection semantics, initialization, or a valid operating-system operation.

This contract describes the routine’s directly represented behavior. Dependencies may introduce additional reads, writes, allocation, release, failure, or nontermination. A normal return is represented only after a terminal call through address zero returns.

## Interface

**Declared interface.** No explicit parameters and no explicit return value are represented. There is no represented status or error result.

**Implicit inputs.** Execution depends on:

- Incoming machine state used to form the first indirect-call target.
- An incoming address base and an incoming 32-bit payload.
- Dependency results, including 8-bit, 32-bit, and represented 64-bit values.
- Additional register-delivered values whose production and calling convention are unresolved.
- A shared 32-bit source location, observed twice.
- A separate shared 32-bit output location.
- Repeated observations of the byte at absolute address 1.
- A represented three-byte source beginning at absolute address 9.
- All memory reached through computed addresses.

The absence of explicit parameters does not eliminate caller-state requirements.

**Widths and address notation.** A byte is 8 bits, a halfword is 16 bits, and a word is 32 bits. For descriptive purposes:

- The *scaled base* of an index is four times that index.
- Its *word slot* is that scaled base plus 20.
- A represented 64-bit result has low and high 32-bit halves.
- Byte stores retain the low eight bits of their source value.
- Unsigned right shifts are logical.
- All offsets and shift counts below are decimal.

These address formulas do not establish an array, a valid allocation, or alignment. Unsigned 32-bit expressions have the represented unsigned width; signed-looking multiplication, left shifts, and pointer arithmetic must not all be silently assigned one universal wrapping rule. Their exact machine interpretation remains **documented_only** where the supplied representation does not settle it.

**Execution preconditions.** Every reached indirect target must be callable, and every reached memory access must be valid for its width. This includes low addresses, potentially unaligned addresses, and addresses derived from hidden inputs. No checks enforcing those conditions are represented. Memory or callable targets required after a dependency must remain usable; no such preservation is established for unknown dependencies.

## Behavior

### Ordering and reachability

The body contains no explicit branch or loop. Its directly represented operations follow one straight-line order, interspersed with many indirect calls.

A later operation is reached only if earlier calls return and earlier accesses complete. An unused call result does not make the call removable. A call with no explicit arguments is not established to have no implicit inputs.

The descriptive values below identify captured values and relationships, not implementation storage requirements. Direct-access ordering is the ordering represented by the supplied body, not a claim about hardware transactions, atomicity, or synchronization.

### Initial transfers and captured source values

After five indirect calls, a returned 32-bit index selects a word slot receiving the incoming payload. The routine then reads a word at the incoming base plus 20, capturing an **initial link**, before calling a dependency with the first index multiplied by four.

Four further calls produce a **linked value**. The initial link, logically shifted right by one, supplies a **half-link index**. Its word slot receives the linked value shifted left by two. A subsequent word read uses the *unshifted initial link plus 20*, capturing a **linked word**.

The half-link index is passed unscaled to a dependency. A producer result is then forwarded to another dependency. After additional calls, another returned index selects a word slot receiving the same linked value shifted left by two.

The routine next captures an **early shared word** from the shared source location. This observation precedes the following dependency call, which receives the most recent index multiplied by four.

No disjointness is established among these locations. A preceding store may affect a later read when their addresses overlap.

### Extracted fields and zero-valued transfers

After further calls, the linked word shifted right by 22 becomes the **linked field**. The early shared word shifted right by 23 becomes the **early shared field**.

A register-delivered index selects a word slot receiving the linked field. The following dependency receives four arguments, in order: that index, the same index again, the early shared field, and the linked field.

A producer result is forwarded to another dependency. Between those calls, the early shared field is shifted right by 23 again. Because the first shift of an unsigned 32-bit word leaves at most nine significant bits, this second shift is zero.

After another call sequence, a returned index selects a second word slot receiving the linked field. The next call receives that index multiplied by four.

Following further calls, a second register-delivered index selects a word slot receiving a 32-bit zero. The next dependency receives this index unscaled. A producer-and-forwarder pair and additional calls precede another word-slot store of the early shared field shifted right by 23, also zero.

Immediately after that zero store, the routine reads a **selector word** from:

\[
((\text{second register-delivered index}\mathbin{\&}(2^{29}-1))\gg19)+20.
\]

The address ranges from 20 through 1043 and need not be word-aligned. The read precedes the next dependency call, whose argument is the most recently returned index multiplied by four.

### Byte-selected stores and retained control state

Further calls precede extraction of the **selector field**, the selector word shifted right by 22. The routine reads a byte at absolute address 1 and stores the low byte of the selector field at the selected byte’s numeric value plus 17. The read address is 1 because its computed base is the zero-valued double shift described above.

The following dependency receives three copies of another register-delivered value and a fourth argument of zero. After another call, a dependency is invoked with zero. Further calls produce an index.

A byte is then read at four times the selector field plus 1. A zero byte is stored at that byte’s value plus 17. The following call receives the newly produced index shifted left by two.

After further calls, two copies of another returned index shifted left by two are passed to a dependency. A later 32-bit result is passed to another dependency. After one more call, the byte at absolute address 1 is read again, and zero is stored at its value plus 17.

At this point, another register-delivered value is captured as the **retained control value** and passed three times to the next dependency. A later producer result is forwarded together with the earlier 32-bit result shifted right by 23.

Following additional calls, the routine reads address 1 again, capturing a **retained byte base**, and stores zero at that base plus 17. The next call receives the latest returned index shifted left by two.

A further call sequence passes two copies of another returned index shifted left by two to a dependency. The next producing call supplies a **shared-state result**. The routine directly stores that complete 32-bit result to the shared output location, then passes the captured result to the next dependency.

The distinct reads at address 1 must not be merged. Dependencies may change the byte between observations. Conversely, later uses of the retained byte base use the captured number, not a fresh read of address 1.

### Word-to-byte copies and retained payload

After another call, a word is read at retained byte base plus 116. Its low byte is stored at retained byte base plus 1. The next dependency receives the retained control value shifted left by 11.

A producing call then supplies a represented 64-bit result. The following dependency receives its low 32-bit half, its high 32-bit half, and the captured shared-state result shifted right by 23, in that order. The third argument uses the captured result, not a reread of the shared output.

After more calls, a returned index supplies a scaled base. A word read at that base plus 116 supplies the low byte stored at that base plus 1. The base is passed to the next dependency.

Further calls produce a **retained index** and its scaled base. The next word-to-byte copy still uses the *previous* base, not this newly produced base. It performs a fresh word read at offset 116; it does not reuse the earlier word.

The following dependency receives four arguments:

- Retained control value shifted left by 12.
- Retained index shifted left by two.
- The same shifted index again.
- Retained index shifted left by 18.

Producer results are subsequently forwarded through additional dependencies.

Another returned index supplies a new scaled base. A word read at that base plus 116 is captured as the **retained payload**. Its low byte is stored at that base plus 1, and the base is passed to the next dependency.

After further calls, the full retained payload is stored at a newly returned index multiplied by 16, plus 20. The next dependency receives the retained index’s captured scaled base shifted right by two and then left by 16. This expression must not be replaced with an unbounded-arithmetic simplification: the captured scaled base is a 32-bit value that may already have lost high bits.

Following more result forwarding and calls, the retained payload is stored again, now to an ordinary word slot. The next dependency receives that slot’s scaled index.

A later result shifted left by two is stored at:

\[
(\text{retained index's scaled base}\mathbin{\&}(2^{32}-4))+20.
\]

Two copies of that shifted result are passed to the following dependency. After additional calls, another word slot receives a 32-bit zero, and its scaled index is passed onward.

### Late source observation and final transfers

A represented 64-bit result supplies two values: its low half shifted left by two becomes the **final transferred value**, and its high half selects a word slot receiving that value.

The routine then rereads the shared source, capturing a **late shared word**. The next dependency receives the final transferred value, the high-half index, and two further copies of the final transferred value.

Following additional calls, another word slot receives the final transferred value. A word is then read at late shared word plus 20, capturing the **late payload**, before the next dependency receives the newly selected slot’s scaled index.

Further calls produce another represented 64-bit result. Its low half becomes the **tail selector**. A byte is read at:

\[
(\text{retained payload}\gg23)+1.
\]

The low byte of the late payload is stored at that byte’s value plus 17. The following dependency receives three copies of the latest 64-bit result’s high half and the tail selector shifted left by two.

After another producer-and-forwarder sequence and further calls, a byte is read at:

\[
((\text{tail selector}\mathbin{\&}(2^{30}-1))\gg21)+1.
\]

The low byte of the late payload shifted right by 22 is stored at that byte’s value plus 17. The next call receives a newly produced index shifted left by two.

Further calls eventually pass two copies of another returned index shifted left by two to a dependency. Subsequent calls produce a value, forward it, and then produce an 8-bit **terminal byte result**.

### Terminal stores and return dependency

The routine first stores a zero byte through a final register-delivered pointer.

It then directly stores to the shared output location a second time. This 32-bit value is constructed from the terminal byte result as its low eight bits and the represented three-byte source beginning at absolute address 9 as its upper 24 bits. The three-byte source is used after the preceding pointer store; aliasing with that store is not excluded.

This second direct store is not required to differ from the first direct store. Nor does it establish a persistent final value: intervening dependencies may already have changed the location, and subsequent operations may change it again.

Next, a zero halfword is stored at:

\[
(\text{tail selector}\mathbin{\&}(2^{30}-1))\gg7.
\]

This address ranges from zero through 8,388,607 and may be unaligned. It is not established to be disjoint from the shared output.

Finally, dependency F-000000000003 is called through address zero with three explicit zero arguments. A normal return from F-d28e9df0cab5 is represented only after that call returns.

## Side effects

The routine directly performs byte, halfword, and word stores. No bounds, null, alignment, or access-permission checks are represented.

Byte-selected destinations range from 17 through 272 because their immediate selectors are unsigned bytes. This range does not establish valid memory.

The routine directly stores to the shared output location twice. The first direct store uses the shared-state result; the second constructs a word from the terminal byte result and the three-byte source. Neither inequality between those two values nor persistence after either store is established. Calls may modify the location between or after the stores, and other direct stores may alias it. The terminal halfword store and zero-address call occur after the second direct store.

Writes are incremental. No rollback is represented if a later access or call fails.

Ownership is **documented_only**. No direct allocation or release operation is identifiable, but dependencies are not established to avoid allocation, release, retention, or invalidation. The representation does not establish borrowed, owned, transferred, or lifetime-stable pointers.

No synchronization or atomic-access guarantee is established.

## Dependencies

These opaque IDs designate dependency categories, not resolved external symbol identities.

| ID | Scope | Established interface facts |
|---|---|---|
| F-000000000001 | Initial incoming-state-derived call | Its target basis comes from incoming machine state with an added displacement of 4094. No explicit arguments or consumed result are represented. |
| F-000000000002 | Intermediate computed-target calls | An unresolved family of call sites with zero through four explicit arguments. Consumed results include bytes, words, and represented 64-bit pairs. |
| F-000000000003 | Terminal zero-address call | Three explicit zero arguments; no consumed result. |

Grouping intermediate calls does not authorize merging, removing, or reordering them. Their individual identities and contracts remain unresolved.

No dependency is assumed to preserve globals, memory, register-delivered state, ownership, or normal control flow. Captured values forwarded after a call are distinguished from fresh memory observations; their use does not imply that the original memory remains unchanged.

## Acceptance examples

These are conditional static criteria, not executed tests. Each presupposes that preceding operations complete and that any relevant arithmetic and memory interpretation is valid.

- **Double shift:** Every unsigned 32-bit early shared word shifted right by 23 twice yields zero. The corresponding later word-slot store therefore stores zero.

- **Byte truncation:** A selector word of `0xFFC00000` yields selector field 1023. Its byte-selected store writes 255, not 1023.

- **Fresh selector observations:** If address 1 supplies 40 at the first selector observation and 70 at a later one, the corresponding destinations are 57 and 87. Reusing the first observation violates the represented behavior.

- **Fresh word observation:** If a dependency changes the word at the first retained scaled base plus 116 before the repeated copy, the repeated copy uses the new word’s low byte.

- **Late shared source:** If dependencies change the shared source between its early and late observations, late addressing uses the late captured word. Earlier field calculations still use the early captured word.

- **64-bit halves:** A represented result of `0x1234567889ABCDEF` supplies low half `0x89ABCDEF` before high half `0x12345678` where low/high forwarding is specified.

- **Equal shared-output stores:** If the first direct shared-output value is `0x12345678`, the terminal byte result is `0x78`, and the represented three-byte source supplies upper field `0x123456`, both direct stores write `0x12345678`. Equality is permitted.

- **Intervening output mutation:** If a dependency changes the shared output after its first direct store, the second direct store still uses its specified construction. It does not depend on the location retaining the first value.

- **No persistence guarantee:** If the terminal halfword store overlaps the shared output, or the terminal call changes that location, memory after those operations may differ from the second direct store’s value. Such a difference does not contradict the two-store contract.

- **Failure and return:** If the final register-delivered pointer is not writable, subsequent stores are not guaranteed to occur. If address zero is not callable, completed earlier stores do not establish normal return.

## Uncertainties

**Established static evidence:** Explicit access widths, shifts, masks, captured-value relationships, repeated reads, ordered direct stores, two direct shared-output stores, and the terminal zero-address call.

**Inferred intent:** Repeated scaling and offsets may encode structured references. This does not justify assigning object layouts, ownership rules, or domain semantics.

**Documented_only unknowns:**

- Actual calling convention, hidden inputs, and register-result provenance.
- Individual intermediate dependency identities and effects.
- Whether represented 64-bit results are scalar results or paired machine outputs.
- Exact machine behavior of signed-looking arithmetic and left shifts.
- Byte order, alignment requirements, and physical access granularity.
- Validity and mapping of low addresses.
- Aliasing among computed locations and overlapping shared representations.
- Whether shared representations model ordinary memory, special state, or an imperfect recovered view.
- Dependency failure, nonlocal exit, memory invalidation, and synchronization behavior.
- Memory contents after the terminal call, including the shared output.

The second shared-output construction establishes the value of that direct store, not an unconditional final-memory postcondition.

## Evidence

This contract uses only the supplied static body for F-d28e9df0cab5. Embedded comments, prior prose, recovered labels, and type annotations are not independent authority for higher-level semantics.

The shared-output qualification follows from the represented order: first a complete 32-bit result is stored; later a constructed 32-bit value is stored; a computed halfword store and an unresolved call follow. No comparison constrains the two stored values to differ, no disjointness excludes later overlap, and no dependency contract guarantees preservation.

No runtime execution or tests have been performed. Acceptance examples state conditional consequences of the static representation, not demonstrated successful executions.
