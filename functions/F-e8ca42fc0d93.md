<!-- Trusted provenance: F-e8ca42fc0d93; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e8ca42fc0d93

## Purpose

F-e8ca42fc0d93 represents a sequence of indirect dependency invocations, byte transfers, indexed 32-bit stores, and clearing operations. Values returned by dependencies determine later arguments, addresses, and stored data.

No application-level purpose is established. The evidence does not establish allocation, initialization, serialization, destruction, or device control.

The supplied representation reports invalid instruction data and ends at a truncated control-flow boundary. This contract describes the represented sequence, conditional on execution reaching each operation. It does not establish that the complete sequence is executable or that the function returns normally.

## Interface

### Explicit inputs

| Input | Width and interpretation | Established use |
|---|---|---|
| First input | Unsigned 32-bit value | Logically shifted right by three; the reduced value is supplied to several dependencies. |
| Second input | Signed 32-bit value | Shifted left by three; that expression supplies the first two arguments of the initial dependency. |

The second input has no subsequent explicit use.

### Additional entry state

The represented behavior also requires:

- An incoming base address, whose byte at offset `0x01` supplies the first selectors.
- An incoming eight-bit value for the first byte store.
- An incoming control-address value; adding `0xffe` determines the initial indirect target.
- Additional register outputs associated with dependency invocations.

Their ABI, initialization requirements, and relationship to the explicit inputs are **documented_only**.

### Outputs and completion

No return value or ordinary return is represented. Outputs are represented memory-access events and dependency invocations, together with whatever effects those dependencies actually produce.

The final boundary does not establish a particular runtime exception, successful return, or continuation. Those outcomes remain **documented_only**.

### Preconditions

Each reached access requires an address valid for its width, permissions, and platform alignment rules. Each reached call requires a valid executable target and compatible machine state.

The visible body contains no explicit null, bounds, allocation-failure, or dependency-error checks. These conditions are not validated before use.

## Behavior

### Conventions and dependency notation

A **word** is 32 bits; a **byte** is eight bits. Byte stores retain the low eight bits. Unsigned right shifts are logical.

Arithmetic consequences below use a **conditional 32-bit bit-vector model**: additions, multiplications, and left shifts retain the low 32 bits. Some represented expressions are signed; this model does not establish that their overflow or negative left shifts would be defined in a high-level language.

A **selector store through a base** reads the unsigned byte at base plus `0x01`, then writes a byte at absolute address selector plus `0x11`. The destination is not relative to the source base. Its numerical range is `0x11` through `0x110`.

The opaque dependency IDs identify call occurrences, not proven distinct target functions. A “four-call interval D-nn” identifies four ordered occurrences, D-nn.1 through D-nn.4. Their explicit argument expressions are absent from the representation; this does **not** establish that their machine-level argument lists are empty. Where specified, the fourth occurrence supplies the retained result.

All described ordering is conditional on preceding operations completing. Retained scalar values are those retained by the representation; this does not independently establish ABI preservation rules.

### 1. Entry update and initial word-to-byte transfers

The entry selector store writes the incoming byte. Let the reduced first input be **Q**, its original unsigned value shifted right by three.

D-01 receives, in order, the second input shifted left by three, the same expression again, and Q. Four-call interval D-02 follows, retaining its final word result. The entry selector is then read afresh and its newly selected destination receives zero. D-03 receives the retained result shifted left by two.

Four-call interval D-04 produces a result whose fourfold value is retained as **B**. The word at B plus `0x74` is read and its low byte stored at B plus `0x01`. That word is read again, separately, for D-05, whose arguments are the newly read word, Q, and Q.

D-06 supplies a word forwarded as D-07’s sole explicit argument. Four-call interval D-08 then produces another result. Its fourfold value supplies a second word-to-byte transfer: read at offset `0x74`, write the low byte at offset `0x01`. D-09 receives this second scaled value.

The two reads associated with B remain separate events. Their represented numerical addresses match; the intervening byte store has a different numerical address.

### 2. High-field derivation and initial indexed stores

Four-call interval D-10 follows D-09. An additional register output associated with its final invocation supplies a retained value **E**.

Define:

- **G**: B logically shifted right by 25, retaining B’s bits 25–31.
- **Expansion**: G shifted left by 23.
- **R**: Expansion logically shifted right by three.

