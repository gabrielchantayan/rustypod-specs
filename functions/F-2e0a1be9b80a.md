<!-- Trusted provenance: F-2e0a1be9b80a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2e0a1be9b80a

## Purpose

The supplied static representation describes an ordered sequence of indirect calls, byte accesses, 32-bit accesses, and arithmetic transformations. It does not establish an application-level purpose.

The visible sequence consumes inherited machine state, uses dependency results as data and address components, writes to low numerical addresses, and ends at a reported invalid-instruction boundary. No ordinary return path is shown.

This contract describes the visible behavior conditionally. The invalid-instruction warnings leave the correctness of the recovered function boundary and instruction interpretation unresolved.

## Interface

### Inputs

The recovered interface declares no explicit parameters, but the body consumes four inherited values:

| Contract name | Apparent width | Visible use |
|---|---:|---|
| Entry shift word | 32 bits | Shifted left by 19 and passed three times to the first dependency |
| Entry byte-source word | Unsigned 32 bits | Supplies the initial two byte payloads |
| Entry base address | 32 bits | Supplies addresses at offsets 1 and 13 |
| Entry control address | 32 bits | Supplies the first indirect-call target after addition of 3816 |

These are descriptive contract names, not established parameter names or ABI declarations. The calling convention and means of initializing this state are unknown.

Further inputs include memory contents, apparent 32-bit and 64-bit dependency results, and an apparent unsigned 32-bit secondary result channel.

A call shown without explicit arguments does not establish that its callee consumes no other machine state.

### Outputs

No ordinary return value is established. Observable outputs comprise:

- The ordered memory writes described below.
- The indirect dependency invocations and their visible arguments.
- Any additional dependency effects, which are unresolved.
- Arrival at the reported invalid-instruction boundary, if all preceding operations complete.

### Widths and notation

The following mathematical notation describes values, not an implementation:

- **low8(x)**: the least significant eight bits of a value.
- **Lₙ(x)**: a left shift by *n*, retaining the low 32 bits under the conditional machine-word interpretation.
- **Rₙ(x)**: an unsigned 32-bit right shift by *n*.
- **byte(a)**: an unsigned eight-bit read at address *a*.
- **word(a)**: a 32-bit read at address *a*.

All byte stores retain eight bits. Word stores retain 32 bits. Byte-derived addresses use the byte’s unsigned value.

The apparent address model is 32 bits. Modular arithmetic descriptions below are conditional on that model. Some shifts, multiplications, and address additions involve signed intermediates in the representation; their overflow behavior is not independently established. The contract does not turn those expressions into unconditional portable arithmetic guarantees.

Byte order, alignment requirements, access atomicity, memory mapping, and fault behavior are unknown.

## Behavior

### Ordering and dependency labels

**F-d001** through **F-d065** identify the 65 visible indirect-call occurrences in execution order. These opaque labels distinguish occurrences, not necessarily different implementations.

Every subsequent operation is conditional on prior operations completing and prior dependencies returning. Retained values below describe the representation’s dataflow; they do not establish a general guarantee that unknown callees preserve memory, globals, or resource lifetimes.

### 1. Entry-state use and initial byte writes

Compute **L₁₉(entry shift word)**. Read a byte at entry base address plus 1, then write **low8(entry byte-source word)** to the sampled byte value plus 17.

Invoke **F-d001**, whose target is entry control address plus 3816, with three identical arguments: the shifted entry word.

Continue in this order:

1. Invoke **F-d002** and retain its apparent unsigned 32-bit result as the **first result**.
2. Invoke **F-d003** with the first result.
3. Invoke **F-d004**, **F-d005**, and **F-d006**, with no explicit arguments.
4. Invoke **F-d007** and retain its apparent 32-bit result as the **first scaled input**.
5. Read a fresh byte at entry base address plus 1. Retain this as the **index byte**.
6. Write **low8(R₂₂(entry byte-source word))** to index byte plus 17.
7. Invoke **F-d008** with **L₂(first scaled input)**.
8. Invoke **F-d009** through **F-d012**, in order, with no explicit arguments.

