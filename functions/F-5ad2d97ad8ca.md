<!-- Trusted provenance: F-5ad2d97ad8ca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5ad2d97ad8ca

## Purpose

F-5ad2d97ad8ca performs ordered validation of six 32-bit inputs and returns a signed 32-bit status. It checks a required nonzero value, a primary name-like value, a mode-dependent numeric selection, a secondary name-like value, and a flag-controlled additional restriction on the secondary value.

**Established static behavior:** rejection stops execution at the first reached failing stage. Explicit return values are 0, 4, 5, 7, 8, 9, and 11. Dependencies may be called repeatedly with the same argument values; those calls are distinct observations and cannot be collapsed without additional guarantees.

**Inferred intent:** the primary name appears intended to match “Standard Theta.” When the final flag is zero, the secondary name appears intended to match “STANDARD.” These interpretations require assumptions about reference contents and dependency semantics. The established contract is the call order and result handling below, not an unconditional string-validation promise.

## Interface

The function accepts six arguments. All have 32-bit representations; A4 is unsigned, while the others are presented as signed integers.

| Input | Established use |
|---|---|
| A1: required value | Tested for zero. Not dereferenced or supplied to a dependency. |
| A2: primary name-like value | Tested for zero, then supplied to comparison-like and length-like dependencies. Its use suggests an address. |
| A3: numeric selection | Checked using unsigned 32-bit subtraction and comparison. |
| A4: mode | Unsigned selector for the allowed numeric range. |
| A5: secondary name-like value | Tested for zero, then supplied to length-like, validation-like, and conditionally comparison-like dependencies. Its use suggests an address. |
| A6: restriction flag | Zero enables the final secondary-name restriction. Every nonzero value bypasses that restriction. |

The result is a signed 32-bit integer.

| Result | Local condition producing it |
|---|---|
| 0 | Every reached validation stage passes. |
| 4 | A1 is zero. |
| 5 | Primary validation passes, but A4 is not 0, 1, 3, or 4. |
| 7 | A2 is zero, or the primary dependency sequence rejects. |
| 8 | A4 is supported, but A3 fails its selected range check. |
| 9 | A5 is zero, its initial length-like result is zero, or its validation-like result is nonzero. |
| 11 | A6 is zero and the final secondary dependency sequence rejects. |

These are meanings established within this function, not definitions of a wider status-code namespace.

### Preconditions and failure boundary

For a normally returning execution, every reached dependency must accept its supplied arguments and return normally.

A nonzero A2 or A5 is not proof of an accessible address, suitable storage extent, or termination. The function performs no explicit capacity or address-validity check. Earlier rejection can prevent a dependency from receiving an otherwise invalid value.

A1 requires no accessible pointee for the visible operations: its numeric zero/nonzero state is the only property inspected.

No behavior is established for invalid memory access, dependency traps, exceptions, or nonreturning calls. No local recovery path for those events is visible.

## Behavior

### Reference preparation

Before any explicit return, the function prepares three stack-resident reference regions:

| Region | Visible layout | Subsequent use |
|---|---|---|
| Primary reference | Four consecutive four-byte portions, totaling 16 bytes | Address supplied with comparison count 14 |
| Validation reference | One four-byte value | Address supplied to the validation-like dependency |
| Secondary reference | Three consecutive four-byte portions, totaling 12 bytes | Address supplied with comparison count 8 |

The first 12 bytes of the primary reference are copied from storage labeled with “Standard Theta”; a further four-byte read supplies its continuation. The first eight bytes of the secondary reference are copied from storage labeled with “STANDARD”; another four-byte read supplies its continuation. The validation reference comes from a separate four-byte read.

The labels suggest text but do not independently establish all bytes, encoding, or termination. In particular, the supplied evidence does not expose the primary continuation bytes, the secondary continuation bytes, or the validation-reference value.

The visible preparation order is:

1. Populate the primary reference through four four-byte copies.
2. Populate the four-byte validation reference.
3. Evaluate whether A1 is zero and select the corresponding provisional status.
4. Populate the secondary reference through three four-byte copies.
5. Act on the provisional status.

Thus even the A1-zero path prepares all three references before returning. No dependency call precedes their preparation. The evidence establishes the displayed read-and-copy order, not atomicity or protection against external changes to the backing storage.

### Required-value stage

If A1 is zero, return 4. No dependency is called, and the remaining arguments do not affect this explicit result.

Otherwise, proceed to primary validation.

### Primary-name stage

If A2 is zero, return 7 without calling a dependency.

For nonzero A2, the stage has the following observable sequence:

1. Call F-7f14f1ab6851 with A2, the primary-reference address, and count 14.
2. If that result is zero, call F-1cb585febd84 with A2.
3. Interpret this length-like result as unsigned 32-bit. If it is greater than 14, return 7 immediately.
4. If execution has not returned, call F-7f14f1ab6851 again with A2, the same primary-reference address, and count 14.
5. If the second comparison-like result is nonzero, return 7. Otherwise, proceed.

