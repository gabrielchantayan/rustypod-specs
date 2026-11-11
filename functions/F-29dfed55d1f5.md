<!-- Trusted provenance: F-29dfed55d1f5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-29dfed55d1f5

## Purpose

The established behavior is a sequence of dependency calls and memory updates culminating in a 20-byte record written to a dependency-supplied destination. Earlier updates affect several other addresses and are part of the observable contract; the operation is not limited to producing the final record.

No application-level purpose is established. In particular, the evidence does not identify an allocator, parser, renderer, serializer, or constructor. The final destination could be newly obtained storage or an existing object.

This contract distinguishes:

- **Established static behavior:** accesses, call arguments, value relationships, and ordering supported by the supplied representation.
- **Inferred intent:** possible interpretations that do not impose requirements.
- **documented_only:** unresolved details and displayed control flow that cannot be reached under the established byte ranges.

Successful execution is conditional on valid memory and returning dependencies. No runtime execution has been performed.

## Interface

### Inputs and result

The operation accepts two 32-bit inputs:

| Input | Established use |
|---|---|
| Arithmetic input | Supplies an argument to the first dependency and determines one byte-store address. |
| Address input | Supplies the address of an initial byte read, a selected signed halfword read, and the first argument to the first dependency. |

These names describe use, not declared application types. The arithmetic input is not established to be an index, count, or valid pointer.

The representation exposes no explicit return value. On ordinary completion, the externally useful result consists of memory effects, including the final record. There is no established success status, error code, or returned ownership token.

Several dependency results are represented as 64-bit values whose lower and upper 32-bit portions are consumed separately. This establishes the caller’s interpretation, not the underlying calling convention.

### Widths and arithmetic

The contract uses:

- Unsigned 8-bit reads for both selector bytes.
- A signed 16-bit read, extended to the 32-bit argument width.
- An unsigned 16-bit read only within the nonselected alternative branch.
- Explicit 8-, 16-, 32-, and 64-bit stores.
- 32-bit address-bearing values and arithmetic.

Address calculations are described using 32-bit machine values, retaining the low 32 bits where necessary. This is not a claim that an equivalent signed source-language expression would have defined overflow behavior. Alignment requirements, address-space mapping, and byte order remain unresolved.

### Conditions for completion

The caller or execution environment must provide readable and writable memory for the accesses described below, callable dependency targets, and dependency results usable by subsequent accesses. The ordinary final destination must support writes at offsets 0 through 19.

There are no visible checks for null values, bounds, alignment, capacity, or dependency failure. These are execution preconditions, not validations performed by the operation.

## Behavior

### Data provenance and captured values

Two independent byte reads govern the ordinary path.

The **initial selector** comes from the address input at offset 17. It selects a signed 16-bit value beginning at offset 74 plus the selector, measured in bytes. Thus, the selected read can begin anywhere from offset 74 through offset 329 and can extend through offset 330. The selector is not scaled by the halfword width.

The selected signed value is captured before the first dependency call. Later changes to the input memory do not cause this value or the initial selector to be read again.

The first dependency supplies a split result:

- Its lower 32 bits contribute to the next dependency’s argument.
- Twice its upper 32-bit portion, truncated to 16 bits, supplies the **payload**.

The payload is always even. It depends only on the lower 15 bits of that upper portion; bits above those do not affect the payload. The two payload byte stores retain only the payload’s low eight bits.

A separate 32-bit global supplies the **captured base**. This global is read after the assignment of 71 to the low-address global and before F-a00000000006. Earlier dependency calls may have changed it. Later dependency calls are not assumed to preserve it, but the ordinary path uses the captured value rather than rereading the global.

The result of F-a00000000006 supplies the **count byte** through an unsigned byte read at offset 13 from its upper 32-bit portion. The lower portion supplies the argument to F-a00000000007.

The count byte determines the final **adjusted count**, equal to twice that byte plus 99. Its range is 99 through 609. The other two scalar fields in the final record are the captured base plus 108 and the captured base minus 1.

### Memory-update contract

Excluding dependency-internal effects, the ordinary path has the following writes:

| Destination | Width | Required value |
|---|---:|---|
| Result of F-a00000000003, offset 4 | 16 bits | Low 16 bits of that same result |
| Twice the arithmetic input plus 9 | 8 bits | Low eight bits of the payload |
| Result of F-a00000000005, offset 20 | 16 bits | Payload |
| Result of F-a00000000005, offset 26 | 8 bits | Low eight bits of the payload |
| Global at absolute address 162 | Unresolved | 71 |
| Captured base plus count byte minus 2 | 32 bits | 51 |
| Final destination, offsets 0–19 | Mixed | Final record described below |

