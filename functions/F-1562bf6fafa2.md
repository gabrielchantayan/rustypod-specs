<!-- Trusted provenance: F-1562bf6fafa2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1562bf6fafa2

## Purpose

F-1562bf6fafa2 performs an ordered sequence of dependency invocations, selector-directed byte updates, and computed-address 32-bit transfers. The supplied static representation does not establish an application-level purpose such as allocation, serialization, object construction, or device initialization.

The supported contract is a conditional trace: each later event requires execution to reach it, preceding dependencies to return, and required memory accesses to succeed. Calls and direct writes are not established as atomic or reversible.

The represented path ends at an invalid-instruction boundary. No ordinary return or success result is visible. Warnings about invalid instruction data and overlapping global storage limit confidence in the recovered interface and control flow.

## Interface

### Apparent arguments

| Position | Represented width | Established explicit use |
|---|---:|---|
| First | 32 bits; signedness unspecified | Passed to the first indirect invocation. |
| Second | 32 bits; signedness unspecified | No explicit use in the supplied body. |
| Third | Signed 32 bits | Shifted left by 26 and supplied as an invocation’s first argument. |
| Fourth | Signed 32 bits | Shifted left by 10 and supplied as an invocation’s third argument. |

The signed representations do not establish portable signed-shift behavior. The second argument’s lack of explicit use does not prove irrelevance to the actual machine interface.

### Implicit inputs

The behavior also depends on:

- Incoming control state used to calculate the first indirect target.
- Two incoming machine-state values used in early selector-address calculations.
- Primary and secondary results associated with dependency activity.
- Two separate observations of one global 32-bit storage location.
- Memory at computed byte and word addresses.

These are not established additional public parameters. The four apparent arguments alone are insufficient to reproduce the trace.

### Outputs and preconditions

There is no visible ordinary return value. Observable outputs comprise dependency invocations, direct memory accesses, and whatever effects dependencies produce.

A reached prefix requires executable targets, compatible machine interfaces, accessible memory, and any platform-required alignment. No visible validation establishes these conditions. The terminal invalid-instruction boundary does not establish a particular exception, termination mechanism, or recovery behavior.

## Behavior

### Widths and address rules

Selector accesses read unsigned 8-bit values. A selector chooses the absolute byte address equal to its value plus 17: addresses 17 through 272 inclusive. A selector-directed write changes one byte, retaining only the low eight bits of the supplied value.

Word accesses are represented as 32-bit reads or writes. Endianness and pointer width are unresolved. Computed addresses are not established object-relative fields.

Two recurring word-address forms are distinct:

- An indexed word address is four times an index plus 20.
- A link-write address is sixteen times an index plus 20.

Writing at the latter does not establish contents at the former.

Unsigned 32-bit arithmetic reduces modulo \(2^{32}\); unsigned right shifts are logical. Relevant extractions include:

- Right shift by 12: at most 20 significant bits remain.
- Right shift by 22, then byte narrowing: original bits 22–29.
- Mask to 30 bits, right shift by 20, then byte narrowing: original bits 20–27.
- Mask to 30 bits, then right shift by 10: a 20-bit result.
- Mask to 29 bits, then right shift by 9: a 20-bit result.

Some scaled addresses and left shifts use signed representations. Machine-width wraparound is a conditional interpretation for those expressions, not an established portable signed-arithmetic guarantee.

### Ordering model

The phases below preserve direct-access order and recovered explicit argument relationships. Unresolved indirect calls, including calls without recovered explicit arguments, separate many events. Such calls are not necessarily argument-free at the machine interface.

Dependencies may modify memory, globals, or machine state, fail, or not return. Repeated reads remain distinct observations. Evaluation order between separate argument expressions within one invocation is unresolved.

Descriptions of reused secondary-result words identify relationships in the supplied representation; they do not establish a general secondary-result preservation convention.

### Entry and first captured word

The first indirect invocation receives the first apparent argument. Further indirect calls produce a result used later in scaled form.

A selector is read from one plus an incoming machine-state address. Its selected byte destination is cleared. The next value-bearing invocation receives the earlier dependency result shifted left by 2.

After further calls, an invocation receives, in order:

1. The third apparent argument shifted left by 26.
2. The retained selector from the clear.

A subsequent dependency produces the first captured unsigned word. Another invocation receives that word twice, followed by the fourth apparent argument shifted left by 10.

