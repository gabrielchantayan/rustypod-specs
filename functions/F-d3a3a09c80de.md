<!-- Trusted provenance: F-d3a3a09c80de; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d3a3a09c80de

## Purpose

F-d3a3a09c80de performs a gate-controlled operation selected by a one-byte state field in a caller-supplied record.

The supplied static representation establishes three principal paths:

- A zero result from an initial gate causes an immediate return without a local access to the record.
- A nonzero gate result and a state other than 3 select a sequence of dependency calls followed by a one-byte store of 3.
- A nonzero gate result and state 3 select an indirect query. A nonzero query result selects another indirect invocation; a zero result selects allocation, context processing, and a conditional final callback.

The meanings of state 3, the operation value, and the associated objects are not established. Terms such as initialization, completion, or notification would describe inferred intent rather than demonstrated behavior.

This contract distinguishes explicit local operations from dependency behavior. It does not assume that dependencies preserve globals, keep objects alive, release memory, or return normally.

## Interface

### Inputs

**Record reference.** A pointer with the following path-dependent layout:

| Byte offset | Access width | Established use |
|---|---:|---|
| +0x00 | 1 byte | State discriminator; compared with 3 and, on one path, overwritten with 3 |
| +0x1C | 4 bytes | Opaque value supplied to dependencies |
| +0x20 | 4 bytes | Pointer-bearing field; its referent supplies a 4-byte value |

No local access to the intervening bytes is shown. No broader record layout is established.

**Operation value.** An opaque 4-byte value forwarded to dependencies or written into allocated storage. Its interpretation as a number, pointer, handle, or packed value is unknown.

### Output and representation

The function has no explicit result value. Every ordinary exit is a return without a result.

The supplied representation uses 4-byte words, pointer-bearing values, and integer results, supporting a 32-bit address model for the described layout. The state is exactly one byte; its signedness does not affect equality with 3.

The gate and query results are tested for zero versus nonzero. A later integer result is compared with signed 32-bit −1, whose bit pattern is 0xFFFFFFFF. No other result value selects the final callback.

Offsets are byte offsets. Dispatch-table offsets +0x1A0 and +0x84 are not entry indices. The allocation request is 12 bytes, or 0x0C.

Address calculations use fixed offsets. No bounds, address-overflow, or wraparound checks are shown. Ordinary execution requires the resulting addresses to designate valid storage; behavior at invalid addresses or arithmetic limits is not established as a supported case.

### Path-dependent preconditions

The gate-zero path requires no local access to the record. This does not establish that the gate itself has no relevant side effects.

After a nonzero gate result, the record’s state byte must be readable.

For a state other than 3:

- The +0x1C and +0x20 fields must be readable.
- The pointer read from +0x20 must permit a 4-byte read.
- The record must remain suitable for the later +0x1C read and final state-byte write.

For state 3, local execution does not read those two associated record fields. Instead, selected dependency-returned objects, global pointer locations, dispatch tables, and any allocation must support their respective accesses.

There are no visible null checks, allocation-failure checks, alignment checks, lifetime checks, or reported validation errors.

## Behavior

### Initial gate

The first operation is a call to F-0a3684290038 with no explicit arguments.

If its result is zero, the function returns immediately. No local record read, record write, allocation, or subsequent dependency invocation occurs.

Otherwise, the function reads the record’s state byte. Selection therefore uses the state present after the gate returns, not a value captured before the gate.

### State other than 3

F-2c3c85059f34 is invoked with three explicit arguments:

1. The 4-byte value obtained by reading the record’s +0x20 pointer field and then reading the first 4-byte word through that pointer.
2. The 4-byte value read from record offset +0x1C.
3. The operation value.

The pointer-field read necessarily precedes the read through that pointer. The relative evaluation order of this argument expression and the separate +0x1C read is not established.

After F-2c3c85059f34 returns, F-76b7ec0cf6ba is called with no explicit arguments. F-90bdc9fd06fd is then invoked with:

1. The 4-byte result of F-76b7ec0cf6ba.
2. A fresh 4-byte read from record offset +0x1C.

The later read is not a reuse of the value supplied to F-2c3c85059f34. Changes made by either preceding dependency can affect the value passed to F-90bdc9fd06fd.

After F-90bdc9fd06fd returns normally, the function writes the one-byte value 3 at record offset +0x00 and returns.

There is no state recheck before this store. A dependency’s intervening state change can therefore be overwritten. If an earlier dependency does not return normally, the final store is not reached.

This invocation does not proceed into the state-3 query after writing 3.

### State 3: indirect query

F-87be4aa684d1 is called with no explicit arguments. Its result is treated as an object pointer. The object’s first 4-byte word supplies a dispatch-table address; the callable entry at table byte offset +0x1A0 is selected and invoked.

No explicit arguments are shown for this query. An implicit receiver or other calling-convention detail is not established.

The query result selects the nonzero or zero path below.

