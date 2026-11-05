<!-- Trusted provenance: F-aaeeb0735be1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aaeeb0735be1

## Purpose

F-aaeeb0735be1 dispatches text to one of two parsing dependencies, converts the resulting temporary representation, and conditionally sets a bit in the converted result.

The wrapper recognizes one leading minus sign. After removing that sign from the parser-facing text position, it selects a hexadecimal-oriented parser for an observed `0x` or `0X` prefix. Other text goes to a second parser whose grammar is not established.

For an explicitly negative input, two particular temporary representations clear the recorded negative indication. After conversion and a cleanup-like call, a retained negative indication causes the wrapper to set bit `0x100` in a 32-bit result field.

**Inferred intent:** numeric text conversion with negative-zero normalization. The exact established contract is the dispatch, representation predicates, call ordering, failure handling, and field operation described below. Numeric syntax, numeric range, ownership, and the semantic meaning of the result depend on unavailable implementations.

All behavior is derived from static evidence. No runtime tests have been performed.

## Interface

### Arguments

The function receives:

1. **An opaque 32-bit value.** The supplied body neither inspects nor forwards it. This says nothing about effects incurred by the caller while producing that argument.
2. **A character pointer.** A null value reaches explicit failure reporting. A nonnull value is inspected for a minus sign and a prefix, then a pointer into the same storage is passed to the selected parser.

The wrapper does not directly write input characters. Dependencies are not established to preserve that storage, however. In particular, F-7d962cda8fd4 executes before any input-character read and could modify the input through an existing alias.

Consequently, **entry-time text contents do not alone determine dispatch**. Dispatch depends on the characters actually observed after F-7d962cda8fd4 returns.

For defined execution, each location the wrapper reads must be readable at the time of access:

- The initial character of a nonnull input.
- The character after an observed leading minus sign.
- The character following an observed `0` at the prefix-detection position.

Dependencies may require additional readable storage, termination, or other conditions. Those requirements are not established here.

### Return value

The exposed return is a 32-bit integer-like value:

- Explicit failure paths return zero after calling F-4fc89ec41a6d.
- Successful paths return the same nonzero value obtained from F-a3ab0f0268b0.

The wrapper may modify storage identified by that nonzero value before returning it. It does not alter the returned value itself.

When the negative indication remains set, the conversion result is used as an address with a readable and writable 32-bit field at byte offset 4. That storage must remain accessible through the later field update, including after F-2642535311f0 returns.

The formal classification of the result as a pointer, handle, or pointer-valued integer encoding is **documented_only**.

### Temporary representation

A local pointer slot is initialized to null and then assigned the result of F-7d962cda8fd4. No visible check rejects a null result from that dependency.

Each parser receives the address of this slot and may replace its contents. Subsequent operations use the slot’s then-current pointer rather than necessarily using the original dependency result.

The wrapper conditionally observes this layout:

| Location | Established interpretation by this wrapper |
|---|---|
| Temporary offset 0 | A 32-bit pointer-like word |
| Temporary offset 4 | A 32-bit value compared with zero and one |
| Address identified by the offset-0 word | A 32-bit value compared with zero when the offset-4 value equals one |

These observations do not establish a complete structure definition. Interpreting the first field as digit storage and the second as a digit count is inferred intent, not a proven dependency contract.

## Behavior

### Initialization precedes input validation and inspection

The wrapper initializes its temporary pointer slot to null, calls F-7d962cda8fd4 without arguments, and stores the returned value in that slot.

This call occurs even when the input pointer is null. It also occurs before any input-character inspection.

If the input pointer is null, the wrapper proceeds to common failure reporting. It does not invoke a parser, F-a3ab0f0268b0, or F-2642535311f0 on that path.

No preservation guarantee applies to input storage across the initialization call. For example, an entry-time buffer containing `0x2a` need not select the hexadecimal-oriented parser if F-7d962cda8fd4 changes that buffer before dispatch reads it.

### Leading-sign recognition

For a nonnull input pointer, the wrapper observes whether the initial character equals `-` and records that result.

If the comparison succeeds, the parser-facing position advances by exactly one character. Otherwise, it remains at the original position.

The wrapper does not itself:

- Skip whitespace.
- Recognize or remove a plus sign.
- Remove more than one minus sign.
- Require a character or digit after the minus sign.
- Interpret the remaining text numerically.

The recorded negative indication is based on the sign-recognition observation. Later dependency changes to input text do not cause sign recognition to be repeated.

### Prefix dispatch

At the position following the optional minus sign, the wrapper checks for `0`. It examines the following character for `x` or `X` only if the `0` comparison succeeds.

The selected parser and text position are:

| Observed prefix condition | Dependency | Text position passed |
|---|---|---|
| `0` followed by `x` or `X` | F-647f70d31437 | Two characters after the prefix-detection position |
| Otherwise | F-c6e30971468c | The prefix-detection position itself |

Each parser also receives the address of the temporary pointer slot.

This is dispatch based on observed characters, not complete syntax validation. The wrapper does not check for a digit after a hexadecimal prefix, confirm full consumption, or reject trailing characters.

