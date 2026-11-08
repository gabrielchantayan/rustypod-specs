<!-- Trusted provenance: F-3e73fa53d4da; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3e73fa53d4da

## Purpose

F-3e73fa53d4da interprets pairs of input bytes as hexadecimal digits and writes each accepted pair as one binary byte into a buffer obtained from a dependency. A colon is skipped only where a new pair would begin. It is not skipped between the two characters of a pair.

The established local behavior includes preparation, buffer acquisition, input traversal, conditional character normalization, hexadecimal validation, sequential output writes, and failure-specific diagnostic and cleanup calls. The apparent intent is hexadecimal decoding with optional colon separators. The external contracts of the dependencies are not established.

Successful output is binary data. The function does not append a zero terminator.

This contract is based on static evidence only. No runtime tests have been performed.

## Interface

The function accepts four arguments and returns either a byte pointer or null.

| Argument | Representation | Established use |
|---|---|---|
| Input | Pointer to 8-bit bytes | Supplies the byte sequence to scan. Null selects an immediate failure path. |
| Length destination | Pointer to a 32-bit integer, possibly null | Passed to preparation. On local success, receives the decoded byte count if non-null. |
| Third argument | Opaque 32-bit value | Forwarded unchanged to preparation. |
| Fourth argument | Opaque 32-bit value | Forwarded twice to preparation, in its fourth and fifth argument positions. |

The represented environment has 32-bit integers and 32-bit pointer representations. Input and output elements are 8-bit bytes.

On success, the returned pointer is the original non-null pointer obtained from F-43da6b23bd44, not the position after the last output byte. If the length destination is non-null, a 32-bit length store immediately precedes the return.

On any locally handled failure, the returned value is null. There is no uniform failure value for the length destination: preparation receives that pointer before parsing begins and may modify its contents. Parsing failures do not perform the final success-length store.

### Preconditions and storage constraints

Ordinary successful operation requires:

- Readable input at every position actually examined.
- A writable 32-bit destination if the success-length store is performed.
- A valid classification handle and readable classification entries at each lookup.
- A writable acquired region with enough space for every output byte.
- Address calculations and the final pointer difference that are valid in the represented environment.
- Dependency behavior compatible with continued execution, including any required storage lifetime.

These requirements are not comprehensively checked locally. There is no visible capacity check, pointer validation, classification-handle validation, or recovery from invalid memory access.

No non-aliasing guarantee is established. Output writes can affect overlapping input or classification storage; the final length store can affect overlapping input or output. Dependencies may also modify accessible storage. Consequently, exact decoded-content expectations require stronger environmental assumptions than merely obtaining a non-null buffer.

## Behavior

### Preparation and acquisition

For null input, the function invokes F-4fc89ec41a6d with the visible argument sequence `0x22, 0x71` and returns null. It makes no local input read and does not invoke preparation, acquisition, classification, normalization, or release.

For non-null input, F-1cb585febd84 is called first with:

1. The input pointer.
2. The length-destination pointer.
3. The third argument.
4. The fourth argument.
5. The fourth argument again.

Its result is represented as a signed 32-bit integer. That result, shifted right by one bit, supplies the first argument to F-43da6b23bd44. The next two arguments are zero.

For a nonnegative preparation result, the shifted value equals division by two rounded down. A negative result is not rejected locally before the shift and acquisition call. The external meaning of such a value, the receiving dependency’s parameter signedness, and any associated failure convention remain unknown.

If acquisition returns null, the function invokes the generic diagnostic with the visible sequence `0x22, 0x71` and returns null. No release call is made for that null result.

If acquisition returns non-null, parsing begins with the output position at the acquired base address. The function does not compare the eventual decoded length against the requested acquisition quantity or any actual capacity.

### Boundary scanning and pair formation

At each pair boundary, the function reads an input byte:

- Zero completes conversion successfully.
- Colon, byte value `0x3A`, advances the input position by one and repeats the boundary read.
- Any other value starts a prospective pair.

Thus leading, trailing, and consecutive colons are locally permitted. They generate no output bytes.

After selecting a non-colon, nonzero first character, the function reads the immediately following byte as the second character. No separator-skipping occurs within a pair.

If that second byte is zero, the pair is incomplete. The function calls F-4fc89ec41a6d with `0x22, 0x71, 0x70, 0, 0`, then calls F-5990a26eaaa3 with the acquired base pointer, then returns null.

This incomplete-pair check precedes classification and hexadecimal validation. An otherwise invalid first character followed immediately by zero therefore takes the incomplete-pair path. No classification or normalization call occurs for that incomplete pair.

### Classification and normalization

For each prospective pair whose second byte is nonzero, the function calls F-2a4c402ba617 once. Its result is used as a pointer to a 32-bit holder containing the classification-table address.

The first classification lookup reads the holder, then reads the byte-sized table entry indexed by the first input byte. If bit `0x10` is set, F-1aef7372e8d8 is called with that byte value. Only the low eight bits of its result are retained.

