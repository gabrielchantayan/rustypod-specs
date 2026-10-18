<!-- Trusted provenance: F-e3cb6fc84b48; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e3cb6fc84b48

## Purpose

F-e3cb6fc84b48 performs one indirect dispatch through a table associated with an embedded receiver location. It packages a supplied 32-bit value into temporary storage and passes that storage, together with the embedded receiver address, to the selected callable.

The established behavior is a forwarding operation, not a demonstrated application-level action. The supplied evidence does not establish what the selected callable does, what the forwarded value represents, or whether the operation is intended as a notification, command, or setter.

The wrapper has no explicit result value. Its principal observable effect is the indirect call and whatever effects that call produces.

## Interface

### Inputs

Two inputs are present:

| Input | Established representation | Role |
|---|---|---|
| Base address | A 32-bit integer-shaped address in the supplied representation | Identifies the storage from which the embedded receiver address is derived. |
| Forwarded value | One 32-bit word | Copied into the first word of temporary argument storage. No conversion, interpretation, or validation is shown. |

The address representation is not evidence that arbitrary integers are valid inputs. Successful execution requires an addressable object layout and a usable indirect-call target.

### Required layout

Let the base address be **B**. The required receiver location is:

**R = B + 0x34**, or 52 bytes after the base.

The wrapper reads a 32-bit word at **R** and treats that word as a dispatch-table address, **T**. It selects the callable stored at:

**T + 0x1c**, or 28 bytes after the table address.

These offsets are byte offsets. In the represented 32-bit layout, the selected entry occupies one 4-byte pointer-sized slot. The evidence does not establish the contents, number, or meaning of other table entries.

The receiver passed to the callable is **R**, the address of the embedded location. It is not **B**, and it is not the table address read from **R**.

### Temporary argument layout

The wrapper provides temporary storage containing two consecutive 32-bit words:

| Offset from temporary-storage address | Size | State before dispatch |
|---|---:|---|
| 0 bytes | 4 bytes | Contains the forwarded value. |
| 4 bytes | 4 bytes | No initialization is shown. |

The callable receives the address of this temporary storage, not the forwarded word as a standalone scalar argument.

Only the first word has a defined value established by this function. The existence of the second word must not be interpreted as a guarantee that it contains zero, another input, or any stable value.

### Output

No explicit return value is provided by the wrapper. Any return value produced by the indirect callable is unused.

The source representation does not establish the complete machine-level calling convention, register effects, or binary interface beyond the visible two-argument indirect call and the widths described above.

## Behavior

### Established execution sequence

The wrapper first stores the forwarded 32-bit value in the first word of its temporary argument storage.

It then obtains the dispatch-table address from the 32-bit word at the embedded receiver location, 52 bytes after the supplied base. Using that table address, it obtains the callable from the entry 28 bytes into the table.

The wrapper invokes the selected callable exactly once on the ordinary execution path, supplying these arguments in this order:

1. The embedded receiver address.
2. The address of the temporary two-word storage.

If the callable returns normally, the wrapper returns without further explicit work.

This sequence establishes a dependency between the dispatch reads: the table address must be obtained before the callable entry can be located. The temporary value is initialized before the call. The supplied source does not establish a broader hardware memory-ordering guarantee or synchronization protocol.

### Value handling

The forwarded value is copied as a 32-bit word. No masking, sign extension, byte swapping, range check, or semantic conversion is visible.

Accordingly, all 32-bit patterns are treated alike by the wrapper itself. Any distinction between signed values, unsigned values, flags, identifiers, or pointer-shaped values belongs to the unknown callable or to an external contract not supplied here.

### Dispatch behavior

The target is selected from the table associated with the supplied receiver location. No fixed callable identity is established.

There is no explicit null check, target validation, fallback dispatch, retry, conditional branch, or error translation in the wrapper. There is also no visible second dispatch after the callable returns.

The visible operations do not reread the table or selected entry after the call. This does not constrain what the callable itself may read or modify.

### Address arithmetic and preconditions

The relevant widths are 32 bits for the supplied address representation, stored table address, callable entry representation, and forwarded value. The offsets are 52 and 28 bytes; the temporary storage comprises 8 bytes.

A successful execution requires at least:

- A base for which the embedded receiver location can be formed and read.
- A readable table address at that location.
- A readable callable entry at the required table offset.
- A callable target compatible with the two supplied arguments.
- Any additional validity conditions imposed by that target.

Overflow, address wrapping, alignment requirements, and faults are not handled explicitly. The evidence does not justify choosing either signed-overflow semantics or modulo-address arithmetic for invalid boundary inputs. Such inputs have no established successful behavior.

