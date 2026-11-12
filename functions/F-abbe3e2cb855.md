<!-- Trusted provenance: F-abbe3e2cb855; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-abbe3e2cb855

## Purpose

F-abbe3e2cb855 performs an ordered sequence of indirect calls and memory accesses. It derives addresses from dependency results, copies selected word values into other locations, performs byte updates through low-address lookup entries, captures one global value, assigns another global value, and finally attempts a call through address zero.

The supplied static representation does not establish an application-level purpose. In particular, it does not establish allocation, object construction, serialization, device access, or ownership transfer. The contract below describes represented effects without assigning those meanings.

All behavior is conditional on execution reaching the relevant operation. A dependency that does not return, or an access that faults, can prevent subsequent effects.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First | 32 bits, represented as signed | Passed unchanged to F-opaque001; later shifted left by one bit for F-opaque013. |
| Second | 32 bits | No explicit use. |
| Third | 32 bits | No explicit use. |
| Fourth | 32 bits, unsigned | Logically shifted right by 23 bits for F-opaque003. |

The absence of explicit uses for the second and third inputs does not establish that their incoming machine-register values are irrelevant to unknown dependencies. The completeness of the represented calling conventions is unresolved.

### Implicit inputs and state

Execution also depends on:

- Incoming link-register state, which determines the first indirect target.
- Results and side effects of the dependencies.
- Memory at computed addresses.
- A 32-bit global slot, called **Global Capture**, whose contents become a saved address.
- A global slot, called **Global Assignment**, which receives a dependency result near the end.
- An incompletely identified output-register-derived pointer used for a byte clear.
- The execution environment’s treatment of low addresses, unaligned accesses, and a call through address zero.

These global labels are descriptive, not recovered symbol names. The supplied representation warns that global symbols can overlap smaller symbols at the same address. Neither global is therefore assumed disjoint from other storage.

### Results, errors, and completion

No explicit return value is represented.

There is no explicit validation, error result, recovery branch, or rollback. The last represented call targets address zero and receives three zero arguments. The represented return is reached only if that call returns normally.

Whether address zero denotes callable code, causes a fault, or has another platform-specific meaning is **documented_only**. Normal completion is not established.

### Width and arithmetic model

The represented access widths are:

- Byte: 8 bits.
- Halfword: 16 bits.
- Word and address computations: 32 bits.
- Paired dependency results: 64 represented bits, divided into low and high 32-bit halves.

This contract describes left shifts, multiplication, and address addition using their resulting 32-bit bit patterns: bits above bit 31 are discarded. Logical right shifts insert zero bits. A byte store uses only the source value’s low eight bits.

This is a machine-bit-pattern interpretation, not a claim that signed overflow or signed left shifts are defined in an original source language. Those source-language semantics remain unresolved.

Memory byte order is not established. Taking a word’s low eight bits is a numeric truncation; it does not identify the byte at that word’s lowest memory address.

## Behavior

Dependency IDs identify call sites in invocation order, not necessarily distinct implementations. Unless arguments are specified below, the representation supplies no explicit arguments. That omission does not establish that a dependency consumes no implicit machine state.

### Initial extraction and first byte copy

F-opaque001 receives the first input. F-opaque002 then produces a represented 64-bit result. F-opaque003 receives its low 32-bit half, its high 32-bit half, and the fourth input logically shifted right by 23, in that order.

After F-opaque004, F-opaque005, and F-opaque006, F-opaque007 supplies a 32-bit result. Four times that result becomes the **first base**.

A word is read at first base plus 116. Its low byte is written at first base plus 1. The read precedes the write. F-opaque008 then receives the first base.

### Repeated byte copy and input-derived arguments

F-opaque009, F-opaque010, and F-opaque011 run before F-opaque012. Retain F-opaque012’s result as the **original link value**, and retain its two-bit left shift as the **shifted link value**.

The function reads the word at first base plus 116 again and writes its low byte at first base plus 1 again. This is a fresh read after the intervening calls; it is not a reuse of the earlier word.

F-opaque013 receives four arguments:

1. The first input shifted left by one bit.
2. The shifted link value.
3. The shifted link value again.
4. The original link value shifted left by 18 bits.

F-opaque014’s 32-bit result is forwarded to F-opaque015. F-opaque016’s 32-bit result is then forwarded to F-opaque017.

### Captured word and its first propagation

F-opaque018, F-opaque019, and F-opaque020 precede F-opaque021. Four times F-opaque021’s result becomes the **capture base**.

The unsigned word at capture base plus 116 is retained as the **captured word**. Its low byte is written at capture base plus 1, after which F-opaque022 receives the capture base.

F-opaque023, F-opaque024, and F-opaque025 precede F-opaque026. The captured word is written as a full word at sixteen times F-opaque026’s result plus 20.

F-opaque027 receives the shifted link value logically shifted right by two and then shifted left by sixteen, with the final result truncated to 32 bits.

