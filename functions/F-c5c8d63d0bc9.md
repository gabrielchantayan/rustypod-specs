<!-- Trusted provenance: F-c5c8d63d0bc9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c5c8d63d0bc9

## Purpose

F-c5c8d63d0bc9 conditionally processes a 32-bit request value against an object and records that value in the object.

A nonzero byte in the object selects a short path: record the request and return without invoking dependencies. Otherwise, the function stores an input byte, obtains a nonzero handle through an embedded object region, performs several dependent operations, constructs temporary state, and conditionally invokes a dispatch dependency. It then records the request and passes the temporary state to a final dependency.

This describes the established call and memory-access behavior. Interpreting the byte as a “busy” flag, the recorded request as a “pending request,” or the dispatch as a particular application operation is inferred intent, not established behavior.

## Interface

The function has five inputs and no explicit return value.

| Input | Representation established by the supplied source | Role |
|---|---|---|
| Object reference | Address represented by a 32-bit integer in the supplied source | Base address for object-relative accesses and an embedded-region address |
| Request value | Opaque 32-bit value | Passed to lookup and ultimately stored in the object |
| Dispatch argument | 32-bit integer | Tested for zero and forwarded unchanged to conditional dispatch |
| Override argument | 32-bit integer | Tested for zero to control selection of a 16-bit dispatch argument |
| Stored byte | Opaque 8-bit value | Written to the object on the processing path |

The object reference is used as an address. Its integer representation does not establish that arbitrary integer inputs are valid addresses.

### Referenced object layout

Offsets are byte offsets from the object base.

| Offset | Width | Direct use |
|---|---:|---|
| +0xB0 | No direct access width established | Address passed to F-1f46730b1ced |
| +0xE0 | 32 bits | Destination for the request value |
| +0xE4 | 8 bits | Destination for the stored-byte input |
| +0xE6 | 16 bits | Conditionally read as a dispatch argument |
| +0xE8 | 8 bits | Initially read to choose the short path or processing path |

The embedded region beginning at +0xB0 has no established size or internal layout. The function does not directly access the byte at +0xE5.

A separate context reference obtained from F-fdd8589eca2d is used with these offsets:

| Offset | Width | Direct use |
|---|---:|---|
| +0x28 | 32 bits | Read from the second context result during temporary-state setup |
| +0x2C | 8 bits | Read from the first context result during temporary-state setup |

The function also provides the address of a 92-byte local temporary region to dependencies. Its internal layout, initialization requirements, and alignment are not established beyond the supplied local declaration and its use as an address.

### Preconditions

For a normal execution:

- The object reference must support the accesses selected by the executed path.
- The initial byte at +0xE8 must be readable.
- The 32-bit destination at +0xE0 must be writable whenever execution reaches either request-recording operation.
- On the processing path, +0xE4 must be writable, and the embedded-region address at +0xB0 must satisfy F-1f46730b1ced’s requirements.
- Context results must support the field reads made through them.
- If the conditional 16-bit read is selected, +0xE6 must be readable.
- All called dependencies must receive arguments satisfying their own contracts.

The function performs no explicit null, bounds, alignment, or object-lifetime validation. Target byte order and alignment rules are not established here.

## Behavior

### Initial path selection

The first direct object access is a one-byte read at +0xE8.

If that byte is nonzero:

1. The function writes the 32-bit request value at +0xE0.
2. It returns.

No dependency is called on this path. The function does not directly write +0xE4 or read +0xE6.

The predicate is strictly zero versus nonzero. Every nonzero byte value selects this path, irrespective of its possible signed-character interpretation.

### Processing-path setup

If the initial byte is zero, the function first writes the stored-byte input at +0xE4.

It then calls F-1f46730b1ced with:

- The address of the embedded region at object offset +0xB0.
- The 32-bit request value.

The dependency’s result is represented as a 32-bit integer and used as a handle.

If the handle is zero, the function calls F-a817d8d58c10. The supplied static evidence identifies that dependency as nonreturning. Consequently, this path has no established normal return, no later direct request write at +0xE0, and no temporary-state setup or finalization by this function.

The +0xE4 write precedes the zero-handle check. There is no direct rollback of that write.

Every nonzero handle passes this check, including values that would be negative under signed 32-bit interpretation.

