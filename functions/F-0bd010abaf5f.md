<!-- Trusted provenance: F-0bd010abaf5f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0bd010abaf5f

## Purpose

The established behavior is a straight-line sequence of indirect dependency calls, 32-bit memory transfers, byte updates, two publications to shared control storage, and a final halfword clear. The last explicit call targets address zero and supplies three zero arguments.

A higher-level purpose is not established. Scaling values by four, accessing words at displacement 20, and copying bytes from words at displacement 116 suggest encoded references or runtime-state manipulation. This is **inferred intent**, not evidence of object types, allocation, reference counting, ownership, or error meanings.

This contract distinguishes:

- **Established static evidence:** visible accesses, argument relationships, widths, and ordering.
- **Conditional arithmetic interpretation:** conclusions requiring 32-bit wrapping arithmetic.
- **documented_only unknowns:** unresolved dependency effects, calling convention, memory mapping, and exceptional behavior.

No runtime tests have been performed.

## Interface

### Explicit arguments and result

The apparent interface accepts two opaque 32-bit arguments and has no explicit return value.

- The first argument has no explicit use in the visible body.
- The second argument is passed to the initial indirect dependency.
- No status result, returned address, or error code is established.
- Reaching the apparent return requires every preceding operation, including the call through address zero, to complete and return.

The lack of an explicit use does not establish that the first argument is irrelevant under the actual calling convention.

### Additional required entry state

The apparent argument list is not a complete description of the function’s inputs. The body additionally consumes:

- Incoming link-related state used to select the first dependency target.
- An incoming 32-bit selector used in an early word-read address.
- An incoming opaque 32-bit payload used in the first explicit word write.
- Auxiliary-register outputs associated with indirect calls.
- A shared 32-bit source read at two distinct points.
- The byte at absolute address 1.
- Shared storage supplying a final 24-bit control component.

These inputs are not validated. The initial dependency target is formed from the incoming link-related value plus `0xf38`; the interpretation and validity of that target remain unresolved.

### Widths and memory notation

The following notation describes observable accesses:

| Notation | Meaning |
|---|---|
| \(R_8(a)\) | Read an unsigned byte at address \(a\) |
| \(R_{32}(a)\) | Read a 32-bit word at address \(a\) |
| \(W_8(a,v)\) | Store the low eight bits of \(v\) |
| \(W_{16}(a,v)\) | Store the low sixteen bits of \(v\) |
| \(W_{32}(a,v)\) | Store a 32-bit value |
| \(S(x)\) | Slot address \(4x+20\) |

A word is four bytes; a halfword is two bytes. These widths do not establish byte order or alignment requirements.

Unsigned right shifts are logical. Narrowing to a byte discards higher bits rather than saturating.

The representation mixes signed and unsigned arithmetic. Where overflow matters, this contract explicitly uses a **conditional 32-bit wrapping model**: arithmetic results retain their low 32 bits, and left shifts discard bits beyond that width. This is not a claim about signed source-language overflow semantics.

## Behavior

### Execution and dependency boundaries

There are no visible conditional branches or loops. Execution follows one apparent path, subject to faults, nonreturning dependencies, and other exceptional transfers.

Every described dependency call remains part of the observable sequence. Calls with no explicit arguments are not assumed to be effect-free. They may inspect or modify memory, registers, and shared state.

Descriptive names below identify captured values. A retained value is not a fresh memory read. Conversely, explicitly repeated reads must not be collapsed across intervening calls.

### 1. Initial payload and reference transfers

The initial dependency receives the second explicit argument. A no-explicit-argument producer follows, and its result is forwarded immediately to another dependency. Further no-explicit-argument calls produce an **initial index**.

The first explicit memory operations are ordered as follows:

1. Store the incoming opaque payload at \(S(\text{initial index})\).
2. Read an **initial descriptor** from  
   \((\text{incoming selector}\gg22)+20\).
3. Call a dependency with four times the initial index, followed by a no-explicit-argument chain producing a **first reference result**.
4. Store four times that reference result at  
   \(S(\text{initial descriptor}\gg1)\).
5. Read a **descriptor payload** from \(\text{initial descriptor}+20\).

