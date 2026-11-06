<!-- Trusted provenance: F-9f1fb2e46c9a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9f1fb2e46c9a

## Purpose

F-9f1fb2e46c9a coordinates an update to a caller-supplied object using a 32-bit value, two 32-bit control arguments, and an 8-bit value.

The established behavior has two paths:

- When the object’s byte at offset `0xE8` is nonzero, write the supplied 32-bit value at offset `0xE0` and return without invoking dependencies.
- Otherwise, record the supplied byte, resolve a handle, perform several dependency-mediated operations, construct a temporary object, conditionally invoke an operation using that temporary, write the supplied 32-bit value at offset `0xE0`, and destroy the temporary.

The roles of these fields and dependencies are not established. Interpreting the operation as selection, activation, notification, or deferred processing would be inferred intent, not part of this contract.

## Interface

The function has five inputs and no explicit return value.

| Input | Width and interpretation | Established use |
|---|---|---|
| Object address | 32-bit address-bearing input in the supplied representation | Base for fixed-offset field accesses and an embedded-region address; also passed to a dependency |
| Update value | 32 bits, interpretation otherwise unspecified | Passed to handle resolution and written to the object |
| Primary control | 32 bits | Tested for zero and passed unchanged to the conditional operation |
| Secondary control | 32 bits | Tested for zero; not directly passed onward |
| Byte value | 8 bits | Written to the object on the full-processing path only |

No Boolean normalization is visible for either control input. Zero and nonzero determine the relevant branches; a nonzero primary control is still forwarded as its original 32-bit value.

The relevant object-relative locations are:

| Offset | Access width | Established role |
|---|---:|---|
| `0xB0` | Address only at this level | Address supplied as the first argument to F-1f46730b1ced |
| `0xE0` | 32 bits | Destination for the update value |
| `0xE4` | 8 bits | Destination for the byte value on the full-processing path |
| `0xE6` | 16 bits | Conditional source for an argument to F-818c773e8c8a |
| `0xE8` | 8 bits | Initial branch selector |

The function also provides a temporary region of 92 bytes to dependencies. Its internal layout, alignment requirements, and ownership rules are not established by the visible body.

**Preconditions.** The initial byte read and each subsequently reached memory access must be valid. On the short path, the visible requirements are a readable byte at `0xE8` and a writable 32-bit location at `0xE0`. On the full path, additional object locations, the embedded region, dependency return values, and dependency-specific arguments must satisfy the requirements of the reached operations.

No visible validation checks object bounds, alignment, address validity, or the availability of storage returned by accessors. Byte order and unaligned-access behavior are not established.

## Behavior

### Initial branch and short path

The first visible operation reads one byte from object offset `0xE8`.

If that byte is nonzero:

1. The function writes the 32-bit update value to offset `0xE0`.
2. The function returns.

This path does not directly write offset `0xE4`, read offset `0xE6`, initialize the temporary, or invoke any listed dependency. The specific nonzero byte value does not otherwise affect the visible behavior.

The branch is selected from this single initial read. There is no visible later recheck of offset `0xE8`.

### Byte recording and handle resolution

If the initial byte is zero, the function first writes the supplied 8-bit byte value at offset `0xE4`.

It then invokes F-1f46730b1ced with:

- The address of the region beginning at object offset `0xB0`.
- The supplied 32-bit update value.

The returned 32-bit result is used as a handle-like value. Its concrete representation is unknown.

If the result is zero, the function invokes F-a817d8d58c10. The supplied static representation marks that dependency as non-returning. Under that annotation, no later normal-path operation is reached: there is no explicit final write to `0xE0`, no temporary construction, and no temporary destruction.

The write to `0xE4` precedes this failure boundary and is not explicitly rolled back. Whether F-a817d8d58c10 terminates execution, transfers control nonlocally, or uses another non-returning mechanism is unknown.

### Handle-mediated operations

For a nonzero resolved handle, the visible call order is:

1. Invoke F-d2f99870cc43 with the handle, obtaining a 32-bit value.
2. Invoke F-bf557f07179f with the object address and that returned value.
3. Invoke F-fa8898fa75ae with the original handle.

The original handle is retained and may later be supplied to F-818c773e8c8a. Consequently, this contract does not describe F-fa8898fa75ae as destroying, releasing, or invalidating it. Those interpretations require dependency evidence not supplied here.

