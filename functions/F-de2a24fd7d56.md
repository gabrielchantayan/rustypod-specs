<!-- Trusted provenance: F-de2a24fd7d56; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-de2a24fd7d56

## Purpose

F-de2a24fd7d56 describes a state-mutating operation whose established effects are dependency invocations and mixed-width memory accesses. The supplied representation does not establish an application-level purpose. No parser, allocator, serializer, collection, or device abstraction is assumed.

This contract preserves distinctions that affect observable behavior: access widths, separately evaluated reads, captured values, repeated writes, result-component relationships, and calls that may modify subsequently accessed memory.

The evidence is static and incomplete. The supplied representation reports invalid instruction data, removed unreachable control flow, and overlapping shared storage. Consequently, this document specifies represented behavior conditional on reachability; it does not establish a complete executable replacement.

No runtime tests have been performed.

## Interface

### Explicit inputs and output

The represented interface has four positional inputs and no declared return value.

| Position | Width and interpretation | Established direct use |
|---|---|---|
| First | Opaque 32-bit quantity | No explicit use in the supplied body |
| Second | Unsigned 32-bit quantity | Repeated dependency arguments and a logical right shift by 10 that produces a memory base |
| Third | Opaque 32-bit quantity | No explicit use in the supplied body |
| Fourth | Signed 32-bit quantity | Left shift by 3 for the first dependency invocation |

Absence of explicit references to the first and third inputs does not prove independence at the underlying executable level.

No success value, error value, returned object, or ordinary-return postcondition is established. The visible endpoint is an invalid-instruction termination marker.

### Implicit inputs

The operation also depends on:

- An incoming unsigned 32-bit payload.
- An incoming address-bearing value used for initial selector reads.
- Incoming link-related state used to derive the first dependency target.
- Primary results and secondary-register outputs from dependencies.
- Represented 64-bit dependency results.
- Absolute shared storage and memory reached through computed addresses.

Secondary-register outputs supply addresses, indices, payloads, and arguments. Their actual calling convention and initialization are **documented_only**.

### Memory widths and access-site layouts

A byte is 8 bits, a halfword is 16 bits, and a word is 32 bits. Represented wide results are 64 bits, divided into low and high 32-bit components.

| Address basis | Established accesses |
|---|---|
| Second input logically shifted right by 10 | Word reads at offsets 0 and 8; address at offset 20 passed to a dependency |
| High component of one captured wide result | Word reads at offsets 8 and 12, in that order |
| Low component of that same result | Later word read at offset 4 |
| High component of a different captured wide result | Byte store at offset 28, with the byte taken from the low eight bits of that **same result** |
| Another low component, adjusted by 12 | Later word read at offset 4 from the adjusted address |
| Four times an index, plus 20 | Recurrent word reads and stores |
| Sixteen times an index, plus 20 | Recurrent word stores |
| Derived addresses plus 52 | Two pairs of consecutive word stores |

These access sites do not establish a common object layout. The adjusted-base read is at offset 16 from its unadjusted base, provided the address calculations have a valid interpretation.

Byte order, alignment requirements, address mapping, and ordinary-memory versus device-memory behavior are unknown.

### Preconditions and ownership

Each reached access requires memory valid for its width and access direction. Each reached invocation requires an executable target and a compatible actual calling convention. No visible validation establishes these conditions.

The representation converts shifted inputs and dependency results into addresses without rejecting small values first.

No allocation, release, ownership transfer, borrowing lifetime, retained-reference policy, or reference-count behavior is established. Passing an address to a dependency does not establish whether that dependency retains it, mutates it, or invalidates its storage.

## Behavior

### Ordering and reachability

The supplied body presents a long forward sequence without an explicit ordinary loop or conditional branch. The reported control-flow defects prevent treating this as proof that executable control flow is complete.

Separately sequenced accesses retain their represented order. Every later effect is conditional on earlier accesses completing and intervening dependencies returning. No transaction or rollback is visible.

Repeated references are independent observations unless a value is explicitly captured and reused. Capturing an address does not capture the memory at that address.

Multiple memory references within one invocation’s argument list remain distinct. Their relative evaluation order is not established. Dependencies may modify globals or pointed-to memory; no preservation guarantee is assumed.

### Initial selectors and input-derived memory