The first write stores an address-derived bit pattern through the address from which it was derived. It is not established to represent a length, identifier, reference count, or pointer field.

The constant 51 is stored as a full 32-bit value. In particular, a count byte of zero does not suppress this write: its starting address is then two bytes before the captured base. No alignment adjustment is shown.

The width of the assignment at absolute address 162 cannot be recovered from the supplied evidence. Neither its small address nor its value establishes a byte-sized write. Adjacent bytes must not be assumed unaffected.

### Final record

F-c260788e6b26 receives 115 and supplies a 64-bit result. Its upper 32-bit portion is the final destination address. The complete result is also the first field written there.

| Byte offset | Field width | Field value |
|---|---:|---|
| 0 | 64 bits | Complete result of F-c260788e6b26 |
| 8 | 32 bits | Adjusted count |
| 12 | 32 bits | Captured base plus 108 |
| 16 | 32 bits | Captured base minus 1 |

These writes occur in ascending field order. There is no dependency call between them or between the final field write and ordinary return.

The first field is represented as one 64-bit assignment. This does not establish atomicity, physical transaction count, alignment tolerance, or byte order. The destination is derived from the dependency result, not reread from the first field after that field has been stored.

The record occupies 20 bytes when interpreted as these byte offsets. No padding, tag, checksum, terminator, or additional initialized region is established.

### Reachability and termination

The displayed invalid-instruction termination depends on a signed-overflow indication for adding 91 to the initial selector. An unsigned byte plus 91 is between 91 and 346, so signed 32-bit overflow cannot occur. The termination is therefore not selected under the displayed widths and value flow.

The other displayed branch requires the adjusted count to be negative. Its established range, 99 through 609, makes that branch nonselected as well.

Consequently, the ordinary path reaches F-c260788e6b26 and the final-record writes, provided earlier accesses succeed and dependencies return. These arithmetic conclusions do not prove that the supplied static representation completely or accurately recovers every executable path.

## Side effects

### Observable ordering

The contract requires the following ordering relationships:

- The initial selector read precedes the selected signed halfword read; both precede F-a00000000001.
- F-a00000000001, F-a00000000002, and F-a00000000003 occur in that order. The address-derived halfword write follows F-a00000000003 and precedes F-a00000000004.
- The payload byte write through the arithmetic-input-derived address occurs after F-a00000000004 and before F-a00000000005.
- The possible invalid-instruction termination is positioned after F-a00000000005 but before writes through its result. On the ordinary path, the payload halfword at offset 20 is written before the payload byte at offset 26.
- Those two writes precede the low-address global assignment. That assignment precedes capture of the separate base global.
- Base capture precedes F-a00000000006. The count-byte read follows that call and precedes the 32-bit constant store.
- The constant store precedes F-a00000000007, which precedes F-c260788e6b26.
- The final-record writes follow F-c260788e6b26 and precede ordinary return.

This ordering matters even where destinations appear unrelated: the evidence does not exclude aliases or memory-mapped effects.

### Aliasing, ownership, and failure

Input memory, globals, dependency-provided destinations, and the final record may overlap unless an external contract excludes that possibility. A dependency can modify memory that a later explicit read observes. Conversely, already captured selectors, payload values, and base-derived values are not refreshed merely because their originating memory changes.

Later writes may overwrite earlier writes. The contract specifies access order and field values at the time of each write, not persistence of every intermediate value until return.

No allocation, release, reference-count adjustment, or ownership transfer is established. A dependency-supplied address is used as storage without evidence that the current operation owns it or that ownership passes to its caller.

There is no visible rollback or cleanup path. If an access faults or a dependency does not return, earlier effects may already have occurred. The evidence supplies no portable exception model or recoverable error result for such failures.

## Dependencies

The F-a0000000000x identifiers are document-local opaque labels for unresolved call targets. They do not assert independently verified identities.

