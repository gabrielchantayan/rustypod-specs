<!-- Trusted provenance: F-aeebbea1ac53; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aeebbea1ac53

## Purpose

F-aeebbea1ac53 conditionally copies a 32-bit word from a selected record view into a caller-provided output location.

The gate is a byte reached through a pointer-valued global reference. The function first evaluates that reference to locate the byte, then reads the pointed-to byte. If the byte is zero, it returns without accessing the output or the record-storage base.

When enabled, the function clears the output, selects a record view, and checks a signed index against two signed limits. If both checks pass, it copies the indexed word to the output.

For stable state, valid accesses, and well-defined address calculations, success requires:

- A nonzero pointed-to enable byte.
- A nonnegative index.
- An index strictly below the signed limit at selected-record offset 68.
- An index strictly below the signed limit at selected-record offset 64.

Both limits are exclusive upper constraints. Neither is an inclusive lower bound.

A gated, bounded lookup is the inferred intent. The supplied body does not establish allocation boundaries, a payload-only region, or application-level meanings for the copied words and status values.

## Interface

### Arguments and return value

| Input | Established interpretation |
|---|---|
| Output location | Address receiving an initial 32-bit zero when processing is enabled, followed by a 32-bit result on success. |
| Index | Signed 32-bit integer used for validation and indexed selection. |
| Selector | Signed 32-bit integer. Exactly 1 selects a displaced record view; every other value selects the undisplaced view. |

The signed input range is −2,147,483,648 through 2,147,483,647.

The return value is 32 bits:

| Value | Normal-return condition |
|---|---|
| 0 | Both index checks passed, and the indexed word was written to the output. |
| 1 | The pointed-to enable byte was zero; the output was not accessed. |
| 6 | Processing was enabled, but index validation rejected the request after the output was cleared. |

These are the only normal return values visible in the supplied body. No visible mechanism converts invalid accesses or address-calculation failures into a status.

The copied word is an uninterpreted 32-bit value. Its signedness and any resource significance are not established.

### Gate reference

The enable gate has two source-level dependencies:

1. The pointer-valued global reference used to locate the enable byte.
2. The byte at the resulting address.

Evaluation of the pointer-valued reference precedes the pointed-byte read. This dependency must not be replaced with an assumed immutable enable-byte address.

Declarations are unavailable. Consequently, the representation does not establish the gate pointer’s width, how its value is stored or resolved, or whether evaluating it entails a distinct physical memory transaction. Any storage actually accessed to resolve that reference must be valid and readable, as must the resulting byte location.

### Record selection and layout

The record-storage base is a separate shared value. Its visible source-level reads are:

| Selector | Selection behavior |
|---|---|
| Any value other than 1 | Read the shared base once and use that value directly. |
| Exactly 1 | Read the shared base, then read it again. Select the second observed value plus 72 bytes. |

Both base reads on the selector-1 path occur after the initial output clear and before index validation. The first value does not determine the selected address on that path.

The selected record has these visible accesses:

| Byte offset | Use |
|---|---|
| 4 × index | Candidate 32-bit result word |
| 64 | Signed 32-bit limit, checked second |
| 68 | Signed 32-bit limit, checked first |

The 72-byte displacement selects another view. It does not prove a 72-byte allocation, a complete record size, or an access boundary. No visible check restricts indexed accesses to offsets below 64 or 72.

### Preconditions for normal completion

Every access actually reached must be valid:

- Any storage needed to resolve the pointer-valued gate reference must be readable.
- The resolved enable-byte location must be readable.
- On enabled paths, the output must permit a 32-bit write.
- On enabled paths, the shared record-storage base must be readable once, or twice for selector 1.
- For a nonnegative index, the selected limit at offset 68 must be readable.
- If that limit permits continuation, the selected limit at offset 64 must be readable.
- On success, the indexed word must be readable and the final output write must be valid.

A negative index still reaches record selection. It does not cause either record limit to be read.

No explicit null, alignment, allocation-length, or aliasing checks are visible. Normal completion also requires address calculations and address conversions to have valid behavior in the execution environment.

## Behavior

### Gate evaluation

The function evaluates the pointer-valued global gate reference and then reads the byte it identifies.

If the byte is zero, the function returns 1 immediately. It does not access the output, read the record-storage base, read a record limit, or read an indexed word.

Thus, the disabled path still depends on valid gate-reference resolution and a valid pointed-byte read. It is not a path without data dependencies.

Any nonzero byte enables processing. Its particular nonzero value has no further visible significance.

The body contains one gate-reference evaluation and one pointed-byte test. It does not explicitly reevaluate either after processing begins. These are source-level observations, not guarantees about physical transaction counts.

### Output clearing and record selection

On an enabled invocation, the function first writes a 32-bit zero to the output.

It then reads the shared record-storage base. For selectors other than 1, that observed value determines the selected record address.