After another call, a selector is read from one plus an incoming unsigned state value shifted right by 9. Its selected byte destination is cleared.

### Selector update preceding the second captured word

The first captured word shifted right by 12 supplies a retained selector-read base.

Before the selector read at that base, an invocation receives a secondary-result word as its sole recovered argument. Further calls follow. A selector is then read at the retained base plus 1, and its destination receives the low byte of that same represented secondary-result word shifted right by 22.

This argument-to-write relationship is part of the contract; the earlier invocation is not unspecified intervening activity.

The next value-bearing invocation receives a newly returned result shifted left by 2. Following further calls, another receives the retained selector first and an explicitly zero-valued argument second.

A dependency then produces the second captured unsigned word. A subsequent invocation receives that word twice, followed by the first captured word shifted left by 10.

After another call, the second captured word is stored at the indexed word address selected by a secondary-result index.

### Two global observations and their uses

Immediately after that word store, the function performs two separate 32-bit reads of the same global storage location:

1. The first observation is retained for a later masked-and-shifted argument.
2. The second observation is multiplied by four to form a retained address base.

No dependency invocation is represented between these reads. Their equality is nevertheless not guaranteed: ordinary, stable, nonvolatile storage semantics are not established.

After both global observations, an invocation receives the secondary-result index used for the preceding second-captured-word store. The index argument therefore occurs after the store and both global reads, not before them.

Further calls produce a fresh index. The second captured word shifted right by 12 is stored at that index’s indexed word address. A following invocation receives four times this fresh index.

A later dependency supplies another unsigned word. A selector is read at the scaled second-global-observation base plus 1. Its destination receives the low byte of four times the newer word.

An invocation then receives the first global observation masked to 30 bits and shifted right by 10. Further calls yield a result retained across the next byte update.

The selector is read again at the scaled second-global-observation base plus 1. This fresh selector chooses the destination receiving the newer word masked to 30 bits, shifted right by 20, and narrowed to one byte.

After that high-field byte write:

- An invocation receives the result obtained before the fresh selector read, shifted left by 2.
- Further calls produce another result.
- An invocation receives that newer result shifted left by 2 as its first argument and the freshly retained selector as its second argument.

Thus the second selector is both used for the high-field write and subsequently passed onward. It is not interchangeable with the first selector.

### Shift-derived addresses and identified dependency

The retained scaled second-global base is shifted left by 26. After the two-argument invocation just described and another call, an invocation receives two separately represented 32-bit reads from this shifted address plus 28.

Both arguments read the same address, but equality of the observed words and their evaluation order are unresolved.

The shifted base is also multiplied by 1024 to obtain another retained address base. Under 32-bit wraparound interpretation, the combined left shifts total 36 positions and produce zero. Because signed expressions participate, that simplification is conditional.

After another call, a selector is read at the computed base plus 1, and its destination is cleared.

Immediately after this clear, the next invocation receives three recovered arguments, in order:

1. Zero.
2. A secondary-result word.
3. The computed address base used for the clear’s selector read.

Following further calls, the selector is read again at that same computed base plus 1. Its newly selected destination receives the low byte of the same represented secondary-result word shifted right by 22.

The clear’s selector is not reused for this data write. The three-argument invocation precedes the fresh selector read and may have effects relevant to that read.

A following invocation receives a newly returned result shifted left by 2. After further calls, another receives the computed base shifted left by 26 first and another returned result shifted left by 2 second.

A dependency then produces a captured 32-bit value. F-674e75a953e8 receives this value in both apparent argument positions.

After another indirect call, the captured value is stored at an indexed word address selected by a secondary-result index. An invocation receives that index first and its masked-to-30-bits, right-shifted-by-10 form second.

Further calls produce a new index. Its indexed word destination receives the retained secondary-result index. A following invocation receives four times the new destination index.

### Further byte updates and paired result

A later dependency produces another unsigned word. A selector read at eight times the value passed to F-674e75a953e8, plus 1, selects the destination receiving the low byte of four times this newer word.

An invocation receives four times the newer word. After further calls, a selector read at 64 times that word plus 1 selects the destination receiving its masked-to-30-bits, right-shifted-by-20 byte form.

Following calls include:

- An invocation receiving a returned result shifted left by 2.
- An invocation receiving another returned result shifted left by 2 first and the retained selector second.
- An invocation receiving two copies of the earlier value passed to F-674e75a953e8, each shifted left by 10.

A subsequent clear reads its selector at an explicitly zero-valued base multiplied by \(2^{26}\), plus 1. Its represented selector-read address is therefore 1.

The next value-bearing invocation receives a secondary-result word twice, followed by sixteen times that word. After intervening calls, a selector is read at sixteen times that secondary-result word plus 1. Its destination receives the low byte of the word shifted right by 22.

Following calls pass a returned result shifted left by 2, then later pass the retained selector first and another returned result shifted left by 2 second.

An invocation receives two explicitly zero-valued arguments. The following dependency produces a represented 64-bit result, separated into high and low 32-bit halves:

- Zero is stored at the indexed word address selected by the high half.
- The high half is passed to an invocation.
- After further calls, a fresh indexed word destination receives the retained high half.
- A following invocation receives four times that fresh destination index.

Whether the 64-bit quantity is a conventional return value or a representation of paired machine results remains unresolved.

### Low-half storage and indexed links

The low half determines a word write at \(2^{28}\) times that half plus 20. The stored value is \(2^{26}\) times the low half. These unsigned products reduce modulo \(2^{32}\).

An invocation receives the stored value. Following further calls, a newly indexed word destination receives the low half masked to 29 bits and shifted right by 9. A subsequent invocation receives four times this destination index.

Five link operations follow. Each obtains an index, stores four times that index at the link-write address of sixteen times the index plus 20, and then performs additional propagation:

| Link | Established behavior after its link write |
|---|---|
| First | An invocation receives a word read at four times the link index plus 20, shifted right by 12; the four-times-scaled link index; and a separate word read from that same address, shifted left by 2. A later indexed destination receives the scaled link index. |
| Second | An invocation receives the scaled link index twice, then the retained low half shifted left by 26. A later indexed destination receives the scaled link index. |
| Third | The scaled link index is passed to an invocation and later stored at a fresh indexed destination. |
| Fourth | A word is captured from four times the link index plus 20 before further calls. The scaled link index is passed onward. A fresh indexed destination later receives the captured word shifted right by 12. |
| Fifth | An invocation receives three values: the fourth link’s destination index multiplied by four, masked to six bits, and shifted left by 14; the fifth scaled link index; and the fourth link’s destination index shifted left by 30. A later indexed destination receives the fifth scaled link index. |

After each propagated indexed store, an invocation receives four times its destination index.

The fifth link’s first argument masks an already four-times-scaled value. It does not extract six unrestricted low bits of the original destination index.

### Final byte updates and boundary

A dependency supplies a final unsigned word. A selector is read at \(2^{28}\) times the retained destination index from the fourth-link phase, plus 1. Its destination receives the low byte of the final word shifted left by 2.

An invocation receives that shifted value twice, followed by the final word shifted left by 4. Another dependency yields a 32-bit result, which is forwarded to the next invocation.

After further calls, the selector is read again at the same retained-index-derived address. Its newly selected destination receives the final word masked to 30 bits, shifted right by 20, and narrowed to one byte.

An invocation receives a returned result shifted left by 2. After further calls, another receives a newly returned result shifted left by 2 first and the retained final selector second. One further indirect invocation precedes the invalid-instruction boundary.

No cleanup, rollback, success indication, or normal return is visible.

## Side effects

### Memory and aliasing

Direct effects include selector reads, byte clears and data writes, two global word reads, and computed-address word reads and writes.

No disjointness guarantee is established. Byte destinations may overlap selectors, globals, or word storage. Word destinations may overlap each other. Earlier writes can therefore affect later reads even without dependency activity.

Repeated selector reads cannot be collapsed. Separate word reads used as invocation arguments cannot be replaced by one captured observation without an additional stability guarantee.

### Ownership and lifetime

No direct allocation, deallocation, reference-count operation, ownership transfer, or lifetime extension is established.

Passing a value to a dependency does not establish whether it denotes storage or whether that storage is retained, consumed, released, or modified. No dependency is assumed to preserve globals or own or release memory.

### Errors and partial effects

No explicit bounds check, null check, error return, retry, or recovery branch is visible. If an access or dependency fails, earlier completed effects are not visibly undone. Neither transactional behavior nor a defined failure result is supported.

## Dependencies

