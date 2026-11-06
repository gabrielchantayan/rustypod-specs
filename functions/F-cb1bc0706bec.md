<!-- Trusted provenance: F-cb1bc0706bec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cb1bc0706bec

## Purpose

F-cb1bc0706bec conditionally performs two checks and a sequence of handle-associated operations for an object described by caller-provided storage.

The established behavior is a sequence of memory reads, dependency calls, and continuation gates. The routine resolves an entry index, captures the first word of the selected entry, performs up to two checks, obtains a handle, and, if all gates permit, performs eight further calls in a fixed order. It has no explicit return value.

The strings `sqlite_master`, `sqlite_temp_master`, and `trigger` suggest trigger-related schema processing. That is **inferred intent**, not an established database effect. The supplied source does not establish whether the routine creates, removes, changes, or records an object, or whether any generated operations execute during this invocation.

One operand in the final sequence comes from **global storage read when a particular call’s arguments are prepared**. It is not established to be constant, immutable, or captured at routine entry. Earlier dependencies may change that storage.

## Interface

The routine accepts four arguments:

| Argument | Established representation and use |
|---|---|
| Context reference | Address of storage whose first 32-bit word is read as a context-associated base address. The reference itself is also passed to dependencies. |
| Descriptor reference | Address of word-addressed storage. The routine directly reads word 0 and word 7 and passes the reference to dependencies. |
| Auxiliary value A | Opaque 32-bit value, supplied twice to the entry-resolution dependency. |
| Auxiliary value B | Opaque 32-bit value, supplied twice to the entry-resolution dependency. |

No signedness, ownership rule, or pointer interpretation is established for the auxiliary values.

The directly visible layout requirements are:

| Storage | Access |
|---|---|
| Context reference | First 32-bit word, at byte offset 0. |
| Captured context-associated base | A 32-bit address-bearing word at byte offset 8. |
| Entry storage reached through that word | Entries selected with a 24-byte stride; the first 32-bit word of the selected entry is read. |
| Descriptor | Word 0 at byte offset 0; word 7 at byte offset 28. |
| Address returned by F-3a33b0e910ba | First 32-bit word. |
| Global operand storage | Stored value read for the third argument of F-d92e1bb371e9. Its declaration and load width are not supplied. |

These access requirements are not complete structure definitions. In particular, a 24-byte stride does not establish the meanings or types of the remaining entry fields.

Normal completion requires every address actually dereferenced to be valid at the time of access. Validity at entry alone is insufficient: dependencies run between several reads and may alter or release storage. The routine contains no explicit pointer, extent, lifetime, index-range, or arithmetic-validity checks.

The interface exposes no explicit status result. Zero and nonzero dependency results determine which later calls occur; they are not returned to the caller.

## Behavior

### Initial capture and entry resolution

The routine first reads the initial word of the context reference and retains it as the context-associated base.

It then invokes F-49f750c2c596 with seven arguments, in this order:

1. The captured context-associated base.
2. The descriptor’s word 7.
3. Auxiliary value A.
4. Auxiliary value B.
5. The descriptor reference.
6. Auxiliary value A again.
7. Auxiliary value B again.

The result is retained as the selected-entry index. The routine performs no explicit rejection of a negative result, special error sentinel, or excessive index.

The descriptor’s word 7 is read while preparing this call. The source does not establish a broader semantic meaning for that word.

### Word-source acquisition and entry-word capture

Next, F-3a33b0e910ba receives the descriptor reference. Its result is retained as an address for a later word read; the pointed-to word is not captured at this stage.

After that dependency returns, the routine reads the address-bearing word at byte offset 8 of the previously captured context-associated base. It uses the selected index and a 24-byte stride to locate an entry, then captures that entry’s first 32-bit word.

The timing establishes several distinct rules:

- The context-associated base is captured before either dependency.
- The entry-storage address is read after both dependencies.
- Changing the first word of the context reference during those dependencies does not replace the retained base.
- Changing the word at offset 8 of the retained base can change which storage is subsequently accessed.
- The selected-entry word is captured once and reused for both checks.

An index other than 1 still participates in this address calculation. The later index comparison is not a validation gate for the entry read.

### Index-dependent check arguments

Exactly one selected-index value receives special treatment:

| Selected index | First-check tag | Schema-name argument for the second check |
|---|---:|---|
| Exactly 1 | 14 | `sqlite_temp_master` |
| Any other value | 16 | `sqlite_master` |

The observable rule is equality with 1. An interpretation involving temporary and other schemas is plausible but does not establish additional index constraints.

### First continuation check

F-0055bebe0706 receives:

1. The context reference.
2. The selected tag, 14 or 16.
3. The current descriptor word 0.
4. The current first word at the address returned by F-3a33b0e910ba.
5. The captured selected-entry word.

