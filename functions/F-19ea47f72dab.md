<!-- Trusted provenance: F-19ea47f72dab; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-19ea47f72dab

## Purpose

F-19ea47f72dab exposes a recovered sequence consisting of two byte writes, ten indirect calls, and an endpoint at which instruction recovery fails. The first write occurs before all ten calls. The second occurs between the fifth and sixth calls.

The two writes select different eight-bit portions of the same implicit 32-bit input. Each write independently obtains its destination from a byte read through another implicit input. The calls also carry two visible data dependencies: the first receives a shifted explicit input, and the sixth receives a shifted result from the fifth.

No application-level purpose is established. In particular, the evidence does not establish that the writes configure a device, encode a message, or update an ordinary in-memory object.

This contract describes the visible recovered sequence. It does not establish that the sequence is executable as recovered or that the apparent terminal action accurately represents the original program.

## Interface

### Explicit inputs

The recovered interface has three explicit input slots:

| Input | Recovered width | Visible use |
|---|---:|---|
| First explicit input | 32 bits | No direct use is shown. |
| Second explicit input | 32 bits | No direct use is shown. |
| Third explicit input | 32-bit signed integer | Shifted left by three positions to form the visible argument to the first call. |

The absence of a direct use of the first two inputs does not establish that their incoming machine-register contents are unobservable to indirect callees. The calling convention and complete call signatures are not supplied.

### Implicit inputs

The recovered body also depends on incoming state outside the explicit parameter list:

- **Payload word:** a 32-bit unsigned value used by both byte writes.
- **Selector base:** an address-bearing value represented as a 32-bit signed integer. A byte is read at an offset of one from this value.
- **Dispatch base:** a link-register-derived incoming value used to calculate the first indirect call target.

The source does not establish how callers initialize these values or whether the explicit interface adequately describes the actual entry convention.

### Output and completion

No normal return value is shown. One intermediate dependency result is treated as a 32-bit signed integer and used to construct a later call argument.

After the tenth call, the recovered representation ends at an invalid-instruction marker. It does not show an ordinary return. That marker establishes a recovery boundary, not a verified runtime exception, process termination, or hardware halt.

### Memory layout visible to this function

Only the following layout is established:

- A one-byte selector is read at `selector base + 1`.
- That selector is interpreted as an unsigned value from 0 through 255.
- The destination address for a one-byte write is the selector value plus 17.

Consequently, the displayed destination calculation ranges from address 17 through address 272, inclusive. The recovered expression does not add the selector base to that destination. Treating the selector as an offset into an object at the selector base would change the visible behavior.

Whether these low addresses denote ordinary memory, mapped device state, inaccessible storage, or incorrectly recovered addressing is unknown.

## Behavior

### 1. Initial selector read and byte write

Before any indirect call, the function reads one byte from `selector base + 1`.

It then writes the low eight bits of the payload word to the address obtained by adding 17 to the unsigned selector byte.

The write is one byte wide. Although the recovered representation converts the payload to a character-sized value, the byte pattern being stored is its low eight bits; the signedness of a character interpretation does not change that pattern.

This selector read and destination write precede the first call on the displayed execution path.

### 2. First call

The first indirect target is calculated from the incoming dispatch base by adding 4,076.

The visible argument is the third explicit input shifted left by three bit positions. For nonnegative inputs whose mathematical product by eight remains representable as a signed 32-bit integer, the argument is that product.

The representation does not settle negative-input or signed-overflow semantics. In particular, it is not sufficient evidence to promise either unrestricted mathematical multiplication or modulo-\(2^{32}\) wrapping for every input.

No result from this call is visibly consumed by the remainder of the recovered body.

### 3. Four further calls

Four more indirect calls follow, in this order:

1. F-000000000002
2. F-000000000003
3. F-000000000004
4. F-000000000005

These are calls two through five overall. Each target is calculated from a fixed literal in the recovered body, rather than from the incoming dispatch base.

No explicit arguments are shown for these four calls. This means only that no arguments are represented in their displayed call expressions; it does not establish complete zero-argument interfaces.

The result of F-000000000005 is retained as a 32-bit signed integer. No other result from this group is visibly used.