The two reads at entry base address plus 1 are separate observable accesses. The intervening calls, or an aliasing earlier write, may affect the second sample.

### 2. Saved-byte writes and secondary result

Define:

- **Small base** = twice the index byte, with a mathematical range of 0 through 510.
- **Saved byte** = **low8(R₃(first result))**.

The sequence continues:

1. Write the saved byte at entry base address plus 13.
2. Write the index byte at absolute address 13.
3. Invoke **F-d013** with small base, 152, and 608.
4. Invoke **F-d014** with no explicit arguments.
5. Invoke **F-d015** with the apparent unsigned 32-bit secondary result channel shifted right by 3.
6. Invoke **F-d016**, **F-d017**, and **F-d018**, with no explicit arguments.
7. Invoke **F-d019**, obtaining an apparent 32-bit result.
8. Invoke **F-d020** with that result shifted left by 2; retain its apparent 32-bit result.
9. Write the saved byte at small base plus 13.
10. Write 192 at absolute address 13.
11. Invoke **F-d021** with the result of **F-d020** shifted left by 2.
12. Invoke **F-d022**, **F-d023**, and **F-d024**, with no explicit arguments.
13. Invoke **F-d025** and retain its apparent 32-bit result as the **record result**.

The secondary channel is exposed after **F-d014**. Its apparent unsigned width supports the displayed logical right shift, but its actual ABI role is unknown.

The second saved-byte write uses the retained saved byte. It does not read back the earlier saved-byte destination.

### 3. Word accesses using a loaded address

Write **L₂(record result)** as a 32-bit word at small base plus 36. Then read a 32-bit word at small base plus 116 and retain it as the **loaded address word**.

Define the **wide-scaled result** as **L₅(record result)**.

Invoke **F-d026** with these four arguments, in order:

1. Loaded address word.
2. **L₁(loaded address word)**.
3. Wide-scaled result.
4. **R₂(loaded address word AND 0x7fffffff)**.

Invoke **F-d027**, **F-d028**, **F-d029**, and **F-d030**, with no explicit arguments.

Next, write the wide-scaled result as a 32-bit word at loaded address word plus 36. Read a 32-bit word at loaded address word plus 116 and pass that sample to **F-d031**.

Invoke **F-d032**, **F-d033**, and **F-d034**, with no explicit arguments.

The loaded address word is retained across these calls. The later read at its offset 116 occurs after the write at its offset 36. Neither base is proven to denote an object or structure; the offsets establish locations, not field meanings.

### 4. Paired results and indexed word writes

Invoke **F-d035**, obtaining an apparent 64-bit result. Split it numerically into low and high 32-bit halves. Define the **paired scaled value** as the low half shifted left by 2.

Write **R₃(first result)** as a full 32-bit word at four times the high half plus 20.

Invoke **F-d036** with:

1. Paired scaled value.
2. Paired scaled value.
3. Paired scaled value.
4. High half shifted right by 3.

Invoke **F-d037**, obtaining a second apparent 64-bit result. Invoke **F-d038** with its low half, its high half, its low half again, and its high half shifted right by 3.

Continue:

1. Invoke **F-d039**, **F-d040**, and **F-d041**, with no explicit arguments.
2. Invoke **F-d042** and retain its apparent 32-bit result as the **destination index**.
3. Write the paired scaled value as a 32-bit word at four times the destination index plus 20.
4. Invoke **F-d043** with four times the destination index.
5. Invoke **F-d044**, **F-d045**, and **F-d046**, with no explicit arguments.
6. Invoke **F-d047** and retain its apparent unsigned 32-bit result as the **late result**.

The first indexed word write stores the full 32-bit shifted first result, not the saved byte used earlier.

The two apparent 64-bit results may represent paired machine-result channels rather than semantic scalar values. Only their visible low/high extraction and subsequent use are established.

### 5. Late-result byte writes

Define the **late scaled value** as **L₂(late result)**.

Read a byte at loaded address word plus 1. Write **low8(late scaled value)** to the sampled byte value plus 17.

Then:

1. Invoke **F-d048** and **F-d049**, with no explicit arguments.
2. Invoke **F-d050**, retaining its apparent 32-bit result.
3. Read a fresh byte at loaded address word plus 1.
4. Write **low8(R₂₀(late result AND 0x3fffffff))** to this new sample plus 17.
5. Invoke **F-d051** with the result of **F-d050** shifted left by 2.
6. Invoke **F-d052**, **F-d053**, and **F-d054**, with no explicit arguments.
7. Invoke **F-d055**, retaining its apparent 32-bit result.

The second payload is formed by masking the 32-bit late result, shifting it right, and only then truncating to eight bits. It is not formed by shifting an already-truncated byte.

Both destinations range from 17 through 272. They need not be equal.

### 6. Terminal zero writes and boundary

Define the **lookup address** as late scaled value multiplied by 1024, plus 1, under the conditional 32-bit arithmetic model.

Read a byte at that address and write zero to the sampled byte value plus 17.

Invoke **F-d056** with:

1. **L₃(late scaled value)**.
2. The same value again.
3. Late scaled value.
4. **L₂(result of F-d055)**.

Continue:

1. Invoke **F-d057**, **F-d058**, and **F-d059**, with no explicit arguments.
2. Invoke **F-d060**, retaining its apparent 32-bit result.
3. Read a fresh byte at the lookup address.
4. Write zero to this new sample plus 17.
5. Invoke **F-d061** with **L₂(result of F-d060)**.
6. Invoke **F-d062**, **F-d063**, **F-d064**, and **F-d065**, with no explicit arguments.
7. Reach the reported invalid-instruction boundary.

The lookup address calculation is retained, but the byte stored there is sampled twice. The two zero writes remain separate operations even if their destinations coincide.

## Side effects

### Direct memory effects

For a traversal of the entire visible sequence, the direct write counts are:

| Category | Byte writes | 32-bit writes |
|---|---:|---:|
| Initial byte-derived destinations | 2 | 0 |
| Saved-byte destinations at base plus 13 | 2 | 0 |
| Absolute address 13 | 2 | 0 |
| Small-base and loaded-address destinations at offset 36 | 0 | 2 |
| Indexed destinations at offset 20 | 0 | 2 |
| Late-result byte-derived destinations | 2 | 0 |
| Terminal zero byte-derived destinations | 2 | 0 |
| **Total** | **10** | **4** |

There are **six**, not eight, byte writes through byte-derived addresses: two initial writes, two late-result writes, and two terminal zero writes. The two saved-byte writes and two absolute-address-13 writes are additional categories, producing ten byte writes overall.

The explicit direct reads comprise six byte reads and two 32-bit reads. Dependency accesses are not included in these counts.

### Aliasing and observable order

No disjointness among accessed addresses is established.

- Small base plus 13 equals absolute address 13 when the index byte is zero.
- Byte-derived destinations may overlap byte sources or other low-address accesses.
- Word accesses may overlap byte accesses or each other.
- Dependencies may alter memory later sampled by this function.

Repeated writes must not be collapsed, and repeated reads must not be replaced by cached samples. The representation does not establish one universal final memory image.

These accesses are not proven to address ordinary RAM. Device mappings, reserved regions, or unmapped addresses could change their observable consequences.

### Ownership and lifetime

No explicit allocation, release, ownership transfer, locking, or cleanup operation is visible.

This does not constrain unknown dependencies to avoid those effects. Values used as addresses have no established ownership, lifetime, provenance, or alignment guarantees.

### Errors and interruption

No visible validation, recoverable error result, rollback, or cleanup branch precedes the accesses.

If an access faults or a dependency does not return, later operations are not established to occur. Earlier writes may already have taken effect.

The terminal warning does not establish a particular exception, signal, process exit, or hardware response. Behavior beyond that boundary is unknown.

## Dependencies

The 65 opaque labels identify call occurrences only. Their semantic identities are unresolved.

