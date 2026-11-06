<!-- Trusted provenance: F-e8a51cc3cc3b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e8a51cc3cc3b

## Purpose

F-e8a51cc3cc3b conditionally obtains a byte-addressed value and delivers a derived value to an operation on a supplied receiver. A selector byte in a separate object determines which of three lookup bindings is used.

For supported selectors, the function distinguishes three outcomes:

- No lookup value, or a lookup value whose first byte is zero: invoke the receiver’s fallback operation.
- A lookup value whose first byte is nonzero, followed by a zero result from the receiver’s value operation: return without further action.
- A lookup value whose first byte is nonzero, followed by a nonzero result from the receiver’s value operation: invoke an additional indirect operation with that result and the original lookup address.

Any selector other than 1, 2, or 3 causes an immediate return.

These are established control-flow behaviors. Interpreting the lookup value as text, the receiver operations as particular application actions, or the final indirect operation as a resource-management operation would require evidence not supplied here.

## Interface

### Inputs

The function accepts two machine-level arguments:

| Input | Required interpretation |
|---|---|
| Selector-object address | Address of an object containing a readable selector byte at byte offset 124, hexadecimal 0x7C. No other field in this object is directly accessed. |
| Receiver address | Address of an object whose first 32-bit word supplies a dispatch-table address when a receiver operation is invoked. |

The supplied representation uses 32-bit integer and address-sized values. Dispatch entries occur at byte offsets 8 and 12, hexadecimal 0x08 and 0x0C, from the table address. This establishes the access layout used by this function; it does not establish a portable ABI or a complete object type.

The selector is read as one byte. Whether the source-level character type is signed does not change the accepted set: only the byte values 1, 2, and 3 enter the lookup path.

### Receiver layout and operations

When the value operation is reached:

1. The receiver’s first 32-bit word is read as a table address.
2. The callable entry at table byte offset 8 is obtained.
3. The operation is invoked with three arguments: the receiver address, the result from F-8810f53aaa0b, and a zero-valued argument.
4. Its result is interpreted as a 32-bit integer for the subsequent zero/nonzero decision.

When the fallback operation is reached:

1. The receiver’s first 32-bit word is read as a table address.
2. The callable entry at table byte offset 12 is obtained.
3. The operation is invoked with the receiver address as its sole visible argument.

No return value from the fallback operation is consumed.

### Output

The function has no explicit return value. When execution returns normally, no result is conveyed directly to its caller.

Normal return is not guaranteed merely by entering the function: any dependency or indirect target may fail to return, and invalid memory or invalid call targets may prevent completion.

### Path-specific preconditions

The selector-object address must permit the initial one-byte read at offset 124.

For an unsupported selector, the receiver need not be dereferenced by this function. No supported-selector work is performed.

For a supported selector, the receiver must provide valid storage and a callable dispatch entry at the point a receiver operation is invoked. A valid dispatch table at function entry alone is insufficient to establish this condition because dependencies execute before the table is read.

A non-null lookup address must permit a one-byte read. If that byte is nonzero, the address must also satisfy the requirements of F-8810f53aaa0b. Those further requirements are not established by the supplied evidence.

No explicit validation enforces these preconditions.

## Behavior

### 1. Read and classify the selector

The function first reads the selector byte at offset 124 from the selector object.

- Selector 1 selects the first lookup binding.
- Selector 2 selects the second lookup binding.
- Selector 3 selects the third lookup binding.
- Every other byte value causes immediate return.

On the unsupported-selector path, this function performs no dependency calls, no receiver dispatch, and no direct receiver-memory reads.

The selector is not reread later. Changes to the selector after the initial read do not cause the function to choose a different branch during that invocation.

### 2. Obtain a lookup context and the selected binding

For every supported selector, the function calls F-9d0ecc5f6adf with no visible arguments. Its 32-bit result is retained as the first argument for the lookup dependency.

After that call returns, the function reads the value of the binding corresponding to the previously selected branch. The three branches refer to three distinct binding locations. Their stored values are not provided and need not be distinct.

This ordering matters: the binding value is read after F-9d0ecc5f6adf returns, not captured before that call. If the dependency changes the selected binding, the later lookup uses the value read afterward.

The function then calls F-c0b020cd3882 with:

- The result of F-9d0ecc5f6adf.
- The selected binding value.

No null check or other validation is applied to either argument before this call.

