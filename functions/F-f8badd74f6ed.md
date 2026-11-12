<!-- Trusted provenance: F-f8badd74f6ed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f8badd74f6ed

## Purpose

F-f8badd74f6ed performs a mode-dependent sequence of numeric processing and update calls for a caller-supplied object. Exactly one mode-byte value selects five updates; all other values select two updates.

The established behavior comprises dependency-call order, object and global reads, split-word arithmetic, temporary-state handling, and normal-return cleanup calls. Numeric results are not explicit arguments to the update dependency. Their effect on updates, if any, is unresolved.

Descriptions such as ratio calculation, normalization, rendering, or resource accounting are inferred intent, not established behavior. This contract assigns no such domain meaning.

The function has no explicit return value. Its externally observable effects arise from reads and dependency calls, including any effects those dependencies produce.

## Interface

### Arguments and result

| Argument | Established width or representation | Use |
|---|---|---|
| Object reference | Address represented by a 32-bit argument | Base for direct field reads; forwarded to dependencies |
| Second argument | 32 bits | No explicit use |
| Context | 32 bits | Forwarded unchanged to temporary initialization |

There is no explicit result value or error status.

The object reference must permit the accesses required by the selected branch. No null, bounds, alignment, or object-type validation is shown. The five-update branch additionally requires a readable global root reference and readable storage at offset `0x30` from that root.

The unused second argument has no established direct effect. This does not establish broader calling-convention behavior.

### Directly accessed object fields

Offsets are byte offsets from the object reference.

| Offset or pair | Width | Established role |
|---|---:|---|
| `0xE8` | 8 bits | Mode selector |
| `0xF8`, `0xFC` | 32 bits each | Two-update branch, first update |
| `0xEC`, `0xF0` | 32 bits each | Two-update branch, second update |
| `0x114`, `0x118` | 32 bits each | Five-update branch, first update |
| `0x120`, `0x124` | 32 bits each | Five-update branch, second update |
| `0x12C`, `0x130` | 32 bits each | Five-update branch, third update |
| `0x138`, `0x13C` | 32 bits each | Five-update branch, fourth update |
| `0x108`, `0x10C` | 32 bits each | Five-update branch, fifth update |

Each pair is read after its corresponding temporary-conversion call and before its update call. The first listed field supplies the earlier field argument. The relative evaluation order of the two reads is not established.

Field meanings and signedness are unknown. The 32-bit values are forwarded without explicit transformation.

### Numeric representations and global storage

Numeric suppliers and several helpers produce 64-bit results. Explicit helper operands use two 32-bit words per 64-bit value, low word first.

Four distinct numeric global-word pairs are referenced:

| Designation | Use |
|---|---|
| Two-update default pair | Initial candidate numeric value in the two-update branch |
| Two-update comparison pair | Second numeric operand to F-11ed8242c40c in that branch |
| Five-update seed pair | Initial operand for aggregate construction |
| Five-update replacement pair | Second numeric operand to F-11ed8242c40c and, conditionally, a later replacement value |

Each pair consists of a low 32-bit word and a high 32-bit word. Their values, numeric interpretation, and stability are unknown. They are global reads, not established immutable constants.

The representation does not establish atomic 64-bit loads or the order of the two word reads within a pair. Reads separated by dependency calls must not be collapsed into one snapshot.

## Behavior

### Entry and branch selection

F-04d52a6d2e3a is called without explicit arguments. After it returns, the function reads the mode byte at object offset `0xE8`.

A byte exactly equal to `1` selects the five-update branch. Every other byte selects the two-update branch. The mode is not explicitly reread.

Consequently, a change to the mode made by the entry dependency affects branch selection. Later changes do not cause an explicit branch switch.

All subsequent sequences assume normal dependency returns.

### Shared update-cycle behavior

Each update cycle has this order:

1. F-0eb164d8526b receives the scratch-storage address and initialized-temporary address, returning a 32-bit value.
2. The two object fields for that cycle are read.
3. F-353caf38ed9a receives the object reference, the returned 32-bit value, and the two field values.
4. F-282eebd7c413 receives the scratch-storage address.

The scratch storage is reused across cycles. Each cycle nevertheless makes a fresh F-0eb164d8526b call and fresh field reads.

“Initialization,” “conversion,” “update,” and “cleanup” describe call-site roles. They do not establish allocation, destruction, formatting, ownership transfer, or purity.

### Two-update branch

#### Preparation and acquisition

