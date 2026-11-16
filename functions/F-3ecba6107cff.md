<!-- Trusted provenance: F-3ecba6107cff; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3ecba6107cff

## Purpose

F-3ecba6107cff is an effectful routine that interleaves indirect calls with byte, halfword, and word memory operations. Its visible actions include scaled-index word stores, byte-selected byte stores, copying the low byte of a word into another field, and updating a shared word.

No application-level purpose is established. The repeated access patterns could represent manipulation of encoded references or records, but the supplied material does not establish an object model, allocator, device protocol, or valid address space.

This contract describes the supplied static representation. Execution, successful completion, and the accuracy of recovered calling conventions have not been verified.

## Interface

### Explicit inputs

The displayed interface has four positional inputs and no explicit return value:

| Input | Displayed width | Visible use |
|---|---:|---|
| First | Unsigned 32 bits | Passed to the first indirect call; later shifted right by one to form a byte-read address. |
| Second | 32-bit uninterpreted value | No explicit use shown. |
| Third | 32-bit uninterpreted value | No explicit use shown. |
| Fourth | Unsigned 32 bits | Passed to indirect calls directly and after logical right shift by 22. |

The apparent non-use of the second and third inputs does not establish that machine-level callees cannot observe their incoming register contents.

### Additional entry state

The routine also depends on state not represented by its four explicit inputs:

- An incoming link-like register value used to calculate the first call target.
- An incoming 32-bit register value written by the first direct word store.
- Secondary register outputs attributed to preceding calls.
- Two shared word locations, distinguished here as **shared source word** and **shared result word**.
- A shared byte at numeric address 1.
- A three-byte value associated with numeric address 9, used in the final shared-result update.
- Memory and executable targets reachable through calculated addresses.

These are effective prerequisites, not independently validated parameters.

### Width and notation conventions

For the formulas below:

- **W(address)** means a 32-bit memory read or write.
- **B(address)** means an unsigned 8-bit memory read or an 8-bit write.
- **H(address)** means a 16-bit write.
- “Low byte” means the least significant eight bits of the value, not necessarily the byte at the lowest address.
- Right shifts of explicitly unsigned values are logical.
- Scaled addresses are described using 32-bit machine-word arithmetic. Overflow behavior in the recovered signed expressions is not independently established.

The supplied representation uses signed-looking intermediate types for some multiplication and left shifts. Those expressions must not be treated as portable signed-language arithmetic guarantees. Where an address formula is used below, it describes the displayed machine-oriented calculation; validity still depends on address arithmetic and the target memory model.

## Behavior

### Execution structure

The displayed body has no explicit branch, loop, input validation, or error return. It follows one sequence of calls and memory operations.

Numerous indirect calls separate the operations described below. Those calls are observable boundaries: they may alter memory, globals, register outputs, or control flow. The phase descriptions preserve the ordering of direct accesses but do not imply that successive accesses are adjacent instructions.

Most call targets are calculated from embedded code-location values plus offsets. The first depends on incoming register state; the last has numeric target zero. Target identity and callee semantics are unresolved.

### 1. Entry calls and initial byte-selected update

1. The first indirect call receives the first input and the fourth input shifted right by 22.
2. After four further indirect calls, the final call’s primary result supplies an index. A word store writes the incoming auxiliary 32-bit register value at four times that index plus 20.
3. A following call receives four times the index.
4. After further calls, an 8-bit call result is obtained. The routine reads a byte at the first input shifted right by one, plus 1. That byte selects an address by adding 17. The routine writes the low eight bits of the call result shifted left by two at the selected address.

The selector is an unsigned byte. Consequently, a byte-selected destination of this form is numerically between 17 and 272 inclusive, before considering address-space interpretation.

### 2. Fourth-input-derived selector and scaled zero store

The fourth input shifted right by 22 is a ten-bit quantity. It is passed in both the first and third argument positions of an indirect call, with the unshifted fourth input between them.

A subsequent call result is forwarded with two copies of that ten-bit quantity.

After another group of calls:

1. The routine reads a selector byte at the ten-bit quantity plus 1.
2. It writes zero at that selector plus 17.
3. It passes a newly obtained primary result shifted left by two to another call.

A later primary result is shifted left by two and retained as a scaled value. That value is passed to a call. A subsequent primary word result is forwarded through another call, and a later result is retained as an address-like value.

The routine then writes zero to the byte at the retained scaled value multiplied by 8192, plus 9. At the bit-pattern level, the combined scaling corresponds to shifting the original result left by 15, subject to 32-bit truncation.

### 3. Retained address and word-chain construction

A call receives:

- The retained address-like value.
- The recently obtained word shifted right by 22.
- The unshifted word.

After more calls, a displayed 64-bit result is split into low and high 32-bit portions.

The next direct operations are:

1. Write the low byte of the retained address-like value at that value plus 9.
2. Read a word at that same value plus 52.
3. Invoke a call with the low result word shifted left by two, the high result word, and the word just read.

After another call group, a secondary-register result supplies a new value. The routine writes the retained address-like value at four times this secondary result plus 20, then reads a word at the secondary result plus 20. These are different address formulas.

