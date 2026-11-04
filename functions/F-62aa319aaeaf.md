<!-- Trusted provenance: F-62aa319aaeaf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-62aa319aaeaf

## Purpose

F-62aa319aaeaf prepares invocation-local storage, selects an object through an indirect reference, and determines whether to invoke that object’s callback.

Eligibility is established by either a matching identifier in a bounded, zero-terminated list or a nonzero dispatch flag. A match permits dispatch even when the flag is zero. If neither condition permits dispatch, the function reports reason `0x6e`. If dispatch is permitted but the callback address is zero, it reports reason `0x68`.

A callback invocation supplies the function’s 32-bit result unchanged. A diagnostic path returns zero after the diagnostic dependency returns normally.

These are static control-flow properties. The application-level meanings of the object, identifiers, flag, prepared storage, and diagnostic constants are not established.

## Interface

### Input roles and calling convention

The supplied representation declares no explicit parameters, but execution depends on inherited register-associated values and stack-associated storage. It does not establish a complete external calling convention or a safely callable parameterless interface.

The required input roles are:

| Role | Representation | Use |
|---|---|---|
| Identifier reference | Address of a 32-bit cell | Supplies the value compared with nonzero list entries |
| Dispatch flag | 32-bit integer | Permits dispatch after an unsuccessful scan when nonzero |
| Object-holder reference | Address of a 32-bit cell containing an object address | Selects the object after preparation |
| Value slot | Addressable 32-bit stack-associated cell with a represented incoming value | Passed by address during preparation; later supplies the callback’s third argument |
| Payload slot | Addressable invocation-local storage | Passed by address during preparation and as the callback’s second argument |
| Workspace | Addressable invocation-local storage | Passed to both preparation dependencies |

The representation uses 32-bit object words and a 32-bit object-address value. The exact external ABI, caller setup, alignment requirements, and complete frame layout remain **documented_only**.

The dispatch flag is tested only for zero versus nonzero. Its signed interpretation does not affect the decision.

### Invocation-local layout

The addresses exposed to preparation have these relative positions:

| Location | Byte displacement from workspace address |
|---|---:|
| Workspace | `0` |
| Value slot | `16` |
| Payload slot | `20` |

The payload-slot address is therefore four bytes after the value-slot address.

These are relationships between starting addresses, not declarations of independent, nonoverlapping buffer extents. The workspace’s extent is unknown and may reach the other locations. The payload slot’s extent and semantic type are also unknown.

No initial contents are established for the workspace or payload storage. The value slot has a represented incoming 32-bit value, but preparation receives its address and may change it. The callback receives the payload-slot **address**, not a word loaded from that address.

### Object layout

The selected object is interpreted through these fields:

| Byte offset | Access width | Role |
|---:|---:|---|
| `0` | 32 bits | Callback’s first argument |
| `36` | 32 bits | Callback address; zero means absent |
| `44` | 32 bits | First identifier-list entry |
| `48` | 32 bits | Second identifier-list entry |
| `52` | 32 bits | Third identifier-list entry |
| `56` | 32 bits | Fourth identifier-list entry |

No meaning is established for intervening storage.

The list contains at most four entries relevant to this operation. Zero terminates inspection before identifier comparison, so zero cannot itself establish a match.

### Output

The normal result is a 32-bit value:

- Callback path: the callback’s result, unchanged.
- Diagnostic path: zero, after the diagnostic dependency returns.

Zero is not an exclusive failure indicator because the callback may also return zero.

No normal result is established when a dependency or callback fails to return normally.

## Behavior

### Preparation and object selection

The first observable call is to F-0b1e8b902739, with these arguments in order:

1. Workspace address.
2. Payload-slot address.
3. Value-slot address.

After that call returns, F-990a2f44da95 is called with the same workspace address.

Neither return value controls a local branch. No local preparation-status check, retry, or alternate preparation path is shown.

Only after both calls return does the function read the object-holder cell and interpret its 32-bit contents as the object address. The function therefore selects the object using the holder’s contents at that later point, not a value captured before preparation.

Preparation may affect subsequently observed inputs through memory or other state. Its write extents and global effects are unknown. In particular, the second preparation dependency is not established to preserve storage produced by the first.

### Bounded identifier inspection

The function begins at object byte offset `44` and considers successive 32-bit entries in increasing address order.

For each of the four possible positions:

- A zero entry ends inspection without a match at that position.
- Otherwise, the entry is compared for equality with the 32-bit value reached through the identifier reference.
- Equality immediately establishes eligibility and prevents inspection of later positions.
- Inequality advances inspection, unless the fourth position has already been considered.

The function does not inspect a fifth entry. With four nonzero mismatches, the last accessed list word begins at offset `56` and extends through offset `59`.