The descriptor word and helper-provided word are read while preparing this call. The supplied source does not establish their relative evaluation order. Neither is an entry-time snapshot.

A nonzero result ends the routine without the second check, handle acquisition, or final sequence. Zero permits the second check.

Calling this an authorization check is **inferred intent**. Its established role is a zero/nonzero continuation gate.

### Second continuation check

If the first check returns zero, F-0055bebe0706 is called again with:

1. The context reference.
2. Tag 9.
3. The selected schema-name string.
4. Zero.
5. The previously captured selected-entry word.

A nonzero result ends the routine. Zero permits handle acquisition.

The selected-entry word is not refreshed after the first check. Changes to the underlying entry therefore do not change the fifth argument of the second check.

### Handle acquisition

After both checks return zero, F-9b3ea8470521 receives the context reference.

A zero result ends the routine without any final-sequence calls. A nonzero result is retained as the handle for the handle-associated operations. The routine performs no additional validity check on it.

The meaning of zero is unknown. The source does not distinguish allocation failure, unavailable state, an existing error, or another condition.

### Final sequence and global-operand read

For a nonzero handle, these calls occur in order:

| Order | Dependency | Arguments |
|---:|---|---|
| 1 | F-274b53956040 | Context reference; zero; selected index. |
| 2 | F-d5cde6301672 | Context reference; selected index. |
| 3 | F-d92e1bb371e9 | Handle; tag 9; the value then read from global operand storage. |
| 4 | F-2b905b6e3991 | Handle; result of call 3 plus 1; current descriptor word 0; zero. |
| 5 | F-2b905b6e3991 | Handle; result of call 3 plus 4; string `trigger`; 32-bit value `0xfffffffe`. |
| 6 | F-2e31112ab290 | Context reference; selected index. |
| 7 | F-2ea344827b50 | Handle; tag 32; zero; zero; zero. |
| 8 | F-0b0cd8d831b7 | Handle; tag 57; selected index; zero; zero; current descriptor word 0; zero. |

The global operand is read **after F-274b53956040 and F-d5cde6301672 return**, while preparing the arguments of F-d92e1bb371e9. The value passed is the contents of that storage at this point, not the storage’s address, an address literal, or a previously captured value.

Earlier dependencies may change the global operand. No supplied dependency contract establishes that they preserve it. The routine does not inspect the operand’s contents or branch on its value before passing it onward.

Both arithmetic positions in calls 4 and 5 derive from the result of F-d92e1bb371e9. The second position does not derive from either F-2b905b6e3991 result.

Descriptor word 0 is read separately for calls 4 and 8. Together with the first-check read, these are three distinct reads separated by dependency calls. Their values need not agree.

There are no local result-dependent gates within this eight-call sequence. Only the result of call 3 is visibly used in subsequent argument construction. This does not guarantee that every call returns normally: dependency failures or control transfers are outside the supplied behavior.

### Widths and arithmetic

The visible interface uses 32-bit words and address-bearing values. The selected index and the result used for the two position calculations are represented as 32-bit integers in the supplied source model.

The relevant calculations are:

- Selected-index displacement using a 24-byte stride.
- Addition of that displacement to the entry-storage address.
- Addition of 1 and 4 to the result of F-d92e1bb371e9.

No checked arithmetic, saturation, overflow branch, or bounds enforcement is visible. Portable behavior for overflowing integer calculations or invalid resulting addresses is not established. Defined normal-operation examples therefore require valid address calculations and representable position results.

The literal `0xfffffffe` is a 32-bit value whose signed two’s-complement interpretation is −2. Its recipient’s interpretation is unknown.

The global operand’s declaration is absent. Its precise storage width, signedness, and any call-boundary conversion remain unresolved; the 32-bit widths of other operands do not independently establish those properties.

## Side effects

The routine directly reads context-associated storage, descriptor storage, the helper-provided word source, entry storage, and the global operand storage. No explicit direct stores to those locations are visible.

Dependencies can have additional effects. The supplied evidence does not establish that any dependency:

- Preserves global storage, including the operand later read for F-d92e1bb371e9.
- Preserves the context, descriptor, entry-storage address, or entry contents.
- Leaves referenced storage allocated and accessible.
- Only records operations rather than executing work.
- Acquires, retains, transfers, or releases ownership of any argument.

The handle and helper-provided address are not explicitly released by this routine. That observation establishes neither a leak nor ownership retention: acquisition and release responsibilities are unknown.

There is no explicit rollback, cleanup path, diagnostic emission, exception handling, or error-code propagation. Ending at a continuation gate prevents later calls but does not undo effects already produced.

If execution ends before the final sequence, the global operand read described above is not reached. If the final sequence is reached normally, that read occurs only after its first two calls.

## Dependencies

