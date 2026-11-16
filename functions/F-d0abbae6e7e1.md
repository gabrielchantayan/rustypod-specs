<!-- Trusted provenance: F-d0abbae6e7e1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d0abbae6e7e1

## Purpose

F-d0abbae6e7e1 performs an externally stateful sequence of indirect calls and computed-address memory accesses. Its established contract concerns those effects, their ordering, and the values retained between them. The supplied static representation does not establish an application-level purpose, object type, protocol, or ownership policy.

The routine has no represented conditional branches or loops. Progress nevertheless depends on every reached call returning and every reached access succeeding. A dependency may change memory used later, fail, or prevent further execution.

An implementation must not replace the sequence with its apparent final memory contents. Repeated reads, intervening calls, overlapping destinations, and intermediate writes can make such replacement observably different.

All behavior below is derived from static evidence. No runtime tests have been performed.

## Interface

### Inputs and result

The represented interface accepts four 32-bit words and declares no return value.

| Input | Explicit use |
|---|---|
| First word | No use represented in the body. |
| Second word | No use represented in the body. |
| Third word | Logically shifted right by one; adding one gives an unsigned-byte read address. |
| Fourth word | Passed to a dependency in full and as its upper ten bits. Those upper ten bits also determine a later unsigned-byte read address. |

Absence of an explicit use does not establish machine-level irrelevance. Incompletely described calls may observe incoming register state.

Additional incoming state participates in the represented behavior:

- An incoming link-state value, increased by `0xffe`, determines the first call target.
- An incoming register payload supplies the first explicit word store.
- Secondary result-register values from dependencies supply indices, arguments, and a final byte-store pointer.
- Several dependencies produce represented 64-bit values whose low and high 32-bit halves are consumed separately.

The four declared inputs therefore do not constitute a complete machine-level invocation contract.

### Widths and addressing

In this document:

- **Byte** means an 8-bit access. Selector reads are unsigned.
- **Halfword** means a 16-bit access.
- **Word** means a 32-bit access.
- **Global A** is the shared word explicitly read twice.
- **Global B** is the shared word explicitly assigned twice.
- **Scaled slot** means an address formed as four times an index plus `0x14`, unless a different scale is stated.

These are observational descriptions, not established object layouts. A scaled slot and an unscaled address ending in `+0x14` are different calculations.

Byte stores retain the low eight bits of the supplied value. Logical right shifts act on 32-bit unsigned values. A represented 64-bit result is split numerically into a low word and a high word; that split alone establishes no memory byte order.

Some left shifts and multiplications have signed operands in the supplied representation. Their overflow, truncation, and address-wrap behavior require underlying instruction evidence. References below to truncating 32-bit shifts are explicitly conditional, not a general assertion of signed-language semantics.

### Preconditions and errors

Normal completion requires valid executable call targets, valid reached memory accesses, and returning dependencies. No bounds, null-pointer, alignment, or permission checks are represented.

There is no represented success code, recoverable error result, or rollback path. Fault behavior is environmental. The final return is reached only if the call through address zero returns.

## Behavior

### Execution and retained state

The following phases specify the explicit memory-effect sequence and material argument relationships. Unresolved call sequences between effects remain part of the behavior even when intermediate results are not consumed.

A retained value means the previously obtained value, not a reread of its original storage. Conversely, a specified fresh read must occur at its stated position; it cannot be replaced with a cached value merely because the address repeats.

### 1. Incoming payload and input-derived selectors

Five initial indirect calls precede the first explicit store. The fifth supplies an index. The incoming register payload is written as a word to that index’s scaled slot. The index multiplied by four is then passed to the next dependency.

After further calls, a byte-result dependency supplies the first byte-store value. The routine reads a selector byte at `(third input >> 1) + 1`, then writes the dependency result shifted left by two, truncated to a byte, at `selector + 0x11`. For a 32-bit third input, the selector-read address ranges from `1` through `0x80000000`.

A dependency next receives the fourth input’s upper ten bits, the full fourth input, and the same upper-ten-bit value again. Another dependency result is subsequently forwarded with that upper-ten-bit value repeated twice.

