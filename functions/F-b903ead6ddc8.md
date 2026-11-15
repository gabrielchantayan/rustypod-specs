<!-- Trusted provenance: F-b903ead6ddc8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b903ead6ddc8

## Purpose

F-b903ead6ddc8 executes an ordered sequence of indirect calls and direct memory accesses. It forwards its explicit argument to the first call, then uses ordinary call results, secondary register-carried results, incoming register state, and shared memory to determine later arguments, addresses, and stored values.

The established behavior includes byte, halfword, and word stores; unsigned bit-field extraction; several distinct address-scaling rules; two reads of one shared word; and two writes to another shared word. The final explicit control transfer calls address zero with three zero arguments.

No application-level purpose is established. Object initialization, collection management, supervisor-service behavior, and ownership transfer are not supported conclusions.

Every later operation described here is conditional on all earlier calls returning and earlier accesses completing.

## Interface

**Explicit input:** one 32-bit value. Its only direct use is as the first indirect call’s argument. No pointer interpretation, range restriction, or ownership implication is established.

**Return:** no explicit result value. A return follows the address-zero call, but is reached only if that call returns.

**Implicit inputs:**

- Incoming link/control state used to calculate the first call target.
- An incoming address-like value and an incoming byte used for the first direct update.
- Secondary register-carried results following particular calls.
- Two shared 32-bit locations, designated **G1** and **G2**.
- Bytes at absolute addresses, including address 1 and the three-byte region at addresses 9–11.
- Memory at the other calculated addresses.

**Observable outputs:** direct stores, dependency effects, and changes to G2. No separate success or failure status is shown.

**Widths and arithmetic:**

- Byte, halfword, and word mean 8, 16, and 32 bits.
- A two-word result is decomposed into its low and high 32-bit portions. This does not establish a memory layout or complete calling convention.
- Byte loads used as address components are unsigned, with values 0–255.
- Byte stores retain only bits 7–0 of their supplied value.
- Right shifts of the unsigned word values described below are logical.
- Scaling and left shifts in the examples use 32-bit bit-vector arithmetic, discarding bits above bit 31. This is not a claim that all signed operations in the supplied representation have portable overflow semantics.
- Address addition, alignment, access validity, and the execution environment’s arithmetic rules remain prerequisites where not otherwise established.

Descriptive labels below identify captured values. They do not imply object fields, allocation boundaries, or source-level interfaces.

## Behavior

### Ordering and call boundaries

The supplied representation contains no conditional branches or loops. Its visible path is straight-line, subject to faults, nonreturning calls, and dependency-controlled behavior.

The stages below preserve the order of direct memory accesses and the specified argument relationships. Intervening calls remain significant: an access must not be moved across a call merely because the call has no explicit arguments or its result is unused.

“No explicit arguments” does not mean a dependency has no register, stack, or shared-state inputs. Captured values and fresh memory reads are distinguished throughout.

### 1. Initial byte update and scaled-address clear

The first indirect call receives the explicit input. Its target depends on incoming link/control state. Further calls produce an ordinary word result.

The first direct memory update reads a byte from the incoming implicit address plus 1, then stores the incoming implicit byte at the loaded unsigned byte plus 17.

Consequently, this destination is always in the inclusive address range **17–272**. The incoming address is the lookup base, not the store base.

A dependency receives the preceding ordinary result shifted left by two. A later result is also shifted left by two and retained as the **first scaled value**.

Following further calls:

- An **initial packed word** is captured and forwarded.
- An **initial base** is captured from a later result.
- A zero byte is stored at \(8192 \times \text{first scaled value} + 9\).
- F-000000000002 receives initial base, initial packed word shifted right by 22, and initial packed word.

The zero-byte destination is not initial base plus 9.

### 2. Initial-base accesses and differently scaled addresses

After additional calls, a two-word result is obtained. Before it is passed onward, the low byte of initial base is stored at initial base plus 9.

F-000000000003 then receives:

- The result’s low word shifted left by two.
- Its high word.
- A word freshly loaded from initial base plus 52.