### Handle-dependent operations

For a nonzero handle, the function performs these calls in order:

1. F-d2f99870cc43 receives the handle and produces an opaque 32-bit value.
2. F-bf557f07179f receives the object reference and that produced value.
3. F-fa8898fa75ae receives the handle.

No ownership transfer, reference-count change, release, or invalidation semantics are established for these calls. In particular, the later conditional dispatch receives the same saved handle after F-fa8898fa75ae has been called. A contract for that dependency must account for this sequence; this function alone does not establish that the handle has been released or remains valid.

### Temporary-state setup

The function obtains context information through two distinct calls to F-fdd8589eca2d:

1. It calls the dependency and reads one byte at +0x2C from the returned reference.
2. It calls the dependency again and reads a 32-bit value at +0x28 from this second returned reference.

It then calls F-2c5a9f9c1eb3 with:

- The address of the 92-byte local temporary region.
- The 32-bit value read through the second context result.
- The byte read through the first context result.

The order matters. The context results need not be equal, and the fields need not describe one consistent snapshot. The first byte is captured before the second context call occurs.

The source does not directly initialize the temporary region before passing it to F-2c5a9f9c1eb3. Whether that dependency initializes all bytes, initializes a smaller structured object, or has other requirements is documented_only.

### Conditional dispatch

After temporary-state setup, the function calls F-5c5958cfd5fe and compares its 32-bit integer result with exactly 8.

If the result equals 8, the function skips dispatch. This does not bypass the preceding handle operations or temporary-state setup.

For every other result, the function calls F-fdd8589eca2d once more to obtain the first argument for F-818c773e8c8a.

The 16-bit argument supplied to dispatch is selected as follows:

| Override argument | Dispatch argument | Selected 16-bit value |
|---|---|---|
| Nonzero | Any value | Zero |
| Zero | Zero | Zero |
| Zero | Nonzero | Value read from object offset +0xE6 |

The third context call occurs before any selected read at +0xE6. Thus, if that context call changes the object, the conditional read observes the object after the call, not a value captured earlier.

Dispatch is a call to F-818c773e8c8a with these ordered arguments:

1. The third context result.
2. The address of the local temporary region.
3. The saved nonzero handle.
4. The original 32-bit dispatch argument.
5. The selected 16-bit value.

The override argument is not forwarded. The dispatch argument is forwarded unchanged, not converted to a Boolean.

The selected argument contains either zero or the 16-bit field’s bit pattern. The supplied evidence does not fully specify the dependency’s ABI-level extension or interpretation of that argument beyond this 16-bit selection.

### Completion

After dispatch returns, or immediately after the dispatch-skipping decision when the selector result is 8, the function:

1. Writes the original 32-bit request value at object offset +0xE0.
2. Calls F-920630e53283 with the address of the local temporary region.
3. Returns.

The write occurs before the final dependency call. It is therefore not necessarily the value present in the object at function return: that dependency may have side effects, including effects through aliases or global state.

No dispatch or finalization result is inspected. The function has no explicit success or failure result.

## Side effects

### Direct memory effects

On the short path, the only direct write is the 32-bit request write at +0xE0.

On the processing path:

- The 8-bit write at +0xE4 occurs before lookup.
- The 16-bit field at +0xE6 is read only for a non-8 selector result, zero override argument, and nonzero dispatch argument.
- The request write at +0xE0 occurs only after all earlier processing-path calls have returned.
- The request write precedes F-920630e53283.

The function does not directly write the branch-control byte at +0xE8.

No direct allocation or explicit deallocation is shown. This does not exclude allocation or deallocation inside dependencies.

### Ownership and lifetime

The local temporary region exists for the duration of this invocation. Its address is supplied to setup, optional dispatch, and finalization dependencies. There is no established permission for a dependency to retain that address beyond the invocation.

The final dependency’s position suggests temporary-state finalization, but resource-release semantics are not established. Likewise, the handle-processing dependencies must not be assigned acquisition or release semantics solely from their ordering.

Nothing establishes that dependencies preserve the object, globals, context identity, or other reachable state. The sequence of calls and direct accesses is therefore part of the contract; replacing repeated context calls with one cached call would require additional evidence.

### Error and interruption behavior