For selector 1, it reads the shared base again and selects the address 72 bytes beyond the second observed value. If the two reads observe A and B respectively, selection uses B plus 72, not A plus 72.

Record selection completes before negative-index rejection. Even an index that will certainly be rejected does not avoid these base reads.

The output clear precedes every record-base read. If the output overlaps the base’s storage, subsequent selection may therefore use a value changed by that clear.

### Index validation

A negative index returns 6 without reading either limit or the indexed word. The output has already received its initial zero write.

For a nonnegative index, the function reads the signed 32-bit limit at offset 68:

- An index equal to the limit rejects.
- An index greater than the limit in signed order rejects.
- Only an index strictly below the limit continues.

Rejection here returns 6 without reading the limit at offset 64.

If the first comparison passes, the function reads the signed 32-bit limit at offset 64. Equality or a greater index again rejects with status 6. Only a strictly smaller index reaches the lookup.

For stable, nonaliased state, the accepted indices are therefore the nonnegative signed 32-bit values strictly below both limits. A nonpositive limit prevents success whenever its comparison is reached.

The comparisons constrain the index numerically. They do not independently establish that the indexed address lies within a valid allocation.

### Successful lookup

After both checks pass, the function reads one 32-bit word at the selected record address plus four times the index. It then writes those observed bits to the output and returns 0.

The indexed read precedes the final output write. A zero indexed word is a valid successful result; output contents alone cannot distinguish that success from enabled rejection.

There is no visible interpretation of the word, allocation, release, reference-count operation, or further lookup.

### Widths and arithmetic

The supplied representation establishes:

- An 8-bit enable-byte test.
- Signed 32-bit index, selector, and limit values.
- A 32-bit output transfer.
- A 32-bit return value.
- Record-address calculations carried through a 32-bit integer representation.

It does not establish the gate pointer’s width merely by establishing the widths above.

Validation uses signed ordering. The subtraction-related conditions and signed-overflow indicators jointly express the comparisons. Treating the sign of a potentially overflowing subtraction as the entire comparison would give incorrect results for extreme operands.

Selector 1 adds 72 bytes to the second observed record base. Indexed selection multiplies the index by four bytes and adds that displacement to the selected address.

Passing both comparisons does not establish representability or address validity. For example, limits of 2,147,483,647 permit index 2,147,483,646 at the comparison level, but its mathematical four-byte displacement exceeds 32 bits.

The full overflow, conversion, and address interpretation rules are not supplied. Such cases are documented_only; this contract does not invent wraparound, rejection, or fault behavior.

## Side effects

### Explicit writes

The only explicit writes target the output:

| Path | Writes |
|---|---|
| Disabled | None |
| Enabled, rejected | One 32-bit zero write |
| Enabled, successful | One 32-bit zero write, then one 32-bit result write |

The old output value is not explicitly read. Aliasing can nevertheless cause a later input access to read bytes from the same storage.

### Observable source-level access order

For a successful invocation with a selector other than 1:

1. Evaluate the pointer-valued global gate reference, including any access required to resolve its value.
2. Read the byte at the resolved address.
3. Write zero to the output.
4. Read the shared record-storage base.
5. Read the selected limit at offset 68.
6. Read the selected limit at offset 64.
7. Read the indexed word.
8. Write that word to the output.

For selector 1, the sequence is the same except that the shared record-storage base is read twice before either limit. The second observed base, plus 72 bytes, determines the selected view.

A disabled invocation stops after the pointed-byte test. A negative index stops after record selection and rejection. Failure at the first limit prevents the second-limit read. Failure at the second limit prevents the indexed read.

These sequences describe the supplied representation. They do not establish separate hardware transactions for every evaluation, memory barriers, atomicity, or device-memory semantics.

### Aliasing and changing state

No disjointness check is visible.

The initial output clear can affect later base selection, limit validation, or the indexed result when the output overlaps the corresponding storage.

If the output overlaps the enable-byte storage or storage used to resolve its pointer, the clear occurs after gate evaluation. The visible body does not reevaluate the gate, so that change does not by itself reverse the completed enable decision.

For selector 1, the two base reads need not observe identical values if intervening changes are permitted by the execution environment. After selection, there is no further shared-base read.

No lock, retry, atomic operation, or snapshot mechanism is visible. The gate reference, pointed byte, record base, limits, and result word are not established to form one coherent snapshot. Partial overlaps, alignment-sensitive accesses, and concurrent mutation remain subject to unspecified environmental rules.

### Ownership and lifetime

No allocation, deallocation, ownership transfer, or lifetime extension is visible. Copying a 32-bit word does not establish ownership of any resource that word might represent.

The body does not establish who owns the storage used to resolve the gate pointer, the pointed byte, the records, or the output. It provides no visible lifetime guarantee under shared-state changes.

## Dependencies

No calls to other functions appear in the supplied body. There are no callable dependencies requiring additional opaque function IDs.

