<!-- Trusted provenance: F-46c4c03e4abe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-46c4c03e4abe

## Purpose

F-46c4c03e4abe performs a lookup-driven update on a receiver-associated structure, marks a byte in the receiver, and invokes a final indirect operation.

The established behavior has two branches:

- A zero lookup result causes acquisition of a value, preparation of temporary storage, a request for 16 bytes of storage, construction or initialization through a dependency, and submission of the resulting value to the receiver-associated structure.
- A nonzero lookup result causes a different dependency to operate on that structure and the lookup result.

Both branches converge on a one-byte write of value 1 and a final indirect operation.

Descriptions such as “lookup,” “construction,” and “submission” express inferred roles. The supplied static evidence establishes the calls, argument values, offsets, and control flow, but not the application-level meanings of the structures or dependencies.

## Interface

### Inputs

The function accepts four arguments:

| Argument | Established representation | Observable use |
|---|---|---|
| Receiver | Address used with 32-bit word indexing | Supplies fields, embedded structure addresses, and indirect dispatch targets. |
| Payload word | 32-bit value | Initially copied into a local word. The address of that local word is passed to two dependencies on the zero-result branch. |
| Third word | 32-bit value | Copied into local storage; no separate subsequent use is visible. |
| Fourth word | 32-bit value | Copied into local storage; no separate subsequent use is visible. |

The represented layout uses four-byte words and addresses carried in those words. This contract does not establish portability to a different pointer width.

The third and fourth words are shown in local storage following the payload word. Their presence does not establish that they form a public aggregate, nor that a callee receiving the payload word’s address is entitled to access all three words. The extent and format of that temporary object remain undocumented.

### Receiver layout used by this function

All offsets below are byte offsets from the receiver:

| Offset | Access |
|---|---|
| 0 | Read as a dispatch-table address for the final operation. |
| 32 | Read as a 32-bit value supplied to several dependencies. Referred to below as the key. |
| 48 | Start address of an embedded structure passed to dependencies. The word at this address also supplies a dispatch-table address on the zero-result branch. |
| 136 | Read as an address of another dispatch-bearing object. |
| 156 | Written as a single byte with value 1 after either branch completes normally. |

Only these accesses are established. In particular, the byte at offset 156 is not evidence that the surrounding four-byte word is a Boolean or that the other three bytes are modified.

### Output and completion

No meaningful return value is established. The visible normal path ends after the final indirect operation.

There is no explicit error result, validation failure return, allocation-failure branch, or exception handler in the supplied evidence. Normal completion therefore depends on all executed dependencies and memory accesses completing successfully.

### Preconditions

For the executed path, the receiver and all traversed addresses must support the indicated reads, writes, and calls. This includes:

- Readable receiver fields at offsets 0, 32, 48, and 136 as needed.
- A writable byte at receiver offset 156.
- Valid dispatch-table entries at the offsets described below.
- A valid embedded-structure representation beginning at receiver offset 48.
- Dependency-specific preconditions that cannot be recovered from this function alone.

The function contains no visible checks enforcing these requirements.

## Behavior

### Entry preparation and initial indirect operation

The function copies the receiver and the three additional input words into local storage. The receiver copy is later overwritten on the zero-result branch.

Before the lookup, the function performs an indirect operation obtained through the address stored at receiver offset 136:

1. It reads that address from the receiver.
2. It reads the dispatch-table address at the beginning of the referenced object.
3. It obtains the callable entry at dispatch-table byte offset 108.
4. It invokes that entry.

The supplied representation shows no explicit arguments for this call. It does not establish whether an implicit receiver or other arguments are supplied by the calling convention.

This operation precedes the lookup and every branch-specific action. No assumption may be made that it preserves the receiver, its fields, global state, or reachable objects.

### Lookup and branch selection

After the initial indirect operation returns, the function calls F-6372f72ded3a with:

- The address at receiver offset 48.
- The current 32-bit key read from receiver offset 32.

The result is represented as a 32-bit integer. The only interpretation applied by this function is whether it equals zero.

