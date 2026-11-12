<!-- Trusted provenance: F-cd538fbd5c40; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cd538fbd5c40

## Purpose

F-cd538fbd5c40 describes a side-effecting, incompletely recovered execution path containing indirect calls and computed-address memory accesses. Its established behavior is to:

- Write nine 32-bit words, using either an incoming value, scaled dependency results, or a shifted memory value as payloads.
- Read one unsigned 32-bit word and later propagate its logical right shift by twelve.
- Read two unsigned bytes separately and use each byte’s numeric value to select a destination for a single-byte write.
- Supply selected dependency results, shifts, and masked fields to later dependencies.
- Reach an instruction-recovery boundary after the final visible call, without an established normal return.

No application-level purpose is established. The repeated scaling and fixed displacements do not establish allocation, object management, serialization, or hardware access.

This contract covers the visible path only. Every later event is conditional on the preceding calls returning and preceding accesses completing.

## Interface

### Explicit inputs

| Argument position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | Passed to the first indirect call. |
| Second | 32 bits | No direct use appears in the recovered expressions. |
| Third | 32 bits | No direct use appears in the recovered expressions. |
| Fourth | Signed 32 bits | Shifted left by 26 for an argument to dependency site 11. |

The absence of visible uses for the second and third arguments does not establish that they are irrelevant to the actual calling convention or to hidden machine-state inputs.

### Additional incoming state

The visible path also depends on:

- An incoming control-flow-related value used to form the first call target by adding `0xFFE`.
- An incoming 32-bit register-derived value used as the payload of the first direct word write.

Their initialization, validity requirements, and calling-convention roles are unknown. Four explicit arguments alone are therefore not an established complete invocation interface.

### Output and termination

No ordinary return value or normal return is visible.

After dependency site 64, the recovered path reaches a bad-instruction boundary. This establishes a recovery limit, not a particular executable trap, exception class, exit status, or successful completion condition.

### Width and arithmetic conventions

“Word” means a 32-bit access; “byte” means an 8-bit access. Byte reads are unsigned. Byte writes retain the low eight bits of the specified payload expression.

Captured results used in signed multiplication or signed shifts are represented as signed 32-bit values, except:

- Result 50 is represented as unsigned 32-bit.
- Result 52 is an uninterpreted 32-bit value forwarded to the next call.
- The direct word read and its right shift are unsigned.
- The scaled result 35 is held as an unsigned 32-bit value for subsequent masking.

The representation does not independently settle signed-overflow behavior, signed-left-shift behavior, pointer width, or effective-address wrapping. Arithmetic examples that discard high bits explicitly assume 32-bit bit-vector evaluation; they do not resolve those uncertainties.

## Behavior

### Dependency notation and sequencing

The 64 visible indirect call sites are assigned document-local opaque IDs **F-000000000001** through **F-000000000064**, in execution order. “Site 05,” for example, denotes **F-000000000005**. These IDs identify sites, not authenticated callee identities; different sites may reach the same callee.

“Result 05” means the value captured from site 05. Such labels are document notation, not implementation identifiers.

A call described as having no explicit arguments has no arguments shown in the recovered representation. This is not proof that it has no register, stack, global, or other hidden inputs.

The visible path contains no branch or loop. Its call-and-access ordering is:

| Phase | Ordered calls before the associated access | Associated direct accesses | Following call |
|---|---|---|---|
| Initial propagation | Site 01 receives the first input; sites 02–05 have no explicit arguments | Word write 1 | Site 06 receives four times result 05 |
| First scaled self-store | Sites 07–10 have no explicit arguments | Word write 2 | Site 11 receives three arguments specified below |
| First copied payload | Sites 12–15 have no explicit arguments | Word write 3 | Site 16 receives four times result 15 |
| Second scaled self-store | Sites 17–20 have no explicit arguments | Word write 4 | Site 21 receives four times result 20 |
| Second copied payload | Sites 22–25 have no explicit arguments | Word write 5 | Site 26 receives four times result 25 |
| Third scaled self-store and read | Sites 27–30 have no explicit arguments | Word write 6, then the word read | Site 31 receives four times result 30 |
| Shifted-word propagation | Sites 32–35 have no explicit arguments | Word write 7 | Site 36 receives four times result 35 |
| Fourth scaled self-store | Sites 37–40 have no explicit arguments | Word write 8 | Site 41 receives three arguments specified below |
| Third copied payload | Sites 42–45 have no explicit arguments | Word write 9 | Site 46 receives four times result 45 |
| First byte transfer | Sites 47–50 have no explicit arguments | First byte read, then first byte write | Site 51 receives three arguments specified below |
| Intermediate forwarding | Site 52 has no explicit arguments; site 53 receives result 52 | None | Execution continues to sites 54–57 |
| Second byte transfer | Sites 54–57 have no explicit arguments | Second byte read, then second byte write | Site 58 receives result 57 shifted left by two |
| Final forwarding | Sites 59–62 have no explicit arguments | None | Site 63 receives two arguments specified below; site 64 then has no explicit arguments |