No status check is visible between these operations. No explicit recovery or rollback surrounds them.

### Temporary construction inputs

The function next obtains the arguments for F-2c5a9f9c1eb3 using two separate calls to F-fdd8589eca2d:

1. Call F-fdd8589eca2d.
2. Read one byte at offset `0x2C` from that call’s returned address, retaining the byte.
3. Call F-fdd8589eca2d again.
4. Read a 32-bit value at offset `0x28` from the second call’s returned address.
5. Invoke F-2c5a9f9c1eb3 with the 92-byte temporary region, the second accessor’s 32-bit value, and the first accessor’s retained byte.

The two accessor calls are observably distinct. Their return addresses need not be equal. Their backing state need not be stable between calls. The construction inputs therefore are not guaranteed to represent a single coherent snapshot.

Neither accessor result is visibly checked before its associated read.

F-2c5a9f9c1eb3 is used in a constructor-like position, but its initialization, allocation, and failure semantics remain unknown. The visible caller performs no result check.

### Mode selection and conditional operation

After the construction call returns, the function invokes F-5c5958cfd5fe and compares its 32-bit result with exactly `8`.

If the result is `8`, the function skips F-818c773e8c8a and proceeds to the final update and cleanup. This skip occurs after temporary construction, not before it.

For every result other than `8`, the function:

1. Calls F-fdd8589eca2d once more and retains its returned value as the first argument for F-818c773e8c8a.
2. Selects a 16-bit argument.
3. Invokes F-818c773e8c8a.

The 16-bit argument is selected as follows:

| Secondary control | Primary control | Selected 16-bit value |
|---|---|---|
| Nonzero | Any value | Zero |
| Zero | Zero | Zero |
| Zero | Nonzero | A 16-bit read from object offset `0xE6` |

The additional F-fdd8589eca2d call occurs before the conditional read from `0xE6`. If that dependency changes the object, the read observes the field after that call, subject to the execution environment’s memory semantics. The function does not capture `0xE6` earlier.

The arguments to F-818c773e8c8a, in order, are:

1. The result of the additional F-fdd8589eca2d call.
2. The temporary region.
3. The original resolved handle.
4. The unchanged 32-bit primary control.
5. The selected 16-bit value.

A zero primary control does not suppress this call. Only a mode result equal to `8` visibly suppresses it.

No status result from F-818c773e8c8a is inspected by this function.

### Final update and cleanup

After either skipping or completing the conditional operation, the function writes the supplied 32-bit update value at object offset `0xE0`.

It then invokes F-920630e53283 with the temporary region and returns if that dependency returns.

This ordering is significant:

- The conditional operation, when reached, precedes the explicit write to `0xE0`.
- The explicit write to `0xE0` precedes the cleanup call.
- There is no second explicit write to `0xE0` after cleanup.

The value at `0xE0` on return cannot be guaranteed solely from the explicit assignment: dependency behavior is unknown, and cleanup executes afterward.

## Side effects

The function’s explicit object writes are limited to:

- One 32-bit write at `0xE0` on the short path.
- One 8-bit write at `0xE4` near the start of the full path.
- One 32-bit write at `0xE0` near the end of a normally progressing full path.

It performs no explicit write to `0xE6` or `0xE8`.

Dependencies may introduce additional writes, global-state changes, allocation, deallocation, callbacks, or nonlocal control transfer. None of those effects can be excluded from the supplied evidence.

There is no explicit transaction or rollback. If a dependency fails or does not return, earlier writes and dependency effects may already have occurred. In particular:

- The byte write precedes handle-resolution failure handling.
- The final update write occurs only after the earlier full-path calls return.
- Cleanup follows the final update write.

The temporary’s address is passed to construction, the optional operation, and cleanup. The caller-visible body contains no explicit heap allocation or deallocation. It does not establish whether dependencies allocate internal resources, retain addresses, or transfer ownership.

No locking, atomic access protocol, or reentrancy protection is visible. Thread-safety and callback-related guarantees are documented_only.

## Dependencies

Only opaque identifiers are used below. Descriptions identify visible call roles, not independently established dependency semantics.

