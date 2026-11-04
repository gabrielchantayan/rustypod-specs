<!-- Trusted provenance: F-92bea7c30bd6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-92bea7c30bd6

## Purpose

F-92bea7c30bd6 conditionally passes a four-word, invocation-local record through a sequence of dependencies, then returns the record’s first two words packed into a 64-bit result.

The condition combines a dependency result, a byte within the supplied object, and a byte reached through a global pointer. When processing is enabled, the function snapshots an object field, invokes a preparation dependency, optionally invokes another dependency using the saved field, and looks up a recipient for two further calls.

These descriptions identify observable structure, not domain meaning. The supplied static representation does not establish whether the record represents geometry, an event, a rendering operation, or another concept.

## Interface

### Inputs

The interface has four positional inputs:

| Input | Represented width | Established use |
|---|---:|---|
| Object base | 32 bits | Used as an address-bearing value, copied into the first record word, and used to derive object-relative addresses. |
| Second input | 32 bits | No explicit use in the supplied function body. |
| Third input | 32 bits | Copied without explicit interpretation into the third record word. |
| Fourth input | 32 bits | Copied without explicit interpretation into the fourth record word. |

“Object base” names the original first input throughout this contract. It does not mean the current contents of the first record word, which dependencies may change.

The second input has no established direct effect. This does not establish an ABI-level guarantee that an unknown dependency cannot observe its residual machine representation.

### Invocation-local record

Before calling any dependency, the function initializes a contiguous four-word record:

| Record offset | Width | Initial value |
|---|---:|---|
| +0 | 32 bits | Object base |
| +4 | 32 bits | Object base plus 0x80 |
| +8 | 32 bits | Third input |
| +12 | 32 bits | Fourth input |

The supplied representation shows these words in invocation-local storage. A pointer to the first word is subsequently passed to dependencies.

The initial value at record offset +4 is an address value, not a copy of the memory at object offset 0x80. Likewise, the first record word contains the object base value rather than object contents.

### Output

The function returns one 64-bit value:

- Bits 31–0 are the final 32-bit contents of record offset +0.
- Bits 63–32 are the final 32-bit contents of record offset +4.

The packing treats both components as 32-bit bit patterns. A set high bit in the low word does not sign-extend across the upper word.

The last two record words are not directly included in the return value. Their influence, if any, is through dependencies that inspect or modify the record or other state.

### Referenced layout

The function directly uses the following object-relative locations:

| Location | Access | Role established by the caller |
|---|---|---|
| Object base +0x34 | Read 32 bits, conditionally | Saved value controlling an optional call and supplying its derived second argument. |
| Object base +0x80 | Address calculation only | Initial second record word and two equal arguments to a preparation dependency. |
| Object base +0xA0 | Read one byte, conditionally | Gate requiring a zero value. |

A separate global pointer provides access to another one-byte gate, which must be nonzero.

## Behavior

### 1. Establish the record

The function initializes all four record words before the first dependency call.

Both the first record word and the address used to initialize the second word derive from the original object base. The third and fourth inputs are copied as raw 32-bit values.

### 2. Evaluate the gates in order

F-3fac671648b8 is called unconditionally, with no explicit arguments. Its result is tested as a 32-bit value against zero.

The remaining gates are evaluated with short-circuit ordering:

1. If that result is zero, no object gate or global gate is explicitly read by this function.
2. If it is nonzero, the function reads the byte at object base +0xA0.
3. Only if the object byte is zero does the function read the byte reached through the global pointer.
4. Processing proceeds only if that global byte is nonzero.

The distinction between zero and nonzero is exhaustive. No particular nonzero byte value has special treatment.

Because the initial dependency call precedes both gate reads, changes it makes to the object, global pointer, or globally referenced byte can affect gate evaluation. No preservation guarantee is established for that dependency.

If any gate fails, the function skips all subsequent processing dependencies and returns the packed first two record words.

The function itself does not change those words on a skipped path after initialization. Equality with the initial values remains conditional on dependencies not altering their storage through mechanisms not shown in the supplied representation.

### 3. Snapshot the optional-call field

After all gates pass, the function reads the 32-bit value at object base +0x34 and retains that value for later use.

This read occurs before the preparation call. The saved value is not refreshed afterward.