The first represented effect reads an unsigned selector byte at offset 1 from the incoming address-bearing state. It then stores the low eight bits of the incoming payload at the absolute address equal to that selector plus 17.

The destination therefore lies between 17 and 272 inclusive. It is not an offset within the selector’s containing object.

The first dependency invocation follows the store. Its target is derived from incoming link-related state. It receives three copies of the second input and the fourth input shifted left by 3. Because the fourth input is signed, negative operands and nonrepresentable shifted values have no established modular interpretation.

After intervening invocations, the initial selector-source expression is read again. The new selector chooses the destination for a byte formed from the incoming unsigned payload logically shifted right by 22. Neither the selector nor its selected destination is assumed unchanged.

A subsequent dependency supplies a represented 64-bit result. The second input, logically shifted right by 10, then supplies a memory base. Word reads occur at offsets 0 and 8, in that order. A dependency subsequently receives:

- The wide result’s low 32 bits.
- Its high 32 bits.
- The input-derived base plus 20.
- Another copy of the low 32 bits.

The captured offset-0 word later supplies a selector-source address through multiplication by 1,024 and addition of 1. The captured offset-8 word supplies byte payloads through its low eight bits and, later, through a logical right shift by 22. Intervening calls separate those selector observations.

### Recurring selector-byte operations

Many later effects read an unsigned selector byte from a computed address, then store one byte at the absolute address equal to the selector plus 17.

Selector-source calculations include scales of 2, 4, 8, 16, 64, 1,024, 4,096, and larger powers of two, together with masks, shifts, and addition of 1. These calculations retain their represented widths and signedness.

Payloads include:

- Zero.
- Low bytes of captured words and dependency results.
- Low bytes after left shifts by 2.
- Low bytes after logical right shifts by 22.
- A 29-bit-masked value logically shifted right by 19.
- A 30-bit-masked value logically shifted right by 20.
- A 22-bit-masked value logically shifted right by 12.
- A captured 64-bit result logically shifted right by 54.

Final byte conversion retains only eight bits. For example, masking to 30 bits and shifting right by 20 can leave ten significant bits before truncation.

The shift by 54 operates on the full unsigned 64-bit result. Its stored byte corresponds to original result bits 54 through 61; it is not a 32-bit shift.

Absolute address 1 is repeatedly read as a selector source. Each reference is independent. One dependency also receives two separately expressed byte reads from address 1 as arguments, without an established ordering between those argument evaluations.

### Indexed word staging

Word destinations repeatedly use four times an index plus 20. Other destinations use sixteen times an index plus 20. These scales are not interchangeable.

Early staging includes:

- A zero word stored at a secondary-output-derived indexed destination.
- That secondary output later stored at another indexed destination.
- A four-times-scaled result stored at a sixteen-times-scaled destination.
- A word captured from a four-times-scaled address plus 20.
- The captured word later stored after a logical right shift by 11.
- A separate word read whose address is the captured word logically shifted right by 22, plus 20.

Later regions stage retained values, secondary outputs, masked values, and quantities shifted right by 12. No evidence establishes array, link, handle, or encoded-pointer semantics.

Near the end, a wide result’s high component supplies an index for a zero word store and is later stored at another indexed destination. Its low component supplies an unsigned destination calculation scaled by 268,435,456, plus 20, and a payload scaled by 67,108,864. A 29-bit-masked portion of that low component is subsequently stored after a logical right shift by 9.

The final region repeats sixteen-times-scaled staging and four-times-scaled stores. One invocation contains two independent word references to the same derived address plus 20: one supplies a right-shift-by-12 argument, the other a left-shift-by-2 argument. Their equality and evaluation order are not established. A later captured word is separately reused after a right shift by 12.

### Shared storage at addresses 124 and 40

A 32-bit dependency result is stored at absolute address 124 before a dependency receives two copies of that retained result. Those arguments are not represented as rereads of address 124. The dependency may nevertheless modify that storage.

Address 40 has three distinct word references:

1. An initial read captures a value for a later indexed store.
2. A subsequent invocation contains two additional, separately expressed word reads from address 40.
3. After that invocation and another invocation, the initial capture is stored at a secondary-output-derived indexed destination.

The initial capture precedes both argument references. No ordering between the two argument references is established, and neither is a reuse of the initial capture.

