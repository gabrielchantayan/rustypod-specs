<!-- Trusted provenance: F-2f0ebe43dc9e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2f0ebe43dc9e

## Purpose

F-2f0ebe43dc9e conditionally resolves a working object, performs setup and two ordered operations, invokes a helper with a temporary descriptor, and publishes four 32-bit values into a caller-supplied context.

The normal publication path requires a nonzero initial indirect-call result, a nonzero request, and two nonzero resolution results. A zero first resolution returns zero. A zero second resolution invokes a dependency represented as non-returning.

Two details materially constrain the contract:

- The shared global is referenced twice before the first ordered operation: once to save a value for the **second** operation and again to supply the **first** operation’s argument. These values are equal only under an appropriate stability assumption.
- The descriptor’s first two slots are reused after its address has escaped to a helper. Those slots are written before another dependency call and read afterward for publication. Their contents cannot unconditionally be identified with the values written before that call.

Interpretations such as attachment, registration, replacement, or resource release are **inferred intent**, not established semantics.

## Interface

### Inputs and result width

The representation exposes three inputs and a 32-bit integer result.

| Input | Width | Established use |
|---|---:|---|
| Context | 32 bits, address-bearing | Supplies an object pointer at offset `0x34`; is passed to setup; identifies four publication fields beginning at offset `0x88`. |
| Request | 32 bits | Tested for zero and otherwise passed unchanged to F-f7490d12dfa0. |
| Payload | 32 bits | Copied unchanged into the temporary descriptor. |

No signed comparison, request-range check, or payload interpretation is visible. Zero/nonzero tests operate on the represented 32-bit values.

The result is:

- The initial indirect-call result if that result is zero or the request is zero.
- Zero if the first F-682adfd5a561 result is zero.
- On normal completion of the publication path, the contents read from descriptor offset `0x00` at the final return.

That last value was previously assigned the final indirect-method result, but an intervening dependency can potentially change the address-exposed storage. Equating the return value with the method result requires that storage to remain unchanged. Equating the return value with the value stored at context offset `0x88` additionally requires stability between that publication read and the return read.

There is no ordinary return after a zero second resolution if F-a817d8d58c10 satisfies its non-returning annotation.

### Context and object layout

All offsets below are byte offsets; all listed fields occupy four bytes.

| Location | Established role |
|---|---|
| Context `0x34` | Pointer to the object used by the initial indirect call. |
| Context `0x88` | Receives the descriptor offset-`0x00` contents read during publication. |
| Context `0x8c` | Receives the descriptor offset-`0x04` contents read during publication. |
| Context `0x90` | Receives the third F-b3f4a7a52609 result. |
| Context `0x94` | Receives the second F-9dfe5f6048f1 result. |
| Initial object `0x00` | Method-table pointer; entry at table offset `0xe4` supplies the initial call target. |
| Working object `0x00` | Method-table pointer; entry at table offset `0x1a8` supplies the final indirect target. |
| Working object `0x38` | Four-byte input to the second resolution. |

The publication region spans context offsets `0x88` through `0x97`, inclusive.

### Temporary descriptor layout and reuse

The descriptor is four contiguous four-byte slots in temporary call-frame storage.

| Descriptor offset | Value supplied to F-78801ca166f2 | Later explicit reuse |
|---|---|---|
| `0x00` | Address of context offset `0x88` | Overwritten with the final indirect-method result. |
| `0x04` | Working-object pointer | Overwritten with the first F-9dfe5f6048f1 result. |
| `0x08` | Original payload | No subsequent explicit read or write is shown. |
| `0x0c` | Constant `1` | No subsequent explicit read or write is shown. |

The first two slots are therefore not permanently pointer-valued fields. Their later contents are treated as 32-bit values for publication and return.

### Preconditions

The function contains no explicit validation of the context, object pointers, method tables, or indirect targets.

Normal execution requires:

- Valid initial-call structures even when the request is zero.
- A usable working object throughout every reached access and dependency invocation.
- Callable indirect targets with compatible calling conventions.
- Writable publication storage on the publication path.
- Continued validity of any storage accessed through a pointer retained by a dependency.

These are required execution conditions, not evidence that unknown dependencies preserve objects or refrain from retaining pointers. Invalid accesses have no visible recovery path.

## Behavior

### 1. Invoke the initial dependency before testing the request

Read the object pointer at context offset `0x34`, read its method-table pointer, and select the target at table offset `0xe4`.

For this contract, **F-000000000001** denotes that indirect call site, not a resolved function identity. The supplied representation shows no explicit arguments for the invocation. Its full receiver convention and other ABI-level inputs are unknown.

After the call:

- A zero result causes an immediate zero return.
- A nonzero result with a zero request is returned unchanged.
- Otherwise, continue to the first resolution.

A zero request does not suppress this indirect call or any effects it produces.

### 2. Resolve the working object

Call F-682adfd5a561 with the initial result and the 32-bit constant `0x1700`.

If it returns zero, return zero without setup or explicit publication writes. Otherwise, retain its result as the working-object pointer.

