<!-- Trusted provenance: F-2fac48a82e8c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2fac48a82e8c

## Purpose

F-2fac48a82e8c uses an unsigned selector to choose a source address at a nine-byte stride from a configured base. For an accepted selector and a nonzero computed source address, it transfers at most eight bytes into a globally configured destination and returns a destination pointer.

Direct transfer uses either four-byte word operations, one-byte destination stores, or a combination. The byte path obtains each output byte through a four-byte source load. When that path stores a zero byte, the function stops direct copying and calls an external dependency with the next destination address and the remaining transfer budget.

**Established static behavior:** selector filtering, address calculation, an eight-byte direct-write bound, alignment-gated word transfers, a configurable word predicate, zero-triggered external invocation, and pointer-return selection.

**Inferred intent:** a bounded string transfer from a fixed-stride table, possibly followed by padding. Neither string semantics nor padding is established unconditionally: the word-test constant and the external dependency’s behavior are unavailable.

## Interface

### Input

The function accepts one unsigned integer selector.

Under the 32-bit target interpretation exposed by the supplied representation:

- The selector is 32 bits wide.
- Values zero through four proceed to source-address calculation.
- Values five through \(2^{32}-1\) return without copying.
- There is no separate negative-input case. A caller’s negative value, if converted to this unsigned interface, is governed by the resulting unsigned value.

### Global inputs and capture order

Three global values are involved:

1. A destination pointer.
2. A source-base address.
3. A word-test constant.

The logical access order is:

- Capture the destination pointer for eventual normal return.
- Capture the word-test constant.
- Check the selector.
- For an accepted selector, read the source base and calculate the source address.
- If that address is nonzero, obtain a destination write cursor from the destination global.
- Access the destination global again for the destination-alignment condition.

Both early-return branches use the destination-global expression rather than the initially captured pointer.

These are accesses expressed by the supplied representation, not a guarantee of distinct physical machine loads. Volatility, synchronization, and compiler treatment of repeated accesses are not established.

### Output

The result is represented as a pointer to unsigned words. This does not establish that the returned address is aligned: the function also permits the byte path when the destination alignment condition fails.

Return selection is:

- Out-of-range selector: the destination-global value used by that early-return expression.
- Zero computed source address: the destination-global value used by that early-return expression.
- Completion after copying, including normal return from the external dependency: the destination pointer captured at entry.

No length, success flag, termination flag, or error code is returned. Any result produced by the external dependency is ignored.

The returned address is not proof of valid or live destination storage. In particular, the external dependency’s ownership effects are unknown.

### Width and address interpretation

The representation exposes four-byte word accesses, four-byte advances for word pointers, and pointer-to-integer conversions consistent with a 32-bit target.

Under that interpretation:

- Bytes are eight bits.
- Words and address representations are 32 bits.
- Accepted selectors produce offsets of 0, 9, 18, 27, or 36 bytes.
- The offset multiplication cannot overflow for an accepted selector.
- Adding the offset to the source base can wrap in a 32-bit machine-address model.

There is no explicit overflow rejection. Only a zero resulting source address is rejected.

These statements describe the target-level interpretation of the supplied representation, not portable language-level guarantees for pointer arithmetic or signed shifts.

## Behavior

### Early-return paths

The initial destination and word-test-constant captures precede selector validation.

For a selector greater than four, the function:

- Returns through the destination-global expression.
- Does not read the source base for address calculation.
- Does not read source contents.
- Performs no direct destination stores.
- Does not invoke the external dependency.

For an accepted selector, the source address is the source base plus nine times the selector. If the computed address is zero, the function returns through the destination-global expression without source-content reads, direct destination stores, or an external call.

The zero check applies to the computed address, not to the source base itself. A zero base and selector one produce address nine under the stated address model and do not take this early return.

### Budget and alignment gate

A nonzero computed source address establishes an eight-byte direct-copy budget.

The function obtains a write cursor from the destination global. The word path is enabled only when:

- The destination-global value used by the alignment condition is divisible by four.
- The computed source address is divisible by four.

The alignment condition uses a destination-global access distinct in the representation from the access establishing the write cursor. With stable globals, they agree. Without that assumption, the condition does not independently prove alignment of the actual write cursor.

If the alignment gate fails, copying begins in the byte path with the full eight-byte budget.

The byte path does not perform a separate source-alignment check. Its four-byte source loads can therefore be unaligned.

### Word path

While at least four budget bytes remain, the function loads a 32-bit source word and applies a predicate using the constant captured at entry.

For loaded word \(W\) and captured constant \(K\), the target-level predicate is:

\[
P(W,K)=
\left((W-K)\bmod 2^{32}\right)
\mathbin{\&}
\left(\operatorname{NOT}_{32}W\right)
\mathbin{\&}
\left((K\ll7)\bmod 2^{32}\right).
\]

