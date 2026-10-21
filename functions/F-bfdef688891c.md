<!-- Trusted provenance: F-bfdef688891c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bfdef688891c

## Purpose

F-bfdef688891c selects a 32-bit result using a single 32-bit selector. It returns either zero, the fixed value `0x00001E00`, or a value obtained from one of four distinct global data references.

The established behavior is a small selector-based lookup. The meaning of the selected values is unknown. In particular, the supplied material does not establish that they represent sizes, addresses, flags, timing values, or status codes.

## Interface

### Input

One 32-bit argument supplies the selector.

The interface does not establish a signed integer type. Selection depends on exact equality with the values zero through five. Signedness does not change the outcome for any input bit pattern: only the five patterns representing one through five produce potentially nonzero results; every other pattern produces zero.

The argument is used as a value, not explicitly dereferenced as an address. No caller-owned buffer, length, output pointer, or object reference appears in the interface.

### Output

The function returns one 32-bit value.

| Selector | Result |
|---|---|
| `0x00000000` | `0x00000000` |
| `0x00000001` | `0x00001E00`, decimal 7680 |
| `0x00000002` | Value supplied by global data reference A, as returned through the 32-bit result |
| `0x00000003` | Value supplied by global data reference B, as returned through the 32-bit result |
| `0x00000004` | Value supplied by global data reference C, as returned through the 32-bit result |
| `0x00000005` | Value supplied by global data reference D, as returned through the 32-bit result |
| Every other 32-bit pattern | `0x00000000` |

References A through D are labels used by this contract to distinguish the four global data sources. They are not additional parameters or callable dependencies.

### Preconditions and representation limits

The supplied body contains no explicit validation or initialization checks.

For selectors two through five, normal completion depends on the selected global reference being readable under the execution environment. The supplied material does not establish how these references are initialized or whether their accessibility is guaranteed.

The input and return widths are established as 32 bits. The declarations and storage widths of the four global references are not supplied. Consequently, their exact load widths and any conversions into the 32-bit result are documented_only unknowns.

No calling convention, register assignment, stack layout, or byte order is established.

## Behavior

### Selection

The function chooses behavior solely from the input selector.

Selector zero reaches a zero-returning completion path. Selector one returns the fixed constant `0x00001E00`. Selectors two, three, four, and five each return a value from their respective global reference. Every selector outside zero through five also reaches a zero-returning completion path.

There is no observable distinction in the returned value between selector zero and an unsupported selector. The interface does not provide a separate validity indicator.

The supplied body does not perform arithmetic on the selector or the selected value. There is no shown scaling, offset calculation, masking, accumulation, or overflow-sensitive operation. In particular, the constant returned for selector one is not shown as the result of a calculation.

### Global-value selection and read ordering

At the level represented by the supplied body:

- Selector zero does not evaluate any of the four global data references.
- Selector one does not evaluate any of the four global data references.
- Selector two evaluates reference A, but not B, C, or D.
- Selector three evaluates reference B, but not A, C, or D.
- Selector four evaluates reference C, but not A, B, or D.
- Selector five evaluates reference D, but not A, B, or C.
- Any unsupported selector does not evaluate any of the four global data references.

For a global-backed result, the observable relationship is selection, evaluation of the selected reference, and return of the resulting 32-bit value. No subsequent transformation is shown.

The four references are distinct in the supplied representation. Their presentation as four selectable values does not establish an array, a contiguous layout, a common allocation, or a shared ownership model.

This contract does not elevate a source-level reference evaluation into a guarantee about the number or width of machine-level memory transactions. Those details require evidence beyond the supplied body.

### State dependence and repeated calls

Selectors zero, one, and all unsupported selectors have results independent of the four global values, as represented by the supplied body.

Selectors two through five depend on the selected global value when that reference is evaluated. The function contains no visible cache, remembered selector, counter, or explicit state update.

Accordingly, repeated calls with the same global-backed selector may return different values if the corresponding global data changes between evaluations. The function does not establish how such changes occur or whether concurrent changes are permitted.

### Zero-valued results

A zero result is not uniquely diagnostic:

- Selector zero returns zero.
- Every unsupported selector returns zero.
- Any of the four global-backed selectors may return zero if its selected reference supplies zero.

