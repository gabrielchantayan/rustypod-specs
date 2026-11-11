<!-- Trusted provenance: F-9cc79e1f1b5d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9cc79e1f1b5d

## Purpose

F-9cc79e1f1b5d specifies a represented sequence of narrow memory writes, global zero assignments, and calls, ending at an invalid-instruction boundary.

The supplied static representation supports an ordered low-level behavioral contract. It does not establish an application-level purpose, a valid executable entry point, or successful execution. In particular, initialization, reset handling, device control, and decoding are not established purposes.

The representation includes warnings about invalid instruction data and overlapping global symbols. All behavior below is conditional on the represented operations accurately describing the executed path. No runtime tests have been performed.

## Interface

### Inputs and machine state

The displayed interface declares neither parameters nor a return value. Nevertheless, the represented behavior depends on these incoming quantities:

| Contract name | Width | Meaning within this contract |
|---|---|---|
| P | 32 bits | Address used for initial halfword writes; also supplies their data |
| B | 32 bits | Base address for two early word writes |
| U | 32 bits, unsigned | Input data used directly and in address and data calculations |
| R | 32 bits | Control-address quantity used to derive the first call target |

These are descriptive contract names, not source identifiers or a verified public calling convention. Their placement in incoming machine state is **documented_only**.

The function also assigns zero to a global object called **G** here. G’s address, access width, storage layout, and overlap with other objects are not supplied.

### Outputs and completion

No explicit return value or normal return is represented.

Ten calls expose represented 64-bit results. For result number \(n\):

- \(L_n\) denotes its low 32 bits.
- \(H_n\) denotes its high 32 bits.

These portions supply later writes and call arguments. Whether they correspond to an actual 64-bit return convention or another machine-state interpretation remains unresolved.

After the final call, the representation terminates at an invalid-instruction marker. The actual exception, trap, recovery, or termination mechanism is **documented_only**.

### Width and layout rules

The arithmetic descriptions below use the represented 32-bit machine model:

- Address addition, doubling, multiplication, and left shifts retain the low 32 bits.
- Right shifts of unsigned quantities insert zero bits.
- A halfword write transfers 16 bits.
- A word write transfers 32 bits.
- Narrowing to 16 bits retains the low 16 bits.
- Signed interpretations of argument patterns do not change their 32-bit representation.

This is a bit-pattern interpretation of the supplied material, not a guarantee about portable language arithmetic.

The low and high portions of a result are numerical bit ranges, not a claim about their order in memory. Endianness and alignment requirements are unknown.

## Behavior

### Ordering and progress

The represented path has no explicit conditional branch. Progress to each later operation requires all preceding accesses to complete and all preceding calls to return.

Every listed write is a separate represented operation. Consecutive identical writes must remain distinct in this contract; reducing them to a final memory value loses ordering and access-count information.

No explicit data-memory read appears in the visible body. Values come from incoming state, constants, and represented call results. Dependencies may perform additional reads, writes, or other effects.

A dependency’s effects occur at its position in the sequence. No dependency is assumed to preserve G, other globals, memory contents, or ownership.

### Initial writes and result 1

Before the first result-producing call, the represented sequence is:

1. Two consecutive halfword writes at address P, each containing P’s low 16 bits.
2. One halfword write at the low-32-bit value of twice P, containing the same halfword.
3. One word write of U at address B + 20.
4. A call to F-dep-01, whose target is R + `0x00000FFE`.
5. A call to F-dep-02, producing result 1.

All address arithmetic wraps to 32 bits.

After result 1, two consecutive halfword writes occur at address \(L_1\) shifted right by eight bits. Each contains bits 8 through 23 of \(L_1\).

### Input-derived writes and result 2

Define **A** as U’s low 28 bits shifted right by 18. Its range is 0 through 1,023.

Following the pair of writes derived from result 1:

- Two consecutive halfword writes occur at address A.
- One halfword write occurs at address twice A.
- Each of these three writes contains A.
- A second word write of U occurs at B + 20.
- F-dep-03 is called with A, \(H_1\), and A + `0x08910015`.
- F-dep-04 is called, producing result 2.