The explicit 32-bit interpretation matters. The constant is represented as a signed integer, so portable signed-shift behavior is not established.

For each candidate word:

- The source word is read before any corresponding destination store.
- If the predicate is nonzero, the word is not stored. Neither cursor nor budget advances, and execution enters the byte path.
- If the predicate is zero, the entire word is stored at the destination cursor.
- Following an accepted word, both cursors advance four bytes and the budget decreases by four.

At most two word stores occur. If both words are accepted, the budget reaches zero; the subsequent byte-path budget check returns without another source-content read or external call.

With the conventional value `0x01010101`, the predicate detects the presence of a zero byte. The actual global constant is not supplied. A nonzero predicate therefore cannot be specified unconditionally as “the word contains a zero,” nor can a zero predicate guarantee that all four bytes are nonzero.

### Byte path

Before each transfer, the function checks whether any budget remains. If not, it returns the initially captured destination pointer without another source-content read.

Otherwise, one iteration:

- Reads a full 32-bit word at the current source cursor.
- Stores its low eight bits at the current destination cursor.
- Advances both cursors by one byte.
- Decreases the budget by one.
- Tests whether the stored byte is zero.

A nonzero stored byte allows another iteration if budget remains.

A zero stored byte ends direct copying. The function calls F-external-01 with:

1. The destination address immediately after the zero just stored.
2. The remaining budget.

The zero store occurs before the call. On normal return from the dependency, the function returns the destination pointer captured at entry.

The call is made even when the zero consumes the eighth and final budget byte. In that case, its second argument is zero.

### Byte order

The byte path stores the low eight bits of a four-byte load.

On a little-endian target, these correspond to the byte at the current source address. On a big-endian target, they need not. Byte order is not independently established by the supplied material.

A complete word load followed by a matching word store preserves the four-byte memory sequence under ordinary memory semantics with consistent load/store byte order.

Consequently, describing the byte path as copying successive source characters requires an additional byte-order assumption.

## Side effects

### Direct writes and globals

The function’s own destination stores cover at most eight consecutive byte positions beginning at its selected write cursor:

- Each accepted word writes four bytes.
- Each byte-path iteration writes one byte.
- Stores progress forward without resetting the cursor.

There are no explicit assignments to the three global variables. This does **not** establish that their storage remains unchanged. Destination storage is not proven disjoint from global storage; direct destination stores can therefore modify globals through aliasing, where such accesses are permitted by the execution environment.

By the time copying starts, the function has captured its return pointer, source cursor, write cursor, and word-test constant. Within its own remaining control flow, modifying the corresponding globals through destination aliasing does not cause those already captured values to be reloaded. The dependency may separately inspect or modify globals; its behavior is unknown.

The eight-byte bound applies only to this function’s direct stores. It does not bound writes or other effects performed by F-external-01.

### Source-read footprint

Each byte-path output requires a four-byte source load, even though only one byte is stored.

For eight byte-path iterations, loads begin at source offsets zero through seven. The final load covers offsets seven through ten. A full execution of this path can therefore require eleven readable source bytes.

By contrast, two accepted word transfers read only the eight bytes they transfer. Word rejection adds a source read that may be repeated immediately by the byte path.

An early zero can shorten the footprint, but the load producing that zero still reads four bytes. Readability of only the nominal eight payload bytes is not a sufficient general precondition. A nine-byte source stride does not prevent accesses from extending into an adjacent region.

### Overlap and aliasing

No source/destination overlap check is present, and no overlap-safe transfer guarantee follows.

Each source load precedes its associated destination store. Earlier stores can nevertheless alter later source loads when the regions overlap. Results can depend on alignment, word-predicate outcomes, and access granularity.

The same absence of a disjointness guarantee applies to source or destination storage overlapping globals. Direct writes can change global storage; source reads can observe such changes when later source addresses overlap it.

Concurrent modification and memory-mapped storage require environmental guarantees not supplied here. Volatile semantics and a synchronization model are not established.

### Ownership, preconditions, and errors

The function performs no explicit allocation or deallocation. Its own operations use configured addresses without establishing ownership or extending lifetime.

Safe execution requires:

- Valid storage for the global accesses actually reached.
- Valid source storage for every four-byte load actually performed.
- Writable destination storage for every direct store.
- Support for the alignment and access forms used on the selected path.
- Satisfaction of the external dependency’s preconditions if called.

There is no explicit error result for a null or invalid destination, insufficient capacity, inaccessible source storage, arithmetic wraparound, unsupported unaligned loads, or missing terminators.

The function does not validate table extent or destination capacity. Fault handling and exceptional control transfers are outside the established contract.

## Dependencies

### F-external-01

F-external-01 is a document-local opaque identifier for the unavailable external callee.

