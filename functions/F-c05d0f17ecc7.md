<!-- Trusted provenance: F-c05d0f17ecc7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c05d0f17ecc7

## Purpose

F-c05d0f17ecc7 scans an initial sequence of classified bytes, consumes at most one sign byte, delegates conversion, conditionally corrects an output position, and adjusts a 32-bit result according to the consumed sign. Certain result patterns cause saturation and a 32-bit error-code store of **2**.

The apparent intent is signed textual conversion with leading-character skipping and range handling. That interpretation is **inferred intent**, not a complete established conversion contract. The supplied body establishes the wrapper’s accesses, branches, argument forwarding, arithmetic, and stores. It does not establish the accepted numeric grammar, the meaning of the third argument, or the behavior of the opaque dependencies.

Output-position correction is an intermediate operation, not a guarantee about the slot’s final contents. A later error-state dependency may modify that slot, and the subsequent error-code store may overwrite it if the returned error-state address aliases it.

## Interface

### Inputs

1. **Original input address**
   - Addresses a sequence of 8-bit bytes.
   - The wrapper reads successive bytes until it encounters zero or a nonzero byte whose classification has bit 0 clear.
   - There is no length argument, local bounds check, or null-address check.
   - Accessible storage is required for every byte inspected locally and for any additional accesses made by dependencies.

2. **Optional output-position address**
   - Either null or the address of a readable and writable 32-bit slot.
   - The slot holds an input address in the apparent 32-bit target model.
   - Its address is forwarded unchanged to the conversion dependency.
   - When non-null, the wrapper reads the slot after conversion and conditionally overwrites it.
   - Whether conversion initializes the slot is unknown. If it does not, the wrapper still attempts its post-call read.

3. **Opaque 32-bit conversion argument**
   - Forwarded unchanged to the conversion dependency.
   - Not validated or interpreted locally.
   - A specific meaning such as radix or conversion mode is not established.

### Result and representation

The result is a signed 32-bit integer. Relevant limits are:

- Minimum: **−2,147,483,648**.
- Maximum: **2,147,483,647**.
- Modulus for the observed 32-bit negation: **4,294,967,296**.

Input elements and classification entries are 8-bit quantities. The classification-table base and output-position value occupy 32-bit words used as addresses. These uses establish an apparent 32-bit address model; applicability to wider pointer representations is **documented_only**.

### Preconditions and failure boundaries

For the locally described accesses to complete, input bytes, classification metadata, and any non-null output-position slot must remain accessible when accessed. On saturation paths, the error-state dependency must return an address permitting a 32-bit store.

The wrapper does not validate these conditions, detect address wraparound, or provide local recovery from invalid memory accesses. It does not establish how dependency failures are reported.

## Behavior

### Initial scanning

Scanning begins at the original input address.

For each candidate, the wrapper reads one byte and advances its tentative position by one byte. A zero candidate stops scanning immediately, without a classification dependency call.

For every nonzero candidate, the wrapper:

- Calls **F-2a4c402ba617** without arguments.
- Reads a 32-bit classification-table base through the returned address.
- Reads one classification byte at that base plus the candidate’s unsigned value.
- Continues scanning only when bit 0 of that classification byte is set.

The candidate is zero-extended from 8 bits. Classification indices therefore range from **1 through 255**; index zero is not accessed by this scan.

Each examined nonzero byte causes a separate dependency call, including the byte that ends scanning because its classification bit is clear. Neither the returned metadata address nor the table base is locally cached across candidates. Changes made by dependency calls can therefore affect later classifications.

The wrapper does not establish a semantic name for the classified category. In particular, treating it as whitespace would require information not supplied here.

The **stopping byte** is the first zero byte or the first nonzero byte classified with bit 0 clear.

### Sign recognition and delegated position

The stopping byte determines whether one sign is consumed:

- **0x2B**: consume the byte and retain nonnegative adjustment mode.
- **0x2D**: consume the byte and select negative adjustment mode.
- Any other value: do not consume the stopping byte and retain nonnegative adjustment mode.

The resulting address is the **delegated input position**.

Sign recognition uses the stopping byte’s already-read value. The wrapper does not reread that byte after classification.

Classification takes precedence over sign recognition. A plus or minus whose classification bit 0 is set is skipped during scanning; it does not select an adjustment mode at that point. A later stopping byte determines whether a sign is consumed.

Only one stopping sign is consumed locally. The suffix is not rescanned. A second sign, a classified byte following the sign, or a zero immediately following the sign is passed directly to conversion.

When zero ends scanning, the tentative one-byte advance is not retained: conversion receives the address of the zero byte, unless an earlier stopping sign has already ended scanning.

### Conversion call

The wrapper calls **F-eef3bf576bc2** exactly once after scanning completes, with these arguments in order:

1. Delegated input position.
2. Original optional output-position address.
3. Unchanged opaque 32-bit conversion argument.

Its 32-bit return value is retained for subsequent adjustment.

There is no local check for missing digits, an empty suffix, unsupported syntax, an invalid third argument, or a dependency-reported error. The wrapper’s later decisions use the returned value and, when present, the output-position slot; they do not independently validate a conversion.

### Output-position correction

After conversion returns, the wrapper handles the optional slot:

- If its address is null, no local slot read or correction occurs.
- Otherwise, the wrapper reads its 32-bit value and compares that value, as an address, with the delegated input position.
- If equal, the wrapper stores the original input address into the slot.
- If unequal, the wrapper performs no output-position correction **at this stage**.

This comparison is the entire local correction criterion. The returned integer does not participate.

Equality therefore causes the reported position to move back over any locally skipped prefix and consumed sign. This resembles a no-conversion correction, but the dependency’s interpretation of that position is unknown.

If the original and delegated input addresses are identical, equality still causes a store of the original address. The store is not omitted merely because its value equals the slot’s existing value.

This stage precedes result adjustment and any error-state dependency call. Neither the equality nor inequality case guarantees the slot’s final contents: subsequent dependency effects and an aliasing error-code store remain possible.

### Nonnegative adjustment mode

This mode applies whenever no stopping minus was consumed, including when a stopping plus was consumed.

Interpreting the conversion result as signed 32-bit:

- **0 through 2,147,483,647**: return unchanged.
- **−2,147,483,648 through −1**: call **F-4af36158f863**, store the 32-bit value **2** through its returned address, and return **2,147,483,647**.

The decision depends on the result’s sign bit. It is not an independent assessment of the input’s magnitude or validity.

### Negative adjustment mode

The wrapper forms the conversion result’s 32-bit negation and interprets that bit pattern as signed.

Saturation occurs only when both conditions hold:

- The original conversion result is nonzero.
- Its 32-bit negation is nonnegative.

On that path, the wrapper calls **F-4af36158f863**, stores the 32-bit value **2** through its returned address, and returns **−2,147,483,648**.

Otherwise, it returns the negated value without a local error-code store.

The resulting boundaries are:

- **0** becomes **0**.
- **1 through 2,147,483,647** become their mathematical negatives.
- **−2,147,483,648** retains its bit pattern and is returned as **−2,147,483,648**, without local saturation.
- **−2,147,483,647 through −1** cause saturation to **−2,147,483,648**.

The minimum-value case requires 32-bit treatment. Unbounded mathematical negation would produce a positive value and would give the wrong saturation decision. The described behavior is the supplied body’s apparent 32-bit target behavior, not a claim about portable signed-overflow semantics in another implementation environment.

## Side effects

### Direct accesses

The wrapper directly:

- Reads input bytes during initial scanning.
- Reads a 32-bit table base and one classification byte for each nonzero candidate.
- Reads a non-null output-position slot after conversion.
- Conditionally stores the original input address into that slot.
- Conditionally stores the 32-bit value **2** through an error-state address.

It contains no direct input-byte store and no direct allocation or release operation.

### Observable order

On completing paths, the local sequence is:

1. Input scanning, interleaved with per-candidate classification calls and table reads.
2. One conversion call.
3. Optional output-position read and equality-based correction.
4. Result adjustment.
5. If saturation is selected, one error-state call followed by one 32-bit error-code store.
6. Return.

There is at most one local error-state call and one corresponding local error-code store per invocation.

On paths without that store, the wrapper does not directly clear or otherwise write error state. This does not imply unchanged error state: the classification and conversion dependencies may have their own effects.

### Aliasing and final slot contents

No local non-aliasing requirement is checked. In particular:

- The error-state dependency may modify the output-position slot during its call.
- Its returned address may equal the output-position address.
- If those addresses are equal, the wrapper’s later 32-bit store replaces the output-position value with **2**, whether or not an earlier position correction occurred.
- Other overlaps must be understood according to the actual ordered accesses; no broader safe-aliasing guarantee is established.

Thus, a position unequal to the delegated input position is merely exempt from the equality-based correction. It is not protected from later changes.

### Ownership and lifetime

No ownership transfer, retention guarantee, or lifetime extension is established for any argument or dependency-returned address.

The wrapper does not directly release storage. Dependency allocation, release, retention, and mutation behavior remain unknown. Dependency calls must not be assumed to preserve globals, input storage, classification metadata, or output storage beyond what their missing contracts would establish.

## Dependencies

