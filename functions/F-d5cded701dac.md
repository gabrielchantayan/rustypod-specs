<!-- Trusted provenance: F-d5cded701dac; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d5cded701dac

## Purpose

F-d5cded701dac provides a character-valued lookup from ambient storage with a signed upper-bound guard. Candidate lookup locations are spaced 276 bytes apart. On normal completion, the selected character is converted to an integer and returned.

The guard distinguishes indices greater than 2 from all other indices. It does not establish a valid array range: negative indices pass the guard, and storage validity is not checked locally. Describing this operation as a safe three-element lookup would therefore overstate its guarantees.

An index greater than 2 invokes F-a817d8d58c10. The supplied static representation marks that dependency as nonreturning. Normal completion for such an index is consequently not established.

Retrieving a discriminator or status field from regularly spaced records is a plausible interpretation, but it is **inferred intent**, not an established storage schema. Neither the number of allocated records nor the meaning of the returned character is supplied.

## Interface

### Explicit inputs

The recovered interface has two arguments:

- **An opaque 32-bit argument.** The function body contains no explicit use of this argument. Its interpretation, if any, is outside the supplied evidence.
- **A signed integer index.** The recovered interface represents this as a 32-bit integer. It determines both whether the dependency is invoked and which storage location is selected.

The index comparison is signed. A negative index does not become a large positive index for purposes of the guard.

Changing only the opaque argument does not change any operation explicitly represented in this function. This is not a claim that the dependency cannot observe information associated with that argument through the calling environment.

### Ambient input

The lookup uses an ambient base-address value rather than an explicitly supplied pointer. The evidence does not establish:

- Who initializes or owns that value.
- The type or extent of its backing allocation.
- Whether the value can change concurrently.
- Whether the selected address lies within a live object.
- What alignment, address-space, or access restrictions apply.

A successful lookup requires the selected location to support the character read. This is an environmental precondition, not a condition enforced by the function.

### Result

Normal completion returns a signed integer, represented as 32 bits in the recovered interface. The result is obtained by converting the selected character directly to that integer type.

The character type’s signedness is unresolved. For an eight-bit character representation:

- Values from 0 through 127 have the same nonnegative result under signed and unsigned interpretations.
- An unsigned interpretation gives results from 128 through 255 for high-bit values.
- A conventional signed eight-bit interpretation gives negative results for those same high-bit patterns.

The numerical high-bit examples below are conditional on those representation assumptions. No independent target description establishes character signedness or the complete machine ABI.

There is no separate status result, sentinel, or structured error value. A negative return value, if character conversion produces one, is not established as an error indication.

## Behavior

### Index eligibility and guard behavior

Indices at or below 2 bypass F-a817d8d58c10. This includes zero, both positive indices below the threshold, and every representable negative index.

Indices greater than 2 invoke F-a817d8d58c10. No explicit argument values are supplied in the represented invocation. The actual calling convention and any implicit inputs are unknown.

The function has no local lower-bound rejection, clamping, normalization, or substitute lookup location. An implementation that rejects every negative index would add behavior not present in the supplied evidence.

### Lookup location and accessible layout

When execution reaches the lookup, the index selects a nominal displacement from the ambient base, with 276 bytes between consecutive selections.

For ordinary address arithmetic without overflow or wraparound:

- Index 0 selects the base location.
- Index 1 selects the location 276 bytes after the base.
- Index 2 selects the location 552 bytes after the base.
- Index −1 selects the location 276 bytes before the base.
- Index −2 selects the location 552 bytes before the base.

Only the character at the selected location contributes to the result. The function does not inspect the rest of a 276-byte interval, validate a record header, follow an embedded pointer, or search for a terminator.

The spacing alone does not establish 276-byte objects. It also does not establish that three entries exist, that their lifetimes overlap, or that the ambient base points to the beginning of an allocation. For example, a negative index could select accessible storage if the base points into a larger allocation; accessibility must not be inferred merely from the guard accepting that index.

### Arithmetic limits

The displacement calculation uses integer arithmetic with a multiplier of 276. No explicit widening conversion, overflow check, or checked-address operation is shown.

Under the recovered signed 32-bit interpretation, sufficiently negative indices have mathematical displacements outside the representable signed range. Such indices still bypass the upper-bound guard. Passing the guard therefore does not establish that displacement arithmetic has a defined result.

The supplied evidence does not resolve how to interpret overflow at the machine level. This contract does not require wrapping, trapping, saturation, or any particular address for an overflowing case.

The ambient base’s representation and the arithmetic used to combine it with the displacement are also unresolved. Even when the displacement itself fits, address addition and object-boundary behavior require additional evidence. The ordinary offsets listed above are conditional descriptions, not guarantees that every corresponding address is valid.

### Result selection

A completing lookup reads one character-sized value from the selected location and converts that value to the integer result.

There is no visible decoding, filtering, range check, or writeback of the character. Its meaning is immaterial to the locally represented behavior. A zero character is returned as zero rather than treated as a terminator.

If the selected location is unreadable, the function provides no local recovery branch. The evidence does not establish a fault type, exception, diagnostic, termination action, or return value for that situation.

### Dependency termination boundary

The nonreturning annotation on F-a817d8d58c10 supports the expectation that an index greater than 2 does not produce a normal return from this function. It does not identify why execution fails to return.

The represented body places the lookup after the conditional invocation, without a separate local result for that branch. If the dependency returned contrary to its annotation and execution continued as represented, the lookup would remain the subsequent operation using the index.

