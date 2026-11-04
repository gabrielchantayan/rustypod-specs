<!-- Trusted provenance: F-dddd9a40027c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dddd9a40027c

## Purpose

F-dddd9a40027c answers whether an integer input is enabled by a selected bit in an externally supplied capability word.

The function recognizes several numeric values through fixed comparisons. It also recognizes a configurable selector value, whose equality case takes precedence over every fixed-value case. Recognition alone does not produce a true result: the selected capability bit must be set.

The numeric values resemble sampling frequencies, but that interpretation is inferred intent, not an established interface guarantee. The established behavior is an integer-to-bit membership query controlled by external state.

This contract describes static behavior only. No runtime tests have been performed.

## Interface

**Input**

One integer argument, represented as a signed `int` in the supplied source representation.

There is no explicit input validation, range normalization, unit conversion, or special treatment of negative values. An input that equals the external selector can reach the selector-equality case regardless of its numeric value.

**External inputs**

The function depends on:

- An externally held base address.
- An unsigned integer capability word located at byte offset 16 from that base.
- An external selector used first for equality and, if equality fails, for an ordering comparison against the input.

The external selector is not supplied as an explicit argument. Its declaration, initialization, and permitted values are not included in the available evidence.

**Output**

A Boolean result:

- **True:** a recognized case selects a capability bit, and that bit was set in the capability word read by this invocation.
- **False:** no applicable fixed-value case is recognized, or the selected bit was clear.

The function exposes no separate error result.

**Widths and layout**

The conventional interpretation of the supplied integer types is a signed 32-bit input and an unsigned 32-bit capability word. Under that interpretation, the word occupies four bytes beginning at offset 16, and the highest inspected bit is bit 8.

Exact ABI widths are not independently established by the supplied fragment. A platform-specific publication must confirm that interpretation rather than treating the type spellings alone as an ABI declaration.

All masks fit within nine bits. There is no input addition, subtraction, multiplication, division, narrowing conversion, or shift. The only address calculation shown is the addition of 16 to the external base.

**Memory preconditions**

For a defined execution, the external base must permit the unsigned-word read at offset 16, with the alignment and access properties required by the actual execution environment.

This precondition applies even when the numeric input will eventually be rejected.

## Behavior

### Initial capability read

The function obtains the capability word before deciding whether the input is recognized.

Consequently, an unsupported input is not a safe probe that avoids accessing external memory. If the capability location is invalid, the function has no shown path that returns false without first attempting that read.

The word is read once in the supplied representation. Subsequent bit tests use that captured value rather than explicitly fetching the capability word again.

### Selector equality has priority

The function first compares the input with the external selector.

If they compare equal, it selects mask 16, corresponding to bit 4, and returns whether that bit is set.

This case overrides the fixed numeric mappings. For example, if the selector equals 8000 and the input is 8000, bit 4 controls the result; bit 0 does not.

Equality also permits selector values that do not appear among the fixed numeric cases. There is no independent restriction requiring the selector to be one of those values.

### Ordering selects a fixed-value group

If equality fails, the function compares the selector with the input using a strict less-than comparison.

When the selector compares less than the input, only the following fixed-value cases are considered:

| Input | Selected mask, decimal | Selected bit |
|---|---:|---:|
| 24000 | 32 | 5 |
| 32000 | 64 | 6 |
| 44100 | 128 | 7 |
| 48000 | 256 | 8 |

Any other input in this branch returns false.

When that less-than comparison is false, only these fixed-value cases are considered:

| Input | Selected mask, decimal | Selected bit |
|---|---:|---:|
| 8000 | 1 | 0 |
| 11025 | 2 | 1 |
| 12000 | 4 | 2 |
| 16000 | 8 | 3 |

Any other input in this branch returns false.

There is no fallback between the two groups. A value listed in one group is rejected if execution enters the other group, unless the earlier selector-equality case already accepted it for testing against bit 4.

For a stable selector and ordinary signed integer comparisons, these groups can be described as inputs above and below the selector. Because the selector’s declaration is absent, the authoritative distinction is the result of the comparisons as evaluated in the actual type environment. Signedness and integer-conversion details remain unresolved.

### Fixed comparisons and result formation

Within the higher group, fixed comparisons are considered in this order: 24000, 32000, 44100, then 48000.

Within the lower group, they are considered in this order: 8000, 11025, 12000, then 16000.

Once a case selects a mask, the result is whether the captured capability word has a nonzero intersection with that mask.

Each mask contains exactly one bit. Therefore:

- Other set bits cannot compensate for a clear selected bit.
- Additional set bits do not invalidate a match.
- Bits above bit 8 do not affect the result.
- A zero capability word produces false for every input, assuming execution completes with defined memory access.
- Setting all nine relevant bits does not make arbitrary non-selector inputs valid; the branch and fixed-value restrictions still apply.

### Selector stability

The selector appears in two separate source-level comparisons: equality first, ordering second.

The fragment does not establish that these comparisons use a single captured selector value. If external mutation can occur between them, the second comparison may observe a different value, subject to the language, compilation, and concurrency rules of the actual environment.

For deterministic interpretation through the tables above, require the selector to remain stable during the invocation. The function itself provides no synchronization.

## Side effects