Within each range, calls occur in ascending order. Direct accesses in a row occur after its preceding calls and before its following call.

### Word destinations and payloads

The nine word writes have the following numeric address expressions and 32-bit payloads:

| Write | Destination address | Payload |
|---:|---|---|
| 1 | Four times result 05, plus `0x14` | Incoming register-derived 32-bit value |
| 2 | Sixteen times result 10, plus `0x14` | Four times result 10 |
| 3 | Four times result 15, plus `0x14` | Four times result 10 |
| 4 | Sixteen times result 20, plus `0x14` | Four times result 20 |
| 5 | Four times result 25, plus `0x14` | Four times result 20 |
| 6 | Sixteen times result 30, plus `0x14` | Four times result 30 |
| 7 | Four times result 35, plus `0x14` | Retained unsigned word read, logically shifted right by twelve |
| 8 | Sixteen times result 40, plus `0x14` | Four times result 40 |
| 9 | Four times result 45, plus `0x14` | Four times result 40 |

These are address calculations, not established object layouts. In particular, writes 2, 4, 6, and 8 use a factor of sixteen for the destination contribution while storing a value scaled by four.

The sole direct word read occurs immediately after write 6, before site 31. Its address is **four times result 30, plus `0x14`**, not sixteen times that result plus `0x14`.

Consequently, the read cannot generally be replaced by write 6’s payload. No dependency intervenes between that write and read, but their addresses are generally different. The unsigned read value is retained across subsequent calls and supplies write 7 after a logical right shift by twelve.

### Multi-argument calls

Site 11 receives, in order:

1. Four times result 10.
2. The same scaled value.
3. The fourth explicit input shifted left by 26.

Site 41 receives, in order:

1. The low six bits of the **scaled result 35**, shifted left by fourteen.
2. Four times result 40.
3. The **unscaled result 35** shifted left by thirty.

The first argument to site 41 is not formed from the low six bits of the unscaled result. Under 32-bit bit-vector evaluation, scaling by four makes the bottom two bits zero before the mask is applied.

Site 51 receives, in order:

1. Result 50 shifted left by two.
2. The same shifted value.
3. Result 50 shifted left by four.

Result 50 is unsigned for these shifts. Each shifted result has 32-bit width. The single-byte write preceding this call uses only the low eight bits of the first shifted value.

Site 63 receives, in order:

1. Result 62 shifted left by two.
2. The value obtained by the **second** byte read, zero-extended to 32 bits.

### Byte-source addressing and writes

Both byte reads use an address formed from **result 35 multiplied by `0x10000000`, plus one**. This is a numeric address expression subject to the unresolved arithmetic and pointer rules.

The two reads are separate observable accesses:

- The first occurs after site 50 and before site 51.
- The second occurs after site 57 and before site 58.

For each read, its unsigned byte value is increased by `0x11`; that sum is the destination address of the corresponding byte write. No independently established base pointer participates in this destination calculation.

The first byte write stores the low eight bits of result 50 shifted left by two.

The second byte write stores the low eight bits of result 50 after masking with `0x3FFFFFFF` and logically shifting right by twenty. It does **not** use result 57 as its payload.

The second read’s value is retained for site 63 independently of the second write. A later change to the source memory does not, in the represented data flow, replace that retained value.

The first read cannot substitute for the second. Intervening dependency effects, external activity, or an aliased earlier write may cause the reads to differ.

## Side effects

The complete visible path contains:

- **Nine direct 32-bit word writes**, associated with results 05, 10, 15, 20, 25, 30, 35, 40, and 45.
- **Two direct byte writes.**
- **One direct unsigned word read.**
- **Two direct unsigned byte reads.**
- **64 indirect calls.**

These counts describe represented accesses, not guaranteed machine transactions, atomic operations, or successful completions.

The byte destinations range numerically from `0x11` through `0x110`, inclusive. This follows from zero-extending an unsigned byte and adding `0x11`. It does not establish that any destination is mapped, writable, ordinary memory, or safe.

Computed accesses may alias. A later word write may overwrite an earlier one; the word read may observe an earlier write; and a byte write may alter a location subsequently read. The byte writes do not establish a contiguous two-byte output.

No validation, rollback, or recovery branch is visible. If a dependency does not return or an access fails, earlier effects may remain. No particular error code, exception, or cleanup action is established.

No direct allocation, release, ownership transfer, locking, or named global-state access is visible. This does not constrain dependency behavior. Dependencies may mutate memory or globals, acquire or release resources, or change execution state; preservation and ownership must not be assumed.

## Dependencies

