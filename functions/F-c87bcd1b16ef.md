<!-- Trusted provenance: F-c87bcd1b16ef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c87bcd1b16ef

## Purpose

The supplied static representation describes a straight-line sequence of indirect calls, selector-directed byte writes, and computed 32-bit writes. Its final represented action is an invalid-instruction termination marker; no normal return is shown.

No application-level purpose is established. Initialization, allocation, device configuration, and serialization are possible interpretations, not supported contracts. The addressed storage and indirect callees have no established application-level meaning.

The behavior below is a contract for the supplied representation, conditional on its recovered control flow and widths being accurate. Warnings about bad instruction data and overlapping global symbols limit that confidence.

## Interface

### Explicit inputs

Four explicit argument positions are represented:

| Position | Width | Directly represented use |
|---|---:|---|
| First | 32 bits | None. |
| Second | 32 bits | Repeated in the first three argument positions of F-D001. |
| Third | 32 bits | None. |
| Fourth | 32 bits, represented as signed | Shifted left by two for F-D001’s fourth argument. |

The absence of a direct use does not prove that an input is irrelevant at machine level. Indirect callees may observe state outside their displayed arguments.

No valid range is established for the fourth argument. Its signed representation does not establish portable semantics for negative left shifts or overflowing shifts.

### Ambient inputs

The representation also depends on:

- **Base word:** an unsigned 32-bit incoming value used in selector addresses and logical right shifts.
- **Payload word:** an unsigned 32-bit incoming value supplying the first two stored bytes.
- **Incoming control word:** an incoming value used to derive F-D001’s target.
- **Post-call register word:** a represented 32-bit register-derived value consumed after F-D027.

The last value is not established as a formal return value or as either half of the earlier 64-bit result. The four explicit arguments alone are insufficient to determine the represented behavior.

### Outputs and termination

No ordinary return value is shown. Completion of the displayed sequence reaches an invalid-instruction termination marker.

Whether this produces a hardware exception, process termination, another control transfer, or merely reflects failed static recovery is **documented_only**.

### Widths and notation

For this contract:

- **Reduced base** means the base word logically shifted right by three: unsigned division by eight.
- **Derived selector base** means reduced base multiplied by `0x80000`.
- **Fourfold**, **eightfold**, and **sixteenfold** mean multiplication by four, eight, and sixteen.
- **Low byte** means the least significant eight bits.
- A **selector-directed byte write** reads an unsigned byte at a stated lookup address and writes one byte at `0x11` plus that selector’s value.

Ordinary consumed call results are represented as 32-bit words. F-D026 supplies a represented 64-bit result.

Where modular arithmetic is explicitly assumed, each 32-bit operation retains its low 32 bits. That assumption supports the conditional arithmetic examples; it does not resolve signed source-language overflow, pointer validity, or alignment rules. Byte truncation retains eight bits regardless of the signed interpretation of an intermediate byte.

## Behavior

### Ordering and dependency notation

F-D001 through F-D096 are opaque labels for the 96 indirect-call occurrences, in execution order. They are not resolved callee identities. Different labels may reach the same target.

“No displayed arguments” means only that the representation supplies no explicit arguments at that occurrence. It does not establish an argument-free machine interface.

Every continuation below requires preceding accesses to complete and preceding calls to return to the displayed continuation. No conditional branch, retry, error check, or recovery path is represented.

### 1. Initial selector writes and call chain

Before F-D001, the function reads a selector byte at base word plus one. It writes the payload word’s low byte at `0x11` plus that selector.

Calls then proceed as follows:

- F-D001 receives the second explicit argument three times, followed by the fourth explicit argument shifted left by two.
- F-D002 has no displayed arguments and supplies a word passed to F-D003.
- F-D004 through F-D006 have no displayed arguments.
- F-D007 has no displayed arguments and supplies a word.

After F-D007, the function reads the selector at base word plus one again. It writes the low byte of the payload word logically shifted right by 22 at the newly selected destination. This byte contains original payload bits 22 through 29.

The second selector is a fresh read, not reuse of the first. Intervening callees may have changed selector storage.

