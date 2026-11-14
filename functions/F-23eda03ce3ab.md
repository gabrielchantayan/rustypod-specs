<!-- Trusted provenance: F-23eda03ce3ab; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-23eda03ce3ab

## Purpose

F-23eda03ce3ab accepts one 32-bit word and returns one unsigned 32-bit word. Its visible behavior consists of externally controlled dispatch, fixed-stride record addressing, byte reads and arithmetic transformations, and retention or replacement of candidate words.

**Established static evidence:** there are no explicit calls, allocations, releases, or writes to externally addressed storage. The routine has one return operation, which returns the retained result.

**Inferred intent:** the record addresses, paired byte processing, and candidate state suggest lookup or matching. The supplied material does not establish a string format, comparison relation, table size, traversal order, or selection policy.

External words determine dispatch transitions and participate in address and predicate arithmetic. Their values are absent. A complete entry-to-return input/output mapping is therefore **documented_only**. That limitation does not obscure the unconditional dispatcher reads or the ordering of the paired byte reads described below.

## Interface

### Input

The sole argument is a signed 32-bit word. On applicable paths, its bit pattern is copied into an active byte-storage address.

The active address can subsequently be replaced by a word read from a record. A later zero test therefore applies to the current active address, not necessarily to the original argument.

No unconditional validation of the argument is established. In particular, a zero argument is not proven to reach a rejection path before any dereference.

### Output

The returned value is an unsigned 32-bit word held in retained-result state.

That state:

- Starts at zero.
- Can receive the current candidate word.
- Can be reset to zero.
- Is returned without a final validity check.

Candidate state can contain a calculated record address. A nonzero return can consequently identify a record, but the evidence does not establish that every nonzero return is a valid record address.

There is no separate status output, length output, error object, or established error-code namespace.

### Widths and memory requirements

The visible operations use:

| Quantity or access | Width and interpretation |
|---|---|
| Argument | Signed 32-bit word |
| Return value | Unsigned 32-bit word |
| Loaded byte | Unsigned 8-bit value, widened before transformation |
| Record-index snapshot | Signed 16-bit narrowing of working index state |
| Record fields | 32-bit word loads |
| Working arithmetic and addresses | Predominantly 32-bit, with signed and unsigned interpretations |
| Selected candidate tests | Unsigned comparisons against zero and one |

Every reached read requires readable storage of the corresponding width. No bounds, alignment, or address-validity checks are visible.

The supplied representation distinguishes arithmetic and logical right shifts. Exact target rules for signed overflow, out-of-range signed conversions, address overflow, word alignment, and byte order are not independently established. Arithmetic descriptions involving wrapped addresses or signed extraction are conditional on the corresponding 32-bit machine interpretation.

### Ownership

The routine visibly creates no independent storage and performs no release or reference-count operation.

If the returned word identifies table storage, its validity and lifetime depend on that storage. The function does not establish ownership transfer for the argument, records, byte buffers, or returned address.

## Behavior

### Initialization and dispatch

Working counters, candidates, saved addresses, comparison intermediates, and retained-result state are initialized to zero before the first dispatch attempt. An external word supplies the initial control value.

There are 42 accepted selectors, numbered 0 through 41 inclusive. Every dispatch attempt has the following established order:

1. Read the external selector-offset word and form the selector from that word and the current control value.
2. Read the external fallback-control word and replace the current control value with it.
3. Test the selector as an unsigned 32-bit value.
4. If the selector exceeds 41, repeat dispatch without executing a selected operation.
5. Otherwise, take a signed 16-bit snapshot of the working record index.
6. Read the external default-transition word.
7. Execute the selected operation.

The fallback-control read occurs even when the selector is already in range. The default-transition read occurs after every accepted selector, including the selector that immediately returns the retained result. Neither read may be treated as conditional on whether its value is subsequently useful.

An accepted operation can leave the previously loaded fallback control in place, replace control directly, or combine transition values through arithmetic derived from a predicate. The default-transition value may be used or superseded by the selected operation; its read still occurs.

A selected operation can perform further external reads before dispatch resumes. Repeated reads of an external word are separate accesses rather than an established invocation-wide snapshot.

Missing external values prevent resolution of the complete path, but not the ordering above. If successive fallback attempts remain out of range, dispatch can continue indefinitely. No timeout or iteration bound is visible.

### Record addressing and layout

The visible record layout is:

| Relative position | Observed use |
|---|---|
| Offset 0 | Calculated record address; no direct field read at this offset is shown |
| Offset 8 bytes | 32-bit word tested for nonzero or loaded as a byte-storage address |
| Offset 12 bytes | 32-bit word used in a coefficient-dependent branch calculation |
| Stride | 184 bytes |

A record address is calculated from the externally read table base plus 184 times the signed 16-bit index snapshot taken for the current accepted dispatch.

That snapshot precedes execution of the selected operation. It is not a fresh narrowing performed after any index update within that operation. The table base itself is read by the operations that use it; no permanent base snapshot is established.

