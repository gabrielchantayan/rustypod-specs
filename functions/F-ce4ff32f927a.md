<!-- Trusted provenance: F-ce4ff32f927a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ce4ff32f927a

## Purpose

F-ce4ff32f927a records a caller-supplied byte in an existing object, then chooses between a warning-related transition and a delegated alternative transition.

The warning-related transition is selected only when three successive conditions hold: a service query returns exactly one, a predicate over a temporary value returns nonzero, and a second predicate over that value returns zero. On that path, the function conditionally invokes a service action, performs two further operations, conditionally writes a 16-bit state value, and submits a constructed value to an object method.

The visible behavior establishes the selection conditions and operation order. Interpreting the transition as a rental warning is inferred from the embedded label `GotoRentalWarningLayout`; the actual presentation and downstream transition behavior are not established.

## Interface

**Inputs**

- An object address, referring to writable storage with a virtual-dispatch pointer at byte offset zero.
- An 8-bit value to store in that object.

The visible representation is 32-bit: object words, addresses, and virtual-dispatch entries are represented as four-byte quantities. No signed interpretation of the input byte is required; its eight bits are copied unchanged.

**Output**

The function has no explicit return value. Its observable results are memory changes and dependency calls.

**Object layout used directly**

| Byte offset | Width | Use |
|---|---:|---|
| `0x00` | 32 bits | Virtual-dispatch pointer, used on the warning-related path |
| `0xB8` | 32 bits | Value read and passed to F-936a8728139d |
| `0xBF` | 8 bits | Destination for the caller-supplied byte |

These locations do not overlap. The function does not directly read the previous value at `0xBF`.

**Temporary storage**

The function provides:

- A 24-byte temporary region to F-ba5a4c1e0746, several later operations, and finally F-4e292a3dd745.
- A four-byte temporary region to F-9b9c1c7483f4 and then F-8a5aadac202f.
- Another four-byte temporary region passed to F-9b9c1c7483f4.

The supplied evidence does not establish the temporary regions’ types, internal layouts, or ownership rules. No explicit initialization of either four-byte region is visible before its address is passed onward.

**Preconditions for a defined normal completion**

The input object must support the accesses and dispatches actually reached. In particular, the byte at `0xBF` must be writable. The warning-related path additionally requires a readable 32-bit value at `0xB8` and a valid object dispatch through table offset `0x11C`.

Every reached service acquisition and virtual dispatch must provide valid storage and callable entries. No general null-safety guarantee is visible. Any additional dependency-specific preconditions remain documented_only.

## Behavior

### 1. Prepare the temporary value and capture the first query

The function first calls F-ba5a4c1e0746 with the address of the 24-byte temporary region and two zero-valued arguments.

It next calls F-b7af80e829ec and uses the returned address to perform an indirect call through virtual-table byte offset `0x13C`. The result is retained as a 32-bit condition value.

No explicit receiver argument is shown for this indirect call. Its full calling convention and any implicit receiver behavior are not established.

This query occurs before the caller-supplied byte is stored in the input object.

### 2. Populate or otherwise process the temporary value

The function calls F-b7af80e829ec again. This acquisition is independent of the previous one; the visible behavior does not cache or require the same result.

If the returned address is nonzero, it is adjusted backward by 20 bytes. That displacement is five four-byte words. If the returned address is zero, it remains zero.

The function then uses the resulting address as an object address, reads its dispatch pointer, and invokes the entry at virtual-table byte offset `0x15C`. The explicit arguments are the adjusted address and the address of the 24-byte temporary region.

The nonzero check controls only the address adjustment. It does **not** skip the dispatch when acquisition returns zero. A zero result therefore leads to an attempted dispatch through address zero, rather than a normal fallback or early return.

The role of the 20-byte adjustment is unknown. A subobject-to-containing-object interpretation is possible, but is not required by this contract.

### 3. Store the input byte

After the preceding indirect call returns, the function writes the supplied 8-bit value to the input object at byte offset `0xBF`.

This write occurs before branch selection and on both normal branch paths. The function does not condition the write on the byte’s value.

Dependencies called afterward may modify the same object or other state. Consequently, the direct write is established, but the byte’s final value at function return is not guaranteed without dependency contracts.

### 4. Select the transition path

The warning-related path is taken only when all of the following tests succeed, in this order:

