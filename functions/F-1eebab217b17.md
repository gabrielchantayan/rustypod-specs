<!-- Trusted provenance: F-1eebab217b17; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1eebab217b17

## Purpose

The available static evidence shows a routine that interleaves 56 indirect calls with nine explicit memory writes. Its visible work includes forwarding three supplied words, deriving values from two additional entry-state words, writing scaled dependency results to calculated addresses, and performing byte-wide writes through byte-derived addresses.

A higher-level purpose—such as initialization, serialization, hardware configuration, or table construction—is not established. The evidence does not support naming the touched memory as a particular structure or device.

The recovered path ends at an invalid-instruction boundary. It does not establish a normal return, a return value, or successful completion.

This contract distinguishes:

- **Established static evidence:** operations and ordering present in the supplied representation.
- **Inferred interpretation:** a 32-bit machine-word model used to explain scaling and truncation.
- **Documented-only unknowns:** dependency behavior, execution validity, ownership, and behavior beyond the invalid-instruction boundary.

No runtime tests have been performed.

## Interface

### Explicit inputs

The visible interface has three explicit inputs, named here **P**, **Q**, and **R**. Each is a 32-bit word with no established signedness or semantic type.

All three are forwarded, in their original order, to the first indirect dependency. The visible body does not otherwise inspect or explicitly forward them.

### Additional entry state

The routine also consumes:

- **S:** an additional 32-bit word interpreted as unsigned for right shifts.
- **T:** an additional 32-bit word used as the payload of the first explicit memory write.
- **L:** entry control state used to calculate the first indirect call target.

These are not established as ordinary explicit parameters. A caller that supplies only P, Q, and R has not, on the available evidence, supplied all state needed to characterize execution.

Define the following descriptive quantities:

- **H₃:** S shifted right logically by three bits.
- **H₆:** S shifted right logically by six bits.
- **B:** H₃ multiplied by 524,288.
- **J:** the byte-read address B + 1.

H₃ lies between 0 and 536,870,911. H₆ lies between 0 and 67,108,863. B can exceed the range of a 32-bit word before truncation.

### Results and termination

No normal output value is established. The visible effects are dependency invocations, memory accesses, and arrival at an invalid-instruction boundary if all preceding operations complete.

The recovered interface presents no normal return value. Because the control-flow recovery is explicitly incomplete, that presentation is not proof of the original callable interface or of a defined non-returning API.

### Preconditions for following the visible path

Continuing through the entire visible sequence requires:

- Every reached indirect target to be executable under the actual platform’s calling rules.
- Every dependency to return sufficiently for the following operation to occur.
- All calculated read and write addresses to be valid for their access widths.
- Required alignment, access permissions, and memory-mapped behavior to permit each operation.
- The recovered instruction interpretation to correspond to executable behavior.

No checks enforcing these conditions are visible. The routine does not visibly validate indices, addresses, lengths, or dependency results.

## Behavior

### Dependency labels and arithmetic notation

The opaque dependency IDs **F-U001** through **F-U056** identify successive indirect call occurrences. They are local descriptive IDs, not verified identities of distinct functions. Different occurrences may resolve to the same implementation.

For describing machine-word effects, let **W(x)** mean the low 32 bits of x, and **B₈(x)** mean the low eight bits. Expressions below describe observable values and addresses; they do not prescribe an implementation.

Multiplication of recovered signed values can exceed signed 32-bit range. The supplied representation does not establish portable signed-overflow semantics. Where wrapping matters, the examples and formulas use an explicitly conditional 32-bit machine interpretation.

### Initial forwarding and first word write

The first call, F-U001, receives P, Q, R, and H₃, in that order. Its target is derived from L with an offset of 4,094. No return value from this call is explicitly consumed.

F-U002 through F-U005 follow in order. F-U005 supplies a 32-bit result, called **A** here.

The routine then writes the 32-bit word T at the address obtained by scaling A by four and adding 20. Under the wrapping interpretation, the address is W(4A + 20).

This is an address calculation from the numeric result itself. No separately loaded base pointer is visible.

### Further word writes

The next stages establish four more word writes:

| Dependency interval | Result used | Explicit write after the interval |
|---|---|---|
| F-U006 through F-U010 | F-U010 supplies **C** | Write W(4C) at address W(16C + 20) |
| F-U011 through F-U015 | F-U015 supplies **D** | Write the previously calculated W(4C) at address W(4D + 20) |
| F-U016 through F-U020 | F-U020 supplies **E** | Write W(4E) at address W(16E + 20) |
| F-U024 through F-U027 | F-U027 supplies **G** | Write H₃ at address W(4G + 20) |

