<!-- Trusted provenance: F-38ffb5fafd28; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-38ffb5fafd28

## Purpose

F-38ffb5fafd28 maps eight consecutive 32-bit input values, from `0x00000200` through `0x00000207` inclusive, to the corresponding zero-based values `0` through `7`. Every other input produces the 32-bit result `0xFFFFFFFF`.

The established behavior is an exact-value mapping. It does not accept an input merely because its low-order bits match an accepted value.

The apparent intent is to normalize a small group of numeric identifiers into compact indices, using an all-ones result for nonmembers. The normalization behavior is established; the meanings of the identifiers, the intended consumers, and the interpretation of the nonmember result are not.

## Interface

### Input

The function takes one 32-bit scalar value.

The supplied representation does not establish whether callers regard that value as signed, unsigned, or an opaque identifier. This distinction does not change the mapping when inputs are specified by their 32-bit patterns.

There are exactly eight accepted patterns:

| Input, hexadecimal | Input, decimal |
|---|---:|
| `0x00000200` | 512 |
| `0x00000201` | 513 |
| `0x00000202` | 514 |
| `0x00000203` | 515 |
| `0x00000204` | 516 |
| `0x00000205` | 517 |
| `0x00000206` | 518 |
| `0x00000207` | 519 |

No input structure, buffer, pointer target, length, encoding, alignment requirement, or ownership transfer is exposed by the supplied function.

Behavior for values wider than 32 bits is outside this interface. Any conversion from a wider caller-side value into the 32-bit argument occurs outside the established contract.

### Output

The function returns one 32-bit scalar result:

- Accepted inputs produce a value in the inclusive range `0` through `7`.
- All other inputs produce exactly `0xFFFFFFFF`.

The nonmember result has every bit set. If interpreted as an unsigned 32-bit integer, it is 4,294,967,295. If interpreted using signed two’s-complement 32-bit representation, it is −1. The contract requires the bit pattern; it does not establish which interpretation callers use.

There are no additional output parameters or returned aggregate layouts.

## Behavior

### Exact mapping

The complete accepted mapping is:

| Input | Required result |
|---|---|
| `0x00000200` | `0x00000000` |
| `0x00000201` | `0x00000001` |
| `0x00000202` | `0x00000002` |
| `0x00000203` | `0x00000003` |
| `0x00000204` | `0x00000004` |
| `0x00000205` | `0x00000005` |
| `0x00000206` | `0x00000006` |
| `0x00000207` | `0x00000007` |

For an accepted input \(x\), the returned value can be described mathematically as \(x - 512\). This is a description of the result, not a requirement to perform subtraction.

For every input outside that eight-value set, the required result is `0xFFFFFFFF`. In particular, the function does not:

- Extend the mapping below 512 or above 519.
- Clamp an out-of-range value to either accepted boundary.
- Wrap an out-of-range value into the accepted range.
- Mask away high-order bits before matching.
- Return the input unchanged for an unrecognized value.

### Width and arithmetic boundaries

All accepted inputs and their corresponding results fit within both signed and unsigned 32-bit integer ranges. Consequently, the accepted mapping has no signedness-dependent boundary case.

The mathematical subtraction description applies only to accepted inputs. Applying subtraction to all possible inputs would not describe this function: for example, `0x00000208` must return `0xFFFFFFFF`, not `0x00000008`.

An implementation may use a different internal selection mechanism, but it must preserve full-width input matching and the exact 32-bit output patterns. No intermediate arithmetic behavior, overflow mode, or instruction sequence is required by this contract.

### Evaluation and observable order

The established operation consumes the scalar argument, selects its corresponding result, and returns that result.

There are no source-visible reads from external objects and no source-visible writes to external objects. Accordingly, there is no ordering contract between multiple external memory accesses, because none are exposed.

The supplied representation does not establish the order or number of underlying comparisons, register accesses, or instruction fetches. Those details are not part of the behavioral contract.

### Preconditions and failures

No value-level precondition restricts the 32-bit input domain. Every possible 32-bit pattern has a specified result.

An unrecognized input is handled by returning `0xFFFFFFFF`; there is no separate error output. No diagnostic, exception, callback, retry, or termination operation is present in the supplied body.