The input-derived halfword data equals A because the represented intermediate narrowing does not discard any surviving significant bits before the final right shift. This equality is derived from the widths; it is not an application convention.

A is supplied as an address to F-dep-03. That does not establish the referenced object’s size, ownership, lifetime, or accessibility.

### Two result-derived write forms

Subsequent stages use the following forms. Each form means **two consecutive halfword writes** to the specified address.

| Form | Address derived from low result portion L | Halfword data |
|---|---|---|
| Byte-shift | L shifted right by 8 | Bits 8–23 of L |
| Masked-bit | L’s low 27 bits shifted right by 3 | Bits 3–18 of L |

For either form, the written halfword equals the destination address’s low 16 bits.

The masked-bit data calculation includes a 32-bit left shift by five, followed by an unsigned right shift by eight and narrowing to 16 bits. It is not an unrestricted-width shift calculation. Under the represented widths, the surviving bits are exactly bits 3–18 of L.

Both forms yield addresses from `0x00000000` through `0x00FFFFFF`. Neither form guarantees alignment.

### Results 2 through 10

Define two additional quantities:

- **D**: U multiplied by 16, retaining the low 32 bits.
- **V**: U shifted left by eight bits, retaining the low 32 bits.

For each row below, the order is: the result-derived halfword pair, then a zero assignment to G, then the listed word write if present, then the listed calls.

| Result | Producing dependency | Halfword form | Following word write | Following calls |
|---|---|---|---|---|
| 2 | F-dep-04 | Byte-shift | U at D + 20 | F-dep-05, then F-dep-06 |
| 3 | F-dep-06 | Masked-bit | U at D + 20 | F-dep-07, then F-dep-08 |
| 4 | F-dep-08 | Masked-bit | U at D + 20 | F-dep-09, then F-dep-10 |
| 5 | F-dep-10 | Masked-bit | U at D + 20 | F-dep-11, then F-dep-12 |
| 6 | F-dep-12 | Masked-bit | U at D + 20 | F-dep-13, then F-dep-14 |
| 7 | F-dep-14 | Byte-shift | V at D + 20 | F-dep-15, then F-dep-16 |
| 8 | F-dep-16 | Masked-bit | V at D + 20 | F-dep-17, then F-dep-18 |
| 9 | F-dep-18 | Masked-bit | V at D + 20 | F-dep-19, then F-dep-20 |
| 10 | F-dep-20 | Byte-shift | None | Final call sequence |

The destination D + 20 remains derived from the original U. Changing the stored data from U to V does not change that destination.

For results 2 through 9, the first following call receives zero, the current result’s high portion, and the corresponding continuation token listed under Dependencies. The second following call produces the next result.

G receives nine represented zero assignments: one after each result-derived pair for results 2 through 10. “Zero assignment” does not establish a byte count or hardware meaning.

### Final calls and boundary

After result 10’s halfword pair and the ninth zero assignment to G:

1. F-dep-21 receives four represented arguments:
   - `0xFFFFFF1C`, or −228 under signed interpretation;
   - \(H_{10}\) minus 612, retaining the low 32 bits;
   - 75;
   - zero.
2. F-dep-22 is called.
3. F-dep-23 is called.
4. F-dep-24 is called.
5. F-dep-25 is called.
6. The represented path reaches the invalid-instruction boundary.

No explicit memory writes appear between those final calls. Their implementations may nevertheless modify memory or other state.

## Side effects

### Direct writes

If all operations are reached, the visible body contains:

| Write category | Count | Detail |
|---|---:|---|
| Halfword writes | 26 | Three derived from P, three derived from A, and two for each of ten results |
| Word writes | 10 | Two at B + 20; eight at D + 20 |
| Zero assignments to G | 9 | Access width unknown |

Of the eight word writes at D + 20, five contain U and three contain V.

