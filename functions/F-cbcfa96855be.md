<!-- Trusted provenance: F-cbcfa96855be; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cbcfa96855be

## Purpose

F-cbcfa96855be passes an input value through an embedded receiver component, queries that component, writes a transformed one-byte result into the receiver, and invokes a receiver-selected callback.

Its established normal-path behavior is:

- Prepare two local 32-bit values from the primary and auxiliary inputs.
- Invoke F-46dc1d40e2e7 with the embedded component address and the address of the primary temporary.
- Invoke F-92e8c88e6cb8 with the same component address.
- Toggle bit zero of the query’s byte result and store that byte in the receiver.
- Invoke a callback with the receiver, a value read from a global data operand, the current primary temporary’s low eight bits, and zero.
- Return a 64-bit value containing the receiver’s 32-bit address representation in its upper half and one in its lower half.

An apparent intent is to process an input, update a status field, and notify the receiver. That interpretation is inferred rather than established. In particular, the status transformation is a bit toggle, not logical negation.

The constant one in the normal return does not establish validated success. The function does not visibly inspect either the first dependency’s result or the callback’s result, and it has no explicit failure-return branch.

## Interface

### Inputs

| Input | Established representation | Explicit use |
|---|---|---|
| Receiver | Pointer used in a 32-bit address model with four-byte element indexing | Locates the component, status byte, and dispatch-table reference |
| Primary value | Unsigned 32-bit value | Initializes a writable temporary whose address is passed to F-46dc1d40e2e7 |
| Auxiliary value | Opaque 32-bit value | Initializes a second local temporary; no subsequent explicit read is shown |

The receiver-relative layout used by this function is:

| Byte offset | Access |
|---|---|
| 0 | Read of a 32-bit dispatch-table address during callback selection |
| 72, or 0x48 | Address supplied to both direct dependencies |
| 158, or 0x9e | One-byte write of the transformed query result |

The component address is the receiver address plus eighteen four-byte elements, totaling 72 bytes. The function passes this interior address directly; it does not load a component pointer from that location.

The callback target is selected through an entry at byte offset 16, or 0x10, from the dispatch-table address read from the receiver. The complete table layout and callback calling convention are not established.

### Global input

The callback’s second argument is the value read from the global data operand identified at 0x082581ec.

This is a by-value use of that operand, not an address-of operation and not a directly represented constant argument. Its stored value, width, type, and semantic role are not established by the supplied representation.

The global read belongs to the callback expression, which occurs after both direct dependency calls and the status-byte store. The function must not be described as capturing this value at entry.

### Output

On normal completion, the function returns a 64-bit packed value:

- Bits 63–32 contain the receiver’s 32-bit address representation.
- Bits 31–0 contain one.

For a receiver address representation of 0x12345678, the returned bit pattern is 0x1234567800000001.

This is concatenation of two 32-bit portions, not addition of one to the receiver address. The representation alone does not establish whether callers interpret it as an aggregate, a status-and-receiver pair, or another abstraction.

### Preconditions

Normal completion requires valid storage and callable targets for every operation that actually occurs:

- The component address must meet both direct dependencies’ requirements.
- The byte at receiver offset 158 must be writable when the store occurs.
- The receiver’s first word must be readable and contain a usable dispatch-table address when callback selection occurs.
- The dispatch-table entry at offset 16 must be readable and identify a compatible callable target.
- The global operand must be readable when the callback expression evaluates it.
- Receiver and temporary storage must remain usable for accesses performed by this function and its callees.

No explicit null check, bounds check, dispatch-target check, global-validity check, or lifetime check is visible.

The representation uses 32-bit address operations. Behavior for addresses outside that model, or address calculations that do not identify valid required storage, is not established.

## Behavior

### Local preparation

Before the first dependency call, the function initializes a writable 32-bit temporary from the primary value. It also initializes a second 32-bit temporary from the auxiliary value.

Only the primary temporary’s address is explicitly passed onward. No later explicit read of the auxiliary temporary appears.

The displayed storage representation suggests adjacent local words. It does not establish that the first dependency receives an eight-byte argument object or is authorized to access both words. The auxiliary value’s purpose and any dependency access to it remain documented_only.

### First dependency call

The function invokes F-46dc1d40e2e7 with:

1. The receiver’s embedded component address at offset 72.
2. The address of the primary temporary.

No return value from this dependency is explicitly consumed.

