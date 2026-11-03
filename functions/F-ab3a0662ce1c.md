<!-- Trusted provenance: F-ab3a0662ce1c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ab3a0662ce1c

## Purpose

F-ab3a0662ce1c chooses between a stored 32-bit result and the result of an indirect operation on a supplied object.

The stored-result path requires both:

- A first indirect operation returns a nonzero status.
- A 32-bit state field equals the value of a temporary after that operation.

When both conditions hold, the function clears the state field and returns the adjacent stored result. Otherwise, it calls a second indirect operation and uses that operation’s zero/nonzero result to choose between two further indirect operations. The selected operation supplies the return value.

This is an established description of the supplied static evidence. Interpreting the state field as a token, the adjacent word as a cached result, or the indirect operations as queries or lifecycle operations would be inferred intent, not an established contract.

## Interface

### Inputs

The function has four argument positions:

| Position | Contract |
|---|---|
| First | A base address for state accessed at byte offsets `0x28` and `0x2c`. It is accessed directly only on paths where the first indirect operation returns nonzero. |
| Second | An object address whose first word supplies an indirect-call table address. Every indirect call receives this object address as its first argument. |
| Third | An uninterpreted 32-bit argument. The supplied body does not read, forward, store, or otherwise use it. |
| Fourth | A 32-bit integer value used to initialize a temporary passed by address to the first indirect operation. |

The supplied representation uses 32-bit integer and address-bearing values. This contract describes that recovered data model; the original language declarations, calling convention, and ABI are not independently established.

### Return value

The return value is one 32-bit word. It comes from exactly one of these sources:

- The state word at offset `0x2c`, read after clearing the word at offset `0x28`.
- The return value of one of two selected indirect operations.

No return bits are interpreted, normalized, or converted by the visible body. In particular, a zero result is permitted on either path and is not assigned an error meaning.

### State layout

The directly accessed state is:

| Byte offset from first argument | Width | Use |
|---|---:|---|
| `0x28` | 32 bits | Compared with the temporary; overwritten with zero on the stored-result path. |
| `0x2c` | 32 bits | Read as the return value on the stored-result path. |

These are distinct adjacent words. No access to any other state field is established.

### Indirect-call layout

The second argument addresses an object whose first word is treated as the address of a call table. Table offsets are byte offsets, not entry indices.

The following opaque IDs are document-local labels for unresolved indirect targets. They do not establish identities outside this document.

| Dependency ID | Table byte offset | Visible invocation |
|---|---:|---|
| F-01 | `0x168` | Object address and address of a mutable 32-bit temporary; returns a status compared with zero. |
| F-02 | `0xe8` | Object address; returns a selector compared with zero. |
| F-03 | `0x94` | Object address; returns the function’s 32-bit result when F-02 returns zero. |
| F-04 | `0x90` | Object address; returns the function’s 32-bit result when F-02 returns nonzero. |

The targets are resolved dynamically. The supplied evidence does not prove that a given slot retains the same target across calls.

## Behavior

### Initial operation

The function initializes a local 32-bit temporary from the fourth argument. It then resolves F-01 through the second argument’s current call table and invokes it with:

- The second argument’s object address.
- The address of the initialized temporary.

F-01 may modify the temporary. Subsequent comparison uses its value after F-01 returns, not necessarily the original fourth argument.

The visible body places no restrictions on other effects of F-01. In particular, it does not establish that F-01 preserves the state object, the call-table pointer, global state, or memory ownership.

### Stored-result eligibility

After F-01 returns normally, the function first tests its status against zero.

If that status is zero, the function takes the fallback path without reading the state word at offset `0x28` for the eligibility comparison. A temporary value matching that state word does not override a zero status.

If the status is nonzero, the function reads the 32-bit word at offset `0x28` from the first argument and compares it with the temporary’s post-call value.

- Equal values select the stored-result path.
- Unequal values select the fallback path.

The condition uses equality only. It performs no arithmetic, range check, positivity check, or special treatment of negative integer values.

### Stored-result path

On this path, the direct state operations occur in this order:

1. Write a complete 32-bit zero to offset `0x28`.
2. Read the complete 32-bit word at offset `0x2c`.
3. Return that word.

The stored result is therefore read after the clearing write, not prefetched before it. The visible body does not clear or otherwise overwrite offset `0x2c`.

Neither F-02 nor either final-result dependency is invoked on this path.

Clearing the comparison field does not, by itself, guarantee one-time consumption. For example, if a later F-01 invocation returns nonzero with a temporary value of zero, the already-cleared state field can match again. A single-use guarantee would require an additional invariant not present in the supplied evidence.

### Fallback path

The fallback path is entered when either:

- F-01 returns zero; or
- F-01 returns nonzero and the state word differs from the post-call temporary.

The function resolves and calls F-02 through the object’s then-current call table.

After F-02 returns normally:

- A zero selector causes resolution and invocation of F-03.
- Any nonzero selector causes resolution and invocation of F-04.

The selected final dependency receives only the object address. Its 32-bit return value becomes the return value of F-ab3a0662ce1c.

The final target is resolved after F-02 returns. Consequently, changes made by F-02 to the object’s call-table pointer or table contents can affect which target is invoked. The contract must not substitute a table captured before F-02.

