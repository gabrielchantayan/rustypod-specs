<!-- Trusted provenance: F-5b10e4bc7b2b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5b10e4bc7b2b

## Purpose

F-5b10e4bc7b2b describes a fixed sequence of indirect calls, value transformations, and memory accesses. It forwards dependency results, derives addresses from selected results and memory contents, performs byte and word updates, clears memory, and finally attempts a call to address zero.

Only one global destination is explicitly identified: **global S**, assigned twice. **Global N** and a separate **24-bit global fragment** are explicitly read. Computed-address writes could alias these or other globals, but no second global destination is established.

No application-level purpose is established. Repeated scaling and stores at common offsets might represent encoded references or fields, but neither a particular data structure nor allocation, cleanup, or supervisor-service semantics follows from the evidence.

This contract separates established static behavior from inferred intent and **documented_only** unknowns. All execution descriptions are conditional on the supplied representation faithfully describing executable behavior. No runtime tests have been performed.

## Interface

### Explicit inputs and result

| Position | Represented type | Visible use |
|---|---|---|
| First | Opaque 32-bit value | No explicit use. |
| Second | Opaque 32-bit value | No explicit use. |
| Third | Signed 32-bit integer | Left-shifted by 11 and 12 bits for dependency arguments. |
| Fourth | Unsigned 32-bit integer | Logically right-shifted by 23 bits for a dependency argument. |

No explicit return value is represented. The visible return is reached only if all preceding operations complete, including the final address-zero call.

The first two inputs have no explicit uses, but their possible visibility through ambient machine state or an unresolved calling convention is **documented_only**.

### Implicit inputs and memory requirements

Additional inputs include:

- A link-related machine value used to derive the first call target.
- An incoming address for an initial byte read at offset 1.
- An incoming unsigned 32-bit value supplying bits for an initial byte store.
- Global S, a 32-bit destination.
- Global N, containing a represented 32-bit address-bearing value.
- A 24-bit global fragment read during the final assignment to global S.
- A pointer available after F-C097, used for a byte clear.
- Dependency results and any dependency-induced changes to accessible memory or machine state.

These are execution requirements, not additional declared arguments. Their initialization, validity, and ABI treatment are unresolved.

Successful execution requires every reached call target and memory access to be usable under the actual machine environment. No validation of these conditions is visible.

### Widths and layout

A **byte**, **halfword**, and **word** contain 8, 16, and 32 bits respectively. A 64-bit result has a **low half** containing bits 0–31 and a **high half** containing bits 32–63. Offsets are byte displacements.

“Scaled by four” denotes the represented multiplication by four or left shift by two; these operations do not all have the same represented signedness. Unsigned 32-bit shifts discard bits beyond bit 31. Signed shifts, signed multiplication, and signed address arithmetic require the qualifications in Uncertainties.

A word-to-byte transfer stores the word value’s low eight bits. Whether those bits were originally at the word’s lowest address depends on endianness.

No object layout is established beyond the access widths and offsets below.

## Behavior

### Ordering and dependency notation

F-C001 through F-C098 are opaque identifiers for invocation positions. Different identifiers do not guarantee different target functions. “No explicit arguments” means none are represented; it does not exclude implicit ABI inputs.

The following stages execute in order, provided earlier calls return and earlier accesses complete. There are no visible branches, loops, validation paths, or recovery handlers. Calls whose results are not consumed remain observable operations.

### Initial extraction and first global assignment

F-C001 is invoked without explicit arguments at a target derived by adding 0xFEC to the incoming link-related value. F-C002 then supplies a 32-bit result to F-C003. F-C003 receives that result first and the fourth input logically shifted right by 23 second; the second argument ranges from 0 through 511.

F-C004, F-C005, and F-C006 follow without explicit arguments. F-C007 then supplies a represented signed 32-bit result.

Before F-C008, one unsigned byte is read from the incoming address plus 1. Its zero-extended value becomes the **initial base**, ranging from 0 through 255. A byte is stored at initial base plus 0x11, using bits 22–29 of the incoming unsigned machine value. F-C008 receives the F-C007 result shifted left by two.

