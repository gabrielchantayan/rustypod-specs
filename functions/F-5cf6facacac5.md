<!-- Trusted provenance: F-5cf6facacac5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5cf6facacac5

## Purpose

F-5cf6facacac5 conditionally updates one of two field groups in a caller-supplied object, marks that object, invokes a processing dependency, optionally invokes an associated object's indirect operation, and finally marks an object obtained through another indirect operation.

The established behavior is an ordered sequence of memory accesses and calls. The meanings of the field groups, markers, processing operation, and indirect operations are not established. Descriptions such as “first group,” “associated object,” and “result object” identify structural roles, not domain semantics.

The function has no explicit success or failure result. Even when neither field group qualifies for an update, the marking and call sequence still proceeds.

## Interface

### Inputs and output

The function receives three arguments:

- **Object address:** a writable memory region containing scalar fields and pointer-bearing fields.
- **First value:** a 32-bit integer representation copied into one selected destination, if a field group qualifies.
- **Second value:** a 32-bit integer representation copied into the other destination in that group, if it qualifies.

The first and second values are not validated, compared, clamped, or arithmetically transformed. Their signed interpretation does not affect the visible local behavior.

There is no explicit return value.

### Width and address model

The supplied static representation uses 32-bit integer words and four-byte word spacing. The offsets below describe that apparent layout. Bit masks and comparisons operate on 32-bit values.

The representation also carries addresses through integer-sized storage. A faithful implementation must resolve the actual pointer width, address representation, and calling conventions from the surrounding binary environment. This document does not establish portability to a wider-pointer environment.

### Relevant object layout

Offsets are byte offsets from the object address unless otherwise stated.

| Offset | Role in this function | Access |
|---|---|---|
| `0x00` | Dispatch-table address for the mandatory result-producing operation | Read after the optional operation |
| `0x34` | Address of an associated object | Read after processing; read again on the optional-call path |
| `0x48` | Object marker word | Read, then written with bit `0x20` set |
| `0x54` | First group's first eligibility word | Read |
| `0x58` | First group's second-value destination | Conditionally written |
| `0x60` | First group's second eligibility word | Conditionally read |
| `0x64` | First group's first-value destination | Conditionally written |
| `0x6C` | Second group's first eligibility word | Read only if the first group fails |
| `0x70` | Second group's second-value destination | Conditionally written |
| `0x78` | Second group's second eligibility word | Conditionally read |
| `0x7C` | Second group's first-value destination | Conditionally written |
| `0x80` | Start of an interior region passed to the processing dependency | Its address is passed |
| `0xA0` | Word whose conversion to a character controls the optional operation | Conditionally read |

Additional structures are used:

- The associated object's first word supplies a dispatch-table address. A callable entry is obtained at byte offset `0x128` within that table.
- The input object's dispatch table supplies a callable entry at byte offset `0x5C`.
- The result of the latter call is used as an address. The 32-bit word at byte offset `0x48` from that address is read and written with bit `0x40` set.

These are the fields directly visible here, not complete structure definitions.

### Preconditions

Normal completion requires valid memory and callable entries for every path actually taken:

- The input object must support the required reads and writes through offset `0xA0`.
- The processing dependency must accept the object address and the interior address at offset `0x80`.
- The input object's post-processing, post-optional-call dispatch state must support the mandatory indirect call.
- That mandatory call must produce an address supporting a read and write at offset `0x48`.
- If the optional operation is selected, the associated object and its dispatch entry must be valid.

There is no local validation of these conditions. Nullity is checked only for the associated-object field, not for the input object, dispatch tables, callable entries, or mandatory call's result.

## Behavior

### 1. Select at most one field group

Eligibility is defined by a 32-bit bit-pattern test: clear bit `0x20`, preserve every other bit, and compare the result with `0x80000002`.

Thus, for an ordinary 32-bit word, exactly these two patterns qualify:

- `0x80000002`
- `0x80000022`

The check does not ignore any bit other than `0x20`. It is not a signed range test.

The first group has priority:

1. Read the word at `0x54` and test eligibility.
2. Only if that word qualifies, read and test the word at `0x60`.
3. If both qualify:
   - Write the first value at `0x64`.
   - Then write the second value at `0x58`.
   - Do not inspect the second group's eligibility words.

