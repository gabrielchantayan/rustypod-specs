<!-- Trusted provenance: F-c5053bfcbd97; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c5053bfcbd97

## Purpose

The available static evidence describes an ordered sequence of indirect calls interleaved with byte reads, byte writes, and 32-bit writes. The sequence forwards selected call results, derives addresses from other results, repeatedly clears a fixed memory location, and writes several scaled values into derived locations.

A higher-level purpose is **not established**. In particular, the evidence does not justify describing this function as an initializer, allocator, decoder, device driver, or object constructor.

The recovered sequence ends at an invalid-instruction boundary rather than a demonstrated normal return. This contract therefore describes the **visible static prefix**, conditional on execution reaching each operation. It does not establish that the entire prefix is executable or that the apparent endpoint is an intentional trap.

## Interface

### Explicit inputs

The recovered interface has two 32-bit arguments:

| Input | Established use |
|---|---|
| First argument | No explicit use appears in the visible prefix. |
| Second argument | Passed to the first indirect dependency call. |

The apparent lack of use of the first argument does not establish that it is irrelevant to the actual machine-level interface. An indirect target could observe argument registers or other entry state not represented as explicit arguments.

### Additional entry state

The visible operations also depend on three incoming machine-state values outside the recovered argument list:

- **Entry address value:** a 32-bit unsigned value used for byte-read addresses, logical right shifts, derived addresses, and stored data.
- **Entry packed value:** a 32-bit unsigned value whose bits 22–29 determine an early byte write.
- **Entry linkage value:** used with an offset of `0xFC4` to derive the first call target.

These names describe analytical roles, not a confirmed calling convention. Their provenance, initialization requirements, and stability under the real ABI are unknown.

A later call also exposes an additional 32-bit result through a secondary return channel. That value is used in a subsequent three-argument call.

### Outputs

No explicit return value is established. The recovered declaration is void-like, but normal return is not visible.

Observable outputs within the recovered prefix consist of:

- Indirect dependency invocations and their explicit arguments.
- Direct memory reads and writes.
- Any additional effects performed by dependencies.
- Encountering the unresolved terminal instruction boundary, if execution reaches it.

### Preconditions

For a particular execution to reach a later operation:

- Every preceding indirect target must be executable and must return.
- Every preceding direct memory access must be valid for its width and access type.
- Required entry state and dependency results must be available under the actual machine calling convention.
- Address arithmetic and alignment must be meaningful on the target machine.

No validation, bounds check, null check, or local error recovery is visible.

## Behavior

### Interpretation and notation

The following description preserves the order shown by the static evidence. It does not certify the recovered control flow.

Unresolved calls receive opaque, call-site-specific dependency labels **F-d001** through **F-d096**. A different label does not prove a different underlying function. Conversely, repeated target patterns do not prove that two calls are equivalent.

For compactness:

- **R7**, for example, denotes the result observed from F-d007.
- A call range lists consecutive invocations in ascending order.
- “No explicit arguments” means none are represented in the supplied evidence; it does not establish that the target observes no register or global state.
- Unused results are not assumed to have any particular value or lifetime.
- A “word” below is exactly 32 bits, not the host platform’s native word size.

Let:

- **E** be the entry address value.
- **P** be the entry packed value.
- **Q** be the logical unsigned shift `E >> 3`.
- **B0** be the unsigned byte read from address `E + 1`.
- **T** be the derived address quantity `Q × 0x80000`.

Products and shifts require the arithmetic qualifications described below.

### Ordered visible prefix

