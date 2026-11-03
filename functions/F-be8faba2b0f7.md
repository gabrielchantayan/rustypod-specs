<!-- Trusted provenance: F-be8faba2b0f7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-be8faba2b0f7

## Purpose

F-be8faba2b0f7 conditionally performs an indirect operation on an object referenced by a caller-supplied record.

The established behavior has three main paths:

- A null record pointer causes a call to F-a817d8d58c10.
- A non-null record whose initial 32-bit word is zero causes an immediate return.
- A non-null record whose initial word is nonzero causes a call to F-f54286f5d91f, followed—if that dependency returns—by an indirect operation using the previously captured object address.

The indirect operation receives the captured object address, the original record address, and an 8-bit value read from the record after F-f54286f5d91f returns.

A plausible intent is to prepare or detach a record before invoking an object-specific operation. That interpretation is not established. In particular, the available evidence does not establish destruction, reference counting, ownership transfer, or a particular meaning for the byte passed to the indirect operation.

## Interface

### Inputs

The function has two explicit inputs.

| Input | Established representation | Use |
|---|---|---|
| First input | One 32-bit value | Not referenced by the supplied function body. |
| Record address | Pointer to storage accessed using 32-bit words and byte offsets | Tested for null, read, passed to dependencies, and used as the source of a later byte read. |

No behavioral restriction on the first input is established beyond its 32-bit representation. It is not forwarded as an explicit argument to either direct dependency or to the indirect operation.

The supplied representation uses 32-bit values as addresses. A contract for another pointer width would require additional evidence; the layout below must not silently expand with a host machine’s native pointer size.

### Record layout

Only two locations in the record are directly accessed by this function:

| Byte offset | Access width | Meaning established by use |
|---|---:|---|
| 0 | 32 bits | Initial object address; zero suppresses further processing. |
| 26, hexadecimal 0x1A | 8 bits | Third logical argument to the indirect operation. |

The function does not establish the meanings of the intervening bytes.

For the nonzero-object path, the record must initially permit the 32-bit read at offset 0. If F-f54286f5d91f returns and execution proceeds to dispatch, the original record address must still permit the byte read at offset 26. An ordinarily contiguous allocation would therefore need to cover at least 27 bytes, although contiguity and allocation boundaries are not independently established by the supplied evidence.

The zero-object path does not directly access offset 26.

### Object and dispatch layout

For successful indirect dispatch, the captured object address participates in two pointer-sized reads:

1. The 32-bit word at object offset 0 supplies a dispatch-table address.
2. The 32-bit word at dispatch-table offset 0 supplies the indirect operation’s entry address.

The indirect operation receives these logical arguments, in order:

1. The object address captured from the record before F-f54286f5d91f.
2. The original record address.
3. The 8-bit value read at record offset 26 after F-f54286f5d91f returns.

The evidence establishes the byte-wide source of the third argument, not its complete machine-level argument-slot representation or a signed interpretation. It should be treated as an opaque 8-bit value unless the indirect target’s interface supplies further meaning.

### Output

No return value is exposed by this function.

A normal return is visible on the zero-object path and after a normally returning indirect operation. The null-record path is annotated as involving a non-returning dependency, but that dependency’s behavior is not independently available.

## Behavior

### Null-record path

The record address is checked for null before the function attempts its initial record read.

If the address is null, the function calls F-a817d8d58c10 with no explicit arguments.

The supplied source includes an annotation that this dependency does not return. This supports an expected terminal path, but the dependency body is absent. The precise result—such as termination, an exception-like transfer, or another nonlocal exit—is unknown.

There is no explicit return or safe fallback following that call in the supplied body. If F-a817d8d58c10 unexpectedly returns normally, execution proceeds toward a 32-bit read through the null record address. The contract therefore does not promise a graceful return for a null record.

Neither F-f54286f5d91f nor the indirect operation is reached on the expected non-returning null-record path.

### Initial object capture

For a non-null record address, the function reads the 32-bit word at offset 0 and retains that value as the object address.

This initial read occurs before F-f54286f5d91f is invoked. The object address used later is the captured value, not a fresh read of record offset 0.

If the captured value is zero, the function returns immediately. In that case:

- F-f54286f5d91f is not called.
- No indirect operation is invoked.
- The byte at record offset 26 is not read by this function.
- No object or dispatch-table dereference occurs.

The null test does not validate any nonzero object address. A nonzero but invalid address follows the same control-flow decision as a valid one.

### Preparation dependency

For a nonzero captured object address, the function calls F-f54286f5d91f with the original record address as its sole explicit argument.

The supplied body imposes no visible restriction on that dependency’s effects. It must not be assumed to preserve:

- The contents of the record.
- The object referenced by the initial record word.
- The dispatch-table pointer stored in the object.
- The dispatch-table contents.
- Global state.
- Allocation lifetimes or ownership relationships.

The call’s result is not consumed as a value.

If the dependency does not return normally, the supplied body establishes no subsequent dispatch or normal return.

### Indirect dispatch

After F-f54286f5d91f returns, the function forms an indirect call using the object address captured before that call.

The dispatch table and its first entry are obtained through that captured object address. Their contents are read after the preparation dependency has returned; they are not visibly cached beforehand.

The byte at record offset 26 is also read after the dependency returns. Consequently, a change to that byte by F-f54286f5d91f affects the value passed to the indirect operation.

The established ordering is:

1. Null-check the original record address.
2. Read and retain the initial object address.
3. Return if that address is zero.
4. Call F-f54286f5d91f.
5. After its normal return, obtain the indirect target and read the byte argument.
6. Invoke the indirect operation.
7. Return if that operation returns normally.

