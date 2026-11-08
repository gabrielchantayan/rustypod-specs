<!-- Trusted provenance: F-d03768eedb4b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d03768eedb4b

## Purpose

F-d03768eedb4b processes a request to change an indexed item’s state. It filters some requests as duplicates, updates shared storage for accepted requests, optionally invokes a callback, and selects either descriptor-based dispatch or special-item processing.

“Activation” denotes a nonzero requested state; “deactivation” denotes zero. These are descriptive terms, not established application semantics.

Two identifiers receive exceptional treatment:

- **0:** duplicate filtering applies, but an accepted request skips the counter increment and indexed state write.
- **126:** duplicate filtering is bypassed; every invocation performs the counter increment and indexed state write.

The function’s behavior depends on when shared references are evaluated. References saved earlier and references loaded later must not be treated as interchangeable unless their stability is an explicit condition.

## Interface

### Inputs and result

| Input | Representation | Established use |
|---|---|---|
| Item identifier | Unsigned 32-bit integer | Indexed byte accesses, callback argument, event-word construction, and tracked-item comparison |
| Requested state | Signed 32-bit integer | Zero/nonzero selection and unchanged callback argument |
| Dispatch argument | Unsigned 32-bit integer | Unchanged second argument to F-5836e250b35a when that dependency is called |

A negative requested state follows the activation branch. The callback receives the original value, not a normalized Boolean.

No return value is established. Observable outcomes are memory accesses, shared-state changes, and dependency calls.

### Shared-storage roles

| Role | Established layout or use |
|---|---|
| State storage | A 32-bit counter at offset 0; indexed state bytes beginning at offset 4 |
| Context storage | Status byte at offset 0; metadata byte at offset 1; repeat-count byte at offset 3; last-item byte at offset 4; callback reference at offset 8; 32-bit value at offset 24 |
| Classification storage | Indexed bytes; zero selects ordinary processing, nonzero selects special processing |
| First auxiliary storage | Indexed bytes tested and cleared during ordinary deactivation |
| Second auxiliary storage | Indexed bytes cleared during ordinary deactivation |
| Shared object | Contains a 32-bit field at offset 24 |
| Comparison token | Compared with the shared object’s field |
| Activation and deactivation descriptors | 32-bit values selected for the first word of a temporary record |

These roles do not establish allocation sizes, ownership, external structure definitions, or application meaning.

### Preconditions

No bounds checking, lifetime checking, general null checking, or synchronization is visible. The callback-presence test checks one evaluated callback reference for zero; it does not validate separately evaluated references used for the subsequent call.

Every access actually performed must address suitable live storage:

- Duplicate filtering requires the indexed state byte except for identifier 126.
- State-updating paths require a writable counter through the references evaluated for that operation and a writable indexed state byte through the entry-saved reference.
- Accepted requests require context storage and an indexed classification byte.
- Ordinary deactivation requires the applicable auxiliary bytes.
- Special activation requires the object field and context tracking fields.
- An executed indirect callback must have a valid target and compatible calling convention.

The unsigned identifier type does not establish support for every 32-bit identifier. Address expressions use the supplied representation’s address-width arithmetic; they provide no bounds protection or guarantee against address wraparound.

## Behavior

### Entry capture and duplicate filtering

At entry, the function saves the current state-storage address. This saved address is used later for the indexed state write, not explicitly for duplicate filtering or the counter operation.

For identifiers other than 126, the function evaluates the global state-storage reference and reads its byte at offset `4 + identifier`.

- A zero request returns immediately if the byte is zero.
- A nonzero request returns immediately if the byte is nonzero.

Any nonzero stored byte counts as active. It need not equal 1.

These returns occur before counter updates, indexed state writes, context capture, callback processing, classification-reference loading, and downstream calls. The entry reference has already been saved.

Identifier 126 skips the filtering byte read and both duplicate checks.

### Counter and indexed state update

An accepted request for an identifier other than 0:

1. Reads and increments the counter using evaluations of the global state-storage reference.
2. Writes the normalized requested state through the state-storage address saved at entry.

The indexed write stores 0 for a zero request and 1 otherwise. The counter update precedes this byte write, and both precede callback processing.