### 4. Second selector read and byte write

After the fifth call returns, the function reads the byte at `selector base + 1` again. It does not visibly reuse the selector read before the first call.

The new selector determines the second destination by the same unsigned-byte-plus-17 calculation.

The stored byte is obtained by logically shifting the unsigned payload word right by 22 positions and retaining the low eight bits. It therefore contains original payload bits 22 through 29:

- Original bit 22 becomes stored bit 0.
- Original bit 29 becomes stored bit 7.
- Original bits 30 and 31 do not contribute to this write.
- Original bits 0 through 21 do not contribute to this write.

The recovered dataflow uses the same payload input for both writes. No intervening assignment to that payload is shown.

The destination may differ from the first destination because the selector is reread after five calls. If both selector reads return the same value, both writes target the same address. In ordinary writable memory, with no other intervening modification, the second write replaces the byte written by the first.

### 5. Sixth call

The next call is F-000000000006. Its visible argument is the retained result of F-000000000005 shifted left by two positions.

For a nonnegative retained result whose mathematical product by four fits in a signed 32-bit integer, the argument is that product.

Negative values and signed-overflow cases remain unresolved for the same reason as the first call’s shift. The evidence does not justify silently choosing wrapping, trapping, or another machine-specific interpretation.

The second byte write occurs before this call.

### 6. Final four calls and recovery boundary

Four more indirect calls follow:

1. F-000000000007
2. F-000000000008
3. F-000000000009
4. F-00000000000a

These are calls seven through ten overall. Their targets are derived from fixed literals. No explicit arguments are shown, and no returned values are visibly consumed.

After call ten, the displayed control flow reaches an invalid-instruction recovery boundary. No subsequent behavior is established.

### Conditional nature of the sequence

Every later operation is conditional on preceding operations completing and calls returning to the displayed continuation.

There is no visible branch that skips a call or chooses an alternate normal path. Nevertheless, a memory fault, invalid call target, nonreturning dependency, or other behavior inside a dependency can prevent later operations from occurring. This function shows no local recovery path for those conditions.

## Side effects

### Directly visible effects

If the entire recovered sequence is reached and each operation completes, the body directly performs:

- Two one-byte selector reads.
- Two one-byte destination writes.
- Ten indirect calls, in the stated order.

These counts exclude effects performed by dependencies. They also do not describe instruction fetches or other implementation-level accesses.

The source establishes program-order placement of the direct operations. It does not establish volatile semantics, atomicity, memory barriers, inter-thread synchronization, or device-level ordering guarantees.

### Dependency effects

The callees may have effects not described here. No preservation guarantee is available for global memory, selector storage, destination storage, external state, or machine registers.

The second selector read is especially important: changes made during the first five calls can affect the second destination. The contract must not cache the initial selector on the assumption that dependencies leave it unchanged.

The recovered dataflow reuses the incoming payload and selector base. Whether the actual calling convention preserves the machine state required to realize that dataflow is not established by the supplied material.

### Ownership and lifetime

No allocation, explicit release, ownership transfer, or object-lifetime operation is directly shown.

The function requires access to the selector byte and whichever destination bytes are selected if execution is to continue without an access failure. That requirement does not establish ownership of those locations.

The call sites do not provide enough information to determine whether dependencies allocate, retain, transfer, or release memory.

### Error handling

No explicit validation, error return, retry, rollback, or cleanup path is visible.

If execution stops after an earlier write, no visible mechanism restores the previous byte. Likewise, nothing shown reverses side effects of completed dependencies.

## Dependencies

The following opaque IDs are local contract labels for the distinct visible dependency sites. Their implementations and original identities are unavailable.

