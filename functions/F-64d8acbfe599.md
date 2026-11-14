<!-- Trusted provenance: F-64d8acbfe599; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-64d8acbfe599

## Purpose

F-64d8acbfe599 performs a fixed, call-heavy sequence of indirect dependency invocations, computed-address memory writes, memory reads, and updates to shared state. It has no declared parameters and no declared return value, but its behavior depends on implicit machine state supplied at entry and on values made available by dependencies.

The supplied static representation does not establish an application-level purpose. In particular, it does not justify describing the routine as an allocator, serializer, supervisor-call implementation, or object initializer. Repeated writes at offsets 1, 17, 20, and 116 suggest structured state manipulation, but the meaning and validity of those structures remain unknown.

The principal externally observable behavior is the **ordered interaction trace**, not a returned result. The routine ends by attempting an indirect call to address zero with three zero arguments. A normal return is represented only after that call returns.

## Interface

### Declared interface

- Explicit parameters: none.
- Declared result: none.
- Explicit error result: none.
- Explicit validation, recovery, or rollback: none.

The absence of explicit parameters does not make this routine independent of its caller.

### Implicit inputs

The represented execution consumes:

- An incoming return-link value, used to derive the first indirect call target.
- An incoming 32-bit word, later shifted right by 22 bits.
- Another incoming 32-bit word, stored without interpretation.
- A shared 32-bit location, designated **G-A**, read twice at substantially different points.
- A shared byte at address 1, designated **G-C**, read on separate occasions.
- A shared three-byte quantity beginning at address 9, designated **G-D**, used near termination.
- Primary dependency results represented as 32-bit words or 64-bit pairs.
- Additional register-derived outputs from dependencies whose calling contracts are unavailable.

A further shared 32-bit destination, **G-B**, is overwritten twice.

These labels describe storage roles, not source-level names or independent allocations. Overlap between global storage views is explicitly possible.

### Width and address notation

The following notation describes values without assigning application semantics:

- **W(x)**: the low 32 bits of a value.
- **B(x)**: the low 8 bits of a value.
- **Q(x)**: four times a word-sized value, interpreted as a 32-bit machine quantity where that interpretation is valid.
- **Slot(x)**: address Q(x) + 20.
- **Byte destination(x)**: address 17 plus the unsigned byte read at address x.

All right shifts discussed below operate on values represented as unsigned 32-bit quantities. A word shifted right by 23 therefore lies in 0–511; a word shifted right by 22 lies in 0–1023.

The static representation also contains multiplication and left shifts of signed-typed quantities. Their intended machine interpretation is not independently established. A 32-bit wrapping interpretation is a useful conditional model, not a proven portable-language contract.

### Execution preconditions

Successful completion requires all reached call targets to be executable and all reached memory accesses to be supported by the execution environment.

This includes low memory:

- A byte read at address 1 is required on the ordinary represented path.
- Computed byte writes may target addresses 17–272.
- A final two-byte write may target address zero.
- The final call targets address zero.

No alignment, bounds, executable-memory, or null checks are shown. These requirements are environmental preconditions, not checks performed by the routine.

## Behavior

### Control-flow shape

The represented body is straight-line: no explicit loop, conditional branch, retry, or early success exit appears. Each dependency invocation and memory operation is reached only if all earlier operations return or complete normally.

Intervening calls are behaviorally significant. They must not be removed, merged, or treated as memory-neutral simply because their displayed results are unused. They may affect shared storage subsequently read by the routine.

The sequence below groups related effects rather than prescribing implementation statements.

### 1. Initial dependency exchange and first word stores

The first dependency invocation uses a target derived from the incoming return-link value plus 4094. Its 32-bit result is forwarded to the next dependency. Several further indirect calls precede a dependency result used as a word index.

The routine then:

1. Stores the incoming uninterpreted 32-bit word at Slot of that result.
2. Reads G-A and retains this first snapshot.
3. Calls a dependency with the scaled result Q(index).
4. Performs another sequence of indirect calls.

An auxiliary register output from that latter sequence supplies another index. The routine stores the incoming shifted word—its original value shifted right by 22—at Slot of this auxiliary index.

A dependency then receives four words, in order:

- The auxiliary index.
- The same auxiliary index.
- The first G-A snapshot shifted right by 23.
- The incoming word shifted right by 22.

The incoming shifted value remains available for a later store. The G-A-derived quantity is also retained; dependencies between its computation and later use do not cause an explicit reread of G-A.

### 2. Repeated word publication and a forced-zero value

A new dependency result is forwarded to another dependency. Further calls produce an index at whose Slot the incoming word shifted right by 22 is stored again. A subsequent dependency receives the scaled index.

Another call sequence exposes an auxiliary unsigned word. Zero is stored at its Slot, and the unscaled auxiliary word is then supplied to a dependency.

After another result-forwarding exchange and call sequence, a new Slot receives the first G-A snapshot shifted right by 23 twice in succession.

This value is necessarily zero under the represented 32-bit unsigned widths: after the first shift, at most nine bits remain; shifting that result right by another 23 clears them all.

Immediately after this zero store, the routine reads a 32-bit word at:

**20 + ((auxiliary word AND 0x1fffffff) shifted right by 19).**

This address is not an ordinary four-byte-stride index expression. Its possible numerical range is 20–1043, and four-byte alignment is not guaranteed.

The value read here is retained through subsequent calls, then shifted right by 22.

### 3. First byte writes and repeated auxiliary arguments

The first byte-selector read uses the twice-shifted G-A-derived quantity plus 1. Under the established unsigned-width calculation, that selector address is exactly 1.

The byte read there selects a destination at 17 plus the selector byte. The routine writes B of the retained memory word shifted right by 22 to that destination.

Consequently:

- The retained shifted word lies in 0–1023.
- Only its low eight bits are written.
- Bits 8 and 9 of that shifted value are discarded by the byte store.

A dependency receives the same auxiliary 32-bit output in its first three argument positions and zero in its fourth position.

After another dependency invocation, a further dependency is explicitly passed zero. This zero is derived from a local zero shifted right by 23, rather than from a new memory read.

Subsequent calls produce a primary result, but the next byte destination is selected using the previously retained 0–1023 shifted value, not that new result. The routine reads a selector byte at four times that retained value plus 1 and stores zero at 17 plus the selector.

A dependency receives the new primary result scaled by four. Later calls produce another result, whose scaled value is supplied twice to a dependency.

### 4. Shared-byte selections and first shared-word update

A dependency returns a 32-bit word that is retained and forwarded to another dependency. After another call, the routine reads G-C and writes zero at address 17 plus that byte.

An auxiliary output from this region is saved for substantially later shift-based arguments. A dependency receives that auxiliary output three times.

A subsequent returned word is forwarded alongside the retained word shifted right by 23.

Further calls produce a result used in another scaled-argument invocation. Before that invocation, the routine reads G-C again, retains the byte as an unsigned base, and writes zero at base + 17.

These two G-C reads are separate observations. Dependency side effects between them may change the selected address.

Another repeated-scaled-argument exchange produces a new 32-bit word. The routine:

1. Stores that word into G-B.
2. Forwards the same word to a dependency.
3. Invokes another dependency.
4. Reads a 32-bit word at the retained byte base + 116.
5. Writes the low byte of that word to the retained base + 1.

The retained base is not refreshed from G-C for this read/write pair.

### 5. Paired-result handling and repeated offset-116 reads

The previously saved auxiliary output is shifted left by 11 and passed to a dependency. A following dependency produces a 64-bit result.

The next dependency receives:

- The low 32 bits of that result.
- The high 32 bits.
- The word most recently stored in G-B, shifted right by 23.

Further calls produce a word index. Its scaled form becomes a retained base. The routine reads a 32-bit word at that base + 116 and writes its low byte at that base + 1.

A dependency receives this base. After additional calls yield a second index, the routine again reads at the **same retained base + 116** and again writes the low byte at the same retained base + 1.

This is a fresh read, not a reuse of the previous word. Intervening dependencies can therefore change the second byte written.

The second index, scaled by four, is retained separately. A dependency receives four arguments:

- The saved auxiliary output shifted left by 12.
- The second scaled index.
- The same second scaled index again.
- The unscaled second index shifted left by 18.

### 6. Word capture and propagation