The supplied expression does not establish a relative order between reading the byte argument and resolving the indirect target. Within target resolution, obtaining the dispatch-table address necessarily precedes reading its first entry. No stronger ordering between the independent byte and target accesses should be required.

The function does not recheck record offset 0 after preparation. If that field changes to zero or to another object address, the current invocation still uses the original captured address.

## Side effects

The supplied body contains no explicit writes to the record, object, dispatch table, or global storage.

That absence does not make the function side-effect-free. Its dependencies can perform writes and other externally visible actions:

- F-a817d8d58c10 handles the null-record path.
- F-f54286f5d91f receives access to the original record.
- The indirect operation receives both the captured object address and the original record address.

The function itself establishes no allocation, deallocation, ownership acquisition, ownership release, or lifetime extension. It also establishes no rollback if a dependency changes state and a later access fails.

For the indirect path to complete successfully, the relevant record byte, captured object, dispatch table, and callable entry must remain usable when accessed. If preparation invalidates any of them, this function provides no visible recovery mechanism.

There is no visible locking, atomic access, or synchronization. Concurrent mutation is not given a defined consistency model by this evidence.

## Dependencies

### F-a817d8d58c10

- **Invocation condition:** The original record address is null.
- **Explicit arguments:** None.
- **Return value:** None consumed.
- **Established role:** Receives control before any attempted record read on the null path.
- **Inferred expectation:** Does not return normally, based on the supplied annotation.
- **Unknowns:** Exact failure mechanism, external effects, global-state changes, and whether the annotation accurately describes every execution.

### F-f54286f5d91f

- **Invocation condition:** The record address is non-null and its initially read 32-bit word is nonzero.
- **Explicit argument:** Original record address.
- **Return value:** None consumed.
- **Ordering:** Called after capturing the object address and before resolving the indirect operation or reading the byte argument.
- **Unknowns:** Memory mutations, lifetime effects, ownership behavior, global-state changes, and failure or non-returning behavior.

### Runtime-selected indirect dependency

Its opaque function ID is unresolved in the available evidence.

- **Selection:** First entry in the dispatch table referenced by the first word of the captured object.
- **Arguments:** Captured object address, original record address, post-preparation 8-bit record value.
- **Return value:** Not exposed by this function.
- **Unknowns:** Operation semantics, calling-convention details beyond the visible logical arguments, ownership effects, state changes, and error behavior.

No dependency is assumed to preserve globals or to own, retain, or release any supplied storage.

## Acceptance examples

These are conditional behavioral criteria, not executed tests.

### Non-null record with a zero initial word

Given a readable record whose word at offset 0 is zero:

- The function returns normally.
- Neither direct dependency is called.
- No indirect operation is called.
- Offset 26 need not be readable for this function’s direct accesses.

Changing only the unused first input does not change this path.

### Preparation changes the byte argument

Given a record initially referencing a valid object, suppose F-f54286f5d91f changes the byte at offset 26 from 0x12 to 0xE7 and returns.

If dispatch storage remains usable, the indirect operation receives the 8-bit value 0xE7, not 0x12. The contract does not additionally prescribe signed extension or upper argument-slot bits.

### Preparation replaces the record’s object reference

Given that record offset 0 initially identifies object A, suppose preparation replaces that word with the address of object B and returns.

If A remains usable, dispatch still proceeds through A. The indirect operation receives A as its first argument and the original record address as its second argument. The newly stored address of B is not used to select this invocation’s object.

### Preparation changes the dispatch selection

Given an initial object A, suppose preparation changes A’s dispatch-table pointer, or changes the first entry of its table, while leaving the required storage usable.

The later dispatch uses the values obtained after preparation. Selection is not fixed merely by the initial capture of A’s address.

### Preparation returns after invalidating required storage

Suppose preparation invalidates A or makes record offset 26 unreadable, then returns.

A safe normal return is not promised. The function has no visible lifetime validation or fallback before its subsequent accesses. The exact failure outcome is documented_only.

### Null record

Given a null record address, F-a817d8d58c10 is invoked before a record dereference.

If that dependency does not return, no later operation in this function occurs. If it returns normally, the supplied body does not provide a safe null-handling continuation.

## Uncertainties

The following remain documented_only:

- The semantic meaning of the first input.
- The complete record and object layouts.
- The meanings of the byte at offset 26 and dispatch-table entry zero.
- The exact calling convention and argument extension rules.
- Memory byte order and alignment requirements.
- The behavior and side effects of both direct dependencies.
- The identity and behavior of the indirect dependency.
- Whether the non-returning annotation is universally accurate.
- Ownership, retention, release, and destruction semantics.
- Fault behavior for invalid addresses or non-callable entries.
- Concurrency guarantees and device-memory access semantics.
- The relative evaluation order of the byte read and indirect-target resolution.

There is no size arithmetic, iteration, or counter manipulation in the supplied body. The relevant width constraints are the 32-bit words used for address-bearing fields and the 8-bit read at offset 26. No widening of those fields is justified by the evidence.

## Evidence

This contract is based solely on static inspection of the supplied function body and the provided mapping to opaque dependency IDs.

Established evidence includes the null branch, initial 32-bit object capture, zero-object early return, call to F-f54286f5d91f, post-call dispatch through the captured object, byte read at offset 26, and absence of explicit stores or a returned value.

The non-returning characterization of F-a817d8d58c10 comes from an annotation, not an inspected dependency implementation. Preparation, cleanup, destruction, and ownership interpretations are not established facts.

No runtime tests have been performed. The acceptance examples describe required observations under their stated preconditions; they do not claim observed execution results.
