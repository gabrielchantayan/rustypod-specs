<!-- Trusted provenance: F-d4720ffa0ebc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d4720ffa0ebc

## Purpose

F-d4720ffa0ebc performs bounded identifier-based dispatch through a descriptor obtained from its first argument. Before selecting a handler, it prepares temporary state, obtains an associated data area and a 32-bit auxiliary value through dependencies, and invokes a cleanup dependency.

Selection examines an ordered list of at most four 32-bit identifiers. Zero terminates that list. A matching nonzero identifier permits dispatch through a single callback field shared by the descriptor.

On successful dispatch, the function returns the callback’s 32-bit result unchanged. If selection fails or the callback field is null, it reports a diagnostic and returns zero.

These are established properties of the supplied static representation. The semantic meanings of the identifiers, temporary data, auxiliary value, callback result, and diagnostic codes are not established.

## Interface

The interface consists of four arguments and one 32-bit return value.

| Argument | Established representation | Use |
|---|---|---|
| Descriptor holder | Address of a 32-bit cell containing a descriptor address | Passed to a preparation dependency; dereferenced afterward to select the descriptor |
| Forwarded value | One 32-bit value | Passed unchanged to the selected callback |
| Output cell | Address of a writable 32-bit cell | Cleared at entry; its address is later passed to the callback |
| Request record | Address of a record containing 32-bit fields | Supplies the requested identifier and an additional callback argument |

The static representation uses 32-bit address-bearing cells. It does not establish a portable interface for wider pointers.

### Descriptor layout

Offsets are in bytes from the descriptor address obtained through the descriptor holder.

| Offset | Width | Established role |
|---|---|---|
| 0 | 32 bits | First callback argument |
| 36 | 32 bits | Callback address; zero means unavailable |
| 44 | 32 bits | First selectable identifier |
| 48 | 32 bits | Second selectable identifier |
| 52 | 32 bits | Third selectable identifier |
| 56 | 32 bits | Fourth selectable identifier |

Fields outside these offsets have no established direct role in this function. In particular, no behavior is established for the descriptor word at offset 40.

All selectable identifiers share the callback at offset 36. The list is not a table of separate identifier-and-handler pairs.

### Request-record layout

| Offset | Width | Established role |
|---|---|---|
| 0 | 32 bits | Requested identifier |
| 12 | 32 bits | Sixth callback argument |

The fields at offsets 4 and 8 have no established direct use here. Dependencies may nevertheless access caller-provided storage according to contracts not supplied.

### Temporary storage

The function provides three local storage regions:

- A 16-byte working region passed through preparation, extraction, and cleanup dependencies.
- A 36-byte data region supplied to the extraction dependency and later passed by address to the callback.
- A 4-byte auxiliary cell supplied to the extraction dependency and later read as a callback argument.

These capacities do not establish the logical length or format of any extracted data. In particular, the auxiliary value is not proven to be a length.

### Preconditions

For defined execution, every address actually accessed must be valid for the corresponding access and satisfy the target’s alignment requirements. This includes:

- A writable output cell at function entry.
- Storage accepted by each dependency.
- A readable descriptor-holder cell after the dependency sequence.
- Readable descriptor fields along the selected execution path.
- Readable request fields when those fields are used.
- A callable callback address with a compatible calling convention when dispatch occurs.
- Dependency-produced values suitable for their later use.

The function performs no visible null-pointer, bounds, callback-signature, or initialization validation.

## Behavior

### 1. Clear the output cell

The first established operation writes the 32-bit value zero to the output cell.

This precedes every dependency call and every direct descriptor-selection read. It is not merely a default applied on an error path.

Aliasing can therefore affect subsequent behavior. If the output cell overlaps the descriptor holder, request record, or storage observed by a dependency, the initial write changes that storage before it is consumed. No non-aliasing guarantee is established.

The entry write does not guarantee that the output cell remains zero. Dependencies may have access to overlapping or shared storage, and the callback receives the output-cell address explicitly.

### 2. Prepare and extract temporary data

The function invokes the following dependencies in order:

1. F-99497b78e41d receives the address of the 16-byte working region.
2. F-ef3ab54e3806 receives that same address and the descriptor-holder argument.
3. F-0b1e8b902739 receives the working-region address, the 36-byte data-region address, and the auxiliary-cell address.
4. F-990a2f44da95 receives the working-region address.