Following further calls, a secondary register-carried result provides a **first secondary value**. The routine stores initial base as a word at four times this value plus 20, then reads a word from the unscaled value plus 20. The loaded word becomes **first link**.

A dependency receives first secondary value.

The store and read addresses differ. They must not be collapsed into a single field access.

### 3. Word stores and link-derived reads

A later result provides a **second index**. Initial base is stored at four times second index plus 20.

Next, a word is read from first link shifted right by 22, plus 20. This becomes **second link**. A dependency receives second index multiplied by four.

Another returned word becomes **payload source**; its left shift by two is retained as **payload**.

The subsequent accesses and call are ordered as follows:

- Store payload at four times second link shifted right by one, plus 20.
- Read a word from second link plus 20, retaining **third link**.
- Call a dependency with second link shifted right by one.

After more calls, payload is stored at four times another returned index plus 20. G1 is then read, producing **first global snapshot**, before a dependency receives that returned index multiplied by four.

### 4. High-field propagation and forced-zero stores

Two extracted values are retained:

- **Link field:** third link shifted right by 22.
- **Global field:** first global snapshot shifted right by 23.

After intervening calls, a secondary result supplies a destination index. Link field is stored at four times that index plus 20. F-000000000004 receives the index twice, followed by global field and link field.

A later returned index selects another word destination, again four times the index plus 20, receiving link field. A dependency receives that index multiplied by four.

Another secondary result supplies **masked-index source**. Zero is stored at four times masked-index source plus 20, then a dependency receives masked-index source.

After further calls:

- Zero is stored at four times another returned index plus 20.
- A word is read from
  \[
  ((\text{masked-index source}\mathbin{\&}\text{0x1fffffff})\gg19)+20.
  \]
  This becomes **extracted word**.
- A dependency receives the returned index multiplied by four.

The second zero store uses global field shifted right by another 23. Since the original snapshot is 32 bits, the successive logical shifts by 23 and 23 necessarily produce zero.

### 5. Byte-indirected updates and zero propagation

After intervening calls, **extracted field** is formed by shifting extracted word right by 22.

The routine reads the byte at address 1 and stores the low byte of extracted field at the loaded unsigned byte plus 17. F-000000000005 then receives the same secondary result in its first three explicit positions and zero in its fourth.

The address-1 lookup follows from the double extraction producing zero; it does not require G1 to contain zero.

After more calls, a dependency explicitly receives zero. A later returned word is used for a shifted argument, but before that call the routine reads a byte from four times extracted field plus 1 and stores zero at the loaded byte plus 17.

Another dependency later receives two copies of a returned word shifted left by two. A subsequent result becomes **middle packed word**, which is forwarded.

Following another call, the routine again reads address 1 and stores zero at the loaded byte plus 17. A secondary result is captured as **retained control value**, and F-000000000006 receives that value three times.

Every byte-indirected destination in this stage lies in 17–272, independently of the lookup address.

### 6. Shared-word replacement and byte mirroring

A forwarding call receives middle packed word shifted right by 23 as an additional argument.

After further calls, the byte at address 1 is freshly read and retained as **small base**. Zero is stored at small base plus 17, then a dependency receives a returned word shifted left by two.

Further calls include one receiving two identical shifted arguments. A later result becomes **replacement word**.

Replacement word is stored to G2 before being forwarded to another dependency.

Following another call, a word is read from small base plus 116, and its low byte is stored at small base plus 1. These addresses use the previously captured small base, not a new read of address 1.

A dependency receives retained control value shifted left by 11. A later two-word result is forwarded as its low word, its high word, and replacement word shifted right by 23.

### 7. Repeated mirroring and retained payload

A later result multiplied by four becomes **mirror base**. A word is read from mirror base plus 116, and its low byte is stored at mirror base plus 1. A dependency receives mirror base.

After further calls, a result becomes **later source**. Its left shift by two is retained as **later scaled value**.

The word read from mirror base plus 116 and low-byte store to mirror base plus 1 are then repeated. This is a fresh read; intervening dependency effects may change its value.

F-000000000007 receives:

- Retained control value shifted left by 12.
- Later scaled value twice.
- Later source shifted left by 18.