Likewise, the table used to resolve F-02 is obtained after F-01 returns. Effects of F-01 can therefore influence subsequent dispatch.

The fallback path contains no direct write to either documented state word and no direct read of offset `0x2c`. Dependencies may nevertheless access or modify those locations.

## Side effects

### Direct effects

The only visible write to caller-addressed state is the 32-bit zero written at offset `0x28` on the stored-result path.

The function also initializes its temporary and exposes that temporary’s address to F-01. No explicit allocation, deallocation, resource acquisition, or resource release appears in the visible body.

This does not establish that the indirect operations avoid those effects.

### Ownership and lifetime

No ownership transfer is explicitly represented for either address argument or for the stored result. A returned word is not established to be a pointer, handle, count, status, or owned resource.

The temporary is local to this invocation. Its address is supplied during the F-01 call, but the visible body provides no lasting storage or lifetime extension. Whether F-01 retains that address is unknown; the body does not establish safe use after the invocation ends.

### Preconditions and failures

There are no explicit checks for null addresses, accessible memory, table bounds, valid call targets, alignment, or compatible calling conventions.

For normal completion, the memory and targets actually used on the selected path must support the indicated accesses and calls. In particular:

- The second argument must support initial table resolution and invocation of F-01.
- The state comparison requires a readable word at offset `0x28` only when F-01 returns nonzero.
- The stored-result path additionally requires that word to be writable and offset `0x2c` to be readable.
- The fallback path requires valid resolution and invocation of F-02 and the selected final dependency.

No controlled error result is specified for invalid memory or invalid dispatch targets. No exception handling, fault recovery, or retry is visible. If a dependency does not return normally, this contract establishes no subsequent actions.

No synchronization is visible. Atomicity, cross-thread visibility, and safe concurrent mutation are not established.

## Dependencies

### F-01

This dependency is always invoked first. Its return value gates whether the state comparison occurs. Its possible modification of the temporary determines the comparison operand.

Unknowns include its semantic purpose, permitted output values, effects on other memory, and ownership behavior. Its zero result means only “take the fallback path” within this function.

### F-02

This dependency runs only on the fallback path. Its return value selects a table slot: zero selects F-03; every nonzero value selects F-04.

There is no evidence that this selector is a Boolean type or that nonzero values have distinct meanings here.

### F-03 and F-04

Exactly one is invoked after a normally returning F-02. The selected dependency supplies the returned 32-bit word without visible post-processing.

Their names, purposes, relationship, side effects, and memory-management behavior are unknown. Slot placement alone does not establish that either operation destroys, releases, copies, or retrieves anything.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Each assumes valid required memory and normally returning dependencies.

1. **Nonzero status and matching value.**  
   F-01 returns nonzero and leaves the temporary as `7`. The subsequent state read yields `7`. If offset `0x2c` contains `0xa1b2c3d4`, the function clears offset `0x28`, then reads and returns `0xa1b2c3d4`. No fallback dependency is called.

2. **F-01 replaces the initial value.**  
   The fourth argument is `7`, but F-01 changes the temporary to `9` and returns nonzero. A state word of `9` selects the stored-result path. A state word of `7` does not.

3. **Zero status overrides a possible match.**  
   F-01 returns zero. The state field is not read for eligibility, even if it would equal the temporary. F-02 is invoked.

4. **Nonzero status with mismatch.**  
   F-01 returns nonzero, the temporary is `3`, and the state word is `4`. The function takes the fallback path and does not directly clear the state word.

5. **Zero fallback selector.**  
   F-02 returns zero. The function resolves F-03 from the then-current table and returns its result.

6. **Nonzero fallback selector.**  
   F-02 returns any nonzero value, including a negative 32-bit integer value. The function resolves F-04 and returns its result.

7. **Zero matches zero.**  
   F-01 returns nonzero and leaves the temporary as zero; the state word is also zero. The stored-result path is taken. Zero is not treated as an invalid matching value.

8. **Dispatch changes during a dependency.**  
   If F-02 changes the object’s table pointer before returning, final-target resolution uses that changed pointer. This is conditional on the changed object and table remaining valid.

9. **Unused third argument.**  
   Changing only the third argument does not change any direct operation in the supplied body. No broader claim is made about behavior outside this function.

## Uncertainties

The following remain **documented_only**:

- Original types, ABI, byte order, and alignment requirements.
- Semantic identities of the indirect targets.
- Whether the stored result represents a resource or carries ownership.
- Whether dependencies preserve globals, state fields, object layout, or memory validity.
- Whether the two address arguments may safely alias.
- Whether external invariants prevent repeated stored-result returns after clearing.
- Behavior under concurrent mutation, invalid accesses, abnormal dependency termination, or incompatible targets.

An inferred intent is that the first operation establishes whether a stored result is applicable. That interpretation is not needed to specify the observable branch and access behavior, and it does not justify additional validation or lifecycle guarantees.

## Evidence

The supplied static body establishes the argument uses, 32-bit data operations, state offsets, indirect-call offsets, branch conditions, direct access ordering, and return-value sources described above.

The dependency IDs in this document are local labels for unresolved dispatch targets, not recovered symbols or verified identities. Comments, external documents, and semantic naming are not needed for the established behavior.

No runtime tests have been performed. The acceptance examples are static consequences under their stated preconditions, not observations of an executing implementation.