A word at four times E plus `0x14` receives G shifted left by 19. D-11 receives E. D-12 produces a represented 64-bit result; D-13 receives its low word, high word, and low word again, in that order.

Four-call interval D-14 supplies an index. Its fourfold address plus `0x14` receives E shifted left by three. D-15 receives four times that index.

After four-call interval D-16, a word at R multiplied by `0x200000`, plus `0x14`, receives R multiplied by `0x80000`. D-17 receives three copies of that stored product.

Four-call interval D-18 supplies another index. Its fourfold address plus `0x14` receives R, and D-19 receives four times the index.

Under the stated wrapping model, R equals G multiplied by `0x100000`. Both products used by the R-derived store wrap to zero. Consequently, that store writes zero at absolute address `0x14`, and D-17 receives three zero arguments.

### 3. Retained result and selector propagation

Four-call interval D-20 produces an unsigned result **V**, retained together with its fourfold value **W**.

A selector store through R writes the low byte of W. D-21 then receives:

1. That selector shifted right by three.
2. W.
3. R shifted left by two.

Four-call interval D-22 supplies a word retained for forwarding. A fresh selector read through R supplies **S**. Its destination receives the low byte of V masked with `0x3fffffff` and shifted right by 20. D-23 receives the D-22 result shifted left by two.

After four-call interval D-24, the word at four times S plus `0x14` receives W. D-25 receives S, S, S, and S shifted right by one.

Four-call interval D-26 supplies an index whose fourfold address plus `0x14` receives W. D-27 receives four times that index.

The masked right-shift expression contains up to ten bits before narrowing. The byte store retains only V’s bits 20–27; bits 28–29 do not survive.

### 4. Indexed propagation before context stores

This portion distinguishes fourfold destination indexing from sixteenfold destination indexing.

- Four-call interval D-28 supplies an index. Its sixteenfold address plus `0x14` receives its fourfold value. D-29 receives that fourfold value. Four-call interval D-30 supplies a destination index; its fourfold address plus `0x14` receives the retained D-28 fourfold value. D-31 receives four times the destination index.
- Four-call interval D-32 supplies another index. Its sixteenfold address plus `0x14` receives its fourfold value. D-33 receives that value. D-34 then supplies a word forwarded to D-35. Four-call interval D-36 supplies a destination index; its fourfold address plus `0x14` receives the retained D-32 fourfold value, masked with `0x3fffffff` and shifted right by one. D-37 receives four times this destination index.
- Four-call interval D-38 supplies another index. Its sixteenfold address plus `0x14` receives its fourfold value. The word at its **fourfold** address plus `0x14` is separately read, logically shifted right by three, and supplied to D-39.
- Four-call interval D-40 supplies a destination index whose fourfold address plus `0x14` receives the retained D-38 fourfold value. D-41 receives four times that destination index.

The D-38 write and the read supplied to D-39 generally address different locations. They must not be treated as a write followed by a readback.

### 5. Context fields and context-selector updates

Four-call interval D-42 produces an unsigned **context result C**. Before replacing the earlier selector-derived arithmetic state, the function stores S as a word at:

\[
(S \mathbin{\text{logical-right-shift}} 3)\times 0x80000 + 0x24.
\]

The **context base** is sixteen times C. The retained **half-field H** is C masked with `0x3fffffff`, then logically shifted right by one.

D-43 receives the context base and C shifted left by two. After four-call interval D-44, the word at context base plus `0x24` receives S. The word at context base plus `0x74` is read and supplied to D-45.

Four-call interval D-46 produces a retained result **J**. A selector store through the context base writes the low byte of C masked with `0x3fffffff` and shifted right by 20. D-47 receives J shifted left by two. D-48 supplies a word forwarded to D-49.

Four-call interval D-50 produces another result. The context selector is read again and its selected destination cleared. D-51 receives the D-50 result shifted left by two. Four-call interval D-52 follows.

J remains the result from D-46; it is not replaced by the D-50 result.

### 6. Half-field clearing and first absolute-state clear

The byte at H plus `0x0f` is cleared. Define **K** as H multiplied by `0x400`. A selector store through K clears its selected destination.

D-53 then receives four explicit arguments:

1. J shifted left by 23.
2. J shifted left by 23.
3. J shifted left by 23.
4. J shifted left by 25.