1. The retained result of the first service query equals exactly `1`.
2. F-7183096dcb3a, called with the 24-byte temporary region, returns a nonzero result.
3. F-d188fd98fcb6, called with the same temporary region, returns zero.

Evaluation is short-circuited:

- If the first result is anything other than `1`, neither predicate is called.
- If F-7183096dcb3a returns zero, F-d188fd98fcb6 is not called.
- Any nonzero result from F-d188fd98fcb6 selects the alternative path.

There is no normalization of the first result to Boolean truth. In particular, `2` and a negative nonzero value do not satisfy the first test.

### 5. Execute the warning-related path

When all three conditions hold, the following operations occur in order.

**Conditional service action**

The function acquires another service address through F-b7af80e829ec and calls its virtual entry at table byte offset `0x10C`.

If that query returns nonzero, the function acquires a service address again through F-b7af80e829ec and calls its virtual entry at table byte offset `0xAC`.

The second acquisition is not performed when the query returns zero. When it is performed, its result is not required by the visible code to match the address used for the query. Neither indirect call shows explicit arguments.

**Two ordered operations**

The function reads the 32-bit object value at byte offset `0xB8` and passes that value to F-936a8728139d.

It then calls F-ac356e5603c7 with no visible arguments.

The object-field read occurs at this point, after the conditional service action. It is not a snapshot taken at entry. If earlier calls alter this field, the later value is the one read.

**Conditional 16-bit state update**

The function calls F-78443a7fbacd.

If the result is nonzero, it calls F-fdd8589eca2d and writes the numeric value `6`, using a 16-bit store, at byte offset `0x80` from the returned address.

If the result is zero, F-fdd8589eca2d is not called and this direct store does not occur.

The function does not inspect the previous 16-bit value, validate the returned destination, or explicitly write adjacent bytes.

**Construct and submit the transition value**

The function calls F-9b9c1c7483f4 with:

- The address of the first four-byte temporary region.
- The label `GotoRentalWarningLayout`.
- The address of the other four-byte temporary region.

Its 32-bit result is passed to the input object’s virtual entry at table byte offset `0x11C`, together with the input object address.

After that method returns normally, the function calls F-8a5aadac202f with the first four-byte temporary region.

The constructor-like call’s result is not checked for zero or any other status. The function does not itself establish whether that result is an address, handle, or another 32-bit value.

### 6. Execute the alternative path

If any selection condition fails, the function calls F-4d5e4ca0e6d5 with the input object address.

No warning-path operation is performed after entering this alternative path.

This delegated call occurs after the write at object offset `0xBF`, so the object has received that write before delegation begins. What the dependency subsequently reads or changes is unknown.

### 7. Finalize the main temporary

After either path returns normally, the function calls F-4e292a3dd745 with the 24-byte temporary region, then returns without a value.

This establishes normal-path finalization. It does not establish cleanup following an invalid memory access, an abnormal dependency exit, or a non-returning indirect call.

## Side effects

**Direct memory effects**

The visible function directly performs:

1. An unconditional one-byte write to input-object offset `0xBF`, after initial service processing.
2. A conditional two-byte write of numeric value `6` to offset `0x80` from an address returned by F-fdd8589eca2d.

“Unconditional” here means that no branch guards the first store once execution reaches it. Earlier failures can prevent that point from being reached.

Other modifications can occur through the temporary addresses and dependency calls. Their extent is not established.

**Ordering and shared state**

Calls and explicit stores occur in the order described above. Repeated F-b7af80e829ec calls remain separate observable acquisitions.

No dependency is assumed to preserve global state, the input object, service identities, or previously acquired addresses. The initial query result is retained for selection, but subsequent addresses and the object field at `0xB8` are obtained when their respective operations are reached.

**Ownership and lifetime**

The caller’s object is used in place; no explicit allocation, copy, or release of that object appears.

The visible code passes addresses of local temporary storage to dependencies. It neither establishes permission for those dependencies to retain the addresses beyond the call nor proves that they do not retain them.

F-8a5aadac202f and F-4e292a3dd745 occur at cleanup-like positions. That ordering alone does not prove that they free memory, release references, or own the resources represented by their arguments.

**Errors**

There is no explicit error return, rollback, retry, or validation of service addresses. A partially completed invocation may already have written the input byte or performed earlier side effects before a later failure.

## Dependencies

