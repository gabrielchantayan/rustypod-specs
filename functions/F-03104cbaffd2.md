<!-- Trusted provenance: F-03104cbaffd2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-03104cbaffd2

## Purpose

F-03104cbaffd2 prepares context-associated state, submits a local input cursor to a dependency, and dispatches according to the 16-bit value found at the resulting cursor.

The established behavior is a four-way numeric dispatch. The recognized values are 0x0044, 0x004D, 0x0057, and 0x0059. Each selects one distinct dependency. An unrecognized value produces no dispatch call, but only after both preparation calls and the discriminator read have occurred.

The values coincide with the character codes for D, M, W, and Y in common character encodings. Interpreting them as command letters is inferred intent, not an established encoding or parsing contract. No surrounding text format, delimiter convention, or meaning for the four operations is established.

## Interface

The function accepts four arguments and has no explicit return value:

| Argument | Established role | Width and interpretation |
|---|---|---|
| Context address | Base used to form two context-relative addresses; also passed unchanged to the selected handler | Represented as a 32-bit integer in the supplied source representation |
| Unused value | Accepted but not explicitly read, stored, or forwarded by this function | Opaque 32-bit value |
| Initial cursor | Initializes a local pointer whose address is passed to dependencies | Pointer to a 16-bit signed element; pointer width is not independently established |
| Auxiliary value | Copied into a separate local storage location before either preparation call | Opaque 32-bit value |

There is no returned status, resulting cursor, or handler result. Any caller-visible result must arise through dependency effects, context changes, changes to reachable memory, or other external effects.

### Context-relative layout

Only two offsets are established:

- Context base plus 0x0C: supplied as the second argument to F-77303ea85faa.
- Context base plus 0x48: supplied as the first argument to F-77303ea85faa and as the second argument to F-868e5d53f111.

These identify addresses, not proven field types or extents. The available evidence does not establish whether either address denotes a scalar, an embedded object, a buffer, or another structure. It also does not establish the amount of memory accessed by the dependencies.

The address calculations use offsets of 12 and 72 bytes in the supplied representation. No overflow, wraparound, alignment, range, or context-validity checks are explicit. A portable replacement must not silently select an arithmetic-overflow policy without additional platform evidence. For ordinary execution, the context address must support the address calculations and all accesses actually performed by the dependencies.

### Local cursor and auxiliary storage

The initial cursor is copied into a local pointer object. Dependencies receive the address of that pointer object, rather than the initial cursor value alone. This permits a dependency to replace the cursor that the wrapper subsequently reads.

The auxiliary argument is also copied to local storage. The wrapper does not explicitly read that copy afterward or pass its address as a distinct argument. The evidence does not establish a public aggregate layout connecting this value to the local cursor. Its placement does not justify specifying a pointer-and-length pair, a second cursor field, or a particular byte offset.

## Behavior

### Preparation

The established local sequence begins by storing the initial cursor and auxiliary value in their respective local storage locations.

The function then calls F-77303ea85faa with these arguments, in order:

1. The address at context base plus 0x48.
2. The address at context base plus 0x0C.

This call occurs regardless of the initial cursor value and regardless of which discriminator will ultimately be read. There is no explicit branch that skips it.

After that call returns normally, the function calls F-868e5d53f111 with:

1. The address of the local cursor object.
2. The address at context base plus 0x48.

The second call therefore follows the first call and receives the same context-relative address used as the first call’s first argument. It is not established that the contents at that address remain unchanged between the calls; the first dependency may modify them.

Neither preparation call has a return value that this function explicitly examines. There is no visible status-based retry, rollback, or early return between them.

### Discriminator read

After F-868e5d53f111 returns normally, the function reads one 16-bit signed element through the current local cursor.

The cursor used for this read is the value present after the dependency call. It need not equal the initial cursor argument. Consequently, the value initially pointed to by the caller is not necessarily the discriminator.

The read requires the current cursor to designate storage that can be read as a 16-bit element under the target platform’s access rules. The wrapper performs no explicit null, bounds, or alignment check. It also provides no explicit end-of-input check.

The discriminator is read once by the wrapper and retained for branch selection. This does not constrain how often dependencies may read the same memory.

The comparison constants are all positive values representable in a signed 16-bit element. Thus the recognized bit patterns are exactly 0x0044, 0x004D, 0x0057, and 0x0059. A value such as 0x0144 does not match 0x0044 merely because its low byte is the same. Negative 16-bit values match none of the four constants.

### Dispatch

The comparisons occur in this order:

| Discriminator | Selected dependency |
|---|---|
| 0x0044 | F-ab9b9a705de1 |
| 0x004D | F-33f828947f25 |
| 0x0057 | F-978cdde47a14 |
| 0x0059 | F-9dc23492acea |

A selected dependency receives:

1. The original context base value.
2. The address of the same local cursor object used during preparation.

Only one handler is selected. Following a normally returning handler call, this function returns without another wrapper-level discriminator read or dispatch attempt.

For any other discriminator, no handler is called and the function returns. This is not a side-effect-free rejection: preparation has already occurred.

### Cursor movement

The wrapper does not explicitly advance the cursor before dispatch, after dispatch, or on an unrecognized discriminator.

Any cursor movement must therefore be attributed to a dependency, not to an implicit one-element consumption rule. In particular, there is no established guarantee that a recognized discriminator is consumed, that an unrecognized discriminator remains unconsumed, or that a handler receives the original input address.