All visible calls are indirect.

The first target is derived from incoming control-flow-related state plus `0xFFE`. Later targets are formed from embedded address anchors with an added displacement. Site 51 uses displacement `0xFEC`; the other visible sites use `0xFFE`.

The different displacement at site 51 must be preserved in a source-faithful interpretation. Neither proximity nor similar arguments establish that it invokes the same dependency as another site.

Results are explicitly consumed from sites:

**05, 10, 15, 20, 25, 30, 35, 40, 45, 50, 52, 57, and 62.**

Other return values have no explicit data use in the recovered expressions. Their calls remain observable events and cannot be omitted on that basis.

For every dependency, the following remain unknown:

- Actual identity, signature, and calling convention.
- Hidden inputs and additional outputs.
- Memory, global-state, and resource effects.
- Ownership acquisition, retention, transfer, or release.
- Whether it returns normally, faults, or transfers control elsewhere.
- Preservation of machine state beyond what the recovered data flow represents.

Traversal requires each target to be callable and each dependency to return sufficiently for the next event to occur. Those are conditional execution prerequisites, not checks performed by this routine.

## Acceptance examples

These are conditional static acceptance criteria, not runtime observations. Unless stated otherwise, they assume valid accesses, returning dependencies, and arithmetic representable without overflow.

### Different word-store and word-read addresses

If result 30 is 2:

- Write 6 targets `0x34` and stores 8.
- The following word read targets `0x1C`.

The read must use memory at `0x1C`; returning 8 solely because of the preceding store would not preserve the represented behavior.

### Coincident addresses

If result 30 is zero, write 6 and the following read both address `0x14`, and the write payload is zero.

Under ordinary coherent memory semantics, with no external modification or access-specific behavior, the read observes zero. That memory model is an explicit condition, not an established property of the address.

### Unsigned word propagation

If the retained word read is `0xABCDEF01`, write 7 stores `0x000ABCDE`.

An arithmetic right shift producing sign-extension bits would not satisfy the unsigned operation.

### Byte payloads and independent destination selection

If result 50 is `0x12345678`:

- The first byte payload is `0xE0`.
- The second byte payload is `0x23`.

If the first source-byte read returns `0x20`, the first destination is `0x31`. If the second source-byte read returns `0x21`, the second destination is `0x32`, and site 63’s second argument is `0x00000021`.

The implementation must not reuse the first read’s `0x20` for either second-read obligation.

### Scaled masking

Under 32-bit bit-vector evaluation, if result 35 is 3:

- Its scaled value is 12.
- Site 41’s first argument is `0x00030000`.
- Site 41’s third argument is `0xC0000000`.

Masking the unscaled value instead would produce a different first argument.

### Fourth-input shift

Under 32-bit bit-vector evaluation:

- A fourth input of 1 gives site 11 a third argument of `0x04000000`.
- A fourth input of 64 gives zero because the shifted bit lies beyond the retained width.

The second case is not a claim that the represented signed shift has independently established machine semantics.

### Partial execution

If site 36 does not return, write 7 has already occurred. The visible path does not require any subsequent write, byte access, or call to occur, and establishes no rollback.

## Uncertainties

The following remain **documented_only**:

- Whether the recovered path faithfully represents executable instructions; the supplied material explicitly reports bad instruction data and truncated control flow.
- Actual ABI, pointer width, hidden inputs, and register-preservation rules.
- The origin and validity of the additional incoming state.
- Signed multiplication overflow, signed left shifts, and effective-address wrapping.
- Memory mappings, permissions, alignment requirements, byte order, atomicity, and device or volatile semantics.
- Every dependency’s identity, behavior, ownership effects, and global-state effects.
- Whether any computed address denotes an object field, global location, device register, or invalid address.
- The actual outcome at the terminal recovery boundary.
- Whether execution outside the recovered path provides additional branches or normal completion.

**Inferred intent:** repeated scaling and displacements could reflect encoded references or field-like accesses. That interpretation is not required for the contract and does not justify assigning object types, resource ownership, or an application role.

## Evidence

**Established static evidence:** the supplied representation shows four explicit arguments, additional incoming-state uses, 64 ordered indirect call sites, nine word writes, two byte writes, one unsigned word read, two unsigned byte reads, the stated arithmetic and data dependencies, and a terminal bad-instruction report.

The nine-word-write count is supported by the distinct writes associated with captured results 05 through 45 at five-site intervals. The scaled self-stores and copied-payload stores are separate writes, while the access following write 6 is a read, not a tenth word write.

**Evidence limits:** recovered types, apparent call signatures, and instruction boundaries are not independent proof of the original machine-level interface. Retained-value relationships describe the supplied representation, not a verified ABI.

No runtime tests have been performed. No dependency implementations, executable-memory mappings, or external behavioral observations were available.
