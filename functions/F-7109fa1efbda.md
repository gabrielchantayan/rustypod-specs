<!-- Trusted provenance: F-7109fa1efbda; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7109fa1efbda

## Purpose

F-7109fa1efbda describes a state-dependent sequence of indirect calls interleaved with four single-byte memory writes. Selected call results determine write addresses and arguments to subsequent calls.

No application-level purpose is established. The available evidence does not identify the written locations as ordinary memory, device registers, buffers, or object fields. It also does not establish that the repeated call patterns perform equivalent operations.

This contract distinguishes:

- **Established static evidence:** operations and dependencies represented in the supplied listing.
- **Conditional interpretation:** arithmetic and execution behavior requiring a confirmed machine model.
- **documented_only unknowns:** properties not established by that listing.

The supplied representation explicitly reports invalid instruction data and truncated control flow. Accordingly, this is a contract for the represented execution prefix, not a claim that the complete executable function has been recovered correctly.

## Interface

### Entry state

The displayed interface has no declared parameters and no declared return value. Nevertheless, the represented behavior depends on three incoming machine-state values:

| Input | Represented width | Use |
|---|---:|---|
| Entry control-link value | 32-bit address-sized value, subject to architecture confirmation | Determines the first indirect-call target after addition of 4,094 |
| Entry auxiliary word | 32 bits | Supplies the alternative address calculation for the first write |
| Entry payload word | 32 bits, unsigned | Supplies the first byte written and a later extracted byte |

These are ambient machine-state inputs, not established public ABI parameters. Their availability and preservation across dependency calls are not independently verified.

Call results used for arithmetic are represented as 32-bit words. Other forwarded results are represented as opaque four-byte values. No structure layout, string encoding, buffer length, or object type is established.

### Outputs

There is no established normal return value or normal return path in the supplied prefix.

Observable outputs consist of:

- Four direct byte writes, if execution reaches them.
- Arguments delivered to indirect dependencies.
- Any effects those dependencies perform.
- Whatever happens at the unrecovered terminal boundary.

The terminal boundary must not be described as a successful return, a defined application error, or a guaranteed processor halt.

### Preconditions

To execute the entire represented prefix, the environment must permit all indirect transfers, all four writes, and all intervening dependency returns. This is a condition for observing the full sequence, not validation performed by the function.

No bounds check, pointer-validity check, alignment check, allocation check, or dependency-status check is visible.

## Behavior

### Arithmetic interpretation

The listing uses 32-bit integer quantities, shifts, multiplication, and integer-derived addresses. Some calculations are expressed through signed intermediates. The listing alone therefore does not establish portable language-level overflow behavior.

For precise conditional examples, this document uses the following **32-bit machine interpretation**:

- Left shifts discard bits above bit 31.
- Multiplication used in address or argument formation retains the low 32 bits.
- Address arithmetic is interpreted in a 32-bit address domain.
- A byte write retains the low eight bits of its value.
- Right shifting the unsigned payload word by 22 positions is logical.

Under that interpretation, scaling by four or eight is modulo \(2^{32}\), not unbounded mathematical multiplication. This interpretation requires confirmation against the actual instruction set and executable bytes.

### Dependency notation and ordering

The locally assigned opaque dependency IDs **F-D001 through F-D221** identify successive indirect-call occurrences. They are documentation labels, not externally verified function identities. Different labels do not establish different implementations or non-aliasing targets.

A call described as having “no explicit arguments” has no arguments shown in the supplied representation. That does not establish that the actual callee receives no register values, reads no ambient state, or has a zero-argument ABI.

A repeated group below consists of four calls without explicit arguments followed by one call receiving a transformed result from the fourth call. Each occurrence remains ordered and independently observable; the grouping does not authorize combining or omitting calls.

### First write and initial call segment

F-D001 is invoked first. Its target is formed from the entry control-link value plus 4,094. Its 32-bit result selects the first write address:

- If bit 30 of that result is set, the intermediate address quantity is the result scaled by four.
- If bit 30 is clear, the intermediate address quantity is the entry auxiliary word shifted left by ten positions.

The first write stores the low byte of the entry payload word at twice this intermediate quantity.

Thus, under the 32-bit machine interpretation:

| Selection | First write address | Following argument value |
|---|---|---|
| Bit 30 set | Eight times the F-D001 result | Four times the F-D001 result |
| Bit 30 clear | Entry auxiliary word shifted left by eleven | Entry auxiliary word shifted left by ten |

The write precedes F-D002. F-D002 receives the intermediate quantity twice, in two explicit argument positions.

The rest of this segment has the following dependency structure:

| Dependency IDs | Observable contract |
|---|---|
| F-D003–F-D006 | Four ordered calls without explicit arguments |
| F-D007 | Receives the opaque four-byte result of F-D006 unchanged |
| F-D008–F-D052 | Nine ordered five-call groups; each group’s fifth call receives the fourth call’s result scaled by four |
| F-D053–F-D056 | Four ordered calls without explicit arguments; the result of F-D056 determines the second write and the next two arguments |

No return-status branch is visible in this segment. Discarded results are not evidence that the calls themselves have no effects.

### Second write and second call segment

The second direct write occurs after F-D056 returns and before F-D057 begins.

Its address is the F-D056 result shifted left by three positions. Its value is the low byte remaining after logically shifting the entry payload word right by 22 positions. Consequently, payload bits 22 through 29 determine the stored byte; bits 30 and 31 do not contribute.

F-D057 receives two explicit arguments derived from the same F-D056 result:

1. The result scaled by four.
2. The result shifted left by ten positions.

The remaining calls in this segment are:

| Dependency IDs | Observable contract |
|---|---|
| F-D058–F-D066 | Nine ordered calls without explicit arguments |
| F-D067 | Receives the F-D066 result scaled by four |
| F-D068–F-D107 | Eight ordered five-call groups, each forwarding four times its fourth call’s result |
| F-D108–F-D111 | Four ordered calls without explicit arguments; the final result determines the third write and the next arguments |

The nine-call lead-in is significant. It must not be reduced to the four-call lead-ins used elsewhere.

### Third write and third call segment

The third direct write stores a zero byte at the address obtained by shifting the F-D111 result left by three positions.

This write precedes F-D112. F-D112 receives two identical explicit arguments, each equal to the F-D111 result shifted left by fourteen positions.

The subsequent sequence is:

| Dependency IDs | Observable contract |
|---|---|
| F-D113–F-D116 | Four ordered calls without explicit arguments |
| F-D117 | Receives the opaque four-byte result of F-D116 unchanged |
| F-D118–F-D162 | Nine ordered five-call groups, each forwarding four times its fourth call’s result |
| F-D163–F-D166 | Four ordered calls without explicit arguments; the final result determines the fourth write and the next arguments |

The zero write is exactly one byte. It does not establish clearing of a word, object, range, or allocation.

### Fourth write and final represented segment

The fourth direct write stores a zero byte at the address obtained by shifting the F-D166 result left by three positions.

F-D167 follows that write and receives the F-D166 result shifted left by fourteen positions in both explicit argument positions.

The final represented calls are:

| Dependency IDs | Observable contract |
|---|---|
| F-D168–F-D171 | Four ordered calls without explicit arguments |
| F-D172 | Receives the opaque four-byte result of F-D171 twice |
| F-D173–F-D217 | Nine ordered five-call groups, each forwarding four times its fourth call’s result |
| F-D218–F-D221 | Four ordered calls without explicit arguments |

F-D172 differs from F-D007 and F-D117: it receives two copies of the forwarded value, not one.

After F-D221, the supplied representation reaches an invalid-instruction/truncated-control-flow boundary. No subsequent behavior is established.

## Side effects

### Direct memory effects

Exactly four direct writes are visible in the represented prefix. Each has width one byte.

Their order is:

1. Low byte of the entry payload word.
2. Payload bits 22 through 29.
3. Zero.
4. Zero.

There are no explicit direct data-memory reads in the displayed body apart from whatever is implicit in execution and indirect calls. This does not imply that dependencies perform no reads.

There is no read-before-write requirement for the four destinations. The writes do not preserve selected bits within their target byte.

The destinations may coincide. If they do, later direct writes replace earlier byte values, subject to intervening dependency effects. No non-aliasing guarantee is established between destinations, executable storage, or dependency-accessible memory.

### Ownership and shared state

No direct allocation, release, ownership transfer, or lifetime management is visible.

Passing an integer-derived argument does not establish that it is a borrowed pointer, owned pointer, size, handle, or offset. A dependency may access or mutate shared state, allocate memory, release memory, or terminate execution; none of those behaviors can be excluded from the supplied evidence.

