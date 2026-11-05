<!-- Trusted provenance: F-6b8cfbc6a126; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6b8cfbc6a126

## Purpose

F-6b8cfbc6a126 coordinates an operation on an opaque 32-bit input. It obtains an initial context value, initializes two temporary objects, and asks an object associated with the input to populate one temporary. A subsequent zero/nonzero classification selects one of two execution paths.

The zero-classification path invokes an operation using a previously captured global value, a fixed all-ones argument, a field from the associated object, and the second input. It then invokes an input-related operation unconditionally, conditionally processes the initial context value, and conditionally invokes another input-related operation.

The nonzero-classification path obtains another context value, passes that value and the input to a dependency, and then processes the initial context value.

Both paths converge on two ordered temporary-object cleanup calls when control returns normally.

These statements describe visible call sequencing and data flow. Business meanings such as removal, activation, notification, resource release, or successful completion are not established.

## Interface

**Inputs**

| Input | Established representation | Observable use |
|---|---|---|
| Primary input | Opaque 32-bit value | Passed to F-f1a0113c72f4 and, depending on the branch, to F-7b5e4937c518, F-8d2a99b5c349, or F-8f9bd59e3ec5. |
| Secondary input | Signed integer in the supplied representation, analyzed here as 32-bit | Passed unchanged to F-a73566bce62e on the zero-classification path. Separately tested for inequality to zero to control F-8d2a99b5c349. |

The primary input is not directly dereferenced by F-6b8cfbc6a126. Its validity requirements are imposed by the dependencies that receive it.

The secondary input is not normalized to a Boolean before being passed to F-a73566bce62e. Its complete integer value matters at that interface. For the later conditional call, any nonzero value satisfies the condition, including a negative value.

**Output**

There is no explicit return value. Results are observable through dependency calls, their side effects, and any changes they make to external state or objects accessible through the supplied values.

**Local storage and accessed layouts**

- One temporary has four bytes of visible local storage. Its address is supplied first to F-f1a0113c72f4, subsequently to F-e38e6836b640, and finally to F-561858688f94.
- A second temporary has twenty bytes of visible local storage. Its address is supplied first to F-1c06d0a5910b, then to an indirect object operation, then to F-7d1048432426, and finally to F-c1466ea05103.
- Objects used for indirect calls have a dispatch-table pointer at their beginning in the supplied representation.
- Dispatch entries are read at byte offsets 0x168, 0x1F4, and 0xE4 in their respective tables.
- On the zero-classification path, a four-byte field is read at byte offset 0x40 from an object returned by F-e38e6836b640.

The temporary sizes identify visible storage extents, not complete type definitions. Field meanings, alignment requirements, and object ownership are undocumented.

**Preconditions for ordinary execution**

The dependencies and indirect calls must return normally for the described sequence to complete. Returned objects, dispatch tables, selected dispatch entries, the global source location, and the field at offset 0x40 must be readable and valid whenever accessed. No local null checks or validity checks protect those accesses.

## Behavior

### Initial context acquisition

The function first calls F-938c950e5046.

From the returned object, it reads the dispatch-table pointer and selects the entry at byte offset 0x168. It invokes that entry and retains a 32-bit result as the **initial context value**.

The supplied representation shows no explicit arguments at this indirect call site. Its full calling convention, including any implicit receiver convention, is undocumented.

The initial context value is captured once. Later calls may change global or object state, but F-6b8cfbc6a126 does not reacquire this value before passing it to F-50afc298d6d3.

### Temporary initialization and global capture

The function next calls F-1c06d0a5910b with the address of the twenty-byte temporary and two zero arguments.

After that call returns, it reads a 32-bit value from a designated global location and retains the result as the **captured global value**.

This placement is significant:

- The capture occurs after F-1c06d0a5910b, so changes that dependency makes to the global location can affect the captured value.
- The capture occurs before F-f1a0113c72f4 and the subsequent object operations.
- Later changes to the global location do not cause this function to reload the value for F-a73566bce62e.

The function then calls F-f1a0113c72f4 with the address of the four-byte temporary and the primary input.

These calls establish the visible initialization sequence. They do not establish whether either temporary owns a resource.

### Population and classification

The function passes the address of the four-byte temporary to F-e38e6836b640 and treats the returned value as an object pointer.