| Dependency | Call position | Target basis | Visible argument | Visible result use |
|---|---:|---|---|---|
| F-000000000001 | 1 | Incoming dispatch base plus 4,076 | Third explicit input shifted left three positions | None |
| F-000000000002 | 2 | Fixed literal-derived target | None shown | None |
| F-000000000003 | 3 | Fixed literal-derived target | None shown | None |
| F-000000000004 | 4 | Fixed literal-derived target | None shown | None |
| F-000000000005 | 5 | Fixed literal-derived target | None shown | Retained as a 32-bit signed value |
| F-000000000006 | 6 | Fixed literal-derived target | Retained result shifted left two positions | None |
| F-000000000007 | 7 | Fixed literal-derived target | None shown | None |
| F-000000000008 | 8 | Fixed literal-derived target | None shown | None |
| F-000000000009 | 9 | Fixed literal-derived target | None shown | None |
| F-00000000000a | 10 | Fixed literal-derived target | None shown | None |

The fixed target expressions use different literals. This establishes distinct displayed target calculations, not necessarily distinct semantic operations under every possible execution environment.

No dependency is established to preserve globals, return normally, accept only the displayed arguments, or have any particular ownership behavior.

## Acceptance examples

These are conditional specification examples, not executed tests.

### Stable selector, different payload portions

Assume:

- Both selector reads return 3.
- The payload word is `0x0A80005A`.
- All accesses succeed.
- All ten dependencies return to their displayed continuations.

The first direct write stores `0x5A` at address 20. After call five, the second direct write stores `0x2A` at address 20.

This example distinguishes the second extraction from either the top byte or a shift by a byte-aligned amount. It does not constrain intervening dependency writes to that address.

### Selector changed by an intervening dependency

Assume the initial selector is 1, and an effect during the first five calls causes the second selector read to return 200.

The first direct destination is address 18. The second direct destination is address 217. Reusing the first destination would not satisfy the displayed sequence.

### Selector boundaries

A selector byte of zero selects address 17. A selector byte of 255 selects address 272.

Interpreting 255 as a signed negative byte, or adding the selector base to either destination, would not match the visible calculation.

### Representable shifted arguments

Assume the third explicit input is 7, F-000000000005 returns 9, and execution reaches both argument-bearing calls.

F-000000000001 receives the visible argument 56. After the second byte write, F-000000000006 receives the visible argument 36.

No comparable unconditional numeric expectation is established here for negative or overflowing shift inputs.

### Payload bits excluded from the second write

With a payload of `0xC0000000`, the first stored byte is zero and the second stored byte is also zero. The payload’s two highest bits do not belong to the second extracted field.

### A dependency does not return

If F-000000000005 never returns, the first direct write has already occurred, but the second selector read, second direct write, and calls six through ten are not reached on the displayed path.

No cleanup behavior is shown for that case.

## Uncertainties

The following items are **documented_only**:

- **Instruction validity:** recovery warnings occur at the function level and at the terminal boundary. Executability of the recovered sequence is unverified.
- **Actual entry convention:** the relationship between the explicit inputs, implicit incoming state, and machine calling convention is unknown.
- **Address semantics:** address width, overflow behavior for address calculations, memory mapping, alignment requirements, and access permissions are not fully established.
- **Signed left shifts:** negative operands and results outside the signed 32-bit range have no established runtime semantics in this material.
- **Dependency contracts:** complete signatures, side effects, termination behavior, error behavior, and ownership rules are unavailable.
- **State preservation:** no assumption may be made that unknown callees preserve globals or registers.
- **Concurrency and device semantics:** external mutation, volatile behavior, atomicity, and ordering beyond the displayed sequence are unknown.
- **Terminal outcome:** the recovery boundary does not prove a particular runtime halt or exception.
- **Application intent:** no supported domain-level interpretation is available.

These unknowns limit executable guarantees; they must not be replaced with conventional defaults without additional evidence.

## Evidence

**Established static evidence:** the supplied recovered representation shows two selector reads, two byte writes, ten indirect calls, two shifted call arguments, one retained dependency result, and no ordinary return before an invalid-instruction boundary. Its recovered types support the stated 32-bit payload, signed intermediate values, unsigned selector interpretation, and eight-bit writes.

**Inferred intent:** the two writes may be selecting separate fields from one payload, and the surrounding calls may form a staged operation. Neither interpretation establishes an application-level purpose or additional behavior.

**Evidence limits:** only the supplied static representation was available. No dependency bodies, calling-convention documentation, executable validation, or runtime observations were supplied. No runtime tests have been performed.