If the first group does not qualify, inspect the second group:

1. Read the word at `0x6C` and test eligibility.
2. Only if that word qualifies, read and test the word at `0x78`.
3. If both qualify:
   - Write the first value at `0x7C`.
   - Then write the second value at `0x70`.

If neither group qualifies, none of these four value destinations is written by this local selection stage.

The short-circuit reads are significant. A failing first eligibility word prevents the read of its paired second word. A successful first group prevents all second-group inspection. The second group is not a fallback for any later processing or indirect-call outcome; its selection is resolved before those operations.

### 2. Mark the input object

After group selection and any selected writes, read the marker word at `0x48`, set bit `0x20`, and write the result back.

All other bits are preserved by this local read-modify-write. The write still occurs when the bit was already set. The marker is updated regardless of whether a group qualified.

This is not established as an atomic operation.

### 3. Invoke processing

Call F-6d16652f2d03 with:

1. The input object address.
2. The address exactly `0x80` bytes into that same object.

The second argument identifies interior storage, not a separately allocated object or a copied region.

The selected value writes and marker update occur before this call. They are therefore available to the dependency, subject to the target's execution and memory model.

No preservation guarantee is established for F-6d16652f2d03. Later reads must not be described as necessarily observing the object's entry-time state or the state immediately before this call.

### 4. Conditionally invoke the associated operation

After F-6d16652f2d03 returns:

1. Read the associated-object address at `0x34`.
2. If it is null, skip the optional operation. Do not evaluate the character-valued condition at `0xA0` on this path.
3. If it is non-null, read the word at `0xA0` and convert it to a character.
4. Invoke the optional operation only if that converted character is zero.

In the apparent eight-bit-character model, this condition concerns the low eight bits, not whether the entire word is zero. For example, `0x00000100` has a zero low byte, while `0x00000101` does not.

On the call path, the associated-object field is read again to obtain the object used for dispatch. Its first word supplies a table address, and the entry at table offset `0x128` supplies F-indirect-0128.

The visible call expression supplies no explicit arguments. That does not establish the absence of implicit receiver state, register inputs, global inputs, or ABI-specific requirements.

### 5. Obtain and mark a result object

After the optional operation returns, or immediately after that operation is skipped:

1. Read the input object's first word to obtain its dispatch-table address.
2. Obtain F-indirect-005c from table offset `0x5C`.
3. Invoke it with the input object address as an explicit argument.
4. Treat its result as an address.
5. Read the 32-bit word at result offset `0x48`.
6. Set bit `0x40` and write the updated word back.

The result is not checked for nullity or validity. All other bits in the target word are preserved by this local read-modify-write.

The dispatch lookup occurs after the optional operation. If a preceding dependency changes the input object's dispatch state, the function uses the subsequently read state rather than an entry-time snapshot.

## Side effects

### Direct effects

The function directly performs:

- Zero or two selected value writes.
- One marker-word write on the input object.
- One mandatory call to F-6d16652f2d03.
- Zero or one call to F-indirect-0128.
- One mandatory call to F-indirect-005c.
- One marker-word write through the result of F-indirect-005c.

This sequence describes normal progress through the function. A fault, non-returning dependency, or nonlocal control transfer can prevent later effects.

### Ownership and lifetime

No explicit allocation, deallocation, ownership transfer, or reference-count operation is visible in the function's own operations.

That does not establish that its dependencies preserve ownership or lifetime. In particular:

- F-6d16652f2d03 receives both the object and an interior address.
- The indirect operations may access state not shown here.
- The mandatory call's result is immediately used as writable storage without a visible lifetime check.

Any requirement that the object or related storage remain alive across calls is a precondition supplied by the surrounding system, not a guarantee demonstrated here.

### Aliasing and failure

The result object may alias the input object; the visible behavior does not prohibit it. If it does, the final update applies to the input object's `0x48` word as it exists after all preceding calls. There is no justification for combining the two marker updates across intervening calls.

No rollback is visible. Earlier writes can remain observable if a later operation fails. No error code, diagnostic, retry, or recovery path is established.