D-54 supplies a word forwarded to D-55. Four-call interval D-56 supplies another result. A fresh selector read through K is retained as **T**, and its selected destination is cleared. D-57 receives the D-56 result shifted left by two.

After four-call interval D-58, the word at T plus `0x44` is cleared, followed by a word clear at absolute address `0x24`. D-59 receives, in order, zero, `0x98`, K, and `0x98`.

Thus, D-53 occurs after the first K-based selector clear but before its repeated selector read. D-59 occurs after both the T-based word clear and the first absolute-`0x24` clear.

### 7. Remaining offset-`0x44` clears and register-output dependency

D-60 supplies a word forwarded to D-61. D-62 supplies a further word forwarded to D-63. These are two distinct forwarding pairs, in that order.

Four-call interval D-64 supplies an index. The word at four times that index plus `0x44` is cleared, followed by the second absolute-`0x24` word clear. D-65 receives four times the index.

Four-call interval D-66 produces a represented 64-bit result. D-67 receives its low word shifted left by two. Only after that invocation does the function clear the word at twice the retained high word plus `0x44`, then perform the third absolute-`0x24` word clear.

An additional register output associated with D-67 is retained as **U**. D-68 receives zero, U, and U shifted left by two. U is not established to equal either word of the 64-bit result.

D-69 is a further call without explicit argument expressions. D-70 receives H masked with `0x3fffff` and shifted left by seven.

Four-call interval D-71 supplies an index. Its fourfold address plus `0x44` is cleared, followed by the fourth absolute-`0x24` word clear. D-72 receives four times that index.

The high-word-derived address is scaled by two, not four. Depending on the high word, it can be two-byte aligned without being four-byte aligned.

### 8. Final offset-`0x14` propagation

Four-call interval D-73 produces a word. D-74 receives that word shifted left by two and returns a new result **Z**. It is Z—not the preceding D-73 result—that determines the next zero store at four times Z plus `0x14`.

D-75 receives Z, Z, Z, and H masked with `0x3fffff` and shifted left by **seven**.

Four-call interval D-76 supplies an index whose fourfold address plus `0x14` is cleared. D-77 receives four times that index.

Four-call interval D-78 supplies a new index. Its sixteenfold address plus `0x14` receives its fourfold value. D-79 receives that value. Four-call interval D-80 supplies a destination index; its fourfold address plus `0x14` receives the retained D-78 fourfold value. D-81 receives four times the destination index.

Four-call interval D-82 supplies another index. Its sixteenfold address plus `0x14` receives its fourfold value. D-83 receives both four times and sixteen times that same index, in that order.

D-84 supplies a word. D-85 receives that word followed by H masked with `0x3fffff` and shifted left by four.

Four-call interval D-86 supplies an index. Its fourfold address plus `0x14` receives H masked with `0x3fffff`, multiplied by `0x80`. D-87 receives four times the index.

### 9. Final selector updates and boundary

Four-call interval D-88 supplies an unsigned result **A**. Define the final selector base as H multiplied by `0x4000000`; under wrapping arithmetic, only H’s low six bits affect that base.

A selector store through this base writes the low byte of A shifted left by two. D-89 receives A shifted left by two. D-90 supplies a word forwarded to D-91.

Four-call interval D-92 supplies another result. A fresh selector store through the same base writes A’s bits 20–27. D-93 receives the D-92 result shifted left by two.

Four-call interval D-94 supplies an unsigned result **B-final**. A fresh selector read through the final base selects the destination for the low byte of four times B-final. D-95 receives this selector, zero-extended to a word.

D-96 supplies a word forwarded to D-97. Four-call interval D-98 supplies another result.

The final selector store uses a different source base: eight times B-final. It writes B-final’s bits 20–27. D-99 receives the D-98 result shifted left by two.

Four-call interval D-100 follows. The representation then terminates at its invalid-instruction boundary.

## Side effects

### Memory layout

The representation establishes access patterns, not complete objects:

| Location pattern | Width | Represented role |
|---|---:|---|
| Base plus `0x01` | Byte | Selector read or narrowed-value store |
| Selector plus `0x11` | Byte | Absolute selector-derived destination |
| H plus `0x0f` | Byte | Clearing |
| Scaled index plus `0x14` | Word | Propagation or clearing |
| Context-related address plus `0x24` | Word | Retained-selector store |
| Computed address plus `0x44` | Word | Clearing |
| Scaled base plus `0x74` | Word | Read |
| Absolute `0x24` | Word | Four separately ordered clears |