The later store therefore uses the initial observation even if address 40 subsequently changes. Environmental changes or side-effecting memory reads can distinguish all three observations. A mutation performed inside the invoked dependency occurs after its argument evaluations.

### Two observations of an unresolved shared word

A later shared word has no established absolute identity. It is read twice consecutively after an indexed store:

- The first 32-bit observation is retained for a later mask-and-shift argument.
- The second independently supplies multiplication by four, establishing a selector-source base.

No call intervenes between these reads. That does not establish equality or memory stability.

After intervening calls and another indexed store, a selector is read from the second observation’s scaled base plus 1. The selected byte destination receives a retained dependency result shifted left by 2 and truncated to a byte.

A following invocation receives the first observation masked to 30 bits and logically shifted right by 10. Later, the selector-source address derived from the second observation is read again. Its newly observed selector chooses the destination for the same retained dependency result masked to 30 bits and shifted right by 20, then truncated to a byte.

The second observation’s scaled base also feeds later signed shifts and multiplication. Two separately expressed word reads from a derived address plus 28 subsequently supply invocation arguments. They are not a single captured read.

Overflow and signed-shift interpretation in these downstream calculations remain **documented_only**. That uncertainty does not merge the earlier independent observations.

### Wide results and dependent memory

Different wide-result uses must remain distinct.

For one captured wide result, the high component supplies a base for word reads at offsets 8 and 12, in that order. The low component is retained and later supplies a word read at offset 4. Intervening calls may change the accessed memory.

For another captured 64-bit dependency result, a direct byte store combines components of **that same captured result**:

- Its high 32 bits supply the address base.
- The destination is that base plus 28.
- Its low eight bits supply the stored byte.

This store is not selector-mediated. It does not combine the low byte of one dependency result with the high component of another. It occurs before the following invocation, which receives two copies of the retained low 32-bit component.

Other wide-result uses forward low and high components as separate arguments, sometimes repeating the low component. Another low component is adjusted by 12, forwarded with its corresponding high component, and retained for a later word read at offset 4 from the adjusted address.

### Consecutive writes and zero-derived addresses

Two word-store pairs preserve order and multiplicity:

- At one derived address plus 52, a captured selector is stored, immediately followed by that selector shifted left by 1.
- At another derived address plus 52, the same left-shifted word expression is stored twice.

Even if stable ordinary memory exposes only the final value, the represented access trace contains both writes.

A quantity explicitly assigned zero later supplies:

- A word-store destination through multiplication by eight: absolute address zero.
- A halfword-store destination through a 31-bit mask and logical right shift by 11: also absolute address zero.
- A byte-store destination through four times zero plus an independently derived offset: the offset alone.

The last offset comes from the low component of an earlier wide result, masked to 30 bits and logically shifted right by 7. Its byte payload comes from a later wide result. This is a separate cross-result relationship, unlike the same-result byte store at high-component base plus 28.

### Arithmetic, errors, and termination

Unsigned 32-bit arithmetic retains the low 32 bits. Unsigned right shifts are logical. Narrow stores retain the low 8 or 16 bits.

Signed multiplications, left shifts, and address additions do not have a blanket established wraparound interpretation. Their affected cases remain unresolved rather than being silently evaluated with arbitrary-precision or unsigned arithmetic.

No explicit argument rejection, bounds check, null check, error-code return, cleanup, or rollback path is visible.

The represented endpoint is an invalid-instruction marker. It does not identify a processor exception, operating-system signal, or normal-return alternative.

## Side effects

Established represented effects include:

- Selector reads and selector-derived byte writes.
- Direct byte writes, including the same-result high-component-plus-28 store.
- Computed word reads and writes.
- Absolute shared-storage accesses.
- A word write and a halfword write to address zero.
- Numerous dependency invocations with unknown additional effects.

Shared storage may overlap narrower storage. Byte writes may affect later word observations, and word writes may affect later byte reads. No nonaliasing guarantee is supplied.

No atomicity, synchronization, thread-safety, reentrancy, or cross-thread ordering guarantee is established. Dependencies cannot be assumed to preserve globals, retain storage validity, or avoid ownership changes.

## Dependencies

The supplied identity mapping establishes these dependencies in relative invocation order. Other invocations and memory operations occur between them.

