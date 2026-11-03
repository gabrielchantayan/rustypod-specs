<!-- Trusted provenance: F-dc5033eeec80; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dc5033eeec80

## Purpose

F-dc5033eeec80 converts selected integer character codes into integer values. A selector equal to 120 enables special handling for the twelve integer codes corresponding to ASCII `A`–`F` and `a`–`f`. Those codes produce values 10 through 15. Every other case uses the first input minus 48, subject to the arithmetic qualifications below.

**Established static behavior:** this is a permissive numeric transformation, not a digit validator. It does not reject inputs outside a decimal or hexadecimal character range. The special handling depends on the selector being exactly 120.

**Inferred intent:** the constants suggest a digit-value helper with an optional hexadecimal mode selected by ASCII `x`. This interpretation explains the values but does not establish any caller-side parsing, encoding, or validation requirements.

## Interface

The interface has two integer inputs and one integer result:

| Item | Meaning |
|---|---|
| First input | An integer code to transform. |
| Second input | A selector. The value 120 enables the letter mappings described below; every other value selects the ordinary subtraction behavior. |
| Result | Either an explicitly mapped value from 10 through 15, or the first input minus 48. |

The supplied representation declares both inputs and the result as signed `int`. It does not independently establish the target integer width, calling convention, argument extension rules, or register layout. In particular, a 32-bit width must not be treated as proven solely from that declaration.

The interface contains no pointer, buffer, length, string, structure, or caller-supplied output parameter. No byte layout or byte order is involved in the explicit behavior. Although the constants resemble character codes, the inputs are integers: the function does not load a character from an address or decode text.

There is no visible requirement that the first input lie in an 8-bit range. Likewise, the selector is not visibly narrowed to a byte before comparison. Under the declared interface, a selector whose low byte is 120 but whose full integer value differs from 120 does not enable special handling.

For subtraction cases, a portable mathematical-result guarantee requires the first input minus 48 to be representable in the signed result type. Behavior outside that domain is **documented_only**, because the supplied material does not settle overflow semantics.

## Behavior

### Selector-dependent mapping

When the second input equals 120, the following mappings apply:

| First input | Character interpretation | Result |
|---:|:---:|---:|
| 65 or 97 | `A` or `a` | 10 |
| 66 or 98 | `B` or `b` | 11 |
| 67 or 99 | `C` or `c` | 12 |
| 68 or 100 | `D` or `d` | 13 |
| 69 or 101 | `E` or `e` | 14 |
| 70 or 102 | `F` or `f` | 15 |

These are exact integer matches. The character labels are explanatory; they do not imply locale processing, alphabetic classification, or an encoding conversion.

Both letter cases are accepted in this mode, but the selector itself has no corresponding case-insensitive rule. Selector 88, conventionally ASCII `X`, is an ordinary nonmatching selector.

### Ordinary subtraction

The result is the first input minus 48 whenever:

- The selector is not 120; or
- The selector is 120, but the first input is not one of the twelve mapped values.

Consequently, integer codes 48 through 57 produce 0 through 9 with every selector. There is no separate decimal-digit validation step.

Inputs outside recognized digit ranges also take the subtraction path. For example, 47 produces −1 and 58 produces 10. The latter result overlaps a special letter result without indicating that the input was recognized as a valid digit.

No result is reserved as an error indicator. A negative result is an arithmetic outcome, not an established failure status.

### Decision and observable ordering

The selector determines whether the exceptional letter mappings are eligible. If eligible and matched, the function returns the associated constant rather than applying the subtraction rule. Otherwise, it returns the subtraction result.

There are no explicit observable reads from pointed-to storage, global storage, files, devices, or other external state. There are no explicit writes before the result is returned. The contract therefore establishes no ordering relationship among external reads or writes.

The supplied representation evaluates integer conditions internally. It does not provide a basis for promising a particular instruction sequence, timing profile, or constant-time execution.

### Arithmetic boundaries

All exceptional results, 10 through 15, are representable in any conforming signed `int` type.

The subtraction rule moves the mathematical value downward by 48. It therefore creates a lower-bound representability issue, not an upper-bound one. For a signed result type with minimum value \(L\), the mathematical result is representable when the first input is at least \(L + 48\).

If the interface uses 32-bit signed integers:

- The minimum is −2,147,483,648.
- The lowest subtraction input with a representable result is −2,147,483,600.
- That input produces −2,147,483,648.
- Inputs below −2,147,483,600 require overflow semantics not established here.

A wrapping result, trap, diagnostic, or other outcome for those lower-bound cases must not be invented from the supplied representation.

## Side effects

The explicit behavior has no externally visible state change other than returning an integer.

Specifically, it contains no visible:

- Allocation, deallocation, or ownership transfer.
- Buffer mutation or string traversal.
- Global-state access.
- I/O, logging, callback, or external function invocation.
- Error-object update or status-channel write.

There is no memory object whose lifetime the caller must extend for this function, because neither input denotes an object under the declared interface.

For inputs whose arithmetic is defined, the visible body contains no state-dependent source of variation: the result depends only on the two input values. This does not extend into a guarantee about instrumentation, machine faults, ABI violations, or overflow behavior.

## Dependencies

No dependency calls appear in the supplied body. No additional opaque function IDs are needed to describe its explicit behavior.

Execution still depends on the surrounding integer representation, arithmetic semantics, and calling convention. Those environmental properties are not established by this excerpt.

No assumptions about unknown callees preserving globals or managing memory are required, because no callee is visible.

## Acceptance examples

The following examples specify expected returned values where the arithmetic is representable. They are static acceptance criteria, not reported test results.

| First input | Selector | Expected result | Reason |
|---:|---:|---:|---|
| 65 | 120 | 10 | Special uppercase mapping. |
| 97 | 120 | 10 | Corresponding lowercase mapping. |
| 70 | 120 | 15 | Upper endpoint of the uppercase mapping. |
| 102 | 120 | 15 | Upper endpoint of the lowercase mapping. |
| 48 | 120 | 0 | Unmapped input uses subtraction. |
| 57 | 0 | 9 | Decimal-code subtraction does not require special mode. |
| 65 | 0 | 17 | Nonmatching selector disables the letter mapping. |
| 97 | 88 | 49 | Uppercase `X` is not selector 120. |
| 64 | 120 | 16 | Value immediately before uppercase `A` is not rejected. |
| 71 | 120 | 23 | Value immediately after uppercase `F` is not mapped. |
| 96 | 120 | 48 | Value immediately before lowercase `a` uses subtraction. |
| 103 | 120 | 55 | Value immediately after lowercase `f` uses subtraction. |
| 47 | 120 | −1 | Negative arithmetic results are returned normally. |
| 58 | 120 | 10 | A result of 10 does not prove letter recognition. |
| 0 | 0 | −48 | Zero has no special rejection behavior. |

Additional conditional acceptance criteria:

- Under a 32-bit signed interface, first input 321 with selector 120 produces 273. Sharing the low byte of 65 does not make it match 65.
- Under that same interface, first input 65 with selector 376 produces 17. Sharing the low byte of 120 does not make the selector match 120.
- Under defined signed arithmetic, equal input pairs produce equal results without requiring any persistent state.
- Under a 32-bit signed interface, first input −2,147,483,600 produces −2,147,483,648 with either selector 120 or selector 0.
- For a first input below that boundary, no exact result is accepted on the basis of this contract alone. That case remains **documented_only** pending evidence of execution semantics.

## Uncertainties

**Documented_only: exact integer width and ABI.** The signed `int` declaration is available; independent target-width and calling-convention evidence is not.

**Documented_only: overflow behavior.** The ordinary subtraction may fall below the signed minimum. The supplied material does not justify choosing wrapping, trapping, or another behavior.

**Documented_only: caller expectations.** No evidence establishes whether callers validate character ranges beforehand, interpret negative values as errors, or restrict selectors to character codes.

**Inferred intent: hexadecimal digit conversion.** The constants support this reading, but the function itself does not enforce hexadecimal validity or restrict results to 0 through 15.

## Evidence

This contract is based solely on static inspection of the supplied function representation. It establishes one exact selector comparison, twelve exceptional input mappings, and a subtraction fallback.

The representation contains no calls or explicit external-storage operations. That supports the side-effect and dependency descriptions above without relying on assumptions about other functions.

No runtime tests, execution traces, ABI inspection, or independent arithmetic-width verification have been performed. The acceptance examples are derived requirements, not observed execution output.