The identifier reference is used for comparisons against nonzero entries; the representation does not establish an identifier snapshot taken before the scan. If the first entry is zero, no identifier-reference dereference is needed by this scan.

The comparison is 32-bit equality, not signed ordering. Nonzero values with the high bit set can match. A requested identifier of zero cannot match any entry: zero entries terminate, and nonzero entries differ from zero.

The scan index remains within the four candidate positions. Its increment does not approach the limit of a 32-bit integer. Object-relative accesses nevertheless require valid address formation and accessible storage; the function supplies no recovery for invalid addressing.

### Eligibility and flag precedence

A match immediately enters the callback-eligible path without testing the dispatch flag.

If inspection ends without a match:

- A zero flag selects diagnostic reason `0x6e`.
- Any nonzero flag enters the callback-eligible path.

The list is inspected before the fallback flag decision. A nonzero flag does not bypass list accesses or the identifier comparisons those accesses require.

An unmatched request with a zero flag is rejected before callback presence is checked. Consequently, reason `0x6e` takes precedence over a simultaneously absent callback.

### Callback presence and invocation

On the eligible path, the function explicitly checks whether the 32-bit callback address at object offset `36` is zero.

- Zero selects diagnostic reason `0x68`.
- Nonzero leads to an indirect callback invocation.

The callback receives three arguments:

| Position | Supplied value |
|---:|---|
| 1 | The 32-bit word at object offset `0` |
| 2 | The payload-slot address |
| 3 | The 32-bit contents of the value slot at argument evaluation |

The first argument is passed as a word. This function does not dereference it or establish whether it represents a pointer, handle, integer, or another application value.

The third argument is not necessarily the incoming value: preparation can change the addressed slot. The second argument remains the storage address regardless of the contents placed there.

The callback’s 32-bit result is returned immediately. No subsequent diagnostic call or explicit cleanup call is shown.

The supplied representation uses the callback field both for the presence check and for the indirect invocation. It does not establish an atomic, immutable callback snapshot. Exact machine-level load counts and ordering among callback-argument evaluations remain unresolved.

### Diagnostic reporting

Both diagnostic branches call F-4fc89ec41a6d with three scalar arguments:

1. `6`
2. `0x6b`
3. Either `0x6e` or `0x68`, as selected above

The diagnostic dependency’s result is ignored. If it returns normally, F-62aa319aaeaf returns zero.

The constants distinguish branches in this function. Their external names, severity meanings, formatting, and destination are not established.

## Side effects

### Established reads and calls

The function exposes invocation-local addresses to the preparation dependencies before inspecting the object. It then:

- Reads the object-holder cell.
- Reads a prefix of the four-entry identifier list.
- Reads the identifier cell when a nonzero entry requires comparison.
- Checks the callback field only on an eligible path.
- Reads the object’s first word and the value slot when invoking the callback.
- Calls either the callback or the diagnostic dependency after preparation, provided execution reaches that stage.

The list prefix ends at the first zero, the first match, or the fourth entry. Later entries are not required on an earlier terminating path.

No direct write to the selected object, object-holder cell, or identifier cell is shown in the function’s own logic. This does not imply that dependencies or the callback leave those locations unchanged.

### Validation and error boundaries

The callback address receives an explicit null check on the eligible path. An absent callback selects reason `0x68`; it is not invoked.

By contrast, the object-holder address, the selected object address, and the identifier-reference address receive no explicit null checks. The function also shows no explicit bounds or alignment validation. A nonzero callback address receives no executable-address validation.

Normal completion requires valid storage for every reached access, including:

- The local regions accessed by preparation dependencies.
- The object-holder cell.
- Each inspected object-list entry.
- The identifier cell whenever comparison is reached.
- The callback field when dispatch is eligible.
- The object’s first word when the callback is invoked.
- A callable nonzero callback address when invocation occurs.

These requirements are path-dependent. An immediate zero terminator avoids identifier-cell reads. Rejection for no match and a zero flag avoids the callback-field check.

Invalid pointers, inaccessible storage, unsuitable alignment, and non-callable nonzero callback addresses have no established recovery behavior. They are not shown being translated into either diagnostic reason.

### Ownership and lifetime

No explicit allocation, deallocation, ownership transfer, or reference-count operation appears in this function’s own logic.

Passing addresses does not establish ownership transfer. The dependencies may allocate, retain, release, or mutate other state, but their implementations are unavailable. Their names and call positions do not establish cleanup or preservation guarantees.

The payload-slot address is exposed to the callback. No permission to retain that invocation-local address beyond the invocation is established. Any such lifetime rule belongs to an external contract and is not enforced here.

No explicit cleanup follows callback dispatch or diagnostic reporting. This observation does not establish whether cleanup already occurred inside preparation or whether additional cleanup is required externally.

## Dependencies

