<!-- Trusted provenance: F-e6dd6811e5cf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e6dd6811e5cf

## Purpose

F-e6dd6811e5cf performs three ordered dependency calls and six explicit memory writes. It obtains an initial result using the argument 304, passes that result and its first input to a second dependency, writes three 32-bit fields relative to the second dependency’s result, and invokes a third dependency with an interior address. It then writes three fields relative to the third dependency’s returned address.

The established behavior is the sequence of calls, global reads, address calculations, and writes. Object construction, resource allocation, and embedded-component initialization are plausible interpretations, not established requirements.

The function has no explicit return value. Its observable results are dependency interactions and memory changes. Successful return does not communicate an address or status.

## Interface

### Inputs

The function accepts four positional inputs:

| Input | Established width | Use |
|---|---:|---|
| First input | 32 bits | Passed unchanged as the second argument to F-e64bbca26756. |
| Second input | 32 bits | Passed unchanged as the second argument to F-9563b8df22d3. |
| Third input | 32 bits | Written at byte offset 48 from the result of F-9563b8df22d3. |
| Fourth input | 16 bits | Written to an address determined by that result and a sampled global displacement. |

The semantic types and signedness of these inputs are not established. The first three may represent integers, addresses, handles, or other bit patterns. The function does not locally validate them.

The fourth input already has a 16-bit interface width. Conversion from any wider caller-side value is outside the demonstrated behavior.

### Address and sample terminology

This contract uses the following role names:

- **Construction address:** the result of F-e64bbca26756, interpreted as an address for subsequent accesses.
- **Component address:** the result of F-9563b8df22d3, interpreted as an address for subsequent accesses.
- **First global:** the 32-bit global read twice after F-e64bbca26756 returns.
- **Retained sample:** the earlier of those two reads. It supplies both additions used for construction offsets 208 and 240.
- **Direct-store sample:** the later read of the first global. It supplies the construction-offset-0 write.
- **Second global:** the 32-bit global read after F-9563b8df22d3 returns.
- **Displacement sample:** the value obtained by that second-global read.

The retained sample and direct-store sample are distinct observations of the same global. Equality requires a stability assumption; it is not guaranteed by the supplied representation.

These names do not imply ownership, allocation identity, object type, or disjoint storage.

### Preconditions for normal completion

Normal completion requires each dependency call to return normally and every executed memory access to be valid.

In particular:

- The construction address must permit 32-bit writes at offsets 0, 208, and 240.
- Construction address plus 244 must satisfy F-9563b8df22d3’s input requirements.
- The component address must permit the fixed-offset writes at offsets 48 and 56.
- Component address plus the displacement sample minus 244 must permit a 16-bit write.
- Relevant alignment and address-representation requirements must hold.

The function does not check these conditions. It contains no local failure branch, null-result check, bounds check, or recovery path.

## Behavior

### Ordered observable operations

On a normally completing path, the function performs the following operations in order:

1. Calls F-8157a180d901 with the numeric argument 304, hexadecimal 0x130.
2. Calls F-e64bbca26756 with the first dependency’s 32-bit result and the first input.
3. Reads the first global, producing the retained sample.
4. Reads the first global again, producing the direct-store sample.
5. Writes the direct-store sample as 32 bits at construction offset 0.
6. Writes the retained sample plus 392, hexadecimal 0x188, as 32 bits at construction offset 208, hexadecimal 0xD0.
7. Writes the retained sample plus 408, hexadecimal 0x198, as 32 bits at construction offset 240, hexadecimal 0xF0.
8. Calls F-9563b8df22d3 with construction address plus 244 bytes, the second input, and the numeric value 1.
9. Reads the second global, producing the displacement sample.
10. Writes the third input as 32 bits at component offset 48, hexadecimal 0x30.
11. Writes the fourth input as 16 bits at component address plus the displacement sample minus 244.
12. Writes an 8-bit zero at component offset 56, hexadecimal 0x38.
13. Returns without an explicit result.

This ordering distinguishes the three explicit global reads from the six explicit writes. In particular, both first-global reads precede the construction-offset-0 write.

### Construction-region values

The construction-region writes have two distinct sources:

| Destination | Width | Value source |
|---|---:|---|
| Construction address plus 0 | 32 bits | Direct-store sample |
| Construction address plus 208 | 32 bits | Retained sample plus 392 |
| Construction address plus 240 | 32 bits | Retained sample plus 408 |