The secondary result is subsequently passed to a call. After further calls:

1. A new primary result selects another destination at four times the result plus 20; the retained address-like value is written there.
2. The previously loaded word, shifted right by 22, supplies an address base; a word is read at that base plus 20.
3. A call receives four times the new primary result.

A later primary result is shifted left by two. The newly loaded word is shifted right by one, and the shifted primary result is stored at four times that shifted word plus 20. A second word is then read at the unshifted loaded word plus 20.

After another call group, the same shifted primary result is written at four times another primary result plus 20.

### 4. Shared-source extraction and established zero calculations

The routine reads the shared source word and retains that snapshot across subsequent calls.

It derives:

- A ten-bit quantity by shifting the most recently loaded chain word right by 22.
- A nine-bit quantity by shifting the shared-source snapshot right by 23.

The ten-bit quantity is stored at four times a secondary-register result plus 20. A call receives that secondary result twice, followed by the nine-bit and ten-bit quantities.

After a subsequent call, the nine-bit quantity is shifted right by 23 again. Because its value is at most 511, this second shift produces zero.

The following direct accesses occur in order, with calls between them:

1. Store the ten-bit quantity at four times a new primary result plus 20.
2. Store a zero word at four times a new secondary-register result plus 20.
3. Store the nine-bit quantity shifted right by 23 at four times another primary result plus 20. This stored value is zero.
4. Read a word at 20 plus the earlier secondary result masked to 29 bits and shifted right by 19.

The word from the final read is shifted right by 22. Its low byte is written to a byte-selected destination whose selector-read address is the twice-shifted shared-source value plus 1. Under the displayed 32-bit arithmetic, that selector read is therefore at address 1.

A separate working value is explicitly set to zero. A later call receives three copies of a secondary-register result and a fourth argument of zero; another call subsequently receives zero shifted right by 23, also zero.

These zero conclusions follow from the explicit widths and assignments, not from assumptions about a callee’s return value.

### 5. Further selector clears and shared-result update

The ten-bit value derived from the preceding word read is multiplied by four. The routine reads a selector byte at that value plus 1, then clears the byte at the selector plus 17.

Following further call groups:

1. A primary result shifted left by two is passed twice to a call.
2. A later word result is retained and forwarded.
3. The shared byte at address 1 is read; the byte at its value plus 17 is cleared.
4. A secondary-register result is retained and passed three times to a call.
5. Another result is forwarded with the retained word shifted right by 23.
6. The shared byte at address 1 is read again and captured separately. The byte at this new snapshot plus 17 is cleared.

The two shared-byte reads are not interchangeable. Intervening calls may change address 1.

After more calls, a new word result is obtained. It is written to the shared result word before being passed to another call.

Using the most recent captured shared-byte value, the routine then reads a word at that value plus 116 and writes its low byte at that value plus 1.

### 6. Repeated word-to-byte copies

The retained secondary-register value from the previous phase is shifted left by 11 and passed to a call. A later displayed 64-bit result supplies two word arguments; a third argument is the recent shared-result value shifted right by 23.

A subsequent primary result is multiplied by four to obtain a base. The routine:

1. Reads a word at that base plus 116.
2. Writes its low byte at that base plus 1.
3. Calls using that base.
4. Executes further calls and obtains another primary result.
5. Reads the word at the original base plus 116 again.
6. Writes its low byte at the original base plus 1 again.

The repeated read is significant. The second copy need not reproduce the first value.

A later call receives four arguments: the retained secondary value shifted left by 12; two copies of the newly obtained primary result shifted left by two; and that primary result shifted left by 18.

After more calls, another primary result multiplied by four establishes another base. A word at that base plus 116 is loaded and retained, and its low byte is stored at the base plus 1.

### 7. Replication, link writes, and final shared-source read

The retained full word from the last word-to-byte copy is subsequently written:

1. At sixteen times a new primary result plus 20.
2. After intervening calls, at four times another primary result plus 20.

Between these stores, a call argument is formed by logically shifting the earlier retained two-bit-scaled result right by two and then left by 16. This does not generally recover the original result: the initial left shift may already have discarded its top two bits.

A later primary result is shifted left by two and stored at the earlier two-bit-scaled value masked with `0xfffffffc`, plus 20. Another call receives this new shifted result twice.

Following further calls:

1. A zero word is stored at four times a new primary result plus 20.
2. A displayed 64-bit result is split. Its low word shifted left by two is stored at four times its high word plus 20.
3. The shared source word is read again, producing a new snapshot.
4. A call receives the shifted low word, the high word, and two further copies of the shifted low word.
5. After more calls, the same shifted low word is stored at four times another primary result plus 20.
6. A word is read at the new shared-source snapshot plus 20.

The second shared-source snapshot is independent of the earlier snapshot.

### 8. Final byte-selected writes and termination attempt

A subsequent displayed 64-bit result supplies another low/high pair.

The routine reads a selector byte at the retained word from phase 6 shifted right by 23, plus 1. It writes the low byte of the latest chain word at the selector plus 17.

