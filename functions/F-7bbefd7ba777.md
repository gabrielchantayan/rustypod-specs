<!-- Trusted provenance: F-7bbefd7ba777; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7bbefd7ba777

## Purpose

The established behavior is an ordered sequence of unresolved dependency calls, 32-bit integer transformations, indirect memory accesses, global-storage updates, and a final call to address zero.

No application-level purpose is established. Repeated scaling and fixed offsets suggest structured state manipulation, but that is **inferred intent**, not evidence of allocation, serialization, dispatch, or resource management.

This contract describes the supplied static representation. It does not establish runtime reachability, address accessibility, or the accuracy of recovered dependency interfaces.

## Interface

### Declared inputs and result

| Input | Width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit consumption is shown. |
| Second argument | 32 bits | Passed to F-D01-01. |
| Third argument | Unsigned 32 bits | Logically shifted right by 23 and stored as a word. |

No return value is declared. Normal return requires all preceding operations, including the final address-zero call, to complete and return.

The absence of an explicit use of the first argument does not prove that dependencies cannot observe its incoming machine-level representation.

### Additional execution state

The representation also consumes:

- Two incoming unsigned 32-bit context values, called **context A** and **context B**.
- Incoming link-related state used to construct the first call target.
- An auxiliary opaque 32-bit word.
- An auxiliary signed 32-bit value captured for later calls.
- An auxiliary byte pointer used near completion.
- A 32-bit global destination, called **global word**.
- A separate global value captured as a signed address base.
- A three-byte global fragment supplying the high 24 bits of the final global-word assignment.

These are not established public parameters. Their initialization, provenance, ABI interpretation, and lifetime are **documented_only**.

### Width and arithmetic rules

A byte is 8 bits, a halfword is 16 bits, and a word is 32 bits. Each displayed wide result contains 64 bits, divided into low and high 32-bit halves.

Unsigned 32-bit arithmetic wraps modulo \(2^{32}\). Unsigned right shifts insert zeros; unsigned left shifts discard bits beyond bit 31. These rules apply to intermediate operations, not merely final assignments.

Signed multiplication, addition, and left shifts occur separately. Overflow and nonportable signed-shift cases are **documented_only**. Assigning a signed-shift result to an unsigned word does not retroactively make the shift unsigned.

A byte store retains the low 8 bits of its numerical input. Word stores write 32 bits; the final halfword store writes 16 zero bits.

Calculated addresses are addresses themselves, not locations from which an additional pointer is fetched unless a lookup is explicitly described. No bounds, null, alignment, or overflow checks are visible.

## Behavior

### Ordering notation

Each dependency call has an opaque call-site ID **F-Drr-cc**: `rr` identifies the phase below, and `cc` identifies the call’s position within that phase. The IDs identify call sites, not necessarily distinct implementations.

In the table:

- Each semicolon separates successive calls.
- “None” means no explicit arguments are displayed, not that the actual ABI has no other inputs.
- An arrow names a consumed result.
- “Forwarded result” refers to the immediately identified result in that call sequence.
- Repeated arguments occupy separate argument positions.
- Calls precede the direct effects listed for their phase.
- Direct effects occur in their listed order; a lookup read precedes its dependent write.
- “Signed ×4,” “signed ×16,” and “signed shift” preserve the signed qualification of the displayed arithmetic.

Dependencies may change accessible memory between direct effects. No preservation of globals, tables, or other storage is assumed.

### Ordered calls and memory effects

