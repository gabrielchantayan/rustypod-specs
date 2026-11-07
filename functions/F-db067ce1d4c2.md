<!-- Trusted provenance: F-db067ce1d4c2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-db067ce1d4c2

## Purpose

F-db067ce1d4c2 selects among three formatting-oriented paths using two successive arithmetic-helper results, a zero-versus-nonzero presentation selector, and an exact comparison of mode with 1.

**Established static behavior:** the function calls F-d7be9cfb4000 twice, with constants 3,600 and 60, and subtracts corresponding products from the numeric input. It selects one of two static addresses. Two paths make one call to F-1f62160dfd52; the third prepares local-address-based state, makes two calls to that dependency, and invokes F-289ec4311c64 on two local addresses.

The function returns a 64-bit concatenation of two local 32-bit words. It does not visibly return a formatting dependency’s result.

**Inferred intent:** the arithmetic and template selection resemble duration formatting, with a specialized representation when the two helper results are zero. This interpretation does not establish quotient semantics, time units, valid duration ranges, or rendered output.

## Interface

### Inputs

The supplied representation exposes four arguments:

| Argument | Represented width | Established use |
|---|---:|---|
| Numeric input | Signed 32 bits | Passed to the first arithmetic helper and reduced using both helper results. |
| Presentation selector | Signed 32 bits | Compared with zero to choose a static address; copied initially into the upper return word. |
| Output operand | Uninterpreted 32 bits | Passed by value to the caller-directed formatting call on every path; also copied into local storage. |
| Mode | Signed 32 bits | Compared with exactly 1 when both helper results are zero; also copied into local storage. |

The output operand’s representation does not establish whether it is a pointer, handle, or another value. Its validity requirements, mutability, and ownership depend on unavailable dependency contracts.

No explicit argument validation is visible. The function does not reject negative numeric inputs, negative selectors, or a zero output operand. This absence of rejection is not a guarantee that dependencies accept them.

### Local storage

The supplied representation identifies four successive 32-bit local words, initially containing:

1. The numeric input.
2. The presentation selector.
3. The output operand.
4. The mode.

The first two words provide the final return value. On the specialized path, the first and third words have their addresses passed to dependencies.

These word-sized initializations do not establish the sizes of objects subsequently operated on through those addresses. A dependency receiving the first word’s address must not be assumed to write only that word or preserve the neighboring return word.

The original output operand remains the value used for caller-directed formatting. That operation does not instead use the potentially modified contents of the third local word.

### Return value

The result is a 64-bit bit pattern:

- Bits 0–31 come from the first local word.
- Bits 32–63 come from the second local word.

This is concatenation, not sign extension of the lower word.

On ordinary paths, explicit assignments establish the candidate return contents before the formatting call. On the specialized path, address-taking operations can affect the words before return. Dependency effects and abnormal termination remain subject to the limitations below.

## Behavior

### Arithmetic and selection

The function first copies the four arguments into local words. It then calls F-d7be9cfb4000 with the original numeric input and 3,600.

After that call returns, the function reduces the numeric value by subtracting 3,600 times the first helper result. It passes this reduced value, not the original input, to F-d7be9cfb4000 with 60.

After the second helper call returns, the presentation selector chooses a static address:

- Zero selects alternative A.
- Any nonzero value selects distinct alternative B.

Negative selectors therefore select the same alternative as positive nonzero selectors. The contents, length, encoding, and termination of both alternatives are unavailable. Selecting an address does not establish a read of the bytes stored there.

The function then subtracts 60 times the second helper result from the reduced numeric value. This produces the final reduced value used below.

Both helper results and the arithmetic expressions are represented as 32-bit integers. The visible arithmetic corresponds to successive subtraction of the two products; it does not establish division, quotient rounding, or mathematical remainder properties. Overflow behavior is not established for products or additions outside the signed 32-bit range. Consequently, unbounded-integer calculations are valid acceptance criteria only where the represented operations do not overflow.

The ordering is significant: the second helper call precedes static-alternative selection, and selection precedes the final reduction.

### Extended ordinary path

A nonzero first helper result selects the extended ordinary path. Neither the second helper result nor mode can redirect this case to the specialized path.

Before formatting, the function:

- Selects the extended static template.
- Writes the second helper result into the lower return word.
- Writes the final reduced value into the upper return word.

It then invokes F-1f62160dfd52 with four explicit operands, in order:

1. The original output operand.
2. The extended template address.
3. The selected alternative address.
4. The first helper result.

After that call returns normally, the function returns the concatenated local words. Absent intervening effects on those words, the lower half is the second helper result and the upper half is the final reduced value.

