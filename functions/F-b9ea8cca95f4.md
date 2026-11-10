<!-- Trusted provenance: F-b9ea8cca95f4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b9ea8cca95f4

## Purpose

F-b9ea8cca95f4 dispatches among states selected by global control values. Some states call F-1661479163f4 or F-a1e5ead1c5b9 using values read from a caller-supplied object and word array. Dependency results determine subsequent state addresses through a 32-bit arithmetic predicate. A separate state updates the low word of the eventual return value.

**Established static behavior:** The supplied representation identifies the dispatch cases, memory accesses, dependency arguments, arithmetic, source-level sequencing, and return construction.

**Inferred intent:** The routine appears to coordinate conditional operations. No evidence establishes that those operations are validation, comparison, allocation, or any other application-level activity.

**Unknown:** Concrete dispatch-global values and dependency behavior are absent. There is no established unconditional call sequence, termination guarantee, or success/failure interpretation.

## Interface

### Inputs and layouts

The routine takes two inputs:

1. An object address represented by a 32-bit value.
2. An address of an array of 32-bit words.

Addresses stored in object fields are also represented as 32-bit values. A wider-address implementation would require an explicit layout decision; replacing these fields with native-width addresses would not preserve the observed layout.

The object has these potentially accessed fields:

| Byte offset | Width | Use |
|---|---:|---|
| +4 | 32 bits | Third argument to F-a1e5ead1c5b9 |
| +8 | 32 bits | Third argument to F-a1e5ead1c5b9 |
| +12 | 32 bits | Third argument to F-a1e5ead1c5b9 |
| +16 | 32 bits | Address of a secondary object |
| +20 | 32 bits | Third argument to F-a1e5ead1c5b9 |

The secondary object has potentially accessed 32-bit words at byte offsets +0, +4, and +8.

The array has potentially accessed elements 0 through 6, at four-byte intervals. Element 0 contributes to a scaling expression. Elements 1 through 6 supply first arguments to F-a1e5ead1c5b9.

These are conditional access requirements, not a claim that every execution reads every location. Object offset +0 is not directly read.

### Output

The declared result is 64 bits, constructed from:

- A low 32-bit accumulator initialized to zero.
- A high 32-bit stack-derived word with no visible initialization.

State 17 can replace the low accumulator. State 16 returns the two-word result.

The upper word is not established as zero, sign extension, a meaningful second result, or stable between calls. Consequently, the full result is not established as a defined 64-bit status value. This contract records the visible return construction without assigning meaning to the uninitialized component.

### Preconditions and errors

Each reached state requires valid storage for its actual reads. In particular:

- Object and array accesses require sufficient readable extent.
- A secondary-object access requires both a readable address field and readable storage at the resulting address.
- Storage accessed after a dependency call must remain valid at that later access.
- Both dependencies must support the observed argument shapes.

No visible null check, length check, address validation, or recoverable invalid-input path exists. Invalid accesses have no established recoverable outcome.

The routine initially retains null-equivalent working addresses. State 0 installs the supplied addresses. If dispatch reaches a memory-reading state before that installation, the routine does not substitute the original inputs automatically.

## Behavior

### Arithmetic and captured values

The observed data operations use 32-bit words. Products involving the captured multiplier, predicate operations, and successor selection are described by their 32-bit bit patterns, with arithmetic reduced modulo \(2^{32}\).

The representation also contains signed negation, addition, subtraction, and multiplication. This contract does not claim that a literal signed-language translation has defined overflow behavior in every case. The intended machine-level treatment of such overflow remains **documented_only**.

The global roles are:

- **Multiplier:** Captured once at entry.
- **Coefficient:** Read twice at entry.
- **Dispatch base:** Read twice during initialization and once for each dispatch.
- **Initial target:** Read in state 0.
- **Per-call targets:** Seven globals, one for each dependency-calling state.
- **Post-result target:** Read in state 17.

The coefficient observations must remain distinct:

- The **positive snapshot** is the first coefficient observation.
- The **negated snapshot** is the 32-bit negation of the second coefficient observation.

They are arithmetic opposites only if the two coefficient observations agree, subject to the arithmetic qualification above. Later changes to either source global do not refresh these retained snapshots or the captured multiplier.

### Initialization

The global reads occur in this order:

1. Capture the multiplier.
2. Capture the positive coefficient snapshot.
3. Read the coefficient again and form the negated snapshot.
4. Read the dispatch base and retain that value plus 17 as a cached fallback address.
5. Read the dispatch base again to obtain the initial state address.

The low accumulator and carried control word start at zero. The working object and array addresses start null-equivalent.

Because the dispatch base is read repeatedly, the initial state is not unconditionally state 0. That conclusion requires the initialization and first-dispatch base observations to agree.

### Dispatch mechanics

For each dispatch, the routine:

1. Retains the incoming carried control word.
2. Reads the current dispatch base and subtracts it from the current state address to obtain the dispatch index.
3. Sets the next carried control word to the captured multiplier.
4. Sets the next state address to the cached fallback address.
5. Applies the selected state's overrides or returns.

With a stable base, the initial index is 0 and the cached fallback selects index 17. If the base changes, the cached fallback is not recomputed.

The state address is a control value used in subtraction; the routine does not dereference it as an instruction or data address.

### States without dependency calls

| Index | Behavior |
|---|---|
| 0 | Read the initial target as the next state address, preserve the incoming control word, and install the original input addresses as the working addresses. |
| 3, 5, 6, 8, 11, 13, 15 | Keep the default next control word and cached fallback address. |
| 4 | Replace the next control word with zero; keep the cached fallback address. |
| 16 | Return immediately with the current low accumulator and uninitialized upper word. |
| 17 | Replace the low accumulator with the negated snapshot multiplied by the incoming control word; preserve that control word; read the post-result target as the next address. |
| Any other index | Keep the default next control word and cached fallback address. |

Reentering state 0 reinstalls the original addresses but does not reset the low accumulator. Reentering state 17 replaces, rather than accumulates into, the low result.

With a stable base, default states lead to state 17. This does not guarantee return: subsequent targets must eventually select state 16.

### Dependency arguments

For every F-a1e5ead1c5b9 call, the second argument is:

\[
\text{negated snapshot}
\times \text{captured multiplier}
\times \text{current array element 0}
\pmod{2^{32}}.
\]

The calls and successor offsets are:

| State | Dependency | Arguments, in positional order | Adjusted-target offset |
|---|---|---|---:|
| 1 | F-1661479163f4 | Literal 1; secondary-object word at +8 | +11 |
| 2 | F-a1e5ead1c5b9 | Array element 1; scaled element 0; object word at +12 | −4 |
| 7 | F-a1e5ead1c5b9 | Array element 5; scaled element 0; secondary-object word at +0 | −5 |
| 9 | F-a1e5ead1c5b9 | Array element 2; scaled element 0; object word at +4 | −6 |
| 10 | F-a1e5ead1c5b9 | Array element 6; scaled element 0; secondary-object word at +4 | −6 |
| 12 | F-a1e5ead1c5b9 | Array element 3; scaled element 0; object word at +8 | +11 |
| 14 | F-a1e5ead1c5b9 | Array element 4; scaled element 0; object word at +20 | +10 |

States may be skipped or revisited. No single-pass traversal is established.

Every secondary-object argument requires a fresh read of object offset +16. The secondary address is not cached across calls. Likewise, array element 0 is read anew for each F-a1e5ead1c5b9 invocation.

The source establishes that argument-producing reads precede the associated call, but not the relative evaluation order of independent argument expressions. Within a nested access, the secondary address must be obtained before its field can be read.

### Dependency result and successor selection

Each dependency-calling state follows this source order:

1. The dependency returns a 32-bit result.
2. The state's transition-target global is read for the first time, and the state's offset is added to that observation.
3. The predicate is computed and its selector extracted.
4. The same transition-target global is read a second time.
5. The successor address is calculated from the adjusted first observation, selector, and unadjusted second observation.

The first target read and offset addition therefore precede predicate computation. The second target read follows predicate computation and selector extraction. This contract does not treat moving both target reads after the predicate as an equivalent ordering.

For dependency result \(r\), let:

\[
P =
\operatorname{NOT}\left(
(\text{negated snapshot}\times\text{multiplier}\times r)
\operatorname{OR}
(\text{positive snapshot}\times\text{multiplier}\times r)
\right),
\]

where products and bitwise operations are 32-bit. The selector is bit 31 of \(P\), obtained by logical right shift, so it is exactly 0 or 1.

The next state address is:

- **Selector 0:** First target observation plus the state's offset.
- **Selector 1:** Second target observation without adjustment.

The full 32-bit value \(P\), not only its selector, becomes the carried control word for the next dispatch.

Both target reads occur for either selector. The successor calculation does not eliminate the unselected observation from the source-level sequence. The two reads must not be collapsed into one assumed-stable value.

If both coefficient observations equal \(c\), with multiplier \(m\), the selector equals 1 exactly when:

\[
c m r \equiv 0 \pmod{2^{32}}.
\]

This is a modular zero-product test, not necessarily a test for a zero dependency return. If the coefficient observations differ, the two-product predicate remains controlling; this simplification does not apply.

### Ordering and changing globals

A dependency can affect later reads if it changes relevant storage or globals. Its return precedes the first transition-target read, which precedes predicate computation, which precedes the second target read.

Later dispatches reread the dispatch base. Later dependency states reread their argument-producing memory.

These source-level observations do not establish synchronization, atomicity, volatile access semantics, or a concurrent-memory execution model. Such behavior remains unresolved rather than being used to justify reordering or global stability.

## Side effects

The routine contains no explicit caller-visible stores to the object, secondary object, array, or dispatch globals. Its explicit state updates affect local control values, working addresses, and the low return accumulator.