| Phase | Dependency calls, in order | Direct effects and retained state |
|---|---|---|
| 01 | Second argument; none → forwarded result; forwarded result; none; none; none; none → initial index. | Store third argument shifted right by 23 at **signed ×4(initial index) + 20**. Then read and retain **initial word** at **(context A >> 22) + 20**. |
| 02 | Signed ×4(initial index); none; none; none; none. | Retain **context region = context B >> 22** and **fragment = initial word >> 22**, both unsigned. Read a byte at **context B + 1**, using unsigned wraparound. Write fragment’s low byte at **lookup byte + 17**. |
| 03 | Three copies of the auxiliary opaque word, followed by context region; none; context region >> 23; none; none; none; none → second index. | Read a byte at **4 × fragment + 1**. Clear the byte at **lookup byte + 17**. Fragment is unsigned and at most 1023. |
| 04 | Second index signed-shifted left by 2; none; none; none; none → third index; two copies of third index signed-shifted left by 2; none → first retained word; first retained word; none. | Read the sign-selected byte described below, then clear the byte at **lookup byte + 17**. |
| 05 | Three copies of a newly captured signed auxiliary value; none → forwarded result; forwarded result and first retained word >> 23; none; none; none; none → fourth index. | Reread the sign-selected byte. Retain its unsigned value as **small base**, then clear **small base + 17**. Preserve the captured auxiliary value for phases 08 and 10. |
| 06 | Fourth index signed-shifted left by 2; none; none; none; none → fifth index; two copies of fifth index signed-shifted left by 2; none → second retained word. | Assign second retained word to global word. |
| 07 | Second retained word; none. | Read a word at **small base + 116** and write its low byte at **small base + 1**. |
| 08 | Captured auxiliary value signed-shifted left by 11; none → first 64-bit result; its low half interpreted as signed, its high half interpreted as signed, and second retained word >> 23; none; none; none; none → first copy index. | Form signed **first copy base = signed ×4(first copy index)**. Read a word at **first copy base + 116** and write its low byte at **first copy base + 1**. |
| 09 | First copy base; none; none; none; none → retained index. | Signed-shift retained index left by 2, retaining the result in unsigned **retained scaled word**. Reread the word at **first copy base + 116** and write its low byte at **first copy base + 1**. |
| 10 | Captured auxiliary value signed-shifted left by 12, retained scaled word twice, and retained index signed-shifted left by 18; none → forwarded result; forwarded result; none → forwarded result; forwarded result; none; none; none; none → second copy index. | Form signed **second copy base = signed ×4(second copy index)**. Read and retain unsigned **copied word** at **second copy base + 116**. Write its low byte at **second copy base + 1**. |
| 11 | Second copy base; none; none; none; none → wide-stride index. | Store copied word at **signed ×16(wide-stride index) + 20**. |
| 12 | **(retained scaled word >> 2) << 16**, both shifts unsigned; none → forwarded result; forwarded result; none → forwarded result; forwarded result; none; none; none; none → narrow-stride index. | Store copied word at **signed ×4(narrow-stride index) + 20**. |
| 13 | Signed ×4(narrow-stride index); none; none; none; none → link index. | Form signed **link value** by shifting link index left by 2. Store its 32-bit representation at **(retained scaled word AND 0xfffffffc) + 20**, with unsigned address addition. |
| 14 | Two copies of link value; none → forwarded result; forwarded result; none → forwarded result; forwarded result; none; none; none; none → clear index. | Store a zero word at **signed ×4(clear index) + 20**. |
| 15 | Signed ×4(clear index); none; none; none; none → second 64-bit result. | Interpret both halves as signed 32-bit values. Form **pair link** by signed-shifting the low half left by 2. Store pair link at **signed ×4(high half) + 20**. Then read the separate global base and retain its signed value as **captured base**. |
| 16 | Pair link, the preceding high half, pair link, pair link; none; none; none; none → repeat-link index. | Store pair link at **signed ×4(repeat-link index) + 20**. Then read and retain unsigned **late word** at **captured base + 20**, whose displayed address addition is signed. |
| 17 | Signed ×4(repeat-link index); none; none; none; none → third 64-bit result. | Retain unsigned **final low word** and the high-half bit pattern. Read a byte at **(copied word >> 23) + 1**. Write late word’s low byte at **lookup byte + 17**. |
| 18 | Three copies of the third result’s high half, followed by **final low word << 2**, unsigned; none → forwarded result; forwarded result; none; none; none; none → transient index. | Read a byte at **((final low word AND 0x3fffffff) >> 21) + 1**. Write the low byte of **late word >> 22** at **lookup byte + 17**. |
| 19 | Transient index signed-shifted left by 2; none; none; none; none → final pair index; two copies of final pair index signed-shifted left by 2; none; none → forwarded result; forwarded result; none → final byte. | Write zero through the auxiliary byte pointer. Then assign global word with the three-byte fragment in its high 24 bits and final byte in its low 8 bits. Finally, store a zero halfword at **(final low word AND 0x3fffffff) >> 7**. |
| 20 | F-D20-01: address-zero call with three explicit zero arguments. | Return without a value only if that call returns. |

### Phase 19 call-site distinction

Phase 19 contains ten calls. Its final four calls are distinct:

| Call site | Explicit inputs | Consumed result | Target-offset classification |
|---|---|---|---|
| F-D19-07 | None displayed | None retained | Usual offset pattern |
| F-D19-08 | None displayed | Forwarded 32-bit word | Unusual offset |
| F-D19-09 | The word returned by F-D19-08 | None retained | Usual offset pattern |
| F-D19-10 | None displayed | Final byte | Usual offset pattern |

Thus the unusually offset target belongs to **F-D19-08**, not F-D19-09. The forwarding relationship does not merge those call sites.

### Address selection and width consequences

The sign-selection expression uses a signed interpretation of context B and a signed right shift by 31. Under arithmetic signed-right-shift behavior, phases 04 and 05 read absolute address 1 when the sign bit is clear and absolute address 2 when it is set. These are not offsets added to context B. The underlying signed-shift interpretation remains **documented_only**.

The third argument’s extracted value is 0–511. Context region and fragment are each 0–1023. Consequently, F-D03-03 receives zero: context region shifted right by another 23 bits has no remaining set bits.

Unsigned address addition in phase 02 wraps before the access. Context B equal to `0xffffffff` therefore selects lookup address zero.

The phase 13 mask does not prevent addition overflow. Retained scaled word `0xfffffffc` produces destination address 16 after adding 20 modulo \(2^{32}\).

The phase 12 argument can lose bits during its unsigned left shift. Retained scaled word `0x00040000` produces argument zero. Likewise, final low word `0xc0000001` produces fourth argument 4 at F-D18-01.

Established address bounds include:

- Phase 03 lookup: 1–4093.
- Phase 07 word read: 116–371.
- Phases 17 and 18 lookups: 1–512.
- Any byte destination formed as an unsigned lookup byte plus 17: 17–272.
- Final halfword destination: 0–8,388,607, without guaranteed two-byte alignment.

These bounds do not establish accessibility.

## Side effects

### Observable memory ordering

The direct effects include scaled word stores, lookup-dependent byte writes, word-to-byte truncating transfers, two explicit assignments to global word, and a final halfword clear.

The following ordering distinctions are contractual:

- Phase 01’s word store precedes its initial-word read.
- Phase 05 rereads the sign-selected byte after intervening calls; phase 04’s lookup result is not reused.
- Phase 09 rereads the word used by phase 08 after intervening calls.
- Phase 10 retains the complete copied word before its byte store. Phases 11 and 12 use that retained word, not a refreshed source read.
- Captured base is read after phase 15’s store and before phase 16’s calls. The later access uses that captured value.
- The auxiliary-pointer clear precedes the final global-word assignment, which precedes the halfword clear and address-zero call.

Dependencies may overwrite storage between these operations. Neither global-word assignment is guaranteed to remain the final value: later direct writes may alias it, and subsequent dependencies may change it.

### Aliasing, layout, and ownership

No disjointness is established among calculated addresses, global storage, narrower global views, dependency-visible memory, or the auxiliary pointer. Earlier writes may therefore affect later reads.

A word-to-byte transfer truncates the loaded numerical value. It is not independently specified as copying a particular physical byte lane.

The final global assignment places the fragment numerically in bits 31–8 and the final byte in bits 7–0. Endianness, the fragment’s physical byte ordering, its precise load granularity, and overlap with other storage are **documented_only**. In particular, no assumption excludes overlap between the auxiliary-pointer clear and the fragment subsequently used.

No allocation, deallocation, ownership transfer, synchronization, atomicity, or volatile-access guarantee is established. Dependencies are not assumed to preserve globals or acquire or release memory.

### Errors and partial execution

No explicit validation, error result, rollback, or cleanup path is visible.

An inaccessible address, invalid target, unsupported alignment, or address-zero call may prevent completion. Exact failure behavior is platform-dependent and **documented_only**. No visible mechanism reverses writes already completed.

## Dependencies

All F-Drr-cc call sites are unresolved.

F-D01-01 constructs its target from incoming link-related state. Subsequent targets use fixed anchors with offsets; several offset patterns occur. The usual nearby target offset is `0xffe`.

Within phase 19, **F-D19-08 uses offset `0xb0`**, returns the forwarded word, and is followed by **F-D19-09**, which consumes that word and uses the usual `0xffe` offset. F-D19-10 then supplies the final byte. F-D20-01 directly targets address zero.