| Opaque ID | Established call-site role |
|---|---|
| F-49f750c2c596 | Produces the selected index from seven arguments. |
| F-3a33b0e910ba | Produces an address whose first word is read for the first check. |
| F-0055bebe0706 | Supplies the two zero/nonzero continuation gates. |
| F-9b3ea8470521 | Supplies the zero/nonzero handle gate. |
| F-274b53956040 | Receives context, zero, and selected index before the global-operand read. |
| F-d5cde6301672 | Receives context and selected index before the global-operand read. |
| F-d92e1bb371e9 | Receives the argument-time global value and produces the base for two position arguments. |
| F-2b905b6e3991 | Receives two separately positioned operands, one using descriptor word 0 and one using `trigger`. |
| F-2e31112ab290 | Receives context and selected index after those operand calls. |
| F-2ea344827b50 | Receives the handle, tag 32, and three zeros. |
| F-0b0cd8d831b7 | Receives the handle, tag 57, selected index, and a fresh descriptor-word-0 value among its arguments. |

These roles specify visible consumption and control flow, not the internal behavior or ownership contracts of the dependencies.

## Acceptance examples

These are conditional static examples, not executed tests. Unless stated otherwise, they assume valid storage, normally returning dependencies, and valid, representable arithmetic.

### Index 1 with all gates permitting continuation

If the resolver returns 1, both checks return zero, and handle acquisition returns nonzero:

- The selected-entry word is captured at a 24-byte displacement.
- The first check receives tag 14.
- The second check receives tag 9 and `sqlite_temp_master`.
- All eight final calls occur in the documented order.
- The global operand is read after the first two final calls return.

### Index 0 with all gates permitting continuation

If the resolver returns 0 under equivalent conditions, the entry-word displacement is zero. The first check receives tag 16, and the second receives `sqlite_master`. Index 0 is not locally treated as an error.

### First or second check stops continuation

If the first check returns nonzero, the resolver, word-source acquisition, entry-word capture, and first-check argument reads have already occurred. The second check and all later calls are skipped.

If only the second check returns nonzero, both checks occur, but handle acquisition and the final sequence are skipped.

In either case, the final-sequence global operand read is not performed.

### Handle acquisition returns zero

If both checks return zero but handle acquisition returns zero, none of the eight final calls occurs. The global operand is not read for F-d92e1bb371e9, and no explicit cleanup call is made.

### Global operand changes before consumption

Suppose global operand storage initially contains value A. F-274b53956040 changes it to B, and F-d5cde6301672 subsequently changes it to C.

If execution reaches F-d92e1bb371e9 and no further intervening change occurs, its third argument is C. Capturing A at routine entry, retaining B before the second call, or passing the storage address would not preserve the documented behavior.

### Descriptor changes between consumers

Suppose descriptor word 0 contains A during first-check preparation, B during preparation of the first F-2b905b6e3991 call, and C during preparation of the final call.

Those consumers receive A, B, and C respectively. A single cached descriptor value would not preserve these reads.

### Position-producing dependency returns 100

If F-d92e1bb371e9 returns 100, the two F-2b905b6e3991 position arguments are 101 and 104. The latter call also receives `trigger` and `0xfffffffe`, irrespective of the first operand call’s return value.

## Uncertainties

The following remain **documented_only**:

- Complete argument types, structure definitions, and ABI details.
- Valid index ranges and any resolver error sentinels.
- Semantic meanings of the selected-entry word and helper-provided word.
- The global operand’s initial value, declaration, width, signedness, interpretation, and call-boundary conversions.
- Which dependencies modify that global storage, if any. Preservation is not established.
- Meanings of tags 9, 14, 16, 32, and 57.
- Whether zero or `0xfffffffe` expresses a length, ownership rule, operand mode, or another convention.
- Dependency allocation, ownership, release, diagnostics, global effects, and error behavior.
- Behavior for invalid pointers, invalid indices, arithmetic overflow, or dependency failure to return normally.
- Whether handle-associated operations execute immediately, later, or at all.

The global operand’s **read timing and storage-backed nature are established**. Its value and semantic interpretation are not. The schema-related strings support inferred intent without resolving these unknowns.

## Evidence

Established static evidence consists of the supplied routine’s explicit memory accesses, argument construction, comparisons, arithmetic, conditional continuation, and dependency-call order. The opaque dependency mapping identifies call targets but provides no implementation guarantees for them.

In particular, the source shows a global-storage value used directly as the third argument of F-d92e1bb371e9, after the two preceding final-sequence calls. It does not show an earlier capture of that value or establish immutability.

No runtime tests have been performed. No dependency implementations, global declaration, ABI specification, or execution trace were supplied. Database effects, ownership rules, and behavior beyond the visible gates therefore remain documented unknowns rather than assumed guarantees.