### 3. Classify the lookup result

The result of F-c0b020cd3882 is treated as a byte-addressed pointer.

If that address is null, the function does not read through it and proceeds to the fallback operation.

If the address is non-null, the function reads exactly its first byte for the local branch decision:

- A zero first byte selects the fallback operation.
- Any nonzero first byte selects the value operation.

The function itself does not scan for a terminator, calculate a length, inspect a second byte, or validate an encoding. A first byte of 128 or 255 is nonzero and therefore selects the same path as a first byte of 1.

The branch depends on the byte read at this stage. No second emptiness check occurs after subsequent calls.

### 4. Perform the fallback operation

For either a null lookup address or a zero first byte, the function invokes the receiver entry at table byte offset 12, passing the receiver address.

Neither the lookup address nor a transformed value is passed to this operation by the visible call.

After the fallback operation completes normally, F-e8a51cc3cc3b has no further explicit work.

### 5. Produce and deliver the derived value

For a non-null lookup address with a nonzero first byte, the function calls F-8810f53aaa0b with the original lookup address.

Its result is a 32-bit value. The supplied evidence does not establish whether that value represents an address, handle, count, status, or another kind of datum.

After F-8810f53aaa0b returns, the function obtains the receiver operation from table byte offset 8 and invokes it with:

1. The receiver address.
2. The result of F-8810f53aaa0b.
3. Zero.

There is no intervening test of the derived value. In particular, a zero result from F-8810f53aaa0b is still passed to the receiver’s value operation.

The receiver dispatch is resolved after F-8810f53aaa0b runs. The function does not establish that the receiver or its table remains unchanged across that dependency call.

### 6. Process the receiver operation’s result

If the receiver’s value operation returns zero, F-e8a51cc3cc3b returns immediately. It does not invoke the fallback operation or the additional indirect target.

If the receiver’s value operation returns any nonzero 32-bit integer, the function invokes an additional indirect target with two arguments:

1. The nonzero receiver-operation result.
2. The original lookup address returned by F-c0b020cd3882.

Both positive and negative nonzero results take this path. The test is not a positivity check, and no normalization of the result is visible.

The second argument is the original lookup address, not the result of F-8810f53aaa0b. Its storage contents may nevertheless have changed during intervening calls.

The supplied representation identifies this final transfer as an indirect operation but does not establish its complete low-level control-transfer semantics. At the visible behavioral level, no subsequent explicit action is specified.

## Side effects

### Direct memory accesses

The function directly performs the following reads when their paths are reached:

- One selector-byte read from the selector object.
- One read of the selected lookup binding after F-9d0ecc5f6adf returns.
- One first-byte read from a non-null lookup result.
- Reads needed to obtain the applicable receiver dispatch target.
- A read or resolution of the additional indirect target on the nonzero receiver-result path.

No direct write to the selector object, receiver object, lookup storage, or binding storage is visible.

This absence of direct writes is not a guarantee that those objects remain unchanged. Every called dependency or indirect operation may have side effects that are not established here.

### Observable call ordering

For a supported selector with a nonempty lookup result, the established order is:

F-9d0ecc5f6adf; selected binding read; F-c0b020cd3882; first-byte read; F-8810f53aaa0b; receiver value operation; optionally the additional indirect operation.

For a supported selector with a null lookup result, the first-byte read and value-conversion call are skipped before fallback dispatch.

For a supported selector with a zero first byte, the first-byte read occurs, but the value-conversion call and receiver value operation are skipped.

There is no visible retry, alternate-binding search, or recovery call after a nonzero result from the receiver value operation.

### Ownership and lifetime

No explicit allocation, copy, release, or ownership transfer is established by this function’s own operations.

The lookup address is retained across F-8810f53aaa0b and the receiver value operation so that it can be supplied to the additional indirect target if needed. Correct execution therefore requires those intervening operations to leave that later use valid on paths where it occurs. Their actual lifetime and ownership contracts remain unknown.

The additional indirect operation must not be characterized as freeing memory, reporting an error, or releasing a resource solely from its position in this flow.

## Dependencies