Several return-value forwarding exchanges lead to another word index. At its scaled base, the routine:

1. Reads a 32-bit word at offset 116.
2. Retains the full word.
3. Writes its low byte at offset 1.
4. Passes the base to a dependency.

A later result supplies an index for a different layout: the full retained word is stored at **16 times that index + 20**.

A dependency then receives the retained second scaled index shifted right by 2 and subsequently left by 16. This expression must not automatically be replaced by the original unscaled index shifted left by 16: an earlier fourfold scaling may already have discarded high bits under a wrapping model.

After more forwarding exchanges, the full retained word is stored again, this time at Slot of another dependency result. A dependency receives that index scaled by four.

A further result is scaled by four and stored at:

**20 + (the retained second scaled index AND 0xfffffffc).**

That destination derives from the earlier retained scaled index, not the newly returned one. The newly scaled result is then passed twice to a dependency.

### 7. Zeroing, paired indices, and the second G-A snapshot

Following additional result forwarding, zero is stored at Slot of another dependency result. A dependency receives that result scaled by four.

A later dependency returns a 64-bit pair. The low half, scaled by four, is stored at Slot of the high half.

Only after that store does the routine read G-A again. This is a second, independent snapshot; it is not assumed equal to the first.

A dependency receives four words:

- The scaled low half.
- The unscaled high half.
- The scaled low half again.
- The scaled low half a third time.

Further calls produce an index at whose Slot the same scaled low half is stored. The routine then reads a 32-bit word at the second G-A snapshot + 20, before passing the new index scaled by four to a dependency.

Thus the second G-A snapshot is used as a direct address base, unlike the first snapshot’s shift-derived uses.

### 8. Final selector writes and termination

Another dependency returns a 64-bit pair. Its low half and high half remain distinct.

The first following byte selector is read at:

**1 + (the full word captured in phase 6 shifted right by 23).**

The selected destination, 17 plus that selector byte, receives the low byte of the word read through the second G-A snapshot.

A dependency receives the new pair’s high half three times, followed by its low half scaled by four.

After another forwarding exchange and call sequence, a second byte selector is read at:

**1 + ((the pair’s low half AND 0x3fffffff) shifted right by 21).**

The selected destination receives the low byte of the G-A-based memory word shifted right by 22.

Subsequent dependencies receive scaled primary results, including one invocation with the same scaled result twice. More calls lead to a dependency whose returned word is forwarded to another dependency. A final dependency produces a byte result and an auxiliary pointer.

The terminal effects occur in this order:

1. Store zero through the auxiliary pointer as a one-byte write.
2. Read the three-byte shared quantity G-D and combine it as the upper 24 bits with the final byte result as the lower eight bits.
3. Store that combined 32-bit value to G-B.
4. Store a two-byte zero at address  
   **(the retained pair’s low half AND 0x3fffffff) shifted right by 7**.
5. Attempt a call to address zero with three zero arguments.
6. Return only if that call returns.

The halfword destination ranges from zero through 8,388,607. Neither nonzero address nor two-byte alignment is guaranteed.

## Side effects

### Memory and shared state

The routine performs byte, two-byte, and four-byte writes. Full-width reads followed by low-byte stores still constitute full-width reads; they must not be reduced to byte reads without additional evidence about the environment.

Potentially overlapping destinations include:

- Four-byte-stride slots at offset 20.
- Sixteen-byte-stride slots at offset 20.
- Byte fields at offsets 1 and 17.
- Word fields at offset 116.
- Low-memory selector tables and their selected destinations.
- G-A, G-B, G-C, and G-D storage views.
- The final auxiliary-pointer destination and halfword destination.

No non-aliasing guarantee is established. Earlier writes or dependency side effects may alter later reads. The order of accesses is therefore part of the contract.

### Ownership and lifetime

No ownership transfer, allocation, deallocation, reference counting, or lifetime extension is established.

Values used as indices or address bases are not proven to identify allocated objects. Passing them to dependencies does not establish whether those dependencies acquire, release, retain, or invalidate memory.

### Errors and partial completion

No status is checked after dependency calls. No memory access is guarded.

If an invocation fails to return, a memory operation faults, or the environment rejects an access, later effects are not established. Earlier effects are not rolled back.