These counts describe represented operations, not distinct addresses or verified hardware transactions. Addresses may coincide or partially overlap. G may overlap another destination or other global storage. Dependencies may introduce additional effects between visible writes.

### Preconditions and failure behavior

Full traversal requires:

- Every reached address to permit the represented access width.
- Every reached call target to be executable under the actual execution environment.
- Every dependency to return to the represented continuation.
- The represented machine-state interpretation to be applicable.

The visible body contains no null, bounds, alignment, permission, or dependency-error check. A computed zero address is not rejected before use.

If a write faults or a call does not return, later represented operations are not reached through the displayed path. No rollback, cleanup, retry, or explicit error-reporting path is shown. Earlier completed effects are not explicitly undone.

### Ownership and lifetime

No explicit allocation, release, reference-count update, or ownership transfer is visible in this body.

Dependency ownership behavior remains unknown. In particular, the contract does not assume that dependencies preserve allocations, refrain from releasing memory, or avoid retaining the address supplied to F-dep-03.

## Dependencies

The IDs below are document-local opaque identifiers. They identify call sites and their roles; they are not authoritative cross-document callee identities.

**Every call has a represented target.** Missing authoritative callee identities do not make the target arithmetic unknown. Target values below are represented control-address patterns, not verified executable addresses. No instruction-set mode adjustment, alignment correction, or low-bit removal is implied.

“No explicit arguments” describes only the supplied representation. It does not establish that a dependency ignores other incoming machine state.

| Dependency | Represented target | Represented arguments or result |
|---|---|---|
| F-dep-01 | R + `0x00000FFE`, modulo \(2^{32}\) | No explicit arguments |
| F-dep-02 | `0x08911011` | No explicit arguments; produces result 1 |
| F-dep-03 | `0x08911013` | A, \(H_1\), A + `0x08910015` |
| F-dep-04 | `0x08911079` | No explicit arguments; produces result 2 |
| F-dep-05 | `0x0891107B` | 0, \(H_2\), `0x0891007D` |
| F-dep-06 | `0x089110E1` | No explicit arguments; produces result 3 |
| F-dep-07 | `0x089110E3` | 0, \(H_3\), `0x089100E5` |
| F-dep-08 | `0x08911149` | No explicit arguments; produces result 4 |
| F-dep-09 | `0x0891114B` | 0, \(H_4\), `0x0891014D` |
| F-dep-10 | `0x089111B1` | No explicit arguments; produces result 5 |
| F-dep-11 | `0x089111B3` | 0, \(H_5\), `0x089101B5` |
| F-dep-12 | `0x08911219` | No explicit arguments; produces result 6 |
| F-dep-13 | `0x0891121B` | 0, \(H_6\), `0x0891021D` |
| F-dep-14 | `0x08911281` | No explicit arguments; produces result 7 |
| F-dep-15 | `0x08911283` | 0, \(H_7\), `0x08910285` |
| F-dep-16 | `0x089112E9` | No explicit arguments; produces result 8 |
| F-dep-17 | `0x089112EB` | 0, \(H_8\), `0x089102ED` |
| F-dep-18 | `0x08911351` | No explicit arguments; produces result 9 |
| F-dep-19 | `0x08911353` | 0, \(H_9\), `0x08910355` |
| F-dep-20 | `0x089113B9` | No explicit arguments; produces result 10 |
| F-dep-21 | `0x08910BB3` | `0xFFFFFF1C`, \(H_{10}-612\), 75, 0 |
| F-dep-22 | `0x08910CD9` | No explicit arguments |
| F-dep-23 | `0x0850C4DE` | No explicit arguments |
| F-dep-24 | `0x08112CF0` | No explicit arguments |
| F-dep-25 | `0x08D2551A` | No explicit arguments |

The fixed targets for F-dep-02 through F-dep-20 result from adding `0x00000FFE` to their respective represented address constants. F-dep-21 and F-dep-22 instead use an increment of `0x000007F6`. F-dep-23 through F-dep-25 have statically designated targets rather than the preceding addition form.