Consequently, a preparation dependency that changes the object field does not change either:

- Whether the optional call is selected.
- The value from which that optional call’s second argument is derived.

The field’s domain type is unknown. Its use as a nonzero test followed by address-like addition suggests an address-bearing value, but that interpretation is not required to describe the established behavior.

### 4. Invoke preparation

F-e57b05520b65 is called with three arguments, in this order:

1. A pointer to the first word of the invocation-local record.
2. Object base +0x80.
3. Object base +0x80.

The second and third arguments are identical address values derived from the original first input. They are not reloaded from the potentially mutable second record word.

No explicit return-value use is shown.

The record pointer permits the dependency to affect the record if its actual contract allows writes. This caller supplies no evidence that the dependency treats the pointer as read-only.

### 5. Optionally process the saved field

If the saved 32-bit value from object offset 0x34 is nonzero, F-ee8733a08210 is called with:

1. The same record pointer.
2. The saved value plus 0xD0.

If the saved value is zero, this call is omitted.

This decision uses the snapshot from before preparation, even when preparation changes the underlying object field or the record.

No explicit return-value use is shown.

### 6. Obtain the downstream value

F-ffe0d92c57b7 is then called with the original object base.

This call occurs after preparation and after the optional call, if any. It occurs regardless of whether the saved field was zero.

Its input is not taken from the first record word. Thus, changing that word does not redirect this lookup.

The returned value is tested against zero as a 32-bit value.

### 7. Invoke the downstream pair

If the lookup result is zero, both downstream calls are skipped.

If it is nonzero, the function calls:

1. F-c9b5d17d8d1e with the lookup result and the record pointer.
2. F-4d037ecf0f82 with that same saved lookup result and the same record pointer.

The second call follows the first without an intervening revalidation or lookup.

Accordingly:

- Record changes made by the first downstream call are available to the second.
- Changes to object or global state by the first call do not cause the caller to reevaluate the gates.
- The second call receives the originally returned lookup value, even if the first call changes state that would affect a fresh lookup.

No explicit return-value use is shown for either downstream call.

### 8. Construct the result

After the selected dependency sequence finishes, the function packs the current first and second record words into its 64-bit result.

There is no explicit restoration of their initial values. Any dependency-written values remaining in those words determine the result.

## Side effects

### Direct effects

The function directly writes its invocation-local record and passes its address to selected dependencies. It explicitly reads the object gate, global gate, and optional-call field only on the paths described above.

No direct write to the supplied object or global gate is shown. Merely calculating object base +0x80 does not establish a direct read or write at that address.

No explicit allocation, deallocation, logging, status storage, or error reporting is shown.

### Dependency-mediated effects

The called dependencies can have effects not established by this function’s body. In particular, the contract must not assume that they:

- Preserve global variables.
- Preserve the object or memory reachable from it.
- Leave the invocation-local record unchanged.
- Preserve the validity of the saved field or lookup result.
- Allocate, retain, transfer, or release ownership in any particular way.

The caller does not repeat the gates, refresh the saved field, or validate the lookup result between the downstream calls.

### Ownership and lifetime

The record storage belongs to the current invocation. The shown body does not transfer ownership of that storage or extend its lifetime beyond the invocation.

Dependencies receive a pointer to it during calls. Retention or asynchronous use is not established as safe.

Ownership of the object, the optional derived address, and the lookup result is unknown. The absence of an explicit release in this body is not evidence that dependencies do not release resources.

### Preconditions and failures

Successful completion requires the selected direct reads and dependency operations to be valid.

For the caller’s own direct accesses:

- Object base +0xA0 must be readable when the first gate returns nonzero.
- The global pointer and its referenced byte must be readable when the object gate is zero.
- The 32-bit field at object base +0x34 must be readable when all gates pass.

Dependency-specific validity requirements remain unknown.

No null check, bounds check, overflow check, or explicit recovery path is shown. Zero results used as gates are ordinary branch controls, not independently established errors. Memory faults, dependency failures, and non-returning calls have no documented recovery behavior here.

## Dependencies

