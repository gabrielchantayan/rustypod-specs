<!-- Trusted provenance: F-1b9e6fff9bc9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1b9e6fff9bc9

## Purpose

F-1b9e6fff9bc9 conditionally performs a dependency-mediated state operation, then drains a linked sequence through two dependency calls per element. It returns a 64-bit value combining the original input pointer representation with a 32-bit state value.

The condition is the initial value of a 32-bit field at byte offset 16 in the input record:

- If that field is zero, the function skips all dependency calls and sequence processing. The upper return word comes from the second argument.
- If that field is nonzero, the function prepares three mutable local words, invokes two dependencies, and then processes the sequence reachable through the record’s first word. The upper return word comes from one of those mutable local words.

This describes established control flow. Interpreting the operation as destruction, resource release, or ownership transfer is inferred intent, not an established dependency contract.

## Interface

### Arguments

| Argument | Established use |
|---|---|
| First | Pointer to a mutable record containing 32-bit fields. Its original pointer representation contributes the low 32 bits of the return value. |
| Second | 32-bit scalar used as the upper return word only when the initial field at offset 16 is zero. Although represented as a signed integer in the supplied material, this function performs no signed arithmetic or comparison on it. |
| Third | 32-bit value not explicitly read or forwarded by this function. |
| Fourth | 32-bit value copied into a mutable local word and passed by address to F-e3037e3e9815, but only on the nonzero branch. |

No argument validation is visible.

### Referenced layouts

The input record has these explicitly referenced fields:

| Byte offset | Width | Use |
|---|---:|---|
| 0 | 32 bits | Sequence-head reference; read on every loop condition and overwritten when an element is selected. |
| 16 | 32 bits | Initial branch discriminator and source of subsequent dependency arguments. On the nonzero branch it is also treated as a reference to another object. |

The referenced object reached through offset 16 must support a 32-bit read at byte offset 8 when that read occurs.

Each selected sequence element has these explicitly referenced fields:

| Byte offset | Width | Use |
|---|---:|---|
| 0 | 32 bits | Successor reference copied into the input record’s head field. |
| 4 | 32 bits | Second argument to the first F-ff1bdc376dcd call for that element. |
| 8 | 32 bits | First argument to that call. |

These accesses do not establish complete object layouts, allocation sizes, or the meanings of unused fields.

The supplied representation uses 32-bit words and pointer-valued words. The contract therefore describes a 32-bit address/reference representation. It does not establish portability to wider pointers.

### Return value

The return value is a 64-bit concatenation:

- Bits 31–0: the 32-bit representation of the original first argument.
- Bits 63–32: the selected 32-bit state word.

This is bitwise packing, not a numeric sum, an ordinary pointer return, or an established success/error encoding. In particular, a high word with bit 31 set remains that exact word; it is not sign-extended across the entire result.

### Preconditions

Defined execution requires every actual read and write to address valid storage with the access properties required by the target environment. In particular:

- The input record must support the initial read at offset 16.
- On the nonzero branch, the subsequently loaded reference from offset 16 must support the read at its offset 8.
- After both preliminary dependency calls, the input record must remain accessible for sequence-head reads and any required writes.
- Each selected nonzero element reference must support the fields actually accessed.
- Dependencies must return normally for execution to reach the final packed return.

These are access requirements, not claims that the function checks them.

## Behavior

### Initial branch selection

The function first reads the input record’s 32-bit field at offset 16 and saves that value in a mutable local state word.

If the saved value is zero, the function returns without traversing the sequence. The upper return word is the second argument, and the lower return word is the original input pointer representation.

Consequences of this branch include:

- A nonzero sequence head does not independently trigger processing.
- The fourth argument is not used.
- The third argument remains unused.
- No dependency call occurs.
- No explicit write to the input record occurs.

### Preparation on the nonzero branch

If the initial saved value is nonzero, the function prepares two more local 32-bit words:

1. It reads the record’s field at offset 16 again, treats that newly loaded value as an address/reference, and reads a 32-bit word at byte offset 8 from that referenced object.
2. It copies the fourth argument into another local word.

The second load of the record’s field is distinct from the initial branch-selection load. No synchronization or guarantee of stability between those loads is shown.

The function then invokes F-e3037e3e9815 with four arguments, in this positional order:

1. Address of the local word initialized from the fourth argument.
2. The input record pointer.
3. Address of the local word initialized from the referenced object’s offset 8.
4. Address of the local state word initialized from the initial offset-16 value.

All three local words are therefore exposed to that dependency through addresses. The supplied evidence does not establish whether any are input-only, output-only, or input/output parameters. For the wrapper’s subsequent behavior, the state word matters because it supplies the eventual upper return word. No explicit later use of the other two local words is shown.

After F-e3037e3e9815 returns, the function invokes F-1b2e0c144326 with:

1. The input record pointer.
2. The value currently read from the record’s offset-16 field.
3. A zero-valued scalar.

The second argument is loaded after the preceding dependency call. It is not necessarily the initial offset-16 value: F-e3037e3e9815 may have modified the record.

No return value from either preliminary dependency is explicitly used.

### Sequence processing

Only after both preliminary calls return does the function inspect the sequence head at offset 0.

For each condition check, it reads the current head from the input record and copies the current local state word into the value used for the upper return word. A zero head ends processing.

For each nonzero selected head:

1. The function reads that element’s successor field at offset 0.
2. It writes the successor reference into the input record’s head field.
3. It calls F-ff1bdc376dcd with the element’s offset-8 value as the first argument, its offset-4 value as the second argument, and zero as the third argument.
4. After that call returns, it calls F-ff1bdc376dcd again, this time with the selected element reference as the first argument, one as the second argument, and zero as the third argument.
5. After the second call returns, it repeats the condition check by rereading the input record’s head.

The first dependency call’s two element-field reads both occur after the head update and before that call begins. Their order relative to each other is not established by the supplied expression.

The head update precedes both calls for the selected element. There is no later explicit restoration of that head value.

The next iteration does not rely on a privately cached successor. It rereads the record. Consequently, changes that a dependency makes to the head affect subsequent traversal. A dependency may replace the head, set it to zero, or otherwise alter the sequence visible to the wrapper.

Similarly, the selected element reference used by the second call remains the reference selected for the current iteration, even if the first call changes the record’s head.

### Completion and edge behavior

On the nonzero branch, the second argument does not determine the upper return word. The loop condition copies the mutable local state word even when the first sequence-head read is zero. Thus an initially empty sequence after the preliminary calls still returns that local state word.

There is no explicit:

- Element-count limit.
- Cycle detection.
- Null-input handling.
- Bounds validation.
- Error translation.
- Rollback after a head update.
- Assignment clearing the record’s offset-16 field.

A cyclic sequence is not guaranteed to terminate. Its behavior additionally depends on what F-ff1bdc376dcd does to elements and reachable state. The wrapper alone does not justify treating repeated references as safe.

## Side effects

The wrapper’s explicit externally visible writes are assignments to the input record’s offset-0 head field while processing elements.

It also writes local temporary words and passes their addresses to F-e3037e3e9815. Those addresses expose mutable local storage during the call. The evidence does not establish whether the dependency retains them; no safe post-return lifetime for retained addresses is established.

All three dependencies may have additional effects not specified here. In particular, there is no basis for assuming that they:

- Preserve global state.
- Preserve the input record or sequence.
- Leave the mutable state word unchanged.
- Allocate, release, or retain memory in a particular way.
- Return normally.
- Report failure through a particular mechanism.

The two calls per element resemble a payload operation followed by an element operation. That resemblance does not establish ownership. Whether the first call releases a resource, whether the second releases the element, and whether either call invalidates other reachable storage are documented_only unknowns.