| Occurrences | Explicitly consumed result |
|---|---|
| **F-d002** | Unsigned 32-bit first result |
| **F-d007**, **F-d019**, **F-d020**, **F-d025** | Apparent 32-bit values subsequently shifted or stored |
| **F-d014** | Associated with the subsequent apparent unsigned 32-bit secondary channel |
| **F-d035**, **F-d037** | Apparent 64-bit values split into 32-bit halves |
| **F-d042** | Apparent 32-bit destination index |
| **F-d047** | Unsigned 32-bit late result |
| **F-d050**, **F-d055**, **F-d060** | Apparent 32-bit values shifted and forwarded |
| Remaining occurrences | No returned value explicitly consumed |

The first target depends on inherited control state. Other targets are reconstructed from embedded code-location expressions and offsets. Their actual identities and executable validity are not established.

Unknown callees are not assumed to preserve globals, addressed memory, allocations, or object lifetimes. Their additional effects remain `documented_only`.

## Acceptance examples

These are conditional static examples, not executed tests. They assume valid interpretation of the represented operations, successful accesses, returning dependencies, and the stated arithmetic model.

1. **Fresh initial sampling.** If the first byte read at entry base plus 1 yields 5, the initial destination is address 22. If the second read yields 9, the second destination is address 26 and small base is 18. Reusing the first sample would change the specified behavior.

2. **Byte truncation versus word storage.** If the first result is `0x12345678`, its unsigned right shift by 3 is `0x02468acf`. The saved-byte writes carry `0xcf`; the first indexed word write carries `0x02468acf`.

3. **Address-13 aliasing.** If the index byte is zero, small base is zero. The second saved-byte write targets address 13 and is immediately followed by a write of 192 there. Both stores remain observable operations.

4. **Distinct record scales.** If the record result is 3, the word at small base plus 36 receives 12. The later word at loaded address word plus 36 receives 96.

5. **Paired-result addressing.** If **F-d035** yields low half 7 and high half 9, the paired scaled value is 28. The first indexed word write targets address 56. **F-d036** receives 28, 28, 28, and 1.

6. **Late payload width.** If the late result is `0xffffffff`, masking produces `0x3fffffff`; shifting right by 20 produces `0x3ff`; byte truncation produces `0xff`.

7. **Fresh terminal sampling.** If the terminal lookup reads yield 4 and 8, the zero writes target addresses 21 and 25. Reusing the first sample for the second destination would violate the access sequence.

8. **Complete write accounting.** If the entire visible path completes, it contains six byte-derived destination writes, four other byte writes, and four word writes. Aliasing does not reduce these operation counts.

9. **Interrupted traversal.** If **F-d026** does not return, the word write at small base plus 36 precedes that interruption. The later word write at loaded address word plus 36 is not established to occur.

## Uncertainties

### Established static evidence

The supplied representation establishes the displayed access widths, explicit dataflow, call ordering, argument construction, masks, shifts, result-half extraction, and reported invalid-instruction endpoint, subject to its decoding limitations.

### Inferred intent

Repeated offsets and scaling may suggest structured state or device-oriented operations. Neither interpretation is sufficiently supported to become contractual.

### `documented_only`

The following remain unresolved:

- Whether the region is correctly decoded and constitutes a valid function.
- The real ABI, entry-state requirements, and function boundaries.
- Actual callee identities, signatures, and implicit inputs.
- Whether apparent 64-bit results are scalar returns or paired machine values.
- The secondary channel’s actual ABI meaning and physical width, despite its apparent unsigned 32-bit representation.
- Signed-intermediate overflow, left-shift, multiplication, and address-arithmetic behavior.
- Memory mapping, byte order, alignment, atomicity, and fault semantics.
- Dependency effects on memory, globals, ownership, and resource lifetimes.
- Any normal return path or behavior beyond the terminal boundary.

## Evidence

The sole evidence is the supplied static representation for **F-2e0a1be9b80a**, including its invalid-instruction warnings. The accompanying dependency map identifies only this function and resolves no callee identities.

The direct-write count follows the visible destinations: six byte-derived writes, two saved-byte writes, two absolute-address-13 writes, and four 32-bit writes.

No implementation access, instruction-level validation, runtime execution, or tests have been performed. Acceptance examples express conditional arithmetic and ordering consequences, not observed runtime behavior.