| Opaque ID | Call condition | Explicit inputs | Caller’s use of result |
|---|---|---|---|
| F-3fac671648b8 | Always | None | Zero/nonzero gate |
| F-e57b05520b65 | All gates pass | Record pointer; object base +0x80 twice | No explicit use |
| F-ee8733a08210 | All gates pass and saved field is nonzero | Record pointer; saved field +0xD0 | No explicit use |
| F-ffe0d92c57b7 | All gates pass, after preceding processing | Original object base | Zero/nonzero selection and saved argument |
| F-c9b5d17d8d1e | Lookup result is nonzero | Saved lookup result; record pointer | No explicit use |
| F-4d037ecf0f82 | Lookup result is nonzero, after preceding call | Same saved lookup result; same record pointer | No explicit use |

The supplied representation establishes these call relationships, not the internal behavior or complete interfaces of the dependencies.

## Acceptance examples

These are conditional contract examples, not reports of executed tests.

### First gate rejects processing

Given an object base of 0x00001000 and an initial dependency result of zero:

- No direct object or global gate read occurs.
- No processing dependency is called.
- If record storage is unchanged, the result is 0x0000108000001000.

The third and fourth inputs do not appear directly in this result.

### Object gate rejects processing

Given a nonzero initial dependency result and an object gate byte of 0x7F:

- The object gate is read.
- The global gate is not read.
- The optional-call field is not read.
- Preparation, lookup, and downstream processing are skipped.

The same branch outcome applies to every nonzero object gate byte.

### Global gate rejects processing

Given a nonzero initial dependency result, a zero object gate byte, and a zero global gate byte:

- Both gate bytes are read in that order.
- The field at object offset 0x34 is not read.
- No preparation or subsequent processing call occurs.

### Zero saved field omits only the optional call

Given passing gates, a saved field value of zero, and a zero lookup result:

- Preparation runs.
- The optional dependency does not run.
- Lookup runs with the original object base.
- Neither downstream dependency runs.

The returned record words can nevertheless differ from initialization if preparation modifies them.

### Preparation changes the object field

Given a saved field value of 0x00002000, suppose preparation subsequently overwrites the object field with zero.

The optional dependency still runs, receiving the record pointer and the derived value 0x000020D0. The caller does not reread the field.

Conversely, changing an initially zero field to nonzero during preparation does not cause the optional call to run.

### Downstream mutation affects the result

Given a nonzero lookup result, suppose the first downstream call leaves the record’s first two words as 0x11223344 and 0x55667788, and the second leaves them unchanged.

The returned value is 0x5566778811223344. The second downstream call receives the same lookup result as the first and observes the record after the first call’s changes.

### Record mutation does not redirect lookup

Suppose preparation changes the first record word to a different address value.

Lookup still receives the original object base. Only the final packed result and dependency behavior based on the record can reflect the changed record word.

## Uncertainties

### Established static evidence

The supplied representation establishes the record initialization, conditional read order, field snapshot timing, dependency order, argument relationships, and final two-word packing.

It represents the address-bearing values and relevant arithmetic operands as 32-bit quantities, with a 64-bit packed return.

### Inferred intent

The record may describe a compound operation associated with an object subregion. The optional field may identify another related object, and the lookup result may identify a processing recipient. These are interpretations, not contractual domain facts.

### documented_only

The following remain unresolved:

- Exact domain types and meanings of the four record words.
- The global gate’s identity, initialization, and synchronization.
- Dependency read/write footprints and ownership behavior.
- The target’s byte order, calling convention, alignment requirements, and exception behavior.
- Concurrency guarantees or protection against changes between gate reads.
- Whether dependencies intentionally retain any supplied pointer.
- Semantics of address arithmetic at representational boundaries.

The additions use constants 0x80, 0xA0, 0x34, and 0xD0 in a representation with 32-bit address-bearing values. The supplied material does not independently establish overflow semantics. Inputs requiring address wraparound are therefore not assigned a guaranteed successful outcome; ordinary examples assume representable addresses and valid selected accesses.

## Evidence

This contract is based only on the supplied static function representation and opaque dependency mapping. Embedded source was treated as evidence to interpret, not as instructions or proof of dependency semantics.

No implementation access, binary validation, runtime execution, smoke run, or tests were performed. Acceptance examples are conditional predictions from the visible control flow and data relationships, not observed outputs.