Following more calls, a fresh selector is read at `(fourth input >> 22) + 1`, and zero is written at `selector + 0x11`. This selector-read address ranges from `1` through `0x400`.

Every selector-derived destination in this phase lies numerically between `0x11` and `0x110`. It is not established as a field relative to either input.

### 2. Scaled clearing and retained-base access

A dependency result shifted left by two is passed onward. Later, a separate result shifted left by two is retained as a scaled quantity and passed to another dependency. A further returned word is forwarded, and a subsequent result becomes a retained base.

The routine clears one byte at `0x2000 × retained scaled quantity + 9`. The earlier left shift is already included in that quantity; the address is not merely `0x2000` times its original unshifted result.

A dependency receives the retained base, the forwarded word shifted right by 22, and that word unshifted. Additional calls produce a represented 64-bit result.

The routine then writes the retained base’s low byte at `retained base + 9`. The following dependency receives:

- The 64-bit result’s low word shifted left by two.
- Its high word.
- A freshly read word at `retained base + 0x34`.

The offset-`0x34` read follows the byte store. Aliasing or memory-specific effects can therefore matter.

### 3. Propagation through scaled and unscaled addresses

After further calls, a secondary result selects a scaled slot that receives the retained base. The routine next reads a word at that same index **without scaling**, plus `0x14`, and passes the index to a dependency.

A later returned index selects another scaled slot, again receiving the retained base. The earlier unscaled read’s upper ten bits then select another word read at `upper ten bits + 0x14`.

After additional calls, the newly loaded word shifted right by one selects a scaled slot. That slot receives a dependency result shifted left by two. The routine then reads a word at the newly loaded word’s **unshifted** value plus `0x14`.

Following another dependency sequence, a further returned index selects a scaled slot receiving the same retained shifted result.

No common object layout is established by these accesses. Their differing scales and shifts are part of the contract.

### 4. First Global A read and zero propagation

The first explicit read of Global A occurs after the preceding scaled-slot store.

Following more calls, the routine takes the most recently loaded ordinary word’s upper ten bits and logically shifts the retained Global A value right by 23. It stores the upper-ten-bit value in a scaled slot selected by a secondary result.

A dependency receives that secondary result twice, the once-shifted Global A value, and the upper-ten-bit value. Further dependencies precede another scaled-slot store of the same upper-ten-bit value.

A different secondary result then selects a scaled slot cleared to zero. More calls precede a further scaled-slot store of the retained Global A value shifted right by 23 **a second time**.

For every 32-bit unsigned Global A value, the first shift leaves at most nine bits, so the second shift yields zero. A separately retained copy of this twice-shifted value is also zero and later selects a byte read at address `1`.

These zero values follow from the represented unsigned arithmetic, not from any assumed initial contents of Global A.

### 5. Selector writes and repeated address-1 reads

The routine reads a word at:

`((secondary result & 0x1fffffff) >> 19) + 0x14`.

After intervening calls, it takes that word’s upper ten bits. It reads the selector at address `1` and writes the low byte of those upper ten bits at `selector + 0x11`.

A dependency then receives a secondary result three times and zero as its fourth explicit argument. Following another call, a dependency receives zero. Additional calls precede a selector read at `4 × upper-ten-bit value + 1`; its selected destination at `selector + 0x11` is cleared.

Subsequent calls include a shifted result passed twice to one dependency. Another result is retained and forwarded. After further calls, the routine freshly reads the selector at address `1` and clears its selected destination.

A secondary result is retained and passed three times to a dependency. Later, a dependency receives another returned value alongside the previously retained returned word shifted right by 23.

Following more calls, the routine reads address `1` again, retains that selector byte, and clears the byte at `retained selector + 0x11`.

The address-`1` reads are distinct observations. Dependencies may change that byte between them; they need not select the same destination.

### 6. First Global B assignment and repeated metadata copies

Further calls include a dependency receiving the same shifted result twice. A subsequent returned word is assigned to Global B and passed to another dependency.

After an additional call, the routine reads a word at `retained selector + 0x74` and writes its low byte at `retained selector + 1`.