### F-674e75a953e8

One explicit invocation receives the same captured 32-bit value in both apparent argument positions. No ordinary return value is explicitly consumed.

Its calling convention, implementation, global effects, memory effects, ownership behavior, and failure behavior are **documented_only**.

### Unresolved indirect dependencies

No opaque function identities are supplied for the remaining indirect targets; none are invented here. Targets derive from incoming control state or code-associated address expressions.

Their represented outputs include primary words, indices, secondary-result state, and a paired 64-bit quantity. Recovered argument relationships are specified above, but complete machine interfaces are unknown.

Purity, determinism, successful return, memory preservation, and ownership contracts are **documented_only**.

## Acceptance examples

These are conditional static expectations, not executed tests. Each assumes the relevant event is reached and accesses succeed.

1. **Selector bounds.** A selector of 200 chooses byte address 217. A clear changes exactly one byte there.

2. **Early secondary-result relationship.** If the represented secondary-result word is `0x00800000`, the invocation before the first-captured-word-derived selector read receives that word. The subsequent right-shift-by-22 byte write stores 2 at the selected destination.

3. **Separate global observations and index forwarding.** If the global observations are `0x00000400` and `0x00000800`, the masked right-shift-by-10 argument is 1, while the repeated selector address is `0x00002001`. If the preceding second-word store used index 7, the index-forwarding invocation receives 7 after both global reads.

4. **Fresh global selector passed onward.** Suppose the two global-base selector reads yield 4 and 12. The byte writes target addresses 21 and 29. The later two-argument invocation uses 12 as its second argument, alongside the newly returned result shifted left by 2.

5. **Clear, three-argument call, fresh selector.** If the computed base is zero, the clear reads address 1. Suppose it observes selector 3, while the later read observes 9. The clear targets address 20. Between these reads, the immediate invocation receives zero, the represented secondary-result word, and zero. For secondary word `0x00800000`, the later write stores 2 at address 26.

6. **Byte narrowing.** Right-shifting `0xffc00000` by 22 yields `0x3ff`; the byte store retains `0xff`. Masking `0x3ff00000` to 30 bits and shifting right by 20 also yields `0x3ff`, narrowed to `0xff`.

7. **Identified dependency.** A captured value of `0x12345678` supplies both arguments to F-674e75a953e8. This imposes no preservation or ownership requirement.

8. **Distinct link locations.** For link index 3, value 12 is stored at address 68. A subsequent read at four times the index plus 20 accesses address 32, not 68.

9. **Scaled mask.** If the fourth-link destination index is 17, four times it is 68. Masking that scaled value to six bits gives 4; shifting left by 14 gives 65536.

10. **Partial execution.** If a dependency does not return after a completed clear, no subsequent effect is required. No visible mechanism restores the byte.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose, intended data model, and object layouts.
- Actual entry, exit, and dependency calling conventions.
- Validity of recovered control-flow and call boundaries.
- Origins, lifetimes, and preservation rules for implicit machine-state inputs.
- Signed overflow, signed shifts, and unsigned-to-signed conversions.
- Pointer width, alignment, endianness, address interpretation, and memory-map validity.
- Stability and access semantics of global and computed-address storage.
- Meaning of overlapping global storage and narrower accesses.
- Identities and contracts of unresolved dependencies.
- Interpretation of the represented 64-bit result.
- Runtime consequences of the invalid-instruction boundary.

Repeated extraction and indexed propagation suggest packing or state-linking intent. This is inference only, not evidence of a serialized format, ownership model, or valid execution environment.

## Evidence

Established static evidence is the supplied function body and opaque dependency mapping. It supports the represented widths, direct-access ordering, arithmetic relationships, separate global observations, repeated selectors, explicit dependency arguments, duplicated inputs to F-674e75a953e8, and absence of a visible normal return.

In particular, the representation ties the early secondary-result argument to a later byte extraction; places the preceding store index’s invocation after both global reads; passes the fresh global-base selector alongside a newly scaled result; and places the zero/secondary-result/base invocation immediately after the shift-derived clear.

The invalid-instruction and overlapping-storage warnings are limitations, not proof of a specific runtime outcome.

No external implementation inspection, instruction-level validation, runtime execution, or runtime tests have been performed. The acceptance examples describe conditional consequences of static evidence, not proof that the complete path is executable.