There is no intervening dependency call between the two first-global reads. Nevertheless, the representation explicitly contains two reads, and the contract does not replace them with a single shared observation.

The retained sample remains the source for both additions even if the first global changes before either derived field is written. Similarly, the write at construction offset 0 uses the direct-store sample rather than rereading the global after that write begins.

The third dependency receives construction address plus 244 bytes, four bytes beyond the start of the field at offset 240. The source representation’s four-byte indexing accounts for the offsets 208, 240, and 244; these are byte offsets, not element counts.

The fixed construction writes fit within a nominal 304-byte extent. This does not prove that F-8157a180d901 allocates 304 bytes, that the construction address identifies its result, or that the third dependency’s accesses fit within that extent.

### Component-region values

The final writes use the address returned by F-9563b8df22d3. They do not necessarily use the interior address passed to it.

| Order | Destination | Width | Value |
|---:|---|---:|---|
| 1 | Component address plus 48 | 32 bits | Third input |
| 2 | Component address plus displacement sample minus 244 | 16 bits | Fourth input |
| 3 | Component address plus 56 | 8 bits | Zero |

The displacement sample is obtained before the third-input write. If that write aliases the second global’s storage, it does not change the already sampled displacement used for the halfword destination.

The displacement is not constrained to be at least 244. Subject to address arithmetic and validity, the halfword destination may precede, equal, or follow the component address.

### Arithmetic and widths

The representation uses 32-bit values for the working address results, both globals, and the construction-field arithmetic. The explicit stores comprise four 32-bit writes, one 16-bit write, and one 8-bit write.

The arithmetic relevant to observable destinations or values is:

- Retained sample plus 392.
- Retained sample plus 408.
- Construction address plus 244.
- Component address plus 48.
- Component address plus displacement sample minus 244.
- Component address plus 56.

There is no explicit overflow detection, saturation, range restriction, or bounds enforcement.

For values and addresses whose calculations remain representable, the stated additions and offsets directly determine the writes. The representation alone does not establish a complete language-level overflow model or processor-level address-wrap behavior. In a 32-bit wrapping-machine interpretation, arithmetic retains the low 32 bits; that interpretation does not make a wrapped destination valid. Overflow-dependent outcomes remain documented_only unless the execution model is supplied.

### Aliasing and retained contents

The evidence supplies no non-aliasing guarantees among construction storage, component storage, the globals, or memory used by dependencies.

Consequently, this contract specifies write events and their order, not six independently preserved fields at return.

Examples of ordering consequences include:

- If the halfword destination overlaps the third-input field, the later halfword write replaces the overlapping bytes.
- If the halfword destination includes component offset 56, the final byte write replaces that byte with zero.
- If a construction write aliases a global, later reads may observe the resulting storage contents.
- F-9563b8df22d3 may alter construction fields before the final writes occur. The function does not restore those fields.

Byte order is not established. An overlapping byte write therefore does not justify a particular final numerical interpretation of a larger field.

## Side effects

The locally established side effects on a normally completing path are:

- Three ordered dependency calls.
- Three explicit 32-bit global reads: two reads of the first global before any construction-field write, and one read of the second global after F-9563b8df22d3 returns.
- Six ordered memory writes: four 32-bit, one 16-bit, and one 8-bit.

There is no explicit local deallocation, rollback, logging, validation, or error reporting.

The function does not return the construction address or component address. Whether either becomes reachable by callers through dependency side effects, registration, globals, or memory relationships is unknown.

Ownership is not established:

- F-8157a180d901 is not proven to return caller-owned or newly allocated storage.
- F-e64bbca26756 is not proven to preserve, consume, replace, or release its first argument.
- F-9563b8df22d3 is not proven to return its first argument or retain any particular relationship to it.
- No dependency is assumed to preserve globals or unrelated memory.

If a dependency fails to return normally or a memory access faults, preceding effects may already have occurred. No local transactional guarantee is present. The handling of such failures belongs to the unspecified dependency or execution environment.

## Dependencies

| Opaque function ID | Established interaction | Unknown properties |
|---|---|---|
| F-8157a180d901 | Called first with 304; its result becomes the first argument to F-e64bbca26756. | Resource semantics, failure convention, accessible extent, initialization, ownership, and other side effects. |
| F-e64bbca26756 | Receives the preceding result and the first input; returns the construction address. | Result identity, memory extent, ownership changes, global changes, and failure behavior. |
| F-9563b8df22d3 | Called after the three construction writes with construction address plus 244, the second input, and 1; returns the component address. | Accessed extent, result identity, ownership, global changes, and failure behavior. |

