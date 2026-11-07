<!-- Trusted provenance: F-c75ef411cf3c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c75ef411cf3c

## Purpose

F-c75ef411cf3c dispatches a property request against one of eight accepted record identifiers. It obtains a context through a dependency, validates the identifier, selects a record-relative property, and either delegates output production to a conversion dependency or copies one 32-bit field directly to caller-provided storage.

The established behavior is a selector-based accessor with a 32-bit success indicator. Domain meanings for the records, properties, and converted values are not established. In particular, the available evidence does not justify naming any property as a color, vector, coordinate, flag, or floating-point value.

Successful dispatch returns 1. An unavailable context or an unsupported identifier or selector returns 0. Unsupported requests additionally invoke an error-reporting dependency when a context is available.

## Interface

### Inputs

The function has three inputs:

| Input | Established representation | Contract |
|---|---|---|
| Record identifier | 32-bit integer | Accepted bit patterns are 0x4000 through 0x4007, inclusive. |
| Property selector | 32-bit value | Recognized selectors are 0x1200 through 0x1209, inclusive. |
| Destination | Pointer to 32-bit storage | Receives one directly copied 32-bit value for scalar selectors, or is passed to an output dependency for other selectors. |

The destination’s required capacity, alignment, and representation depend on the selector. Only the scalar cases establish a four-byte output write directly in this function. The sizes of intermediate temporary buffers do not establish the sizes of delegated outputs.

There is no explicit destination-null check. A null destination is therefore not a supported general-purpose query mechanism. Some early-return paths do not use the destination, but that does not establish a broader null-pointer contract.

### Return value

The return value is 32 bits:

- **0:** The context dependency returned zero, or identifier/selector validation failed.
- **1:** A recognized property path completed its visible processing and its dependencies returned normally.

For delegated paths, returning 1 does not establish that the output dependency successfully produced a meaningful result. Dependency return values are not tested.

No other normal return value is visible.

### Address model

Let the context value be the nonzero result of F-7fba0909c452, and let the record bias be the address-calculation value read from external storage.

The selected record address is formed from:

- the context value;
- the **original record identifier multiplied by 168**, equivalent to 0xA8 bytes per identifier;
- the external record bias.

The function does not visibly replace the identifier with its zero-based position within the accepted range. An implementation must not silently substitute “identifier minus 0x4000” while leaving the bias unchanged.

The 32-bit identifier range check is established. Exact address-overflow semantics and the validity of the resulting addresses are not established independently of the target execution environment.

## Behavior

### 1. Context acquisition precedes validation

The first visible operation is a call to F-7fba0909c452 with no arguments.

If it returns zero, the function returns 0 immediately:

- The record identifier is not validated.
- The selector is not dispatched.
- The record bias and record fields are not visibly read.
- The destination is not directly accessed.
- The error-reporting dependency is not called.

Any side effects of context acquisition remain possible even on this early-return path.

### 2. Identifier validation

With a nonzero context, the function checks the identifier through an unsigned 32-bit subtraction from 0x4000 followed by comparison with 7.

This accepts exactly:

- 0x4000
- 0x4001
- 0x4002
- 0x4003
- 0x4004
- 0x4005
- 0x4006
- 0x4007

Values below the range are rejected through unsigned wraparound, rather than accepted as negative offsets. Values above the range are also rejected.

On rejection, the function calls F-375c8199e7c2 with the acquired context and the error value 0x500, then returns 0. It does not visibly calculate a record address or directly write the destination.

### 3. Record selection and selector dispatch

For an accepted identifier, the function forms the record address using the context, identifier, stride, and external bias described above. It then dispatches on the property selector.

Address formation precedes selector rejection. An unsupported selector therefore differs from an invalid identifier: the record address calculation has already occurred, although no record-field dereference is visible on the unsupported-selector path.

The supported property paths are:

| Selector | Record-relative location | Visible processing |
|---|---:|---|
| 0x1200 | +0x00 | Prepare an eight-byte temporary through F-a325b0adc5e9, then pass it and the destination to F-01be51f7fed7. |
| 0x1201 | +0x10 | Same dependency sequence, using this source address. |
| 0x1202 | +0x20 | Same dependency sequence, using this source address. |
| 0x1203 | +0x30 through +0x3F | Snapshot four consecutive 32-bit fields into a 16-byte temporary, then pass the temporary and destination to F-e1bd00d0c4e2. |
| 0x1204 | +0x4C | Prepare a 12-byte temporary through F-eaf776d4a45e, then pass it and the destination to F-230deefc2f3b. |
| 0x1205 | +0x64 | Read one 32-bit field and write it directly to the destination. |
| 0x1206 | +0x68 | Read one 32-bit field and write it directly to the destination. |
| 0x1207 | +0x58 | Read one 32-bit field and write it directly to the destination. |
| 0x1208 | +0x5C | Read one 32-bit field and write it directly to the destination. |
| 0x1209 | +0x60 | Read one 32-bit field and write it directly to the destination. |

All paths in this table return 1 after the described operations complete normally.

Any other selector calls F-375c8199e7c2 with the acquired context and 0x500, then returns 0.

### 4. Two-stage delegated paths

For selectors 0x1200, 0x1201, and 0x1202:

1. F-a325b0adc5e9 receives the address of an eight-byte temporary as its first argument and the selected record-relative address as its second.
2. After that dependency returns, F-01be51f7fed7 receives the temporary address and the caller’s destination.
3. The function returns 1 after the second dependency returns.

For selector 0x1204, the same ordering applies with a 12-byte temporary and dependencies F-eaf776d4a45e followed by F-230deefc2f3b.

No explicit initialization of these temporaries is visible before their preparation calls. Their complete initialization, internal format, and interpretation are dependency contracts, not established facts about this function.

The source offsets do not establish the number of bytes read by the preparation dependencies. For example, the 0x10 spacing of the first three property locations does not prove a 16-byte read at each location.

### 5. Four-field snapshot path

For selector 0x1203, the function explicitly reads four 32-bit fields in this order:

1. Record offset +0x30.
2. Record offset +0x34.
3. Record offset +0x38.
4. Record offset +0x3C.

It places the values, in the same order, into consecutive positions of a 16-byte temporary. Only after all four reads does it call F-e1bd00d0c4e2 with the temporary address and the destination.

This establishes a snapshot-before-delegation sequence, not an atomic 16-byte snapshot. No synchronization is visible. Concurrent modification of the record could therefore produce values from different moments.

The function does not itself copy these 16 bytes directly to the destination.

### 6. Scalar paths

Selectors 0x1205 through 0x1209 each perform one visible four-byte source read followed by one four-byte destination write. The value is transferred without visible arithmetic, masking, validation, or numeric conversion.

The copied bits have no established numeric interpretation. Signedness, floating-point meaning, and application-level units remain unknown.

## Side effects

### Directly established effects

- Context acquisition occurs on every invocation.
- Invalid identifiers and unsupported selectors invoke the error dependency only after successful context acquisition.
- Each scalar success path directly writes four bytes to the destination.
- Selector 0x1203 writes four fields into local temporary storage before invoking its output dependency.
- Other delegated success paths provide temporary storage to dependencies.
- No direct write to a record field or to the external record bias is visible.
- No explicit allocation or release operation is visible in this function.

### Dependency-mediated effects

No supplied dependency body establishes whether a dependency:

- mutates records, globals, or context state;
- allocates or releases memory;
- changes an error indicator;
- accesses storage reachable through arguments;
- retains a supplied pointer;
- returns normally for every input.

Accordingly, a return of 0 does not imply that the call had no side effects. Likewise, a successful accessor path is not established to be globally read-only.

The function does not visibly restore state changed by dependencies.

### Ownership and lifetime

The destination originates with the caller. This function does not explicitly release it or visibly establish a transfer of ownership.

Intermediate temporaries have invocation-local lifetime. Their addresses are supplied synchronously to dependencies, but pointer non-retention cannot be assumed without those dependencies’ contracts. Retaining a temporary’s address beyond the invocation would create a lifetime hazard.

No cleanup is visible after a delegated output operation. Whether cleanup is unnecessary or performed inside a dependency is unknown.

### Aliasing and failures

No non-aliasing requirement is checked. If a scalar destination overlaps its selected source, the visible operation still reads the source value before writing the destination.

