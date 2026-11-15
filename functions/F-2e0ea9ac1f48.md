<!-- Trusted provenance: F-2e0ea9ac1f48; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2e0ea9ac1f48

## Purpose

The supplied static representation shows a stateful routine that interleaves indirect calls with byte and 32-bit memory accesses. Values obtained from entry register state, memory, and call results are repeatedly scaled, masked, and used as addresses or stored data.

No application-level purpose is established. In particular, the representation does not establish that this routine constructs objects, processes a buffer, manages a collection, or implements a recognizable protocol. Such interpretations would require additional evidence.

This contract describes the visible execution prefix, conditional on the supplied representation accurately reflecting executable behavior. That qualification is material: the representation reports invalid instruction data and terminates with truncated control flow. It does not establish a normal return or a complete function boundary.

## Interface

**Declared interface.** The routine has two nominal 32-bit inputs and no declared return value.

- The first nominal input has no visible use.
- The second nominal input is forwarded as the second argument of the first indirect call.
- No meaningful return value is established.

**Additional entry state.** The visible behavior also depends on three incoming machine-state values outside the nominal parameter list:

- An address-like 32-bit value, called the **entry base** here.
- An unsigned 32-bit value, called the **entry quantity**.
- A link-state value used to derive the first indirect call target.

These are descriptive names for contract inputs, not established parameter names or a recovered calling convention.

Several later operations consume additional call-associated register outputs. Their producing calls and ABI meaning are not reliably established. They must not be treated as initialized explicit parameters or as ordinary documented return values.

**Memory notation and widths.**

- A byte read produces an unsigned value from 0 through 255.
- A byte store retains only the low eight bits.
- A word read or store accesses 32 bits.
- A displayed 64-bit call result is used as two 32-bit halves: low and high. Its representation does not establish whether the underlying ABI returns a scalar, a pair, or independent register values.
- Address arithmetic and scaling are described as 32-bit bit-pattern operations, with truncation to 32 bits. This is the appropriate conditional machine-level interpretation; the supplied signed expressions do not establish portable arithmetic semantics.
- Logical right shifts introduce zero bits.
- Endianness and alignment requirements are unknown.

“Byte-selected store” below means: read one byte from the stated source address, add `0x11` to its unsigned value, and store a byte at that resulting address. Its destination is therefore between `0x11` and `0x110`, not relative to the source address.

**Execution preconditions.** To execute any prefix successfully, every memory location reached by that prefix must support the indicated access, and every reached indirect target must be executable under the actual ABI. These conditions include low absolute addresses. There are no visible checks enforcing them.

## Behavior

### Ordering and call interpretation

The visible prefix is linear: no conditional branch or loop is represented. Each dependency call and direct memory operation occurs in the displayed order, provided preceding operations return or complete normally.

Calls with no displayed arguments are not proven to have no effective machine-level inputs. Likewise, unused displayed return values do not make calls removable: their memory effects and register effects are unknown.

For precision, the stages below name important values by their origin. These names do not assert object types, ownership, or business meaning. Values retained across calls are distinguished from memory that is reread after calls.

### 1. Entry processing and initial byte-selected updates

The routine first forms the **doubled entry quantity** by multiplying the entry quantity by two.

Before the first call, it reads a word at entry base plus `0x74`, then writes that word’s low byte at entry base plus one.

The first indirect call receives, in order:

1. The doubled entry quantity.
2. The second nominal input.
3. The entry quantity logically shifted right by two.
4. The entry base.

The next call supplies a displayed 64-bit result. Its low half, high half, low half again, and the entry quantity masked by `0x7fffffff` then shifted right by one are passed to the following call.

Four further calls follow. The final result is multiplied by four to form a temporary base. The routine reads a word at that base plus `0x74`, stores its low byte at that base plus one, and calls a dependency with the temporary base. Four more calls occur.

Next, a byte-selected store uses doubled entry quantity plus one as its source address and writes zero. A call receives an unresolved call-associated register value. The next call supplies a retained **record base**, which is passed to another call. Four calls follow.

The routine then rereads the byte at doubled entry quantity plus one. It uses that fresh byte to select a destination and writes the top byte of the doubled entry quantity there. This is not necessarily the destination of the earlier zero store: intervening calls may have changed the selector.

### 2. Derived records and low-address state

A call receives the preceding call result shifted left by two. Four calls follow, producing a result whose fourfold value becomes the **scaled record base**.

The routine reads a word at retained record base plus four; call this the **record link**. It performs a byte-selected zero store using record link plus one as the selector source.

A dependency receives the scaled record base three times. Four calls follow. The routine then reads a byte at record link logically shifted right by two, plus one. Call this byte the **small base**. It stores zero at small base plus `0x11`.

Another call receives the preceding call result shifted left by two. Four calls follow.

The next direct accesses are ordered:

1. Read a word at scaled record base plus four; retain it as the **packed field**.
2. Read a word at small base plus `0x74`; retain it as the **linked word**.
3. Store the linked word’s low byte at small base plus one.

A call receives the linked word, the scaled record base shifted left by 19 twice, and the most recent call result shifted left by two.