F-a4c26c2b4842 receives a local temporary address, the immediate 32-bit value zero, and the context argument. F-7fa874cbac7a then receives the object reference and that temporary address.

F-3d3200bf0b3e supplies the first 64-bit quantity. F-beaa1252c720 subsequently supplies the second. Both suppliers are called regardless of whether the first quantity is zero.

After both suppliers return, the two-update default global-word pair is read and assembled as the initial candidate value. This read occurs even when the nonzero path subsequently replaces the candidate.

#### Conditional numeric processing

The zero test covers the complete first 64-bit quantity, not merely its low word.

If that quantity is zero, the two conversions and combining call described below are skipped. The candidate remains the value captured from the default global pair.

If it is nonzero:

- F-eef4045aa4cc receives the first quantity as low and high words.
- The second quantity is subtracted from the first at 64-bit width.
- F-eef4045aa4cc receives that difference as low and high words.
- F-596e18a33aec receives the converted difference first and the converted first quantity second.
- Its 64-bit result replaces the candidate.

The subtraction propagates a borrow from the low word exactly when the first low word is unsigned-less-than the second low word. The resulting bit pattern is the mathematical difference modulo \(2^{64}\). No explicit rejection, saturation, or clamp handles a negative mathematical difference.

After the conditional processing finishes, the two-update comparison global-word pair is read. F-11ed8242c40c receives the candidate followed by this newly read pair.

Thus, the default-pair read precedes any conditional conversions, while the comparison-pair read follows them. A dependency invoked between those points may change global storage. No stability guarantee is established.

No explicit result assignment or branch follows F-11ed8242c40c. Its operation cannot be specified as a clamp, comparison, or validation solely from this use.

#### Updates and completion

The function performs two shared update cycles:

| Cycle | Field pair |
|---:|---|
| 1 | `0xF8`, `0xFC` |
| 2 | `0xEC`, `0xF0` |

After the second cycle’s scratch cleanup, F-282eebd7c413 receives the initialized-temporary address. The function then returns.

### Five-update branch

#### Preparation and captured handle

F-a4c26c2b4842 receives a local temporary address, zero, and the context argument. F-7fa874cbac7a then receives the object reference and temporary address.

Next, the function reads a global root reference and reads the 32-bit field at offset `0x30` from the referenced storage. That field value becomes the captured external handle.

This capture occurs before either initial quantity supplier. Subsequent handle-dependent calls use the captured value; the root and its field are not explicitly reread for those calls.

#### Component construction

F-3d3200bf0b3e supplies the first 64-bit quantity, followed by F-beaa1252c720 supplying the second. F-eef4045aa4cc converts the first quantity. Its result is the common operand.

Four components are then produced in order:

| Component | Established construction |
|---|---|
| A | F-c077d567b5f1 receives the captured handle. F-eef4045aa4cc is then called with no explicit operands shown. F-596e18a33aec combines that conversion result first and the common operand second. |
| B | F-3b2936fe6771 supplies a handle-associated 64-bit quantity, then F-e833a6939895 supplies another. The former is subtracted from the latter, the difference is converted by F-eef4045aa4cc, and F-596e18a33aec combines the converted difference first and the common operand second. |
| C | F-b5685315bcf0 receives the captured handle. F-eef4045aa4cc is then called with no explicit operands shown. F-596e18a33aec combines that conversion result first and the common operand second. |
| D | F-eef4045aa4cc converts the quantity originally returned by F-beaa1252c720. F-596e18a33aec combines that result first and the common operand second. |

The two conversions without explicit operands have unresolved effective inputs. They must not be interpreted as conversions of zero or of the immediately preceding dependency’s result.

During component B construction, the function saves whether the unsigned low word from F-3b2936fe6771 is less than or equal to the unsigned low word from F-e833a6939895.

That saved condition also determines the low-word borrow for subtracting the former quantity from the latter. The subtraction produces a modulo-\(2^{64}\) bit pattern; the condition itself is not a full-width ordering comparison.

Unlike the two-update branch, this branch does not explicitly test the first initial quantity for zero before using its conversion as the common operand.

#### Aggregate and global-read timing

Only after component D has been computed does the function read the five-update seed global-word pair.

F-8a693c103d1c is then called four times in a left-associated chain:

- The first call receives the captured seed value first and component A second.
- The next receives that result first and component B second.
- The next receives that result first and component C second.
- The final call receives that result first and component D second.

The final result is the aggregate. No arithmetic meaning is assigned to this helper.