A zero result selects the acquisition-and-submission branch. Every nonzero result selects the alternate branch, including values that would be negative under signed 32-bit interpretation. There is no visible range check or validity check beyond equality with zero.

### Zero-result branch

The following operations occur in order.

**Acquire an intermediate object.** The function again reads the address at receiver offset 136, obtains its dispatch table, and invokes the entry at byte offset 100. The supplied representation again shows no explicit arguments. The result is treated as an address.

**Acquire a value using that object.** The function reads the dispatch-table address at the beginning of the returned object and invokes its entry at byte offset 16. The explicitly represented arguments are:

- The returned object’s address.
- The current key from receiver offset 32.

The call’s result is represented as a 32-bit value. The returned object must support the dispatch-table access; no null check is visible.

**Prepare the payload temporary.** The function calls F-a438503882d6 with:

- The address of the local word initially containing the second input argument.
- The acquired 32-bit value.

That dependency can modify the addressed storage. The subsequent call uses the value in this local word after the dependency returns, not necessarily the original payload input.

**Request storage.** The function calls F-8157a180d901 with the value 16. This is the only established allocation-size-like argument. Its result is forwarded without a visible null or failure check.

**Initialize or construct a result.** The function calls F-1570aa6a9c5b with:

- The result of F-8157a180d901.
- A fresh read of the key at receiver offset 32.
- The current value of the payload local word.

The returned value is stored into a local slot that initially held a copy of the receiver. Consequently, the value subsequently submitted is the result of F-1570aa6a9c5b, not necessarily the original storage address returned by F-8157a180d901.

**Submit the result.** The function reads the dispatch-table address stored at receiver offset 48 and invokes its entry at byte offset 28. Its explicitly represented arguments are:

- The embedded-structure address at receiver offset 48.
- The address of the local slot holding the result from F-1570aa6a9c5b.

Passing the slot’s address allows the callee to inspect or alter that slot. This alone does not establish whether the callee takes ownership of the referenced object, retains the slot address, or copies its contents.

**Process the payload temporary after submission.** The function calls F-ed06b5c03b38 with the address of the payload local word.

Its position suggests a temporary-finalization role, but its actual resource effects are unknown. It is reached only if all preceding zero-branch operations return normally.

### Nonzero-result branch

The function calls F-8a03dc695dc9 with:

- The embedded-structure address at receiver offset 48.
- The nonzero result returned by F-6372f72ded3a.

The function does not itself dereference that result or establish whether it is an identifier, position, pointer, or other token.

None of the zero-branch operations described above is executed on this branch.

### Common completion

After the selected branch returns normally, the function writes the value 1 to the single byte at receiver offset 156.

It then performs a final indirect operation:

- The dispatch-table address is read from receiver offset 0.
- The entry at dispatch-table byte offset 88 is selected.
- The explicitly represented arguments are the receiver and two operands designated by static data references.

The values, representations, and meanings of those two data-referenced operands are not supplied.

The byte write precedes this final operation. A final callee that observes the receiver begins after that write, although it may itself overwrite the byte or otherwise alter state.

The supplied representation includes a qualification that this final indirect transfer was treated as a call. Its exact machine-level transfer form is therefore not established.

## Side effects

The only explicit write to receiver storage performed directly by the function is the one-byte store of 1 at offset 156. It is unconditional after normal branch completion and does not depend on the byte’s prior value.

Additional effects can occur through every dependency and indirect operation. These effects are not bounded by this function’s visible direct writes. In particular:

- The initial indirect operation may affect state used by the later lookup.
- The key is read at multiple call sites rather than cached once for the whole function.
- The address at receiver offset 136 is read separately for the initial operation and the zero-branch acquisition operation.
- The embedded dispatch-table address is obtained after the earlier zero-branch calls.
- The receiver’s final dispatch-table address is obtained after the branch and the byte write.

If a dependency changes these locations and returns normally, later operations use the subsequently read values. The evidence does not establish synchronization or make concurrent mutation safe.

No explicit deallocation of the storage requested from F-8157a180d901 appears in this function. That observation does not establish a leak: construction, submission, or another dependency may manage it. Conversely, no ownership transfer or eventual release is guaranteed by the supplied evidence.

