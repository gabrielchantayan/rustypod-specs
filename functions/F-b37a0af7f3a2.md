<!-- Trusted provenance: F-b37a0af7f3a2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b37a0af7f3a2

## Purpose

F-b37a0af7f3a2 resolves an object from a context and request, validates access to associated metadata, selects an operation, and dispatches that operation to dependencies.

The established behavior is a dispatcher, not an independently defined lifecycle operation. Its dependencies determine the meanings of the selected operations, whether objects remain valid afterward, and whether memory is retained or released.

Two branches precede normal operation selection:

- A nonzero low byte in a resolver-controlled word selects an alternate handling path.
- Certain context flags, combined with a metadata byte above 127, select a special handler followed by an indirect object call.

Normal operation selection can be overridden by context flags. Selected operations sometimes write one-byte context markers after their dependency calls.

## Interface

### Inputs

The function accepts six values, represented in the supplied evidence as 32-bit arguments:

| Position | Contract name | Established use |
|---|---|---|
| 1 | Context | Address passed to dependencies and used for byte-sized state accesses. |
| 2 | Request | Address whose 32-bit field at offset `0x04` is read; also passed unchanged to dependencies. |
| 3 | Selector | Passed to the resolver. Later, zero versus nonzero controls operation 28. |
| 4 | Original token | Copied into a writable 32-bit temporary and also passed separately, unchanged, to dependencies. |
| 5 | Previous operation | Compared with 8 when a context override is active. No other direct use is established. |
| 6 | Auxiliary selector | Passed to operation selection on the alternate path. For operations 15 and 23, equality to 1 enables additional effects. |

“Context,” “Request,” and “object” describe usage, not established application-level types.

The fourth argument is treated as an opaque 32-bit word. Its low byte is tested only after a dependency has had an opportunity to modify the temporary copy. The original argument remains distinct from that copy.

### Output and termination

No return value is defined by the supplied interface.

The function can:

- Return immediately when resolution produces a null object.
- Return after dispatching one or more dependency calls.
- Enter F-a817d8d58c10 on either of two validation failures. The supplied static evidence marks this dependency as nonreturning.

There is no locally returned error code, diagnostic string, or recovery result.

### Memory layout used directly

| Base | Offset | Width | Access |
|---|---:|---:|---|
| Request | `0x04` | 32 bits | Read for the resolver call. |
| Resolved object | `0x00` | Address-sized, represented as 32 bits | Read as an indirect-call table pointer. |
| Resolved object | `0x34` | 32 bits | Read and passed to metadata lookup. |
| Resolved object | `0x11D` | 8 bits | Read for alternate-path lookup. |
| Metadata result | `0xE8` | 8 bits | Read for conditional gating and one override. |
| Context | `0x9F` | 8 bits | Written with 1 by operation 12. |
| Context | `0xA0` | 8 bits | Written with 1 by operation 14. |
| Context | `0xA3` | 8 bits | Written with 1 by the extra-effects sequence. |
| Context | `0xA4` | 8 bits | First special-path flag. |
| Context | `0xA5` | 8 bits | Highest-priority normal-operation override flag. |
| Context | `0xA6` | 8 bits | Second special-path flag, read conditionally. |
| Context | `0xA7` | 8 bits | Lower-priority normal-operation override flag, read conditionally. |
| Indirect-call table | `0xA0` | Function address | Read for an indirect call with the object and zero. |

The evidence uses a 32-bit address representation. It does not establish portable behavior for a different pointer width or ABI.

## Behavior

### 1. Resolve the object

The function first initializes a writable 32-bit temporary with the Original token.

It then calls F-23c46a4f09c5 with these six arguments, in order:

1. Context.
2. The 32-bit field read from Request at offset `0x04`.
3. The address of the temporary.
4. Original token.
5. Request.
6. Selector.

The dependency may modify the temporary through its address.

If the returned object is null, the function returns. No subsequent metadata lookup or dispatch is performed by this function. This does not imply that resolution itself had no side effects.

