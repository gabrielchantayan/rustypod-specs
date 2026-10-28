<!-- Trusted provenance: F-7216eddec5d8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7216eddec5d8

## Purpose

F-7216eddec5d8 has a narrowly established effect: it attempts two assignments to global storage, followed by a 16-bit write whose address and value derive from an implicit machine-state input. The supplied representation then reaches an undecodable instruction boundary. It does not establish a normal return or any behavior beyond that boundary.

No higher-level purpose is established. In particular, the evidence does not justify describing this operation as initialization, exception recovery, a reset handler, or a device-control operation.

This contract distinguishes the effects directly represented in the supplied material from assumptions needed to interpret machine widths and from behavior that remains unknown. No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface accepts three ordered 4-byte arguments:

| Input | Established use |
|---|---|
| First argument | No use is shown before the undecodable boundary. |
| Second argument | No use is shown before the undecodable boundary. |
| Third argument | Supplies the value assigned to the second global-storage destination. |

The arguments have no established signedness, pointer interpretation, ownership, or application-level meaning. The third argument is treated as a 4-byte value in the supplied representation; there is no evidence of dereferencing it.

The lack of visible uses for the first two arguments applies only to the represented prefix. It does not establish that those arguments are irrelevant to any execution after the undecodable boundary.

### Implicit input

An incoming register value supplies both the address calculation and the data for the final visible write. Its initialization is not shown, and it is not one of the three explicit arguments.

For the arithmetic described below, this contract calls it the **incoming word** and interprets it as an unsigned 32-bit value. That interpretation is consistent with the supplied unsigned-word representation, but the underlying type definitions and target-machine specification were not supplied. The width assumption is therefore explicit rather than silently treated as independently verified.

A caller relying on the final write must arrange the incoming word through the actual machine interface. The source-level three-argument signature alone does not describe how to do so.

### Global destinations

Two global-storage destinations are identified here as:

- **Global destination A:** assigned zero first.
- **Global destination B:** assigned the third argument second.

These labels denote storage destinations, not semantic roles. Their addresses, allocation sizes, alignment, and relationship to one another are not established.

The supplied material warns that some global symbols overlap smaller symbols at the same addresses. Consequently, the two assignments cannot safely be interpreted as updates to independent, non-overlapping objects.

### Outputs and completion

No explicit return value is represented. There is no demonstrated successful-return path.

The visible effects are storage writes. The final boundary is reported as bad instruction data, but the evidence does not establish whether actual execution would trap, continue through an unrecognized instruction, transfer control, or behave otherwise.

## Behavior

### Ordered visible prefix

Subject to execution reaching each operation and the relevant accesses completing, the represented order is:

1. Assign zero to global destination A.
2. Assign the third explicit argument to global destination B.
3. Perform a 16-bit write at an address derived from the incoming word.
4. Reach the boundary beyond which the supplied representation provides no reliable control flow.

This is the ordering shown by the representation. It is not a separately verified claim about processor barriers, bus transactions, device visibility, or compiler ordering.

There is no visible condition selecting among these three writes. There is also no visible validation before them.

### Global assignments

The first assignment supplies the numeric value zero to global destination A. The second supplies the third argument to global destination B.

The widths of the global destinations are not declared in the supplied material. The third argument has an established 4-byte representation, but that alone does not prove the second assignment writes exactly four bytes: the destination type and machine instruction are unavailable. The corresponding write width, any conversion, and the exact bytes affected remain documented_only.

Likewise, assigning zero does not establish how many bytes are cleared at global destination A.

The two operations remain ordered even if their storage overlaps. An observer must not assume the first destination still contains an independently readable zero after the later assignments.

### Derived address

Under the unsigned 32-bit interpretation of the incoming word:

- Discard its lowest 22 bits.
- Multiply the remaining value by 256.
- Use the result as the numeric destination address for the 16-bit write.

The retained value ranges from 0 through 1,023. Therefore, the derived address ranges from **0x00000000 through 0x0003FF00**, inclusive, in steps of **0x100**.

The address calculation does not use either of the first two explicit arguments, the third argument, or a visible base pointer.

The multiplication cannot overflow a 32-bit unsigned value: its maximum result is 261,888. Every derived address is divisible by 256 and is therefore also divisible by two. This arithmetic alignment does not establish that the address is mapped, writable, or suitable for a halfword access.

The representation uses the computed integer directly as an address. The actual integer-to-address mapping is target-dependent and has not been independently established.

### Derived write value

The final visible operation stores a 16-bit quantity derived from the incoming word.

Under the ordinary machine interpretation of narrowing to a 16-bit store, the written bit pattern is the incoming word’s lowest 16 bits. Its possible bit patterns range from **0x0000 through 0xFFFF**.

The supplied representation presents a signed 16-bit conversion before the store. For values whose low halfword is at least 0x8000, interpreting that conversion at a language level requires implementation details not supplied here. The low-halfword description is therefore conditional on ordinary machine narrowing semantics, rather than a portable-language guarantee.

The address and stored data use disjoint portions of the incoming word:

- Bits 31–22 select the destination.
- Bits 15–0 supply the stored halfword.
- Bits 21–16 do not affect either result in the represented calculation.

No byte order is established. A 16-bit value can be specified without asserting which byte appears at the lower address.

### Failure and incomplete execution

There is no visible address check, permission check, recovery branch, error return, or rollback.

If an earlier write faults or otherwise prevents continuation, the later operations are not established to occur. If the final write faults, the two preceding assignments may already have taken effect.

The evidence does not show transactional behavior. Earlier changes must not be assumed to be undone following a fault or the undecodable boundary.

## Side effects

The visible side effects are limited to the three attempted storage updates described above. Their actual consequences depend on the identity and mapping of the destinations.

The computed address may refer to ordinary memory, memory-mapped hardware, protected storage, or an unmapped location. None of those possibilities is selected by the supplied evidence. In particular, address zero is a possible calculated result; the function does not visibly reject it.

Aliasing remains significant:

- Global destinations A and B may overlap.
- The computed destination may overlap either global destination.
- Other symbols may expose smaller portions of the same storage.

If the final 16-bit write overlaps storage changed earlier, it may replace some of the bytes affected by an earlier assignment. Exact resulting global values cannot be specified without destination addresses, widths, and byte order.

No allocation, release, ownership transfer, buffer-length handling, logging, or synchronization operation is shown. This absence describes only the represented prefix, not behavior beyond the undecodable boundary.

## Dependencies

No ordinary callable dependency is established in the visible prefix.

The marker at the terminal boundary reports an inability to represent subsequent instructions reliably. It is not sufficient evidence of a real application function call, a particular exception, or a specified process-termination mechanism.

Execution depends on environmental prerequisites rather than identified helper functions:

- The calling convention must supply the three explicit arguments and the implicit incoming word.
- The two global destinations must exist in the execution environment.
- The calculated address must support the attempted 16-bit write if that operation is to complete.
- The target’s integer widths, address interpretation, and narrowing semantics must match the assumptions used for the numerical description.

No unknown dependency is assumed to preserve global state, allocate storage, or release memory.

## Acceptance examples

These examples describe conditional observations of the visible prefix. They are not executed tests and do not establish successful function completion.

### Zero incoming word

Given an incoming word of **0x00000000**, the calculated address is **0x00000000**, and the final halfword value is **0x0000**.

If the first two assignments complete, the next visible operation attempts that write at address zero. No validation or redirection is shown. Whether the access succeeds is environmental.

### Small nonzero address selector

Given an incoming word of **0x00401234**, the calculated address is **0x00000100**, and the final halfword value is **0x1234**.

A compatible execution should attempt the two global assignments first, then the halfword write. The example does not prescribe the byte order of 0x1234.

### Bits unused by the calculation

Incoming words **0x00401234** and **0x007F1234** produce the same calculated address, **0x00000100**, and the same halfword value, **0x1234**.

Their difference lies in bits discarded from the address calculation and outside the stored halfword. This equivalence applies only to the represented calculation, not to unknown behavior beyond the boundary.

### Maximum incoming word

Given an incoming word of **0xFFFFFFFF**, the calculated address is **0x0003FF00**. Under ordinary machine narrowing semantics, the final stored bit pattern is **0xFFFF**.

The address multiplication does not wrap under the 32-bit assumption. Successful access and subsequent execution remain unknown.

### Changing explicit arguments

With the incoming word and third argument held fixed, changing only the first or second explicit argument does not alter any represented write operand.

With the incoming word held fixed, changing the third argument changes the value supplied to global destination B. It does not change the represented operands of the final write. Aliasing may nevertheless change the resulting observable storage state.

## Uncertainties

The following remain **documented_only**:

- Exact calling convention and mechanism for supplying the implicit incoming word.
- Independent confirmation of unsigned-word and short-integer widths.
- Addresses, widths, types, alignment, and alias relationships of the global destinations.
- Exact machine instructions and hardware-visible write ordering.
- Byte order and language-level signed-narrowing behavior.
- Memory mapping and access permissions at calculated addresses.
- Fault behavior, exception handling, and recovery.
- Instruction semantics and control flow beyond the undecodable boundary.
- Whether the represented entry point or signature is complete and correct.

A plausible interpretation is that selected high bits choose a 256-byte-spaced destination while low bits provide a halfword payload. That describes the arithmetic, but it does not establish the operation’s application-level intent.

## Evidence

The supplied static representation directly supports three ordered assignments: zero to one global destination, the third 4-byte argument to another, and a narrowed incoming-register value to a calculated address. It also supports the absence of visible uses of the first two arguments in that prefix.

The supplied warnings limit confidence in global-object boundaries and terminate reliable control-flow interpretation after the final visible write.

The numeric bounds and examples above are arithmetic deductions under the stated unsigned 32-bit and ordinary 16-bit narrowing assumptions. They are not observations from execution.

No runtime tests, memory traces, caller inspection, dependency inspection, or independent machine-level validation have been performed.