Because the primary temporary is addressable and writable, its later value need not equal the original primary input. The callback uses the temporary’s value after the preceding dependency activity, not necessarily the entry input.

The dependency’s implementation is unavailable. No preservation guarantee is established for receiver contents, global data, dispatch information, or the supplied temporary. Whether it retains either address is also unknown.

### Query call

After F-46dc1d40e2e7 returns normally, the function invokes F-92e8c88e6cb8 with the same component address.

An eight-bit result is consumed from this call. If a wider machine-level result exists, the supplied representation establishes only the byte used here.

There is no intervening branch based on the first dependency’s result. The query follows whenever execution returns normally from the first call.

The component address is based on the original receiver address, not on a component pointer loaded from receiver storage. Changes to receiver contents do not by themselves change this address calculation.

### Status-byte write

After the query returns, the function toggles bit zero of the query byte and writes the resulting byte at receiver offset 158.

The transformation preserves bits one through seven:

| Query byte | Explicitly stored byte |
|---|---|
| 0x00 | 0x01 |
| 0x01 | 0x00 |
| 0x02 | 0x03 |
| 0x80 | 0x81 |
| 0xFF | 0xFE |

This is not Boolean normalization. A nonzero query result does not generally produce zero.

The explicit write is one byte wide. It does not directly overwrite neighboring bytes. Effects of the dependencies or callback on neighboring storage are separate and unknown.

### Callback expression

After the status store, the function evaluates an indirect callback expression. That expression obtains the dispatch-table address from the receiver, selects the target at table offset 16, reads the global operand, obtains the primary temporary’s masked value, and invokes the selected target.

The positional arguments are:

| Position | Value |
|---|---|
| First | The receiver |
| Second | The value read from the global data operand at 0x082581ec |
| Third | The current unsigned 32-bit primary temporary masked to its low eight bits |
| Fourth | Zero |

The third argument is numerically in the range 0 through 255. The mask does not establish a signed eight-bit parameter type or a one-byte machine-level argument slot.

The second argument is the global operand’s value, not the address of its storage. That value could itself have an address-like meaning, but its type and meaning are unknown.

Both direct dependencies execute before this global read. They are not known to preserve the global. Consequently, an entry-time cached value is not justified: a dependency’s change to the global can affect the callback’s second argument.

Likewise, callback selection uses a receiver dispatch-table reference read in the expression after both dependencies and the status store. The evidence does not justify using a dispatch-table reference cached at entry.

The expression’s placement is established, but its internal evaluation order is not. In particular, the representation does not establish a complete order among:

- Reading the receiver’s dispatch-table reference.
- Loading the callback target.
- Reading the global operand.
- Reading and masking the primary temporary.
- Evaluating the other arguments.

No callback result is explicitly consumed.

### Normal return and interrupted execution

If the callback returns normally, the function returns the packed 64-bit value described above. There is no subsequent explicit status-byte write.

The callback can therefore change the status field after the function’s own write. The contract requires the earlier write, not that its value remain present at return.

The packed return preserves the receiver’s address representation; it does not prove that receiver storage remains alive after the callback.

No explicit error path, rollback, exception handler, or recovery operation is shown. If a dependency or callback does not return normally, subsequent operations and the normal return are not established. The failure mechanisms themselves remain unknown.

## Side effects

The function’s explicit effects are:

- Initialization of two local 32-bit temporaries.
- An ordered call to F-46dc1d40e2e7.
- A subsequent call to F-92e8c88e6cb8.
- One byte written at receiver offset 158.
- Reads for callback selection and argument formation, including the global operand read.
- One indirect callback invocation.

The function does not explicitly write the global operand. Its value can nevertheless change through callee effects or surrounding activity.

Neither dependency nor the callback is known to preserve global state, receiver fields, dispatch data, resources, or object lifetime. Their effects must not be reduced to the explicit operations performed by this caller.

The primary temporary belongs to this invocation’s local storage. Passing its address does not establish an extension of its lifetime. If a callee retains that address, the evidence does not establish safe deferred use.

No explicit allocation, deallocation, reference-count operation, or ownership transfer appears. This does not imply that the callees perform none.

There is no visible synchronization or rollback. Atomic publication, thread safety, freedom from data races, and restoration after interrupted execution are not established.

## Dependencies

### F-46dc1d40e2e7

Receives the embedded component address and the address of the primary temporary.

Established observations:

- It executes before the query.
- Its return value is not explicitly used.
- It can influence later behavior through changes to the supplied temporary or other accessible state.
- No guarantee establishes that it preserves the global operand used by the callback.

Unknowns include its accepted memory extent, validation, ownership rules, retained references, failure behavior, and any use of the auxiliary temporary.

### F-92e8c88e6cb8

Receives the embedded component address after the first dependency returns.

Established observations:

- Its consumed result is eight bits.
- That result determines the explicit status write through a bit-zero toggle.
- It executes before callback selection and the callback argument’s global read.

A Boolean result restriction is not established. Nor is preservation of the primary temporary, receiver state, or global data.

### Receiver-selected callback

The callback target comes from the receiver’s dispatch-table entry at byte offset 16.

It receives the receiver, the value read from the global operand, the masked primary temporary, and zero. Its return value is ignored by this function.

Its parameter types, calling convention, side effects, lifetime effects, ownership rules, and failure behavior are unknown. No opaque function ID for this dynamically selected target is supplied.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

### Unchanged primary temporary

Assume the primary input is 0x123456AB, neither dependency changes the primary temporary, the query returns zero, required storage is valid, and the callback returns normally.

Required observations:

- F-46dc1d40e2e7 precedes F-92e8c88e6cb8.
- The explicit status store writes one.
- The callback follows that store.
- Its third argument is 171, or 0xAB.
- Its second argument comes from the global read in the callback expression.
- The normal return’s lower 32 bits equal one.

### Primary temporary changed by processing

Assume F-46dc1d40e2e7 changes the primary temporary to 0x87654321 and nothing subsequently changes it before argument evaluation.

The callback’s third argument must be 33, or 0x21. Forwarding the original input’s low byte would not match the established behavior.

### Non-Boolean query result

Assume the query returns 0x80.

The explicit status write must store 0x81. Storing zero because the result is nonzero would substitute logical negation for the established bit toggle.

### Global changed by a dependency

Assume the global operand contains value A at function entry. One dependency changes it to value B, and it remains B through the callback expression’s read.

The callback’s second argument must be B. Passing A from an entry-time cache would not preserve the established read placement.

This case makes no assumption about the global’s type or access width. A and B denote distinguishable values valid for that operand’s actual representation.

### Dispatch changed by a dependency

Assume a dependency validly replaces the receiver’s dispatch-table reference, and that replacement remains in place until callback selection.

The callback target must be selected through the later reference, not a reference captured at function entry.

### Callback changes the status field

Assume the query returns one, causing the explicit store to write zero. The callback then writes 0x7F to that field and returns normally.

A final field value of 0x7F is consistent with the function. The required zero write occurs before callback entry; no restoration follows.

### Packed return

Assume the receiver’s address representation is 0x12345678 and execution completes normally.

The return must have bit pattern 0x1234567800000001, regardless of an otherwise ignored callback result.

### Invalid required storage

Assume a required receiver, dispatch-table, component, or global access cannot be performed validly.

No defined fallback or error return is established. The visible function provides no guard for that condition.

## Uncertainties

The following remain documented_only:

- Semantic identities of the receiver, component, and status field.
- The global operand’s value, type, width, and meaning. Its by-value read is established; whether that value denotes a pointer, identifier, or something else is not.
- Evaluation order within the callback expression.
- Whether the query result is contractually restricted to zero and one.
- The auxiliary input’s purpose and whether the first dependency intentionally accesses adjacent temporary storage.
- Complete callee signatures and calling conventions.
- Ownership, retention, release, and lifetime effects of all calls.
- Exceptions, nonlocal exits, termination, and other callee failure behavior.
- Alignment, aliasing, concurrency, and external synchronization guarantees.
- Caller interpretation of the packed return.

The notification-style purpose is inferred intent. The call order, offsets, byte transformation, global read placement, argument mask, and return composition are established static observations.

## Evidence

This contract is based solely on the supplied function representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent evidence.

The representation establishes two ordered direct calls, a subsequent one-byte store, an indirect callback expression, and a packed 64-bit return. The callback expression uses the identified global data operand by value without an address-of operation. Its placement after the calls and store is established; finer evaluation order within that expression is not.

No dependency implementation, callback implementation, complete receiver layout, global declaration, or caller contract was supplied.

No runtime tests have been performed. The acceptance examples specify conditional outcomes derived from static evidence, not observed executions.