### Nonzero query result: independently evaluated dispatch lookup and argument

This path uses a global anchor to obtain an object pointer from anchor offset +4. The supplied expression contains **two separate evaluations** of that lookup:

- **Callable-selection evaluation:** Read the global anchor, read the object pointer at anchor +4, read that object’s first 4-byte word as a dispatch-table address, and obtain the callable entry at table byte offset +0x84.
- **First-argument evaluation:** Independently read the global anchor and the object pointer at anchor +4. Supply this pointer as the invocation’s first explicit argument.

The other explicit arguments are the operation value and zero, in that argument order.

These are not established as one lookup whose result is reused. The relative evaluation order of callable selection and the first-argument expression is unspecified by the supplied representation. Within each pointer chain, an address must be obtained before it can be dereferenced, but there is no established total order between the two chains.

Consequently, there is no guaranteed shared snapshot. If the global anchor or its +4 field changes between relevant reads, the object used to resolve the callable can differ from the pointer passed as the first argument. Neither expression is shown as an atomic snapshot of the entire chain.

After the indirect invocation, the representation shows a return. It also flags this transfer as an indirect jump represented as a call. The selected target and explicit arguments are established; exact linkage, stack behavior, and ordinary return mechanics remain unresolved.

No direct record write occurs on this path.

### Zero query result: allocation and context processing

The following operations occur in order, provided each preceding operation completes normally:

1. Call F-76b7ec0cf6ba and retain its 4-byte result as a context value.
2. Call F-8157a180d901 with the size argument 12.
3. Write the 4-byte value 5 at offset +0 of the returned storage.
4. Write the 4-byte operation value at offset +8 of that storage.
5. Call F-bf0b7433249b with the retained context.
6. Call F-583c269a6d82 with the same retained context.
7. Read a 4-byte pointer-bearing value from context offset +0x20.
8. Call F-9813035c7ce7 with explicit arguments 1 and that pointer-bearing value.
9. Compare its integer result with −1.

The allocation’s locally established layout is:

| Byte offset | Width | Local action |
|---|---:|---|
| +0 | 4 bytes | Store 5 |
| +4 | 4 bytes | No visible local access |
| +8 | 4 bytes | Store the operation value |

The middle word is not initialized by this function. Its contents depend on the allocation dependency and prior storage state, neither of which is supplied.

The allocated pointer is not visibly passed as an explicit argument to the subsequent context operations, returned, or released. A registration mechanism, implicit communication channel, ownership transfer, or relationship to the context cannot be inferred from those facts.

The retained context is not refreshed by another F-76b7ec0cf6ba call. Its +0x20 field is read only after both context-processing dependencies return. Either dependency may affect the value obtained by that read.

### Final result and callback selection

If F-9813035c7ce7 returns any 32-bit value other than −1, the function returns without the final callback.

If its result is −1, the function performs both of the following reads:

- A read of the global function-pointer location that selects the final callback target.
- A 4-byte read at offset +4 from the associated pointer value retained before the F-9813035c7ce7 call. This supplies the callback’s sole explicit argument.

Both reads occur after F-9813035c7ce7 returns and after the −1 branch is selected. Their relative evaluation order is **not established**. The contract does not require the argument read to precede the global target read, or the reverse.

The selected callable is invoked with the argument obtained by the object-field read. A change to the callback global can affect target selection according to when its read occurs; a change to the associated object can affect the argument according to when its field is read. No atomic pairing of target and argument is established.

The associated pointer itself is retained across F-9813035c7ce7. The context’s +0x20 field is not reread. If that dependency changes the retained object’s +4 field, the later argument read can observe the change. If it invalidates the object, validity of the later read is not established.

This final indirect transfer is also flagged as an indirect jump represented as a call. Exact transfer and return mechanics remain unresolved.

## Side effects

### Direct writes

The only explicit local writes are:

- One byte containing 3 at record offset +0x00 on the state-not-3 path, after its dependency sequence returns.
- Four bytes containing 5 at allocation offset +0 on the zero-query path.
- Four bytes containing the operation value at allocation offset +8 on that same path.

No direct record write is shown on either state-3 subpath.

Global reads include the independently evaluated anchor lookups on the nonzero-query path and the final callback function-pointer read on the selected −1 path. Dispatch selection also reads the relevant objects and tables.

Dependencies and indirect targets can have additional effects not established here.

### Ownership and lifetime

The function neither explicitly allocates nor explicitly releases the input record. No local ownership transfer of that record is shown.

The 12-byte allocation is not explicitly released or returned. This is evidence of no local release, not proof of a leak. External ownership or allocator-side registration is unknown.

No ownership, lifetime extension, preservation, or release guarantee is established for dependency-returned contexts, query objects, dispatch objects, associated objects, or values stored in them. Required later accesses remain conditional on those objects still being valid.

### Errors and synchronization

There is no explicit error result, diagnostic, rollback, or cleanup branch. The −1 result selects a callback rather than becoming a returned error code.