## Side effects

### Direct effects

The wrapper directly initializes the first word of its temporary storage and performs the dispatch-related reads. It shows no direct write to the base object or dispatch table.

It performs no explicit heap allocation, explicit deallocation, global-state update, or external I/O operation.

These statements apply only to the visible wrapper operations. They do not establish that the complete call is allocation-free, side-effect-free, or nonblocking.

### Effects delegated to the callable

The selected callable receives a receiver address and a writable address to temporary argument storage. Its behavior is unknown. It may change receiver state, other reachable memory, globals, or the temporary words; it may also perform external operations.

The wrapper does not inspect the temporary storage after the call. Consequently, modifications to that storage are not converted into an explicit wrapper result.

No assumption is established that the callable preserves globals, preserves the receiver’s dispatch-table field, or leaves the supplied value unchanged.

### Ownership and lifetime

The wrapper passes addresses but performs no explicit ownership transfer or release.

The temporary storage is local to this invocation. Its address is available during the indirect call; the wrapper establishes no valid lifetime for using that address after it returns. A retained pointer must not be treated as a durable argument object merely because the callable received it.

Ownership of the base object and its embedded receiver is unspecified. The wrapper neither visibly acquires nor releases that ownership. Whether the callable may destroy, release, retain, or otherwise affect the receiver is unknown.

### Failures and abnormal completion

There is no explicit error result or recovery path. Invalid receiver storage, an invalid table address, an unusable entry, or an incompatible callable can prevent successful completion, but the precise failure mechanism is not established.

Likewise, if the callable does not return normally, the wrapper has no demonstrated mechanism to recover and complete its ordinary return path.

## Dependencies

### F-indirect-01

**F-indirect-01** is a document-local opaque reference for the unresolved callable selected from the dispatch table. It is not an established identity shared by every invocation.

Its observed invocation contract is:

- First argument: the address 52 bytes after the supplied base.
- Second argument: the address of two consecutive 32-bit temporary words.
- Defined initial argument content: the first temporary word equals the supplied forwarded value.
- Second temporary word: initialization not established.
- Callable result: ignored by F-e3cb6fc84b48.

No dependency body or external contract was supplied. Its semantic action, memory ownership rules, global-state effects, error behavior, and retention behavior remain unknown.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

### Valid dispatch

Given a readable base object whose location at offset 52 contains a valid table address, and a compatible callable entry at table offset 28:

- The selected callable is invoked once.
- Its first argument is the base address plus 52 bytes.
- Its second argument points to temporary storage whose first 32-bit word equals the forwarded value.
- If the callable returns normally, the wrapper returns without an explicit result.

### Distinguishing receiver addresses

Given a valid object and table, an implementation that passes the outer base address instead of the embedded address does not satisfy the contract. Passing the table address as the receiver also does not satisfy it.

### Preserving the forwarded bits

Given the forwarded value `0x80000001`, the first temporary word presented to the callable must contain that same 32-bit pattern. No interpretation as a negative number or special flag combination is required by this wrapper.

### Using the selected table entry

Given a valid table with different callable entries at different offsets, dispatch must use the entry 28 bytes into that table. Another entry is not interchangeable merely because it accepts the same arguments.

### Callable return value

Given a callable that returns a status value and otherwise returns normally, the wrapper does not expose that status as its own explicit return value.

### Uninitialized trailing word

A consumer requirement that the second temporary word be zero is not supported. Acceptance criteria must not assert any particular value for that word.

### Invalid dispatch data

Given a null or unreadable dispatch component, no graceful rejection, no-op, fallback, or error code is established. An acceptance example cannot require one without additional evidence.

## Uncertainties

The following remain **documented_only**:

- The application-level purpose of the call and meaning of the forwarded word.
- Whether the embedded location belongs to a class, interface, or another structure.
- The identity and full contract of the selected callable.
- Whether the callable reads the second temporary word.
- Ownership or release behavior performed by the callable.
- Concurrency, synchronization, and reentrancy requirements.
- Exact ABI details and behavior for malformed addresses, arithmetic boundaries, or incompatible targets.

A table-based receiver dispatch is consistent with object-oriented method invocation, but that is inferred intent rather than a requirement established by the evidence.

## Evidence

The supplied static function body establishes the 32-bit value copy, the two-word temporary storage, receiver offset 52, table-entry offset 28, two-argument indirect invocation, ignored callable result, and normal return without further explicit operations.

No surrounding type definitions, caller contracts, dependency implementations, or runtime observations were supplied. No runtime tests have been performed. The contract therefore distinguishes visible wrapper behavior from dependency effects and does not claim guarantees for either unknown callees or invalid inputs.