The second lookup similarly uses the second input byte and conditionally invokes F-1aef7372e8d8. The table-address holder is dereferenced again for this lookup.

The observable order is significant:

1. Both pair bytes are read from input.
2. The second byte is checked for zero.
3. The classification handle is obtained.
4. The first classification lookup and any first normalization occur.
5. The second classification lookup and any second normalization occur.
6. Hexadecimal validity is checked.

The two input-byte values used for this pair have already been captured before these dependency calls. Dependency changes to input can nevertheless affect later input reads.

No preservation of classification state is assumed. In particular, first-character normalization can change the holder or table before the second lookup. The second lookup may therefore observe a different table address or different table contents.

Both optional normalizations occur before either character is rejected as non-hexadecimal. Even when the first normalized character will fail validation, second-character classification and any normalization have already occurred.

Case conversion is an inferred purpose of F-1aef7372e8d8, not an established mapping. Uppercase acceptance depends on the classification entries and the returned low bytes.

### Validation and output

After optional normalization, each character must lie in one of these byte ranges:

| Range | Hexadecimal value |
|---|---|
| `0x30`–`0x39` | 0–9 |
| `0x61`–`0x66` | 10–15 |

The range checks use unsigned 32-bit subtraction and comparisons. Subtraction below a range’s lower endpoint wraps in that unsigned arithmetic and fails the associated range test. Such values are not accepted as negative digit values.

The first normalized character is validated before the second. The first accepted value supplies the output byte’s high four bits; the second supplies its low four bits.

Only after both characters pass validation does the function write one byte at the current output position. It then advances the input position past both pair characters and advances the output position by one byte.

There is no direct acceptance rule for whitespace, a `0x` prefix, or a colon inside a pair. A byte outside the accepted ranges can become acceptable only through a dependency-provided normalization result in those ranges.

No output byte is directly written for a rejected or incomplete pair. Earlier accepted pairs may already have caused writes. The function performs no explicit rollback or erasure of those writes. This does **not** establish that earlier output remains unchanged: subsequent classification, normalization, diagnostic, or release calls may modify that region, and their preservation behavior is unknown. In particular, the incomplete-pair diagnostic executes before release.

### Successful completion

A zero read at a pair boundary ends parsing. That zero is not copied.

If the length destination is non-null, the function stores the difference between the current output position and the acquired base, using the represented 32-bit pointer-to-integer calculation. Under ordinary representable conditions, this is the number of output bytes written by the function.

No large-length or arithmetic-overflow check is visible. Behavior outside valid, representable address and difference calculations is not established.

The acquired base pointer is returned without a local release call. A success-length store that overlaps produced bytes can change final buffer contents immediately before return.

Empty input and colon-only input reach this success path if acquisition returns non-null. They cause no direct output-byte writes and produce a local decoded count of zero. Whether acquisition can succeed for the corresponding request is dependency-specific.

### Failure ordering

| Failure condition | Established local order |
|---|---|
| Null input | Generic diagnostic; return null |
| Null acquisition result | Preparation; acquisition; generic diagnostic; return null |
| Zero second character | Five-argument diagnostic; release handoff; return null |
| Invalid normalized character | Release handoff; generic diagnostic; return null |

The generic diagnostic has the visible argument sequence `0x22, 0x71`. The incomplete-pair diagnostic has the visible sequence `0x22, 0x71, 0x70, 0, 0`.

Diagnostic-before-release and release-before-diagnostic are distinct observable paths. Neither order implies that either dependency preserves output or other state.

There is no visible handling for a dependency that does not return normally.

## Side effects

Direct memory effects consist of:

- Input-byte reads in traversal order.
- Classification-holder and classification-table reads.
- One sequential byte write per accepted pair.
- One 32-bit length store on success when the destination is non-null.

Dependencies can add effects not characterized by this body. Preparation receives both the input and length-destination pointers. Classification and normalization can affect classification state or other storage. Diagnostic and release calls can also modify state, including storage containing previously produced bytes.

The acquired base pointer is passed to F-5990a26eaaa3 on parsing failure. This establishes a cleanup handoff, not a proven deallocation mechanism or preservation guarantee.

On success, the pointer is returned to the caller. The eventual release obligation and appropriate release operation are unknown. There is no visible local release of input storage.

Diagnostic invocation details do not establish an error message, logging destination, global error value, exception, or other specific diagnostic effect.

## Dependencies

| Opaque function ID | Established use | Unresolved contract |
|---|---|---|
| F-1cb585febd84 | Called once for non-null input before acquisition; receives the four arguments with the fourth repeated; returns the value used for sizing. | Counting, validation, failure signaling, pointer writes, and state effects. |
| F-43da6b23bd44 | Receives the shifted preparation result and two zeros; supplies the candidate output pointer. | Capacity, initialization, allocation mechanism, zero-size behavior, signedness, and ownership. |
| F-2a4c402ba617 | Called once per pair with a nonzero second byte; supplies the classification handle. | Handle validity, table contents, lifetime, mutation, and failure behavior. |
| F-1aef7372e8d8 | Conditionally receives a byte selected by classification bit `0x10`; only its returned low byte is retained. | Mapping and effects on storage or external state. |
| F-5990a26eaaa3 | Receives the acquired base on parsing failure. | Release mechanism, ownership effects, storage mutation, and other side effects. |
| F-4fc89ec41a6d | Called on each visible failure path with the argument sequences above. | Exact ABI, diagnostic meaning, storage mutation, external state effects, and normal-return behavior. |