The representation provides no concurrency protection or atomicity guarantee.

## Dependencies

No resolved dependency identities are supplied beyond F-64d8acbfe599 itself. For this contract, the following opaque identifiers classify unresolved invocation roles; they do **not** assert that different call sites target the same function:

| Opaque ID | Observed role |
|---|---|
| F-U001 | Entry invocation derived from the incoming return-link value. |
| F-U002 | Intervening indirect invocations, including result producers and calls with no displayed explicit arguments. |
| F-U003 | Invocations consuming forwarded 32-bit results. |
| F-U004 | Invocations consuming indices, scaled indices, repeated arguments, or shift-derived arguments. |
| F-U005 | Invocations producing results represented as 64-bit pairs. |
| F-U006 | Terminal invocation producing a byte and an auxiliary pointer. |
| F-U007 | Final invocation at address zero, supplied three zero arguments. |

These IDs are documentation-local opaque references, not recovered symbol mappings. Target equality between invocation sites is unknown.

Most targets are formed from embedded code anchors plus 4094; some use different offsets. Exact executable target identities and calling conventions are unresolved. A call displayed without explicit arguments does not prove that no register or stack state is consumed.

No dependency is assumed to preserve globals, preserve pointed-to storage, or manage ownership in any particular way.

## Acceptance examples

These are conditional static examples, not runtime test results.

1. **Double-shift zero.**  
   For every unsigned 32-bit first G-A snapshot, shifting right by 23 twice produces zero. If preceding calls return normally, the corresponding Slot store writes four zero bytes, and the subsequent first byte-selector read uses address 1.

2. **Byte truncation after a shift.**  
   If the phase-2 memory word is 0xffc00000, its unsigned right shift by 22 is 1023. The first selected byte destination receives 255, not 1023. The later selector address based on four times the retained shifted value is 4093.

3. **Intervening mutation remains visible.**  
   If the two reads at the retained base + 116 in phase 5 observe 0x11223344 and 0x55667788 respectively, their byte writes are 0x44 and 0x88. Reusing the first read for the second store would violate the represented behavior.

4. **Distinct G-A snapshots.**  
   If dependencies change G-A between its reads, the first snapshot still determines the early shift-derived values. The second snapshot determines the later direct-base word read.

5. **Selector boundaries.**  
   A selector byte of zero chooses destination address 17; a selector byte of 255 chooses address 272. There is no demonstrated check rejecting either value.

6. **Terminal combined word.**  
   If G-D’s three-byte value is 0x123456 and the final dependency byte is 0xab, G-B receives 0x123456ab, conditional on those observations remaining the values used at that step.

7. **Zero terminal destination.**  
   If the retained pair’s low half has no set bits within mask 0x3fffffff, the final two-byte zero store targets address zero. No protective branch is shown.

8. **Final call failure.**  
   If address zero is not executable, the routine need not return normally. The preceding terminal writes are nevertheless attempted before that invocation, subject to their own successful completion.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and data-model interpretation.
- Exact executable identities of indirect dependencies.
- Calling conventions and the provenance of auxiliary register outputs.
- Whether apparently unused dependency results are irrelevant at machine level.
- Signed overflow and signed-left-shift interpretation.
- Endianness, alignment support, and memory-mapped I/O behavior.
- Allocation boundaries, pointer validity, ownership, and object lifetimes.
- Global overlap details and concurrent mutation.
- Whether address zero is executable in the intended environment.
- Whether the final represented return is reachable.
- Any stronger success, failure, preservation, or postcondition guarantee.

Repeated offsets and shift patterns may indicate an encoded runtime representation, but that interpretation is inferred intent, not established behavior.

## Evidence

The established evidence is limited to the supplied static function representation: explicit access widths, address expressions, unsigned shifts, truncations, argument relationships, indirect invocation order, two G-A reads, two G-B writes, and the terminal zero-address call.

The representation also warns that global storage views may overlap. It exposes incoming machine-state values and auxiliary outputs without complete interface definitions.

No dependency implementation, authoritative ABI, memory map, or runtime observation was supplied. No runtime tests have been performed. All acceptance examples above are conditional arithmetic or ordering consequences of the static evidence, not claims of observed execution.