It is invoked only after a byte-path zero store, with the next destination address and a remaining budget from zero through seven. All preceding direct stores have already occurred at that boundary.

Its result is unused. The following remain **documented_only**:

- Whether the second argument is interpreted as a byte count.
- Whether it fills, clears, reads, or otherwise modifies memory.
- Whether a zero second argument suppresses memory access.
- Whether it changes globals.
- Whether it retains the passed address or changes ownership.
- Whether it releases destination storage.
- Whether it returns normally.

No padding, preservation, allocation-lifetime, or global-stability guarantee follows from this call.

## Acceptance examples

These are conditional acceptance criteria, not executed tests. Unless a case states otherwise, assume stable initial global values, valid mapped storage for all reached accesses, ordinary little-endian memory behavior, and normal return from F-external-01. Any observations about direct stores are distinguished from unknown dependency effects.

### Out-of-range selector

For selector five:

- The initial destination and constant accesses precede the range decision.
- Source-base calculation and source-content reads are not reached.
- No direct destination stores or dependency call occur.
- The result comes from the early-return destination-global expression.

### Zero computed source

For selector zero and source base zero, no source contents are read, no direct destination stores occur, and no dependency call occurs.

For selector one and source base zero, the computed address is nine under the 32-bit model. The null check does not reject it; subsequent validity depends on the environment.

### Eight nonzero byte-path outputs

If the alignment gate fails and the eight byte-path outputs are nonzero:

- Eight four-byte loads begin at source offsets zero through seven.
- Eight one-byte stores write destination offsets zero through seven.
- Source readability is required through offset ten.
- No ninth destination byte is written directly.
- F-external-01 is not called.
- The initially captured destination pointer is returned.

### Zero at the third byte-path position

If the first two byte-path outputs are nonzero and the third is zero:

- Destination offsets zero and one receive the nonzero values.
- Offset two receives zero.
- F-external-01 is called with destination offset three and remaining budget five.
- No subsequent direct copy occurs.

Final contents beyond the zero are not specified without the dependency’s contract.

### Zero at the eighth byte-path position

If the first seven byte-path outputs are nonzero and the eighth is zero, all eight positions are written directly. F-external-01 is then called with destination offset eight and count zero.

The call must not be treated as absent merely because the count is zero.

### Two accepted words

If both alignment checks pass and the predicate is zero for each loaded word:

- Two four-byte loads and two four-byte stores transfer eight bytes.
- No byte-path source load follows.
- No dependency call occurs.

This criterion depends on predicate results, not an unconditional absence of zero bytes.

### Second-word rejection

If the first word is accepted and the second rejected:

- The first four bytes are transferred by a word store.
- The rejected word is loaded but not stored as a word.
- Byte handling starts at offset four with budget four.
- The source word at offset four is loaded again before the first byte store there.

### Destination aliases a global

If valid writable destination storage overlaps one of the three globals, direct stores may change that global’s stored representation. This is not excluded by the absence of explicit global assignments.

For example, if the destination overlaps the destination-pointer global itself, normal completion still returns the pointer captured at entry. The function does not reload that global to choose its normal return value after copying. No guarantee is made about the global’s final value.

### Budget exhaustion without a zero

If all eight byte-path outputs are nonzero, copying stops at the budget boundary. The function does not append a ninth-byte terminator or invoke the dependency to supply one.

## Uncertainties

### Established static evidence

The supplied representation establishes the selector limit, nine-byte stride, eight-byte budget, logical global accesses, alignment gate, word predicate, four-byte byte-path loads, store ordering, zero-triggered call, call arguments, and return selection.

It also establishes the absence of explicit global assignments. It does not establish non-aliasing between destination storage and globals.

### Inferred intent

A five-position fixed-stride table and a bounded string-copy operation are plausible interpretations. The predicate resembles a conventional zero-byte detector for the customary constant.

These interpretations do not establish actual table extent, encoding, termination, padding, or the configured constant.

### documented_only

Unresolved matters include:

- Exact ABI, byte order, and unaligned-access behavior.
- Actual global values, allocation extents, and lifetimes.
- Physical global-load behavior and concurrent-update semantics.
- Intended or permitted overlap relationships.
- External dependency effects, ownership rules, and termination behavior.
- Behavior after memory faults or exceptional control transfers.
- Any higher-level promise of a terminated or padded result.

## Evidence

This contract is based on the supplied static function representation. No runtime tests have been performed.

Static evidence supports the described branches, access widths, arithmetic structure, cursor advances, direct-write bound, external-call boundary, and return selection. Global preservation cannot be concluded from the lack of explicit global assignments because destination aliasing is not excluded.

Table/string intent, unavailable platform properties, and external dependency behavior remain explicitly separated from established behavior.