The supplied function contains no explicit writes to the capability storage, selector, or other external state.

It also contains no explicit:

- Allocation or deallocation.
- Ownership transfer.
- Resource acquisition or release.
- Logging or diagnostic reporting.
- Callback or helper invocation.

The base address and referenced storage are used for reading only. The function does not establish ownership of that storage, and the evidence does not establish which component owns it or how long it remains valid.

The source-level order of external accesses is:

1. Obtain the base address needed for the capability access.
2. Read the unsigned capability word at offset 16.
3. Read the selector for equality.
4. If equality fails, evaluate a further selector comparison.

These are source-level access relationships, not a guarantee of processor bus transactions or cross-thread visibility. The fragment supplies no volatile declarations, atomic operations, barriers, or memory-ordering contract.

A bad address or unsuitable alignment is not converted into a Boolean error by any mechanism shown here. The resulting failure behavior depends on the execution environment.

## Dependencies

No direct function dependencies are visible. There are therefore no additional opaque function IDs to enumerate.

The behavioral dependencies are external data:

| Dependency | Required contribution | Established limitation |
|---|---|---|
| Base-address storage | Supplies the address used for the capability read | Initialization, validity checks, and lifetime are not shown |
| Capability storage at offset 16 | Supplies the unsigned word tested by the invocation | No structure declaration or access synchronization is supplied |
| Selector storage | Controls equality precedence and fixed-group selection | Declaration, signedness, initialization, and stability are not supplied |

No assumptions about an unknown callee preserving globals or releasing memory are needed: no callee is visible.

The function does not inspect any preceding fields at offsets 0 through 15. Their meanings, if any, are outside this contract.

## Acceptance examples

These are conditional acceptance criteria derived from the static evidence, not observed test results.

Unless stated otherwise, assume valid capability storage, a stable selector, the conventional 32-bit width profile, and comparisons that order the positive example values numerically.

| Selector | Input | Capability word | Expected result | Reason |
|---:|---:|---:|---|---|
| 22050 | 8000 | 1 | True | Lower-group case selects bit 0 |
| 22050 | 11025 | 2 | True | Lower-group case selects bit 1 |
| 22050 | 12000 | 4 | True | Lower-group case selects bit 2 |
| 22050 | 16000 | 8 | True | Lower-group case selects bit 3 |
| 22050 | 22050 | 16 | True | Selector equality selects bit 4 |
| 22050 | 24000 | 32 | True | Higher-group case selects bit 5 |
| 22050 | 32000 | 64 | True | Higher-group case selects bit 6 |
| 22050 | 44100 | 128 | True | Higher-group case selects bit 7 |
| 22050 | 48000 | 256 | True | Higher-group case selects bit 8 |

The example selector 22050 is illustrative. The evidence does not establish it as the actual configured value.

**Selected bit must be present.** With selector 22050, input 44100, and capability word 256, the result is false. Bit 8 is set, but this input requires bit 7.

**Equality overrides a normal mapping.** With selector and input both 8000, capability word 1 produces false, while capability word 16 produces true.

**An unlisted selector value can match.** With selector and input both 12345, capability word 16 produces true. Changing the input to 12346 produces false under numeric ordering, even with capability word 511.

**A listed value can be routed away from its group.** With selector 7000, input 8000, and capability word 511, the result is false. The higher-group branch has no 8000 case.

**The reverse routing restriction also applies.** With selector 50000, input 48000, and capability word 511, the result is false. The lower-group branch has no 48000 case.

**An unsupported input still requires readable storage.** Input 12346 does not permit bypassing the initial capability read merely because no fixed case recognizes it.

## Uncertainties

The following are **documented_only** limitations, not behavior to guess or implement by convention:

- **Exact ABI:** concrete integer widths, Boolean return representation, pointer width, byte order, and alignment requirements are not declared in the fragment.
- **Selector type:** its signedness and any conversions affecting equality or ordering are unknown.
- **Selector configuration:** no actual value or valid-value range is established.
- **Storage layout beyond offset 16:** no enclosing structure, buffer extent, or neighboring field semantics are available.
- **Initialization and lifetime:** the fragment does not show who creates, updates, or invalidates the external storage.
- **Concurrent access:** selector stability and capability-update synchronization are not established.
- **Fault handling:** no local recovery is shown; process-level handlers or platform exceptions are outside the evidence.
- **Domain meaning:** interpreting the numeric inputs as rates is plausible but unproven.
- **Runtime fidelity:** no execution evidence confirms behavior on a particular target.

These unknowns do not obscure the fixed masks, equality precedence, initial capability access, or branch-specific recognition rules visible in the supplied representation.

## Evidence

Established static evidence consists of:

- One explicit integer argument and a Boolean result.
- An initial unsigned-word read at base offset 16.
- Selector equality selecting mask 16 before fixed-value recognition.
- A subsequent strict ordering comparison choosing one of two fixed-value groups.
- Four fixed cases per group, using masks 1 through 8 and 32 through 256 respectively.
- False returns for unrecognized inputs within either group.
- A final nonzero test of the selected bit.
- No explicit external writes or function calls.

The behavioral description above is independent of comments or presumed domain intent. It does not claim a known selector value, a verified target ABI, or successful runtime execution.