The working index is derived through width-sensitive arithmetic. One operation multiplies a counter by \(3\cdot2^{24}\) and shifts the signed result right by 24. Under wrapping 32-bit arithmetic with an arithmetic right shift, this extracts the signed low byte of three times that counter.

Consequences:

- A record index can be negative.
- Record addressing is not proven to remain at or above the table base.
- The stride does not establish allocation size or record count.
- Neither ascending traversal nor unit increments are established.

The offset-8 field has separate read operations for a nonzero test, saving a byte-storage address, and replacing the active address. These are not a single cached read. A successful presence test does not establish what a later read will observe.

The offset-12 field is multiplied by an external coefficient before participating in a predicate. Its meaning as length, status, flags, or other metadata is unknown.

### Byte reads and transformations

Reached byte operations calculate an address from an active or saved base address and an externally scaled working position. The scale is not established to be one, positive, or stable.

A loaded byte is treated as a value from 0 through 255 before the common transformation:

\[
T(b)=(171b)\bmod256.
\]

Another transformation, applied to selected arithmetic intermediates, is:

\[
R(x)=213(x+224)\bmod256.
\]

The low-byte masks retain exactly eight bits. For the second expression, the displayed modular form describes its low-byte result under the apparent 32-bit arithmetic interpretation; it does not settle every signed-overflow question in the supplied representation.

These transformations are not established encoding or decoding operations. In particular,

\[
171\cdot213=36423\equiv71\pmod{256},
\]

so the two multipliers are not multiplicative inverses modulo 256. No inverse relationship between the displayed transformations is established.

### Paired-read ordering

One operation reads bytes through two saved base addresses using the same internal position counter. Its externally observable read sequence is:

1. Read the external scale word for the first address calculation.
2. Read the byte at the first saved base plus that scaled position.
3. Read the external scale word again for the second address calculation.
4. Read the byte at the second saved base plus that independently scaled position.

The first byte is transformed and retained before the second byte-processing statement. The second byte is also transformed and retained. Subsequent arithmetic derives control information from the first transformed byte, while another operation processes the second.

The two accesses share a counter, not a captured scale value. Their effective offsets are equal if the scale reads return the same value, or if the resulting products otherwise coincide under the applicable arithmetic. Equal offsets are not unconditional.

If the first byte read cannot complete normally, the source does not establish that the second scale read or second byte read occurs. The routine contains no intervening explicit write to either byte source.

This paired processing supports a comparison or classification inference. It does not establish equality versus ordering, case sensitivity, a character set, termination on zero, maximum length, or prefix handling.

### Counters and predicates

Several working counters have explicit reset operations. Their updates use external coefficients rather than an independently established increment by one. A recurring update combines a counter with an external additive factor and then multiplies by external factors.

Other calculations combine complements, products, sums, differences, masks, and sign-bit extraction. Constants including 64, 180, and 181 appear in predicates, but do not independently establish byte-class boundaries or length limits.

Width distinctions must be preserved:

- Unsigned right shift by 31 extracts a bit valued zero or one.
- A signed right shift by 31, followed by addition of one, is sign-derived under arithmetic-shift semantics.
- Signed 16-bit narrowing affects record addressing.
- Some control calculations narrow working values to a character-sized type.
- Low-byte masking discards higher product bits.

The signedness of the character-sized type and the target’s signed conversion rules are not independently supplied. The affected control calculations cannot safely be replaced by ordinary high-level comparisons without further evidence.

### Candidate state and return

Candidate state can hold calculated record addresses, zero, and saved earlier candidate values. Visible operations calculate, save, restore, clear, and replace these words.

Some candidate-dependent decisions explicitly distinguish zero, one, and values greater than one using unsigned comparisons. These are numeric tests, not pointer validation.

The active-address zero-test operation resets the retained result to zero before selecting its subsequent control value. That reset occurs whether the active address is zero or nonzero.

The return operation returns the current retained result without dereferencing or validating it. However, it is still preceded by the accepted-dispatch index snapshot and mandatory default-transition read. A value being ready for return does not eliminate those prerequisites.

The intended policy—first match, last match, unique match, or another rule—remains **documented_only**.

## Side effects

### External reads

Visible external reads include:

- The initial control word.
- The selector-offset and fallback-control words on every dispatch attempt, in that order.
- The default-transition word after every accepted selector and its index snapshot.
- Additional transition, coefficient, and mask words used by selected operations.
- The table base during record calculations.
- Record words at offsets 8 and 12.
- Bytes through active or saved addresses.

Within the paired operation, the order is first scale read, first byte read, second scale read, second byte read. These reads follow that operation’s mandatory dispatch reads.

A complete invocation-wide read trace remains dependent on external values. These source-level sequences do not establish hardware synchronization or atomicity guarantees.

### Writes and state changes

No explicit write to the input buffer, table, referenced byte storage, or external control storage is visible. Assignments update invocation-local state.