| Opaque ID or dispatch | Established use |
|---|---|
| F-ba5a4c1e0746 | Receives the 24-byte temporary address and two zeros before all service operations |
| F-b7af80e829ec | Repeatedly supplies addresses used for indirect calls |
| Service entry `0x13C` | Produces the initial result compared with exactly `1` |
| Adjusted-service entry `0x15C` | Receives the adjusted address and 24-byte temporary address |
| F-7183096dcb3a | First temporary-value predicate; nonzero permits the next predicate |
| F-d188fd98fcb6 | Second temporary-value predicate; zero permits the warning path |
| Service entry `0x10C` | Determines whether the additional service action is invoked |
| Service entry `0xAC` | Invoked only after a nonzero `0x10C` result |
| F-936a8728139d | Receives the 32-bit object value read at `0xB8` |
| F-ac356e5603c7 | Called after F-936a8728139d |
| F-78443a7fbacd | Gates the conditional 16-bit state write |
| F-fdd8589eca2d | Supplies the base address for that write |
| F-9b9c1c7483f4 | Receives two temporary addresses and the warning-transition label; returns a 32-bit value |
| Object entry `0x11C` | Receives the input object and the constructed value |
| F-8a5aadac202f | Receives the first four-byte temporary after object dispatch |
| F-4d5e4ca0e6d5 | Implements the alternative path |
| F-4e292a3dd745 | Receives the 24-byte temporary after either normal path |

Semantic names for the indirect methods and complete dependency contracts are documented_only.

## Acceptance examples

These examples are conditional acceptance criteria, not reported executions. Unless stated otherwise, all reached dependencies return normally and all accessed memory is valid.

1. **Initial result is noncanonical truth.**  
   If the first query returns `2`, the byte is still stored at `0xBF`. Neither temporary predicate is called. F-4d5e4ca0e6d5 is called, followed by F-4e292a3dd745.

2. **First predicate rejects.**  
   If the first query returns `1` and F-7183096dcb3a returns zero, F-d188fd98fcb6 is skipped. The alternative path runs.

3. **Second predicate rejects.**  
   If the first query returns `1`, the first predicate returns nonzero, and the second returns nonzero, both predicates run in that order and the alternative path follows.

4. **Warning path without either conditional action.**  
   If the selection conditions pass, the service query at `0x10C` returns zero, and F-78443a7fbacd returns zero, no `0xAC` dispatch or 16-bit state store occurs. The two ordered operations, transition construction, object dispatch, and normal-path cleanup still occur.

5. **Warning path with state update.**  
   If F-78443a7fbacd returns nonzero, F-fdd8589eca2d is called once at that point. A 16-bit value of `6` is written at its returned base plus `0x80` before transition construction.

6. **Changing acquired addresses.**  
   If successive service acquisitions return different valid addresses, each dispatch uses the address from its corresponding acquisition. Only the acquisition used for entry `0x15C` receives the conditional 20-byte backward adjustment.

7. **Zero transition value.**  
   If F-9b9c1c7483f4 returns zero, the object method still receives zero as its second explicit argument. No visible error branch replaces the dispatch.

8. **Zero adjusted-service acquisition.**  
   If the second service acquisition returns zero, the function attempts the `0x15C` dispatch through zero before storing the input byte. A safe fallback is not an acceptance requirement; it is absent from the visible behavior.

## Uncertainties

The following remain documented_only:

- The semantic purpose and internal representation of the 24-byte temporary.
- The service’s identity, lifetime, and stability across acquisitions.
- The meaning of the 20-byte address adjustment.
- Full calling conventions for indirect calls without explicit arguments.
- Whether the embedded label causes visible navigation, warning display, or another action.
- Ownership transfer, allocation, release, reference counting, and retained-address behavior inside dependencies.
- The contents and initialization requirements of the four-byte temporary regions.
- Global-state preservation, reentrancy, thread safety, and synchronization.
- Exception, fault, and abnormal-exit cleanup.
- Runtime byte order and external binary-layout requirements beyond the visible access widths and offsets.

The warning interpretation is inferred intent. Branch conditions, direct stores, argument relationships, and normal-path ordering are established static observations.

## Evidence

This contract is based solely on static inspection of the supplied function representation and its dependency-ID mapping, treated as evidence rather than behavioral instructions.

The evidence establishes the visible control flow, explicit access widths, byte offsets, arguments, and ordering. It does not include dependency implementations or runtime observations.

No runtime tests have been performed. The acceptance examples describe required behavior under stated conditions; they are not execution results.