| Order | Calls and direct effects |
|---|---|
| 1 | Invoke F-d001 with the second explicit input. Its target depends on the entry linkage value. |
| 2 | Invoke F-d002 with no explicit arguments. Invoke F-d003 with R2. |
| 3 | Invoke F-d004 through F-d007 with no explicit arguments. Retain R7. |
| 4 | Read B0 from `E + 1`. Write one byte at `B0 + 0x11`, taking the low eight bits of `P >> 22`. |
| 5 | Invoke F-d008 with R7 shifted left by two. Invoke F-d009 through F-d012 with no explicit arguments. |
| 6 | Write a zero word at `B0 + 0x44`. Then write a zero word at fixed address `0x24`. The earlier B0 is reused; no new selector read appears here. |
| 7 | Invoke F-d013 with explicit arguments `0` and `0x98`. Invoke F-d014 with no explicit arguments, then F-d015 with R14. |
| 8 | Invoke F-d016 with no explicit arguments, then F-d017 with R16. Invoke F-d018 through F-d021 with no explicit arguments. |
| 9 | Write a zero word at `4 × R21 + 0x44`. Then write a zero word at `0x24`. Invoke F-d022 with `4 × R21`. |
| 10 | Invoke F-d023 through F-d026 with no explicit arguments. R26 is represented as a 64-bit result with low and high 32-bit halves. |
| 11 | Invoke F-d027 with the low half of R26 shifted left by two. Then write a zero word at `2 × high32(R26) + 0x44`, followed by a zero word at `0x24`. |
| 12 | Let **S** denote the secondary return-channel value exposed after F-d027. Invoke F-d028 with `0`, S, and S shifted left by two. |
| 13 | Invoke F-d029 with no explicit arguments. Invoke F-d030 with Q. Invoke F-d031 through F-d034 with no explicit arguments. |
| 14 | Write a zero word at `4 × R34 + 0x44`, then a zero word at `0x24`. Invoke F-d035 with `4 × R34`. |
| 15 | Invoke F-d036 through F-d039 with no explicit arguments. Invoke F-d040 with R39 shifted left by two; retain R40. |
| 16 | Write a zero word at `4 × R40 + 0x14`. Invoke F-d041 with four explicit arguments: R40, R40, R40, and Q. |
| 17 | Invoke F-d042 through F-d045 with no explicit arguments. Write a zero word at `4 × R45 + 0x14`. Invoke F-d046 with `4 × R45`. |
| 18 | Invoke F-d047 through F-d050 with no explicit arguments. Let **V** be `4 × R50`. Write V as a word at `16 × R50 + 0x14`. Invoke F-d051 with V. |
| 19 | Invoke F-d052 through F-d055 with no explicit arguments. Write the retained V as a word at `4 × R55 + 0x14`. Invoke F-d056 with `4 × R55`. |
| 20 | Invoke F-d057 through F-d060 with no explicit arguments. Write `4 × R60` as a word at `16 × R60 + 0x14`. Invoke F-d061 with `4 × R60` and `16 × R60`. |
| 21 | Invoke F-d062 with no explicit arguments. Invoke F-d063 with R62 and the logical unsigned shift `E >> 6`. Invoke F-d064 through F-d067 with no explicit arguments. |
| 22 | Write Q as a word at `4 × R67 + 0x14`. Invoke F-d068 with `4 × R67`. |
| 23 | Invoke F-d069 through F-d072 with no explicit arguments. R72 is represented as an unsigned 32-bit value. Read an unsigned byte at `T + 1`; write the low eight bits of `R72 << 2` at that byte’s value plus `0x11`. Invoke F-d073 with `R72 << 2`. |
| 24 | Invoke F-d074 with no explicit arguments, then F-d075 with R74. Invoke F-d076 through F-d079 with no explicit arguments. Read a fresh unsigned byte at `T + 1`; write the low eight bits of `(R72 & 0x3FFFFFFF) >> 20` at that byte’s value plus `0x11`. Invoke F-d080 with R79 shifted left by two. |
| 25 | Invoke F-d081 through F-d084 with no explicit arguments. R84 is represented as an unsigned 32-bit value. Read a fresh unsigned byte at `T + 1`, retaining it as **B1**. Write the low eight bits of `4 × R84` at `B1 + 0x11`. Invoke F-d085 with B1 zero-extended to 32 bits, not with the newly written byte. |
| 26 | Invoke F-d086 with no explicit arguments, then F-d087 with R86. Invoke F-d088 through F-d091 with no explicit arguments. Read an unsigned byte at `8 × R84 + 1`; write the low eight bits of `(R84 & 0x3FFFFFFF) >> 20` at that byte’s value plus `0x11`. Invoke F-d092 with R91 shifted left by two. |
| 27 | Invoke F-d093 through F-d096 with no explicit arguments. The supplied recovery then terminates at an invalid-instruction boundary. No normal return or subsequent operation is established. |

### Widths, truncation, and address layout

The byte reads are unsigned: a read value is between 0 and 255. Consequently, adding `0x11` selects an address between `0x11` and `0x110`. The first word destination, `B0 + 0x44`, lies between `0x44` and `0x143`.

These are literal address expressions in the available evidence. There is no established object base, allocation, array, or structure underlying them. In particular, a selector byte must not silently be reinterpreted as a full pointer.

Each byte write retains eight bits:

- The early write selects bits 22–29 of P.
- The writes derived from multiplication by four have their lowest two bits clear after truncation.
- The masked right-shift writes ultimately select bits 20–27 of the corresponding retained unsigned result. Bits 28–29 survive the mask and shift but do not survive the final byte truncation.

All direct word writes occupy four bytes. Some derived addresses can be unaligned. The evidence does not establish whether the target accepts, traps on, or otherwise handles such accesses.

The representation uses both signed 32-bit values and unsigned 32-bit values. Signed multiplication and signed left shifts must not be assigned unrestricted mathematical semantics: overflow and negative operands are not resolved by the supplied evidence. For machine-level interpretation, 32-bit truncation is a plausible model, but its exact application to every recovered expression is **documented_only**, not independently verified.

Unsigned shifts of E, R72, and R84 have the represented logical-shift interpretation. T is stored through a signed 32-bit representation despite being derived from unsigned arithmetic. The exact treatment of high-bit-set addresses remains a target-level question.

No byte order is established for nonzero word stores. Zero word stores are byte-order independent.

## Side effects

### Direct memory effects

The visible prefix contains:

- Five byte reads used as selectors.
- Five byte writes.
- Four zero-word writes at derived addresses with offset `0x44`.
- Four separate zero-word writes at fixed address `0x24`.
- Two zero-word writes at derived addresses with offset `0x14`.
- Four nonzero-or-result-dependent word writes at derived addresses with offset `0x14`.