| Dependency | Caller-facing contract established here |
|---|---|
| F-a00000000001 | Receives the address input plus 74; 59; twice the arithmetic input plus 104; and the selected sign-extended halfword. Its result is consumed as two 32-bit portions. Target formation depends on incoming execution context. |
| F-a00000000002 | Receives the lower portion of the first result plus 80, followed by 50. Supplies a 32-bit value passed onward. |
| F-a00000000003 | Receives the preceding result and the address of an opaque static object. Its 32-bit result supplies both a store address and the bit pattern truncated for that store. |
| F-a00000000004 | Has no explicit arguments in the representation. No result is consumed. |
| F-a00000000005 | Has no explicit arguments. Its 32-bit result supplies the destination for the payload writes at offsets 20 and 26. |
| F-a00000000006 | Receives 29. Its upper result portion supplies the count-byte read address; its lower portion supplies later arithmetic. |
| F-a00000000007 | Receives twice the quantity obtained by subtracting 54 from the lower portion of F-a00000000006’s result. No result is consumed. |
| F-c260788e6b26 | Receives 115. Supplies both the final destination and the complete first record field. |
| F-943b4f706744 | Appears only in the nonselected negative-count branch, with an opaque global value and 67 as explicit arguments. |

“No explicit arguments” does not establish absence of hidden calling-convention inputs. “No result consumed” does not establish absence of effects.

The opaque static object passed to F-a00000000003 has no established size, contents, layout, mutability, or lifetime beyond its use as an address argument. None of the dependencies is established to preserve globals, validate memory, allocate storage, or release storage.

## Acceptance examples

These examples are conditional static criteria, not observed test results. They assume that relevant accesses succeed and dependencies return.

1. **Signed selected input.** With an initial selector of zero, the halfword read begins at address-input offset 74. If that halfword has bit pattern 0xFFFF, the fourth argument to F-a00000000001 is −1, not 65535.

2. **Maximum selection offset.** With an initial selector of 255, the selected halfword begins at offset 329 and occupies offsets 329–330. The selector is not doubled. Adding 91 still cannot select the invalid-instruction termination.

3. **Payload truncation.** If the first result’s upper portion is 0x00008081, the payload is 0x0102. The payload halfword write carries 0x0102, while both payload byte writes carry 0x02.

4. **Address-derived halfword.** If F-a00000000003 returns 0x12345678, the halfword write targets 0x1234567C and carries bit pattern 0x5678. No interpretation as a length or pointer field is required.

5. **Minimum count.** With a count byte of zero, the constant 51 is written as four bytes starting at captured-base minus 2. The adjusted count is 99; the ordinary final-record path remains selected.

6. **Maximum count.** With a count byte of 255, the constant store starts at captured-base plus 253. The adjusted count is 609 and is not negative.

7. **Captured-base stability.** If the captured base is 0x1000 and the count byte is 1, the final scalar fields are 101 at offset 8, 0x106C at offset 12, and 0x0FFF at offset 16. A later dependency’s modification of the originating global does not replace that captured base.

8. **Unusable destination.** If a dependency returns an address unsuitable for its required access, successful completion is not guaranteed. An error status, fallback record, cleanup operation, or rollback must not be inferred.

## Uncertainties

**documented_only — Recovery completeness.** The supplied representation reports invalid instruction data, removal of unreachable regions, overlapping global symbols, and an unresolved indirect transfer. These limitations prevent treating it as a fully verified executable specification.

**documented_only — Alternative branch.** The negative-count branch contains additional reads, writes, calls including F-943b4f706744, values attributed to unresolved call outputs, and an unresolved final indirect transfer. It is not selected by any established count byte. No supported input is invented to reach it, and its operations are not ordinary-path requirements. Its complete behavior remains unresolved.

**documented_only — ABI and execution context.** Exact calling conventions, hidden inputs, preservation rules, and the physical mechanism for split results are unknown.

**documented_only — Memory representation.** Endianness, alignment tolerance, memory-mapped semantics, the low-address global’s width, and address-space behavior at wrapping boundaries are unknown.

**documented_only — Storage contracts.** Destination capacity, object identity, lifetime, ownership, and dependency failure conventions are unknown.

**Inferred intent.** The final writes resemble population of a compact record containing a returned value, a bounded scalar, and two base-derived values. This resemblance establishes no domain schema or ownership policy.

## Evidence

The contract is based solely on the supplied static representation: explicit access widths, argument values, result consumption, memory-effect ordering, and branch arithmetic. Prior descriptive claims are not independent evidence.

The two reachability conclusions follow from unsigned-byte bounds:

- The initial selector plus 91 lies between 91 and 346.
- Twice the count byte plus 99 lies between 99 and 609.

No dependency implementation was inspected, and no runtime execution, smoke run, or tests were performed. The acceptance examples express conditional obligations derived from static evidence, not verified executions.