It reads that object's dispatch-table pointer and invokes the entry at byte offset 0x1F4. This call has two explicit arguments: the returned object pointer and the address of the twenty-byte temporary.

After that call returns, F-7d1048432426 is called with the address of the twenty-byte temporary. Its integer result selects the path:

- Exactly zero selects the zero-classification path.
- Any nonzero value selects the nonzero-classification path.

The classification result is not otherwise used. A negative nonzero result is not handled as a distinct error case by this function.

### Zero-classification path

First, F-e38e6836b640 is called again with the address of the four-byte temporary. This is a fresh call; the function does not reuse the object pointer from the earlier population step.

From this newly returned object, it reads the four-byte field at byte offset 0x40.

It then calls F-a73566bce62e with these four arguments, in order:

1. The captured global value.
2. The 32-bit all-ones value, 0xFFFFFFFF.
3. The four-byte field just read from offset 0x40.
4. The unchanged secondary input.

The all-ones value has the signed interpretation −1 when treated as a 32-bit signed integer. The supplied evidence does not establish whether F-a73566bce62e interprets it as a signed value, an unsigned value, or a sentinel.

The integer result of F-a73566bce62e is retained. The next call is always F-7b5e4937c518 with the primary input, provided F-a73566bce62e returns normally. This call occurs before either subsequent conditional operation.

If the retained result from F-a73566bce62e is nonzero, the function calls F-50afc298d6d3 with the initial context value. If that result is zero, this call is omitted.

The function then independently checks the secondary input. If it is nonzero, F-8d2a99b5c349 is called with the primary input. If it is zero, that call is omitted.

The two conditions are independent. A zero result from F-a73566bce62e does not suppress F-8d2a99b5c349 when the secondary input is nonzero. Conversely, a zero secondary input does not suppress F-50afc298d6d3 when the retained result is nonzero.

### Nonzero-classification path

The function calls F-938c950e5046 again. It does not reuse the object returned during initial context acquisition.

From this newly returned object, it reads the dispatch-table pointer and invokes the entry at byte offset 0xE4. The returned 32-bit value is the **alternate context value**. As with the initial context-producing indirect call, the supplied representation shows no explicit arguments.

The function then calls F-8f9bd59e3ec5 with the alternate context value and the primary input.

After that call returns, F-50afc298d6d3 is called unconditionally with the initial context value—not the alternate context value.

This path does not call F-a73566bce62e, F-7b5e4937c518, or F-8d2a99b5c349. It does not consult the secondary input locally. The earlier global capture still occurs, although its captured value is not passed to any dependency on this path.

### Common completion

After either path completes normally:

1. F-561858688f94 is called with the address of the four-byte temporary.
2. F-c1466ea05103 is called with the address of the twenty-byte temporary.
3. The function returns without an explicit result.

This cleanup order is fixed. There is no visible early return between classification and common completion.

## Side effects

**Directly established accesses**

The function explicitly reads:

- Dispatch-table pointers and the selected indirect-call entries.
- A 32-bit value from a global location.
- A four-byte object field at offset 0x40, only on the zero-classification path.

It supplies writable local-storage addresses to dependencies. The visible body contains no explicit direct write to the captured global location or to the accessed object's field.

**Transitive effects**

All direct and indirect dependencies may have side effects not described by this evidence. In particular:

- The initial and later calls to F-938c950e5046 need not return the same object.
- The two calls to F-e38e6836b640 need not return the same object.
- A copied context or global value preserves its captured bits, not necessarily the lifetime or validity of anything those bits designate.
- No callee is known to preserve global state.
- No callee is known to acquire, transfer, retain, or release ownership merely because of its position in the sequence.

The last two calls have the structural role of cleanup operations, but resource destruction or deallocation is inferred intent, not established semantics.

**Errors and abnormal completion**

No explicit error value is returned. No local recovery, retry, rollback, or failure-reporting branch is visible.

The results of F-7d1048432426 and F-a73566bce62e are tested only for zero versus nonzero; neither test proves a success/failure convention.

The supplied evidence establishes cleanup only after normally returning calls. It does not establish cleanup during exceptions, faults, termination, or other nonlocal exits.

## Dependencies