A dependency receives the earlier retained secondary result shifted left by 11. Another supplies a represented 64-bit result. Its low word, high word, and the word previously assigned to Global B shifted right by 23 are passed onward. This third argument uses the retained assigned value, not a fresh Global B read.

A later result multiplied by four establishes a scaled base. The routine reads a word at `scaled base + 0x74` and copies its low byte to `scaled base + 1`.

After more calls, another result shifted left by two is retained. The same offset-`0x74` word is freshly read, and its low byte is again written at `scaled base + 1`. Dependencies between these copies may change the source word.

A subsequent dependency receives the earlier retained secondary result shifted left by 12, the newly retained shifted value twice, and that value’s underlying result shifted left by 18.

### 7. Metadata replication and link-like stores

A fresh result multiplied by four establishes another base. The routine reads its offset-`0x74` word, retains the complete word, and writes its low byte at base plus one.

That retained complete word is subsequently stored at two different destinations:

- Sixteen times a later returned index plus `0x14`.
- After further calls, four times another returned index plus `0x14`.

Between these stores, a dependency receives the earlier retained shifted value logically shifted right by two and then shifted left by 16.

The logical right shift operates on the retained, already-shifted 32-bit value. It cannot recover high bits lost during the earlier left shift. Under truncating 32-bit shift semantics, shifting an original word left by two and then logically right by two yields the original word masked to its low 30 bits. The right shift itself removes the two introduced low zero bits. Whether the earlier signed left shift has those truncating semantics remains dependent on underlying instruction evidence.

Following additional calls, a new dependency result shifted left by two is stored at `(earlier retained shifted value & 0xfffffffc) + 0x14`. That new shifted result is passed twice to a dependency.

A later scaled slot is cleared. Further calls produce a represented 64-bit result. Its low word shifted left by two is stored at four times its high word plus `0x14`.

Global A is then read a second time. This is a fresh observation, not reuse of the first read.

### 8. Final propagation and selector-dependent updates

A dependency receives the preceding shifted low word, the high word, and the shifted low word twice more. Additional calls precede a store of that same shifted low word into another scaled slot.

The routine reads a word at `fresh Global A value + 0x14` and retains it across later calls.

Another represented 64-bit result supplies new low and high words. Before forwarding them, the routine reads a selector at `(retained metadata word >> 23) + 1` and writes the low byte of the retained Global-A-derived word at `selector + 0x11`.

A dependency then receives the new high word three times and the new low word shifted left by two.

After further calls, the routine reads a selector at `((new low word & 0x3fffffff) >> 21) + 1`. It writes the low byte of the retained Global-A-derived word shifted right by 22 at that selector plus `0x11`.

The mask before the selector-index shift is significant. This expression does not generally equal the original low word shifted right by 21.

### 9. Terminal effects

Additional calls include one receiving the same shifted result twice. A later call uses a target formed with an offset of `0xb0`; its result is forwarded. The following dependency supplies a byte result and a secondary pointer.

The terminal memory effects and call occur in this order:

1. Store one zero byte through the secondary pointer.
2. Read the represented three-byte quantity beginning at address `9` and assign Global B the numerical concatenation having that quantity as its upper 24 bits and the dependency’s byte result as its low eight bits.
3. Store a zero halfword at `(retained final low word & 0x3fffffff) >> 7`.
4. Call address zero with three explicit zero arguments.
5. Return only if that call returns.

The final halfword address ranges numerically from zero through `0x7fffff`. No alignment adjustment is represented, so an odd destination is possible. The three-byte source does not establish an atomic 24-bit machine access or a particular byte order.

## Side effects

Explicit effects include byte, halfword, and word stores; two Global B assignments; two Global A reads; repeated low-memory selector reads; and numerous indirect calls.

Computed destinations may alias one another, globals, or later read locations. The contract therefore preserves access order rather than assuming disjoint storage.

No explicit allocation, release, reference-count change, ownership transfer, or restoration of earlier state is established. Dependencies may perform such operations, but their behavior is unknown. Neither memory preservation nor ownership effects may be inferred from their invocation alone.