These counts describe recovered operations, not guaranteed completed accesses.

The four fixed-address clears are distinct ordered events. They cannot be collapsed into one clear merely because their values match. Dependencies execute between them and may read or modify that address.

Likewise, the three reads from `T + 1` are distinct. A dependency may change the selector between reads. The later read at `8 × R84 + 1` uses a different expression and is not established to select the same location.

### Aliasing and ownership

No non-aliasing guarantee is visible. Derived destinations can coincide with one another, selector locations, or fixed address `0x24`. Later writes can therefore overwrite earlier effects.

No allocation, ownership acquisition, ownership transfer, release, or resource lifetime rule is established. No dependency may be assumed to preserve globals, preserve referenced memory, or release memory.

### Failure behavior

There is no visible local status check, exception handler, rollback, cleanup path, or retry. If a dependency fails to return or a memory access faults, the later visible operations are not established to occur.

Earlier completed effects are not shown being undone.

## Dependencies

F-d001 through F-d096 identify unresolved indirect call sites in execution order. Their explicit inputs and consumed outputs are specified in the behavior table.

The available private mapping identifies only F-c5053bfcbd97 itself. It does not provide authoritative function identities for these dependencies. The call-site labels therefore provide traceability without claiming resolution.

Established dependency characteristics:

- F-d001 has an entry-state-dependent target.
- Later targets are expressed using fixed-address-derived calculations in the recovered material.
- F-d026 supplies a represented 64-bit result.
- F-d027 exposes a secondary result channel consumed by F-d028.
- Several results are forwarded unchanged, while others are scaled or split.
- Every call can have additional unknown effects.

Unknown dependency characteristics include prototypes, calling conventions, hidden arguments, register preservation, global-state effects, memory ownership, termination behavior, and error reporting.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

1. **Initial byte selection.** If `E + 1` yields `0x20`, P is `0xA5C00000`, and execution reaches the first byte write, that write stores `0x97` at address `0x31`. The destination derives from the unsigned selector; the stored value derives from bits 22–29 of P.

2. **Retained initial selector.** If B0 is `0x20` and intervening dependencies change the memory at `E + 1` to `0x40`, the first zero-word destination remains `0x64`. The recovered sequence does not reload B0 before that store.

3. **Ordered clearing.** If R21 is 3 and arithmetic is valid, the corresponding direct effects are a zero word at `0x50`, then a zero word at `0x24`, then F-d022 invoked with 12. Reversing those events would not match the visible order.

4. **Wide result halves.** If R26 has low half 3 and high half 5, F-d027 receives 12. After that call returns, the next derived zero-word destination is `0x4E`. The high half is multiplied by two, not four. Successful completion additionally depends on the target’s alignment behavior.

5. **Retained scaled value.** If R50 is 3 and R55 is 7, the sequence stores 12 at `0x44`, then later stores the same retained 12 at `0x30`, assuming intervening calls return. The second stored value is not 28.

6. **Repeated selector reads.** If the first read at `T + 1` yields `0x10` and the second yields `0x20`, their byte-write destinations are `0x21` and `0x31`. Reusing the first selector would violate the represented read sequence.

7. **Byte truncation and forwarded selector.** If R84 is `0x12345678` and B1 is `0x20`, the byte written at `0x31` is `0xE0`. F-d085 receives `0x20`, not `0xE0`.

8. **Last masked byte.** With R84 equal to `0x12345678`, the last byte write stores `0x23`. Its selector is freshly read from the address derived from `8 × R84 + 1`, subject to unresolved address-width arithmetic.

9. **Interrupted execution.** If F-d040 does not return, the subsequent zero-word write derived from R40 and all later listed operations are not required to occur. No compensating action is visible.

## Uncertainties

### Established static evidence

The supplied representation establishes the listed operations, their represented widths, explicit dataflow, and visible ordering. It also explicitly reports invalid instruction data and overlapping global representations.

### Inferred intent

Repeated clears and scaled stores could indicate state construction or configuration. That interpretation is not necessary to describe the visible behavior and is not an acceptance requirement.

### documented_only

The following remain unresolved:

- Whether the recovered entry, instruction boundaries, and full linear sequence are correct.
- The actual ABI and origins of the extra entry-state values.
- Whether the apparent low-memory accesses are genuine accesses or recovery artifacts.
- Exact signed-overflow, address-wrap, and alignment behavior.
- Byte order for nonzero word stores.
- Dependency identities, prototypes, effects, and preservation guarantees.
- Whether the secondary return channel after F-d027 is correctly attributed.
- Whether execution continues beyond the reported invalid instruction.
- Whether any normal return, caller-visible error, or intentional termination exists.

## Evidence

This contract is based solely on the supplied static representation for F-c5053bfcbd97. No implementation access, independent instruction inspection, dependency inspection, or runtime tests have been performed.

The representation’s explicit warnings limit confidence in control-flow recovery and global-object interpretation. Accordingly, this document specifies the visible prefix conditionally and leaves unsupported machine-level and semantic claims unresolved.