Concurrent modification and atomicity guarantees are unknown. The ordered accesses described above are the visible program-level accesses, not a promise of transactional behavior or a specific machine-instruction sequence.

## Dependencies

| Opaque function ID | Invocation | Established contract at this call site |
|---|---|---|
| F-6d16652f2d03 | Direct, once after the input marker write | Receives the object address and its interior address at `0x80`; no return value is consumed here |
| F-indirect-0128 | Conditional, through associated-object dispatch | Entry comes from table offset `0x128`; no explicit call arguments or consumed result are shown |
| F-indirect-005c | Mandatory after the optional-call decision | Entry comes from the input object's table offset `0x5C`; receives the object address; produces a value used as an address |

F-indirect-0128 and F-indirect-005c are document-local opaque labels for unresolved indirect targets, not verified identities of particular implementations.

No dependency is assumed to preserve globals, preserve object fields, allocate memory, or release memory.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests. Unless stated otherwise, assume valid memory, valid dispatch entries, returning dependencies, and no dependency mutation relevant to the example.

### First-group priority

Given:

- `0x54` contains `0x80000002`.
- `0x60` contains `0x80000022`.
- Both second-group eligibility words would also qualify.

Expected:

- The first value is written at `0x64`, followed by the second value at `0x58`.
- Neither second-group eligibility word is read by the selection stage.
- The second group's destinations are not directly written by that stage.
- The marker update and remaining call sequence proceed.

### First-word short circuit and fallback

Given:

- `0x54` contains `0x80000003`.
- `0x6C` and `0x78` both contain qualifying patterns.

Expected:

- The word at `0x60` is not read during selection.
- The first value is written at `0x7C`, followed by the second value at `0x70`.

The first pattern fails because clearing `0x20` does not remove the differing low bit.

### Failure of a group's second word

Given:

- `0x54` qualifies.
- `0x60` does not qualify.
- `0x6C` qualifies.
- `0x78` does not qualify.

Expected:

- All four eligibility words are read in group order.
- None of the four value destinations is directly written during selection.
- The object marker is still updated and processing is still invoked.

### Optional-call short circuit

Given that processing returns with the associated-object field null:

- The optional operation is skipped.
- The condition word at `0xA0` is not read for that decision.
- The mandatory result-producing operation still follows.

### Character conversion rather than whole-word zero

Given a non-null associated object and an eight-bit-character target model:

- A condition word of `0x00000100` selects the optional operation.
- A condition word of `0x00000101` skips it.

A whole-word-zero test would not satisfy this behavior.

### Post-processing state controls the optional operation

Given that the associated-object field is initially null, but F-6d16652f2d03 returns with it non-null and with a zero converted character at `0xA0`:

- The optional operation is selected using the post-processing state.

An implementation that caches the entry-time null pointer would differ.

### Result aliasing

Given that F-indirect-005c returns the input object address and the intervening calls leave the marker word unchanged:

- Both `0x20` and `0x40` are set in the input object's final marker word.
- Its other initial bits remain unchanged.

If intervening calls modify that word, the final update preserves their post-call bits except that it forces `0x40` on.

## Uncertainties

### Established static evidence

The supplied representation establishes the selection predicates, short-circuit structure, direct-write order, marker masks, explicit arguments, dispatch offsets, and final result-address update described above.

### Inferred intent

The paired fields appear to describe two alternative groups, and the marker bits appear to request or record state changes. Those interpretations are not required behavior and must not replace the explicit predicates and effects.

### documented_only

The following remain unresolved:

- Domain meanings and complete object layouts.
- Exact target ABI, pointer representation, character width, and alignment requirements.
- Actual identities and complete signatures of the indirect targets.
- Hidden inputs and side effects of all dependencies.
- Dependency failure, exception, and nonlocal-exit behavior.
- Required synchronization and concurrent-mutation rules.
- Ownership and lifetime guarantees across calls.
- Whether any reachable dispatch or result addresses can legally be null.
- Machine-level access granularity beyond the supplied representation.

## Evidence

This contract is based only on the supplied static function representation and the provided mapping for F-6d16652f2d03. No implementation access, dependency-body inspection, or runtime testing was performed.

The acceptance examples are consequences of the visible control flow under stated preconditions; they are not reported observations.