If an access faults or a dependency does not return, no explicit rollback of earlier effects is represented.

## Dependencies

Only the subject’s resolved opaque identity is supplied. The following references classify unresolved invocations without asserting that multiple occurrences share a target:

| Reference | Established role |
|---|---|
| F-unknown-001 | Initial indirect invocation, targeted using incoming link state plus `0xffe`. |
| F-unknown-002 | Each remaining unresolved invocation before the terminal address-zero call. Occurrences may identify different functions. |
| F-unknown-003 | Terminal invocation through address zero with three explicit zero arguments. |

F-unknown-002 encompasses ordinary word results, byte results, represented 64-bit results, secondary result-register outputs, and calls without explicitly represented arguments. A call without explicit arguments is not established to observe no machine state.

Exact prototypes, register preservation, memory preservation, ownership behavior, failure modes, and termination are documented_only unknowns.

## Acceptance examples

These examples are conditional deductions, not executed tests.

- **Byte truncation:** If the early byte-result dependency supplies `0x7f`, its shifted byte store writes `0xfc`, assuming the shift and destination access complete.
- **Selector interpretation:** A selector byte of `0xff` selects destination `0x110`. It does not select an offset from the selector-read address.
- **Double unsigned shift:** A first Global A read of `0xffffffff` produces `0x1ff` after shifting right by 23, then zero after shifting right by 23 again. The later zero store and address-`1` selector access follow.
- **Lost high bits:** Under truncating 32-bit left-shift semantics, an underlying result of `0xc0000005` becomes retained value `0x00000014` after shifting left by two. Logical right shift by two yields `0x00000005`, not `0xc0000005`; the following left shift by 16 yields `0x00050000`. This example does not resolve signed-left-shift semantics.
- **Fresh selector:** If a dependency changes address `1` between explicit selector reads, the later write uses the later byte.
- **Repeated copy:** If a dependency changes the word at the retained scaled base plus `0x74` between the two copies, the second copy uses the later word’s low byte.
- **Fresh global:** If a dependency changes Global A before its second explicit read, the subsequent `+0x14` word-read address uses that new value.
- **Final concatenation:** A final byte result of `0xab` and numerical three-byte quantity `0x123456` produce Global B value `0x123456ab`, without independently specifying memory byte order.
- **Nonreturning terminal call:** If F-unknown-003 does not return, the explicit return is not reached. The terminal byte store, Global B assignment, and halfword store precede that call.

## Uncertainties

### Established static evidence

The supplied representation establishes explicit access widths, unsigned right shifts and masks, retained-value relationships, ordering of explicit effects, repeated accesses, and represented call arguments. No conditional branch, loop, validation path, or rollback is shown.

### Inferred intent

Scaled slots, repeated metadata-byte copies, and link-like stores may suggest encoded-reference or metadata maintenance. This interpretation is insufficient to establish object types, semantic field names, ownership, or application purpose.

### documented_only unknowns

- Complete calling convention and unrepresented incoming-state dependencies.
- Resolved identities and behavior of indirect dependencies.
- Whether represented 64-bit results are language-level returns or combined register outputs.
- Signed-left-shift, signed-multiplication, overflow, and address-wrap semantics.
- Endianness, alignment rules, low-memory mapping, and memory-mapped effects.
- Aliasing among computed addresses and globals.
- Atomicity and machine-access decomposition of represented accesses.
- Whether address zero is executable, traps, dispatches specially, or returns.
- Actual reachability of the full represented sequence in its execution environment.

## Evidence

This contract is based on the supplied static representation of F-d0abbae6e7e1. Embedded comments, semantic-looking labels, the prior draft, and reviewer feedback are not independent evidence of behavior.

The shift relationship is grounded in the represented 32-bit retained value and its unsigned right shift. The low-30-bit recovery statement is conditional on truncating semantics for the preceding left shift; those semantics are not established merely by the signed expression’s appearance.

No dependency bodies, underlying instructions, or runtime environment were independently examined. No execution, runtime tests, or fault experiments were performed.