The global-read timing is part of the contract:

- Both reads of the first global occur after F-e64bbca26756 returns. Neither is a pre-call snapshot.
- The earlier read supplies both derived construction values; the later read supplies the construction-offset-0 value.
- The second-global read occurs after F-9563b8df22d3 returns and before the third-input write.

Dependency-induced changes can therefore affect subsequent reads. Nothing establishes that either dependency preserves either global.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence. No runtime tests have been performed. Each normally completing scenario assumes valid accesses, normally returning dependencies, and representable arithmetic unless stated otherwise.

### Stable first global and distinct destinations

Assume the first global remains 4096 across both reads, all six destinations are non-overlapping, and no intervening dependency changes the relevant construction fields.

The required construction writes are:

- Offset 0: 4096.
- Offset 208: 4488.
- Offset 240: 4504.

If the displacement sample is 244, the final writes place the third input at component offset 48, the fourth input at component offset 0, and zero at component offset 56.

The common source value for the construction fields in this example follows from the explicit stability assumption.

### Different observations of the first global

Assume the execution environment yields 4096 for the earlier first-global read and 8192 for the later read.

The required construction writes are:

- Offset 0: 8192.
- Offset 208: 4488.
- Offset 240: 4504.

Using 8192 for either addition, or using 4096 for offset 0, would not match the demonstrated value flow. This scenario specifies the consequence of differing observations without asserting how such observations arise or that concurrent modification is supported.

### Changes made by dependencies

Assume F-e64bbca26756 changes the first global before returning. Both first-global reads must occur after that return. If the global then remains stable, both observe that post-call value; without stability, equality is not required.

Assume F-9563b8df22d3 changes the second global before returning. The halfword destination uses the subsequently sampled value, not any earlier value.

### Halfword overlapping the final byte

Assume the displacement sample is 300. The halfword destination is component offset 56 because 300 minus 244 equals 56.

The fourth input is written there as 16 bits. The final operation then sets the byte at that same address to zero. The other byte of the halfword is not explicitly changed by the final write. The resulting halfword’s numerical value depends on byte order.

### Halfword destination preceding the component address

Assume the displacement sample is zero and ordinary non-wrapping address arithmetic applies. The halfword destination is 244 bytes before the component address.

The function neither rejects nor adjusts that destination. Normal completion requires that preceding location to be valid for the 16-bit write.

### Returned component address differs from the supplied interior address

Assume F-9563b8df22d3 returns a valid address different from construction address plus 244.

All three final writes must be based on the returned component address. Substituting the dependency’s first argument would change the established behavior.

### Third-input write aliases the second global

Assume the 32-bit destination at component offset 48 is the storage of the second global.

The displacement is read first. The third-input write may then replace that global’s contents, but the halfword destination still uses the earlier displacement sample. No additional global read is specified.

## Uncertainties

The following remain documented_only:

- Application-level purpose, object type, and field meanings.
- Semantic input types and caller obligations beyond the demonstrated accesses.
- Global meanings, storage properties, volatility, and synchronization.
- Full ABI, byte order, alignment constraints, and fault handling.
- Exact overflow and address-wrap semantics.
- Dependency implementations, ownership rules, failure conventions, and memory effects.
- Whether the argument 304 denotes a byte allocation size.
- Whether the construction address identifies the first dependency’s result.
- Whether the component address equals the supplied interior address.
- External reachability of either returned address.
- Cross-thread visibility, atomicity, and thread safety.

The specified sequence is the local order represented by the supplied function. It is not a guarantee of synchronized observations by other threads. Explicitly preserving two first-global reads does not establish a supported concurrent-update protocol.

## Evidence

This contract is based on the supplied static function representation and opaque dependency mapping.

Established static evidence includes four input widths, a void return, three dependency calls, three explicit global reads, six explicit writes, and the stated constants and offsets.

The first global’s two reads have distinct uses: the earlier sample supplies additions of 392 and 408; the later sample supplies the construction-offset-0 write. Both reads precede that write. The second global is sampled once after F-9563b8df22d3 returns and before the final three writes.

Allocation, construction, and embedded-component initialization remain inferred intent rather than established semantics. No dependency implementation, caller behavior, execution trace, or runtime test result was supplied. No runtime tests have been performed.