F-C009, F-C010, and F-C011 precede F-C012. F-C013 receives two equal arguments, each the F-C012 result shifted left by two.

F-C014 returns an unsigned 32-bit value. That value is assigned to global S and then passed to F-C015. After F-C016, a fresh word is read at initial base plus 0x74, and its low byte is stored at initial base plus 1.

The initial base is retained across these calls; its source byte is not reread. The word read occurs after F-C016, so intervening dependency writes may affect it.

### Third-input transformations and repeated reads

F-C017 receives the third input shifted left by 11. F-C018 supplies a 64-bit result. F-C019 receives, in order, its low half, its high half, and the retained F-C014 result logically shifted right by 23. The two halves are represented as 32-bit integer arguments.

The F-C019 argument uses the retained F-C014 result, not the current contents of global S.

After F-C020, F-C021, and F-C022, F-C023 supplies a signed 32-bit result. Multiplication by four forms the **first scaled base**. A word is read at that base plus 0x74; its low byte is stored at the same base plus 1. F-C024 receives the first scaled base.

F-C025, F-C026, and F-C027 precede F-C028. Its signed 32-bit result is retained, and its left shift by two is retained as the unsigned **shared scaled value**.

A second word read at first scaled base plus 0x74 supplies another low-byte store at first scaled base plus 1. This is a separate read, not reuse of the first word.

F-C029 then receives four arguments:

1. The third input shifted left by 12.
2. The shared scaled value.
3. The shared scaled value again.
4. The F-C028 result shifted left by 18.

### Captured word and its distribution

F-C030 supplies a 32-bit result to F-C031. F-C032 similarly supplies a result to F-C033. F-C034, F-C035, and F-C036 follow without explicit arguments.

F-C037 supplies a signed 32-bit result whose multiplication by four forms the **capture base**. A fresh unsigned word read at capture base plus 0x74 becomes the **captured word**. Its low byte is stored at capture base plus 1, then F-C038 receives the capture base.

F-C039, F-C040, and F-C041 precede F-C042. The captured word is stored as a full word at sixteen times the F-C042 result plus 0x14.

F-C043 receives the shared scaled value logically shifted right by two, then shifted left by 16. The staged computation is established. Under 32-bit bit-vector semantics, it equals the original F-C028 result shifted left by 16: bits lost during the initial scaling would also be discarded by the final shift. This equality does not resolve the signed-language validity of the original shift.

### Further word stores

F-C044 supplies its result to F-C045, and F-C046 supplies its result to F-C047. F-C048, F-C049, and F-C050 follow.

F-C051 supplies a signed 32-bit result. The captured word is stored at four times that result plus 0x14, after which F-C052 receives the fourfold result.

After F-C053, F-C054, and F-C055, F-C056 supplies another signed 32-bit result. Its left shift by two is stored as a word at the shared scaled value masked with 0xFFFFFFFC, plus 0x14. F-C057 receives two copies of the shifted F-C056 result.

The explicit mask is retained in the address formula, although the shared scaled value already has two zero low bits under 32-bit shift semantics. The write might resemble a link update; that interpretation is not an established layout or ownership rule.

### Zero store and selected-word acquisition

F-C058 supplies its result to F-C059, and F-C060 supplies its result to F-C061. F-C062, F-C063, and F-C064 follow.

F-C065 supplies a signed 32-bit result. A zero word is stored at four times that result plus 0x14. F-C066 then receives the fourfold result.

F-C067, F-C068, and F-C069 precede F-C070, which supplies a 64-bit result. Its low half, interpreted as a signed 32-bit value and shifted left by two, becomes the **stored scaled value**. That value is stored as a word at four times the signed high half plus 0x14.

Global N is read next, retaining its address-bearing value. F-C071 receives, in order:

1. The stored scaled value.
2. The high half of the F-C070 result.
3. The stored scaled value.
4. The stored scaled value.

F-C072, F-C073, and F-C074 precede F-C075. The stored scaled value is written at four times the F-C075 result plus 0x14.

Only after that store is a word read at the retained global-N address plus 0x14. This becomes the **selected word**. F-C076 then receives four times the F-C075 result.