## Acceptance examples

These are conditional static expectations, not observed executions.

Unless an example explicitly selects null input or acquisition failure, assume:

- Preparation and acquisition return normally and acquisition supplies a non-null region with adequate writable capacity.
- Every dependency invoked returns normally.
- Input is readable through all examined positions and remains unchanged by dependencies.
- Output writes do not overlap unread input or classification storage.
- Dependencies do not alter produced output bytes during the scenario. For failure examples, this assumption covers diagnostic and release calls as applicable; it does not imply that released storage remains readable afterward.
- The classification handle and relevant table entries remain valid throughout every lookup.
- Normalization leaves the stated digits and lowercase hexadecimal letters unchanged. Other required mappings are specified individually.
- Any non-null length destination is writable and does not overlap produced bytes when final buffer contents are asserted.
- Address calculations and the final decoded length are representable.

These assumptions are stronger than the function’s established guarantees.

1. **Contiguous pairs.** Input `0123af` followed by zero returns the acquired base containing `0x01, 0x23, 0xAF`. A non-null length destination receives 3. No output terminator is appended.

2. **Boundary separators.** Input `::01::af:` followed by zero produces `0x01, 0xAF` and length 2. The leading, repeated, and trailing colons are skipped at boundaries.

3. **Colon within a pair.** Input `a:f` followed by zero fails on the first prospective pair if normalization leaves the colon unchanged. No output byte is directly written. Release precedes the generic diagnostic. The trailing `f` is not reached as a new pair.

4. **Incomplete pair.** Input `a` followed by zero invokes the five-argument diagnostic before release and returns null. No classification call or output-byte write occurs for that pair.

5. **Invalid unpaired character.** Input `g` followed by zero also takes the incomplete-pair path. Its first character is not classified or validated before that failure.

6. **Failure after a successful write.** Input `01g0` followed by zero directly writes `0x01`, then fails on the second pair if normalization leaves `g` unchanged. The second pair causes no output-byte write. Release precedes the generic diagnostic, and there is no final length store. The first byte’s preservation during later calls follows only from the shared assumptions, not from a rollback or preservation guarantee.

7. **Conditional uppercase acceptance.** Input `AF` followed by zero produces `0xAF` if both relevant classification entries trigger normalization and the returned low bytes are `a` and `f`. If both bytes instead remain uppercase, validation fails.

8. **Empty result.** Empty input, or `:::` followed by zero, returns the non-null acquired pointer and stores zero when a length destination is supplied. No output byte is directly initialized to represent emptiness.

9. **Null length destination.** Input `10` followed by zero can produce `0x10` without a direct length store. Preparation still receives the null destination; its compatibility with that argument is an additional dependency requirement.

10. **Null input.** Assuming the diagnostic returns normally, the function returns null after the generic diagnostic. Preparation, acquisition, classification, normalization, and release are not locally invoked.

11. **Overlapping length destination.** If the success-length destination overlaps produced bytes, the final 32-bit length store can change them. The pointer can still be returned successfully; examples asserting final decoded contents do not apply without the non-overlap assumption.

## Uncertainties

The following remain **documented_only**:

- Preparation’s sizing rules, validation behavior, mutations, and failure signaling.
- Whether acquisition capacity covers every locally accepted input.
- The external interpretation of negative or high-bit-set acquisition arguments.
- Buffer ownership, release obligations, lifetime, and release semantics.
- Classification contents and normalization mappings.
- Preservation or mutation of input, output, classification state, and globals by dependencies.
- The exact diagnostic ABI and meaning of the differing visible argument sequences.
- Concurrency, reentrancy, and externally changing storage.
- Behavior after invalid memory access, unrepresentable arithmetic, or interrupted dependency execution.

Hexadecimal decoding is strongly supported by the explicit digit ranges and four-bit packing. Interpreting dependencies as a sizing routine, allocator, case converter, diagnostic reporter, or deallocator remains inferred intent rather than an established external contract.

## Evidence

The supplied static body establishes the local branches, visible call arguments, call ordering, byte-access order, classification mask, normalization truncation, unsigned range checks, output packing, and success-length store.

The supplied dependency map establishes the opaque function identities. It does not supply dependency implementations or preservation, allocation, ownership, or diagnostic contracts.

The body shows no explicit rollback or erasure. That absence does not establish preservation across intervening dependency calls.

No runtime tests, diagnostic captures, allocation experiments, or mutation checks have been performed. The acceptance examples state conditional consequences of the static behavior, not empirical results.
