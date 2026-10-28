<!-- Trusted provenance: F-be7d4809f6cf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-be7d4809f6cf

## Purpose

F-be7d4809f6cf coordinates up to two indirect calls through a caller-supplied object and returns two 32-bit results packed into one 64-bit value.

The first indirect call produces a 32-bit result. A result whose bit pattern is exactly `0xFFFFFFFF` suppresses the second call. Every other result, including zero and other negative signed values, causes a second indirect call.

The returned low 32 bits always contain the first call’s result. The returned high 32 bits contain either the original second input or the second call’s result, depending on whether the second call occurred.

This is an established description of the supplied static representation. The meanings of the object, its dispatch entries, the supplied values, and the returned words are not established. In particular, interpreting the first result as an index, handle, status, or error code would be inferred intent rather than a demonstrated contract.

## Interface

### Inputs

The function receives four inputs:

| Input | Representation | Established use |
|---|---|---|
| Object reference | Pointer to storage whose initial word is represented as a 32-bit integer | Supplies the dispatch-table base and is passed to both indirect calls |
| Initial value | 32-bit word | Initializes writable temporary storage and supplies the high return word when the second call is skipped |
| Follow-up value | 32-bit word | Passed to the second indirect call, if that call occurs |
| Companion value | 32-bit word | Initializes a temporary word immediately following the temporary initialized from the initial value in the represented stack layout |

No signedness is established for the three value inputs. Their 32-bit bit patterns are the relevant contract.

The first indirect call’s result is represented as a 32-bit signed integer because it is compared with signed negative one. This does not establish that all possible results have a signed semantic meaning.

### Output

The function returns a 64-bit value composed of two 32-bit words:

- **Low word:** the first indirect call’s result, preserving all 32 bits.
- **High word:** the original initial value if the low word is `0xFFFFFFFF`; otherwise, the second indirect call’s 32-bit result.

Equivalently, the result consists of the high word shifted left by 32 bits, combined with the zero-extended low-word bit pattern. This describes bit placement, not a signed arithmetic conversion.

Consequently, a negative first result does not sign-extend into the high word. The high word is independently selected.

The source representation does not establish whether callers interpret the complete 64-bit value as a number, a pair of fields, or an ABI-level composite return.

### Required validity

For a defined normal execution, the object reference must support the dispatch accesses actually reached:

1. Reading its initial 32-bit word.
2. Resolving the first callable entry at byte offset `0x4C` from the resulting dispatch base.
3. If the first call returns anything other than `0xFFFFFFFF`, reading the object’s initial word again and resolving the callable entry at byte offset `0x3C` from that base.

The resolved entries must be callable using the argument and result conventions described below. Their own preconditions remain unknown.

No explicit validation of the object reference, dispatch base, or callable entries appears in the supplied representation.

## Behavior

### Temporary input preparation

Before the first indirect call, the function initializes two local 32-bit words:

- The first receives the initial value.
- The following word receives the companion value.

The represented stack offsets place the companion word four bytes after the first word. The first indirect call receives the address of the first word.

This supports a two-word temporary layout in the supplied representation:

| Relative byte offset | Initial contents |
|---|---|
| `0` | Initial value, 32 bits |
| `4` | Companion value, 32 bits |

Only the address of the first word is explicitly passed. The supplied function does not demonstrate which bytes the callee reads or writes, nor does it independently establish a declared two-element array or record type.

These temporary words are initialized copies. The function is not passing the address of the caller’s original input storage.

### First indirect call

The function reads the object’s initial word as the dispatch base and selects the callable entry at byte offset `0x4C`.

It invokes that entry with:

1. The object reference.
2. The address of the first temporary word.

The first call’s 32-bit return value is retained for the eventual low return word and for deciding whether to invoke the second dependency.

The order in which the two temporary initialization stores occur is not an externally demonstrated synchronization guarantee. Both are initialized before control enters the first dependency.

### Sentinel decision

After the first indirect call returns normally, the function selects the original initial value as the default high return word.

It then compares the first result against negative one, whose 32-bit bit pattern is `0xFFFFFFFF`.

- If the result is `0xFFFFFFFF`, the second call is skipped.
- Otherwise, the second call occurs.

There is no demonstrated range check, positivity requirement, zero check, retry, or alternate sentinel.

The original initial value is used for the skipped-call result even if the first dependency changed the temporary word that originally held a copy of that value. The shown function does not read that temporary word back to form either return word.

### Second indirect call

When the first result is not `0xFFFFFFFF`, the function again reads the object’s initial word and resolves the callable entry at byte offset `0x3C`.

This second dispatch lookup occurs after the first call. It is not shown as a lookup cached before the first call. If the first dependency validly changes the object’s dispatch base, the second lookup uses the base then present.

The second call receives:

1. The object reference.
2. The first call’s 32-bit result.
3. The follow-up value.

The second call’s 32-bit return value replaces the default high return word. Its value is not checked against any sentinel.

### Return construction

On normal completion, the function combines the selected high word with the retained first result.

No additional call, output write, or result normalization is shown after that combination.

A second-call result of `0xFFFFFFFF` is preserved as the high word. It does not trigger a retry or cause the original initial value to be restored.

## Side effects

### Direct effects

The function directly:

- Initializes local temporary storage.
- Reads the object’s dispatch-base word.
- Reads the first dispatch entry.
- Conditionally rereads the object’s dispatch-base word and reads the second dispatch entry.
- Transfers control to one or two indirect dependencies.
- Returns the packed 64-bit result.

No direct write to the object, its dispatch table, or caller-provided output storage appears in the supplied representation.

### Dependency effects

Both indirect dependencies receive the object reference. The first also receives writable local storage.

Their effects are not supplied. They may change object state or other accessible state according to contracts not available here. No guarantee is established that they preserve globals, preserve the dispatch base, avoid allocation, or leave ownership unchanged.

The function performs no explicit allocation, deallocation, reference-count adjustment, or ownership transfer. That observation does not rule out such actions inside a dependency.

### Temporary lifetime

The passed temporary address refers to local storage belonging to this invocation. The function does not establish storage that remains valid after it returns.

Whether the first dependency retains that address is unknown. Retention cannot be treated as granting a longer lifetime. Any use after the storage’s lifetime ends would require a separate validity guarantee absent from the supplied evidence.

### Failures and abnormal completion

There is no shown local error handler, fallback for an invalid dispatch entry, or cleanup path around either call.

The only demonstrated branch that suppresses the second call is the first result equal to `0xFFFFFFFF`. Calling that value an “error” would be an interpretation.

If a dependency fails to return normally, this function’s normal packed return is not established. The supplied representation does not specify exception propagation, traps, process termination, or recovery.

## Dependencies

The following opaque identifiers are specification-local labels for unresolved indirect dependencies. They do not assert resolved implementation identities.

| Opaque ID | Selection | Inputs | Result |
|---|---|---|---|
| F-indirect-004c | Dispatch entry at byte offset `0x4C`, using the object’s initial word before the first call | Object reference; address of initialized temporary storage | 32-bit result |
| F-indirect-003c | Dispatch entry at byte offset `0x3C`, using the object’s initial word after the first call | Object reference; first result; follow-up value | 32-bit result |

F-indirect-004c is invoked once on every normal path reaching dependency execution. F-indirect-003c is invoked once only when the first result differs from `0xFFFFFFFF`.

These identifiers describe call roles, not necessarily distinct target functions. The supplied representation does not establish whether either entry aliases another entry or resolves differently between invocations.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Each assumes valid accesses, compatible callees, and normal dependency returns.

### Sentinel preserves the original initial value

Given:

- Initial value: `0x12345678`.
- First result: `0xFFFFFFFF`.

Required outcome:

- F-indirect-003c is not called.
- The returned value is `0x12345678FFFFFFFF`.

If F-indirect-004c changes the first temporary word to `0xAABBCCDD`, that mutation does not change this expected return.

### Zero invokes the second dependency

Given:

- First result: `0x00000000`.
- Follow-up value: `0x11223344`.
- Second result: `0x89ABCDEF`.

Required outcome:

- F-indirect-003c receives zero as its second argument and `0x11223344` as its third.
- The returned value is `0x89ABCDEF00000000`.

Zero is not a skip condition.

### A negative nonsentinel result still invokes the second dependency

Given:

- First result: `0xFFFFFFFE`, interpreted as signed negative two.
- Second result: `0x00000007`.

Required outcome:

- The second dependency is invoked.
- The returned value is `0x00000007FFFFFFFE`.

The low word does not sign-extend into the high word.

### The second result has no sentinel handling

Given:

- First result: `0x00000005`.
- Second result: `0xFFFFFFFF`.

Required outcome:

- The returned value is `0xFFFFFFFF00000005`.
- No further call is made by this function.

### Dispatch changes affect the second lookup

Given that F-indirect-004c validly changes the object’s initial dispatch-base word and returns a nonsentinel result:

- The second entry is resolved relative to the changed base.
- The function does not use a previously cached second entry.

This case is conditional on the changed object and dispatch storage remaining valid.

### Companion initialization is observable to a compatible dependency

If F-indirect-004c is permitted to read a 32-bit word four bytes after its temporary-storage argument, it observes the supplied companion value there before making its own changes.

This does not establish permission to access additional storage beyond the represented two-word region.

## Uncertainties

The following remain **documented_only**:

- The semantic types and meanings of all input and output words.
- Whether the sentinel denotes failure, absence, completion, or another condition.
- The object’s full layout and the dispatch table’s extent.
- The exact machine ABI, pointer width, calling convention, and alignment requirements. The representation uses 32-bit words for dispatch-base computation and pointer-like operations, but does not independently establish the complete target ABI.
- Byte order. The high/low return-word relationship is established independently of memory endianness.
- The first dependency’s intended temporary-storage schema and permitted access extent.
- The dependencies’ object mutations, global effects, memory ownership behavior, and failure mechanisms.
- Whether concurrent modification, reentrant calls, or asynchronous observation are supported.

An index-or-status operation followed by a related lookup is a possible interpretation of the call structure. It is not required behavior and supplies no additional validation or ownership guarantees.

## Evidence

The available evidence is a single supplied static function representation.

It establishes two indirect dispatch offsets, the arguments supplied at each call site, a comparison of the first 32-bit result with negative one, conditional execution of the second call, and a 64-bit concatenation with the first result in the low word.

The represented local offsets support the stated adjacent temporary-word layout. The function shows no readback of those words after the first call.

No dependency bodies, caller contracts, runtime traces, executable observations, or tests were provided. No runtime tests have been performed.