Thus, global N supplies a saved address before F-C071, while the selected word is fetched later. Changes to global N do not redirect this read; changes to memory at the saved address may affect it, including changes caused by an overlapping preceding store.

### Nested byte-address selection

F-C077, F-C078, and F-C079 precede F-C080, which supplies a retained 64-bit result.

The captured word is logically shifted right by 23 and increased by 1 to form a byte-read address. The unsigned byte read there is zero-extended and increased by 0x11 to form a byte-write address. The selected word’s low byte is written there.

F-C081 receives the F-C080 high half three times, followed by its unsigned low half shifted left by two.

F-C082 supplies its result to F-C083. F-C084, F-C085, and F-C086 precede F-C087.

The retained F-C080 low half is masked with 0x3FFFFFFF, logically shifted right by 21, and increased by 1 to form another byte-read address. The unsigned byte read there, increased by 0x11, selects the next byte-write address. That store uses bits 22–29 of the selected word. F-C088 then receives the F-C087 result shifted left by two.

Both intermediate byte-read addresses range from 1 through 512. Both resulting byte-write addresses range from 0x11 through 0x110. These arithmetic bounds do not establish valid mappings or allocated storage.

### Final clearing, global composition, and terminal call

F-C089, F-C090, and F-C091 precede F-C092. F-C093 receives two copies of the F-C092 result shifted left by two. F-C094 follows, then F-C095 supplies a 32-bit result to F-C096.

F-C097 supplies an 8-bit result. A zero byte is written through the pointer available after that call. The pointer is not established as an ordinary declared return value of F-C097.

Global S is then assigned a 32-bit composition: the F-C097 result occupies bits 0–7, and the separately read 24-bit global fragment occupies bits 8–31. The fragment is used at this point, after the byte clear; its earlier value is not retained by the visible body.

A zero halfword is next stored at the address formed by masking the retained F-C080 low half with 0x3FFFFFFF and logically shifting right by seven. This address ranges from zero through 0x007FFFFF and need not be aligned. No alignment correction is visible.

Finally, F-C098 attempts a call to address zero with three explicit zero arguments. A return without an explicit result follows only if that call returns.

## Side effects

The body contains **17 explicit stores**:

| Width | Count | Destinations |
|---|---:|---|
| Byte | 8 | Initial computed destination; initial-base copy; two first-scaled-base copies; capture-base copy; two nested destinations; final pointer clear. |
| Word | 8 | Two assignments to global S; two captured-word destinations; shifted F-C056 destination; zero-word destination; two stored-scaled-value destinations. |
| Halfword | 1 | Final masked-and-shifted address. |

These are operation counts, not counts of distinct destinations. They exclude all dependency effects.

Explicit reads include three byte-based address selections, word reads at offset 0x74, global N, the selected-word read through its saved address, and the 24-bit fragment. Global S is explicitly written twice; neither global N nor the fragment is explicitly assigned.

Aliasing may connect any computed destination with earlier or later read locations, global storage, or other destinations. In particular, the final pointer clear can affect the subsequently read fragment if their storage overlaps. The final halfword clear occurs after the global-S composition and can overwrite overlapping storage.

No explicit allocation, release, ownership transfer, or lifetime extension is shown. Retaining a pointer or address across dependencies does not prove its referent remains valid.

There is no visible status return, bounds check, null check, rollback, or cleanup path. A faulting access or nonreturning dependency prevents later operations; earlier effects are not visibly undone. Atomicity, synchronization, volatile behavior, and concurrent-observer guarantees are unresolved.

## Dependencies

The identifiers F-C001–F-C098 name call sites only. No dependency implementation or semantic identity is supplied.

Most targets are represented by fixed address calculations. F-C001 additionally depends on incoming machine state; F-C098 explicitly targets zero. These facts do not establish executable mappings or successful calls.

Consumed result widths are:

- **64 bits:** F-C018, F-C070, F-C080.
- **8 bits:** F-C097.
- **32 bits:** Other consumed results.

