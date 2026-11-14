<!-- Trusted provenance: F-5e42dd1dc61e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5e42dd1dc61e

## Purpose

F-5e42dd1dc61e performs a fixed, call-interleaved sequence of word and byte transfers, address transformations, global-state updates, and indirect calls. The visible body contains no explicit conditional branches or loops.

Its application-level purpose is not established. In particular, the evidence does not justify describing it as initialization, serialization, allocation, cleanup, or a supervisor operation. Names attached to global storage are not sufficient evidence of those roles.

The observable contract depends on more than its two declared arguments. It also consumes incoming machine state, secondary results associated with indirect calls, shared storage, and memory at unusually low numerical addresses.

The sequence ends with a halfword clear and an indirect call to numerical address zero with three zero arguments. Normal completion is not established: execution reaches the displayed return only if that final call returns normally.

## Interface

### Declared inputs and output

- Two declared inputs, each 32 bits wide.
- The first declared input has no explicit use in the visible body.
- The second declared input is the sole explicit argument to the first indirect call.
- No explicit return value is provided.

The lack of an explicit use of the first input does not establish that its incoming register contents are unobservable to callees. Calling conventions and complete register effects are unavailable.

### Additional execution inputs

The body also depends on:

- An incoming return-link value used to form the first call target.
- An incoming 32-bit register value used in the first word store.
- Another incoming 32-bit register value whose logical right shift by 22 supplies later word stores.
- Several secondary register results attributed to preceding indirect calls.
- Two separately observed reads of a shared 32-bit global.
- A shared byte at numerical address 1.
- Three bytes participating in a final four-byte global assignment.
- Memory selected by call results and arithmetic on those results.

These are requirements of the displayed execution, not additional conventional parameters that can safely be reconstructed into an ordinary two-argument API.

### Width and address conventions

The following notation describes observable dataflow without assigning application meaning:

- **W(address)**: a 32-bit memory access.
- **B(address)**: an unsigned 8-bit memory access.
- **H(address)**: a 16-bit memory access.
- **Q**: the shared 32-bit global read twice during the sequence.
- **S**: the shared storage receiving an intermediate 32-bit value and, later, a four-byte assembled value.
- **Scale4(value)**: the address-width bit pattern obtained by shifting the 32-bit value left by two.
- **Slot(value)**: address Scale4(value) + 20.

These labels are specification notation, not recovered source identifiers.

Unsigned right shifts are logical. Byte stores retain only the low eight bits of their input. No saturation, range validation, or checked arithmetic is shown.

Several scaled addresses originate from values displayed as signed integers. Their multiplication or left shift can cross signed ranges. Exact processor behavior is not established merely by the displayed high-level types. Acceptance examples involving those operations are conditional on the intended 32-bit address arithmetic, or use values for which the distinction does not matter.

### Preconditions and ownership

Reaching the end requires every executed call target and memory access to be valid in the execution environment. This includes accesses near address zero, potentially unaligned word accesses, and the final call through address zero.

No allocation, release, ownership transfer, reference-count change, or object-lifetime contract is established for any dependency. Passing a numerical address does not establish that the callee owns it or that it is a conventional object pointer.

## Behavior

The following phases preserve the visible ordering of direct effects. Calls between accesses are significant: they may change shared memory, perform additional effects, fail, or not return. No preservation of globals by those calls is assumed.

### 1. Entry and first word stores

The first indirect call receives the second declared input. Its target is formed from the incoming return-link value plus an offset.

A zero-explicit-argument call then produces a 32-bit result, which is passed to another call. A chain of zero-explicit-argument calls follows; the last supplies a value used as a scaled address.

The function:

1. Stores the incoming ambient 32-bit payload at that result’s Slot address.
2. Reads Q and retains its value locally.
3. Calls a dependency with the scaled address, without the additional 20-byte slot displacement.
4. Executes another zero-explicit-argument call chain.
5. Obtains a secondary register result associated with that chain.