The mode has no further visible role on this path.

### Short ordinary path

When the first helper result is zero, the specialized path requires both a zero second helper result and mode equal to exactly 1. If either condition fails, the function takes the short ordinary path.

Before formatting, it selects the short static template and writes the final reduced value into the lower return word. It does not explicitly replace the upper return word, which initially contains the presentation selector.

It invokes F-1f62160dfd52 with four explicit operands, in order:

1. The original output operand.
2. The short template address.
3. The selected alternative address.
4. The second helper result.

After normal completion, it returns the two local words. Absent intervening effects on them, the upper half contains the original selector’s bit pattern and the lower half contains the final reduced value.

The explicit fourth formatting operand is the second helper result, not the final reduced value. The nearby local-word assignment does not, by itself, establish an additional formatting argument.

### Specialized path

The specialized path requires all three conditions:

- The first helper result is zero.
- The second helper result is zero.
- Mode equals exactly 1.

The function begins this path by calling F-9d0ecc5f6adf without explicit arguments. Only after that call returns does it read a static datum and pass that datum as the second operand to F-c0b020cd3882. The first operand is the result obtained from F-9d0ecc5f6adf.

**The static datum’s width is unknown.** Neither its declaration nor the parameter types of F-c0b020cd3882 are supplied. The 32-bit local representation used to capture a dependency result does not determine the width of this separate operand.

The result captured from F-c0b020cd3882 is passed as the second operand to F-7afe554bcf62, whose first operand is the address of the third local word. F-5b24edfb22c0 is then called with the address of the first local word.

The subsequent operations use those two addresses in a fixed order:

- F-44362605ab79 receives the third word’s address.
- Its captured result becomes the fourth explicit operand of F-1f62160dfd52. The preceding operands are the first word’s address, a specialized static formatting-object address, and the selected alternative address.
- F-44362605ab79 then receives the first word’s address.
- Its captured result becomes the second explicit operand of another F-1f62160dfd52 call. That call receives the original output operand first and the final reduced numeric value third.

Thus, local-address-directed formatting precedes caller-directed formatting. The caller-directed call has three explicit operands in the supplied representation, whereas the earlier local-directed call has four.

After caller-directed formatting returns, F-289ec4311c64 is called first with the first local word’s address and then with the third local word’s address. Both calls precede the reads that assemble the 64-bit return value.

There is no visible restoration of the initial argument copies before return. A specialized-path return cannot therefore be predicted from those copies alone. In particular, neither F-289ec4311c64 call is established as preserving the return words.

## Side effects

### Directly established effects

The function initializes four local words and conditionally rewrites the first two. It does not visibly assign to a global variable.

It selects static addresses on every path. The specialized path additionally reads a static datum after F-9d0ecc5f6adf returns. Because that dependency’s global effects are unknown, the datum must not be treated as necessarily having its pre-call value.

Every path invokes F-1f62160dfd52 with the original output operand first in a caller-directed call. The specialized path also invokes it with the first local word’s address first.

### Dependency-mediated effects and ownership

The actual output bytes, output-object state, allocations, releases, and global mutations are not established. They depend on contracts unavailable here.

The specialized sequence resembles preparation, extraction, formatting, and cleanup of temporary objects. That is inferred intent, not an ownership specification:

- F-7afe554bcf62 is not established as acquiring ownership of its second operand.
- F-44362605ab79 is not established as returning either borrowed storage or an owned allocation.
- F-289ec4311c64 is not established as releasing memory, preserving object contents, or tolerating repeated invocation on the same state.
- The extent of any write through an exposed local address is unknown.

The ordinary paths do not explicitly expose these local addresses. Their stated return contents describe the visible local assignments, subject to dependency effects permitted by the actual execution environment.

### Errors and interrupted execution

No dependency result is explicitly checked for failure. No rollback, recovery branch, or alternate cleanup sequence is visible.

If a dependency does not return normally, later calls and the final return are not guaranteed by this representation. If it returns an error indication normally, the visible function does not branch on that indication; captured results continue to be used as described.

No success condition for rendered output can be derived solely from normal return.

## Dependencies