For a nonnull object, the function reads its 32-bit field at `0x34` and calls F-682adfd5a561 with that value and `0x1180`.

A zero result invokes F-a817d8d58c10. A nonzero result is subsequently treated as an address providing the metadata byte at offset `0xE8`.

This lookup occurs before checking the resolver-controlled temporary, including on the alternate path.

### 2. Handle a nonzero resolver-controlled low byte

If the low byte of the temporary is nonzero:

1. Read the object byte at `0x11D`.
2. Call F-058144d59a45 with Context and that byte.
3. If the result is exactly `-1`, invoke F-a817d8d58c10.
4. Otherwise call F-3a6685924f3f with Context, Request, Original token, and Auxiliary selector.
5. If the selection result is not 2, call F-befffbb40ab7 with Context and object, then return.
6. If it is 2, call F-81963d81bb92 with Context, object, the result from step 2, and 1, then return.

Only the low eight bits of the temporary control entry into this path. Nonzero upper bits alone do not activate it.

The value from F-058144d59a45 is rejected only when equal to `-1`. The visible caller applies no other range check.

### 3. Apply the metadata-dependent special path

This stage is reached only when the temporary’s low byte is zero.

The function reads Context byte `0xA4`. If that byte is zero, it also reads Context byte `0xA6`. If `0xA4` is nonzero, `0xA6` is not read at this stage.

If either flag is nonzero, the function reads the metadata byte at `0xE8` and compares it as an unsigned byte against 127.

When that byte is greater than 127:

1. Call F-b82f111644ab with Context and object.
2. Perform the indirect object operation described below.
3. Return.

When both context flags are zero, the metadata byte is not read for this gate.

The indirect operation reads the object’s table pointer at offset zero, obtains the callable entry at table offset `0xA0`, and calls it with object and 0. On the special path, these reads occur after F-b82f111644ab returns; the evidence does not establish that the handler preserves the object or table.

### 4. Select and override a normal operation

If neither earlier path returns, call F-3a6685924f3f with Context, Request, and Original token.

This call is present even when a subsequent override replaces its result.

After that call:

- Read Context byte `0xA5`.
- If it is nonzero, select operation 0 when the metadata byte at `0xE8` has raw value `0xC8`; otherwise select operation 28.
- If `0xA5` is zero, read Context byte `0xA7`.
  - If `0xA7` is nonzero, select operation 12 when Previous operation equals 8; otherwise select operation 8.
  - If `0xA7` is zero, retain the dependency’s selection result.

The `0xC8` comparison is expressed in the evidence as signed eight-bit equality to `-56`. It is a byte equality, not a range test.

These reads occur after operation selection returns. The function does not snapshot the flags beforehand or establish that the selector dependency preserves them.

### 5. Dispatch the selected operation

In the following table, dependency argument lists begin with Context and object unless explicitly stated otherwise. Remaining arguments are exact scalar values.