The counter operation’s global-reference evaluations are distinct from the entry capture. If the reference changes externally, the counter and indexed state write need not affect the same storage region. The supplied representation does not establish a finer ordering among evaluations within the counter assignment.

Identifier 0 skips both updates after passing filtering. Consequently, its indexed state byte can remain opposite to the requested state across repeated accepted calls, provided dependencies and external actors do not change it.

Identifier 126 performs both updates on every invocation, even when its existing state would otherwise constitute a duplicate.

The counter is represented as a signed 32-bit integer. Its overflow behavior is not established; this contract does not guarantee modulo-\(2^{32}\) wraparound.

### Context capture and optional callback

After any state updates, the function saves the current context-storage address. Subsequent metadata, status, tracking, and context-offset-24 accesses use this saved address.

Callback processing differs:

- The callback-presence test evaluates the global context reference and checks the callback reference at offset 8.
- If that test finds a nonzero reference, callback-target lookup separately evaluates the global context reference.
- The callback’s interior-pointer argument also separately evaluates the global context reference.

The callback receives:

1. The original signed 32-bit requested state.
2. The full unsigned 32-bit identifier.
3. An address one byte into the context obtained for that argument.

These evaluations need not identify the same context under external mutation. A successful presence test therefore does not independently guarantee that the later target is nonzero or that its context matches the interior pointer. The representation does not establish an ordering between target lookup and argument evaluation within the call expression.

If invoked, the callback supplies a 32-bit contribution to a local event word. Otherwise, that contribution is zero.

### Classification capture and event-word construction

After callback processing completes, the function loads and saves the classification-storage reference. This load occurs **before** the metadata read and event-word construction.

It then reads the metadata byte at offset 1 through the saved context address and constructs the 32-bit event word by bitwise OR of:

- The callback contribution.
- The zero-extended metadata byte shifted left by 24 bits.
- The unsigned identifier shifted left by 16 bits.

The identifier is not masked to eight bits before shifting. Its low 16 bits survive; identifier bits 8–15 overlap the metadata contribution. The callback contribution can set bits anywhere. The operation combines bits rather than replacing isolated fields.

A callback can therefore affect which classification reference is captured and what metadata is subsequently read. Replacing the global context reference does not redirect those metadata accesses away from the saved context.

### Zero request

The function writes 1 to the saved context’s status byte.

It next captures the current first auxiliary-storage reference for a possible later clearing write. This capture occurs **before** reading the indexed classification byte through the saved classification reference.

#### Special classification

If the classification byte is nonzero, the function returns.

The counter/state processing, callback processing, classification-reference capture, event-word construction, status write, and first auxiliary-reference capture have already occurred. The auxiliary capture does not itself read an indexed auxiliary byte.

Neither auxiliary byte is cleared. No descriptor dispatch or special-activation dependency is reached.

#### Ordinary classification

If classification is zero, the function selects the deactivation descriptor as the first word of the temporary record.

It then **reloads** the first auxiliary-storage global and reads the indexed marker byte through that reference. This marker read does not explicitly use the reference saved for clearing.

If the marker is nonzero:

- F-8c9273cab988 is called with the temporary-record address.
- After normal return, the function explicitly writes zero to the record’s first word.

That explicit zero suppresses later F-5836e250b35a dispatch even if F-8c9273cab988 changed the first word to a nonzero value.

After the marker check and any resulting call, the function loads the second auxiliary-storage reference. This load occurs **before either clearing write**.

It then clears:

1. The indexed byte through the first auxiliary reference saved before classification testing.
2. The indexed byte through the newly loaded second auxiliary reference.

Thus, the tested first-auxiliary marker and the cleared first-auxiliary byte can belong to different storage regions. A replacement first-auxiliary global made by F-8c9273cab988 does not redirect the saved clearing address. A replacement second-auxiliary global made before that call returns can affect the second clearing address.

Processing then reaches the descriptor-dispatch tail.

### Nonzero request

The function writes 0 to the saved context’s status byte, then reads the indexed classification byte through the saved classification reference.

#### Ordinary classification

