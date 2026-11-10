<!-- Trusted provenance: F-584c14ac7dda; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-584c14ac7dda

## Purpose

F-584c14ac7dda performs a fixed visible sequence of indirect calls and computed-address memory accesses. Call results determine stored values, memory addresses, and arguments to subsequent dependencies.

The supplied static representation establishes five direct 32-bit stores, one direct unsigned 32-bit read, two direct unsigned byte reads, two direct byte stores, and 42 indirect calls. It does not establish a domain-level purpose such as construction, serialization, allocation, device configuration, or cleanup.

No normal return is represented. After the final indirect call, the representation terminates at a boundary marked as invalid instruction data. This is a limit of the available static evidence, not proof of a particular runtime trap, halt, or successful termination.

All behavior below is conditional on execution reaching the relevant operation. An earlier dependency that does not return, or an earlier memory access that cannot complete, can prevent the remainder.

## Interface

### Explicit and implicit inputs

The represented interface declares no parameters and no return value. This does not establish independence from entry state.

Visible implicit inputs include:

- An incoming link-register value used to calculate the first indirect-call target.
- An unexplained incoming register value used as the first 32-bit store payload.
- Nine indirect-call results used in subsequent operations.
- One unsigned 32-bit memory value.
- Two independently obtained unsigned byte values.
- Any additional machine state consumed by dependencies but not exposed in their represented argument lists.

There is no represented success indicator, error result, output buffer parameter, or explicit ownership-bearing parameter.

### Widths and notation

The representation distinguishes 8-bit bytes, signed and unsigned 32-bit integers, and opaque 32-bit values. Its expressions also treat computed values as addresses. The actual execution environment and address width are not independently verified.

For the conditional 32-bit interpretation used in this contract:

- \(W(x)\) means the low 32 bits of \(x\).
- \(Q(x)=W(4x)\).
- Computed addresses are interpreted using their resulting 32-bit bit patterns.
- A byte store retains the low eight bits of its payload.
- Right shifts of unsigned values are logical.

Some source expressions use signed multiplication or signed left shifts. Their overflow and out-of-range behavior is not independently established. Formulas involving truncation or wrapping therefore describe the conditional machine-bit-vector interpretation, not a claim that every rendered signed expression has defined language-level behavior.

### Value labels

These labels describe data flow; they are not implementation identifiers.

| Label | Meaning |
|---|---|
| Initial payload | Incoming opaque 32-bit value used by the first store |
| Index A | Result of F-U03 |
| Index B | Result of F-U08 |
| Saved word | Unsigned 32-bit value read immediately after the second store |
| Index C | Result of F-U13 |
| Index D | Result of F-U18 |
| Index E | Result of F-U23 |
| Packed word | Unsigned 32-bit result of F-U28 |
| Forwarded token | Opaque 32-bit result of F-U30 |
| Index G | Result of F-U35 |
| Index H | Result of F-U40 |
| Second selector | Unsigned byte obtained by the second selector read |

F-U01 through F-U42 are opaque, document-local dependency call-site IDs. Distinct IDs do not establish distinct underlying functions.

## Behavior

### Observable sequence

The represented path has no conditional branch. It interleaves dependency calls with the direct memory operations below.

“No exposed arguments” means that the representation shows none. It does not establish that the dependency has no implicit inputs or that its actual calling convention takes no arguments.