A caller cannot infer selector validity from a zero result alone.

## Side effects

The supplied body shows no explicit memory writes, allocations, deallocations, I/O operations, callbacks, or synchronization operations.

Its explicit state interaction is limited to evaluating one selected global data reference for selectors two through five. It does not explicitly read all four values before choosing a result.

No ownership transfer is expressed. The function neither accepts an allocation to release nor visibly creates one. Although a returned 32-bit value could be interpreted as an address by surrounding code, this body does not establish pointer semantics, object lifetime, or a release obligation.

There are no calls whose unknown effects must be assumed or accounted for. Nevertheless, the nature of the global storage remains unspecified. If a selected reference has hardware-specific or otherwise externally observable read behavior, that behavior cannot be characterized from this body alone.

No explicit error state, diagnostic message, or failure return is established. Unsupported selectors return zero rather than following a separate error-reporting path. Fault handling for inaccessible global storage is not shown.

## Dependencies

No callable dependencies appear in the supplied body. There are therefore no dependency function IDs to enumerate.

The data dependencies are:

| Contract label | Selected by | Established role |
|---|---:|---|
| Global reference A | 2 | Supplies that selector’s return value |
| Global reference B | 3 | Supplies that selector’s return value |
| Global reference C | 4 | Supplies that selector’s return value |
| Global reference D | 5 | Supplies that selector’s return value |

The values, declared types, initialization mechanisms, mutability, storage lifetimes, and synchronization requirements of these references are not supplied.

No assumption that they are immutable constants is necessary for the selection contract.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

1. **Zero selector:** Given input `0x00000000`, normal completion returns `0x00000000` without evaluating references A through D.

2. **Fixed-value selector:** Given input `0x00000001`, normal completion returns `0x00001E00`, regardless of the values supplied by A through D. None of those references is evaluated.

3. **Reference A selection:** If reference A supplies `0x12345678` without a representation-changing conversion, input `0x00000002` returns `0x12345678`. References B, C, and D are not evaluated.

4. **Reference B selection:** If reference B supplies `0x00000009`, input `0x00000003` returns `0x00000009`. A different value in reference A does not affect this result.

5. **Reference C selection:** If reference C supplies zero, input `0x00000004` returns zero. This zero remains a selected data value, not evidence of an unsupported selector.

6. **Reference D selection:** If reference D supplies `0xFFFFFFFF` without a representation-changing conversion, input `0x00000005` returns the 32-bit pattern `0xFFFFFFFF`. The contract does not assign signed meaning to that result.

7. **First unsupported positive selector:** Input `0x00000006` returns zero without evaluating any of the four global references.

8. **High-bit selector:** Input `0xFFFFFFFF` returns zero. Treating that input as signed negative one does not make it match any supported nonzero case.

9. **No low-bit-only matching:** Input `0x00000102` returns zero, rather than selecting reference A. Selection is not shown as truncating the selector to its low byte.

10. **Changed global value:** If two normally completing calls use selector two, and reference A supplies different values on the two evaluations, each call returns its respective selected value. The supplied body shows no mechanism that forces reuse of the first result.

## Uncertainties

The following remain documented_only:

- The application-level meaning of the selector and each result.
- The declared types and physical storage widths of references A through D.
- Any sign extension, zero extension, truncation, or other conversion associated with those references.
- Their concrete memory layout and whether they share an allocation.
- Their initial values, initialization order, and mutability.
- Their volatility, atomicity, and behavior under concurrent modification.
- Any hardware-specific effects of reading their storage.
- Fault behavior if a selected reference is inaccessible.
- The calling convention and machine-level memory transaction sequence.
- Whether surrounding callers treat returned values as integers, addresses, flags, or another representation.

An inferred intent is that the function exposes a small set of selectable values. That interpretation is not required for, and does not expand, the established behavior.

## Evidence

The supplied static body establishes one 32-bit input, one 32-bit return value, explicit cases for zero through five, one fixed nonzero result, four separately selected global references, and a zero result for all remaining inputs.

It also shows no explicit calls, writes, allocation operations, release operations, arithmetic, or additional validation.

The global declarations, callers, initialization logic, and execution environment were not supplied. No implementation beyond the embedded body was inspected. No runtime tests have been performed.