The function’s use of entry-state values later in the sequence does not independently prove that the actual calling convention preserves their physical storage across calls.

### Failure behavior

There is no visible local recovery, rollback, retry, or error-result interpretation.

A failed write or nonreturning dependency may prevent later operations. Earlier observable effects need not be undone. Dependency behavior on faults or exceptional control transfer is documented_only.

## Dependencies

All represented calls are indirect.

F-D001 has an entry-state-dependent target. Later targets are represented using embedded address-like values or code-reference expressions with the same 4,094-byte displacement. Their actual executable validity, instruction-state interpretation, and identity are unresolved.

The contract requires preservation of:

- Call occurrence order.
- Four-byte result forwarding where shown.
- Explicit argument count and argument position as represented.
- The distinction between unchanged forwarding, scaling by four, shifting by ten, and shifting by fourteen.
- Each direct write’s position relative to surrounding calls.

It does not assign undocumented functional meaning to any dependency. In particular, repeated five-call patterns do not prove repeated invocation of the same five functions.

The 221 call occurrences are a static count of the supplied representation, conditional on reaching each occurrence. They are not an observed runtime trace.

## Acceptance examples

These are conditional review examples, not executed tests. Numerical examples assume the 32-bit machine interpretation, valid destinations, and dependencies that return normally.

### First selection uses the auxiliary input

If F-D001 returns `0x00000003`, bit 30 is clear. With entry auxiliary word `0x00000002` and payload word `0x123456AB`:

- The first write address is `0x00001000`.
- The byte written is `0xAB`.
- F-D002 receives `0x00000800` in both explicit argument positions.

The nonzero F-D001 result does not select its scaled address when bit 30 is clear.

### First selection uses the call result

If F-D001 returns `0x40000001`:

- The intermediate quantity is `0x00000004`.
- The first write address is `0x00000008`.
- F-D002 receives `0x00000004` twice.

This example demonstrates discarded high bits. It does not establish that the low address is writable.

### Second write extracts only eight payload bits

If the entry payload word is `0xFFC00000` and F-D056 returns `0x00000003`:

- The second write address is `0x00000018`.
- The stored byte is `0xFF`.
- F-D057 receives `0x0000000C` followed by `0x00000C00`.

Changing only payload bits 30 and 31 does not change this stored byte.

### Repeated-group arithmetic retains word width

If a group’s fourth call returns `0x40000001`, the group’s fifth call receives `0x00000004` under the conditional arithmetic model.

The preceding three calls still occur. Their results being unused does not permit their omission.

### Zero writes precede their consumers

If F-D111 returns `0x00000003`, the third write stores zero at `0x00000018` before F-D112 receives `0x0000C000` twice.

A trace placing that write after F-D112 would not match the represented ordering.

### Unchanged forwarding is not scaling

If F-D171 produces the four-byte pattern `0x81234567`, F-D172 receives that same pattern in both explicit positions. Neither truncation to a byte nor multiplication by four is specified at that boundary.

## Uncertainties

The following remain **documented_only**:

- Actual function boundaries and complete terminal behavior.
- Whether invalid instruction data caused incorrect instruction decoding, call reconstruction, or control-flow reconstruction.
- Exact processor architecture, address width, instruction-state conventions, and arithmetic behavior.
- Public ABI and interpretation of the incoming machine state.
- Whether the displayed no-argument calls consume implicit register or global inputs.
- Dependency identities, target validity, side effects, ownership behavior, and failure semantics.
- Whether the apparent direct writes target ordinary memory or special-purpose address space.
- Valid address ranges and required execution privileges.
- Concurrency, reentrancy, synchronization, and atomicity guarantees.

A sequencing or orchestration role is a possible inferred intent. No narrower domain interpretation is justified.

## Evidence

This document is based solely on the supplied static representation. No implementation access, executable inspection, or runtime tests have been performed.

The representation supports the reported bit-30 selection, four single-byte writes, payload-byte extraction, result transformations, ordered dependency groups, and explicit forwarding relationships.

Its explicit invalid-instruction and truncation warnings limit confidence in treating that representation as a complete executable specification. The numerical examples validate the stated conditional arithmetic relationships only; they are not evidence that any address is accessible or any dependency returns.