The word store in step 4 precedes the word read in step 5. If their locations overlap, that ordering matters.

A dependency then receives the initial descriptor shifted right by one. A producer and immediate forwarder follow, then another result-producing chain. The function stores four times the retained first reference result at the new result-selected slot.

Only after that store does it read the shared 32-bit source, capturing the **early shared sample**.

### 2. Reduced descriptors and zero-valued stores

A scaled-index dependency and further calls precede capture of an **auxiliary index**. The function then:

1. Stores \(\text{descriptor payload}\gg22\) at the auxiliary index’s slot.
2. Calls a dependency with four arguments, in this order:
   - the auxiliary index;
   - the same auxiliary index;
   - \(\text{early shared sample}\gg23\);
   - \(\text{descriptor payload}\gg22\).
3. Executes a producer, immediate forwarder, and result-producing chain.
4. Stores the same reduced descriptor payload at another result-selected slot.
5. Executes a scaled-index call and further calls.
6. Captures a **second auxiliary index** and stores a 32-bit zero at its slot.

A dependency receives the second auxiliary index. After another producer, forwarder, and result chain, the function stores

\[
(\text{early shared sample}\gg23)\gg23
\]

at another result-selected slot.

This value is necessarily zero: the first logical shift of a 32-bit value leaves at most nine significant bits, and the second shift removes them. The two shifts must not be interpreted as a single shift with a count reduced modulo 32.

The next explicit word read captures a **reduced-table word** from

\[
((\text{second auxiliary index}\mathbin{\&}\texttt{0x1fffffff})\gg19)+20.
\]

### 3. Byte-selected updates

Following another scaled-index call and no-explicit-argument chain, the function reads a selector byte at

\[
((\text{early shared sample}\gg23)\gg23)+1,
\]

which is exactly address 1. It then performs

\[
W_8(R_8(1)+17,\ \text{reduced-table word}\gg22).
\]

The destination is between 17 and 272. The stored byte contains bits 22–29 of the reduced-table word.

A four-argument dependency receives the same opaque auxiliary-register value three times, followed by zero. After another call, the next dependency receives an explicit zero. Further calls produce a result, after which the function performs

\[
W_8(R_8(4(\text{reduced-table word}\gg22)+1)+17,\ 0).
\]

This selector-byte read is at an address between 1 and 4093. It is not generally the earlier address-1 read.

The following sequence includes:

- A dependency receiving a scaled result.
- A result-producing chain.
- A dependency receiving another scaled result twice.
- A producer and immediate forwarder.
- One further call.

The function then freshly reads the byte at address 1 and clears the byte selected by that value plus 17. Earlier reads do not determine this fresh selector because intervening dependencies may modify memory.

### 4. First control publication and field-byte copies

The next dependency receives a newly captured auxiliary-register value three times. That captured value is retained for later shifted arguments.

A producer follows. Its result is forwarded together with a previously captured unsigned result shifted right by 23. Further calls produce an index.

The function then:

1. Freshly reads address 1, retaining the unsigned byte as a **small base**.
2. Clears the byte at \(\text{small base}+17\).
3. Calls a dependency with a scaled index.
4. Executes a result-producing chain.
5. Calls a dependency with a scaled result twice.
6. Captures a **published control value** from the next call.
7. Writes that 32-bit value to shared control storage.
8. Passes the same captured control value to a dependency.
9. Executes another call.
10. Reads a word at \(\text{small base}+116\), then writes its low byte at \(\text{small base}+1\).

Step 10 uses the retained small base, not a new read of address 1.

A dependency then receives the retained auxiliary value shifted left by 11. The next dependency supplies an apparent 64-bit result. A following call receives, in order:

- Its low 32-bit half.
- Its high 32-bit half.
- The published control value shifted right by 23.

Further calls yield a result whose fourfold scaling becomes a **field base**. The function reads the word at \(\text{field base}+116\) and copies its low byte to \(\text{field base}+1\).

A dependency receives the field base. Another call chain produces a **link index**. The function then repeats the word read and byte copy at the same field base before the next multi-argument call.

These are two distinct word reads. An intervening dependency may change the word, so the second copy need not match the first.

The next dependency receives:

1. The retained auxiliary value shifted left by 12.
2. Four times the link index.
3. Four times the link index again.
4. The link index shifted left by 18.

### 5. Captured field word and link construction

Additional producer/forwarder chains yield another scaled base. The function:

1. Reads a full **captured field word** at that base plus 116.
2. Copies its low byte to that base plus 1.
3. Calls a dependency with that base.
4. Executes another result-producing chain.
5. Stores the full captured field word at \(16(\text{new index})+20\).

The last address uses a stride of 16, not the stride of four in \(S\).

The next dependency receives

\[
((4\,\text{link index})\gg2)\ll16,
\]

where the fourfold scaling was captured as an unsigned 32-bit value.

Under the conditional wrapping model, this **final argument equals**
\(\text{link index}\ll16\), also evaluated at 32-bit width. Specifically,

\[
(4x\bmod2^{32})\gg2=x\mathbin{\&}\texttt{0x3fffffff}.
\]

The intermediate right shift does not recover the original top two bits. However, a subsequent left shift by 16 discards those bits even if they had been retained. Therefore,

\[
((4x\bmod2^{32})\gg2)\ll16
=
x\ll16
\pmod{2^{32}}.
\]

The loss of information in the intermediate value must not be extended into a claim that these final shifted results differ.

After further calls, the full captured field word is stored at another result-selected slot \(S(\text{result index})\).

Another call chain produces a reference result. Four times that result is stored at

\[
((4\,\text{link index})\mathbin{\&}\texttt{0xfffffffc})+20.
\]

A dependency receives the scaled reference twice. Further producer/forwarder and result-producing calls precede a 32-bit zero store to another result-selected slot.

A subsequent apparent 64-bit result supplies a low half and a high half. Four times the low half is stored at the slot selected by the high half.

The shared source is then read again, capturing a **late shared sample**. This is independent of the early shared sample.

A dependency receives the scaled low half, the high half, and the scaled low half twice more. After further calls, the same scaled low half is stored at another result-selected slot. Only then does the function read a **late payload word** from \(\text{late shared sample}+20\).

### 6. Final byte operations and termination attempt

Another scaled-index call and result chain yield an apparent 64-bit result, split into a **final low half** and **final high half**.

The function reads a selector byte at

\[
(\text{captured field word}\gg23)+1
\]

and writes the low byte of the late payload word to the selected byte value plus 17.

The next dependency receives the final high half three times and four times the final low half as its fourth argument. A producer/forwarder sequence and further calls follow.

The function then reads a selector byte at

\[
((\text{final low half}\mathbin{\&}\texttt{0x3fffffff})\gg21)+1
\]

and stores the low byte of \(\text{late payload word}\gg22\) at the selected byte value plus 17.

Further calls include scaled-result and repeated-scaled-result arguments. A later producer’s result is forwarded. The final producer supplies an 8-bit result and an auxiliary-register destination.

The explicit ending is:

1. Store a zero byte through the auxiliary destination.
2. Form a 32-bit control word with the final producer’s byte in bits 0–7 and a shared 24-bit component in bits 8–31; write it to shared control storage.
3. Store a zero halfword at  
   \((\text{final low half}\mathbin{\&}\texttt{0x3fffffff})\gg7\).
4. Invoke address zero with three explicit zero arguments.
5. Return only if that invocation returns.

The shared 24-bit component is associated with storage beginning at absolute address 9. Its use belongs to construction of the second control publication, after the byte clear. The representation does not establish its exact physical load sequence or atomicity.

Address zero is not established as executable, invalid, terminating, or intentionally mapped.

## Side effects

Visible effects include word stores, byte stores, one halfword store, shared-state reads, and two 32-bit writes to shared control storage. Unresolved dependencies may add arbitrary effects consistent with their unknown interfaces.

No visible operation establishes:

- Bounds, null, or alignment validation.
- An error-status branch or recovery path.
- Transactional execution or rollback.
- Allocation or release semantics.
- Atomic publication, locking, or synchronization.
- Ownership transfer or a retained-reference protocol.

Each access requires the relevant memory to be accessible when reached. A fault or nonreturning dependency may leave all earlier writes committed.