After the aggregate has been computed, the five-update replacement global-word pair is read. Call this captured value the **pre-call comparison operand**. F-11ed8242c40c receives the aggregate first and that operand second.

The seed read therefore follows every component-construction dependency. The pre-call comparison read follows all four aggregate-building calls. Neither pair is established to remain unchanged across those dependencies.

#### Conditional replacement after F-11ed8242c40c

After F-11ed8242c40c returns, selection uses the saved unsigned low-word condition from component B:

- If the condition is true, the selected value remains the aggregate. No replacement-pair reread is explicitly shown on this path.
- If the condition is false, the function rereads both words of the five-update replacement global pair. This **post-call replacement value**, not the earlier comparison operand, becomes the selected value.

F-11ed8242c40c may modify the replacement storage; its contract is unavailable. The pre-call argument and post-call replacement must therefore remain distinct even when they happen to have identical bits.

The displayed selection is controlled by the saved low-word condition, not by an explicit comparison between the aggregate and either replacement-pair capture.

#### Updates and interleaved operations

Five shared update cycles follow. Additional numeric calls occur only after the indicated cycle’s scratch cleanup:

| Cycle | Field pair | Following operation |
|---:|---|---|
| 1 | `0x114`, `0x118` | F-1b22d7e17791 receives A first and B second; retain its result |
| 2 | `0x120`, `0x124` | F-1b22d7e17791 receives the preceding result first and C second; retain its result |
| 3 | `0x12C`, `0x130` | F-1b22d7e17791 receives the preceding result first and the selected value second |
| 4 | `0x138`, `0x13C` | No additional arithmetic call shown |
| 5 | `0x108`, `0x10C` | No additional arithmetic call shown |

On the false-condition path, the third interleaved operation therefore receives the post-call replacement capture. It does not reread that global pair at the time of this operation.

The third arithmetic result is not explicitly consumed. No calculated numeric value is an explicit argument to F-353caf38ed9a.

After the fifth scratch cleanup, F-282eebd7c413 receives the initialized-temporary address. The function returns.

## Side effects

### Reads and writes

Direct reads comprise the mode byte, selected object field pairs, branch-specific numeric global-word pairs, and, in the five-update branch, the global root and its `0x30` field.

No direct write to caller-owned object fields or global storage is shown. Local storage holds temporary state, captured values, and intermediate results. Dependencies receive temporary addresses and may write through them.

Dependency calls must not be reordered around reads on an assumption of purity:

- Entry effects may change the mode subsequently read.
- Preparation may change the subsequently captured handle.
- Quantity suppliers may change the two-update default pair before its read.
- Conditional numeric processing may change the two-update comparison pair.
- Component processing may change the five-update seed pair.
- Aggregate processing may change the pre-call comparison operand.
- F-11ed8242c40c may change the pair conditionally reread afterward.
- Earlier conversion, update, cleanup, or arithmetic calls may change fields read by later cycles.

There is no established synchronization, stable snapshot, or protection against concurrent changes.

### Ownership and failure

Temporary addresses refer to local storage whose lifetime ends on return. Exact logical object sizes and initialization extents are not established. Permission for dependencies to retain those addresses is also not established.

The 32-bit result from F-0eb164d8526b is forwarded to F-353caf38ed9a before scratch cleanup. Its ownership, pointer status, aliasing, and post-cleanup validity are unknown.

F-282eebd7c413 is called at cleanup positions, but actual resource release is not established.

There is no explicit error result, recovery path, rollback, or exceptional cleanup mechanism. If a dependency does not return normally, later operations and cleanup calls are not guaranteed.

## Dependencies

| Opaque ID | Established call-site role |
|---|---|
| F-04d52a6d2e3a | Unconditional entry call before mode read |
| F-a4c26c2b4842 | Receives temporary address, zero, and context |
| F-7fa874cbac7a | Receives object and initialized temporary |
| F-3d3200bf0b3e | Supplies first initial 64-bit quantity |
| F-beaa1252c720 | Supplies second initial 64-bit quantity |
| F-eef4045aa4cc | Converts explicit split-word values or receives unresolved operands |
| F-596e18a33aec | Combines two ordered 64-bit operands |
| F-c077d567b5f1 | Receives captured handle before component A’s conversion |
| F-3b2936fe6771 | Supplies first handle-associated 64-bit quantity |
| F-e833a6939895 | Supplies second handle-associated 64-bit quantity |
| F-b5685315bcf0 | Receives captured handle before component C’s conversion |
| F-8a693c103d1c | Builds the four-stage aggregate |
| F-11ed8242c40c | Receives candidate or aggregate and a newly read global pair |
| F-1b22d7e17791 | Performs three interleaved numeric operations |
| F-0eb164d8526b | Receives scratch and initialized temporary addresses; returns update argument |
| F-353caf38ed9a | Receives object, conversion result, and current field pair |
| F-282eebd7c413 | Receives temporary addresses at cleanup positions |