No ownership acquisition, reference-count rule, or lifetime guarantee is established by this call.

### 3. Perform setup and initialize the descriptor

Call F-f7490d12dfa0 with the context and request. Pass its result unchanged to F-5b949731133c as the first argument, with the working object as the second argument.

The first setup result is not checked for zero. The second setup result is unused.

After these calls, initialize the descriptor with:

- The address of context offset `0x88`.
- The working-object pointer.
- The original payload.
- The four-byte value `1`.

The descriptor is not explicitly heap-allocated. Its construction does not itself establish ownership transfer or a lifetime beyond this invocation.

### 4. Resolve the second value

After setup, read the working object’s four-byte field at offset `0x38`. Call F-682adfd5a561 with that value and the 32-bit constant `6`.

Because this field is read after setup, setup-induced changes to it can affect the resolution input.

If the resolution returns zero, invoke F-a817d8d58c10. It is represented as non-returning. Before that invocation:

- Setup and descriptor initialization have occurred.
- Neither F-9dfe5f6048f1 call has occurred.
- F-78801ca166f2 has not received the descriptor.
- No explicit final publication store has occurred.

No cleanup or recovery is visible on this branch.

### 5. Perform two ordered operations with distinct shared-global reads

Three four-byte globals are distinguished by role: **shared global**, **first selector**, and **second selector**. These labels do not imply application semantics.

The first operation has this established sequencing:

1. Call F-b3f4a7a52609 with the working object.
2. Read the shared global and retain that value for the second operation.
3. Evaluate the first F-9dfe5f6048f1 call’s arguments, including a read of the first selector and a **separate reference to the shared global**.
4. Invoke F-9dfe5f6048f1 with the second resolution result, first selector, separately read shared-global value, and the first F-b3f4a7a52609 result.
5. Retain this operation’s result.

The saved shared-global read precedes evaluation of the first operation’s arguments. The representation does not establish the relative ordering of the first-selector read and the additional shared-global read within that argument expression.

No dependency call is visibly interposed between the two shared-global reads. Nevertheless, treating them as equal requires the global to remain stable over that interval. They are not both uses of one captured value.

Next:

1. Call F-b3f4a7a52609 again with the working object.
2. Read the second selector for the second operation.
3. Invoke F-9dfe5f6048f1 with the same second resolution result, the second selector, the **earlier saved** shared-global value, and the second F-b3f4a7a52609 result.
4. Retain this second operation’s result.

The second operation does not reload the shared global for its third argument. Mutations after the saved read cannot replace that saved argument, although they can affect the first operation’s separately read argument if they occur before that read.

Neither operation result is checked for zero.

### 6. Expose the descriptor to its helper

Invoke F-78801ca166f2 with the descriptor’s address. Its return value is unused.

At entry, the descriptor has the four initialized fields described above. The helper is not shown to receive immutable storage. It may modify the descriptor, use the publication address, or retain a pointer; its implementation is unavailable.

The working object used afterward is the retained working-object value, not a fresh read of descriptor offset `0x04`.

### 7. Reuse exposed storage, call again, and publish

After F-78801ca166f2 returns, obtain the current method-table pointer from the working object and select the entry at offset `0x1a8`.

For this contract, **F-000000000002** denotes this indirect call site, not a resolved function identity. Invoke it with the working object explicitly supplied.

On its return:

1. Write its 32-bit result into descriptor offset `0x00`.
2. Write the retained first F-9dfe5f6048f1 result into descriptor offset `0x04`.
3. Call F-b3f4a7a52609 a third time with the working object.

The two descriptor writes precede the third call. Because their addresses were exposed through F-78801ca166f2, that call cannot be assumed to preserve them. For example, a retained descriptor pointer could be used by behavior reached during the third call.

After the third call returns, perform the following four explicit context writes in order:

1. Read descriptor offset `0x00` and store that value at context offset `0x88`.
2. Read descriptor offset `0x04` and store that value at context offset `0x8c`.
3. Store the third F-b3f4a7a52609 result at context offset `0x90`.
4. Store the second F-9dfe5f6048f1 result at context offset `0x94`.

Finally, read descriptor offset `0x00` for the return value.

Thus, the first two published values are defined by the **post-call slot contents**, not unconditionally by the values assigned before the third call. The final return is likewise a read of reused storage.

No dependency call appears between these publication stores and the return. This establishes neither atomic publication nor immunity to aliasing, asynchronous modification, or concurrent access.

## Side effects

The function explicitly:

- Invokes the reached dependencies in the described order.
- Initializes temporary descriptor storage.
- Exposes that storage’s address to F-78801ca166f2.
- Overwrites its first two slots after the final indirect method.
- Writes four context fields on the normal publication path.

All dependencies may have additional effects. An early exit excludes this function’s final stores, not all changes to the context or other state. The descriptor helper can receive an address into the publication region and may affect it before publication.

Changes the helper makes to descriptor offsets `0x00` and `0x04` before the later explicit overwrites are superseded by those overwrites, if execution reaches them. Changes during the third F-b3f4a7a52609 call can instead affect the values subsequently read for publication and return.