The continuation tokens in the argument column are data arguments. They must not be confused with the call targets. All adjusted token and argument arithmetic follows the stated 32-bit rules.

Dependency implementations, effects, return guarantees, error conventions, preservation rules, and ownership contracts are **documented_only**.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

### Initial address and data distinction

If P is `0x00001234`, the initial halfword writes are:

- Two writes of `0x1234` at `0x00001234`.
- One write of `0x1234` at `0x00002468`.

If P is `0x80001234`, the third destination is still `0x00002468` under 32-bit wrapping. No overflow rejection is represented.

### Input-derived narrowing

If U is `0xFABCDEF0`, A is `0x000002AF`.

The input-derived sequence therefore contains two halfword writes of `0x02AF` at `0x000002AF`, followed by one at `0x0000055E`. The odd first destination does not trigger any represented alignment check.

If U is zero, A is zero and all three input-derived halfword writes target zero. This is an arithmetic outcome, not a guarantee that the accesses succeed.

### Distinct result forms

For a low result portion of `0x12345678`:

- Byte-shift form: two writes of `0x3456` at `0x00123456`.
- Masked-bit form: two writes of `0x8ACF` at `0x00468ACF`.

Using either form for every result would violate the stage-specific contract.

### Stable destination after data changes

If U is 1, D + 20 is address 36.

The five U-valued writes at that destination contain 1. The three V-valued writes at the same destination contain 256. Recomputing the later destination from 256 would not match the representation.

### Targets are distinct from continuation data

F-dep-02’s represented target is `0x08911011`, obtained from `0x08910013` plus `0x00000FFE`.

F-dep-03’s represented target is `0x08911013`, while its third argument is A + `0x08910015`. Substituting the token for the target would change the represented call.

F-dep-21’s target is `0x08910BB3`, and F-dep-22’s is `0x08910CD9`. If both return, the trailing target order is `0x0850C4DE`, then `0x08112CF0`, then `0x08D2551A`.

These expectations do not establish that any target is executable.

### Final argument wrapping

If \(H_{10}\) is zero, F-dep-21’s second argument is `0xFFFFFD9C`, or −612 under signed interpretation. No rejection of that pattern is represented.

### Interference and incomplete traversal

If a dependency changes G, the next listed zero assignment still occurs if execution reaches it. G’s state during or immediately after the dependency is not predicted.

If the second write of a repeated pair faults, subsequent operations are not established as reached. A final-memory-only observation cannot prove the required repeated-write sequence or call ordering.

## Uncertainties

The following remain **documented_only**:

- Whether the represented entry and path correspond to valid executable instructions.
- Whether invalid instruction data has distorted boundaries, ordering, or calls.
- The actual calling convention and origin of P, B, U, and R.
- The runtime meaning of represented control-address patterns, including instruction-set mode handling.
- The actual validity and implementations of every dependency target.
- Whether the ten represented 64-bit results correspond to true return values.
- G’s address, width, layout, overlap, and semantic role.
- Endianness, alignment rules, memory permissions, and device-memory behavior.
- Aliasing among destinations, G, executable storage, and dependency-managed memory.
- Dependency mutation, memory release, retained references, exceptions, and non-return behavior.
- Actual execution behavior at the terminal invalid-instruction boundary.
- Any application-level purpose.

The repeated patterns may suggest a structured low-level mechanism. That is inferred intent only and adds no behavioral requirement.

## Evidence

**Established static evidence:** the supplied representation contains the ordered writes, widths, shifts, masks, call expressions, concrete target constants, arguments, global zero assignments, and terminal invalid-instruction marker described above. It also reports invalid instruction data and overlapping global symbols.

**Derived static facts:** the fixed target addresses, address ranges, halfword equivalences, write counts, and conditional arithmetic examples follow from the represented constants and widths.

**Not established:** executable validity, successful traversal, a public ABI, callee semantics, global isolation, memory safety, hardware transaction behavior, or normal return.

No runtime tests, executable inspection, dependency inspection, or implementation access have been performed.
