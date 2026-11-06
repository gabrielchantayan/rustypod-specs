<!-- Trusted provenance: F-4d5e4ca0e6d5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4d5e4ca0e6d5

## Purpose

F-4d5e4ca0e6d5 selects behavior using a one-byte field in a caller-supplied object.

- States 1 and 2 select different global label-object addresses, pass the selected address to F-9b9c1c7483f4, and dispatch its result through the primary object.
- States 3 and 4 obtain another receiver through F-b7af80e829ec and invoke different entries in that receiver’s dispatch table, both with the numeric argument 3.
- Every other state causes a one-byte zero write to the selector field.

The established behavior is address selection, dependency invocation, indirect dispatch, and unsupported-state clearing. Navigation-related meaning is inferred from the names associated with the two global label objects. Their definitions and bytes are not supplied, so neither their exact contents nor their interpretation is established.

This contract is based on static evidence only. No runtime tests have been performed.

## Interface

### Inputs

The function has four explicit inputs:

| Input | Representation in the supplied evidence | Established use |
|---|---|---|
| Primary object | Pointer represented as a pointer to a 32-bit integer | Provides the selector byte and, for states 1 and 2, the dispatch-table reference and receiver. |
| Second argument | Opaque 32-bit value | No explicit use in the visible body. |
| Third argument | Opaque 32-bit value | Copied into invocation-local storage whose address is passed to F-9b9c1c7483f4 for states 1 and 2. |
| Fourth argument | Opaque 32-bit value | Copied into separate invocation-local storage whose address is passed to F-9b9c1c7483f4 and subsequently to F-8a5aadac202f for states 1 and 2. |

The two local storage cells each hold 32 bits. Their contents are not interpreted or arithmetically transformed by this function.

The second argument is not explicitly read or forwarded. That observation does not establish its purpose in a wider interface.

### Accessed layouts

Offsets below are byte displacements, not dispatch-table indices.

| Base | Offset | Access |
|---|---:|---|
| Primary object | 0xBF, decimal 191 | One-byte selector read on every invocation; one-byte zero write for unsupported states. |
| Primary object | 0 | Dispatch-table reference read for states 1 and 2. |
| Primary object’s dispatch table | 0x11C, decimal 284 | Callable-entry read for states 1 and 2. |
| Alternate receiver | 0 | Dispatch-table reference read for states 3 and 4. |
| Alternate receiver’s dispatch table | 0xC0, decimal 192 | Callable-entry read for state 3. |
| Alternate receiver’s dispatch table | 0xC4, decimal 196 | Callable-entry read for state 4. |

The supplied representation uses 32-bit integer storage and converts the primary-object pointer to a 32-bit integer for selector addressing. It is consistent with a 32-bit address model. This is not sufficient to establish a portable ABI, pointer encoding, endianness, or complete object layout.

### Selected global objects

Two separately identified global label objects participate in states 1 and 2:

- The **state-1 label object** has a symbol name suggesting “PopNowPlaying”.
- The **state-2 label object** has a symbol name suggesting “PopToMainMenu”.

These phrases describe apparent associations inferred from symbol names. They are not established string contents.

The function passes the selected global address to F-9b9c1c7483f4. It does not visibly read, compare, copy, or modify bytes at that address. The evidence does not supply either object’s definition, size, contents, encoding, termination, or mutability.

### Output and preconditions

The declared interface has no explicit return value.

F-9b9c1c7483f4 produces an opaque 32-bit result, which is forwarded to the primary-object operation. Results of the indirect operations are not explicitly consumed.

Every invocation requires the primary object’s selector byte to be readable. Additional requirements depend on the selected state:

- Unsupported states require that byte to be writable.
- States 1 and 2 require readable dispatch metadata and a callable entry at table offset 0x11C when the lookup occurs.
- States 3 and 4 require F-b7af80e829ec to supply a receiver supporting the subsequent table accesses and indirect invocation.
- All invoked dependencies must accept the supplied arguments under the actual ABI.

Any requirements imposed by F-9b9c1c7483f4 on the selected global object are unknown. This function’s visible behavior alone does not establish that the object is a readable, terminated string.

There are no visible null checks, bounds checks, pointer checks, or defined recovery paths for invalid memory or invalid call targets.

## Behavior

### Initial selection and local values

The supplied representation first reads the selector byte at primary-object offset 0xBF, then initializes the two local 32-bit cells from the third and fourth arguments.

The captured selector determines the branch. The recognized byte values are exactly 1, 2, 3, and 4. All other byte values select clearing. Character signedness does not change this partition.

The selector is not explicitly reread after a dependency call. Changes made by dependencies therefore do not restart branch selection.

The source representation orders the selector read before the local initializations. Exact machine instruction scheduling is not established. Both local cells are initialized before any branch-specific dependency call.

### State 1: first global address and primary-object dispatch

For an initial selector of 1, the function selects the address of the state-1 label object.