Calls to F-1661479163f4 and F-a1e5ead1c5b9 are observable operations. Their effects are unknown. They are not established to:

- Preserve globals or input storage.
- Preserve the allocation or lifetime of referenced storage.
- Avoid retaining supplied values.
- Avoid allocation, release, I/O, or other external effects.

The routine itself shows no allocation or release operation. That absence does not constrain dependency ownership behavior.

There is no visible rollback or cleanup path. Earlier dependency effects are not visibly reversed if execution later loops, encounters an invalid access, or fails to return.

## Dependencies

### F-1661479163f4

Observed interface:

- Two 32-bit arguments.
- First argument: literal 1.
- Second argument: the word at secondary-object offset +8.
- A 32-bit returned value used in the predicate.

Semantic purpose, error conventions, memory effects, ownership effects, and global preservation are **documented_only**.

### F-a1e5ead1c5b9

Observed interface:

- Three 32-bit arguments.
- First: an array word selected by the current state.
- Second: the scaled current array element 0.
- Third: an object or secondary-object word selected by the state.
- A 32-bit returned value used in the predicate.

The first argument is not established to be a pointer, handle, identifier, or length. Semantic types, error conventions, memory effects, ownership effects, and global preservation are **documented_only**.

Neither dependency has an established conventional success value. The routine visibly uses returned bit patterns without a separate error handler. Behavior when a dependency does not return normally is not established.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Return without dependency calls.**  
   If the base remains stable and state 0's target selects state 16, the routine returns with low word zero and an unspecified upper word. No dependency is invoked.

2. **Exact state-10 arguments.**  
   If state 10 is reached with valid working storage, F-a1e5ead1c5b9 receives array element 6, the scaled current element 0, and the word at secondary-object offset +4. The secondary address is freshly obtained from object offset +16.

3. **Required post-call sequence.**  
   After a state-9 dependency call returns, the routine first reads that state's target and subtracts 6. It then computes the predicate and selector, reads the target again, and calculates the successor. An observation trace placing the first target read after predicate computation does not match the supplied ordering.

4. **Different target observations.**  
   Suppose state 9's first and second target observations are \(A\) and \(B\). Selector 0 chooses \(A-6\); selector 1 chooses \(B\), using 32-bit address arithmetic. Both observations still occur in the stated sequence. No mechanism explaining differing observations is assumed.

5. **Zero dependency return.**  
   A zero dependency result makes both predicate products zero, even if the coefficient observations differ. The predicate is all one bits, the selector is 1, and the second target observation supplies the successor. This does not imply semantic success.

6. **Nonzero return with a zero product.**  
   With equal coefficient observations of \(65{,}536\), multiplier \(65{,}536\), and dependency return 1, the modular product is zero. The selector is 1 despite the nonzero return.

7. **Adjusted successor.**  
   With equal coefficient observations of 1, multiplier 1, and dependency return 1, the selector is 0. State 9 chooses its first target observation minus 6.

8. **Default-state result.**  
   With a stable base, state 3 sends the captured multiplier to state 17. If the post-result target selects state 16, the returned low word is the negated snapshot times the captured multiplier, modulo \(2^{32}\).

9. **Zero-forcing state.**  
   With a stable base, state 4 sends zero to state 17. That state writes a zero low accumulator. A subsequent direct return preserves that low word.

10. **Later memory mutation.**  
    If a dependency changes array element 0 and execution later reaches another F-a1e5ead1c5b9 state, the later call uses the changed element, provided the storage remains valid.

11. **No forced termination.**  
    Targets can select a cycle avoiding state 16. The routine contains no visible iteration bound or timeout.

## Uncertainties

The following remain **documented_only**:

- Concrete global values and intended stability.
- Reachability and actual dependency order.
- Whether supported executions terminate.
- Whether repeated coefficient, target, or base observations differ.
- Application meaning of the low result.
- Provenance and meaning of the upper result.
- Dependency error and ownership contracts.
- Alignment, endianness, and invalid-access behavior.
- Concurrency, synchronization, atomicity, and volatile-memory semantics.
- Exact machine-level treatment of signed arithmetic overflow in the supplied representation.

No retry, exception translation, input-rejection protocol, or cleanup guarantee is established.

## Evidence

The basis is the supplied static routine representation and opaque dependency mapping.

Established observations include field widths and offsets, argument selection, repeated global reads, dispatch behavior, local result updates, absence of explicit external-memory stores, and the two-word return construction.

For all seven dependency-calling states, the source places the first target read and offset addition after dependency return but before predicate computation. Selector extraction precedes the second target read; successor calculation follows it.

The zero-product interpretation is conditional mathematical reasoning over 32-bit words with equal coefficient observations. Stable-base dispatch conclusions likewise depend on the stated stability condition.

No runtime tests, dependency executions, memory inspections, or concrete global-value observations have been performed.
