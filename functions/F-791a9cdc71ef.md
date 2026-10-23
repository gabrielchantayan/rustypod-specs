<!-- Trusted provenance: F-791a9cdc71ef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-791a9cdc71ef

## Purpose

F-791a9cdc71ef performs two ordered indirect calls through an object associated with its first input. The first call receives a pointer to a zero-initialized, 32-bit result cell. The second call occurs afterward, and the function returns the result cell’s final 32-bit contents.

The established behavior is dispatch, not a known domain operation. The supplied representation does not establish what either indirect target does. In particular, the second call must not be described as cleanup, release, completion, or cancellation without additional evidence.

The two calls resolve their targets separately. The second dispatch uses the associated object and dispatch table available after the first call returns; it is not established to use the same object or table as the first dispatch.

## Interface

### Inputs

The function has three explicit inputs, referred to here as A, B, and C:

| Input | Established use |
|---|---|
| A | Address-bearing input used to locate an associated-object pointer at byte offset 4. |
| B | An uninterpreted 32-bit value forwarded to both indirect calls. |
| C | An uninterpreted 32-bit value forwarded only to the first indirect call. |

A is represented as an integer in the supplied source representation but is used for memory access. Successful execution therefore requires it to designate a suitable memory region under the target’s address model.

B and C have no established signedness or semantic type. They could represent values, handles, flags, or addresses; the wrapper does not distinguish these possibilities.

### Output

The return value is a 32-bit quantity taken from a local result cell:

- The cell initially contains zero.
- Its address is supplied to the first indirect target.
- The cell is read for the return value after the second indirect call returns.

No interpretation of the returned bits as a status, Boolean, integer, or pointer is established.

### Referenced layout

The supplied representation supports the following operational layout:

| Location | Role |
|---|---|
| Byte offset 4 from A | Associated-object pointer |
| Byte offset 0 from the associated object | Dispatch-table pointer |
| Byte offset 60 from the dispatch table | First indirect-call target |
| Byte offset 44 from the dispatch table | Second indirect-call target |

The offsets are byte offsets, not method indices. A conventional four-byte-slot interpretation would place these targets at zero-based slots 15 and 11, respectively, but the byte offsets are the directly established facts.

The representation strongly indicates a 32-bit address and pointer model: the address-bearing input is an integer, pointer accesses are represented through integer-sized indirections, and the layout is consistent with four-byte entries. The original target ABI and original type declarations are not supplied. The contract must not be silently widened to a native 64-bit layout.

### Call shapes

The first indirect target receives five arguments in this semantic order:

1. The associated-object pointer used for that dispatch.
2. B.
3. The address of the local 32-bit result cell.
4. The first indirect target’s own resolved function-pointer value.
5. C.

The second indirect target receives two arguments:

1. The associated-object pointer used for the second dispatch.
2. B.

The wrapper does not consume either indirect call’s direct return value. Its own return value comes from the result cell instead.

## Behavior

### Initialization

Before resolving the first indirect target, the function initializes its local 32-bit result cell to zero.

This initialization supplies the default return bits if the cell remains unchanged through both calls. It does not establish that zero means success, failure, absence, or any other domain condition.

### First dispatch

The function obtains the associated-object pointer through byte offset 4 from A. It follows that object’s dispatch-table pointer and obtains the target at byte offset 60.

The resolved target is used both as:

- The destination of the first indirect call.
- The fourth argument to that same call.

This identity is significant. The fourth argument is not a second independently selected callback and is not shown as an address of the dispatch-table entry. It is the resolved function-pointer value used for the call.

The associated-object pointer is also obtained from A when preparing the receiver argument. The representation therefore contains separate accesses for target resolution and receiver preparation. It does not establish an atomic snapshot of the object pointer and dispatch table.

Under stable memory, the first call uses the object whose table supplied its target. Under unsynchronized mutation or memory with externally visible read behavior, stronger consistency is not established.

The first indirect target receives writable access to the result cell. The wrapper does not inspect that cell before proceeding to the second dispatch.

### Second dispatch

After the first call returns normally, the function performs a new dispatch lookup using A:

- It obtains the associated-object pointer from byte offset 4.
- It follows the current dispatch-table pointer.
- It obtains the target at byte offset 44.
- It calls that target with an associated-object pointer obtained from A and with B.

The first target’s direct return value does not control whether the second call occurs. There is no visible branch based on that value or on the result cell.

The fresh lookup is behaviorally important. The first target may have changed the pointer stored at offset 4, the referenced object, or its dispatch table. Nothing supplied establishes that it preserves any of these. A conforming description must therefore allow the second dispatch to resolve to a different object or table.

As with the first dispatch, the target lookup and receiver preparation are not established as one indivisible operation. Exact ordering among all argument-preparation reads is not recoverable solely from their textual placement in the call expression.

### Return

After the second target returns normally, the function returns the result cell’s then-current 32-bit contents.

The second call is not directly passed the result-cell address. Nevertheless, the evidence does not prove that the cell is unchanged during that call: the first target received its address and could have made it reachable elsewhere. Consequently, the return is the value after both calls, not necessarily a value captured immediately after the first call.

The wrapper performs no visible conversion, range check, masking, or arithmetic adjustment on that returned value.

### Failure and validity boundaries

There are no visible checks for:

- A being null or otherwise invalid.
- A readable associated-object pointer at offset 4.
- A valid associated object or dispatch table.
- A callable target at either dispatch offset.
- Successful completion of either indirect call.

The supplied representation does not define a recoverable error result for invalid memory or an invalid target. It would be incorrect to specify that such cases return zero merely because the result cell starts at zero.

If the first call does not return normally, no subsequent second dispatch is established. If the second call does not return normally, no normal wrapper return is established. The precise behavior of faults, exceptions, or nonlocal transfers is documented_only.

## Side effects

### Direct effects

The wrapper directly:

- Initializes a local, four-byte result cell to zero.
- Reads pointer-bearing memory reachable through A.
- Reads two dispatch-table entries at different points in execution.
- Invokes two indirect targets in order.
- Reads the result cell for its return value.

No direct store to the memory reachable through A is shown.

The only address arithmetic visible in the wrapper is for the fixed layout offsets. There is no length calculation, scaling of B or C, or arithmetic transformation of the result.

### Effects delegated to indirect targets

The indirect targets may have additional effects not described by the supplied evidence. Neither target is established to preserve global state, the object graph, external resources, or memory reachable through the forwarded inputs.

The wrapper contains no visible allocation, deallocation, reference-count operation, or ownership transfer. This does not establish that the indirect targets perform none of those operations.

The result-cell address is valid during this invocation. No permission or guarantee is established for a callee to retain and use it after the wrapper returns. In particular, the apparent second-dispatch position does not establish that it releases either the object or the result cell.

## Dependencies

The dependency identities cannot be resolved from the supplied mapping. The following opaque IDs are specification-local labels, not verified identities of globally known functions.

### F-D1

Resolved through byte offset 60 of the dispatch table used for the first call.

Its observed interface receives the associated object, B, a writable pointer to the 32-bit result cell, its own resolved function pointer, and C. Its direct return value is ignored.

Its domain behavior, ownership rules, write obligations, and failure semantics are documented_only.

### F-D2

Resolved through byte offset 44 of the dispatch table obtained after F-D1 returns.

Its observed interface receives the associated object and B. Its direct return value is ignored.

Its domain behavior, relationship to F-D1, ownership rules, and failure semantics are documented_only. The distinct table offsets do not prove that F-D1 and F-D2 resolve to distinct function addresses.

## Acceptance examples

These are conditional behavioral criteria derived from static evidence. They are not reports of executed tests.

### Result cell remains untouched

Given valid, stable dispatch data, suppose both targets return normally and neither changes the result cell through any access path.

Expected behavior:

- F-D1 is called first with a pointer to a cell containing zero.
- F-D2 is called afterward.
- The wrapper returns the 32-bit value zero.

### First target supplies a result

Suppose F-D1 writes the 32-bit pattern `0x89ABCDEF` into the supplied result cell, and the cell remains unchanged afterward.

Expected behavior: after F-D2 returns normally, the wrapper returns exactly `0x89ABCDEF`. No signedness conversion or interpretation is required by this contract.

### Direct callee returns differ from the result cell

Suppose both indirect targets produce direct return values, but the result cell remains zero.

Expected behavior: those direct returns do not become the wrapper’s result. The wrapper returns zero after both calls complete normally.

### First target replaces the associated object

Suppose F-D1 changes the object pointer at byte offset 4 from A to another valid object, and that state remains stable during the next lookup.

Expected behavior: the second dispatch resolves through the replacement object’s dispatch table at byte offset 44 and receives the replacement object as its receiver.

### First target replaces only the dispatch table

Suppose the associated-object pointer remains unchanged, but F-D1 installs a different valid dispatch table in that object.

Expected behavior: the second target is obtained from byte offset 44 of the new table, not from a table cached before F-D1 ran.

### Result changes during the second call

Suppose F-D1 makes the result-cell address reachable to F-D2, and F-D2 legally writes `0x00000007` into that cell before returning.

Expected behavior: the wrapper returns 7. It does not return a snapshot taken before F-D2.

### First target does not return normally

Suppose F-D1 terminates execution or transfers control without returning normally.

Expected behavior: this contract does not require a call to F-D2 or a normal return from the wrapper.

## Uncertainties

The following remain documented_only:

- Original source-level types and the precise calling convention.
- Definitive target ABI confirmation beyond the represented 32-bit layout.
- Domain meanings of A, B, C, and the returned bits.
- Identities and full contracts of F-D1 and F-D2.
- Whether either target allocates, releases, retains, or transfers resources.
- Whether either target changes global state or dispatch-related memory.
- Concurrency guarantees and behavior under mutation between separate reads.
- Exact machine-level ordering of reads used to prepare arguments.
- Exception, fault, and nonlocal-transfer semantics.
- Any intended lifecycle relationship between the two calls.

A virtual-dispatch interpretation is consistent with the layout, but class identity and lifecycle intent are inferred rather than established.

## Evidence

The supplied static representation establishes:

- Three explicit inputs and a 32-bit return.
- A zero-initialized 32-bit local result cell.
- Associated-object access at byte offset 4 from A.
- Dispatch-table accesses at byte offsets 60 and 44.
- A five-argument first call, including the result-cell address and the resolved target pointer.
- A later, separately resolved two-argument call.
- Return of the result cell after both calls, with neither direct callee return used.

No runtime tests have been performed. The acceptance examples state consequences of the supplied static evidence under their explicit conditions; they do not establish undocumented callee behavior.