No local synchronization or atomic snapshot operation is shown. Reentrant or concurrent mutation can affect later reads, subject to whatever execution and memory rules apply externally. The static representation does not establish safe concurrent access or freedom from races.

## Dependencies

| Opaque dependency ID | Explicit use |
|---|---|
| F-0a3684290038 | Initial zero/nonzero gate |
| F-2c3c85059f34 | Receives the dereferenced record-associated word, record +0x1C value, and operation value |
| F-76b7ec0cf6ba | Supplies a 4-byte value used by the selected path |
| F-90bdc9fd06fd | Receives that result and a fresh record +0x1C read |
| F-87be4aa684d1 | Supplies the object used for query dispatch |
| F-8157a180d901 | Receives size 12 and supplies storage for two writes |
| F-bf0b7433249b | Receives the retained context after allocation and writes |
| F-583c269a6d82 | Receives the same context afterward |
| F-9813035c7ce7 | Receives 1 and the context-associated pointer; −1 selects the final callback |

Opaque function IDs are unavailable for three indirect boundaries:

- Query callable selected at table offset +0x1A0.
- Callable selected at table offset +0x84 using one anchor lookup, with its first argument obtained by a separate lookup.
- Final callback selected by a global function-pointer read.

Their identities and full contracts remain **documented_only**. No dependency is assumed to preserve globals or own or release memory.

## Acceptance examples

These are conditional expectations from static evidence, not executed tests.

1. **Gate returns zero.**  
   The function returns without a local record dereference. An invalid record pointer does not cause a local record access on this path. Effects inside the gate remain unrestricted by this contract.

2. **Non-3 state completes normally.**  
   With a nonzero gate, state 2, valid required storage, and normally returning dependencies, the calls are F-2c3c85059f34, F-76b7ec0cf6ba, and F-90bdc9fd06fd, followed by the state-byte store of 3. No query is performed.

3. **Associated field changes between calls.**  
   If the first +0x1C read obtains A, F-2c3c85059f34 changes the field to B, and nothing subsequently changes it, F-90bdc9fd06fd receives B rather than A.

4. **Dependency changes the state.**  
   If a dependency on the non-3 path writes state 7 and all calls return normally with valid storage, the final local store replaces 7 with 3.

5. **Nonzero query with stable dispatch storage.**  
   If the global anchor and its +4 object pointer remain unchanged throughout both lookup evaluations, callable selection and the first argument use the same object pointer. The other arguments are the operation value and zero. No allocation occurs.

6. **Nonzero query with changing dispatch storage.**  
   If the two evaluations obtain different valid objects, the first argument need not equal the object used for callable lookup. The contract does not mandate which evaluation occurs first or require a coherent shared snapshot.

7. **Zero query and writable allocation.**  
   The allocation request is 12 bytes. The local stores write 5 at +0 and the operation value at +8. No local store prescribes the contents at +4.

8. **Context-associated pointer changes.**  
   If either context-processing dependency changes context +0x20, F-9813035c7ce7 receives the value read after both calls return.

9. **Final result boundary and retained pointer.**  
   Results 0, 1, and −2 do not select the callback. Result −1 does. Changing context +0x20 during F-9813035c7ce7 does not redirect the later argument read; it remains based on the retained pointer.

10. **Final target and argument reads.**  
    On the −1 path, both the callback-global read and retained-object +4 read follow F-9813035c7ce7. Either read may precede the other. With stable valid storage, the invocation uses the stored callback pointer and the object’s +4 value; with mutation, no paired snapshot is promised.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the state, constants, operation value, and associated fields.
- Dependency behavior, including global mutation, exceptions, nonlocal exits, ownership, and release.
- Query calling convention and any implicit receiver.
- Exact mechanics of the two warned indirect transfers.
- Allocation failure behavior, initialization policy, and registration effects.
- How the allocated storage becomes observable elsewhere.
- Relative evaluation order of independent argument and callable expressions.
- External validity, alignment, synchronization, and arithmetic-limit requirements.
- Stable identities of indirect targets.

It is reasonable to infer that state 3 selects a distinct processing mode and that the allocation holds operation-related data. Neither inference supplies a missing ownership, dispatch, or synchronization guarantee.

## Evidence

The supplied static representation and opaque dependency mapping establish the branch conditions, field widths and offsets, explicit arguments, local stores, and ordering of separate calls and subsequent operations.

The nonzero-query expression contains independent anchor-plus-4 evaluations for callable lookup and the first argument. It does not establish a shared object snapshot or their relative evaluation order.

The final callback expression reads both a global function pointer and the retained object’s +4 argument field. Both follow the −1 result, but their relative evaluation order is not established.

The representation flags uncertainty in two indirect control transfers. No dependency implementations, allocation contracts, ABI documentation, or runtime traces are supplied.

No runtime tests have been performed. All acceptance examples are conditional static expectations.