| Dependency | Visible use | Not established |
|---|---|---|
| F-9d0ecc5f6adf | Called once with no visible arguments for supported selectors; supplies a 32-bit lookup argument. | Meaning of its result, global changes, failure convention, ownership, or termination behavior. |
| F-c0b020cd3882 | Called once with the preceding result and the selected binding value; its result is treated as a byte pointer. | Storage type, length, encoding, ownership, validity duration, or whether null means absence or failure. |
| F-8810f53aaa0b | Called once only when the lookup address is non-null and its first byte is nonzero; receives that address and supplies a 32-bit argument to the receiver. | Transformation semantics, memory effects, ownership effects, or result meaning. |
| Receiver entry at byte offset 8 | Receives the receiver, derived value, and zero; returns the integer controlling the final branch. | Application meaning, mutation behavior, and whether nonzero denotes success, failure, or another condition. |
| Receiver entry at byte offset 12 | Receives the receiver on null or zero-first-byte lookup results. | Application meaning and side effects. |
| Additional indirect target | Receives the nonzero receiver result and original lookup address. | Opaque identity, full ABI, ownership behavior, and normal-return behavior. |

No opaque function ID is supplied for the receiver entries or the additional indirect target. Their identities are documented_only rather than invented.

## Acceptance examples

These examples are conditional acceptance criteria derived from static evidence, not observations from execution. They assume all reached accesses and calls are valid and complete normally unless stated otherwise.

### Unsupported selector

Given a readable selector byte of 0, 4, or 255:

- The function returns after classifying that byte.
- No lookup dependency or receiver operation is called.
- This path does not require dereferencing the receiver.

### Binding read after context acquisition

Given selector 2, and given that F-9d0ecc5f6adf changes the second binding before returning:

- The later lookup uses the second binding’s value read after that change.
- The branch remains associated with selector 2 even if the selector byte itself was also changed during the call.

### Null lookup result

Given any supported selector and a null result from F-c0b020cd3882:

- No lookup byte is read.
- F-8810f53aaa0b is not called.
- The receiver’s offset-12 operation is invoked once.

### Non-null result beginning with zero

Given a supported selector and a valid non-null lookup address whose first byte is zero:

- The first byte is read.
- The receiver’s offset-12 operation is invoked.
- Neither the offset-8 operation nor the additional indirect target is invoked by this path.

Any bytes following the initial zero do not affect this function’s branch.

### Nonempty result and zero receiver result

Given a non-null lookup result beginning with byte 255:

- F-8810f53aaa0b receives that lookup address.
- The receiver’s offset-8 operation receives the dependency result and a zero third argument.
- If the receiver operation returns zero, the function returns without fallback or additional indirect invocation.

This remains true if F-8810f53aaa0b itself returned zero.

### Nonempty result and negative receiver result

Given a valid nonempty lookup result and a receiver value-operation result of −1:

- The result is treated as nonzero.
- The additional indirect target receives −1 and the original lookup address.
- No fallback operation is substituted.

### Receiver mutation during conversion

Given that F-8810f53aaa0b changes the receiver’s dispatch-table address and leaves a valid replacement:

- The subsequent offset-8 dispatch uses the table read after that dependency returns.
- The function does not use a table address cached before the conversion call.

## Uncertainties

The following remain documented_only:

- The selector object’s complete layout and the application meaning of selector values.
- The values and semantic roles of the three lookup bindings.
- The meaning of every dependency result beyond its visible use.
- Whether lookup storage is text, binary data, shared storage, or caller-owned storage.
- Whether dependencies mutate globals, the receiver, or lookup storage.
- Whether any dependency allocates, releases, or transfers ownership.
- The identity and complete control-transfer semantics of the final indirect target.
- Error conventions beyond the explicit null, first-byte-zero, and integer-zero branches.
- Thread safety, synchronization requirements, and concurrent-mutation behavior.
- Target endianness and ABI details beyond the displayed 32-bit accesses and byte offsets.

An inferred intent is that the function selects among three lookup categories and applies available content to a receiver, with a fallback when content is unavailable or begins with zero. This interpretation does not establish domain-specific semantics.

## Evidence

The supplied static representation establishes the selector read, three supported branches, dependency order, lookup-result checks, receiver dispatch offsets and arguments, and zero/nonzero handling of the receiver result.

The representation explicitly leaves uncertainty around indirect control transfers. This contract preserves their visible arguments and branch conditions without assigning an unsupported identity or ownership role.

No runtime tests have been performed. No external dependency implementations, binding contents, dispatch targets, or application-level behavior were available for verification.