The source establishes conditional character comparisons and pointer positions. It does not establish concurrent-access semantics, atomic observation of the text, or an exact machine-level load count.

### Parser status

The selected parser’s return is compared with zero:

- Zero reaches common failure reporting.
- Any nonzero status continues processing.

No distinction is made between positive and negative nonzero statuses, and no additional status information is consumed.

On parser failure, the wrapper does not call F-a3ab0f0268b0 or F-2642535311f0. This remains true if the parser replaced the temporary pointer or modified referenced storage before returning zero.

### Conditional sign normalization

The wrapper performs representation checks only if sign recognition recorded a leading minus.

It reads the temporary’s 32-bit field at offset 4 and applies these exact predicates:

- **Field equals zero:** clear the negative indication without following the first field.
- **Field equals one:** read the 32-bit pointer-like word at offset 0, then read a 32-bit value at the address it identifies. Clear the negative indication only if that referenced value equals zero.
- **Field is neither zero nor one:** retain the negative indication without following the first field.

These are representation-specific conditions, not a general numerical zero test. In particular, an offset-4 field equal to two retains the negative indication even if externally interpreted storage contains only zero words.

No temporary-pointer validity check follows successful parsing. For explicitly negative input, the temporary and any conditionally accessed storage must therefore support these reads.

If sign recognition did not record a minus, these representation reads are skipped entirely.

### Conversion and cleanup-like call

After successful parsing and any sign normalization, the wrapper calls F-a3ab0f0268b0 with:

- The temporary pointer obtained from the slot for that call.
- A second argument equal to zero.

It retains the returned 32-bit value and then calls F-2642535311f0 with the temporary pointer obtained from the slot for that later call.

The cleanup-like call occurs whether conversion returned zero or nonzero. The conversion result is tested for zero only after F-2642535311f0 returns.

The wrapper does not directly assign the temporary slot between these calls. Nevertheless, dependency behavior is unknown: a parser may have retained an alias, and dependencies are not established to preserve indirectly accessible state. The contract therefore does not require the two calls to observe identical temporary state or, where indirect mutation is possible, identical slot contents.

### Successful result handling

If the saved conversion result is nonzero and the negative indication is clear, the wrapper returns that value without directly inspecting result fields.

If the saved conversion result is nonzero and the negative indication remains set, the wrapper:

- Reads the 32-bit field at result offset 4 after F-2642535311f0 has returned.
- Sets bit `0x100` in the value read.
- Writes the resulting 32-bit value to that field.
- Returns the original conversion result.

The write preserves every other bit of the value observed by that post-cleanup read. It is not an assignment of the entire field to `0x100`.

**The field value when conversion returns is not sufficient to predict the written value.** F-2642535311f0 may change that field directly or indirectly. The bit operation applies to the value actually read afterward.

The recorded negative indication is not recomputed after conversion or cleanup-like processing.

### Common failure reporting

The wrapper calls F-4fc89ec41a6d with the ordered numeric arguments:

1. `0x22`, decimal 34.
2. `0x6c`, decimal 108.

It then returns zero. Any return value from that dependency is ignored.

The visible paths to this operation are:

- Null input pointer.
- Zero parser status.
- Zero conversion result, after F-2642535311f0 has returned.

No specific error category, diagnostic message, global error state, or exception behavior is established by these constants.

## Side effects

### Direct memory effects and ordering

The wrapper directly inspects input characters, conditionally reads the temporary representation, and conditionally updates one result field. Its temporary-slot assignments are local state changes.

The established order is:

1. Temporary-slot initialization.
2. F-7d962cda8fd4.
3. Null-input decision.
4. For nonnull input, sign recognition and prefix dispatch.
5. Exactly one parser call on the normal nonnull path.
6. Parser-status comparison.
7. Conditional temporary reads for sign normalization.
8. F-a3ab0f0268b0.
9. F-2642535311f0.
10. Conversion-result comparison.
11. Either the optional result-field read/write followed by successful return, or failure reporting followed by zero return.

On null-input and parser-failure paths, no explicit cleanup-like call occurs.

The ordering matters for both input and output: initialization can affect characters later observed, and cleanup-like processing can affect the field later read and modified.

### Ownership and lifetime

The wrapper contains no explicit allocation or deallocation primitive. Dependency placement does not prove ownership semantics.

Specifically:

- F-7d962cda8fd4 is not proven to allocate fresh storage.
- Parsers may replace the temporary pointer.
- Conversion may copy, share, consume, or otherwise transform temporary state.
- F-2642535311f0 is cleanup-like by placement, but is not proven to release memory.
- Omission of that call on early failure does not independently establish a leak.
- No release obligation for a successful result can be derived for the caller.

If the result aliases storage affected by F-2642535311f0, the later negative-result update still requires accessible storage. The supplied body does not show how that requirement is satisfied.

### Globals and exceptional behavior

The wrapper shows no direct global-state access. Dependencies may read or modify globals and aliased memory; no preservation guarantee is established.

Explicit zero-return paths do not imply safe handling of every malformed pointer, malformed representation, or dependency failure. There is no visible recovery from inaccessible storage or violated representation requirements.