After additional calls and forwarding, another returned index is multiplied by four. A word at that scaled index plus 116 is captured as **retained payload word**. Its low byte is stored at the scaled index plus 1, then a dependency receives the scaled index.

### 8. Payload replication and clears

A later returned index selects a sixteen-byte-stride destination: retained payload word is stored at sixteen times the index plus 20.

A dependency receives later scaled value logically shifted right by two and then shifted left by 16. This operates on the already-truncated scaled value; it does not generally recover the original later source.

Following additional calls, retained payload word is stored at four times another returned index plus 20. A dependency receives that index multiplied by four.

A subsequent returned word shifted left by two becomes **new scaled payload**. It is stored at

\[
(\text{later scaled value}\mathbin{\&}\text{0xfffffffc})+20.
\]

A dependency receives new scaled payload twice.

After more calls, zero is stored at four times another returned index plus 20, and a dependency receives that index multiplied by four.

### 9. Final linking and byte-indirected propagation

A later two-word result supplies:

- **Final scaled payload:** its low word shifted left by two.
- **Final destination index:** its high word.

Final scaled payload is stored at four times final destination index plus 20. G1 is then read again, producing **second global snapshot**. This is a fresh shared-memory read, not reuse of first global snapshot.

A dependency receives final scaled payload, final destination index, and two further copies of final scaled payload.

Following more calls, final scaled payload is stored at four times another returned index plus 20. A word is then read from second global snapshot plus 20, becoming **final read word**. A dependency receives the scaled index.

Another two-word result supplies **final low word** and **final high word**.

Before F-000000000008:

- A byte is read from retained payload word shifted right by 23, plus 1.
- The low byte of final read word is stored at the loaded unsigned byte plus 17.

F-000000000008 receives final high word three times and final low word shifted left by two.

After further calls, a byte is read from

\[
((\text{final low word}\mathbin{\&}\text{0x3fffffff})\gg21)+1.
\]

The low byte of final read word shifted right by 22 is stored at the loaded byte plus 17. A dependency receives another returned word shifted left by two.

Both store destinations remain restricted to 17–272.

### 10. Terminal updates and control transfer

Further calls include repeated shifted arguments, a no-explicit-argument call, and result forwarding. F-000000000009 subsequently returns a byte and supplies a secondary pointer.

The terminal direct sequence is:

1. Store a zero byte through that pointer.
2. Read the three-byte quantity at addresses 9–11 and replace G2 with that quantity in bits 31–8 and the returned byte in bits 7–0.
3. Store a zero halfword at
   \[
   (\text{final low word}\mathbin{\&}\text{0x3fffffff})\gg7.
   \]
4. Invoke F-00000000000a, the address-zero call, with three explicit zero arguments.
5. Return only if that call returns.

The pointer-byte store precedes the three-byte read and may affect it if the pointer addresses that region. The representation does not establish the byte order or internal access order of the three-byte load.

The halfword destination is not multiplied by two and can be odd.

## Side effects

Direct effects are the accesses above, including absolute low-address accesses. They are not confined to an established caller-owned object.

**Byte-indirected destination bounds are established:** every destination formed from an unsigned byte plus 17 lies in **17–272 inclusive**. Such a store cannot directly overlap address 1 or any byte in addresses 9–11. Its lookup address does not expand its destination range.

Overlap with G1 or G2 is possible only if their unresolved locations intersect the relevant destination. No such overlap is established.

Broader aliasing remains unresolved for the other calculated stores, the terminal pointer store, the terminal halfword store, and dependency effects. These may affect later lookup bytes, shared words, or the address-9 region where their addresses overlap. Byte-indirected stores can also affect later reads within their own destination range.

No direct allocation, release, reference-count operation, lock, bounds check, rollback, or error cleanup is shown. Dependencies may perform such actions; their behavior is unknown.

No ownership transfer or lifetime extension is established. Execution requires valid reached control targets and appropriately accessible memory. No explicit recovery path or error result is visible.

## Dependencies

The following opaque IDs identify selected **call sites**, not verified distinct functions:

| ID | Established role |
|---|---|
| F-000000000001 | First indirect call; receives the explicit input. |
| F-000000000002 | Receives initial base and two forms of initial packed word. |
| F-000000000003 | Receives shifted low word, high word, and a fresh word load. |
| F-000000000004 | Receives a repeated destination index and two extracted fields. |
| F-000000000005 | Receives three copies of a secondary result and zero. |
| F-000000000006 | Receives retained control value three times. |
| F-000000000007 | Receives two control/source-derived shifts and repeated later scaled value. |
| F-000000000008 | Receives repeated final high word and shifted final low word. |
| F-000000000009 | Supplies the terminal byte and secondary pointer. |
| F-00000000000a | Calls address zero with three zero arguments. |

Other intervening calls remain unresolved dependencies. Their ordering is significant; their complete interfaces and effects are **documented_only**.

Most targets are calculated using embedded control/address values and offsets. The first additionally depends on incoming control state. No dependency is assumed to preserve globals, preserve pointed-to memory, allocate or release storage, or return normally.

## Acceptance examples

These are conditional arithmetic and sequencing examples, not runtime observations.

- **Destination bounds:** A lookup byte of 0 selects address 17; a lookup byte of 255 selects address 272. No lookup byte selects address 1 or addresses 9–11 through the byte-plus-17 rule.

- **Initial update:** If the incoming lookup address plus 1 contains 200, the first store targets address 217. An incoming byte of 171 is stored there.

- **Distinct scaling:** A first secondary value of 3 selects address 32 for the initial-base word store and address 23 for the following word read.

- **Double extraction:** Every 32-bit first global snapshot produces zero after successive logical right shifts by 23 and 23. The associated later lookup therefore reads address 1.

- **Fresh mirror read:** If the first mirror read has low byte 18 and an intervening dependency changes it to 52, the second mirror write stores 52.

- **Captured base:** Changing address 1 after small base is captured does not change the later small-base-plus-116 address. It can change a separate subsequent read of address 1.

- **Byte truncation:** For final read word 0xFFC000AB, the first final propagation stores 0xAB; the later propagation stores 0xFF.

- **Stride distinction:** Index 3 selects address 68 for the sixteen-byte-stride store, not address 32.

- **Terminal composition:** If the terminal byte is 0x5A and the three-byte quantity read is 0x123456, the terminal G2 write supplies 0x1234565A. A subsequent overlapping halfword store could alter that storage.

- **Terminal alias ordering:** If the terminal pointer addresses byte 9, its zero-byte store occurs before the three-byte quantity is read. This possibility does not arise from the earlier byte-plus-17 destinations.

- **Unaligned halfword:** Final low word 0x80 selects halfword destination 1. No alignment correction is performed; success depends on the execution environment.

## Uncertainties

**Established static evidence:** access widths, constants, masks, unsigned extractions, argument relationships, direct access ordering, bounded byte-indirected destinations, two G1 reads, two G2 writes, and the final address-zero call.

**Inferred intent:** repeated offsets and scaling may represent fields or encoded references. Neither interpretation is required by the evidence.

**Documented-only unknowns:**

- Complete calling convention and meaning of secondary register-carried results.
- Valid address space, endianness, alignment behavior, and memory-mapped effects.
- Exact overflow and address-wrapping behavior.
- G1 and G2 locations and overlap relationships.
- Dependency identities, effects, preservation rules, and termination.
- Whether address zero executes, traps, or has platform-specific meaning.
- Whether the apparent routine boundary describes a complete callable unit.

These unknowns prevent an unconditional successful-execution or application-purpose claim.

## Evidence

The supplied static representation is the sole behavioral evidence. Names, comments, prior prose, and reviewer statements are not independent authority.

Unsigned byte loads followed by addition of 17 establish the 17–272 destination bound. That bound excludes direct overlap with address 1 and addresses 9–11; it does not constrain other calculated stores or unknown dependency effects.

No external implementation was accessed, no dependency was resolved, and no runtime tests were performed. Acceptance examples are conditional consequences of the supplied representation, not execution results.