The next call supplies a word that is forwarded to another call together with linked word logically shifted right by two. Four calls follow. The final result is multiplied by four to form another temporary base. A word is read at that base plus `0x74`, and its low byte is stored at that base plus one.

That base is passed to a call, followed by four more calls. The retained word is then stored at an address equal to four times an unresolved call-associated register output, plus `0x14`.

### 3. Packed-field projection and indexed stores

A call receives the same unresolved register output. The following call supplies a word. Another call receives that word and two identical copies of this bit projection:

- Logically shift the packed field right by 22.
- Retain nine bits.
- Shift those bits left by 21.

Four calls follow. At four times the final result plus `0x14`, the routine stores the packed field shifted right by 22 and then left by 23, subject to 32-bit truncation.

That fourfold result is passed to a call. Four calls follow and produce an **index value**. The routine then:

- Stores four times the index value at sixteen times the index value plus `0x14`.
- Retains the index value masked by `0x3fffffff` as the **masked index**.
- Retains masked index logically shifted right by two as a later byte-store source.
- Calls a dependency with masked index.

Four calls follow and produce a **slot index**. The routine stores masked index at four times slot index plus `0x14`, passes four times slot index to a call, and performs four more calls. The final result becomes the **transfer value**.

### 4. Linked-byte and payload updates

Using the linked word retained earlier, the routine stores the low byte of eight times slot index at linked word plus one. It then stores transfer value shifted left by two at eight times slot index plus `0x34`.

A call receives transfer value shifted left by two. A following call supplies a word, which is forwarded to another call. Four more calls follow, yielding a displayed 64-bit result.

The routine next:

1. Masks transfer value by `0x3fffffff`, producing the **transfer base**.
2. Stores the low byte of eight times slot index at transfer base plus one.
3. Replaces the word at eight times slot index plus `0x34` with the high half of the displayed 64-bit result.
4. Calls a dependency with the low half shifted left by two.

Four calls follow. The routine stores the low byte of sixteen times slot index at the absolute address obtained by shifting transfer base right by 20 and adding `0x11`. This address lies between `0x11` and `0x410`.

It also stores the low byte of four times the earlier index value at sixteen times slot index plus `0x0d`.

A call receives the preceding call result shifted left by two, twice. The next call supplies a word. A following call receives that word and its logical right shift by three. Four calls follow.

The routine then writes the low byte of sixteen times slot index to absolute address `0x11`, and repeats the byte store at sixteen times slot index plus `0x0d`. The intervening calls prevent assuming either repeated operation is redundant.

### 5. Secondary scaled state

A call receives the preceding result shifted left by two. Four calls follow, producing a **secondary value**.

Before the next calls, the routine stores:

- Four times the earlier index value as a word at transfer base plus `0x14`.
- The low byte of transfer base shifted left by one at transfer base plus `0x0d`.

It derives a **secondary base** as secondary value multiplied by 32, and a **secondary fragment** as secondary value masked by `0x07ffffff` then shifted right by 17.

Three calls follow, the last producing a result. The routine writes secondary value shifted left by seven at secondary base plus `0x14`, then writes the low byte of that call result shifted left by two at secondary base plus `0x0d`.

A call receives that result shifted left by two. Four calls follow.

The next stores are:

1. At secondary fragment plus seven, store the low byte of masked index logically shifted right by two and then left by one.
2. Perform a byte-selected store using secondary base plus one as the selector source, writing the low byte of secondary fragment.

A dependency receives three copies of the previously retained call result shifted left by 21. The following call supplies the **byte source value**. It is passed to another call, followed by four calls.

The routine reads and retains a byte from secondary base plus one as the **saved selector**, then writes zero at saved selector plus `0x11`. A call receives the latest call result shifted left by two, followed by four more calls.

### 6. Absolute byte updates and subsequent word propagation

The routine stores the low byte of byte source value logically shifted right by three at secondary base plus `0x0d`. It then stores saved selector at absolute address `0x0d`.

A dependency receives twice saved selector and the constants `0x98` and `0x260`. These constants have no established interpretation as lengths, capacities, or bounds.

The next call has no displayed arguments. Another call receives an unresolved unsigned register output logically shifted right by three. Four calls follow. Their final result, shifted left by two, is passed to another call that supplies a result.

Before the next call, the routine:

- Stores the previously retained byte-source fragment at twice saved selector plus `0x0d`.
- Writes `0xc0` to absolute address `0x0d`.

The latest call result shifted left by two is passed onward. Four calls follow and produce another result. The routine stores that result shifted left by two at twice saved selector plus `0x24`, then reads a word at twice saved selector plus `0x74` as the **late base**.

A call receives late base, late base shifted left by one, the latest result shifted left by five, and late base masked by `0x7fffffff` then shifted right by two. Four calls follow.

The routine writes the retained result shifted left by five at late base plus `0x24`. It then reads a word at late base plus `0x74` and passes it to a call. Four calls follow, yielding a displayed 64-bit result.

### 7. Final visible propagation and termination boundary

At four times that result’s high half plus `0x14`, the routine stores byte source value logically shifted right by three as a full word. The earlier byte store of this expression retained only eight bits; this store retains all 32 bits.