Invalid reads remain significant despite the absence of writes. A fault in dispatch storage, record storage, or byte storage may prevent normal completion. No recovery, defined zero-result fallback, or fault-to-status conversion is shown.

### Concurrency

No lock, atomic operation, or snapshot protocol is visible.

Concurrent modification can affect repeated dispatch reads, table-base reads, field reads, or scale reads. Requirements for stable storage are environmental preconditions, not enforced properties. In particular, equal paired offsets require an appropriate stability condition or coincident arithmetic results.

## Dependencies

No explicit callable dependencies are present; there are no additional opaque function IDs to enumerate.

The data dependencies are:

| Dependency | Behavioral role |
|---|---|
| Initial-control word | Establishes the first control value |
| Selector-offset word | Participates in every selector calculation |
| Fallback-control word | Read on every attempt; supplies retry or retained control |
| Default-transition word | Read after every accepted selector, including return |
| Other transition words | Determine operation-specific control changes |
| Coefficients, masks, and scale words | Affect predicates, transformations, updates, and addresses |
| Record-table base | Origin for signed-index, 184-byte-stride addressing |
| Offset-8 record field | Presence test or byte-storage address |
| Offset-12 record field | Arithmetic predicate input |
| Argument- and record-addressed storage | Byte sources on reached paths |

Their initialization, accessibility, valid ranges, lifetimes, and mutation rules are not supplied. No dependency-preservation or memory-ownership behavior is assumed.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed test results.

1. **Immediate return with initialized state.**  
   If the first accepted selector is the return operation, the result is zero, provided all required reads complete. The selector-offset read, fallback-control read, signed-index snapshot, and default-transition read still precede return.

2. **In-range dispatch.**  
   If an attempt produces selector 16, the fallback-control word is nevertheless read. After range acceptance, the index snapshot and default-transition read occur before the retained result is returned.

3. **Out-of-range dispatch.**  
   If a selector is 42, the selector-offset and fallback-control reads occur, but no accepted-operation index snapshot or default-transition read occurs for that attempt. Dispatch retries using the loaded fallback control.

4. **Mandatory read prevents return.**  
   If the return selector is accepted but the default-transition word is unreadable, normal return is not established even though that operation does not use the loaded transition value.

5. **Signed record index.**  
   With snapshot index −1, a reached record calculation addresses 184 bytes before the observed table base, subject to the target address-arithmetic rules. No lower-bound rejection is visible.

6. **Byte transformation.**  
   If a transforming read observes byte 255, the widened input to multiplication is 255 and the masked result is 85. If it observes zero, the transformed result is zero; this alone does not establish termination behavior.

7. **Changing paired scales.**  
   If the shared counter is 3 and the two scale reads yield 1 and 2, the byte offsets are respectively 3 and 6, assuming no relevant overflow. The first byte read occurs before the second scale read.

8. **Stable paired scale.**  
   If both scale reads yield the same word, both address calculations use the same counter, and arithmetic is consistent, their offsets are equal. Their base addresses can still differ.

9. **Active-address result reset.**  
   If the active-address zero-test operation executes, retained-result state becomes zero for both zero and nonzero active addresses. The subsequent control choice depends on whether that address is zero.

10. **Repeated record-field reads.**  
    If the presence test and a later offset-8 load both execute, each reads the field independently. Acceptance must not require the later operation to reuse the tested value.

11. **Retained candidate.**  
    If a candidate is copied into the retained result and no intervening operation replaces that result, a subsequently completed return yields the copied word. This does not validate the word as an address.

## Uncertainties

The following remain **documented_only**:

- External-word values and stability requirements.
- Resolved entry-to-return paths, reachability, and termination.
- The complete valid argument domain.
- Table size, allocation bounds, and permitted signed indices.
- Record-field meanings and byte-storage lifetimes.
- Byte-sequence format, length, termination, and comparison relation.
- Candidate-selection policy and the meaning of zero results.
- Whether every nonzero result identifies valid record storage.
- Target-specific signed overflow, narrowing, character signedness, shifts, alignment, and byte order.
- Fault behavior and concurrent-mutation guarantees.

These unknowns do not make the mandatory dispatch-read order unknown. Nor do they permit assuming that separate scale reads return the same value.

## Evidence

This contract is based solely on static inspection of the supplied source representation. The prior draft and feedback were not treated as independent behavioral authority.

Established evidence includes the 32-bit interface, zero-initialized retained result, 42 accepted selectors, unconditional selector-offset/fallback read sequence, accepted-dispatch index snapshot and default-transition read, 184-byte record stride, field offsets 8 and 12, separately scaled paired reads, byte masks and multipliers, candidate assignments, and absence of explicit calls or external-memory writes.

The multiplier product is congruent to 71 modulo 256, not 1. Lookup or matching intent remains an inference.

No runtime tests have been performed. External constant values and additional implementation artifacts were not available.