If classification is zero, the activation descriptor becomes the temporary record’s first word, and processing reaches the descriptor-dispatch tail.

Neither auxiliary array is read or cleared on this path.

#### Special classification

If classification is nonzero, the function reads the shared object’s 32-bit field at offset 24 and compares it with the comparison token.

On equality:

- It reads the saved context’s last-item byte as an unsigned byte and compares it with the full 32-bit identifier.
- If equal, it increments the repeat-count byte and stores the result back into that byte.
- If unequal, neither tracking byte is changed by this block.
- It invokes F-da2d75c0df8d with a newly evaluated shared-object reference.

The byte increment discards bits beyond eight bits: a repeat count of 255 becomes 0.

On inequality between the object field and comparison token:

- The repeat-count byte becomes 1.
- The last-item byte receives the identifier’s low eight bits.
- F-da2d75c0df8d is skipped.

Both outcomes then invoke F-689e9e98c8f7 with:

1. The shared-object reference evaluated for that call.
2. The 32-bit value read at offset 24 through the saved context address.

A final dependency, for which no opaque ID is supplied, is then invoked with another evaluated shared-object reference. Processing returns without descriptor dispatch.

These object-reference evaluations must not be assumed identical across dependency calls. The second argument to F-689e9e98c8f7 is read after any F-da2d75c0df8d call, so mutations to the saved context by that dependency can affect it.

Because last-item storage is one byte but comparison uses the full identifier, identifiers above 255 cannot match the stored byte, even when their low eight bits match.

### Descriptor-dispatch tail

Only ordinary classification reaches this tail.

Two separate 32-bit local fields are initialized to zero before any F-5836e250b35a call. If the temporary record’s first word is nonzero, F-5836e250b35a receives the record address and the original dispatch argument. Otherwise, no call occurs.

The temporary record is represented as three 32-bit words. Only its first word is explicitly initialized. The other two words have no established initial values.

The two separately zeroed fields and the event word appear nearby in the represented local storage. Their inclusion in a larger consumer-visible record is not established. In particular, the event word’s construction does not prove that a dependency consumes it.

## Side effects

### Established ordering

For an accepted request, the relevant sequence is:

1. Counter update and indexed state write, except for identifier 0.
2. Context-reference capture.
3. Callback-presence check and, conditionally, callback invocation using separate global-context evaluations.
4. Classification-reference capture.
5. Saved-context metadata read and event-word construction.
6. Saved-context status write.
7. Branch-specific processing.

For zero requests, branch-specific ordering is:

1. First auxiliary-reference capture for clearing.
2. Classification-byte read.
3. Return if special.
4. Deactivation-descriptor selection.
5. First auxiliary-global reload and marker read.
6. Optional F-8c9273cab988 call, followed by explicit first-word zeroing.
7. Second auxiliary-reference load.
8. First auxiliary clearing write.
9. Second auxiliary clearing write.
10. Descriptor-dispatch tail.

On special activation, any F-da2d75c0df8d call precedes F-689e9e98c8f7, which precedes the unresolved final dependency.

### Ownership, failure, and concurrency

The function visibly allocates or releases no dynamic resource and establishes no ownership transfer.

The descriptor address refers to invocation-local storage. Validity is established during the relevant calls, not after the enclosing invocation returns. Whether dependencies retain or copy it is unknown.

The callback receives an interior pointer into shared context storage. No ownership transfer is established.

Dependencies are not assumed to preserve globals, object references, or context contents. Saved addresses remain the function’s chosen addresses even when a global is replaced; their continued validity remains a precondition.

There is no visible error return, rollback, exception handler, or synchronization. A dependency that does not return normally can prevent subsequent writes and calls; earlier writes are not visibly undone. Multi-access operations are not established as atomic or safe against concurrent mutation.

## Dependencies

| Opaque ID | Established invocation |
|---|---|
| F-8c9273cab988 | Ordinary deactivation with a nonzero first-auxiliary marker; receives the temporary-record address before the tail’s separate zero-field initialization |
| F-5836e250b35a | Ordinary classification with a nonzero first descriptor word; receives the temporary-record address and original dispatch argument |
| F-da2d75c0df8d | Special activation when the object-field comparison succeeds, whether or not the tracked identifier matches; receives the shared object |
| F-689e9e98c8f7 | Every special activation; receives the shared object and saved-context 32-bit value at offset 24 |