A call receives three copies of the low half shifted left by two, followed by the high half logically shifted right by three. The next call yields another displayed 64-bit result. Its low half, high half, low half again, and high half logically shifted right by three are passed onward.

Four calls follow. At four times the final result plus `0x14`, the routine stores the retained low half from the preceding stage shifted left by two. That fourfold result is passed to another call. Four calls follow and produce a **final selector value**.

The routine performs a byte-selected store using late base plus one as selector source, writing the low byte of final selector value shifted left by two. Three calls follow. It rereads the selector at late base plus one and writes the low byte of final selector value masked by `0x3fffffff` then shifted right by 20.

A call receives the latest call result shifted left by two. Four calls follow.

A byte-selected zero store then uses final selector value shifted left by 12, plus one, as selector source. A call receives final selector value shifted left by five twice, final selector value shifted left by two, and the most recent call result shifted left by two. Four calls follow.

The same selector source address is read again for another byte-selected zero store. A call receives the latest result shifted left by two, followed by four calls.

The supplied representation stops at an invalid-instruction boundary. No normal return, recovery, rollback, or subsequent behavior is established.

## Side effects

The routine directly writes bytes and words through computed addresses, including absolute addresses `0x0d` and `0x11`. These are observable memory effects, not established object fields or harmless scratch state.

Every indirect call may independently read or modify memory, change global state, fail, or fail to return. No preservation guarantees are supplied. Consequently:

- Repeated reads separated by calls must remain distinct.
- Earlier writes are not guaranteed to survive intervening calls.
- Aliasing can make later operations overwrite earlier destinations.
- No atomicity or transactional behavior is established.
- Failure may leave a partially modified state.

There is no visible direct allocation, release, reference-count operation, or ownership transfer. This does not establish that dependencies perform none. The routine cannot be specified as preserving ownership or keeping referenced storage alive without additional dependency contracts.

## Dependencies

No external dependency identity is resolved by the supplied map. For this contract, **F-unknown-001**, **F-unknown-002**, and subsequent ordinal opaque IDs denote successive indirect call occurrences, in execution order. They are occurrence identifiers only; different occurrences may resolve to the same target.

The first target depends on incoming link state. Subsequent targets are computed from embedded address-like values. The representation establishes call ordering and displayed arguments, but not trustworthy semantic interfaces for the targets.

All dependency semantics remain `documented_only`, including:

- Effective ABI, additional register inputs, and clobbers.
- Memory and global-state effects.
- Allocation, retention, and release behavior.
- Meaning and validity of returned address-like values.
- Exceptions, traps, nonreturning behavior, and synchronization.

The terminal invalid-instruction marker is an analysis boundary, not an established callable service.

## Acceptance examples

These examples are conditional static consequences, not executed tests.

1. **Entry-byte truncation.** If the word at entry base plus `0x74` is `0x123456ab`, the first direct store writes `0xab` at entry base plus one. It does not copy four bytes.

2. **Unsigned selector addressing.** If a byte-selected store reads `0xfe`, its destination is absolute address `0x10f`. It is not source address plus `0x10f`.

3. **Fresh selectors after calls.** If intervening dependencies change a selector byte from `0x20` to `0x30`, a later reread selects `0x41`, rather than the earlier destination `0x31`.

4. **Width-sensitive entry arithmetic.** Under the conditional 32-bit interpretation, entry quantity `0x80000001` doubles to `0x00000002`. Its logical right shift by two is `0x20000000`; masking by `0x7fffffff` and shifting right by one yields zero.

5. **Byte versus word storage.** If byte source value is `0x12345678`, shifting right by three yields `0x02468acf`. The related byte stores write `0xcf`; the later word store writes `0x02468acf`.

6. **Absolute-location ordering.** On an execution reaching the relevant stores, saved selector is written to absolute address `0x0d` before `0xc0` is later written there. Dependencies between those stores may observe or alter that location.

7. **Failure prefix.** If the initial word read faults, the initial byte write and first call are not reached. If a later dependency does not return, no subsequent direct operation is established.

8. **Termination limitation.** Reaching the end of the represented prefix does not satisfy a “returns successfully” criterion. Such a criterion requires evidence beyond the supplied material.

## Uncertainties

**Established static evidence:** displayed widths, literal masks and shifts, direct access ordering, indirect-call ordering, argument relationships, and the truncated termination boundary.

**Inferred interpretation:** 32-bit wrapping address arithmetic and the division of displayed 64-bit results into machine-level output halves. These interpretations support a conditional contract, not a verified ABI.

**Documented only:** actual executable reachability, function boundaries, correctness of decoded operations, platform memory mapping, alignment, endianness, dependency identities, register preservation, ownership, concurrency, application intent, and eventual termination.

The input contains an invalid-instruction warning. Therefore, even apparent straight-line behavior must not be promoted into a validated runtime guarantee.

## Evidence

This document uses only the supplied static representation. The private map resolves the requested function identity but supplies no resolved external dependency contracts.

No implementation was accessed independently. No runtime tests, dependency executions, memory traces, or ABI checks were performed. Acceptance examples express conditional consequences of the visible operations, not observed results.