| Dependency | Contract visible at this call site |
|---|---|
| F-938c950e5046 | Returns an object used for indirect dispatch. Called initially and again only on the nonzero-classification path. |
| F-1c06d0a5910b | Receives the twenty-byte temporary address and two zero arguments before global capture. |
| F-f1a0113c72f4 | Receives the four-byte temporary address and primary input. |
| F-e38e6836b640 | Receives the four-byte temporary address and returns an object. Called once before classification and again only on the zero-classification path. |
| F-7d1048432426 | Receives the twenty-byte temporary address; its zero/nonzero result selects the path. |
| F-a73566bce62e | Receives the captured global value, 0xFFFFFFFF, an object field, and the secondary input. Its zero/nonzero result controls one later call. |
| F-7b5e4937c518 | Receives the primary input after F-a73566bce62e, regardless of that call's result. |
| F-50afc298d6d3 | Receives the initial context value conditionally on the zero-classification path and unconditionally on the nonzero-classification path. |
| F-8d2a99b5c349 | Receives the primary input only when classification is zero and the secondary input is nonzero. |
| F-8f9bd59e3ec5 | Receives the alternate context value and primary input only on the nonzero-classification path. |
| F-561858688f94 | Receives the four-byte temporary address first during common completion. |
| F-c1466ea05103 | Receives the twenty-byte temporary address second during common completion. |

The three indirect-call targets have no supplied opaque function IDs. Their contracts remain identified by dispatch-table offset and role rather than by invented identities.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each assumes valid accesses and normal return from every invoked dependency.

**Zero classification, zero operation result, zero secondary input**

After F-a73566bce62e returns zero, F-7b5e4937c518 must still receive the primary input. Neither F-50afc298d6d3 nor F-8d2a99b5c349 is called. Common cleanup follows in the specified order.

**Zero classification, nonzero operation result, zero secondary input**

F-7b5e4937c518 runs first. F-50afc298d6d3 then receives the initial context value. F-8d2a99b5c349 is omitted.

**Zero classification, zero operation result, nonzero secondary input**

F-7b5e4937c518 runs first. F-50afc298d6d3 is omitted. F-8d2a99b5c349 then receives the primary input. This also applies to a secondary input of −1 under the 32-bit signed interpretation; F-a73566bce62e must have received that unchanged value.

**Zero classification, both conditions nonzero**

The branch-specific post-operation calls occur in this order: F-7b5e4937c518, F-50afc298d6d3, F-8d2a99b5c349. Cleanup follows.

**Nonzero classification**

For either a positive or negative nonzero classifier result, the alternate context-producing dispatch occurs. F-8f9bd59e3ec5 receives its result, followed by F-50afc298d6d3 receiving the initial context value. The secondary input does not alter this local path.

**Global value changes after capture**

If a later dependency changes the global source location, F-a73566bce62e still receives the value captured earlier. This criterion concerns copied bits, not continued validity of a referenced resource.

**Different objects from repeated lookup**

If the second F-e38e6836b640 call returns a different valid object, the field passed to F-a73566bce62e comes from that second object. No identity-equality requirement is visible.

## Uncertainties

The following remain **documented_only**:

- Exact source-level types, pointer encoding, signedness of opaque four-byte values, and complete calling conventions.
- Full layout and alignment of either temporary or any returned object.
- Meaning of the global value, object field, context values, and all-ones argument.
- Whether classification represents emptiness, validity, availability, or another property.
- Whether the result of F-a73566bce62e denotes success, a state transition, or something else.
- Ownership and lifetime rules, including whether any dependency releases or invalidates resources.
- Reentrancy, concurrency, synchronization, exception behavior, and fault handling.
- Any behavior omitted from the supplied representation.

The visible operations require no integer addition, multiplication, truncation, or counter arithmetic on the inputs. Relevant numerical behavior is limited to fixed offsets, four-byte accesses, the 32-bit all-ones argument, and zero/nonzero tests.

## Evidence

This contract is based solely on static inspection of the supplied representation. Embedded comments and labels are not treated as authoritative behavioral specifications.

Established evidence consists of call order, explicit arguments, result-dependent branches, visible storage extents, fixed access offsets, and normal-return cleanup order. Initialization, population, context acquisition, and cleanup are role descriptions inferred from those relationships; they do not establish hidden callee semantics.

No implementation access, runtime execution, or tests were used. The acceptance examples describe behavior to verify conditionally, not observed runtime results.