F-D008 receives the F-D007 result shifted left by two. F-D009 through F-D012 then execute with no displayed arguments.

### 2. Retained-selector clearing

Using the selector captured after F-D007, without reading it again, the function writes:

1. A 32-bit zero at `0x44` plus that selector.
2. A 32-bit zero at absolute address `0x24`.

F-D013 then receives zero and `0x98`.

The selector remains the previously captured value for this clearing operation. Changes to its original storage during F-D008 through F-D012 do not change the represented destination.

### 3. Forwarding and result-indexed clearing

F-D014 supplies a word passed to F-D015. F-D016 supplies a word passed to F-D017. F-D018 through F-D020 have no displayed arguments. F-D021 then supplies a word.

The function writes a 32-bit zero at `0x44` plus fourfold the F-D021 result, followed by a 32-bit zero at absolute address `0x24`.

F-D022 receives fourfold the F-D021 result. F-D023 through F-D025 execute with no displayed arguments.

### 4. Paired result and post-call register word

F-D026 supplies a represented 64-bit result.

F-D027 receives its low 32-bit half shifted left by two. After that call returns, the function writes a 32-bit zero at `0x44` plus twice the previously captured high 32-bit half. The high half is represented as converted to a signed 32-bit value before multiplication. A 32-bit zero write at absolute address `0x24` follows.

F-D028 receives three displayed arguments:

1. Zero.
2. The post-call register word observed after F-D027.
3. That register word shifted left by two.

Neither half of F-D026’s result determines the register word from the available evidence.

### 5. Base-derived argument and further zero stores

F-D029 has no displayed arguments. F-D030 receives reduced base. F-D031 through F-D033 have no displayed arguments. F-D034 supplies a word.

The function writes a 32-bit zero at `0x44` plus fourfold that result, then a 32-bit zero at absolute address `0x24`.

F-D035 receives fourfold the F-D034 result. F-D036 through F-D038 have no displayed arguments. F-D039 supplies a word.

F-D040 receives the F-D039 result shifted left by two and supplies another word. A 32-bit zero is written at `0x14` plus fourfold the F-D040 result.

F-D041 receives the F-D040 result in its first three displayed argument positions and reduced base in its fourth.

### 6. Zeroing and first computed-value store

F-D042 through F-D044 have no displayed arguments. F-D045 supplies a word.

The function writes a 32-bit zero at `0x14` plus fourfold that result. F-D046 receives the same fourfold value.

F-D047 through F-D049 have no displayed arguments. F-D050 supplies a word.

The function retains fourfold the F-D050 result as a 32-bit value and writes that value at `0x14` plus sixteenfold the F-D050 result. F-D051 receives the retained fourfold value.

The destination scaling is sixteen; the stored-value scaling is four. Neither the result nor the stored value is required to be nonzero.

### 7. Retained-value reuse and another scaled store

F-D052 through F-D054 have no displayed arguments. F-D055 supplies a word.

The function writes the retained fourfold value from F-D050 at `0x14` plus fourfold the F-D055 result. The stored value is not recomputed from F-D055.

F-D056 receives fourfold the F-D055 result.

F-D057 through F-D059 have no displayed arguments. F-D060 supplies a word.

The function writes fourfold the F-D060 result at `0x14` plus sixteenfold that same result. F-D061 receives two arguments: fourfold and sixteenfold the F-D060 result, in that order.

### 8. Base-derived value storage

F-D062 supplies a word. F-D063 receives that word followed by the base word logically shifted right by six.

F-D064 through F-D066 have no displayed arguments. F-D067 supplies a word.

The function writes reduced base as a 32-bit value at `0x14` plus fourfold the F-D067 result. F-D068 receives that fourfold result.

F-D069 through F-D071 have no displayed arguments. F-D072 supplies an unsigned 32-bit word.

### 9. First derived-selector byte pair

The function computes derived selector base, reads a selector byte at that base plus one, and writes the low byte of fourfold the F-D072 result at `0x11` plus the selector.

F-D073 receives the full 32-bit shifted result, not merely the stored byte.

