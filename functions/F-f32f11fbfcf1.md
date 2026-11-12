<!-- Trusted provenance: F-f32f11fbfcf1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f32f11fbfcf1

## Purpose

The supplied static representation describes a stateful dispatch sequence: it writes an incoming 32-bit value to memory, invokes a long sequence of unresolved callees, uses selected return values to calculate additional memory locations, transfers and extracts values, performs two byte writes, and reaches an invalid-instruction termination marker.

No application-level purpose is established. In particular, the evidence does not establish that the computed locations are ordinary objects, arrays, device registers, or allocated buffers.

This contract distinguishes:

- **Established static evidence:** operations and ordering explicitly represented in the supplied material.
- **Inferred intent:** interpretations consistent with that representation but not independently established.
- **documented_only:** unresolved behavior, including dependency semantics, arithmetic overflow, and execution beyond the invalid-instruction boundary.

The representation includes a warning about bad instruction data. Consequently, this is a contract for the represented behavior, not a claim that every represented operation corresponds to a valid executable instruction.

## Interface

### Entry inputs

No conventional parameters are declared. Nevertheless, the represented behavior depends on incoming machine state:

| Input | Represented use |
|---|---|
| Initial destination base | Receives a four-byte store at offset `0x14`. |
| Initial 32-bit payload | Value written by that initial store. |
| Incoming dispatch base | Determines the first indirect call target by adding `0xFFE`. |
| Incoming shift operand | Later supplies a value shifted left by 26 bits as an explicit call argument. |
| Memory and execution environment | Supplies all addressed data and callable targets. |

These are implicit inputs. The absence of declared parameters must not be interpreted as independence from caller state.

The initial destination base and incoming dispatch base are used before any represented return value replaces their roles. The incoming shift operand is referenced much later. Whether the underlying calling convention preserves its physical storage across intervening calls is not established.

### Output

There is no established normal return or returned result. The visible sequence ends at an invalid-instruction termination marker after its final indirect call.

Observable outputs consist of:

- Direct memory writes.
- Explicit arguments delivered to unresolved callees.
- Effects of those callees, which are unknown.
- Any fault or termination caused by the represented execution.

### Widths and notation

The representation distinguishes:

- Four-byte values and stores, treated here as 32-bit quantities.
- One-byte unsigned loads.
- One-byte stores that retain the low eight bits of their input.
- Unsigned right shifts, which insert zero bits.

For describing the sequence, the following mathematical labels are used:

- **A** through **J**: selected intermediate 32-bit call results.
- **K**: a later result explicitly used as an unsigned 32-bit value.
- **L** and **M**: the final two selected 32-bit results.
- **P**: the value loaded during the fourth repeated transfer group.
- **Q**: the 32-bit result forwarded directly between two later calls.
- **U**: four times H, represented as an unsigned 32-bit quantity.
- **b₁** and **b₂**: two separately loaded unsigned bytes.

These are specification labels, not interface parameter names.

Arithmetic containing signed multiplication or signed left shifts has unresolved overflow semantics. Equations below preserve the represented operations; they do not silently impose wrapping behavior. Where a 32-bit machine interpretation is needed, examples state that condition explicitly.

### Preconditions

For execution to reach the final visible boundary:

1. Every indirect target reached must be callable in the relevant execution mode.
2. Every dependency must return when execution is expected to continue.
3. Every represented memory access must be valid for its width and access type.
4. Any required alignment and architectural restrictions must be satisfied.
5. Arithmetic must have an established interpretation for the actual input values.

The function contains no visible checks enforcing these conditions.

## Behavior

### Dependency labels and ordering

The unresolved call occurrences are assigned opaque labels **F-d001** through **F-d069**, in execution order. These labels identify occurrences for this contract; they do not establish that the runtime targets are distinct functions.

An occurrence described as having “no explicit arguments” has no argument expressions in the supplied representation. This does not prove an empty machine-level input interface.