| Dependency | Visible use |
|---|---|
| F-1f46730b1ced | Receives the embedded-region address and update value; returns the handle tested against zero |
| F-a817d8d58c10 | Invoked when handle resolution returns zero; annotated as non-returning |
| F-d2f99870cc43 | Receives the resolved handle and returns a 32-bit value |
| F-bf557f07179f | Receives the object address and that 32-bit value |
| F-fa8898fa75ae | Receives the original handle before temporary construction |
| F-fdd8589eca2d | Called twice to source construction inputs, and once more when the mode is not `8` |
| F-2c5a9f9c1eb3 | Receives the temporary region, a 32-bit value, and an 8-bit value |
| F-5c5958cfd5fe | Supplies the result compared with exactly `8` |
| F-818c773e8c8a | Optional five-argument operation involving the temporary and original handle |
| F-920630e53283 | Receives the temporary after the final update write |

Calls are ordered as described above. No dependency is assumed to preserve global state, object fields, accessor results, or memory ownership.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Where final field values are stated, assume dependencies return normally and do not overwrite those fields.

1. **Short path.** Given a nonzero byte at `0xE8`, the function writes the supplied update value to `0xE0` and returns without dependency calls. An existing byte at `0xE4` remains unchanged by the function’s explicit operations.

2. **Zero handle.** Given a zero byte at `0xE8` and a zero result from F-1f46730b1ced, the supplied byte is written to `0xE4` before F-a817d8d58c10 is invoked. Under the non-returning annotation, temporary construction and the explicit write to `0xE0` are not reached.

3. **Mode equals eight.** Given successful progress to a mode result of `8`, temporary construction still occurs. F-818c773e8c8a and the third F-fdd8589eca2d call do not occur. The write to `0xE0` precedes cleanup.

4. **Stored 16-bit argument.** Given a mode result other than `8`, zero secondary control, nonzero primary control, and a field value of `0xFFFF` at `0xE6` when read, F-818c773e8c8a receives the 16-bit pattern `0xFFFF` as its final argument. Whether the callee interprets that pattern as signed or unsigned is unknown.

5. **Forced-zero argument.** Given a mode result other than `8` and nonzero secondary control, the final argument is zero regardless of the primary control or `0xE6`. No explicit read of `0xE6` is performed on this branch.

6. **Zero primary control.** Given a mode result other than `8` and zero primary control, F-818c773e8c8a is still invoked. It receives the unchanged zero primary control and a zero final argument.

7. **Distinct accessor results.** If the first two accessor calls return different readable objects, construction receives the byte from the first object’s `0x2C` location and the 32-bit value from the second object’s `0x28` location.

8. **Cleanup modifies the destination.** If cleanup changes `0xE0`, acceptance requires the explicit update write to have occurred before that change; it does not require the supplied update value to remain at function return.

## Uncertainties

The following are documented_only:

- Semantic meanings of the object fields, controls, handle, mode value, and temporary.
- Dependency contracts, including ownership, retention, global mutation, exceptions, callbacks, and failure behavior.
- Valid object extent, alignment requirements, and platform byte order.
- Whether the non-returning annotation accurately describes every execution of F-a817d8d58c10.
- Whether the handle remains valid after F-fa8898fa75ae; the visible function nevertheless reuses its saved value on the conditional-operation path.
- Temporary layout and whether all 92 bytes are initialized or consumed.
- Concurrency, atomicity, memory ordering, and reentrancy guarantees.
- ABI extension rules for the 8-bit and 16-bit arguments.

The visible address calculations use fixed offsets; there are no loops, size multiplications, or control-argument arithmetic. Valid address computation remains a precondition. Signedness of the control inputs does not affect their zero/nonzero tests, and comparison with the representable value `8` does not distinguish signed from unsigned interpretation.

## Evidence

**Established static evidence:** the supplied body directly supports the field-access widths, branch conditions, call arguments, call order, two separate construction-input accessor calls, mode-dependent third accessor call, 92-byte temporary region, and update-before-cleanup ordering described here.

**Inferred intent:** terms such as “handle,” “construction,” and “cleanup” describe apparent roles only. They do not establish resource ownership or dependency behavior.

**Unknown:** dependency internals and runtime outcomes are not available. The non-returning characterization is supplied annotation rather than independently observed behavior.

No runtime tests have been performed.