No explicit allocation, deallocation, reference-count adjustment, rollback, or ownership-transfer declaration is visible. Such dependency behavior is **documented_only**.

The visible address calculations use fixed byte offsets. The working-object field is fourteen four-byte elements from its base: `14 × 4 = 56`, or `0x38` bytes. No variable-size calculation, overflow check, range normalization, or signed arithmetic decision is shown. Valid address calculations are preconditions; wraparound recovery is not established.

## Dependencies

| Opaque ID | Observed boundary contract |
|---|---|
| F-000000000001 | Contract-local initial indirect site, selected through context `0x34` and table `0xe4`; supplies the gate and first resolution input. Full calling convention unknown. |
| F-682adfd5a561 | Two calls: initial result with `0x1700`, then working-object field `0x38` with `6`. Their zero-result branches differ. |
| F-f7490d12dfa0 | Receives context and nonzero request; result forwarded without validation. |
| F-5b949731133c | Receives the setup result and working object; result unused. |
| F-a817d8d58c10 | Invoked on a zero second resolution; annotated non-returning. |
| F-b3f4a7a52609 | Three working-object calls. First two results feed the ordered operations. Third result is published; that call occurs after the exposed descriptor slots are overwritten and before they are read. |
| F-9dfe5f6048f1 | Two four-argument calls. First uses a fresh shared-global read within its argument expression; second uses the earlier saved read. Equality is conditional on stability across the two reads. |
| F-78801ca166f2 | Receives the temporary descriptor address. Mutation, pointer retention, ownership effects, and preservation guarantees are unknown. |
| F-000000000002 | Contract-local final indirect site at working-object table `0x1a8`; receives the working object. Its result is written to exposed temporary storage before another dependency call. |

The contract-local indirect IDs distinguish call sites only; they do not assert distinct runtime targets.

## Acceptance examples

These are conditional static cases, not executed tests.

1. **Initial gate is zero.**  
   With valid initial structures and an initial result of zero, return zero for any request or payload. Do not reach resolution or explicit publication.

2. **Request is zero.**  
   If the initial result is `0x12345678` and the request is zero, return `0x12345678`. The initial call still occurs.

3. **First resolution fails.**  
   With nonzero initial result and request, a zero resolution using `0x1700` returns zero before setup and explicit publication.

4. **Second resolution fails.**  
   A zero resolution using the post-setup field at `0x38` and selector `6` reaches F-a817d8d58c10. Under its non-returning contract, neither ordered operation nor publication follows.

5. **Ordinary publication with stable reused slots.**  
   Suppose the ordered operations return `0x11` and `0x22`, the final indirect method returns `0x33`, and the third F-b3f4a7a52609 call returns `0x44`. If descriptor offsets `0x00` and `0x04` retain their assigned values through publication, publish `0x33`, `0x11`, `0x44`, and `0x22` at context offsets `0x88`, `0x8c`, `0x90`, and `0x94`. If offset `0x00` also remains stable through the return read, return `0x33`.

6. **Shared-global equality requires stability.**  
   If the saved read yields `0x55` and the additional first-call read also yields `0x55`, both operations receive `0x55` as their third argument. Later mutation to `0x66` does not change the second operation’s saved argument. If an external change instead makes the additional read yield `0x66`, the first operation receives `0x66` and the second receives `0x55`.

7. **Zero method result with stable storage.**  
   If the final method returns zero and descriptor offset `0x00` remains zero through publication and return, publish zero at context `0x88`, publish the other fields, and return zero. This is not an early-resolution exit.

8. **Mutation through retained descriptor storage.**  
   Suppose the final method returns `0x33` and the first operation returned `0x11`. If the third F-b3f4a7a52609 call changes the exposed slots to `0xaa` and `0xbb`, those values—not `0x33` and `0x11`—are published at context `0x88` and `0x8c`, provided they remain stable during the corresponding reads. A stable final slot `0x00` produces return value `0xaa`. The same rule applies if the method originally returned zero.

## Uncertainties

The following remain **documented_only**:

- Application meanings of inputs, selectors, globals, fields, constants, and results.
- The initial indirect call’s complete ABI-level arguments.
- Ownership and lifetime rules for resolved objects and dependency results.
- Whether dependencies retain the descriptor pointer or mutate its reused slots.
- Whether dependencies free, invalidate, or replace objects accessed later.
- Behavior if F-a817d8d58c10 returns despite its annotation.
- Exceptions, faults, nonlocal exits, callbacks, reentrancy, synchronization, and concurrency.
- Aliasing restrictions between supplied objects, context storage, and exposed temporary storage.
- Exact machine-level load ordering or load elimination beyond the supplied representation.

Required validity and stability assumptions do not establish dependency preservation guarantees.

## Evidence

This contract derives from static examination of the supplied function representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent evidence.

Established behavior includes the branch conditions, 32-bit values, fixed offsets, ordered calls, separate shared-global references, descriptor-address exposure, later slot reuse, publication sequence, and return read. Dependency implementations and broader behavioral contracts were not supplied.

No runtime tests have been performed.