No dependency result is visibly checked before proceeding to the next step.

The names “preparation,” “extraction,” and “cleanup” describe apparent roles, not proven internal semantics. The established facts are the arguments, ordering, and subsequent uses of the storage.

Cleanup occurs before descriptor selection and before any callback. There is no visible second cleanup operation after dispatch or diagnostic reporting.

If a dependency does not return normally, subsequent stages are not established to execute.

### 3. Obtain the descriptor

After the four dependency calls return, the function reads the address stored in the descriptor holder and retains it for selection.

Consequently, the selected descriptor is obtained from the holder’s post-dependency contents. If a dependency changes the holder, the function uses the resulting address rather than a descriptor address captured at entry.

The function does not visibly obtain a new descriptor address between candidate positions. Descriptor contents themselves are not copied into an immutable snapshot.

### 4. Search the bounded identifier list

The function considers descriptor offsets 44, 48, 52, and 56 in that order.

At each position:

- A zero identifier terminates selection immediately.
- Otherwise, the identifier is compared with the requested identifier at request-record offset 0.
- Equality selects the descriptor’s callback path.
- A nonmatch advances to the next position, unless four candidates have already been considered.

The zero check precedes the equality check. Thus a zero request identifier cannot match a zero list entry: zero is a terminator, not a selectable identifier.

A zero entry also makes all later entries unreachable. A matching value after such an entry does not count.

The scan performs at most four candidate-position checks. If all four entries are nonzero and none matches, selection fails without requiring or inspecting a fifth entry. Candidate offsets remain bounded through byte offset 56; there is no caller-controlled scan length or index arithmetic.

Identifier equality is equality of 32-bit values. Signed ordering has no role in selection.

### 5. Dispatch on a match

After finding a matching identifier, the function checks the callback field at descriptor offset 36.

If it is nonzero, the callback is invoked with six arguments, in these argument positions:

| Position | Value |
|---|---|
| 1 | The 32-bit descriptor field at offset 0 |
| 2 | Address of the local 36-byte data region |
| 3 | The 32-bit auxiliary value supplied through extraction |
| 4 | The second argument originally supplied to F-d4720ffa0ebc |
| 5 | The original output-cell address |
| 6 | The 32-bit request field at offset 12 |

Argument positions are established; a left-to-right order for evaluating their individual reads is not.

The callback’s 32-bit return value becomes the function’s return value unchanged. No diagnostic is emitted by this function after a normally returning callback, regardless of the callback’s result. A callback return of zero is therefore not converted into a selection error.

Only one callback invocation is attempted. Duplicate identifiers do not produce repeated dispatch.

The local data region remains within this invocation’s lifetime during the synchronous callback. Its address is not established as valid after F-d4720ffa0ebc returns.

### 6. Report a selection failure

There are two locally distinguished failure cases:

| Condition | Diagnostic third argument |
|---|---|
| A requested identifier matches, but the callback field is zero | 104, hexadecimal 0x68 |
| No identifier matches before termination or exhaustion | 110, hexadecimal 0x6E |

For either case, F-4fc89ec41a6d receives three arguments:

1. 6
2. 107, hexadecimal 0x6B
3. The condition-specific value above

After that dependency returns normally, F-d4720ffa0ebc returns the 32-bit value zero. It does not visibly use the diagnostic dependency’s return value.

The meanings of these diagnostic numbers are unknown. They are not established as an operating-system error, severity level, or public status enumeration.

## Side effects

The only unconditional direct write to caller-provided storage is the initial clearing of the output cell.

Additional effects may occur through dependencies and the callback. The supplied representation does not establish that they preserve globals, avoid aliasing, refrain from modifying the descriptor holder, or leave the output cell unchanged.

The function directly reads:

- The descriptor-holder cell after the dependency sequence.
- Candidate identifier fields as required by the bounded search.
- The request identifier when comparing a nonzero candidate.
- The callback field after a match.
- The descriptor’s first field, the auxiliary cell, and request offset 12 when forming a callback invocation.

A terminated scan does not require later candidate reads. An empty list does not require this function’s direct read of the request identifier. A failure path does not require its direct read of request offset 12.

