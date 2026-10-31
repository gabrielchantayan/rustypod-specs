<!-- Trusted provenance: F-36b9d1fbbb1c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-36b9d1fbbb1c

## Purpose

F-36b9d1fbbb1c classifies a supplied integer by exact membership in a fixed six-value set. It returns the integer **1** for a member and the integer **0** for every other representable input.

The accepted values are:

| Decimal | Hexadecimal | Conventional interpretation in ASCII-compatible encodings |
|---:|---:|---|
| 0 | 0x00 | NUL |
| 9 | 0x09 | Horizontal tab |
| 10 | 0x0A | Line feed |
| 12 | 0x0C | Form feed |
| 13 | 0x0D | Carriage return |
| 32 | 0x20 | Space |

**Established behavior:** acceptance depends only on those integer values, not on character names or an encoding lookup.

**Inferred intent:** the function resembles a whitespace-or-terminator predicate. That interpretation is not its contract: zero is accepted, and vertical tab, decimal 11, is rejected. A replacement using a general-purpose whitespace classifier would therefore require independent proof of equivalence.

## Interface

The function accepts one integer argument by value and returns one integer result.

### Input domain and width

The supplied representation declares the input as an `int`. It does not establish the target ABI or the exact bit width of that type. Consequently:

- The behavioral domain is every value representable by the declared integer parameter.
- All six accepted constants are nonnegative and representable even in a 16-bit signed integer.
- The membership rule does not depend on overflow, wrapping, bit patterns, byte order, or arithmetic conversion of a large constant.
- The specification does not establish how a caller’s wider integer is converted before entry. Classification applies to the value actually received.
- The function does not establish whether a caller has sign-extended or zero-extended a narrower character value.

The return is also declared as an `int`. Its exact storage width remains unspecified, but its value is always exactly 0 or exactly 1.

### Preconditions and ownership

No pointer, buffer, object layout, length, or external state is supplied. No memory-validity or ownership precondition is visible.

The function neither takes ownership of an allocation nor returns an owned resource. Passing zero is a valid numeric input; it is not interpreted as a pointer.

The caller must obey the applicable calling convention. Argument registers, stack placement, register preservation, and return-value placement are not established by the supplied representation.

## Behavior

### Classification rule

The complete observable value contract is:

- Return **1** if the received integer equals 0, 9, 10, 12, 13, or 32.
- Return **0** otherwise.

There are no ranges of accepted values beyond those six individual integers. In particular, proximity to an accepted value has no effect. Decimal 8 is rejected despite being adjacent to 9, and decimal 11 is rejected despite lying between accepted values 10 and 12.

A positive result is normalized to the integer 1. The function does not return the original accepted input, an arbitrary nonzero truth value, a category mask, or a count. Thus an input of 32 returns 1, not 32.

### Decision consistency

The supplied body establishes equality facts for 32, 9, 13, 10, and 12, together with whether the original input is nonzero. Those facts are obtained before the candidate return value is changed.

The acceptance condition and rejection condition cover the entire input domain and are mutually exclusive:

- Equality with any of the five nonzero accepted values, or equality with zero, selects acceptance.
- Inequality with all five nonzero accepted values, together with being nonzero, selects rejection.

This matters because the representation contains separate conditional updates rather than an explicit alternative branch. The rejection decision still concerns the original input; it does not reclassify the normalized value written by acceptance. There is no input for which both outcomes apply, and no input for which the original argument survives as the final result without normalization.

### Arithmetic and boundaries

The classification requires equality and inequality comparisons only. There is no addition, subtraction, multiplication, division, shifting, indexing, or narrowing operation in the visible body.

Accordingly:

- Every negative representable input returns 0.
- Every representable input greater than 32 returns 0.
- The minimum and maximum representable parameter values return 0.
- No overflow-dependent exception exists at either boundary.
- Values sharing a low byte with an accepted value are not accepted merely for that reason.

The function does not mask the input to eight bits or interpret it as a sequence of bytes.

### Completion and errors

The visible body has no loop, dependency call, error branch, or exceptional-result value. For a valid invocation under the target execution model, it reaches a normal return with 0 or 1.

No diagnostic, error object, or distinction between different rejected values is exposed. Rejection is a normal classification result, not a reported execution failure.

## Side effects

No externally visible data mutation is present in the supplied body.

The visible data operations are confined to the by-value argument and temporary decision state. Changing that local argument does not write back to a caller’s variable.

There are no visible:

- Reads or writes through pointers.
- Reads or writes of global or static storage.
- Allocations, releases, or ownership transfers.
- File, terminal, network, or device operations.
- Locale queries or character-table accesses.
- Calls to other functions.

Consequently, the source-level result has no visible dependence on prior calls, mutable application state, or locale.

There is no externally observable read/write sequence to preserve beyond receiving the argument and returning the result. The internal comparison order described above does not imply a required machine-instruction order. Stack traffic, register spills, instrumentation, and other target-specific execution details are not established here.

## Dependencies

No function dependencies are visible. F-36b9d1fbbb1c performs its classification directly.

No dependency ownership or global-preservation guarantees are needed to explain this body because it makes no calls. This observation does not establish behavior for caller-side wrappers, instrumentation, or other code not supplied.

The function does not visibly depend on a standard-library whitespace predicate. Its numeric membership set is authoritative even where a platform’s character-classification rules differ.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reported runtime observations.

| Received input | Required return | Reason |
|---:|---:|---|
| 0 | 1 | Zero is explicitly accepted. |
| 9 | 1 | Exact accepted value. |
| 10 | 1 | Exact accepted value. |
| 12 | 1 | Exact accepted value. |
| 13 | 1 | Exact accepted value. |
| 32 | 1 | Exact accepted value; result is normalized. |
| 1 | 0 | Nonzero and outside the accepted set. |
| 8 | 0 | Adjacency to 9 does not confer acceptance. |
| 11 | 0 | Vertical tab is not included. |
| 14 | 0 | Outside the accepted set. |
| 31 | 0 | Adjacency to 32 does not confer acceptance. |
| 33 | 0 | Outside the accepted set. |
| −1 | 0 | Negative values cannot equal an accepted constant. |

Additional conditional criteria:

- If 256 is representable by the parameter type, receiving 256 must return 0. Its low byte being zero does not make it equivalent to input zero.
- Receiving the parameter type’s minimum or maximum representable value must return 0.
- Repeated valid invocations with the same received integer must produce the same result under the visible body, regardless of locale.
- A caller that passes an ordinary integer variable by value must not observe that variable being changed by this function.

## Uncertainties

The following remain **documented_only**:

- Exact parameter and return widths, target ABI, and calling convention.
- Caller-side conversions before invocation.
- Whether callers use the result as a Boolean, a parser delimiter test, or another predicate.
- The intended encoding, if any. Character labels above explain common numeric meanings but do not establish encoding-dependent behavior.
- The reason zero is accepted and vertical tab is excluded.
- Machine-level instruction order and any execution effects introduced outside the supplied body.

None of these unknowns changes the six-value classification rule for an integer already received by the function.

## Evidence

The supplied static representation shows five equality checks against the nonzero accepted constants, one nonzero check, complementary acceptance and rejection conditions, normalization to 1 or 0, and return of that normalized integer.

It shows no calls, pointer accesses, global accesses, arithmetic transformations, or resource operations. Those observations support the value and side-effect contract above.

The whitespace-or-terminator interpretation is inferred intent only. No runtime tests have been performed; the acceptance examples are static contract criteria, not execution evidence.