F-C014 is explicitly unsigned. Several results used for scaling are represented as signed 32-bit integers; their arithmetic qualifications must not be replaced with unconditional wraparound assumptions.

Every dependency is **documented_only** regarding implicit arguments, register preservation, global changes, allocation, release, invalidation, exceptions, termination, reentrancy, and synchronization. No dependency is assumed to preserve globals or own or release memory. Retained-value descriptions state the visible dataflow, not an independently verified ABI guarantee.

## Acceptance examples

These are conditional static examples, not executed tests. Each assumes execution reaches the relevant operation.

1. **Input extraction.** A fourth input of 0xFFFFFFFF supplies 511 as F-C003’s second argument.

2. **Initial byte store.** If the initial byte read yields 0x20 and the incoming unsigned machine value is 0xFFFFFFFF, the store targets 0x31 and writes 0xFF.

3. **Repeated reads remain fresh.** If the first first-scaled-base word read yields 0x11223344 and an intervening dependency changes that location to 0x55667788, the corresponding byte stores write 0x44 and 0x88, respectively.

4. **Retained result, not global reread.** If F-C014 returns 0x80000000, global S is first assigned that value. F-C019’s third argument is 256 even if a dependency subsequently changes global S.

5. **Captured-value reuse.** A captured word of 0xA1B2C3D4 produces an immediate byte store of 0xD4 and later full-word stores of 0xA1B2C3D4, without rereading the capture location.

6. **Staged shift equivalence.** Under 32-bit bit-vector semantics, an F-C028 result with bit pattern 0xC0000001 produces shared scaled value 4 and F-C043 argument 0x00010000. Shifting the original bit pattern left by 16 also yields 0x00010000. This example does not establish portable signed-shift behavior.

7. **Unsigned nested selection.** A nested byte read of 0xFF selects destination 0x110, not a destination based on treating the byte as −1.

8. **Saved address, fresh contents.** If global N supplies address A before F-C071 and later changes to B, the selected word is still read from A plus 0x14. Its value reflects changes to that location before the read.

9. **Final composition and overlap.** If the fragment read at composition yields 0x123456 and F-C097 returns 0xAB, the global-S assignment writes 0x123456AB. A later overlapping halfword clear may change the final memory contents.

10. **Terminal call.** Reaching F-C098 entails an attempted target of zero with three zero arguments. Neither successful invocation nor a particular fault outcome is established.

## Uncertainties

- **documented_only — execution fidelity:** The unusual indirect targets and ambient machine inputs lack instruction-level or runtime confirmation.
- **documented_only — signed arithmetic:** Negative signed left operands, overflowing signed shifts or multiplications, and overflowing signed address additions do not establish portable language-level behavior. Bit-vector examples are conditional, not universal execution guarantees.
- **documented_only — memory environment:** Endianness, address mapping, pointer interpretation, alignment rules, permissions, and device-memory behavior are unspecified.
- **documented_only — hidden pointer:** The origin, validity, and lifetime of the pointer used after F-C097 are unresolved.
- **documented_only — overlapping storage:** Global storage is indicated as potentially overlapping smaller symbols, but exact layout and alias relationships are not established. Independence must not be assumed.
- **documented_only — completion and errors:** No fault model or dependency error contract is supplied. Even the final address-zero call has no established outcome.
- **Inferred intent only:** Fourfold scaling and repeated offset-0x14 stores may encode references or fields. No object model, ownership relation, or application purpose is established.

## Evidence

The supplied static body is the sole behavioral evidence. It establishes the represented call order, consumed-result widths, argument transformations, retained-value reuse, memory-access formulas, and final address-zero call.

The explicit assignments support 17 stores: eight byte stores, eight word stores, and one halfword store. Only global S is an explicitly identified global destination, assigned twice. Global N and the 24-bit fragment are read sources.

The F-C043 equivalence follows from 32-bit bit-vector arithmetic: its final left shift discards the same original high bits removed by the earlier scaling and logical right shift. It does not remove the signed-arithmetic uncertainty.

No dependency implementations, independent ABI definition, memory map, execution trace, or runtime tests were supplied. No runtime tests have been performed.