The explicit arguments surrounding these stages are:

- F-U006 receives W(4A).
- F-U011 receives W(4C).
- F-U016 receives W(4D).
- F-U021 receives W(4E) and W(16E), in that order.
- F-U022 supplies a 32-bit result that F-U023 receives as its first argument; F-U023 receives H₆ as its second.
- F-U028 receives W(4G).

The third word write reuses the scaled result from F-U010. It does not use four times F-U015’s result as its payload.

No return values from the other calls in these intervals are explicitly consumed. Their side effects are nevertheless unknown and cannot be discarded from the ordering contract.

### First byte-write pair

F-U028 through F-U032 execute in order. F-U032 supplies an unsigned 32-bit result called **U**.

After F-U032 returns:

1. The routine reads one unsigned byte at J.
2. It adds 17 to that byte’s numeric value to obtain a destination address.
3. It writes B₈(4U) to that destination.

Because the address selector is an unsigned byte, the destination lies between 17 and 272 inclusive. It is not the address J plus 17.

F-U033 then receives W(4U).

F-U034 supplies a 32-bit result that is forwarded to F-U035. F-U036 through F-U039 follow. F-U039 supplies a 32-bit result called **V**.

After F-U039 returns, the routine reads a byte at J again. It uses this newly read byte plus 17 as the destination address, then writes:

B₈((U masked to its low 30 bits) shifted right by 20).

The stored byte is equivalently bits 20 through 27 of U. Although the intermediate shifted value may contain ten bits, only eight are stored.

F-U040 then receives W(4V). V determines this call argument, not the preceding byte-write payload or selector address.

### Third byte write

F-U041 through F-U044 execute in order. F-U044 supplies an unsigned 32-bit result called **X**.

The routine then:

1. Reads an unsigned byte at J for a third time.
2. Writes B₈(4X) at the address given by that byte plus 17.
3. Calls F-U045 with the byte just read, zero-extended to 32 bits.

F-U045 therefore receives the selector byte, not the written byte, the destination address, or W(4X).

The three reads at J are separate observable accesses. Their results need not agree. Earlier writes can alias J, and dependencies can change memory between reads.

### Final byte write and trailing calls

F-U046 supplies a 32-bit result forwarded to F-U047. F-U048 through F-U051 follow. F-U051 supplies a 32-bit result called **Y**.

The fourth byte write uses a different selector-read address:

- Read one unsigned byte at W(8X + 1).
- Add 17 to that byte to obtain the write address.
- Store B₈((X masked to its low 30 bits) shifted right by 20).

This payload is bits 20 through 27 of X. It does not reuse U.

F-U052 then receives W(4Y). F-U053 through F-U056 execute in order. No return value from these final four calls is explicitly consumed.

The recovered path then encounters an invalid-instruction boundary. No subsequent behavior is established.

### Argument and access ordering

Calls for which this contract lists no explicit arguments have no arguments explicitly represented at that call site. This does not establish that the target ignores register contents, stack contents, memory, or other machine state.

Each described memory access occurs between the stated dependency calls. No explicit write occurs before F-U005 returns. No explicit selector-byte read occurs before F-U032 returns.

There is no visible conditional branch or retry in the recovered path. A dependency that does not return, an invalid memory access, or an invalid call target can prevent all later events.

## Side effects

### Explicit memory operations

The visible body contains:

- Five 32-bit writes.
- Four byte reads.
- Four byte writes.

This count excludes all accesses performed by dependencies and instruction fetching.

The word-write addresses may overlap one another. Byte-write addresses may overlap each other, selector-read addresses, word-write locations, or other meaningful memory. The body neither excludes nor detects such aliasing.

The expressions provide address arithmetic, not evidence of a structure layout. Offsets of 20 and 17 do not establish headers, fields, allocation boundaries, or valid object sizes.

### Shared state and dependency effects

Dependencies occur between almost every explicit memory operation. They may modify shared state, perform I/O, allocate or release resources, change memory mappings, or fail to return. None of those behaviors is established or excluded.

A return value not explicitly consumed is not evidence that the corresponding call is without side effects.

### Ownership and lifetime