Under the stated arithmetic model, this argument is **always equal to the original link value shifted left by sixteen and truncated to 32 bits**. Forming the shifted link value discards the original value’s highest two bits, but those bits could not survive the final sixteen-bit left shift either. The argument contains the original link value’s low sixteen bits in positions 16–31 and zeros in positions 0–15.

### Further propagation and link storage

F-opaque028’s result is forwarded to F-opaque029. F-opaque030’s result is forwarded to F-opaque031.

After F-opaque032, F-opaque033, and F-opaque034, F-opaque035 supplies a result whose fourfold value becomes the **propagation base**. The captured word is written at propagation base plus 20. F-opaque036 then receives the propagation base.

F-opaque037, F-opaque038, and F-opaque039 precede F-opaque040. F-opaque040’s result shifted left by two becomes the **stored link**.

The stored link is written as a word at the shifted link value, with its lowest two bits cleared, plus 20. F-opaque041 then receives the stored link twice.

The shifted link value already has its lowest two bits clear. The explicit masking does not restore any bits discarded by its earlier left shift.

### Word clear, paired-result propagation, and global capture

F-opaque042’s result is forwarded to F-opaque043. F-opaque044’s result is forwarded to F-opaque045.

F-opaque046, F-opaque047, and F-opaque048 precede F-opaque049. Four times F-opaque049’s result becomes the **clear base**. A zero word is written at clear base plus 20, and F-opaque050 receives the clear base.

F-opaque051, F-opaque052, and F-opaque053 precede F-opaque054, which supplies a represented 64-bit result. Its low half shifted left by two becomes the **propagated value**. Its high half becomes the **paired address component**.

The propagated value is written as a word at four times the paired address component plus 20. Only after that write does the function read Global Capture, retaining its contents as the **saved global address**.

F-opaque055 receives, in order:

1. The propagated value.
2. The paired address component.
3. The propagated value.
4. The propagated value.

F-opaque056, F-opaque057, and F-opaque058 precede F-opaque059. Four times F-opaque059’s result becomes the **second propagation base**. The propagated value is written at second propagation base plus 20.

Next, the unsigned word at saved global address plus 20 is read and retained as the **selected word**. F-opaque060 then receives the second propagation base.

The ordering is significant: the Global Capture read follows the first propagated-value store; the selected-word read follows the second propagated-value store. If these locations alias, those stores can affect the observations. Later changes to Global Capture itself do not replace the already saved address.

### First lookup-directed byte update

F-opaque061, F-opaque062, and F-opaque063 precede F-opaque064, which supplies another represented 64-bit result. Retain its halves as the **final low half** and **final high half**.

The first lookup address is the captured word logically shifted right by 23, plus one. An unsigned byte is read there. The function then writes the selected word’s low byte at the absolute address equal to that fetched byte plus 17.

The lookup address lies between 1 and 512 inclusive. The destination lies between 17 and 272 inclusive. The destination is not relative to the lookup address.

F-opaque065 receives the final high half three times, followed by the final low half shifted left by two. F-opaque066’s result is then forwarded to F-opaque067.

### Second lookup-directed byte update

F-opaque068, F-opaque069, and F-opaque070 precede F-opaque071. F-opaque071’s result is retained for the next dependency call.

For the second lookup, the highest two bits of the final low half are cleared. That value is logically shifted right by 21, and one is added. The function reads an unsigned byte at the resulting address.

At the absolute address equal to this fetched byte plus 17, the function writes the low eight bits of the selected word logically shifted right by 22. The stored byte therefore contains original bits 22–29 of the selected word; original bits 30–31 are discarded.

This lookup address also lies between 1 and 512 inclusive, and its destination lies between 17 and 272 inclusive.

F-opaque072 then receives F-opaque071’s result shifted left by two.

The second lookup is a fresh read after the first lookup-directed write and the intervening dependencies. Equal lookup addresses do not permit caching: the first write or a dependency may have changed the entry.

### Final calls, assignment, clears, and terminal transfer

F-opaque073, F-opaque074, and F-opaque075 precede F-opaque076. F-opaque077 receives F-opaque076’s result shifted left by two, supplied twice.

F-opaque078 runs next. F-opaque079’s result is forwarded to F-opaque080. F-opaque081 then supplies the value assigned to Global Assignment.

After that assignment, one zero byte is written through the incompletely identified output-register-derived pointer.

Next, the function clears a halfword at the address obtained by clearing the highest two bits of the retained final low half and logically shifting right by seven. This address lies between 0 and 8,388,607 inclusive and need not be halfword-aligned. No alignment adjustment is represented.

Finally, F-opaque082 is invoked through address zero with three zero arguments. Only a normal return from that call permits the represented function return.

## Side effects

Excluding dependency-internal behavior, the represented effects include:

- Three word-read/low-byte-write pairs.
- Two word stores of the captured word.
- One word store of the stored link.
- One zero-word store.
- Two word stores of the propagated value.
- One read of Global Capture and one later word read through its saved address.
- Two byte lookups, each followed by an absolute-address byte store.
- One assignment to Global Assignment.
- One byte clear through an unresolved pointer.
- One halfword clear.
- Eighty-two dependency invocations if execution reaches the terminal call.