Aliasing is not excluded. Explicit writes may affect subsequent reads, shared control components, or state consulted by dependencies. Repeated shared reads and repeated field reads therefore remain distinct events.

Passing an address-like value does not establish that a dependency owns it, retains it, releases it, or preserves the corresponding storage.

## Dependencies

The following opaque IDs identify dependency roles for this contract. Grouping sites by role does **not** establish that they share a target.

| Opaque ID | Established role |
|---|---|
| F-d10000000001 | Initial link-derived target receiving the second explicit argument |
| F-d10000000002 | No-explicit-argument sites, including 32-bit and apparent 64-bit producers |
| F-d10000000003 | Immediate forwarding sites, sometimes receiving additional arguments |
| F-d10000000004 | Sites receiving indices, scaled values, or repeated scaled values |
| F-d10000000005 | Multi-argument sites receiving repeated auxiliary-register values |
| F-d10000000006 | Sites receiving retained shifted values and link-construction arguments |
| F-d10000000007 | Late producers and forwarders leading to the final byte result and destination |
| F-d10000000008 | Address-zero target receiving three zeros |

For all unresolved dependencies, identity, register preservation, memory effects, failure behavior, ownership, and return guarantees are **documented_only**. Apparent 64-bit results may represent a register-pair interface rather than an independently established scalar return convention.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Unused explicit argument.** With identical relevant entry state and dependency behavior, changing only the first explicit argument changes no explicit argument use visible in the body. This does not prove calling-convention equivalence.

2. **Successive logical shifts.** An early shared sample of `0xffffffff` becomes 511 after shifting right by 23, then zero after the second shift. The corresponding word store is zero, and the associated selector lookup reads address 1.

3. **Byte truncation.** If the reduced-table word shifted right by 22 equals `0x3ff`, the selected byte receives `0xff`.

4. **Distinct field reads.** If the first field-word read returns `0x11223344` and an intervening dependency changes the word to `0x55667788`, the two byte copies store `0x44` and `0x88`.

5. **Retained full word.** A captured field word of `0x89abcdef` produces an immediate byte copy of `0xef`; its later word transfers retain `0x89abcdef`.

6. **Fresh shared source.** If the shared source changes between its early and late reads, the late payload address uses the late captured sample.

7. **Intermediate loss, final equality.** Under wrapping arithmetic, a link index of `0xc0001234` scales to `0x000048d0`. Shifting that value right by two produces `0x00001234`, not the original index. Nevertheless, both that intermediate value shifted left by 16 and the original index shifted left by 16 produce `0x12340000`.

8. **Final control assembly.** If the final producer’s byte is `0xab` and the captured 24-bit component is `0x123456`, the final control word is `0x123456ab`. This describes numeric bit placement, not memory byte order.

9. **Final fault ordering.** If the address-zero invocation faults after preceding accesses succeed, the auxiliary byte clear, final control publication, and halfword clear have already occurred in that order. No rollback is specified.

## Uncertainties

The following remain **documented_only**:

- Actual calling convention and completeness of the apparent interface.
- Which dependency executions define auxiliary-register outputs.
- Semantic types and valid ranges of indices, descriptors, references, and bases.
- Whether low addresses denote ordinary memory, mapped devices, or a misrepresented address space.
- Shared-storage overlap and aliasing with other accesses.
- Target overflow, alignment, byte-order, and fault rules beyond the conditional model.
- Dependency modification of globals, retention of arguments, or memory release.
- Physical access granularity and atomicity for the shared 24-bit component.
- Whether the final invocation returns, faults, terminates, or transfers control elsewhere.

The corrected shifted-link equality is established algebraically **within** the wrapping model; it does not resolve the broader uncertainty about target arithmetic.

## Evidence

The sole evidence is the supplied static function representation. It supports the visible access sequence, widths, argument relationships, captured-value reuse, and absence of visible branching.

The double-shift zero, byte truncations, and shifted-link equality follow from the stated widths and arithmetic assumptions. The shifted-link equality accounts for both intermediate information loss and the final left shift.

No dependency implementation, runtime trace, memory mapping, or execution result was supplied. No runtime tests or implementation validation have been performed.