No allocation, release, ownership transfer, or lifetime extension is identifiable in the visible body. Calculated addresses and dependency results must not be assumed to designate newly allocated or owned storage.

No preservation guarantee for globals or externally visible state is established.

### Errors and partial execution

There is no visible error result, error-status check, rollback, cleanup path, or recovery branch.

If execution stops partway through, earlier memory writes and dependency side effects may remain observable. The terminal invalid-instruction marker is a recovery boundary, not an established application error protocol.

## Dependencies

The 56 opaque IDs are occurrence labels only.

Most targets are represented using a site-associated value plus 4,094. F-U021, F-U033, and F-U045 use an offset of 4,080 instead. F-U001 depends on entry control state rather than solely on a site-associated constant.

The static evidence does not establish:

- That all targets resolve to separate functions.
- The actual calling convention.
- Full dependency input or output signatures.
- Whether a no-explicit-argument call reads residual machine state.
- Whether a dependency preserves values held across calls.
- Dependency ownership, memory, or failure behavior.

The behavior above follows the recovered representation’s use of retained values. It does not independently prove that actual machine execution preserves those values across the indirect calls.

## Acceptance examples

These are conditional acceptance criteria, not executed tests. Unless stated otherwise, they assume valid accesses, returning dependencies, and 32-bit wrapping address arithmetic.

### Explicit input forwarding

If P, Q, and R are arbitrary words and S is 64, F-U001 must receive P, Q, R, and 8 in that order. Later, F-U023 must receive the result from F-U022 followed by 1.

### Word-write scaling and value reuse

If F-U005 returns 3 and T is 0x12345678, the first explicit write stores that 32-bit value at address 32.

If F-U010 returns 5, the next word write stores 20 at address 100. If F-U015 subsequently returns 7, the following word write stores the same value 20 at address 48.

If F-U020 returns 2, its following word write stores 8 at address 52, and F-U021 receives 8 and 32.

### Byte truncation

If F-U032 returns 0x12345678 and the first selector read yields 9, the first byte write targets address 26 and stores 0xE0. F-U033 receives 0x48D159E0.

If the second selector read also yields 9, the second byte write targets address 26 and stores 0x23.

### Selector rereads

If the first two reads at J yield 9 and 12 respectively, the first two byte writes target addresses 26 and 29. Reusing the first selector for the second write would not match the visible behavior.

This criterion also applies when the change results from aliasing rather than from a dependency.

### Third and fourth byte writes

If F-U044 returns 0x00100001 and the third selector read yields 200:

- The third byte write stores 4 at address 217.
- F-U045 receives 200.
- The final selector-read address is 0x00800009.

If the selector read at that final address yields 7, the fourth byte write stores 1 at address 24.

### Overflow and interruption

If A is 0x40000000, the wrapping interpretation places the first word write at address 20, rather than at an unbounded mathematical address.

If F-U033 does not return, the first byte write may already have occurred, but none of the later explicit memory operations occurs along the visible path.

## Uncertainties

The following remain **documented_only**:

- **Instruction validity:** the supplied representation warns about bad instruction data and truncated control flow.
- **Callable interface:** the three-word signature may omit required machine state or misrepresent the original interface.
- **Arithmetic semantics:** recovered signed multiplications and shifts do not establish portable overflow behavior.
- **Address interpretation:** pointer width, instruction-set tagging, alignment, and access permissions are not established.
- **Byte order:** five word writes are visible, but their byte-level memory layout is unknown.
- **Memory model:** the evidence does not establish atomicity, concurrency guarantees, volatility, or device semantics.
- **Dependency contracts:** signatures, clobbers, preservation guarantees, ownership, and failure behavior are unavailable.
- **Termination:** the invalid-instruction boundary may reflect actual execution failure, incorrect decoding, or incomplete recovery.
- **Intent:** scaling and bit extraction are visible; their application meaning is not.

These unknowns prevent treating the routine as a validated executable API.

## Evidence

This contract is based solely on the supplied static representation of F-1eebab217b17, including its recovery warnings.

Direct evidence supports the ordered indirect calls, explicit argument forwarding, five word writes, four selector-byte reads, four byte writes, logical right shifts of S, scaling operations, byte truncation, and terminal invalid-instruction boundary.

The 32-bit wrapping model is an explicitly conditional interpretation for explaining arithmetic and acceptance examples. It is not runtime confirmation.

No dependency bodies, runtime traces, platform memory map, executable validation, or tests were supplied. No runtime tests have been performed.