Unless a dependency fails to return or execution faults, the represented control flow is straight-line: no visible conditional branch, loop, retry, or early-return path appears.

### Initial write and first transfer group

Before any call, the function writes the initial 32-bit payload to the initial destination base plus `0x14`.

It then invokes F-d001 through F-d005 in that order, with no explicit arguments. F-d005 supplies A.

Next:

1. Compute four times A.
2. Store that value as four bytes at address sixteen times A plus `0x14`.
3. Read a 32-bit unsigned value at address four times A plus `0x14` for a right shift by 12.
4. Read from that same address again for a left shift by two.
5. Invoke F-d006 with three explicit arguments:
   - The first read shifted right by 12.
   - Four times A.
   - The second read shifted left by two.

Both reads occur after the store and before F-d006. Their relative evaluation order is not established by the supplied representation. They must not be collapsed into a single snapshot: memory may change between reads, and the addressed region may have read side effects.

F-d007 through F-d010 follow, with no explicit arguments. F-d010 supplies B.

The function then writes four times A as four bytes at address four times B plus `0x14`, and invokes F-d011 with four times B.

### Second transfer group

F-d012 through F-d015 execute with no explicit arguments; F-d015 supplies C.

The function:

1. Computes four times C.
2. Stores it at address sixteen times C plus `0x14`.
3. Invokes F-d016 with:
   - Four times C.
   - Four times C again.
   - The incoming shift operand shifted left by 26.

F-d017 through F-d020 execute with no explicit arguments; F-d020 supplies D.

The function stores four times C at address four times D plus `0x14`, then invokes F-d021 with four times D.

The third argument of F-d016 uses the represented incoming shift operand, not a newly selected result from this group. Its preservation across earlier dependencies is an unresolved machine-level issue.

### Third transfer group

F-d022 through F-d025 execute with no explicit arguments; F-d025 supplies E.

The function stores four times E at address sixteen times E plus `0x14`, then invokes F-d026 with four times E.

F-d027 through F-d030 execute with no explicit arguments; F-d030 supplies F.

The function stores four times E at address four times F plus `0x14`, then invokes F-d031 with four times F.

### Fourth transfer group and captured value

F-d032 through F-d035 execute with no explicit arguments; F-d035 supplies G.

The function then:

1. Stores four times G at address sixteen times G plus `0x14`.
2. Reads a 32-bit unsigned value, P, from address four times G plus `0x14`.
3. Invokes F-d036 with four times G.

F-d037 through F-d040 execute with no explicit arguments; F-d040 supplies H.

Define U as four times H in the represented unsigned 32-bit destination.

The function stores P shifted right by 12 at address U plus `0x14`, then invokes F-d041 with U.

P is captured before F-d036 through F-d040. The later store uses that captured value, not a new read of its original address.

### Fifth transfer group

F-d042 through F-d045 execute with no explicit arguments; F-d045 supplies I.

The function stores four times I at address sixteen times I plus `0x14`, then invokes F-d046 with:

1. The low six bits of U, shifted left by 14.
2. Four times I.
3. H shifted left by 30.

F-d047 through F-d050 execute with no explicit arguments; F-d050 supplies J.

The function stores four times I at address four times J plus `0x14`, then invokes F-d051 with four times J.

The first explicit argument of F-d046 is unsigned and lies between zero and `0xFC000`, inclusive. Under a 32-bit wrapping interpretation of U as four times H, U’s low two bits are zero, further restricting this argument to multiples of `0x10000`. The shift of H remains subject to the signed-arithmetic uncertainty.

### First byte write and value forwarding

F-d052 through F-d055 execute with no explicit arguments; F-d055 supplies K.

The function then:

1. Reads unsigned byte b₁ at address H multiplied by `0x10000000`, plus one.
2. Writes the low eight bits of K shifted left by two to address b₁ plus `0x11`.
3. Invokes F-d056 with:
   - K shifted left by two.
   - K shifted left by two again.
   - K shifted left by four.