| Order | Behavior |
|---|---|
| F-U01, F-U02, F-U03 | Called in order with no exposed arguments. F-U03 supplies Index A. |
| First 32-bit store | Write the Initial payload at \(W(Q(\text{Index A})+20)\). |
| F-U04 | Called with \(Q(\text{Index A})\). |
| F-U05, F-U06, F-U07, F-U08 | Called in order with no exposed arguments. F-U08 supplies Index B. |
| Second 32-bit store | Write \(Q(\text{Index B})\) at \(W(16\text{Index B}+20)\). |
| 32-bit read | Read the Saved word from \(W(Q(\text{Index B})+20)\), after the preceding store and without an intervening dependency call. |
| F-U09 | Called with \(Q(\text{Index B})\). |
| F-U10, F-U11, F-U12, F-U13 | Called in order with no exposed arguments. F-U13 supplies Index C. |
| Third 32-bit store | Write the Saved word logically shifted right by 12 at \(W(Q(\text{Index C})+20)\). |
| F-U14 | Called with \(Q(\text{Index C})\). |
| F-U15, F-U16, F-U17, F-U18 | Called in order with no exposed arguments. F-U18 supplies Index D. |
| Fourth 32-bit store | Write \(Q(\text{Index D})\) at \(W(16\text{Index D}+20)\). |
| F-U19 | Called with three arguments, in order: \((Q(\text{Index C})\mathbin{\&}63)\ll14\), \(Q(\text{Index D})\), and \(W(\text{Index C}\ll30)\). |
| F-U20, F-U21, F-U22, F-U23 | Called in order with no exposed arguments. F-U23 supplies Index E. |
| Fifth 32-bit store | Write \(Q(\text{Index D})\) at \(W(Q(\text{Index E})+20)\). |
| F-U24 | Called with \(Q(\text{Index E})\). |
| F-U25, F-U26, F-U27, F-U28 | Called in order with no exposed arguments. F-U28 supplies the Packed word. |
| First selector read and byte store | Read an unsigned byte from \(W(\text{Index C}\times2^{28}+1)\). Write the low eight bits of \(Q(\text{Packed word})\) at the address obtained by adding 17 to that byte’s numeric value. |
| F-U29 | Called with three arguments, in order: \(Q(\text{Packed word})\), the same value again, and \(W(\text{Packed word}\ll4)\). |
| F-U30 | Called with no exposed arguments; supplies the Forwarded token. |
| F-U31 | Called with the Forwarded token. |
| F-U32, F-U33, F-U34, F-U35 | Called in order with no exposed arguments. F-U35 supplies Index G. |
| Second selector read and byte store | Read another unsigned byte from \(W(\text{Index C}\times2^{28}+1)\), obtaining the Second selector. Write the low eight bits of \((\text{Packed word}\mathbin{\&}0x3fffffff)\gg20\) at address \(\text{Second selector}+17\). |
| F-U36 | Called with \(W(\text{Index G}\ll2)\). |
| F-U37, F-U38, F-U39, F-U40 | Called in order with no exposed arguments. F-U40 supplies Index H. |
| F-U41 | Called with two arguments, in order: \(W(\text{Index H}\ll2)\) and the zero-extended Second selector. |
| F-U42 | Called with no exposed arguments. |
| Representation boundary | The available representation ends at an invalid-instruction marker. No normal return is shown. |

### Address layout and aliasing

The direct accesses use computed absolute addresses. No separately supplied base pointer is visible.

Two word-address families are materially different:

- \(W(4x+20)\).
- \(W(16x+20)\).

In particular, the second store and the immediately following read use different scale factors applied to Index B. They must not be treated as an unconditional write followed by a readback from the same location.

Under the conditional 32-bit arithmetic model, these word addresses are multiples of four. That arithmetic property does not establish mapping, access permission, memory type, or the runtime’s alignment requirements.

No non-aliasing guarantee is present. Computed addresses can coincide, and byte accesses can overlap locations used by word accesses. Byte order is not established, so the effects of a byte store on a later overlapping word interpretation cannot be specified numerically without an additional memory-layout assumption.

The selector byte is used directly as a number to form a destination address. There is no second pointer dereference through that byte. Because its unsigned range is 0 through 255, each byte-store destination is between addresses 17 and 272 inclusive.

The selector reads are separate observable accesses to the same address expression. F-U29 through F-U35 execute between them. Neither equal read addresses nor reuse of Index C establishes equal selector values.

### Retained values and bit selection

The represented data flow retains the Saved word across F-U09 through F-U13, \(Q(\text{Index D})\) across F-U19 through F-U23, and the Packed word across F-U29 through F-U35. It also retains the Second selector for F-U41 across F-U36 through F-U40.

These are static value dependencies, not general guarantees that dependencies preserve registers, globals, or memory.

Under the 32-bit bit-vector interpretation:

- The third store’s payload lies between zero and \(2^{20}-1\).
- F-U19’s first argument selects the low six bits of \(Q(\text{Index C})\) and shifts them left by 14. Since \(Q(\text{Index C})\) has two low zero bits, this argument is a multiple of \(2^{16}\).
- F-U19’s third argument places the original low two bits of Index C in bits 30 and 31.
- The selector-read address depends only on the low four bits of Index C. Its possible forms are \(k\times2^{28}+1\), with \(k\) from 0 through 15.
- The first byte-store payload is the low byte of four times the Packed word; its two low bits are zero.
- The second byte-store payload is original Packed-word bits 20 through 27. The mask and shift can produce ten bits, but the byte store discards the upper two.

## Side effects

The visible body directly performs:

- Five 32-bit writes.
- One unsigned 32-bit read.
- Two unsigned byte reads.
- Two byte writes.
- Forty-two indirect calls.

These counts exclude memory activity or other effects inside dependencies. A dependency whose return value is unused may still have arbitrary observable effects.

The stated ordering is the order exposed by the static representation. It does not establish physical transaction counts, atomicity, volatility, interprocessor ordering, synchronization, or memory barriers.

No bounds checking, pointer validation, access-permission checking, explicit exception handling, rollback, or status reporting is visible. If execution stops after a completed write, no visible mechanism reverses that write. External agents or dependencies could subsequently modify the same memory; their behavior is unknown.