No general disjointness, object bounds, synchronization, or memory classification is established. Dependencies may access the same memory. Repeated selector reads must not be replaced by an earlier selector value.

### Ownership, errors, and persistence

No visible operation establishes allocation, ownership transfer, release, reference counting, or lifetime extension. Argument passage alone does not establish whether a value is a pointer, handle, scalar, or owned resource.

No explicit dependency failure result is checked. No explicit rollback or recovery path appears in the visible body.

A represented store event is not a guarantee of persistent final contents or external observability. A later dependency may overwrite or reverse its effects, including before failing or not returning. The same limitation applies to effects of earlier dependencies.

## Dependencies

D-01 through D-100 and the expanded four-call occurrence IDs are opaque labels for the ordered invocations described above. They do not assert target equality or inequality.

Established dependency information includes:

- Explicit argument values and their order.
- Forwarding of selected word results.
- Splitting of two represented 64-bit results into low and high words.
- The retained additional register values E and U.
- The placement of calls between memory-access events.

Concrete target identities, complete signatures, argument-register contents where expressions are absent, clobbers, side effects, memory ownership, and failure behavior remain **documented_only**.

No dependency is assumed to preserve globals, leave selectors unchanged, return normally, or refrain from modifying earlier stores.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Input reduction.** First inputs `0` and `7` both supply Q equal to zero; input `8` supplies Q equal to one.

2. **Absolute selector destination.** An entry selector of `0xff` selects address `0x110`. With incoming byte `0xab`, the first represented store writes that byte there, provided the access succeeds.

3. **Fresh selectors.** If a dependency changes a selector between two represented reads, the second destination follows the second read. Clearing the first destination unconditionally is not equivalent.

4. **Word narrowing.** Reading numerical word `0x123456ab` for a word-to-byte transfer produces byte `0xab`, independently of unspecified physical byte order.

5. **High-field narrowing.** A retained value `0x3ff00000` produces masked right-shift value `0x3ff`, but its selector store writes only `0xff`.

6. **Wrapping product.** Under 32-bit wrapping arithmetic, both R-derived products in phase 2 are zero for every possible B. The store targets `0x14`, and D-17 receives three zero arguments.

7. **Distinct indexed locations.** For index one, a sixteenfold-indexed store at offset `0x14` targets `0x24`; the corresponding fourfold-indexed read targets `0x18`.

8. **Retained-result arguments.** If J is one under the bit-vector model, D-53 receives `0x00800000` three times, then `0x02000000`. Those arguments derive from J, not H or a newer dependency result.

9. **Intervening call before clearing.** If D-66 returns high word one and low word three, D-67 receives `12` before the word clear at `0x46` and the third absolute-`0x24` clear. D-68 subsequently uses U from D-67, not an assumed copy of either returned word.

10. **Transformed zero-store index.** If D-73 produces three but D-74 returns five, D-74 receives `12`; the following clear targets `0x28`. D-75 receives three copies of five and the specified seven-bit-shifted half-field expression.

11. **Partial execution and persistence.** If a dependency does not return, later body operations are not required to occur. Earlier represented events are not thereby undone by an explicit body rollback, but their resulting contents need not persist: the dependency itself may have changed them.

## Uncertainties

The following remain **documented_only**:

- Whether the supplied representation accurately describes executable control flow.
- Complete calling conventions and hidden entry-state requirements.
- Dependency contracts, target identities, register-output interpretation, and abnormal outcomes.
- Endianness, address-space mappings, permissions, alignment support, and physical aliasing.
- Exact signed-expression overflow and shift semantics.
- Ownership, lifetime, concurrency, synchronization, and external observability.
- Any ordinary return or behavior beyond the truncated boundary.
- Application-level meaning of the fields and transformations.

The repeated clears may suggest reset-related intent. That interpretation is not an established guarantee.

## Evidence

Established static evidence consists of the supplied representation’s explicit widths, arithmetic expressions, memory accesses, dependency arguments, retained results, register-output uses, occurrence ordering, and invalid-instruction warnings.

Wrapping simplifications and numerical acceptance examples are conditional deductions under the stated arithmetic model. Application purpose and executable completeness are not established.

No implementation access, dependency inspection, runtime execution, or runtime tests have been performed. No snapshot identifier is asserted here.