| Opaque function ID | Established use | Unresolved contract |
|---|---|---|
| **F-2a4c402ba617** | Called without arguments for every examined nonzero scan candidate. The returned address supplies a 32-bit table base used for one unsigned-byte-indexed classification read. | Category meaning, table extent, address validity, stability, concurrency, memory effects, ownership, and lifetime. |
| **F-eef3bf576bc2** | Called once with delegated input, the original optional output-position address, and the opaque 32-bit argument. Returns the value adjusted by the wrapper. | Grammar, third-argument meaning, overflow policy, position initialization and updates, errors, memory effects, ownership, and retention. |
| **F-4af36158f863** | Called on local saturation paths, after position correction. Its returned address receives a 32-bit store of **2**. | Storage scope, meaning of code 2, address validity, aliasing, lifetime, and other call effects, including possible output-slot mutation. |

## Acceptance examples

These are conditional deductions from static evidence, not executed tests. They assume accessible storage and the stated dependency behavior.

### Scanning and position behavior

1. **Zero at the original input address**
   - No classification call occurs.
   - Conversion receives the original address.
   - If the non-null output slot equals that address after conversion, the wrapper stores the original address back into it.

2. **Two classified bytes followed by an ordinary unclassified nonzero byte**
   - Three classification calls occur.
   - Conversion receives the third byte’s address.
   - If the output slot equals that address after conversion, the correction stage replaces it with the original input address.

3. **Classified prefix followed by an unclassified plus**
   - The plus is consumed.
   - Conversion starts immediately after it.
   - A conversion result of **17** yields **17**.
   - A slot equal to the delegated position is corrected to the original input address.

4. **Minus with classification bit 0 set**
   - That minus is skipped as a classified byte.
   - It does not itself select negative adjustment mode.
   - A later stopping byte controls sign recognition.

5. **Output position differs from delegated input**
   - Suppose conversion stores a later input address in the slot.
   - The wrapper performs no equality-based correction at that stage, regardless of the conversion result.
   - If saturation follows, the error-state call may modify the slot.
   - If the returned error-state address equals the slot address, the subsequent local store leaves **2** in that slot, assuming the store completes.

6. **Correction followed by an aliasing error-code store**
   - Suppose conversion leaves the slot equal to the delegated input position and returns **−1** in nonnegative mode.
   - The wrapper first stores the original input address into the slot.
   - It then calls the error-state dependency.
   - If that dependency returns the slot’s address, the wrapper overwrites the slot with **2** and returns **2,147,483,647**.

7. **Null output-position address**
   - Null is forwarded unchanged to conversion.
   - The wrapper performs no output-position read or correction.
   - This does not constrain conversion’s handling of null or the effects of other dependencies.

### Numeric adjustment

| Adjustment mode | Conversion result | Wrapper return | Local error-code store |
|---|---:|---:|---|
| No minus | 0 | 0 | None |
| No minus | 2,147,483,647 | 2,147,483,647 | None |
| No minus | −1 | 2,147,483,647 | 2 |
| No minus | −2,147,483,648 | 2,147,483,647 | 2 |
| Minus | 0 | 0 | None |
| Minus | 17 | −17 | None |
| Minus | 2,147,483,647 | −2,147,483,647 | None |
| Minus | −2,147,483,648 | −2,147,483,648 | None |
| Minus | −2,147,483,647 | −2,147,483,648 | 2 |
| Minus | −1 | −2,147,483,648 | 2 |

“None” means no direct error-code store by this wrapper. It does not constrain dependency effects. “2” identifies the stored value, not a distinct or non-aliasing storage location.

## Uncertainties

The following remain **documented_only**:

- Semantic meaning of classification bit 0.
- Conversion grammar and third-argument semantics.
- Acceptance of additional signs, prefixes, or leading classified bytes by conversion.
- Whether a zero result denotes success, failure, or either.
- Whether conversion initializes every non-null output-position slot.
- Dependency error conventions and the external meaning of error code 2.
- Error-state storage scope and possible aliasing with caller-visible storage.
- Final output-slot contents after saturation without further dependency and aliasing constraints.
- Classification-table stability, locale association, and concurrency guarantees.
- Dependency mutation, allocation, release, ownership, and retention behavior.
- Behavior for invalid addresses, address wraparound, inaccessible tables, or unsuitable overlaps.
- Applicability outside the apparent 32-bit target model.

The inferred signed-conversion purpose does not resolve these unknowns.

## Evidence

The supplied static body establishes the byte-scan order, unsigned classification indexing, per-candidate dependency calls, sign-byte recognition, conversion arguments, equality-based position correction, result-adjustment branches, error-code stores, and local access ordering.

In particular, the error-state call and store occur after output-position correction. Nothing in the supplied body establishes disjoint storage or prevents that dependency from modifying the output slot. The static evidence therefore supports only a stage-specific absence of correction when positions differ, not preservation of the slot through return.

Dependency bodies were not supplied. Their grammar, ownership rules, global effects, and storage guarantees are not established.

No runtime tests have been performed. All acceptance examples are conditional static deductions, not observed executions.
