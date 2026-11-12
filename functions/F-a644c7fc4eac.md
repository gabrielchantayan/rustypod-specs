<!-- Trusted provenance: F-a644c7fc4eac; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a644c7fc4eac

## Purpose

F-a644c7fc4eac increments a selected record’s count and either accepts the resulting reference immediately or initializes the record through several dependency calls.

Its 32-bit return value is:

- **1:** the count observed after acquisition exceeds one, or the final initialization dependency returns nonzero.
- **0:** an explicit failure gate is reached and an error-reporting dependency is called.

Success does not independently establish descriptor validity, usable backing storage, resource ownership, or arithmetic safety.

Reference acquisition, storage configuration, and synchronization are **inferred intent**. The established behavior consists of the accesses, arithmetic, branches, calls, and returns described below.

## Interface

### Argument and addressing

The sole argument is an unsigned 32-bit index.

The full index:

- Selects a record with a **508-byte stride**.
- Is passed to F-48a3ec979f59.
- Controls mapping selection.

Its low 16 bits are passed to F-c828afef6d06 and eventually stored in the record.

There is no explicit bounds check. Indices greater than three remain eligible for initialization, although they receive no explicit mapping assignment.

Record-address formation combines a 32-bit base value with the index multiplied by 508. The represented multiplication and addition are unsigned 32-bit operations, reduced modulo \(2^{32}\). Address validity, alignment, backing extent, and integer-to-pointer interpretation are **documented_only**. Arithmetic wrap does not imply accessible storage.

### Global inputs and base indirection

The function reads:

- A global pointer whose target holds a 32-bit record-array base.
- An unsigned 32-bit classification threshold.
- Four 32-bit mapping values.

There are **two distinct reads of the global base-holder pointer**:

1. Immediately after F-42af45817af2 returns, the pointer global is read and captured, even if that dependency returned zero.
2. Only when the availability result is nonzero, the pointer global is read again. The target of this second pointer is dereferenced for the nonzero-base gate.

After F-4c6d30ac2311, record selection dereferences the **initially captured pointer**, not necessarily the pointer used by the gate. These pointers need not identify the same location. Neither their equality nor preservation of their contents is established.

### Record footprint

Offsets are bytes from the selected record. This is an access footprint, not a complete structure definition.

| Offset | Width | Role |
|---|---:|---|
| 0 | 32 bits | Count, incremented and compared as signed |
| 96 | 32 bits | Preparation dependency result |
| 100 | 16 bits | Masked unit size |
| 104 | 32 bits | Unit size minus one |
| 108 | 16 bits | Classification, written twice |
| 112, 116 | 32 bits each | Derived boundaries |
| 120 | 16 bits | Truncated index |
| 124 | 32 bits | Dependency result plus one |
| 128 | 32 bits | Initial address |
| 132 | 16 bits | Rounded count; conditionally written |
| 144 | 16 bits | Constant 32 |
| 440 | 32 bits | Mapping; conditionally written |
| 458 | 16 bits | Copied descriptor scale source |
| 460 | 16 bits | Scale-factor product |
| 462 | 16 bits | Dependency result |
| 464 | 16 bits | Narrowed address product |
| 466 | 8 bits | Multiplier |
| 468 | 16 bits | Descriptor count |
| 472, 480 | 32 bits each | Scaled extents |
| 476 | 8 bits | Copied descriptor byte |
| 484 | 16 bits | Narrowed signed product |
| 486 | 16 bits | Uninterpreted copied quantity |
| 488 | 32 bits | Scaled quantity |
| 492, 496 | 32 bits each | Allocation parameters |
| 500 | 32 bits | Explicitly cleared allocation state |
| 504 | 16 bits | Conditional late descriptor product |

A later argument calculation reads **32 bits beginning at offset 460**, spanning the fields at 460 and 462, before narrowing to signed 16 bits.

### Descriptor footprint

F-c828afef6d06 receives a pointer into temporary local storage. Subsequent accesses imply this layout:

| Offset | Consumed interpretation |
|---|---|
| 0 | Signature byte |
| 10 | Unsigned 16-bit scale source |
| 12 | Unsigned byte factor |
| 14 | Unsigned 16-bit address factor |
| 16 | Multiplier byte |
| 18 | Unsigned 16-bit count |
| 20, 24 | Unsigned 16-bit primary extent factors |
| 22 | Copied byte |
| 26 | Signed 16-bit factor |
| 28 | Uninterpreted 16-bit quantity |
| 30 | Unsigned 16-bit quantity |
| 36, 40 | 32-bit fallback-factor bit patterns |
| 72 | Unsigned 16-bit flags; subsequently overwritten on one branch |
| 76 | Signed 32-bit adjustment |
| 80 | Signed 16-bit allocation factor |
| 88 | Direct 32-bit allocation value |