That continuation is conditional, not an established supported path. In particular, it must not assume that the dependency preserved the ambient base, the selected storage, or any other global state. A hypothetical later lookup would depend on the state available after the call.

## Side effects

On the direct lookup path, the explicit external data access is a character-sized read using the ambient base and selected displacement. The body also uses the ambient base-address value. Its storage representation and the number of machine accesses needed to obtain it are not established.

For an index greater than 2, the dependency invocation precedes any lookup represented after that invocation. No later lookup is established under the dependency’s annotated nonreturning behavior.

The selected address must be available before its character can be read. The evidence does not establish a machine-level ordering between independent displacement arithmetic and acquisition of the ambient base.

There are no explicit writes, allocations, releases, ownership transfers, or reference-count operations in the supplied body. A normal lookup does not visibly acquire ownership of the backing storage.

These local observations do not constrain the dependency’s effects. They also do not establish purity, atomicity, synchronization, thread safety, or immunity to concurrent mutation. No snapshot of the base or storage is guaranteed across the dependency invocation.

## Dependencies

### F-a817d8d58c10

**Invocation condition:** The signed index exceeds 2.

**Explicit inputs:** No argument expressions are represented. Register contents, global state, and other implicit inputs remain outside this contract.

**Return behavior:** Annotated as nonreturning in the supplied static evidence. The annotation is not runtime proof.

**Effects and ownership:** Unknown. No assumption is made that the dependency preserves globals, leaves storage allocated, avoids writes, or owns or releases any resource.

**Error semantics:** No particular error category, message, exit status, exception, or control-transfer mechanism is established.

### Ambient storage prerequisite

The lookup requires a usable base-address value and accessible storage at the selected location. This is an environmental dependency, not an additional identified function.

The available evidence establishes selection spacing and a character read. Allocation extent, initialization, lifetime, mutability, and synchronization requirements remain undocumented.

## Acceptance examples

These examples are conditional consequences of the static evidence. They are not reports of executed tests.

### Base-location lookup

Given index 0, ordinary nonoverflowing address arithmetic, and a readable character with value 42 at the ambient base, the function returns 42 without invoking F-a817d8d58c10.

The example does not require the following 275 bytes to be readable: they are not part of the represented character lookup.

### Threshold index

Given index 2 and a readable character with value 17 located 552 bytes after the base, the function returns 17 without invoking the dependency, provided address formation is valid.

Rejecting index 2 as out of range would contradict the established guard.

### Negative index

Given index −1, valid address formation, and a readable character with value 7 located 276 bytes before the base, the function returns 7 without invoking the dependency.

If that location is inaccessible, the same index still bypasses the guard, but successful completion is not guaranteed. Acceptance by the guard and validity of the memory access are separate conditions.

### Index above the threshold

Given index 3, the function invokes F-a817d8d58c10. Under the annotated nonreturning behavior, no normal integer result follows.

This example imposes no required diagnostic, exit status, exception, or subsequent memory access.

### High-bit character

Given index 1 and an eight-bit character with all bits set at the selected readable location, the dependency is not invoked.

Under an unsigned-character interpretation, the result is 255. Under a conventional two’s-complement signed eight-bit interpretation, the result is −1. The supplied evidence does not select one unconditional expectation.

### Unreadable storage

Given index 0 and an unreadable selected location, there is no represented local validation or recovery result. A particular fault or return value cannot be required from this evidence alone.

### Overflowing displacement

Given a sufficiently negative index whose mathematical displacement exceeds signed 32-bit limits, the guard does not invoke the dependency. The effective address and outcome are **documented_only** pending arithmetic and target evidence.

A wrapped address is not an acceptable unconditional expectation.

### Unexpected dependency return

If F-a817d8d58c10 unexpectedly returns, the represented continuation leads toward a lookup using the index. Any proposed expected result must additionally establish valid arithmetic, the post-call base value, and readable post-call storage.

The dependency’s invocation alone does not establish those conditions.

## Uncertainties

The following remain **documented_only**:

- Character signedness and complete target character representation.
- Independent confirmation of the recovered integer widths and calling convention.
- Machine behavior for displacement overflow and address overflow.
- The ambient base’s representation, initialization, lifetime, and permitted mutation.
- Allocation boundaries and accessibility of each selected location.
- Whether the spacing corresponds to records, slots, or another organization.
- Semantic meaning of the returned character.
- F-a817d8d58c10’s actual effects, resource ownership, and termination mechanism.
- Accuracy of the nonreturning annotation for every invocation.
- Fault behavior for an invalid read.
- Concurrency and synchronization requirements.

These unresolved details do not erase the established signed threshold, absence of a lower-bound check, 276-byte selection spacing, or direct character-to-integer result.

## Evidence

**Established static evidence:** The supplied body exposes two arguments, leaves the first explicitly unused, applies a signed comparison against 2 to the index, conditionally invokes F-a817d8d58c10, and derives a character lookup using 276-byte spacing from an ambient base. It returns the character converted to an integer. No local write, lower-bound check, storage validation, or recovery result is shown.

**Annotation evidence:** The dependency is marked nonreturning. This supports a static control-flow expectation, not a verified termination mechanism.

**Inferred intent:** A regularly spaced record-field lookup is plausible. No record schema, allocation count, or field meaning is established.

**Verification limit:** No runtime tests have been performed. No dependency implementation, independent target ABI, or backing-memory layout was supplied.