Because b₁ is an unsigned byte, the direct byte-write destination is numerically between `0x11` and `0x110`, inclusive. The byte is used directly as an address component; the representation does not show a larger pointer being loaded from that location.

The unsigned shifts of K retain a 32-bit result. The first byte write therefore retains K’s low six bits in destination bits two through seven, with destination bits zero and one clear.

F-d057 executes with no explicit arguments and supplies Q. F-d058 is then invoked with Q as its sole explicit argument.

No allocation, ownership transfer, or release obligation can be inferred from this forwarding.

### Second byte write and final calls

F-d059 through F-d062 execute with no explicit arguments; F-d062 supplies L.

The function then:

1. Reads unsigned byte b₂ from the same represented address expression used for b₁: H multiplied by `0x10000000`, plus one.
2. Writes the low eight bits of `(K masked with 0x3FFFFFFF) shifted right by 20` to address b₂ plus `0x11`.
3. Invokes F-d063 with L shifted left by two.

The second byte load is a fresh access after multiple intervening calls. Equality of b₁ and b₂ is not guaranteed.

The second byte store ultimately writes bits 20 through 27 of K. Although the masked shift produces up to ten significant bits, the one-byte store discards its upper two bits.

F-d064 through F-d067 execute with no explicit arguments; F-d067 supplies M.

F-d068 is invoked with:

1. M shifted left by two.
2. b₂, zero-extended from the earlier byte read.

F-d069 follows with no explicit arguments. The visible representation then reaches the invalid-instruction termination marker. It establishes neither normal return nor subsequent recovery behavior.

## Side effects

### Direct memory accesses

On the complete represented path, excluding dependency effects:

- There are **11 four-byte writes**: the initial write and ten writes across the five transfer groups.
- There are **two one-byte writes**.
- There are **three four-byte reads**: two before F-d006 and one capturing P.
- There are **two one-byte reads**, producing b₁ and b₂.

These are access occurrences, not counts of distinct locations. Aliasing may make several accesses refer to the same storage.

The first write always precedes the first dependency call. Each later write occurs at the position described in the sequence; the function provides no visible rollback if a subsequent access or call fails.

### Aliasing and layout

The offset `0x14` consistently identifies a four-byte access relative to several computed bases. This is insufficient to establish a structure definition or a buffer extent.

The first write in a transfer group uses a base scaled by sixteen. Its paired later write uses a different selected result scaled by four. These address formulas are not interchangeable.

Aliasing can affect behavior. For example, the write based on sixteen times A may overlap a later read based on four times A. No disjointness guarantee is visible.

Byte order is not specified. Overlap between byte and four-byte accesses therefore requires architecture-specific interpretation.

### Ownership and external state

No direct allocation or release operation is identifiable. The function does not establish ownership of any addressed memory.

Dependencies may mutate memory, global state, devices, or execution context. Their effects and preservation guarantees are unknown. The contract does not assume they preserve globals, retain or release forwarded values, or leave the byte-read location unchanged.

## Dependencies

The explicit dependency interface is:

| Occurrences | Explicit arguments | Selected result |
|---|---|---|
| F-d001–F-d005 | None represented | A from F-d005 |
| F-d006 | Shifted read, four times A, shifted second read | Not used |
| F-d007–F-d010 | None represented | B from F-d010 |
| F-d011 | Four times B | Not used |
| F-d012–F-d015 | None represented | C from F-d015 |
| F-d016 | Four times C, four times C, incoming value shifted by 26 | Not used |
| F-d017–F-d020 | None represented | D from F-d020 |
| F-d021 | Four times D | Not used |
| F-d022–F-d025 | None represented | E from F-d025 |
| F-d026 | Four times E | Not used |
| F-d027–F-d030 | None represented | F from F-d030 |
| F-d031 | Four times F | Not used |
| F-d032–F-d035 | None represented | G from F-d035 |
| F-d036 | Four times G | Not used |
| F-d037–F-d040 | None represented | H from F-d040 |
| F-d041 | U | Not used |
| F-d042–F-d045 | None represented | I from F-d045 |
| F-d046 | Masked and shifted U, four times I, shifted H | Not used |
| F-d047–F-d050 | None represented | J from F-d050 |
| F-d051 | Four times J | Not used |
| F-d052–F-d055 | None represented | K from F-d055 |
| F-d056 | K shifted by two, the same value again, K shifted by four | Not used |
| F-d057 | None represented | Q |
| F-d058 | Q | Not used |
| F-d059–F-d062 | None represented | L from F-d062 |
| F-d063 | L shifted by two | Not used |
| F-d064–F-d067 | None represented | M from F-d067 |
| F-d068 | M shifted by two, b₂ | Not used |
| F-d069 | None represented | Not used |