F-D074 supplies a word passed to F-D075. F-D076 through F-D078 have no displayed arguments. F-D079 supplies a word.

The function reads a fresh selector at derived selector base plus one. It writes the low byte of the F-D072 result masked with `0x3fffffff` and logically shifted right by 20. The stored byte selects original result bits 20 through 27; the mask does not change those eight bits.

F-D080 receives the F-D079 result shifted left by two.

### 10. Second derived-result byte pair and termination

F-D081 through F-D083 have no displayed arguments. F-D084 supplies an unsigned 32-bit word.

The function reads a selector at derived selector base plus one, retains it, and writes the low byte of fourfold the F-D084 result at `0x11` plus that selector.

F-D085 receives the retained selector, zero-extended to 32 bits. It does not receive the F-D084 result or the byte just stored.

F-D086 supplies a word passed to F-D087. F-D088 through F-D090 have no displayed arguments. F-D091 supplies a word.

The final selector read uses eightfold the F-D084 result plus one—not derived selector base plus one. The function writes the low byte of the masked F-D084 result logically shifted right by 20 at `0x11` plus this new selector.

The two byte writes derived from F-D084 therefore need not share a selector lookup address or a destination.

F-D092 receives the F-D091 result shifted left by two. F-D093 through F-D096 have no displayed arguments. The invalid-instruction termination marker follows.

## Side effects

### Direct memory effects

A complete traversal of the represented path contains exactly **six unsigned-byte selector reads and six selector-directed byte writes**:

| Position | Selector lookup | Stored byte |
|---|---|---|
| Before F-D001 | Base word plus one | Payload low byte |
| After F-D007 | Base word plus one, freshly read | Payload bits 22–29 |
| After F-D072 | Derived selector base plus one | Low byte of fourfold F-D072 result |
| After F-D079 | Derived selector base plus one, freshly read | F-D072 result bits 20–27 |
| After F-D084 | Derived selector base plus one, freshly read | Low byte of fourfold F-D084 result |
| After F-D091 | Eightfold F-D084 result plus one | F-D084 result bits 20–27 |

The path also contains fourteen direct 32-bit writes:

- Four zero writes to absolute address `0x24`.
- Four zero writes to computed destinations based at `0x44`.
- Two zero writes to computed destinations based at `0x14`.
- Four value writes to computed destinations based at `0x14`.

“Value write” distinguishes the computation from an explicit constant-zero write; its value may still be zero. These counts exclude callee effects.

Each selector is unsigned and ranges from zero through 255. A selector-directed byte destination therefore ranges from `0x11` through `0x110`, inclusive. It is an absolute address, not an offset from the lookup base.

The retained-selector 32-bit clearing destination ranges from `0x44` through `0x143`, inclusive. It need not be aligned.

### Aliasing, state, and observability

No non-aliasing guarantee is established. Destinations may overlap selector storage, other destinations, absolute address `0x24`, or memory accessed by callees.

An earlier write can therefore change a later selector read. Later writes can replace earlier data. Callees can potentially observe intermediate state. A final memory image does not establish that the required intermediate accesses occurred.

The representation establishes access widths and displayed order, not atomicity, barriers, volatile semantics, or endianness. Multi-byte overlap consequences remain conditional on the actual machine and memory mapping.

### Ownership and errors

No direct allocation, release, ownership transfer, or buffer-length contract is established. Numeric dependency arguments must not be treated as owned pointers without further evidence.

Returned words are used in arithmetic and addresses without displayed validation. There is no represented error-result branch, rollback, or cleanup path.

A memory fault, invalid target, or nonreturning dependency can prevent all subsequent effects. Successful traversal requires accessible destinations, readable selector locations, and calls that return to their displayed continuations; the representation does not establish those conditions.

## Dependencies

F-D001 through F-D096 identify occurrences only. No actual callee identities are resolved by the supplied dependency information.

F-D001’s target is derived from the incoming control word by adding `0xfc4`. Later displayed targets are derived from embedded address-like values and offsets. Their validity, instruction mode, signature compatibility, and possible identity overlap are **documented_only**.