Let **A** be the incoming ambient unsigned register shifted right by 22. Thus A is between 0 and 1023.

Let **T** be the retained first Q value shifted right by 23. Thus T is between 0 and 511.

The function stores A at the Slot address selected by the secondary result. The following call receives, in order, that secondary result twice, T, and A.

### 2. Repeated payload and zero stores

A call result is forwarded to another dependency, followed by a zero-explicit-argument chain.

During this phase the function derives T shifted right by 23. This is necessarily zero: T already contains at most nine significant bits. It does not reread Q for this calculation.

The function then:

1. Stores A at a Slot address selected by a primary call result.
2. Calls a dependency with that slot’s scaled base.
3. Executes another call chain.
4. Stores a 32-bit zero at a Slot address selected by an unsigned secondary result.
5. Calls a dependency with that unscaled secondary result.

Retain this secondary result as **K** for the next phase.

### 3. First indirect byte writes

After another forwarded-result call and zero-explicit-argument chain, the function stores T shifted right by 23—therefore zero—at another result-selected Slot address.

It next reads a 32-bit word from:

**((K AND 0x1fffffff) shifted right by 19) + 20.**

The address expression ranges from 20 through 1043 and does not inherently guarantee word alignment.

After further calls, the retained word is shifted right by 22. Call the resulting value **U**, ranging from 0 through 1023.

The function reads a byte from address 1 and uses it as an unsigned address component. It writes the low byte of U at:

**B(1) + 17.**

This destination lies between addresses 17 and 272 inclusive. The byte read from address 1 occurs at this point, not at entry.

A dependency then receives a secondary 32-bit result three times, followed by zero. Subsequent calls include one explicit zero argument.

After another result-producing call, the function reads:

**B(4 × U + 1)**

and stores zero at that byte value plus 17. The primary call result from this point is used as a scaled argument to the next dependency; it does not select this byte-store destination.

### 4. Paired arguments and repeated low-address clears

A call chain produces another result. Its scaled value is supplied twice to a dependency. A subsequent call supplies a 32-bit value, which is forwarded.

After one further call, the function rereads B(1) and writes zero at that value plus 17.

It retains a secondary result as **E**, then calls a dependency with E three times.

A zero-explicit-argument call produces a value passed to another dependency together with the earlier retained 32-bit value shifted right by 23.

After another call chain, the function:

1. Reads B(1) again and retains it as **L**.
2. Stores zero at L + 17.
3. Calls a dependency with a primary result scaled by four.

The rereads of B(1) are distinct observable events. Earlier reads cannot be substituted if intervening callees change that byte.

Another result is then scaled and passed twice to a dependency. The next result, **V**, is written to S before V is forwarded to another call.

### 5. Word-to-byte transfers

After a further call, the function reads W(L + 116) and stores its low byte at L + 1.

This uses the retained L; it does not reread B(1). L may be any unsigned byte, so the word-read address is between 116 and 371 and need not be aligned.

A dependency then receives E shifted left by 11.

The next zero-explicit-argument dependency has a displayed 64-bit result. Its low 32-bit half, high 32-bit half, and V shifted right by 23 are supplied to the next dependency, in that order.

A later call chain selects a scaled base **M**. The function reads W(M + 116), writes its low byte to M + 1, and calls a dependency with M.

A subsequent call chain produces another result, **N**, and retains its scaled form as **P**.

Before the next call, the function performs the transfer from W(M + 116) to B(M + 1) again. This is a new word read. Intervening calls may have changed the source word.

The next dependency receives:

- E shifted left by 12;
- P;
- P again;
- N shifted left by 18.

Further dependencies consume and forward results before the next direct memory transfer.

### 6. Retained word replication

Another call chain supplies a scaled base **R**.

The function reads W(R + 116), retains the full 32-bit word as **J**, and writes J’s low byte at R + 1. A dependency is then called with R.

After another call chain supplies an index, J is stored at:

**16 × index + 20.**

This is a 16-byte stride, unlike the four-byte stride of Slot addresses.

A dependency then receives:

**(P logically shifted right by 2) shifted left by 16.**

This expression must not be replaced with the original N shifted left by 16 without an additional range assumption. Scaling N into the retained 32-bit P may already have discarded high bits.

After result production, forwarding, and another zero-explicit-argument chain, J is stored again at a result-selected Slot address. The following call receives that slot’s scaled base.

### 7. Address-like word stores and the second Q read

A later call result is scaled by four, producing **X**.

The function stores X at:

**(P AND 0xfffffffc) + 20.**

The next dependency receives X twice.

After additional calls, the function stores a 32-bit zero at another result-selected Slot address and calls a dependency with its scaled base.

A subsequent dependency has a displayed 64-bit result. Its low half is scaled by four, producing **Y**. Its high half supplies an index.

The function then:

1. Stores Y at the Slot address selected by that high half.
2. Reads Q again and retains this new value as **Q₂**.
3. Calls a dependency with Y, the high half, Y, and Y.
4. Executes another zero-explicit-argument chain.
5. Stores Y at another result-selected Slot address.
6. Reads W(Q₂ + 20), retaining the word as **Z**.
7. Calls a dependency with the scaled base of the latest Slot address.

The second Q read is independent of the first. Neither the first value nor its shifted forms determine Q₂.

### 8. Final selector-based byte transfers

Another call chain produces a displayed 64-bit result. Retain its low half as **C** and high half as **D**.

The function reads:

**B((J logically shifted right by 23) + 1)**

and writes the low byte of Z at that byte value plus 17.

The selector-read address ranges from 1 through 512.

A dependency receives D three times, followed by C shifted left by two.

After result forwarding and another call chain, the function reads:

**B(((C AND 0x3fffffff) logically shifted right by 21) + 1)**

and writes the low byte of Z shifted right by 22 at that byte value plus 17.

This selector-read address also ranges from 1 through 512. The stored byte corresponds to bits 22 through 29 of Z; bits 30 and 31 do not survive the byte store.

Subsequent calls receive a result scaled by four, then another scaled result twice. Further result production and forwarding lead to a final dependency with a displayed byte result.

### 9. Terminal updates and final call

After the final byte-producing call returns, the direct effects occur in this order:

1. Store zero through a pointer supplied as a secondary register result.
2. Assign S a four-byte value composed of the final byte result as its least-significant byte and three shared bytes as its upper portion.
3. Store a 16-bit zero at address **(C AND 0x3fffffff) logically shifted right by 7**.
4. Call numerical address zero with three explicit zero arguments.
5. Return only if that call returns normally.

The halfword destination ranges from zero through 8,388,607. Even alignment is not guaranteed.

The final S assignment is not established as a read-modify-write preserving S’s existing upper bytes. Its upper portion comes from separately referenced shared storage.

## Side effects

Established direct effects include:

- Multiple 32-bit writes to scaled-result addresses with a 20-byte displacement.
- One 32-bit write using a 16-byte index stride.
- Byte writes selected through unsigned byte reads.
- Repeated word reads followed by low-byte stores.
- Two writes to S, separated by many calls and memory effects.
- A final pointer-directed byte clear.
- A final 16-bit clear.
- Numerous indirect calls with unknown additional effects.

Address expressions may alias one another, Q, S, selector bytes, or dependency-managed state. No disjointness is established. Consequently, later reads must observe the memory state that exists at their own position in the sequence.

No explicit error result, validation branch, rollback, or recovery path appears. If a memory operation faults or a dependency does not return, the later effects are not established to occur. Earlier effects are not undone by any visible mechanism.

## Dependencies

No called implementation is supplied. The following opaque IDs identify dependency **groups by observable interface role**, not proven shared callable identities:

| Opaque ID | Established interface role |
|---|---|
| F-000000000001 | Entry call receiving the second declared input. |
| F-000000000002 | Zero-explicit-argument calls, including result-producing calls. |
| F-000000000003 | Calls consuming forwarded 32-bit results or scaled bases. |
| F-000000000004 | Calls consuming repeated arguments or three-/four-component tuples. |
| F-000000000005 | Calls with displayed 64-bit results subsequently split into halves. |
| F-000000000006 | Final byte-result-producing call. |
| F-000000000007 | Final call to numerical address zero with three zero arguments. |

Membership in a group does not imply identical targets, shared implementation, purity, or interchangeable behavior.

Most call targets are formed from position-associated address values plus offsets; the first instead depends on incoming return-link state. Target resolution, executable mappings, calling conventions, clobbers, exceptions, and secondary-result guarantees remain unknown.

“Zero-explicit-argument” means only that no arguments are displayed. It does not establish that a dependency consumes no ambient machine state.

## Acceptance examples

These are conditional static examples, not executed tests. They assume preceding calls return and the referenced accesses are valid.

1. **First argument independence.** Changing only the first declared argument changes no explicit argument use or direct arithmetic in the visible body. Whole-execution equivalence additionally requires that dependencies cannot observe the changed incoming state.

2. **Ambient high-bit extraction.** An incoming unsigned ambient value of 0xffc00000 yields A = 1023. The associated word stores contain 1023, not a sign-extended negative value.

3. **Repeated shift becomes zero.** If the first Q value is 0xffffffff, T is 511. Shifting T right by 23 yields zero, as it does for every possible first Q value.

4. **Byte truncation.** If U is 0x3ab and the relevant B(1) read returns 7, the destination is address 24 and the stored byte is 0xab.

5. **No cached selector substitution.** If one B(1) read returns 7 and a later one returns 9, their corresponding destinations are 24 and 26. Reusing the first selector violates the visible read order.

6. **Repeated word reads are distinct.** If W(M + 116) changes from 0x11223344 to 0xaabbccdd between the two transfers, the respective byte writes are 0x44 and 0xdd.

7. **Full-word replication.** If J is 0xdeadbeef, the stride-16 store and later Slot store both receive 0xdeadbeef. The preceding byte transfer receives only 0xef.

8. **Second Q observation.** If dependencies change Q between its reads, the later W(Q₂ + 20) access uses the second observed value, not the first.

9. **Final shifted-byte selection.** If Z is 0xffc00000, the later selector-based byte store writes 0xff: the shift yields 1023 and byte truncation retains 255.

10. **Terminal failure boundary.** If the call to address zero faults, normal return is not an acceptance requirement. The preceding direct writes have already been attempted in their specified order.

## Uncertainties

### Established static evidence

The visible body establishes widths of direct accesses, arithmetic expressions, explicit call arguments, direct-effect ordering, two Q reads, two S writes, and the final zero-address call.

### Inferred intent

The repeated scaling and displacement patterns suggest structured storage access. That does not establish a record layout, handle representation, instruction encoding, or hardware interface.

### documented_only

The following remain unresolved:

- Complete calling convention and ambient-register inputs.
- Exact producers and validity of secondary register results.
- Whether displayed 64-bit results represent language-level integers or paired machine results.
- Exact signed-overflow and address-wrap semantics.
- Endianness and alignment requirements.
- Shared-storage overlap beyond the supplied warning that global labels may overlap.
- Allocation, ownership, release, and lifetime behavior.
- Dependency effects on shared memory and control flow.
- Validity and purpose of low-address accesses.
- Whether address zero is executable, trapping, redirected, or otherwise special.
- Whether execution reaches the displayed return.

## Evidence

This contract is based solely on the supplied static representation of F-5e42dd1dc61e. No dependency implementation, external calling-convention specification, memory map, or runtime observation was provided.

No runtime tests have been performed. Acceptance examples describe consequences of the visible operations under stated preconditions; they are not execution results.