Target-construction evidence does not establish semantic identity across sites.

The contract establishes displayed argument flow and result consumption, but not:

- Complete signatures, calling conventions, or hidden machine-level inputs.
- Whether displayed wide results are scalar returns or reconstructed register state.
- Whether similarly constructed targets identify the same implementation.
- Normal return, exception behavior, or memory preservation.
- Allocation, lookup, dispatch, or release semantics.

No dependency implementation was supplied.

## Acceptance examples

These are conditional static examples, not observed executions. Each assumes execution reaches the operation, required calls and accesses succeed, and relevant signed expressions have an applicable defined interpretation.

1. **Input extraction.** Third argument `0xff800000` and initial index 3 cause phase 01 to store word 511 at address 32. A third argument below `0x00800000` produces zero.

2. **Wrapped lookup.** With context B `0xffffffff`, phase 02 reads address zero. If the lookup byte is 7 and fragment is `0x2af`, it writes byte `0xaf` at address 24.

3. **Zero argument.** For every unsigned 32-bit context B, F-D03-03 receives explicit argument zero. A nonzero argument contradicts the displayed unsigned shifts.

4. **Sign-selected reread.** Under arithmetic signed-right-shift behavior, context B `0x80000000` selects address 2. If phase 04 reads 9 and intervening calls change that byte to 12, phase 04 clears address 26, while phase 05 clears address 29 and retains small base 12.

5. **Repeated source read.** If phase 08 reads word `0x000000aa`, it writes byte `0xaa`. If intervening calls change the source word to `0x000000bb`, phase 09 writes `0xbb`.

6. **Full-word retention.** If phase 10 reads copied word `0x123456ef`, its byte store writes `0xef`. Phases 11 and 12 still store `0x123456ef` even if the source location subsequently changes.

7. **Unsigned shift truncation.** Retained scaled word `0x00040000` gives F-D12-01 argument zero. Final low word `0xc0000001` gives F-D18-01 fourth argument 4.

8. **Wrapped destination.** Retained scaled word `0xfffffffc` makes phase 13 store link value at address 16.

9. **Captured base.** If phase 15 captures base 4096 and later dependencies change its source global to 8192, phase 16 still reads late word at address 4116.

10. **Final forwarding identity.** If F-D19-08 returns word `0x12345678`, F-D19-09 receives that word explicitly. The unusually offset target is the producing call, F-D19-08, not the consuming call.

11. **Final layout and alignment.** If the fragment has numerical value `0x123456` and final byte is `0xab`, the explicit global assignment is `0x123456ab`. With final low word `0x00000180`, the subsequent halfword clear targets address 3; no alignment correction is shown.

12. **Completion boundary.** Reaching F-D20-01 does not establish normal return. Address zero must be callable in the execution environment, and that call must return.

## Uncertainties

The following remain **documented_only**:

- Architecture, endianness, alignment rules, address mapping, and executable-target validity.
- Completeness and correctness of the displayed ABI.
- Signed overflow, signed-left-shift cases outside portable semantics, and signed-right-shift interpretation.
- Auxiliary-value and auxiliary-pointer provenance and lifetime.
- Physical overlap among globals, narrower storage views, and computed destinations.
- Exact fragment-read granularity and evaluation details.
- Dependency implementations, side effects, and termination behavior.
- Application-level relationships among indices, tables, and stored words.

These uncertainties do not replace the established unsigned-width rules with unbounded arithmetic. In particular, phase 02’s address addition, phase 13’s masked addition, and the argument shifts in phases 12 and 18 retain their explicit 32-bit behavior.

No complete valid-input domain, structure size, ownership model, or runtime reachability guarantee is established.

## Evidence

**Established static evidence:** The supplied representation exposes three declared 32-bit inputs, additional incoming context, unresolved indirect calls, byte/halfword/word accesses, signed and unsigned transformations, two assignments to one global word, and a final address-zero call with three zero arguments.

Phase 19’s call sequence specifically establishes that its eighth call produces the forwarded word using the unusual target offset, its ninth consumes that word using the usual offset, and its tenth produces the final byte.

**Inferred intent:** Repeated scaling and fixed offsets may reflect structured state manipulation. This supplies no additional behavioral guarantee.

**Unknowns:** Dependency bodies, runtime memory validity, actual ABI behavior, and application semantics were not supplied.

No runtime tests or execution traces have been performed.