The first nonzero comparison result is not itself a rejection. It suppresses the length-like call, after which the second comparison result decides the stage.

A first zero comparison followed by an unsigned length-like result greater than 14 rejects without a second comparison. A length-like result of 14 or less does not alone establish acceptance: the second comparison must still return zero.

There is no independent positive-length or minimum-length requirement here. A length-like result of zero can pass this stage if both required comparison results are zero.

Under ordinary bounded string-comparison and length semantics, with suitable reference bytes and unchanged relevant contents, this sequence would require an exact 14-character primary name. Those assumptions do not follow from this caller alone.

### Mode and numeric-selection stage

This stage occurs only after primary validation passes.

A4 selects the allowed range:

| A4 | Accepted A3 values |
|---|---|
| 0 or 1 | 1 through 10, inclusive |
| 3 or 4 | 1 through 14, inclusive |
| Any other unsigned 32-bit value | No accepted value; return 5 |

For modes 0 and 1, the function subtracts one from A3 using unsigned 32-bit arithmetic and requires the result to be strictly less than 10.

For modes 3 and 4, it performs the same subtraction and requires the result to be strictly less than 14.

Subtraction wraps modulo \(2^{32}\). These are not signed comparisons. Consequently, A3 equal to zero fails, every negative signed A3 fails, and values above the selected positive upper bound fail.

A supported mode with an out-of-range A3 returns 8. An unsupported mode returns 5 regardless of A3. Neither rejection reaches secondary-name dependencies.

### Initial secondary-name stage

If A5 is zero, return 9.

Otherwise:

1. Call F-1cb585febd84 with A5.
2. If its 32-bit result is zero, return 9.
3. Call F-3ec7d07fef90 with A5 and the validation-reference address.
4. If its result is nonzero, return 9.
5. Otherwise, proceed.

The initial secondary length-like result is tested only for equality with zero. There is no visible upper bound at this stage. Any nonzero 32-bit result, including one whose signed interpretation is negative, permits the validation-like call.

A zero result from F-3ec7d07fef90 establishes only that this local rejection condition did not occur. It does not establish that A5’s buffer, any reference storage, or any global state was preserved.

### Flag-controlled secondary-name stage

After initial secondary validation passes:

- If A6 is nonzero, return 0 immediately.
- If A6 is zero, perform the additional sequence below.

Positive and negative nonzero A6 values have identical local behavior.

The additional sequence is:

1. Call F-7f14f1ab6851 with A5, the secondary-reference address, and count 8.
2. If that result is zero, call F-1cb585febd84 with A5.
3. Interpret this new length-like result as unsigned 32-bit. If it is greater than 8, return 11 immediately.
4. Otherwise, call F-7f14f1ab6851 again with A5, the same secondary-reference address, and count 8.
5. Return 11 if the second comparison-like result is nonzero; otherwise return 0.

The initial secondary length-like result is not reused. The later call can observe different contents or state.

As in the primary stage, a first nonzero comparison result skips the intervening length-like call but does not itself reject. A second comparison is still required unless the intervening length check has already rejected.

With conventional string semantics and preserved relevant contents, the apparent intended restriction is an exact eight-character “STANDARD” match. Without preservation, even an initially matching secondary buffer can fail here after successful initial validation.

## Side effects

### Direct effects

The visible body reads backing reference storage and initializes local reference regions. It exposes their addresses to dependencies.

No explicit allocation, release, ownership transfer, write to caller-owned storage, or global write appears in the body. This does not make the function externally side-effect-free: all three dependencies have unspecified side effects.

The body does not inspect the validation-reference contents after F-3ec7d07fef90 returns. It also does not verify that any name or comparison reference remains unchanged.

### Observable call order

A complete execution can contain the following calls, in order:

1. Primary comparison.
2. Conditional primary length observation.
3. Second primary comparison.
4. Initial secondary length observation.
5. Secondary validation.
6. Conditional final secondary comparison.
7. Conditional final secondary length observation.
8. Conditional second final secondary comparison.

Early returns truncate the sequence. The mode and numeric checks occur between the primary and secondary groups.

Repeated calls must remain separate observations. A preceding call can change data or state used by a later call. Conventional comparison and length semantics alone do not imply that another dependency preserves their future inputs.

### Ownership and lifetime

Reference storage is local to the invocation. No caller-visible output region or explicit ownership-transfer protocol is provided.

The dependencies’ retention, mutation, allocation, release, and global-state behavior are unknown. Supplying an address does not establish transfer of ownership or permission to use stack storage after this invocation ends. Conversely, this caller does not prove that dependencies refrain from retaining those addresses.

## Dependencies