| Opaque ID | Established use | Unavailable contract details |
|---|---|---|
| F-d7be9cfb4000 | Called twice with a 32-bit numeric value and constants 3,600 and 60; results are represented as signed 32-bit values. | Arithmetic meaning, rounding, exceptional cases, and side effects. |
| F-9d0ecc5f6adf | Called without explicit arguments on the specialized path; its result is captured in 32-bit storage. | Semantic type, ownership, lifetime, and global effects. |
| F-c0b020cd3882 | Receives the preceding captured result and a subsequently read static datum; its result is captured in 32-bit storage. | Parameter types, static-datum width, result meaning, and ownership. |
| F-7afe554bcf62 | Receives the third local word’s address and the preceding captured result. | Initialization requirements, written extent, and ownership transfer. |
| F-5b24edfb22c0 | Receives the first local word’s address. | Object layout, required initial state, and affected extent. |
| F-44362605ab79 | Receives the third local address, then later the first; each result is captured in 32-bit storage. | Result representation, validity period, ownership, and input mutation. |
| F-1f62160dfd52 | Receives the explicit operands described for each formatting-oriented call. | Complete signature, variadic interpretation, output semantics, and errors. |
| F-289ec4311c64 | Receives the first local address and then the third before specialized-path return. | Cleanup semantics, released resources, and post-call storage contents. |

Captured result widths do not establish complete dependency signatures or the widths of unrelated static operands.

## Acceptance examples

These are conditional static criteria, not runtime test results. Return-value examples assume normal completion and no dependency effects that alter the relevant return words.

### Nonzero first result

For numeric input 3,661, suppose F-d7be9cfb4000 returns 1 for both calls.

The first reduction produces 61; the final reduction produces 1. No overflow occurs.

Expected behavior:

- The extended ordinary path is selected for every mode.
- The formatting call receives the original output operand, extended template, selected alternative, and 1.
- Both return words are assigned 1.
- The conditional return bit pattern is `0x0000000100000001`.

No particular output string is established.

### Zero first result and nonzero second result

For numeric input 61, suppose the helper results are 0 and 1, and the selector is 7.

The short ordinary path is selected even when mode is 1. The selected alternative is B. The explicit fourth formatting operand is 1, and the lower return word is assigned the final reduced value, also 1.

The conditional return bit pattern is `0x0000000700000001`.

### Exact mode comparison

For numeric input 5, suppose both helper results are zero.

- Mode 1 selects the specialized path.
- Modes 0, 2, and −1 select the short ordinary path.

On the short path, the explicit fourth formatting operand is zero, while the lower return word is assigned 5. Whether that local word is additionally consumed by the formatting mechanism is unresolved.

With selector −1, the conditional short-path return is `0xffffffff00000005`; the selector is preserved as an upper-half bit pattern, not derived by sign-extending the lower word.

### Static-datum timing and width

For any input satisfying the specialized-path conditions, the static datum is read after F-9d0ecc5f6adf returns.

If that dependency changes the datum, the subsequent operand is obtained by the post-call read. This criterion does not specify the datum’s width, signedness, or value.

### Specialized-path completion order

On normal completion of the specialized path:

- Local-directed formatting precedes caller-directed formatting.
- Caller-directed formatting receives the original output operand, not the third local word’s current contents.
- The first-address F-289ec4311c64 call precedes the third-address call.
- Both calls precede return-word reads.

A return prediction based solely on the initial argument copies is insufficient.

## Uncertainties

The following remain **documented_only**:

- **Arithmetic semantics:** helper meaning, signed rounding, negative-input handling, and exceptional cases.
- **Overflow:** behavior outside the signed 32-bit range.
- **Static data:** alternative and template contents; the specialized datum’s declaration, width, signedness, and value.
- **Calling convention:** complete dependency signatures and whether staged local words are consumed as additional formatting arguments.
- **Object layout:** sizes, invariants, and valid initial states for address-exposed local storage.
- **Output validity:** interpretation and required state of the output operand.
- **Ownership:** allocation, borrowing, transfer, release, and lifetime responsibilities.
- **Return stability:** dependency effects on local words, particularly after the specialized-path final operations.
- **Failures:** error signaling, partial output, exceptions, and nonlocal exits.
- **Concurrency:** synchronization, reentrancy, and effects of shared mutable state.

These unknowns do not justify inventing validation, conventional duration ranges, specific strings, or resource-management guarantees.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping. Prior prose and review feedback are not independent behavioral evidence.

Established evidence comprises the represented argument and local-word widths, constants 3,600 and 60, comparison conditions, local assignments, explicit dependency operands, static-address selection, static-datum read order, and final two-word concatenation.

The static datum lacks a supplied declaration, and F-c0b020cd3882 lacks supplied parameter types. Its width therefore remains unknown despite the 32-bit storage used for captured dependency results.

Duration formatting, temporary-object lifecycle, and possible additional formatting arguments remain interpretations rather than established behavior.

No dependency implementations or static-data contents were supplied. No runtime tests have been performed.