The first target is derived from the incoming dispatch base plus `0xFFE`. Later targets are represented through embedded address-like bases, generally displaced by `0xFFE`; F-d056 uses displacement `0xFEC`.

Target identities, valid instruction modes, dependency contracts, and whether the apparent targets are executable are **documented_only**.

## Acceptance examples

These examples are conditional obligations of the represented sequence, not executed tests.

### Initial write precedes external execution

Given a valid initial destination base and payload `0x12345678`, the first direct effect is a four-byte store of that value at base plus `0x14`.

F-d001 occurs afterward. If F-d001 does not return, later direct accesses are not reached.

### Independent first-group reads

Suppose A is one, the computed accesses are valid, and the two reads at address `0x18` produce `0x12345000` and `0x00000003`, respectively.

The preceding store writes four to address `0x24`. F-d006 receives `0x12345`, four, and twelve.

This example requires no overflow interpretation. It demonstrates that the two reads need not produce the same value.

### Captured data survives as a represented value

Suppose G is one and the read capturing P produces `0xABCDEFFF`. Suppose H is two.

The function stores `0x000ABCDE` at address `0x1C` before F-d041. Changes to P’s original memory location during intervening calls do not constitute a new read by this function.

### Distinct byte destinations

Suppose K is `0x0AB00035`, the first byte read yields `0x20`, and the later byte read yields `0x40`.

The first byte store writes `0xD4` to address `0x31`. The second writes `0xAB` to address `0x51`.

F-d068 receives `0x40` as its second explicit argument, regardless of subsequent changes to the byte-read location.

### Failure leaves prior effects exposed

If a computed four-byte store faults after earlier groups have completed, the representation contains no compensating writes or cleanup path. Earlier completed writes and dependency effects are not specified to be undone.

### Final boundary

If all 69 calls return and all accesses complete, execution reaches the invalid-instruction marker. Successful ordinary return is not an acceptance requirement supported by this evidence.

## Uncertainties

- **documented_only — instruction validity:** Bad instruction data may compromise the represented control flow or expressions.
- **documented_only — arithmetic:** Signed overflow, negative left shifts, and out-of-range signed shifts cannot be assigned portable behavior from this material.
- **documented_only — addresses:** Address width, truncation rules, mapping, alignment, and access permissions are unavailable.
- **documented_only — calling convention:** Implicit arguments, register preservation, and the validity of retaining incoming values across calls are unresolved.
- **documented_only — dependency effects:** No dependency’s global-state, memory, ownership, error, or termination behavior is established.
- **documented_only — faults:** No local error handling is visible; environmental exception handling is unknown.
- **documented_only — memory semantics:** Volatility, atomicity, concurrency, and device-access rules are unspecified.
- **documented_only — termination:** The final marker does not establish a specific hardware exception, process exit status, or recovery mechanism.

The repeated groups suggest a generated or instruction-like sequence, but that is inferred intent only and imposes no additional contract.

## Evidence

The sole evidence is the supplied static representation. It directly supports the listed access widths, arithmetic expressions, explicit call arguments, represented ordering, and terminal invalid-instruction marker.

No dependency implementations, machine-level validation, memory map, calling-convention specification, or runtime observations were supplied. No runtime tests have been performed.