Consumed dependency outputs include:

- 32-bit words forwarded to later calls or used in scaling and addressing.
- F-D026’s represented 64-bit result, split into two 32-bit halves.
- The separate post-F-D027 register word.

No dependency is assumed to preserve globals, preserve selector storage, own memory, release memory, or return successfully. Values explicitly retained by the representation describe its dataflow; they do not establish a general calling-convention preservation guarantee.

## Acceptance examples

These are conditional static examples, not executed tests. Each assumes the relevant preceding calls return and memory operations succeed.

1. **Fresh selector and retained clearing selector.** If the initial selector is `3`, the payload is `0x00c000ab`, and the selector is `9` at the read after F-D007, the first byte write stores `0xab` at `0x14`. The second stores `0x03` at `0x1a`. The later retained-selector clearing writes a 32-bit zero at `0x4d`, even if the selector storage changes during F-D008 through F-D012.

2. **Explicit argument scaling.** With second explicit argument `7` and fourth explicit argument `5`, F-D001 receives `(7, 7, 7, 20)`.

3. **Separated result halves.** If F-D026 supplies low half `3` and high half `5`, F-D027 receives `12`, and the following computed clearing destination is `0x4e`. These halves do not establish F-D028’s register-derived arguments.

4. **Retained computed value.** If F-D050 returns `3` and F-D055 returns `5`, the first computed-value store writes `12` at `0x44`; the later retained-value store writes the same `12` at `0x28`.

5. **Zero is permitted.** If F-D050 returns zero, its computed-value store writes a 32-bit zero at `0x14`, and F-D051 receives zero. If F-D055 later returns `5`, the retained-value store writes zero at `0x28`. No nonzero-result requirement applies.

6. **Byte truncation versus full argument.** If F-D072 returns `0x12345678`, its first byte write stores `0xe0`; its later byte write stores `0x23`. F-D073 receives `0x48d159e0` under the 32-bit interpretation.

7. **Different final lookup.** If F-D084 returns `2`, the final selector lookup address is `0x11`. The preceding lookup still uses derived selector base plus one. Equal selectors or destinations are not required.

8. **Conditional wraparound.** If F-D021 returns `0xffffffff`, modular 32-bit arithmetic yields fourfold value `0xfffffffc` and clearing destination `0x40`. This is not a portable signed-arithmetic guarantee.

9. **Trace counts, not final contents.** A complete direct-access trace contains six selector reads, six byte writes, and four separate 32-bit zero writes to `0x24`. Callee accesses are excluded. One final zero value at `0x24` cannot prove that all four writes occurred.

## Uncertainties

The following remain **documented_only**:

- Accuracy of the recovered function boundary, interface, and straight-line control flow.
- Whether the terminal marker represents executable invalid-instruction behavior or failed recovery.
- Actual callee identities, hidden arguments, signatures, return conventions, and side effects.
- Whether the represented 64-bit result is one logical return value.
- Meaning and stability of ambient machine-state inputs and the post-call register word.
- Exact machine semantics for signed overflow, signed shifts, and address arithmetic.
- Address-space mapping, permissions, alignment requirements, and fault behavior.
- Endianness, atomicity, and effects of overlapping multi-byte accesses.
- Object layouts, valid index domains, ownership, and allocation extents.
- Whether low absolute addresses denote ordinary storage or special-purpose state.

Repeated scaling suggests indexed storage, and repeated zeroing suggests initialization. Both are inferred intent, not established behavior. No array, record, or device layout is justified.

## Evidence

The evidence is the supplied static representation: displayed arguments, ambient inputs, call ordering, access widths, constants, arithmetic, and termination marker.

The six selector-read/write pairs are individually accounted for before F-D001 and after F-D007, F-D072, F-D079, F-D084, and F-D091. The computed store following F-D050 has no constraint excluding zero.

Bad-instruction and overlapping-symbol warnings limit confidence in recovered control flow and types. No additional implementation inspection, runtime execution, memory tracing, or tests have been performed. Acceptance examples are conditional arithmetic consequences, not observed results.
