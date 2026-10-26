<!-- Trusted provenance: F-a7f0e1eb2c29; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a7f0e1eb2c29

## Purpose

F-a7f0e1eb2c29 performs an indirect operation through a caller-supplied object and conditionally invokes a second operation on that object. It returns the result of the first operation.

The first operation receives the object, the address of a temporary four-byte value, a callable value obtained from the object's dispatch table, the fourth input, and the object again. If that operation returns anything other than signed negative one, the function dispatches a second operation with the object and the returned value. The second operation's ordinary return value does not affect the result.

This is the established control-flow contract. The semantic roles of the two operations are unknown. In particular, the source does not establish that the first operation allocates or acquires a resource, or that the second releases, commits, validates, or registers one.

## Interface

### Inputs

The function has four inputs, described here by position:

| Input | Established role |
|---|---|
| First | An object reference used to obtain dispatch information and passed to both indirect operations. |
| Second | A four-byte value copied into temporary storage before the first indirect operation. |
| Third | A four-byte input with no explicit use in the supplied function. |
| Fourth | A four-byte value forwarded to the first indirect operation. |

The first input must designate storage from which the initial dispatch-table reference can be obtained. Dispatch entries are addressed at byte offsets 76 and 44 from the applicable table reference.

The four-byte designation for the second, third, and fourth inputs is explicit in the supplied representation. Their interpretation is not established: they could represent numeric values, flags, handles, address bits, or other data. This contract does not assign them a meaning.

The integer result and the first input's leading field are represented as integer-sized quantities. A 32-bit integer and address model is consistent with the source, but the source alone does not independently establish every machine-level type, pointer width, or calling-convention detail. The result descriptions below use the represented signed-integer interpretation.

### Result

On normal completion, the function returns the integer result produced by the first indirect operation.

There are two branches:

- A result equal to signed negative one is returned without invoking the second operation.
- Any other result causes the second operation to be invoked before that same result is returned.

Zero is not a skip condition. Other negative values are not skip conditions.

The function does not return the result of the second operation, and it does not reread the temporary four-byte value to determine its return value.

### Directly addressed layout

Only the following locations have an established role:

| Location | Role |
|---|---|
| Beginning of the object referenced by the first input | Holds the value used as a dispatch-table reference. |
| Dispatch-table byte offset 76, hexadecimal 0x4c | Supplies the first indirect target and the callable-valued third argument to that operation. |
| Dispatch-table byte offset 44, hexadecimal 0x2c | Supplies the second indirect target, when the first result is not negative one. |
| Function-local four-byte storage | Receives a copy of the second input and is passed by address to the first operation. |

The offsets are byte offsets, not entry indices. No array length, complete object layout, table size, or other field layout is established.

## Behavior

### 1. Prepare the temporary value

The function copies the second input into four bytes of local storage.

The address of this storage is supplied to the first operation. This exposes the copied value through an address; it does not expose the caller's original argument storage merely because the values initially match.

The function itself performs no subsequent read of the temporary value. Changes made through its address therefore do not directly determine the returned integer or the conditional branch. Such changes can still affect behavior inside the called operation or any other behavior that operation triggers.

The third input contributes to no explicit calculation, memory access, or outgoing argument in the supplied function.

### 2. Invoke the first operation

The first indirect operation is identified in this contract as **F-92d6b4e1a037**. This is a specification-local opaque label for a dispatch role, not an established identity of the runtime target.

Its target is obtained through the dispatch-table reference at the beginning of the first input's object, using byte offset 76.

It receives five arguments in this order:

1. The first input.
2. The address of the local four-byte copy of the second input.
3. A callable value obtained from dispatch-table byte offset 76.
4. The fourth input.
5. The first input again.

The first and fifth arguments carry the same object reference. The second argument identifies writable local storage initialized from the second input.

The dispatch lookup for the call target and the lookup for the third argument are separately expressed accesses. The supplied source does not establish their relative evaluation order or an atomic snapshot of the object and table contents.

If the relevant object and table contents remain stable during call setup, the third argument carries the same callable value as the invoked target. If those contents can change during setup, equality is not established by this source. The contract does not impose synchronization or turn the two lookups into one read.

All argument values and the target needed for this invocation must have been obtained before the operation begins. The precise machine-level order of those reads is unknown.

### 3. Inspect the first result

After the first operation returns normally, the function retains its integer result and compares it with signed negative one.

For a represented 32-bit integer result, negative one has the bit pattern 0xffffffff. This is an exact-value comparison, not a sign test, zero test, or general success/failure classification.

No arithmetic transformation of the returned value is visible before this comparison.

### 4. Conditionally invoke the second operation

If the first result is not negative one, the function invokes **F-c71805ad9b62**, another specification-local opaque label for a dispatch role.

The second target is obtained from byte offset 44 of the dispatch table reached through the object at this later point. The function does not visibly preserve the original table reference for this dispatch.

Consequently, if the first operation changes the object's dispatch-table reference or relevant table contents, those changes can affect the second target. This function does not restore the earlier state.

The second operation receives two arguments:

1. The first input.
2. The integer result of the first operation.

Its ordinary return value is ignored.

The later lookup occurs only on the non-negative-one branch. A call whose first result is negative one does not require this function to read or invoke the offset-44 entry.

### 5. Return the retained result

After the optional second operation returns normally, the function returns the retained result of the first operation.

There is no visible retry, normalization, result translation, or additional notification. If either indirect operation does not return normally, the supplied function establishes no subsequent normal return.