The representation contains no loop or dependency call. No input-dependent nontermination path is visible. Execution still presupposes an ordinary valid invocation; behavior under an invalid calling convention or external execution fault is not established.

## Side effects

The supplied function exposes no state mutation.

Specifically, it contains no visible operation that:

- Reads or modifies a global object.
- Dereferences caller-provided memory.
- Writes through an output address.
- Allocates, releases, retains, or transfers memory.
- Performs input/output.
- Invokes another function.

There is therefore no buffer ownership protocol or resource lifecycle for a caller to satisfy.

For the same 32-bit input, the supplied body selects the same return pattern independently of prior invocations. No persistent state is visible that could alter a later result.

These statements concern the behavior expressed by the supplied representation. They do not assert a particular stack layout, absence of compiler-generated instrumentation, timing characteristics, or guarantees about an otherwise unspecified execution environment.

## Dependencies

No dependency function is invoked by F-38ffb5fafd28 in the supplied representation.

The mapping does not consult an external lookup service, callback, mutable table, or configuration source. Its specified results are fully determined by the input value.

No assumptions about another function preserving globals or acquiring or releasing memory are needed: there is no visible callee on any path.

The calling convention, caller behavior, and broader application protocol remain unspecified. In particular, this contract does not require a caller to reject, propagate, reinterpret, or otherwise act on `0xFFFFFFFF`.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

### Complete accepted set

If the function is invoked separately with each of the eight accepted inputs, each invocation must return the corresponding result in the exact-mapping table. Both endpoints are included: 512 maps to 0, and 519 maps to 7.

### Adjacent rejected values

If the input is `0x000001FF` (511), the result must be `0xFFFFFFFF`.

If the input is `0x00000208` (520), the result must also be `0xFFFFFFFF`.

Together, these cases distinguish the specified finite mapping from an unrestricted offset calculation.

### Full-width matching

If the input is `0x00010200`, the result must be `0xFFFFFFFF`, despite its lower 16 bits matching `0x0200`.

If the input is `0x80000207`, the result must be `0xFFFFFFFF`, despite its low-order portion resembling the final accepted input.

These cases distinguish exact 32-bit matching from narrowing or masking before selection.

### Other nonmember patterns

| Input | Required result |
|---|---|
| `0x00000000` | `0xFFFFFFFF` |
| `0x00000007` | `0xFFFFFFFF` |
| `0x7FFFFFFF` | `0xFFFFFFFF` |
| `0x80000000` | `0xFFFFFFFF` |
| `0xFFFFFFFF` | `0xFFFFFFFF` |

The final row is a rejection case, not evidence of a general identity mapping.

### Repeated and interleaved invocations

If invocations receive 512, then 520, then 512, their required results are respectively `0x00000000`, `0xFFFFFFFF`, and `0x00000000`.

The intervening nonmember input must not change the mapping of the later accepted input. No state transition is specified.

## Uncertainties

### Inferred intent

The consecutive accepted values and zero-based results suggest identifier-to-index normalization. The all-ones nonmember result suggests a sentinel. Neither the identifiers’ domain meaning nor a caller-side error-handling policy is established.

### documented_only

The following remain unknown and are not acceptance requirements:

- The original signedness and semantic types of the argument and result.
- The external ABI, register assignment, and calling convention.
- Whether callers interpret `0xFFFFFFFF` as −1, an unsigned sentinel, or another application-specific marker.
- Whether the application normally supplies only accepted values.
- Any upstream conversion of values wider than 32 bits.
- Instruction selection, execution timing, and constant-time properties.
- Platform-specific instrumentation or execution-environment effects.

These unknowns do not affect the specified mapping of 32-bit input patterns to 32-bit result patterns.

## Evidence

The supplied static representation establishes one 32-bit scalar input, one 32-bit scalar return, eight explicit accepted values with results 0 through 7, and a default result of `0xFFFFFFFF`.

It contains no visible dependency calls, memory dereferences, global accesses, state updates, or resource-management operations.

The tables, boundary conditions, and acceptance examples above follow from that finite mapping and its default outcome. No runtime tests, execution traces, timing measurements, or independent binary observations have been performed.