For selector 0x1203, all four source fields are read before the output dependency is called. This ordering remains relevant if the destination overlaps those fields, but the dependency’s eventual writes are unknown.

There is no visible recovery from an invalid memory access or an abnormal dependency exit. Such failures are not established to become a return value of 0.

## Dependencies

| Opaque function ID | Role established at this call site |
|---|---|
| F-7fba0909c452 | Called first without arguments; supplies a context value, with zero causing immediate failure. |
| F-375c8199e7c2 | Receives the context and 0x500 for invalid identifiers or unsupported selectors. |
| F-a325b0adc5e9 | Receives an eight-byte temporary address and a selected record-relative address for selectors 0x1200–0x1202. |
| F-01be51f7fed7 | Receives that eight-byte temporary address and the caller’s destination. |
| F-e1bd00d0c4e2 | Receives the four-field, 16-byte snapshot and the caller’s destination for selector 0x1203. |
| F-eaf776d4a45e | Receives a 12-byte temporary address and record offset +0x4C for selector 0x1204. |
| F-230deefc2f3b | Receives that 12-byte temporary address and the caller’s destination. |

These roles describe observed argument flow, not complete dependency specifications. In particular, no return-status handling, ownership behavior, or global-state preservation is inferred from a dependency’s role.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

1. **Unavailable context takes precedence.**  
   If F-7fba0909c452 returns zero, even with an invalid identifier and unsupported selector, the result is 0 and F-375c8199e7c2 is not called.

2. **Lower identifier boundary is accepted.**  
   With a nonzero context and otherwise valid memory, identifier 0x4000 reaches selector dispatch.

3. **Upper identifier boundary is accepted.**  
   Under the same conditions, identifier 0x4007 reaches selector dispatch.

4. **Immediately adjacent identifiers are rejected.**  
   With a nonzero context, identifiers 0x3FFF and 0x4008 each cause an error-dependency call carrying that context and 0x500, followed by return 0.

5. **Unsupported selectors use the same error value.**  
   With identifier 0x4000 and a nonzero context, selector 0x120A causes the error-dependency call and return 0. No direct destination write occurs.

6. **Scalar bits are preserved.**  
   For selector 0x1205, if the selected field at +0x64 contains the 32-bit pattern 0x89ABCDEF and both accesses are valid, the destination receives that same pattern and the function returns 1.

7. **Selector order is not field-offset order.**  
   Selector 0x1206 reads +0x68, whereas selector 0x1207 reads +0x58. A conforming implementation must preserve this mapping.

8. **Four-field input precedes delegated output.**  
   For selector 0x1203, F-e1bd00d0c4e2 is called only after the four ordered reads at +0x30, +0x34, +0x38, and +0x3C. Its first argument exposes those four values consecutively.

9. **Delegated completion is not independently validated.**  
   For selector 0x1201, if both dependencies return normally, the function returns 1 without testing a dependency result. The evidence alone cannot specify the destination’s final bytes.

10. **Record stride uses the supplied identifier.**  
    Holding the context and bias fixed, adjacent accepted identifiers select record addresses separated by 0xA8 bytes, subject to the target’s address arithmetic.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning of the context, records, selectors, and error value 0x500.
- The external record bias’s value, initialization, mutability, and intended relationship to identifier 0x4000.
- The exact machine-level behavior of address overflow and pointer conversion.
- Endianness and numeric interpretations of stored fields.
- Dependency read footprints, output formats, output capacities, alignment requirements, and failure behavior.
- Whether preparation dependencies initialize every byte subsequently consumed.
- Whether dependencies preserve global state or retain, allocate, own, or release memory.
- Thread safety, reentrancy, and synchronization requirements.
- Behavior for invalid source or destination storage.

An accessor interpretation and conversion-oriented roles for the delegated paths are inferred intent. They do not establish application-level type names or stronger safety guarantees.

## Evidence

The supplied static representation establishes the control-flow branches, constants, explicit field widths, record-relative offsets, temporary sizes, argument ordering, and normal return values described above.

Dependency implementations and runtime observations were not supplied. No runtime tests have been performed. The acceptance examples are proposed behavioral checks, not observed results.