## Side effects

### Direct effects

The function directly:

- Writes the second input into local four-byte storage.
- Reads object and dispatch-table information for the first operation.
- Calls the first operation.
- Conditionally reads object and dispatch-table information for the second operation.
- Conditionally calls the second operation.

No direct write to the caller's object is visible. No direct global-state write, allocation, deallocation, or external I/O operation is visible.

These observations do not imply that the indirect operations lack those effects.

### Effects delegated to dependencies

Both dependencies receive the object reference. The first also receives the local storage address and the fourth input. Their effects on object state, globals, external systems, and memory are unknown.

This function performs no visible save-and-restore of global state around either call. No preservation of globals may be assumed from this wrapper.

Object and table mutation by the first operation is especially relevant because the second dispatch is resolved afterward.

### Ownership and lifetime

The source establishes no transfer of ownership for the object, fourth input, result, or any resource those values might designate.

The temporary four-byte storage has invocation-local lifetime. Its address is available to the first operation during the call. Retaining that address does not extend the storage's lifetime. Whether the dependency retains it is unknown; the wrapper provides no persistent replacement storage.

The object and all dispatch information actually accessed must remain usable at the times of those accesses. If the first operation invalidates the object and returns a value other than negative one, the wrapper still proceeds toward the later object-based dispatch. No check guards that use.

## Dependencies

| Opaque ID | Invocation role | Established outgoing arguments | Established result use |
|---|---|---|---|
| F-92d6b4e1a037 | First indirect operation, selected at byte offset 76 | Object; local four-byte storage address; callable value from offset 76; fourth input; object again | Integer result selects the conditional branch and becomes the wrapper's normal return value. |
| F-c71805ad9b62 | Conditional indirect operation, selected at byte offset 44 after the first operation returns | Object; first operation's integer result | Ordinary return value ignored. |

These identifiers distinguish the two dispatch roles within this document. They do not establish a one-to-one mapping to concrete implementations. Different object state can select different runtime targets.

The dependencies' complete signatures, calling conventions, error behavior, memory ownership rules, global effects, and reentrancy properties are not established.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

### First operation returns negative one

Given valid dispatch state and a first operation that returns negative one normally:

- The first operation receives the five arguments described above.
- The wrapper does not perform the conditional offset-44 dispatch.
- The wrapper returns negative one.

This remains the branch even if the first operation modified the temporary value.

### First operation returns zero

Given a first operation that returns zero normally:

- The wrapper obtains the second target through the object's then-current dispatch state.
- The second operation receives the object and zero.
- If that operation returns normally, the wrapper returns zero.

Treating zero as a reason to skip the second operation would violate the established behavior.

### First operation returns another negative value

Given a first result of negative two:

- The second operation is invoked with negative two.
- On normal completion, the wrapper returns negative two.

A general “negative means failure” rule is not supported.

### Second operation produces a different result

Given a first result of 23 and a second operation that returns 91 normally:

- The second operation receives 23.
- The wrapper returns 23, not 91.

### First operation replaces the dispatch table

Given that the first operation changes the object's table reference, leaves the object valid, and returns 7:

- The later offset-44 lookup uses the object's post-call dispatch state.
- The selected second operation receives the object and 7.

Using a previously cached initial table for that second lookup would not preserve the visible lookup structure.

### First operation modifies the temporary value

Given a second input with an arbitrary four-byte pattern:

- The temporary initially contains that pattern.
- The first operation receives its address.
- A modification through that address is not subsequently read by the wrapper.
- The branch and normal return remain determined by the first operation's integer result.

### Third input varies

For executions differing only in the third input, with otherwise identical relevant state and dependency behavior, this function has no explicit use through which that input changes its outgoing calls or result.

This does not establish an undocumented dependency ABI in which a callee may inspect unspecified surrounding execution state.

## Uncertainties

The following are **documented_only** rather than guessed:

- **Dependency semantics:** Neither operation has an established application-level purpose.
- **Exact ABI:** Full pointer widths, register assignments, stack layout, and calling conventions are not independently established.
- **Lookup evaluation order:** The relative order of the first target lookup and callable-valued argument lookup is unspecified by the supplied evidence.
- **Concurrency:** No atomicity, synchronization, or race-safety guarantee is visible.
- **Validation failures:** There are no visible null, bounds, alignment, or callable-validity checks. The precise consequence of an invalid access or target is not established.
- **Dependency errors:** Negative one is a branch sentinel here; its broader meaning is unknown. Other error mechanisms, including nonlocal exits, are unspecified.
- **Ownership and preservation:** Neither resource transfer nor preservation of objects, globals, or memory by dependencies is established.
- **Temporary-address retention:** The first dependency's retention behavior is unknown; the storage remains local in lifetime.

Stable dispatch state and compatible callable signatures are practical preconditions for the straightforward interpretation of the calls, not validation services supplied by this function.

## Evidence

The supplied static representation establishes the four-byte local copy, the two offset-76 lookup expressions associated with the first call, its five outgoing arguments, the exact negative-one comparison, the conditional later offset-44 dispatch, and return of the first operation's result.

The same representation establishes no explicit use of the third input and no explicit consumption of the temporary value after the first call.

An object with a dispatch table is an inferred structural interpretation of the leading-field dereference and fixed-offset indirect calls. A particular object system, class identity, resource protocol, or application domain is not established.

No runtime tests have been performed. The acceptance examples specify conditional observable behavior; they do not claim runtime verification.