| Dependency | Represented argument relationship |
|---|---|
| F-23325cd5e50c | One retained 32-bit result |
| F-bbb54b5f036d | Two copies of one retained 32-bit result |
| F-f80b3ac065fc | Low and high components of one wide result, followed by two further copies of its low component |
| F-de15789688a8 | Two copies of one 32-bit result |
| F-ea83f9a17404 | Two copies of one 32-bit result |
| F-674e75a953e8 | Two copies of one 32-bit result |

Other fixed-target and computed-target dependencies lack supplied opaque identities. Their identities remain **documented_only**; no identity is assigned by resemblance.

For every dependency, purpose, accepted domain, actual result convention, secondary-register behavior, global mutations, ownership effects, and failure behavior remain unknown.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

### Shifted input base

If the second input is 4,096, its logical right shift by 10 is 4. If execution reaches the relevant region and memory permits access, word reads occur at addresses 4 and 12, in that order. The subsequent dependency receives address 24 as its third argument.

Using 4,096 as the memory base would not match the represented calculation.

### Selector rereading and truncation

A selector value of 7 chooses absolute destination 24. A payload whose low byte is 52 stores byte value 52 there.

If a subsequent selector read from the same source yields 9, that later store targets 26. Reusing the earlier selector would change behavior.

### Same-result wide byte store

Suppose the wide result captured for the direct byte store has high component 4,096 and low component hexadecimal 123456AB. Assuming valid address arithmetic and writable memory, the store writes byte value 171 at address 4,124.

Both address base and payload must come from this capture. A high component retained from another result cannot replace 4,096. The following invocation receives two copies of the captured low 32-bit component, not two copies of byte value 171.

### Address-40 observations

Suppose the initial word read at address 40 yields 100, while the two argument references yield 101 and 102 in an environment-permitted evaluation order.

The dependency receives each argument reference’s observation in its corresponding argument position. The later indexed store still receives 100. No left-to-right argument-read order is required.

### Independent late shared-word reads

If the first late shared-word observation is 1,024 and the second is 3, the later 30-bit-mask/right-shift-by-10 argument is 1. The selector-source base is 12, making the selector address 13.

Substituting the first observation into the selector calculation would incorrectly produce 4,097. This example establishes nothing about completion of downstream signed arithmetic.

### Duplicate stores and faults

For a retained selector of 7, the first store pair writes words 7 and then 14 at the same address. A single write of 14 is not the same represented trace. The identical-store pair likewise retains two writes.

If the word write to address zero faults, no later effect is guaranteed. If execution continues, the later halfword write still targets zero with a different width. Skipping either write or reporting ordinary successful return is not established.

## Uncertainties

The following remain **documented_only**:

- Fidelity, complete boundaries, and reachability of the represented operation.
- Initialization and meaning of implicit incoming state.
- Actual wide-result and secondary-register conventions.
- Unmapped dependency identities and dependency semantics.
- Signed-overflow, signed-shift, and affected address behavior.
- Alignment, byte order, address-space rules, and low-address accessibility.
- Memory stability, aliasing, and side-effecting reads.
- Ownership, retention, allocation, and release behavior.
- Application-level purpose.
- Actual terminal behavior and any ordinary-return path.

These unknowns do not justify merging reads, deleting duplicate writes, changing result provenance, or inventing validation and cleanup.

## Evidence

**Established static evidence:** the supplied body represents four 32-bit inputs, implicit incoming state, mixed-width accesses, split 64-bit results, secondary-register-dependent operations, mapped and unresolved dependencies, repeated stores, and an abnormal endpoint.

The address-40 region contains an initial capture followed by two separate argument references. The later unresolved shared-word region contains two successive observations with different downstream uses. The direct byte store at high-component base plus 28 derives both its base and payload from the same captured 64-bit dependency result.

**Evidence limitations:** reported invalid instruction data, removed unreachable control flow, and overlapping storage limit conclusions about executable fidelity and completeness. Comments, labels, prior documents, and reviewer feedback do not independently establish behavior.

**Inferred intent:** recurring addressing and invocation patterns may reflect a shared execution or storage convention. No application-specific interpretation is adopted.

**Verification status:** static examination of the supplied representation only. No implementation access, execution, or runtime tests were performed.