Descriptions of subsequent operations assume each invoked dependency returns normally.

## Dependencies

| Opaque function ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-7d962cda8fd4 | Called without arguments before input validation or character reads; result replaces the initial null temporary | Allocation, initialization, failure convention, ownership, effects on aliased input or globals |
| F-c6e30971468c | Receives the temporary-slot address and text after the optional minus; zero means failure to this wrapper | Grammar, base, termination, consumption, mutation, retention, ownership |
| F-647f70d31437 | Receives the temporary-slot address and text after an observed `0x` or `0X`; zero means failure | Accepted syntax, empty-remainder handling, mutation, retention, ownership |
| F-a3ab0f0268b0 | Receives the then-current temporary pointer and zero; returns the saved result | Meaning of zero argument, representation, aliasing, ownership, state changes |
| F-2642535311f0 | Called after every conversion call and before result testing or field access | Release behavior, mutations to temporary or result storage, global effects |
| F-4fc89ec41a6d | Receives 34 and 108 on each explicit failure path | Diagnostics, error-state changes, return behavior, exceptional control flow |

## Acceptance examples

These are conditional static obligations, not observed executions or claims about parser grammar.

For text examples below, the named characters must be those observed during sign recognition and dispatch **after F-7d962cda8fd4 returns**. A stable buffer containing the stated text during those reads satisfies that condition. Merely having that text at function entry does not.

| Condition | Required wrapper behavior |
|---|---|
| Input pointer is null and dependencies return normally | Call F-7d962cda8fd4, report failure with 34 and 108, and return zero; do not call a parser, converter, or cleanup-like dependency |
| Dispatch observes `0x2a` | Select F-647f70d31437 with a pointer at `2a` |
| Sign recognition and dispatch observe `-0X2a` | Record the minus, select F-647f70d31437, and pass a pointer at `2a` |
| Dispatch observes `+0x2a` | Select F-c6e30971468c with a pointer at the plus sign |
| Sign recognition and dispatch observe `--1` | Remove only the first minus; select F-c6e30971468c at the second minus |
| Dispatch observes `0x` followed by the string terminator | Call F-647f70d31437 at the empty remainder; acceptance is dependency-defined |
| Entry-time text is `0x2a`, but initialization changes it to stable `17` before inspection | Select F-c6e30971468c using the subsequently observed text |
| Parser replaces the temporary pointer and returns zero | Report failure without conversion or F-2642535311f0 |
| Negative indication is set and temporary offset 4 is zero when checked | Clear that indication without following the first field |
| Negative indication is set, offset 4 is one, and the referenced 32-bit value is zero when read | Clear that indication |
| Negative indication is set and offset 4 is two when checked | Retain that indication without examining referenced words |
| Conversion returns zero | Call F-2642535311f0 before reporting failure and returning zero |
| Conversion returns nonzero, negativity remains set, and result offset 4 equals `0x00000005` immediately before the post-cleanup read | Write `0x00000105` and return the same conversion result |
| Conversion initially leaves result offset 4 as `0x00000005`, but cleanup changes it to `0x00000020` before the wrapper reads it | Write `0x00000120`, not `0x00000105` |
| Conversion returns nonzero and negativity is clear | Return the saved result after cleanup-like processing, without a direct result-field access by this wrapper |

Empty text, a lone minus, whitespace, trailing characters, and out-of-range text are not independently rejected by this wrapper. Their parser outcomes remain unresolved.

## Uncertainties

The following remain **documented_only**:

- Complete input grammar, termination requirements, range limits, and overflow handling.
- Whether F-c6e30971468c specifically parses decimal text.
- Meanings of nonzero parser statuses.
- Complete temporary and result layouts.
- Semantic meaning of result bit `0x100`.
- Allocation, ownership, retention, release, and aliasing rules.
- Dependency effects on input storage, result storage, globals, or retained slot aliases.
- Thread safety, reentrancy, and concurrent mutation behavior.
- Formal pointer/integer ABI, character encoding, and byte order.
- Diagnostic semantics and non-returning dependency behavior.

The visible word operations and address-like values are 32-bit. Character-position advances are by one for the sign and two for the hexadecimal prefix. Field offsets are byte offsets; the result update is a 32-bit bitwise operation. No numeric magnitude arithmetic, widening conversion, or overflow check is established in the wrapper.

**Inferred intent:** the temporary represents a magnitude, the result field includes a sign flag, and the conditional sign clearing avoids negative zero. These interpretations do not override the exact observed predicates or dependency uncertainties.

## Evidence

The supplied function body establishes the argument use, initialization-before-inspection ordering, dispatch conditions, parser-status handling, temporary representation checks, conversion and cleanup-like call order, post-cleanup field operation, and explicit failure paths.

The timing qualifications follow from that same ordering: input is first inspected after F-7d962cda8fd4, and the result field is read only after F-2642535311f0. Neither dependency is established to preserve aliased storage.

Dependency implementations, external type definitions, and runtime observations were not supplied. No runtime tests have been performed.