If a dependency fails to return after a head update, the wrapper has already performed that update. No compensating action is visible.

## Dependencies

| Opaque ID | Invocation contract established here |
|---|---|
| F-e3037e3e9815 | Called once on the nonzero branch, before any sequence-head inspection. Receives addresses of three local 32-bit words and the input record pointer in the order specified above. Its explicit return value is unused. |
| F-1b2e0c144326 | Called once after F-e3037e3e9815 returns. Receives the input record, its then-current offset-16 value, and zero. Its explicit return value is unused. |
| F-ff1bdc376dcd | Called twice per selected sequence element. The first call receives the element’s offset-8 and offset-4 values plus zero. The second receives the element reference, one, and zero. Both explicit return values are unused. |

The wrapper does not inspect a dependency result before continuing. This does not establish that the dependencies cannot signal errors through nonlocal control flow, shared state, or other mechanisms.

## Acceptance examples

These are conditional static acceptance examples, not runtime observations.

### Zero discriminator with an existing sequence

Suppose the input record’s offset-16 field is zero, its head is nonzero, and the second argument has the word value `0x89ABCDEF`.

Expected wrapper behavior:

- No dependency calls.
- No sequence-head read or write by the wrapper.
- No processing of the existing sequence.
- Return high word `0x89ABCDEF`; return low word equal to the original input pointer representation.

### State mutation with an empty sequence

Suppose the initial offset-16 value is nonzero and all preparatory accesses are valid. F-e3037e3e9815 changes the local state word to `0x10203040`. After F-1b2e0c144326 returns, the sequence head is zero.

Expected wrapper behavior:

- The two preliminary dependencies are called in order.
- No F-ff1bdc376dcd call occurs.
- The high return word is `0x10203040`, provided the state word is not subsequently changed.

### One element without dependency-induced list changes

Suppose, after the preliminary calls, the head selects one valid element whose successor is zero, whose offset-4 value is seven, and whose offset-8 value is `0x12345678`. Assume the dependencies do not change the head or invalidate required accesses.

Expected wrapper behavior:

- The head is written to zero before either element-related call.
- The first F-ff1bdc376dcd call receives `0x12345678`, seven, and zero.
- The second receives the selected element reference, one, and zero.
- The following condition check reads a zero head and terminates.

### Preliminary mutation of the referenced field

Suppose F-e3037e3e9815 changes the record’s offset-16 field from one valid reference to another.

Expected wrapper behavior:

- The earlier offset-8 read used the reference loaded before that call.
- F-1b2e0c144326 receives the newly read offset-16 value.
- No requirement exists that these two references match.

### Head replacement during element processing

Suppose the wrapper advances the head to a successor, then an element-related dependency replaces the head with another valid element.

Expected wrapper behavior:

- The current iteration still makes its second call using the originally selected element reference.
- The next condition check observes the replacement head.
- The wrapper does not restore the previously written successor.

## Uncertainties

The following remain documented_only:

- Semantic meanings of the record, offset-16 reference, sequence elements, and stored values.
- Dependency ownership and memory-release rules.
- Whether any scalar word is a count, flag, handle, pointer, or status beyond its observed use.
- Dependency behavior on zero or invalid argument values.
- Global-state changes, callbacks, reentrancy, and retained-address behavior.
- Required alignment and exact target fault behavior.
- Concurrency guarantees; none are visible.
- A success/error interpretation for the packed return value.
- Behavior after invalid memory access or dependency non-return.

The supplied material establishes fixed-offset accesses and word widths, but does not establish byte order for an external serialized representation. No serialization contract is inferred.

## Evidence

Established static evidence consists of the supplied function body and the supplied mapping to opaque dependency IDs. It supports the branch condition, access widths and offsets, dependency argument positions, call order, explicit head updates, and 64-bit return packing described above.

The cleanup interpretation is inferred intent only. Dependency semantics, ownership, complete layouts, and failure behavior are not established by the available body.

No runtime tests have been performed.