These conditional-read properties do not constrain what the earlier dependencies may read.

No direct allocation, ownership transfer, or release of caller-provided memory is visible. Invoking F-990a2f44da95 does not, by itself, prove which resources it releases or whether extracted data contains references affected by that call.

## Dependencies

| Opaque dependency ID | Established interaction | Unknown contract |
|---|---|---|
| F-99497b78e41d | Called first with the 16-byte working region | Initialization requirements, writes, resource effects, and failure behavior |
| F-ef3ab54e3806 | Called second with the working region and descriptor holder | Reads, mutations, ownership effects, and result meaning |
| F-0b1e8b902739 | Called third with working storage and two output-region addresses | Data format, initialization guarantees, auxiliary-value meaning, and bounds discipline |
| F-990a2f44da95 | Called fourth with the working region | Cleanup semantics, invalidation effects, and shared-state changes |
| F-4fc89ec41a6d | Called on local selection failures with the documented numeric arguments | Reporting destination, side effects, and failure behavior |

The callback is an additional runtime-selected dependency whose opaque identity is unavailable. Its argument positions and result propagation are established; its ownership rules, output semantics, and global effects are not.

## Acceptance examples

These examples are conditional acceptance criteria, not executed tests. Unless stated otherwise, assume valid storage, normally returning dependencies, initialized extraction outputs, and no intervening mutation of relevant fields.

### First-entry match

Given a requested identifier of 17, a first candidate of 17, and a valid nonzero callback:

- Preparation and cleanup complete before dispatch.
- The callback receives the six documented arguments.
- Its 32-bit result is returned unchanged.
- Later candidates are irrelevant.
- No local selection diagnostic is reported.

### Match after a nonmatch

Given candidates 8, 17, 0, 0 and request 17, the second position selects the callback. The following zero does not prevent dispatch because selection has already succeeded.

### Terminator hides a later match

Given candidates 8, 0, 17, 0 and request 17, selection stops at the second position. The function reports diagnostic arguments 6, 107, and 110, then returns zero.

### Zero is not selectable

Given a zero request and a zero first candidate, the function reports the no-match diagnostic. It does not invoke the callback even if the callback field is nonzero.

### Four nonmatching candidates

Given four nonzero candidates different from the request, the function reports the no-match diagnostic after the fourth candidate. No fifth candidate is needed.

### Match with no callback

Given a matching candidate and a zero callback field, the function reports diagnostic arguments 6, 107, and 104, then returns zero. It does not continue searching for another match.

### Callback returns zero

Given a matching candidate and a valid callback that returns zero, the function returns zero without invoking the local diagnostic dependency. Return value alone cannot distinguish this case from a locally reported selection failure.

### Callback changes the output

Given a valid dispatch in which the callback writes a new 32-bit value through the output-cell address, the function does not visibly restore the entry-time zero afterward.

### Descriptor holder changes during preparation

If a dependency replaces the address in the descriptor holder and all resulting accesses are valid, selection uses the replacement descriptor. This follows from reading the holder after the dependency sequence.

## Uncertainties

The following remain **documented_only**:

- Semantic types and meanings of descriptor fields, request fields, identifiers, and callback results.
- Whether the 36-byte region holds bytes, structured data, addresses, or another representation.
- Whether the auxiliary value denotes a size, status, count, or unrelated quantity.
- Whether extraction initializes every byte or guarantees a usable auxiliary value.
- Whether cleanup invalidates references contained in extracted data.
- Dependency ownership, allocation, release, global-state, and nonlocal-exit behavior.
- Callback calling convention beyond the visible six-argument use.
- Reentrancy, synchronization, and behavior under concurrent mutation.
- Target-specific behavior after invalid pointers, incompatible callbacks, or uninitialized dependency outputs.

These unknowns are not resolved by assuming conventional helper behavior.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes the initial 32-bit output write, dependency-call order, temporary-region capacities, descriptor and request offsets, four-position sentinel-terminated search, callback argument positions, diagnostic constants, and return-value propagation.

Preparation, extraction, cleanup, and identifier-based dispatch are inferred role descriptions. They do not establish undocumented callee semantics.

No runtime tests have been performed. The acceptance examples specify conditional expected behavior rather than observed execution results.