These roles are not complete dependency contracts. No dependency is assumed to preserve globals or to acquire or release ownership.

## Acceptance examples

These are conditional consequences of the static representation, not executed tests. They assume accessible storage and normal dependency returns.

1. **Mode selection.** Mode `1` produces five update calls in the documented order. Modes `0`, `2`, and `255` produce two. If the entry dependency changes the mode, selection uses the changed byte.

2. **Zero initial quantity.** In the two-update branch, a zero first quantity still permits the second supplier to run. The default global pair is then read. The conditional conversions and F-596e18a33aec are skipped; F-11ed8242c40c receives the captured default and a subsequently read comparison pair.

3. **Default read despite replacement.** If the first quantity is nonzero, the default pair is still read after both suppliers. The later combining result replaces it. Omitting that read does not preserve the displayed read sequence.

4. **Comparison-pair mutation.** If a conditional numeric helper changes the two-update comparison pair, F-11ed8242c40c receives the pair read after that helper, not an entry-time snapshot.

5. **Borrow propagation.** In the two-update branch, subtracting `0x0000000000000001` from `0x0000000100000000` produces `0x00000000FFFFFFFF` before conversion. Subtracting `2` from `1` produces `0xFFFFFFFFFFFFFFFF`; the converted interpretation remains dependency-defined.

6. **Seed timing.** If component D’s processing changes the five-update seed pair, the first aggregate call receives the changed pair. A seed snapshot taken before component construction would differ from the established sequence.

7. **Low-word selection despite full-width ordering.** Suppose F-3b2936fe6771 returns `0x0000000200000000` and F-e833a6939895 returns `0x0000000100000001`. The saved low-word condition is true despite the opposite full-width ordering. The selected value remains the aggregate; no conditional replacement reread occurs.

8. **Distinct pre-call and post-call values.** Suppose the saved condition is false, the replacement pair reads as `0x0000000000000010` before F-11ed8242c40c, and that dependency changes it to `0x0000000000000020`. The dependency receives the former value. Selection captures the latter, and the third interleaved F-1b22d7e17791 call receives that latter capture as its second operand.

9. **Later mutation does not replace the capture.** In the preceding example, if an update subsequently changes the replacement globals again, the third interleaved operation still uses the selected `0x0000000000000020` capture; no further read is shown.

10. **Late fields and cleanup.** If an earlier update changes a later cycle’s fields, the later cycle reads the changed fields. Normal completion makes three F-282eebd7c413 calls in the two-update branch and six in the five-update branch.

## Uncertainties

The following remain **documented_only**:

- Numeric global values, meanings, mutability constraints, and stability across calls.
- Ordering and atomicity of word reads within each global pair.
- Arithmetic semantics and representation of numeric-helper results.
- Effective operands of the two conversions without explicit arguments.
- Effects of F-11ed8242c40c beyond its explicit arguments.
- Fidelity of reconstructed condition-state behavior; the displayed selection nevertheless uses the saved low-word condition.
- Any mechanism linking numeric processing to updates without explicit numeric update arguments.
- Temporary layout, initialization extent, ownership, retained references, and cleanup semantics.
- Invalid-reference behavior, dependency failures, and nonlocal control transfers.
- Read order within individual object field pairs.
- Thread safety, reentrancy, and concurrent-mutation guarantees.

These unknowns do not justify treating global pairs as constants, collapsing separate reads, or assigning conventional arithmetic meanings to opaque dependencies.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent proof; the documented read points and conditional reread are supported by the supplied representation.

Established static evidence includes argument widths, object offsets, explicit call operands, call ordering, modulo-width subtraction with borrow, the saved unsigned low-word condition, numeric global-read placement, conditional post-call replacement capture, cleanup placement, and normal returns.

The supplied representation reports removed unreachable material and a transformation restart. Those notices limit claims about omitted material and reconstructed implicit behavior; they do not establish additional behavior.

Domain intent, hidden numeric data flow, dependency purity, and ownership remain unresolved. No implementation beyond the supplied representation was accessed. No runtime execution or tests were performed.