There is no visible rollback. If an executed operation does not return normally, later submission, temporary processing, marking, or final dispatch may not occur.

## Dependencies

| Opaque ID or unresolved operation | Established use |
|---|---|
| F-6372f72ded3a | Receives embedded-structure address and current key; its 32-bit result selects the branch. |
| F-8a03dc695dc9 | Receives embedded-structure address and the nonzero lookup result. |
| F-a438503882d6 | Receives the payload-local address and an acquired 32-bit value. |
| F-8157a180d901 | Receives 16; its result is passed to F-1570aa6a9c5b. |
| F-1570aa6a9c5b | Receives requested storage, current key, and current payload-local value; produces the submitted value. |
| F-ed06b5c03b38 | Receives the payload-local address after submission. |
| Unresolved indirect operation at table offset 108 | Executed before lookup through the object referenced at receiver offset 136. |
| Unresolved indirect operation at table offset 100 | Produces an object on the zero-result branch. |
| Unresolved indirect operation at table offset 16 | Invoked on that returned object with the current key. |
| Unresolved indirect operation at table offset 28 | Receives embedded-structure address and the address of the submitted-value slot. |
| Unresolved indirect operation at table offset 88 | Final receiver operation, after the byte store. |

No opaque identity is available for the indirect targets. Table offsets identify dispatch locations, not globally unique functions.

## Acceptance examples

These examples are conditional static acceptance criteria, not observed executions.

### Nonzero lookup result

Given valid required memory and dispatch targets, and dependencies that return normally, if F-6372f72ded3a returns 7:

- The initial offset-108 operation occurs before the lookup.
- F-8a03dc695dc9 receives the embedded-structure address and 7.
- No zero-branch acquisition, storage request, construction, submission, or payload-finalization operation occurs.
- The byte at receiver offset 156 is written with 1.
- The final offset-88 operation follows that write.

The same branch is required for any other nonzero 32-bit result.

### Zero lookup result

If the lookup returns zero and all required operations return normally, the branch follows this order: object acquisition, value acquisition, payload preparation, storage request of 16, result construction, submission by local-slot address, and payload-temporary processing. Marking and final dispatch follow.

The submission must use the result returned by F-1570aa6a9c5b. Equality with the earlier storage-request result is not required by this function.

### Key changes between dependency calls

If payload preparation changes receiver offset 32 and returns normally, the later F-1570aa6a9c5b call receives the changed key. It must not be specified as receiving an entry-time snapshot.

### Existing marker value

If the byte at receiver offset 156 is already 1, normal completion still includes the explicit store of 1 and the final indirect operation. There is no visible early exit based on that byte.

### Failure-like dependency results

If F-8157a180d901 returns zero, the function forwards zero to F-1570aa6a9c5b without a visible check. Whether execution can then continue safely is documented_only; success, rejection, and recovery must not be invented.

## Uncertainties

The following are documented_only:

- Application-level meanings of the receiver, key, marker, and embedded structure.
- Exact prototypes and implicit arguments of unresolved indirect operations.
- Temporary-object size, representation, and whether the third and fourth input words belong to it.
- Ownership, aliasing, retention, and release behavior across dependencies.
- Failure signaling, exception behavior, and nonlocal control flow.
- Meanings and widths of the final two data-referenced operands.
- Exact machine-level form of the final indirect transfer.
- Thread safety and reentrancy.

The request size of 16 does not prove a 16-byte live object layout. Likewise, a nonzero lookup result does not prove that an existing item is found, and the alternate dependency is not established to delete or replace anything.

## Evidence

This contract is based solely on the supplied static representation. Established evidence comprises the argument widths shown there, four-byte receiver indexing, byte-store width, field and dispatch offsets, explicit call arguments, zero-versus-nonzero branch condition, and visible operation order.

Role descriptions such as lookup, construction, submission, and temporary finalization are inferences, not dependency contracts.

No runtime tests have been performed. No concrete dependency implementation, execution trace, allocation outcome, ownership behavior, or final externally visible application result has been verified.