The only explicit failure branch is the zero-handle call to the dependency marked nonreturning.

There is no visible recovery, rollback, exception handling, retry, or unconditional finalization mechanism. If another dependency does not return normally, later writes and calls are not established to occur. In particular:

- Failure before completion can leave the stored byte changed without the final direct request write.
- Interruption after temporary-state setup does not visibly guarantee a call to F-920630e53283.
- Interruption during finalization occurs after the direct request write.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-1f46730b1ced | Receives the embedded-region address and request; returns the handle checked against zero |
| F-a817d8d58c10 | Called for a zero handle; marked nonreturning in the supplied evidence |
| F-d2f99870cc43 | Receives the nonzero handle; produces a 32-bit value |
| F-bf557f07179f | Receives the object reference and the produced 32-bit value |
| F-fa8898fa75ae | Receives the handle before context reads and possible later dispatch |
| F-fdd8589eca2d | Supplies references for two field reads and, conditionally, a third dispatch context |
| F-2c5a9f9c1eb3 | Receives temporary storage, the second context’s 32-bit field, and the first context’s byte |
| F-5c5958cfd5fe | Supplies the selector compared with 8 |
| F-818c773e8c8a | Performs conditional dispatch with context, temporary storage, handle, dispatch argument, and selected 16-bit value |
| F-920630e53283 | Receives temporary storage after the direct request write |

These descriptions specify interactions, not the dependencies’ complete behavior.

## Acceptance examples

These are conditional static acceptance scenarios, not executed tests. Unless a scenario states otherwise, assume valid memory and normally returning dependencies.

1. **Nonzero initial branch byte.**  
   Given any nonzero byte at +0xE8, the function must write the request at +0xE0 and return without dependency calls. The stored-byte input must not be directly written at +0xE4.

2. **Zero handle.**  
   Given zero at +0xE8 and a zero lookup result, the +0xE4 write must precede F-a817d8d58c10. Under its documented nonreturning behavior, no subsequent setup, dispatch, finalization, or direct request write occurs.

3. **Selector equals 8.**  
   Given a nonzero handle and selector result 8, temporary-state setup must still occur. Dispatch and the third context call must not occur. The request write must precede F-920630e53283.

4. **Object-supplied 16-bit argument.**  
   Given selector result other than 8, zero override, and nonzero dispatch argument, dispatch must receive the field read at +0xE6 after the third context call.

5. **Forced zero argument.**  
   Given selector result other than 8 and either nonzero override or zero dispatch argument, dispatch must receive a zero 16-bit selection. The function must not directly read +0xE6 on this path.

6. **Distinct context results.**  
   If the first two context calls return different valid references, setup must use the byte from the first reference and the 32-bit field from the second. Neither reference substitutes for the other.

7. **Late object mutation.**  
   If the third context call changes +0xE6 before returning, a selected field read must observe that later value, subject to the execution environment’s memory rules.

8. **Finalizer changes the request field.**  
   If F-920630e53283 changes +0xE0, that does not violate the local write-order contract. The established requirement is the request write before that call, not an unconditional final-state guarantee.

## Uncertainties

The following remain documented_only:

- Application meaning of the object fields, request, handle, selector value 8, and dispatch arguments.
- Complete dependency contracts, including side effects, ownership, memory retention, synchronization, and failure behavior.
- Whether repeated context calls normally return the same reference.
- Internal layout and required initialization of the 92-byte temporary region.
- Exact ABI argument-extension rules for the selected 16-bit value.
- Target endianness and alignment requirements.
- Behavior if the dependency marked nonreturning unexpectedly returns.
- Reentrancy, concurrent mutation, and whether any dependency can invoke this function recursively.

No runtime tests have been performed.

## Evidence

The contract is based solely on static inspection of the supplied function body.

Established evidence includes the explicit access widths and offsets, zero/nonzero predicates, comparison with 8, dependency argument order, repeated context calls, 92-byte local region, and ordering of direct writes relative to calls.

The nonreturning failure behavior relies on the supplied nonreturn annotation. Semantic descriptions such as lookup, dispatch, setup, and finalization are interaction labels; they do not establish hidden resource or application semantics.

No dependency implementation, runtime trace, external type definition, or independent ABI documentation was supplied.