| Opaque dependency | Established use | Unresolved behavior |
|---|---|---|
| F-0b1e8b902739 | First call; receives workspace, payload-slot, and value-slot addresses | Initialization, write extents, ownership, retention, global effects, failure behavior |
| F-990a2f44da95 | Second call; receives the workspace address | Transformation or release of state, effects on adjacent storage and globals, failure behavior |
| F-4fc89ec41a6d | Diagnostic call with `6`, `0x6b`, and the selected reason | Reporting destination, formatting, termination, ownership, other side effects |
| Object-selected callback | Eligible, nonzero-address invocation; receives three arguments and supplies the returned result | Semantic argument types, mutation, ownership, retention, failure behavior |

The object-selected callback has no fixed opaque function ID in the supplied dependency mapping.

No dependency is assumed to preserve globals, preserve adjacent storage, or own or release memory.

## Acceptance examples

These are conditional consequences of the static representation, not executed tests. Unless a case states otherwise, preparation returns normally, all reached accesses are valid, and relevant memory remains stable during the described operations.

### A match overrides a zero flag

With requested identifier `17`, entries beginning `9, 17`, a zero flag, and a callable nonzero callback:

- Preparation completes before object inspection.
- The second entry establishes eligibility.
- Later entries are not inspected.
- The callback receives the object’s first word, the payload-slot address, and the current value-slot contents.
- Its result is returned unchanged, including when that result is zero.
- No diagnostic call occurs.

### A terminator hides a later matching entry

With requested identifier `17`, entries beginning `0, 17`, and a zero flag:

- The first entry terminates inspection.
- The identifier reference and later entry are not needed by the scan.
- Reason `0x6e` is reported.
- The callback field is not checked.
- The result is zero if reporting returns.

### A nonzero flag permits four mismatches

With four nonzero mismatching entries, any nonzero flag, and a callable callback:

- All four entries are considered.
- No fifth entry is inspected.
- The flag permits dispatch after the unsuccessful scan.
- The callback’s result is returned unchanged.

The flag does not eliminate the scan’s memory-validity requirements.

### An eligible null callback reports its own reason

With either a matching identifier or an unmatched request with a nonzero flag, and a zero callback field:

- The explicit callback null check detects absence.
- No callback is invoked.
- F-4fc89ec41a6d receives `(6, 0x6b, 0x68)`.
- The result is zero if reporting returns.

### Rejection precedes callback validation

With no match, a zero flag, and a zero callback field:

- F-4fc89ec41a6d receives `(6, 0x6b, 0x6e)`.
- Reason `0x68` is not selected.
- The callback-field check is not reached.

### Zero and high-bit identifiers

With requested identifier zero and a zero first entry, inspection terminates without a match.

With requested identifier `0x80000001` and a stable nonzero entry containing `0x80000001`, equality establishes eligibility regardless of any signed interpretation.

### Preparation changes later observations

If preparation changes the object-holder cell to another valid object address, the subsequently selected object is the one referenced after both preparation calls return.

If preparation changes the value slot, the callback receives the value later read from that slot. The payload argument remains the payload-slot address.

### Invalid nonzero callback addresses are not diagnosed as absent

If dispatch is eligible and the callback field contains a nonzero but non-callable address, the zero-address check does not reject it. The representation attempts indirect invocation; the resulting failure behavior is **documented_only**, not specified as reason `0x68`.

## Uncertainties

**Established static evidence:** preparation-call order, local-address relationships, 32-bit field accesses, object-relative offsets, four-entry bound, zero termination, equality matching, flag precedence, explicit callback null check, callback arguments, diagnostic constants, and normal return selection.

**Inferred intent:** the identifier list may express recognition or eligibility, the flag may override an unsuccessful lookup, and preparation may produce callback-consumable state. These interpretations add no requirements.

**Documented_only:**

- Exact public ABI and caller obligations.
- Complete object and invocation-local layouts, endianness, and alignment requirements.
- Workspace and payload extents and semantic types.
- Application meanings of values and diagnostics.
- Dependency mutation, preservation, ownership, retention, and nonreturning behavior.
- Thread safety, reentrancy, and concurrent-mutation semantics.
- Exact instruction-level load counts and argument-evaluation order.
- Failure behavior for invalid accesses or non-callable nonzero addresses.

No runtime tests have been performed.

## Evidence

This contract is derived from the supplied static representation of F-62aa319aaeaf and its opaque dependency mapping.

Four-byte word indexing establishes the listed object offsets. Four candidate positions beginning at offset `44` establish the maximum inspected list extent through offset `59`. The control-flow branches establish match precedence, fallback flag handling, the explicit callback null check, the two diagnostic reasons, and return-value selection.

No dependency implementations, runtime observations, external API declarations, or ownership documentation were supplied. Unresolved behavior is retained explicitly rather than inferred from names or assumed conventions.