The consumed footprint reaches byte 91. The authoritative format, byte order, and actual scratch-storage extent remain **documented_only**.

## Behavior

### Availability, acquisition, and eligibility

F-42af45817af2 runs first. The initial base-holder pointer capture follows its return, before the availability result is tested.

If availability is zero, error **2** is selected. This path includes the pointer-global capture but no explicit dereference of either base holder and no direct record write.

Otherwise, the function rereads the pointer global and dereferences that second pointer. A zero base selects error **3**.

A nonzero checked base permits F-4c6d30ac2311 to run. Record selection then uses the base obtained by dereferencing the **first captured pointer**, plus the index displacement. There is no second nonzero check on that selected base.

The selected record’s count is incremented as a signed 32-bit expression. F-cc60a532ba6c then runs, and the count is read again:

- A signed count greater than one returns **1** immediately.
- One, zero, and negative counts enter initialization.

The decision uses the post-call count, not a saved increment result. Signed increment overflow has no established portable wrapping behavior. Once selected, the record address is retained; later base changes do not redirect processing.

### Preparation and descriptor gates

Initialization begins with another F-4c6d30ac2311 call. F-4042d12a1376 then receives the record address, length 508, and zero.

Next, in order:

1. The count is explicitly set to one.
2. F-1bbf62607642 runs, and its 32-bit result is stored at offset 96.
3. F-cc60a532ba6c runs.
4. F-48a3ec979f59 receives the full index.
5. If that result is nonzero, F-c828afef6d06 receives the low 16 index bits and the temporary descriptor pointer.

A zero result from the full-index gate selects error **4**. A zero descriptor-request result selects error **5**.

Only signature bit patterns **0xE9** and **0xEB** are accepted. Any other signature selects error **1**. There is no further explicit descriptor-validity gate before arithmetic.

The preparation dependency may be intended to reset the record, but its effects are unknown. Fields without subsequent explicit stores must not be assumed zero.

### Initial descriptor projection

For this pre-call calculation phase, define:

- \(B\): the unsigned 16-bit descriptor scale source at offset 10.
- \(S=B\gg9\), ranging from 0 through 127.
- \(C=S\times\) the unsigned descriptor byte at offset 12, ranging from 0 through 32,385.

The initial stores occur in this order:

| Record offset | Value |
|---|---|
| 458 | \(B\) |
| 460 | \(C\), in 16 bits |
| 480 | \(S\) times unsigned descriptor factor 24 |
| 466 | Descriptor byte 16 |
| 468 | Descriptor count 18 |
| 472 | \(S\) times unsigned descriptor factor 20 |
| 476 | Descriptor byte 22 |
| 464 | Low 16 bits of \(S\) times unsigned descriptor factor 14 |
| 484 | Low 16 bits of \(S\) times signed descriptor factor 26 |
| 486 | Descriptor quantity 28 |
| 488 | \(S\) times unsigned descriptor quantity 30 |

These products fit their 32-bit intermediate ranges. The 16-bit stores may discard significant bits.

A zero initial extent at offset 472 triggers replacement with \(S\) times descriptor fallback factor 36. The zero test concerns the computed extent, not merely the original factor.

Next, a zero initial extent at offset 480 triggers replacement with \(S\) times fallback factor 40. Both fallback multiplications use unsigned 32-bit arithmetic and reduce modulo \(2^{32}\), including unsigned interpretation of the fallback factor’s bit pattern.

When \(S=0\), both fallbacks are selected and both results remain zero.

### Address and boundaries

Descriptor flag bit 7 selects the address rule.

When clear, offset 128 receives \(S\) times descriptor address factor 14. On this branch, the function also overwrites the temporary descriptor’s 16-bit slot at offset 72 with the address factor.

When set, offset 128 receives:

\[
(\text{current extent at 480}\times(\text{flags}\mathbin{\&}15))
+(S\times\text{address factor}).
\]

The extent-times-nibble operation is signed 32-bit multiplication and may overflow. The final addition has unsigned 32-bit behavior if the preceding signed operation is defined. After the address store, offset 466 is overwritten with one.

Define:

\[
H=(\text{unsigned multiplier byte at 466})
\times(\text{current extent at 480})
+\text{current address at 128}.
\]

This is unsigned 32-bit arithmetic; the resulting bit pattern is retained in a signed 32-bit intermediate.

For a **zero descriptor count**:

1. Offset 116 receives \(H\).
2. Offset 112 receives \(C\times(\text{signed adjustment at 76}-2)+H\).

The subtraction, multiplication, and final addition in the second expression are signed and may overflow. Offset 132 is not explicitly written on this branch.

For a **nonzero descriptor count**, define:

\[
R=\left\lfloor\frac{\text{count}+15}{16}\right\rfloor.
\]