| Opaque function ID | Supplied inputs and result interpretation | Calls per normally returning invocation |
|---|---|---|
| F-7f14f1ab6851 | Name-like value, local reference address, and 32-bit count 14 or 8; result tested for zero/nonzero | Zero through four |
| F-1cb585febd84 | A2 or A5; 32-bit result tested for zero or compared unsigned against 14 or 8 | Zero through three |
| F-3ec7d07fef90 | A5 and address of the four-byte validation reference; zero permits continuation, nonzero rejects | Zero or one |

“Comparison-like,” “length-like,” and “validation-like” identify roles in this caller, not complete dependency contracts. The supplied counts do not by themselves prove read bounds.

No direct recursive call is visible.

## Acceptance examples

These examples are static, conditional criteria, not executed tests. Unless stated otherwise, earlier stages pass and all reached dependencies return normally.

| Condition | Required local outcome |
|---|---|
| A1 is zero | Return 4; no dependency calls. |
| A1 is nonzero and A2 is zero | Return 7; no dependency calls. |
| First primary comparison is zero; next length-like result is 15 | Return 7; omit the second primary comparison. |
| First primary comparison is nonzero; second is zero | Primary stage passes without a primary length-like call. |
| First primary comparison is zero; length-like result is 0; second comparison is zero | Primary stage passes. |
| First primary comparison is zero; length-like result is 14; second comparison is nonzero | Return 7. |
| Primary stage passes; A4 is 2 or the unsigned maximum | Return 5, regardless of A3. |
| A4 is 0 or 1; A3 is 1 or 10 | Numeric stage passes. |
| A4 is 0 or 1; A3 is 0, 11, or negative | Return 8. |
| A4 is 3 or 4; A3 is 1 or 14 | Numeric stage passes. |
| A4 is 3 or 4; A3 is 0, 15, or negative | Return 8. |
| Numeric stage passes; A5 is zero | Return 9 without a secondary dependency call. |
| Initial secondary length-like result is zero | Return 9; omit F-3ec7d07fef90. |
| Initial secondary length-like result is nonzero; validation-like result is nonzero | Return 9, regardless of A6. |
| Initial secondary validation passes; A6 is any nonzero value | Return 0 without final comparison or length-like calls. |
| A6 is zero; first final comparison is zero; next unsigned length-like result is 9 | Return 11 without a second final comparison. |
| A6 is zero; first final comparison is nonzero; second is zero | Return 0 without an intervening final length-like call. |
| A6 is zero; first final comparison is zero; length-like result is 8; second comparison is zero | Return 0. |
| A6 is zero; the reached second final comparison is nonzero | Return 11. |

### Conditional conventional-string examples

The following examples additionally assume:

- The local references contain the indicated text with suitable termination for every reached dependency.
- Comparison and length dependencies implement ordinary bounded string comparison and ordinary string length.
- Dependency side effects and intervening changes do not alter relevant name or reference contents during the invocation.
- F-3ec7d07fef90 returns zero and all calls return normally.

Under those explicit assumptions:

- Nonzero A1, primary name “Standard Theta,” A3 equal to 10, A4 equal to 0, secondary name “STANDARD,” and A6 equal to zero produce return 0.
- Changing only A3 to 11 produces return 8.
- Appending characters to the primary name produces return 7 when the initial bounded comparison matches and the measured length exceeds 14.
- Appending characters to “STANDARD” produces return 11 when A6 is zero, the initial final comparison matches, and the measured length exceeds 8.

The preservation assumption is necessary for the first example. As a contrasting conditional case, suppose the secondary buffer initially contains “STANDARD,” but F-3ec7d07fef90 replaces it with “ABCDEFGH” while returning zero. With all preceding stages passing, unchanged secondary reference contents, conventional subsequent comparisons, and A6 equal to zero, both final comparisons are nonzero and the function returns 11. The final length-like call is skipped. A successful validation-like result alone therefore does not guarantee return 0.

## Uncertainties

The following remain **documented_only**:

- Exact reference bytes, continuation bytes, terminators, and external encoding beyond the information suggested by labels.
- The four-byte validation-reference value and its meaning.
- Complete dependency signatures, memory-access bounds, address requirements, failure behavior, and interpretation of results.
- Whether dependencies modify name storage, reference storage, other memory, or globals.
- Whether dependencies retain addresses or allocate or release memory.
- Whether external activity changes reference backing storage during preparation or name contents between calls.
- Endianness and synchronization guarantees.
- Semantic roles of A1, the modes, the numeric selection, and the flag beyond their established local uses.
- Broader meanings of the returned status values.

No dependency is assumed to preserve globals, preserve its arguments, or own or release memory. Inferred string intent does not resolve these unknowns.

## Evidence

The established behavior comes from static examination of the supplied function body: declared argument representations, local reference construction, explicit zero tests, short-circuit evaluation, dependency arguments and order, unsigned range checks, and return constants.

Text-name interpretations derive from reference labels. Dependency role descriptions derive from how this caller uses their arguments and results, not from dependency implementations.

No runtime tests have been performed. No dependency implementation, complete reference-byte dump, or external API contract was supplied.