The visible dependency and dispatch sequence is:

1. F-9b9c1c7483f4 receives three arguments, in this order:
   - the address of the local cell initialized from the fourth argument;
   - the address of the state-1 label object;
   - the address of the local cell initialized from the third argument.
2. Its opaque 32-bit result is obtained.
3. The primary object’s dispatch-table reference is read.
4. The callable entry at table byte offset 0x11C is read.
5. That entry is invoked with the primary object as receiver and the opaque result as the other explicit argument.
6. After that invocation returns normally, F-8a5aadac202f receives the address of the same fourth-argument local cell.
7. The function returns without an explicit result.

The dispatch-table lookup occurs after F-9b9c1c7483f4 returns. An implementation must not substitute a table reference captured before that call when doing so would change observable behavior.

No explicit selector write occurs on this branch. Dependencies may nevertheless change the selector or other reachable state.

The global address is established; the apparent “PopNowPlaying” text is not an established payload. F-9b9c1c7483f4 could interpret the address or pointed-to storage in ways not shown here.

### State 2: second global address and primary-object dispatch

For an initial selector of 2, the function selects the address of the state-2 label object.

The subsequent sequence, argument positions, result forwarding, dispatch offset, and final F-8a5aadac202f invocation are the same as for state 1. The established branch difference is which global address occupies the second argument position of F-9b9c1c7483f4.

The apparent “PopToMainMenu” association comes from the object’s symbol name. Exact bytes, textual interpretation, encoding, termination, and immutability remain unknown.

Distinct selected addresses do not establish distinct contents. They also do not establish that either object contains text.

### State 3: alternate receiver and first entry

For an initial selector of 3:

1. F-b7af80e829ec is invoked with no explicit arguments.
2. Its result is treated as a receiver pointer.
3. The dispatch-table reference at that receiver’s offset 0 is read.
4. The callable entry at table offset 0xC0 is read.
5. Control is transferred to that entry with the returned receiver and the numeric argument 3.

Neither F-9b9c1c7483f4 nor F-8a5aadac202f is explicitly invoked on this branch. The third and fourth inputs are copied into local storage but are not explicitly forwarded to the alternate operation.

The supplied representation identifies this indirect transfer as incompletely recovered and represents it as a call followed by return. The receiver, selected offset, and numeric argument are visible; an exact machine-level distinction between ordinary call and tail transfer is not established.

### State 4: alternate receiver and second entry

For an initial selector of 4, behavior matches state 3 except that the callable entry is read at table offset 0xC4.

The numeric argument is still 3. The selector value 4 is not forwarded as that argument.

### Unsupported states

For every selector byte other than 1, 2, 3, or 4, the function writes zero to primary-object offset 0xBF and returns.

This branch performs no explicit dependency invocation and does not explicitly read the primary object’s dispatch-table reference.

An initial zero still causes a one-byte zero write. The established behavior is not merely preservation of an already-zero value.

### Widths and arithmetic

The selector read and clearing write are each one byte wide. Clearing is not an explicit write to a surrounding 32-bit field.

The third and fourth arguments are copied as opaque 32-bit values. F-9b9c1c7483f4’s result is also represented as 32 bits. The numeric argument to the alternate operations has the value 3; exact ABI placement and extension rules are not supplied.

Address calculations use fixed offsets. No input-dependent count, multiplication, loop, or length computation appears. Address overflow, invalid mappings, and alignment faults have no visible checked handling.

## Side effects

### Direct effects and observable ordering

The only explicit write through the primary-object pointer is the unsupported-state zero write.

For states 1 and 2, both local-cell addresses are exposed to F-9b9c1c7483f4. The fourth-argument cell’s address is later exposed to F-8a5aadac202f. These dependencies may modify the cells; preservation of their initial contents is not established.

The visible order requires preparation to return before the primary-object table lookup, and requires the primary-object operation to return before F-8a5aadac202f is called.

The body contains no explicit heap allocation, deallocation, global-object write, I/O, or synchronization operation. Selecting a global label-object address is not evidence of reading its contents.

### Dependency-mediated effects

No supplied dependency implementation constrains preservation of:

- the primary object or selector;
- dispatch-table references or entries;
- the selected global label object or the other label object;
- global state;
- resources reachable through opaque input values;
- memory reachable through other aliases.

Consequently, this contract does not promise that supported states leave the selector unchanged, that the label objects are immutable, or that dispatch-table contents remain stable across calls.

### Ownership and lifetime

The two local cells belong to the current invocation. Passing their addresses does not establish a right to access their storage after the invocation ends.

No ownership transfer is established for the primary object, alternate receiver, global label objects, opaque arguments, or returned 32-bit value.

The preparation-and-final-call sequence may suggest temporary-value construction and cleanup, but that is inferred intent. It does not prove allocation, destruction, release, reference counting, or consumption.