Here \(R\) ranges from 1 through 4096. Offset 112 receives \(H\), offset 132 receives \(R\) in 16 bits, and offset 116 receives \(H+R\), in that order. The last addition is unsigned 32-bit arithmetic.

### Unit size and dependency-derived quantity

The unit-size value is:

\[
U=(C\times512)\mathbin{\&}\text{0xFE00}.
\]

Only the low seven bits of \(C\) contribute. Offset 100 receives \(U\) in 16 bits, followed by unsigned 32-bit \(U-1\) at offset 104. Thus zero unit size produces **0xFFFFFFFF** at offset 104.

F-d32b2075c9a4 runs next. Its 16-bit result is stored at offset 462.

F-d7be9cfb4000 then receives:

- The signed 32-bit difference between the **current** fields at offsets 472 and 116.
- A signed 16-bit narrowing of a **32-bit read** beginning at offset 460.

The subtraction may overflow. The second access must not be replaced contractually with a standalone 16-bit memory read. In the represented narrowing, the loaded value’s low 16 bits supply the signed argument; byte order and portability of out-of-range signed conversions remain unresolved.

No arithmetic semantics are established for F-d7be9cfb4000. Division, rounding, zero-divisor handling, and exceptional behavior are unknown.

Let its signed 32-bit result be \(Q\). The unsigned threshold global is captured after this call, before calculating:

\[
N=Q+1.
\]

The addition is signed, followed by unsigned 32-bit interpretation and a store at offset 124. For \(Q=2^{31}-1\), no portable result is established.

Neither dependency is assumed to preserve record contents or temporary descriptor storage.

### Allocation

The captured threshold is compared with \(N\) as unsigned 32-bit values.

If **threshold < \(N\)**, these stores occur:

1. Clear offset 500.
2. Store the then-current descriptor value at offset 88 into offset 492.
3. Store the descriptor value at offset 88 into offset 496.
4. Store a narrowed product at offset 504.

For the final product, the function **reads descriptor offset 10 again after both preceding dependencies**. Define \(B_{\text{late}}\) as that then-current unsigned 16-bit scale source, and \(T_{\text{late}}\) as the then-current signed 16-bit factor at descriptor offset 80. Offset 504 receives the low 16 bits of:

\[
(B_{\text{late}}\gg9)\times T_{\text{late}}.
\]

This product fits signed 32-bit arithmetic. It equals \(S\times T_{\text{late}}\) only if the late scale read has the same shifted value as the earlier scale. Preservation is not established. Offset 458 is not refreshed to reflect this late read.

Otherwise, define:

\[
A=(N\mathbin{\&}\text{0x1FFFFF})\gg5.
\]

Offset 492 first receives \(A\). If \(A<2\), it receives a second store of two. Offset 496 then receives the current value at offset 492, followed by clearing offset 500. This branch does not explicitly write offset 504.

### Classification, mapping, and finalization

Offset 108 is written twice.

The provisional classification is three when either:

- \(N<3840\); or
- unsigned 32-bit subtraction of 3839 from \(Q\)’s bit pattern yields a result below 247.

Otherwise it is four. Wherever the stated \(Q+1\) relationship is defined, classification three corresponds to \(N=0\) through 4086 inclusive.

The second classification store uses eight if threshold < \(N\); otherwise it repeats the provisional classification. The threshold is not reread.

Mapping behavior uses the full index:

| Index | Mapping reads, in order | Offset-440 store |
|---|---|---|
| 0 | Mapping 0 | Mapping 0 |
| 1 | Mapping 1 | Mapping 1 |
| 2 | Mapping 1, mapping 2 | Mapping 2 |
| 3 | Mapping 1, mapping 2, mapping 3 | Mapping 3 |
| Other | Mapping 1, mapping 2, mapping 3 | None |

For index zero, its mapping read occurs between the two classification stores. For other indices, mapping reads follow the second classification store. Any mapping store follows the applicable reads.

Offset 144 then receives the 16-bit constant 32. Offset 120 receives the low 16 index bits.

F-fb78cb06a857 receives the record address and two zero arguments. A nonzero result returns **1**. Zero selects error **9**.

### Failure handling

Every explicit failure calls F-3ba698310ee1 with its numeric reason and returns zero.

Errors **2** and **3** cause no explicit count reset.

Errors **4**, **5**, **1**, and **9** call the reporter **before** explicitly setting the selected count to zero. The reporter can therefore encounter the pre-reset count and partially initialized state. No other rollback is established.

## Side effects

Explicit persistent writes target the selected record. The function also writes temporary descriptor storage, including the conditional overwrite at descriptor offset 72.

Dependencies may mutate globals, records, descriptor storage, or other resources. Their interfaces do not establish preservation guarantees.

No explicit allocation, deallocation, ownership transfer, or matching release is established. Failure does not prove that dependency-acquired resources were released.