The external data dependencies are:

- The pointer-valued global reference locating the enable byte, including any storage used to resolve it.
- The byte at the resolved gate address.
- The shared record-storage base.
- The selected record storage.
- The caller’s output storage.

The first two are distinct dependencies: locating the byte precedes reading it. The gate address must not be treated as an established constant.

Declarations, initialization rules, allocation extents, lifetimes, and synchronization contracts are not supplied. The selector-1 path contains two ordered source-level record-base reads; the disabled path contains neither.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Unless stated otherwise, assume stable state, valid suitably aligned accesses, well-defined arithmetic, and no output overlap with referenced state.

### Disabled gate

Given a resolvable gate reference pointing to a zero byte and an output containing a nonzero word:

- Resolve the gate reference and read its pointed byte.
- Return 1.
- Preserve the output.
- Perform no record-base, limit, or indexed-word read.

The index and selector do not change this result. An invalid gate reference or unreadable target is outside this normal-return example; status 1 is not established for those cases.

### Negative index with displaced selection

Given a valid nonzero gate, index −1, and selector 1:

- Clear the output after the gate test.
- Read the record base twice.
- Form the displaced selection from the second observed value.
- Return 6 without reading either limit or an indexed word.

For a selector other than 1, the same rejection requires only one record-base read.

### Zero index

Given index 0, limit 5 at offset 68, and limit 3 at offset 64:

- Read the limits in that order.
- Copy the selected record’s first word to the output.
- Return 0.

Changing the first limit to zero rejects before reading the second. Changing only the second limit to zero rejects after both reads. Either rejection leaves the output zero under the stated nonaliasing assumptions.

### Equality at either limit

Given index 4 and first limit 4, return 6 without reading the second limit.

Given index 3, first limit 8, and second limit 3, read both limits and return 6 without reading the indexed word.

Both cases include the initial output clear.

### Both limits constrain from above

Given first limit 8 and second limit 3:

- Indices 0, 1, and 2 succeed.
- Indices 3 through 7 fail the second comparison.
- Index 8 fails the first comparison.
- Negative indices fail before either limit read.

This excludes interpreting the second limit as a lower bound.

### Second base observation controls selector 1

Given selector 1, let the first base read observe A and the second observe B. If the relevant accesses are valid, validation and lookup use B plus 72 bytes.

For selector 2, there is one base read and no displacement. Selectors 0 and −1 follow the same selection rule as selector 2.

This criterion describes source-level evaluation, not a promise about concurrency or physical memory transactions.

### Indexed access reaches metadata

Given index 16, first limit 20, and second limit 19:

- Both comparisons pass.
- The indexed address is offset 64.
- With stable state, the second-limit word is copied to the output.
- Return 0.

The checks do not enforce a separate payload region.

### Output clear changes validation

Given an output exactly overlapping the selected limit at offset 68, without overlap affecting gate evaluation or base selection, and without intervening mutation:

- The initial write clears that limit.
- Every nonnegative index fails against the resulting zero limit.
- Return 6 without reading the indexed word.

### Output clear changes the completed gate

Given a valid nonzero gate byte lying within the writable four-byte output region, with no other affected dependency:

- Read the nonzero byte before clearing the output.
- Continue processing after the clear changes that byte to zero.
- Do not perform a second gate test.

The remaining result depends on record selection and validation, not on the gate byte’s new value.

## Uncertainties

The following remain documented_only:

- The declaration, width, representation, and resolution mechanism of the pointer-valued gate reference.
- Whether resolving that reference requires a separate physical memory read.
- Gate-pointer initialization, mutability, ownership, and lifetime guarantees.
- Application-level meanings of statuses 1 and 6.
- Whether the limits represent counts, capacities, or other metadata.
- Allocation extents and relationships between the record views.
- Caller restrictions on aliasing.
- Whether copied words represent values, addresses, handles, or resources.
- The complete pointer model, byte order, alignment requirements, and fault behavior.
- Overflow and conversion behavior in address calculations.
- Shared-state synchronization and snapshot guarantees.
- External invariants preventing indexed access into metadata or neighboring storage.

The bounded-lookup interpretation is inferred intent. The gate-reference dependency, pointed-byte test, output-write order, selector rule, conditional second base read, signed comparisons, and normal return values are established by the supplied static representation.

## Evidence

The supplied body supports evaluation of a pointer-valued global gate reference before dereferencing its byte target. It also supports the initial 32-bit output clear, initial record-base read, conditional second base read for selector 1, 72-byte displacement from that second value, ordered signed limit checks at offsets 68 and 64, four-byte indexed selection, final output write, and return values 0, 1, and 6.

No surrounding declarations, callers, allocation contracts, or synchronization rules were supplied. Source-level evaluations do not establish hardware transaction counts or an immutable gate address.

No runtime tests have been performed. All acceptance examples are static behavioral conditions, not observed executions.