These accesses have no established atomicity, synchronization, volatile-device semantics, or memory-ordering guarantees beyond their represented sequence.

No allocation, release, reference-count update, or ownership transfer is directly established. Passing an address-like argument does not establish that a dependency acquires ownership. Conversely, nothing supplied establishes that dependencies preserve allocations or refrain from releasing memory.

Aliasing is possible wherever address calculations permit it. No general disjointness guarantee applies to globals, lookup entries, destinations, or dependency-accessible memory. Completed effects are not rolled back if a later operation fails.

## Dependencies

F-opaque001 through F-opaque082 are document-local opaque function IDs. Their explicit arguments, consumed results, and invocation order are specified above.

The first target is derived from incoming link-register state plus 3,976. Intermediate targets use embedded code-location values and fixed displacements; those calculations do not establish semantic identities for the callees. The terminal target is explicitly zero.

Different IDs can resolve to the same implementation. Ignored results do not imply absent side effects.

For every dependency, the following remain **documented_only**:

- Complete calling convention and implicit inputs.
- Meaning of results, including whether paired results are genuine 64-bit returns or combined register outputs.
- Global and pointed-memory mutations.
- Allocation, release, and ownership behavior.
- Failure, exception, and termination behavior.
- Validity and executability of the computed target.

The retained values described in this contract follow the supplied representation. They do not constitute a separate claim that arbitrary callees preserve particular machine registers.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each assumes execution reaches the described operation.

1. **Fourth-input extraction.** For a fourth input of 4,294,967,295, F-opaque003’s third explicit argument is 511. For an input below 8,388,608, it is zero.

2. **Independent repeated reads.** If the first word read at first base plus 116 has low byte 18, and intervening effects change that word to one with low byte 52, the two writes at first base plus 1 contain 18 and 52 respectively.

3. **Captured value remains the propagation source.** If the captured word is 305,419,896, subsequent changes to its original memory location do not change the two later full-word stores that use the retained captured value.

4. **Equivalent link arithmetic despite discarded high bits.** If the original link value is 0xC0001234, the shifted link value is 0x000048D0. F-opaque027 receives 0x12340000. Directly shifting the original link value left by sixteen, with 32-bit truncation, produces the same result. This equivalence holds for every 32-bit original link value.

5. **Absolute lookup destination.** If the first lookup returns byte value 200, the following byte store targets address 217, regardless of the lookup address.

6. **Captured address, later contents.** If Global Capture contains address 4,096 when captured, a later change to that global does not redirect the selected-word read: it still targets address 4,116. Its observed contents remain sensitive to intervening writes at that address.

7. **Lookup aliasing.** If the first lookup-directed store targets the second lookup’s entry, the second read observes that store’s byte unless an intervening effect changes the entry again.

8. **Second stored-byte extraction.** If the selected word is 0xC0000000, the second lookup-directed store writes zero: the original top two bits do not survive the final byte truncation.

9. **Unaligned final halfword address.** If the final low half is 128, the final halfword clear targets address 1. Acceptance does not require an alignment correction; whether that access succeeds depends on the environment.

10. **Nonreturning terminal call.** If address zero is not callable, normal return is not required by the represented behavior. Earlier completed effects remain observable; no recovery or rollback is represented.

## Uncertainties

The following remain **documented_only**:

- Application purpose and data-structure meanings.
- Completeness and accuracy of represented signatures.
- Exact provenance and validity of the final byte-clear pointer, including which dependency supplies its relevant register state.
- Storage overlap involving either global.
- Endianness, valid mappings, alignment rules, and access-fault behavior.
- Executability of computed targets and low-address operations.
- Original-language interpretation of signed overflow and signed shifts.
- Dependency contracts and normal completion.

Avoiding invalid accesses cannot be specified solely as a restriction on the four explicit inputs. Dependency results and mutable external state determine many addresses. A sufficient execution environment would also need valid targets and suitably accessible memory for every reached operation; the supplied material does not establish those conditions.

The apparent use of scaled values as addresses and repeated offset-20 stores may suggest structured state updates, but that is inferred intent, not an acceptance requirement.

## Evidence

Established static evidence consists of the supplied representation’s explicit input uses, indirect call order, result forwarding, retained values, access widths, arithmetic, global access order, and terminal zero-address call.

The arithmetic equivalence for F-opaque027 follows from the stated 32-bit model: both formulations retain exactly the original link value’s low sixteen bits in positions 16–31. Lookup ranges and byte-extraction claims likewise follow from the represented masks, shifts, additions, and truncations.

Opaque dependency IDs and descriptive retained-value labels are explanatory identifiers, not recovered program symbols.

No runtime tests have been performed. Valid memory mappings, dependency behavior, normal completion, and application intent have not been observed.