The descriptor pointer addresses temporary storage. Its lifetime does not support use after this function returns. Whether the recipient retains that pointer is unknown.

The paired F-4c6d30ac2311 and F-cc60a532ba6c calls suggest synchronization, but do not establish mutual exclusion, atomicity, memory ordering, or concurrent safety. Initialization eligibility is tested after the first F-cc60a532ba6c call.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-42af45817af2 | Initial availability gate |
| F-4c6d30ac2311 | Called before acquisition and before preparation |
| F-cc60a532ba6c | Called after each corresponding interval |
| F-4042d12a1376 | Receives record address, 508, and zero |
| F-1bbf62607642 | Supplies the 32-bit offset-96 value |
| F-48a3ec979f59 | Receives the full index; zero rejects |
| F-c828afef6d06 | Receives truncated index and descriptor pointer; zero rejects |
| F-d32b2075c9a4 | Supplies the 16-bit offset-462 value |
| F-d7be9cfb4000 | Receives two record-derived arguments and supplies \(Q\) |
| F-fb78cb06a857 | Receives record address and two zeros; zero rejects |
| F-3ba698310ee1 | Receives error 1, 2, 3, 4, 5, or 9 |

Dependency internals, ownership, cleanup, retention, and preservation remain unknown.

## Acceptance examples

These are conditional static examples, not executed tests. Except where stated otherwise, they require accessible storage, successful preceding gates, defined arithmetic, and no intervening mutation.

- **Unavailable:** availability zero still permits the initial pointer-global capture. Error 2 is reported without explicitly dereferencing the captured pointer or writing a record.
- **Distinct base holders:** the initial pointer capture identifies holder A; the fresh gate read identifies holder B. If B contains a nonzero base, acquisition proceeds, but selection uses A’s contents after F-4c6d30ac2311.
- **Changed selected base:** if that dependency changes A’s contents, selection uses the changed base without another nonzero check.
- **Existing reference:** a count increment from one to two, still observed as two after F-cc60a532ba6c, returns one without requesting a descriptor.
- **Invalid signature:** signature 0x00 reports error 1 before resetting the count.
- **Zero scale:** \(S=0\) selects both extent fallbacks, and both remain zero. The later dependency’s response is not predicted.
- **Flagged address:** \(S=1\), address factor 10, extent 100, and flags 0x82 produce address 210 and multiplier one.
- **Zero-count adjustment:** \(H=100\), \(C=4\), and adjustment five produce offset 116 equal to 100, then offset 112 equal to 112.
- **Rounded count:** count 17 produces \(R=2\), with offset 116 equal to offset 112 plus two modulo \(2^{32}\).
- **Masked unit:** \(C=128\) produces zero at offset 100 and 0xFFFFFFFF at offset 104.
- **Late scale mutation:** earlier descriptor scale 512 gives \(S=1\). If a dependency changes that scale to 1024 before the direct-allocation branch, and the late factor is −3, offset 504 receives **0xFFFA**, not the earlier-scale result **0xFFFD**.
- **Threshold equality:** threshold equal to \(N\) selects calculated allocation and does not force classification eight.
- **Classification boundary:** with threshold at least \(N\), 4086 selects three and 4087 selects four.
- **Minimum allocation:** \(N=63\) stores one at offset 492, then replaces it with two before copying two to offset 496.
- **Unmapped index:** index four reads mappings 1, 2, and 3 without explicitly assigning offset 440.
- **Final rejection:** a zero final result reports error 9 before clearing the count; other initialized fields are not explicitly restored.

## Uncertainties

The following remain **documented_only**:

- Valid indices, pointer validity, backing extent, alignment, and address interpretation.
- Descriptor format, byte order, scratch extent, and complete initialization.
- Dependency mutation of either base-holder pointer, holder contents, record fields, or descriptor storage.
- Preparation effects and conditionally unwritten field contents.
- Ownership, retained pointers, resource cleanup, and synchronization guarantees.
- Signed-overflow behavior and portability of narrowing or signed reinterpretation outside representable ranges.
- F-d7be9cfb4000 semantics and exceptional behavior.
- External invariants that might exclude zero scale, extreme operands, or unsafe arithmetic.

## Evidence

Established behavior derives from the supplied static representation: access widths, indirection levels, conversions, branch conditions, arithmetic, explicit store order, dependency arguments, and returns.

In particular, the representation establishes the unconditional initial pointer-global capture, the separate pointer-global read used by the base gate, and the late descriptor-scale read used for offset 504. It does not establish preservation that would collapse either pair of reads into a single stable value.

Sequencing here does not impose an evaluation order on independent operand reads where none is established. Storage purpose, reset intent, reference ownership, and synchronization intent remain interpretations.

No dependency implementations or authoritative descriptor specification were supplied. **No runtime tests have been performed.**