No direct allocation or release operation is identified. Passing the Forwarded token to F-U31 does not establish whether it denotes memory or another resource, whether ownership transfers, or whether either dependency allocates or releases anything. The same restriction applies to every computed argument and memory address.

## Dependencies

F-U01’s target is formed using the incoming link-register value plus 4094.

The remaining targets are formed from fixed anchors in the supplied representation:

- F-U29 uses an anchor adjustment of 4076.
- Every other fixed-anchor call uses an adjustment of 4094.

These expressions do not establish valid executable destinations. Actual dependency identities, instruction-set state, target alignment handling, executable-memory requirements, and complete signatures remain unresolved.

**Nine indirect-call results are visibly used:** F-U03, F-U08, F-U13, F-U18, F-U23, F-U28, F-U30, F-U35, and F-U40. Their roles are specified in the interface and ordered behavior above. Results from the remaining call sites have no explicit downstream use in the visible body.

No dependency implementation is supplied. Therefore:

- Exposed arguments specify visible data flow, not complete dependency interfaces.
- Additional implicit arguments or state dependencies remain possible.
- No dependency is established as pure or globally preserving.
- Dependencies may read or modify directly accessed memory.
- Allocation, release, ownership, error handling, and non-returning behavior remain unknown.
- Different call-site IDs may resolve to the same implementation.
- A dependency can prevent later visible operations by failing, transferring control elsewhere, or not returning.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, they assume valid accesses, ordinary stable memory, returning dependencies, and the 32-bit arithmetic interpretation defined above.

### Different scale factors remain distinct

If Index B is 2, the second store writes the 32-bit value 8 at address 52. The following read obtains the Saved word from address 28, not address 52.

If address 28 contains `0x89abcdef` at that read, the third store writes `0x00089abc`. The write at address 52 does not itself determine this payload under ordinary non-overlapping memory behavior.

### Exact store/read aliasing

If Index B is zero, both the second store and the immediately following read use address 20. The store writes zero.

With ordinary memory and no external modification between those accesses, the Saved word is zero. The third store therefore writes zero if reached. No dependency-preservation assumption is needed between this particular store and read because no dependency executes between them.

### Index C controls multiple independent fields

If Index C is 3:

- \(Q(\text{Index C})\) is 12.
- F-U19’s first argument is `0x00030000`.
- F-U19’s third argument is `0xc0000000`.
- Both selector reads use address `0x30000001`.

These calculations do not establish that the selector-read address is mapped or readable.

### Byte destinations can differ

If the Packed word is `0x12345678`, the first byte-store payload is `0xe0`, and the second is `0x23`.

If the first selector read returns 5 and the second returns 9:

- The first byte store targets address 22.
- The second targets address 26.
- F-U41 receives 9 as its second argument, even if the selector memory changes again before that call.

If both selector reads instead return 5, both stores target address 22. At the second store, ordinary writable memory at that address receives `0x23`, regardless of its earlier value. This does not prohibit intervening dependency effects or establish the value after later dependencies execute.

Selector values 0 and 255 produce destination addresses 17 and 272 respectively; no signed-byte interpretation is applied.

### Partial progress is observable

If F-U14 does not return, the third 32-bit store has already occurred. The fourth store and every subsequent operation are not reached through the represented continuation. No rollback is shown.

### No successful-return requirement is supported

If F-U42 returns, the next represented event is the invalid-instruction boundary. A claim that F-584c14ac7dda then returns normally, reports success, or performs a particular runtime trap is unsupported.

## Uncertainties

**Established static evidence:** the represented operation order, nine used call results, explicit argument relationships, memory-access widths, arithmetic expressions, and absence of a visible normal return.

**Inferred intent:** repeated scaling, masking, and shifting suggest indexed memory access and bit-field manipulation. No domain-specific interpretation is established.

The following remain **documented_only**:

- Whether the representation faithfully identifies executable instructions throughout; it includes invalid-instruction warnings.
- Whether the apparent integer and address widths match the actual runtime.
- Whether signed multiplication and signed left shifts should be interpreted as wrapping machine arithmetic.
- Whether the actual calling convention supports every retained-value relationship as represented.
- The identities, complete signatures, implicit inputs, and error behavior of dependencies.
- Whether dependencies return, preserve globals, modify relevant memory, allocate resources, or release resources.
- Memory mapping, permissions, byte order, memory type, external mutation, and transaction semantics.
- The runtime consequence of reaching the terminal representation boundary.

Execution of any particular prefix requires valid call targets and successful memory accesses for that prefix. The supplied evidence does not establish these preconditions universally.

## Evidence

This contract is derived from the supplied static representation and its warning annotations. The prior document and reviewer feedback are not treated as independent behavioral evidence.

The dependency count, nine-result count, access counts, and data-flow relationships are static observations. No dependency bodies, calling-convention specification, memory map, or runtime observations were supplied.

No runtime tests have been performed. No snapshot hash is asserted here.