A handler can modify the local cursor through the supplied pointer-object address. Such a modification is not returned directly to the caller, because the caller supplied the initial cursor by value and the function has no cursor return value.

## Side effects

The wrapper’s explicit memory operations are:

- Initialization of the local cursor object.
- Initialization of the local auxiliary storage.
- A read of the local cursor after preparation.
- One 16-bit read through that cursor.

No direct write to context memory or input elements is explicit in the wrapper. Nevertheless, the function cannot be described as read-only: it invokes dependencies with context-relative addresses and an address through which the local cursor can be changed.

The dependency calls form an observable ordering constraint. On a normal recognized path, the sequence is preparation by F-77303ea85faa, preparation by F-868e5d53f111, discriminator read, selected handler, return. On a normal unrecognized path, it is the same sequence without a handler.

There is no explicit allocation, release, ownership transfer, locking, or global access in the wrapper. These absences do not establish that dependencies avoid such actions. Their effects on globals, memory ownership, input storage, context storage, and external resources remain unknown.

The address passed for the cursor refers to local storage. Its lifetime does not extend beyond this invocation. The wrapper supplies no persistent cursor object. Whether a dependency retains that address is unknown; safe later use cannot be inferred.

No explicit error reporting or recovery exists. If a dependency does not return normally, the later sequence is not guaranteed. If the post-preparation cursor is invalid, the wrapper supplies no defined validation outcome or fallback.

## Dependencies

| Opaque ID | Established use | Not established |
|---|---|---|
| F-77303ea85faa | First preparation call, receiving context-relative addresses +0x48 and +0x0C | Object sizes, initialization semantics, ownership, failure behavior |
| F-868e5d53f111 | Second preparation call, receiving the local cursor’s address and context-relative address +0x48 | Cursor normalization, advancement, validation, interpretation of adjacent storage |
| F-ab9b9a705de1 | Handler for 0x0044; receives context and local cursor’s address | Operation meaning, input consumption, mutations, errors |
| F-33f828947f25 | Handler for 0x004D; receives context and local cursor’s address | Operation meaning, input consumption, mutations, errors |
| F-978cdde47a14 | Handler for 0x0057; receives context and local cursor’s address | Operation meaning, input consumption, mutations, errors |
| F-9dc23492acea | Handler for 0x0059; receives context and local cursor’s address | Operation meaning, input consumption, mutations, errors |

None of these dependencies may be assumed to preserve globals, retain ownership, release memory, or leave context and cursor state unchanged without separate evidence.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed test results. Unless stated otherwise, preparation calls return normally and leave a cursor valid for the discriminator read.

1. **Recognized value 0x0044.** If the post-preparation cursor points to 0x0044, the two preparation calls occur in order, followed by F-ab9b9a705de1. None of the other three handlers is selected.

2. **Other recognized values.** Under the same conditions, 0x004D selects F-33f828947f25, 0x0057 selects F-978cdde47a14, and 0x0059 selects F-9dc23492acea. Each receives the context base and the local cursor object’s address.

3. **Unrecognized zero.** If the discriminator is 0x0000, both preparation calls still occur, followed by the discriminator read. No handler is selected. Zero is not an explicit pre-preparation terminator.

4. **Full-width comparison.** A discriminator of 0x0144 selects no handler. Matching the low byte of 0x0044 is insufficient.

5. **Cursor replacement.** Suppose the initial cursor points to 0x0044, but F-868e5d53f111 replaces it with a valid cursor pointing to 0x0059. F-9dc23492acea is selected, not F-ab9b9a705de1.

6. **No repeated dispatch.** If a selected handler changes the local cursor to point to another recognized value and returns normally, the wrapper returns without dispatching that next value.

7. **Invalid initial cursor repaired during preparation.** An initially invalid cursor does not itself prove failure: the wrapper does not dereference it before preparation. If preparation replaces it with a valid cursor and otherwise completes normally, dispatch follows the replacement value.

8. **Invalid resulting cursor.** If preparation leaves an unreadable cursor, no clean rejection or reported error is specified. The wrapper attempts the 16-bit read without an explicit guard.

## Uncertainties

The following are **documented_only**, not guessed requirements:

- Exact platform pointer width, byte order, alignment rules, and address-overflow semantics.
- Context allocation size, field types, and the memory extents used by dependencies.
- Any structured relationship between the local cursor and auxiliary storage.
- Whether the auxiliary value influences a dependency through an unestablished storage convention.
- Text encoding, command meanings, terminators, lengths, and input-consumption rules.
- Dependency return conventions, exceptional exits, resource ownership, global effects, and reentrancy.
- Whether input or context storage is modified indirectly.
- Any caller-level interpretation of unrecognized values.

The unused second argument has no explicit dataflow into this function’s calls or dispatch decision. This is a statement about the visible wrapper, not a universal claim about machine-level observability.

## Evidence

Established static evidence consists of the supplied function body: four arguments, local cursor and auxiliary-value initialization, two ordered preparation calls, one 16-bit discriminator read, four exact comparisons, and the corresponding handler calls.

The preparation and handler roles are descriptive labels for their positions in the sequence. A parsing or command-processing purpose is inferred intent only.

No dependency bodies, independent ABI specification, or runtime observations establish the unresolved details. No runtime tests have been performed.