No opaque IDs are supplied for the optional indirect callback or final special-activation dependency. Their invocation positions and arguments are established above; their identities and internal behavior remain **documented_only**. No replacement IDs or preservation contracts are invented.

Only the optional callback’s returned value is visibly incorporated into subsequent computation. Other dependency results are not used as explicit success or failure statuses.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless otherwise stated, storage is valid, arithmetic avoids unresolved signed overflow, dependencies return normally, and relevant references remain stable.

1. **Duplicate deactivation.** With identifier 5, stored byte 0, and request 0, filtering returns without counter or state writes, context processing, or dependency calls.

2. **Noncanonical active byte.** With identifier 5, stored byte 7, and request −1, filtering returns. The stored byte need not equal 1 to count as active.

3. **Ordinary activation.** With identifier 5, stored byte 0, classification 0, request −1, and a nonzero activation descriptor, the counter increments, the indexed state becomes 1, any callback receives −1 unchanged, status becomes 0, and F-5836e250b35a receives the original dispatch argument.

4. **Identifier exceptions.** Identifier 0 with stored byte 0 accepts a nonzero request without changing the counter or indexed state. Identifier 126 with stored byte 1 accepts another nonzero request and still performs both updates.

5. **Distinct state references.** If state storage is replaced after entry capture but before filtering and counter processing, those operations use their evaluated global references, while the indexed state write targets the entry-saved region. This expectation additionally requires both regions to remain valid.

6. **Distinct callback contexts.** If the saved context, presence-check context, callback-target context, and interior-pointer context differ, no equivalence is required. After the callback returns, metadata, status, tracking, and offset-24 accesses still use the saved context.

7. **Special deactivation.** An accepted zero request writes status 1, captures the first auxiliary reference, and then observes nonzero classification. It returns without reading an auxiliary marker or clearing auxiliary bytes.

8. **Separate marker and clearing regions.** Suppose the first auxiliary reference is captured from one region, but its global changes before the marker read. A nonzero marker in the replacement region triggers F-8c9273cab988; the later first clearing write still targets the earlier region. The second auxiliary reference is loaded after that call and before either clear.

9. **Descriptor suppression.** For ordinary deactivation with a nonzero marker, F-8c9273cab988 runs, the first descriptor word is explicitly zeroed, both auxiliary bytes are cleared, and F-5836e250b35a is skipped.

10. **Repeat-count boundary.** With special activation, a successful object comparison, identifier 9 matching the last-item byte, and repeat count 255, the count becomes 0. Calls proceed through F-da2d75c0df8d, F-689e9e98c8f7, and the unresolved dependency.

11. **Wide identifier.** Identifier 258 cannot match a last-item byte of 2. Its event-word contribution is `0x01020000`; metadata 128 and callback contribution zero produce `0x81020000`. Indexed storage must independently be valid for identifier 258.

## Uncertainties

The following remain **documented_only**:

- Application meanings of activation, classification, descriptors, and comparison token.
- Actual storage extents and valid identifier ranges.
- Signed counter-overflow behavior and address-wrap consequences.
- Fine-grained ordering within expressions containing multiple global-reference evaluations.
- Complete descriptor layout and whether nearby local fields are consumed indirectly.
- Dependency mutation, reentrancy, retention, ownership, failure, and resource-release behavior.
- Identities of dependencies lacking supplied opaque IDs.
- Threading and interrupt requirements.

Control-item and repeat-handling interpretations are inferred intent, not additional behavioral requirements.

## Evidence

Established behavior comes solely from static inspection of the supplied representation and opaque dependency map: branch conditions, widths, offsets, separately evaluated references, saved-address uses, explicit writes, call arguments, early returns, and statement-level ordering.

The representation does not prove dependency internals, larger record boundaries, application semantics, or stability of independently evaluated globals.

No runtime tests, execution traces, or implementation verification have been performed.