A call receives three copies of the latest high result word and the low result word shifted left by two.

After more calls, the routine:

1. Masks that low result word with `0x3fffffff` and shifts it right by 21.
2. Reads a selector byte at that value plus 1.
3. Writes the low byte of the chain word shifted right by 22 at the selector plus 17.

Further calls receive shifted primary results, including a call with two identical arguments. A later call result is forwarded, and a final 8-bit result is obtained.

The final explicit effects are ordered as follows:

1. Write zero through a pointer supplied as a secondary-register result.
2. Replace the shared result word with a composition whose low eight bits are the final 8-bit result and whose upper 24 bits come from the three-byte value associated with address 9.
3. Write a zero halfword at the address formed from the retained low result word masked with `0x3fffffff`, then shifted right by 7.
4. Invoke numeric target zero with three explicit zero arguments.
5. Reach the displayed void return only if that invocation returns.

The first zero-byte write may alias the shared result, the three-byte source, or the later halfword destination. No disjointness is established.

## Side effects

All explicit outputs are side effects. No result value or status code is provided.

Direct effects include:

- Byte, halfword, and word memory writes.
- Word and byte reads at calculated addresses, including low numeric addresses.
- Two reads of the shared source word at different execution points.
- Multiple reads of the shared byte at address 1.
- Two writes to the shared result word.
- A final indirect invocation of numeric target zero.

There is no visible allocation, release, ownership transfer, rollback, synchronization, or explicit error handling. This does not establish that callees avoid those actions.

No validation precedes the displayed accesses. Successful execution requires readable, writable, and executable locations appropriate to every reached operation, plus supported alignment and calling conventions. The final halfword address is not visibly aligned.

If a call fails to return or an access faults, later effects are not established. Earlier completed writes are not visibly undone. The representation supplies no portable exception or fault contract.

## Dependencies

The only supplied resolved opaque function identity is F-3ecba6107cff itself. No independent callee identity is established.

Accordingly, this contract does not invent opaque function IDs for unresolved targets or equate targets merely because their calculations look similar.

Dependency requirements are documented_only:

- Indirect-call destinations must be executable for execution to proceed.
- Primary, paired-word, byte-sized, and secondary-register outputs must have the meanings assumed by the displayed representation.
- Callees may change globals and reachable memory.
- Callees’ ownership, allocation, release, synchronization, and failure behavior are unknown.
- Numeric target zero must resolve to executable behavior that returns if normal completion is expected.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

1. **Fourth-input extraction.** If the fourth input is `0xffffffff`, its logical right shift by 22 is 1023. Calls using this extraction receive 1023, and the corresponding selector-byte read uses address 1024.

2. **Selector width.** If a selector read returns 255, the selected byte-write address is 272. It is not scaled by a word size.

3. **Initial byte truncation.** If the initial byte-sized call result has bit pattern `0x7f`, shifting it left by two and storing one byte writes `0xfc`.

4. **Double-shift zero.** For every 32-bit shared-source snapshot, shifting right by 23 twice produces zero. The dependent selector read therefore uses address 1, and the dependent word store writes zero.

5. **Snapshot independence.** If an intervening callee changes the shared byte at address 1, later selector clears use the newly read value. They must not reuse an earlier snapshot.

6. **Repeated copy observability.** If an intervening call changes a base-plus-116 word from `0x11223344` to `0xaabbccdd`, the two corresponding base-plus-1 writes are `0x44` and `0xdd`.

7. **Shift truncation.** If a 32-bit primary result is `0xc0000001`, shifting left by two yields bit pattern 4 under modular machine arithmetic. Shifting that result right by two yields 1, not the original word.

8. **Conditional completion.** If the invocation of target zero traps or never returns, no normal return is established. If it returns, the displayed body has no further explicit memory operation before returning.

## Uncertainties

- The recovered function boundary, register interface, and indirect-call signatures are not independently verified.
- Secondary-register outputs may reflect incomplete call modeling.
- Displayed 64-bit results may represent paired registers rather than a conventional scalar return.
- Endianness, alignment rules, memory mapping, arithmetic overflow semantics, and fault behavior are not established.
- Shared-symbol overlap means differently described storage may alias.
- The three-byte source’s storage interpretation is less certain than the explicit final 24-bit/8-bit composition.
- No evidence establishes that address zero is invalid or valid as a call target.
- No application-level interpretation of offsets 1, 9, 17, 20, 52, and 116 is established.
- Signed-byte promotion details are uncertain; the described byte stores concern their final low-eight-bit patterns.
- Inferred intent is limited to repeated scaled-reference and field-copy patterns. It is not an established ownership or data-structure contract.

All unresolved points are documented_only.

## Evidence

The supplied static representation establishes the displayed input widths, call order, explicit access widths, address calculations, shifts, masks, shared-state accesses, and absence of visible conditional control flow.

The zero results from repeated unsigned right shifts are arithmetic deductions from that representation.

No dependency bodies, target ABI specification, address map, authoritative record layouts, or runtime observations were supplied. No runtime tests have been performed.