| Operation | Ordered behavior |
|---:|---|
| 0 | F-d4b95dc5d36c with trailing argument `0`. |
| 1 | F-d4b95dc5d36c with trailing argument `1`. |
| 2 | Perform the indirect object operation with object and `0`. |
| 3 | F-1ba4d80b78f0 with trailing argument `1`. |
| 4 | F-1ba4d80b78f0 with trailing argument `0`. |
| 5 | F-40782ddfefb2 with trailing argument `1`. |
| 6 | F-40782ddfefb2 with trailing argument `0`. |
| 7 | F-f6964fedafa2 with trailing arguments `(0, 1, 1, false)`. |
| 8 | F-f6964fedafa2 with trailing arguments `(0, 0, 1, false)`. |
| 9 | F-f6964fedafa2 with trailing arguments `(1, 1, 1, false)`. |
| 10 | F-f6964fedafa2 with trailing arguments `(1, 0, 1, false)`. |
| 11 | F-d4b95dc5d36c with trailing argument `0`. |
| 12 | F-b334ed82a873 with trailing arguments `(2, 0, 1)`; then write byte `1` to Context offset `0x9F`. |
| 13 | F-b334ed82a873 with trailing arguments `(1, 0, 1)`. |
| 14 | F-5e0df028a8e0 with trailing arguments `(4, 1, 1)`; then write byte `1` to Context offset `0xA0`. |
| 15 | Execute the combined operation described below, with its final Boolean argument false. |
| 16 | F-f6964fedafa2 with trailing arguments `(4, 0, 1, false)`. |
| 17 | F-d4b95dc5d36c with trailing argument `0`. |
| 18 | F-b334ed82a873 with trailing arguments `(1, 0, 0)`; then F-f6964fedafa2 with `(1, 1, 0, false)`. |
| 19 | F-b334ed82a873 with trailing arguments `(2, 0, 0)`; then F-f6964fedafa2 with `(2, 0, 0, false)`. |
| 20 | F-f6964fedafa2 with trailing arguments `(0, 1, 1, true)`. |
| 21 | F-f6964fedafa2 with trailing arguments `(1, 0, 1, true)`. |
| 22 | F-f6964fedafa2 with trailing arguments `(4, 0, 1, true)`. |
| 23 | Execute the combined operation described below, with its final Boolean argument true. |
| 24 | F-6868dcb892f9 with trailing arguments `(0, 0, 0)`. |
| 25 | F-6868dcb892f9 with trailing arguments `(0, 0, 14)`. |
| 26 | F-6868dcb892f9 with trailing arguments `(0, 0, 15)`. |
| 27 | F-40782ddfefb2 with trailing argument `1`. |
| 28 | F-b334ed82a873 with trailing arguments `(2, 0, 0)`; then F-f6964fedafa2 with `(2, 0, 0, false)` if Selector is zero, or `(2, 1, 0, false)` otherwise. |
| Any other value | F-d4b95dc5d36c with trailing argument `0`. |

No range error is produced for an unlisted operation.

### Combined operation: 15 and 23

Both operations first call:

1. F-b334ed82a873 with Context, object, and `(1, 0, 0)`.
2. F-f6964fedafa2 with Context, object, and `(1, 1, 0, false)` for operation 15, or `(1, 1, 0, true)` for operation 23.

If Auxiliary selector is not exactly 1, the function then returns.

If Auxiliary selector is 1, it continues in this order:

1. Call F-938c950e5046 with no explicit arguments.
2. Call F-956bc4cb3d20 with no explicit arguments.
3. Pass the second call’s result to F-a925704a7cb6, followed by `(3, 1, 2, 4, 0)`.
4. Write byte `1` to Context offset `0xA3`.
5. Return.

The last write occurs only after all preceding calls return normally.

## Side effects

The direct persistent writes established for this function are the one-byte writes at Context offsets `0x9F`, `0xA0`, and `0xA3`. None is cleared here.

Other effects are delegated. Dependency calls may change context state, objects, global state, external resources, or ownership; their bodies are not supplied. No preservation guarantee follows from the absence of a direct write in this function.

The function also exposes the address of its temporary word to F-23c46a4f09c5. That establishes writable access during the call, not permission to retain the address afterward.

No direct allocation, deallocation, reference-count adjustment, or ownership transfer is established. In particular, F-befffbb40ab7 and the indirect object operation must not be described as destructors without additional evidence.

Required readable and writable regions depend on the selected path. There are no visible null checks for Context or Request before Request is read, nor independent bounds or validity checks for metadata and indirect-call addresses. Validity of those accesses is a precondition for defined execution.

## Dependencies

The opaque dependency contracts visible at this boundary are:

- F-23c46a4f09c5: six-argument resolver; returns an object or null and receives a writable temporary-word address.
- F-682adfd5a561: lookup using an object field and `0x1180`; zero is fatal, nonzero is used as metadata.
- F-a817d8d58c10: nonreturning failure dependency.
- F-058144d59a45: alternate-path lookup; exactly `-1` is fatal.
- F-3a6685924f3f: operation selection. The alternate path supplies four explicit arguments; the normal path supplies three.
- F-befffbb40ab7 and F-81963d81bb92: alternate-path handlers.
- F-b82f111644ab: handler preceding the special-path indirect call.
- F-d4b95dc5d36c, F-1ba4d80b78f0, F-40782ddfefb2, F-f6964fedafa2, F-b334ed82a873, F-5e0df028a8e0, and F-6868dcb892f9: operation handlers with argument combinations specified above.
- F-938c950e5046, F-956bc4cb3d20, and F-a925704a7cb6: ordered extra-effects sequence for operations 15 and 23.
- An unidentified indirect callable at object-table offset `0xA0`: receives object and zero.

The inconsistent explicit argument count for F-3a6685924f3f is preserved as evidence. A default fourth argument on the normal path is not established.

## Acceptance examples

These are conditional contract examples, not executed tests. Unless stated otherwise, dependencies return normally and all accessed memory is valid.

1. **Resolution failure:** If F-23c46a4f09c5 returns null, no metadata lookup or later dispatch occurs.

2. **Metadata failure precedes alternate handling:** If resolution succeeds but F-682adfd5a561 returns zero, F-a817d8d58c10 is invoked even when the temporary’s low byte is nonzero.

3. **Low-byte boundary:** A temporary value of `0x00000100` takes the zero-low-byte path; `0x00000101` takes the alternate path.

4. **Alternate dispatch:** With a nonzero temporary low byte and a lookup result other than `-1`, selection result 2 calls F-81963d81bb92. Any other selection result calls F-befffbb40ab7.

5. **Unsigned metadata boundary:** With Context `0xA4` nonzero, metadata byte `0x80` triggers the special handler and indirect call; `0x7F` does not.

6. **Override precedence:** If Context `0xA5` and `0xA7` are both nonzero after normal selection, `0xA5` controls the result. Metadata `0xC8` selects operation 0; another byte selects operation 28.

7. **Previous-operation override:** With `0xA5` zero and `0xA7` nonzero, Previous operation 8 selects operation 12 and its post-call `0x9F` write. Any other Previous operation selects operation 8.

8. **Operation 28 distinction:** Selector zero yields the F-f6964fedafa2 trailing tuple `(2, 0, 0, false)`; any nonzero Selector yields `(2, 1, 0, false)`. Both follow F-b334ed82a873.

9. **Extra-effects gate:** Operation 23 with Auxiliary selector 1 performs the three additional calls and writes `0xA3`. Auxiliary selector 2 omits that sequence. The Boolean passed to the earlier handler remains true in both cases.

10. **Unknown selection:** With no override and selection result `-1` or 29, the dispatcher calls F-d4b95dc5d36c with trailing zero rather than reporting an invalid operation.

## Uncertainties

The following are documented_only:

- Application meanings of the flags, operation numbers, metadata bytes, and handler arguments.
- Exact dependency prototypes, calling convention, and the reason for the differing operation-selector argument counts.
- Whether any dependency mutates globals, modifies later-read fields, retains pointers, transfers ownership, releases memory, or terminates execution.
- Required object sizes beyond the accessed offsets, alignment rules, and behavior for invalid addresses.
- The identity and full contract of the indirect callable.
- Whether the nonreturning failure mechanism exits, aborts, throws, or performs another transfer of control.
- Reentrancy, thread safety, synchronization, and behavior under concurrent mutation.

The dispatcher interpretation is inferred intent. The call ordering, byte tests, argument constants, direct writes, and visible branch outcomes are established static evidence.

## Evidence

This contract is derived solely from the supplied function body and opaque dependency mapping. No dependency implementations or runtime observations were supplied.

The evidence establishes fixed-offset accesses and conditional control flow, not application semantics or safety of the referenced memory. Arithmetic is limited here to fixed address offsets and table indexing; no length calculation, payload-size multiplication, or numerical overflow policy is exposed.

No runtime tests have been performed. Acceptance examples are deductions from the supplied static evidence, not claims of observed execution.