F-8a5aadac202f receives the same fourth-argument cell address after the primary-object operation returns. It may therefore observe a value changed by earlier dependencies rather than the original fourth argument.

### Errors and abnormal completion

There is no explicit status result, retry, fallback dispatch, or error translation.

The visible path reaches F-8a5aadac202f only after F-9b9c1c7483f4 and the primary-object operation return normally. No cleanup guarantee is established if either fails to return normally. Faults, exceptions, and nonlocal exits are otherwise unspecified.

## Dependencies

| Dependency | Explicit inputs | Established role |
|---|---|---|
| F-9b9c1c7483f4 | Fourth-argument cell address; selected global label-object address; third-argument cell address | Produces the opaque 32-bit value forwarded to primary-object dispatch. |
| F-8a5aadac202f | Fourth-argument cell address | Runs after normal completion of the primary-object operation in states 1 and 2. Exact effects are unknown. |
| F-b7af80e829ec | None | Supplies the receiver used in states 3 and 4. Ownership and lookup behavior are unknown. |

Three indirect invocation sites are established structurally:

- Primary-object dispatch-table entry at offset 0x11C.
- Alternate-receiver dispatch-table entry at offset 0xC0.
- Alternate-receiver dispatch-table entry at offset 0xC4.

Their concrete runtime targets are not identified by the supplied dependency map.

## Acceptance examples

These are conditional static acceptance criteria, not executed test results. Unless stated otherwise, they assume valid required memory and normally returning dependencies.

1. **State 1 selects an address, not a proven string.**  
   With selector 1, the second argument to F-9b9c1c7483f4 is the state-1 label-object address. Acceptance does not require asserting any particular bytes at that address.

2. **State 2 selects the other object.**  
   With selector 2, the second argument is the state-2 label-object address. The primary-object dispatch entry remains at offset 0x11C. Equal contents, if later established, would not erase the address-selection distinction.

3. **Preparation results are forwarded.**  
   If F-9b9c1c7483f4 returns an opaque 32-bit value, the primary-object operation receives that value. F-8a5aadac202f subsequently receives the original fourth-argument cell’s address, even if its contents changed.

4. **Preparation can affect dispatch lookup.**  
   If F-9b9c1c7483f4 changes the primary object’s table reference through an available alias, the subsequent dispatch uses the reference read after that call.

5. **State 3 uses offset 0xC0.**  
   With selector 3, the receiver supplied by F-b7af80e829ec is used for the table lookup and invocation. The selected operation receives that receiver and numeric argument 3.

6. **State 4 changes the entry, not the numeric argument.**  
   With selector 4, the selected table offset is 0xC4 and the explicit numeric argument remains 3.

7. **Unsupported values cause a byte write.**  
   With selector 5, the function writes zero to the selector byte without invoking the listed dependencies. With selector zero, the same explicit write occurs. No wider clearing write is established.

8. **Dependency mutation does not reselect the branch.**  
   If preparation on state 1 changes the selector to 4, the invocation continues the already-selected primary-object path.

9. **Abnormal completion does not establish finalization.**  
   If the primary-object operation does not return normally, the evidence does not require a subsequent F-8a5aadac202f call.

## Uncertainties

The following remain **documented_only**:

- Definitions, exact bytes, sizes, encodings, termination, mutability, and semantic meaning of both global label objects.
- Whether F-9b9c1c7483f4 reads, copies, retains, modifies, or otherwise interprets either selected object.
- Exact ABI, pointer encoding, alignment requirements, and argument-extension rules.
- Complete object layouts and concrete indirect-call targets.
- Meaning of the selector field and numeric argument 3.
- Ownership, aliasing, allocation, retention, consumption, and release contracts of all dependencies.
- Thread safety, reentrancy, and concurrent-mutation behavior.
- Exact machine-level form of the alternate indirect transfer.
- Fault, exception, and nonlocal-exit behavior beyond the visible normal path.

The symbol names suggest navigation-related intent. They do not establish a textual payload or the effects of any invoked operation.

## Evidence

The supplied static body establishes:

- A selector-byte read at primary-object offset 0xBF.
- Four recognized selector values.
- State-specific selection of two global addresses for states 1 and 2.
- Passage of the selected address to F-9b9c1c7483f4, without a visible read of the addressed contents by this function.
- Two local 32-bit cells and their argument relationships.
- Primary-object dispatch at table offset 0x11C after preparation returns.
- Subsequent F-8a5aadac202f invocation with the fourth-argument cell address.
- Alternate receiver acquisition through F-b7af80e829ec.
- Alternate dispatch offsets 0xC0 and 0xC4, each with numeric argument 3.
- A one-byte zero write for every unsupported selector.
- No explicit use of the second input, validation, retry, or error result.

The evidence does not include the global label-object definitions, dependency implementations, or runtime observations. Apparent label text is therefore inferred from symbol names, not established from object bytes.
