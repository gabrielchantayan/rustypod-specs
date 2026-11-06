<!-- Trusted provenance: F-4dfc5ec582b7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4dfc5ec582b7

## Purpose

F-4dfc5ec582b7 conditionally records a 32-bit value in an object or performs a longer sequence involving object lookup, object update, temporary-state preparation, and an optional dependent operation.

The first decision uses a one-byte field in the supplied object:

- If that byte is nonzero, the function writes the supplied 32-bit value to the object and returns immediately.
- If that byte is zero, the function stores a separate supplied byte, resolves an associated value through a dependency, updates the object through another dependency, prepares a 92-byte temporary region, and conditionally invokes an additional operation. On the normal completion path, it then writes the supplied 32-bit value and invokes cleanup for the temporary region.

This ordering is established by the supplied static source. The application meanings of the object, supplied values, temporary region, and dependent operations are not established. Descriptions such as “lookup,” “update,” and “cleanup” identify structural roles, not proven resource-management semantics.

## Interface

The function has five inputs and no explicit return value.

| Input | Represented width | Contract role |
|---|---:|---|
| Object address | 32 bits in the supplied representation | Base address for direct field accesses and an embedded-region address passed to a dependency. |
| Recorded value | 32 bits | Passed to the lookup dependency and written to the object’s field at offset `0xE4`. Its interpretation as an identifier, pointer, or scalar is unknown. |
| Operation argument | 32 bits | Tested for zero and passed unchanged to the optional operation. |
| Suppression selector | 32 bits | Tested for zero to determine whether an object halfword may be forwarded. It does not itself suppress the optional operation. |
| Stored byte | 8 bits | Written at object offset `0xE8` only on the longer path. |

The object has the following directly relevant layout:

| Offset from object base | Width or role | Direct use |
|---|---:|---|
| `0xB4` | Address of an embedded region; extent unknown | Passed to F-1f46730b1ced. |
| `0xE4` | 4 bytes | Receives the recorded value. |
| `0xE8` | 1 byte | Receives the stored byte on the longer path. |
| `0xEA` | 2 bytes | Conditionally read and forwarded to the optional operation. |
| `0xEC` | 1 byte | Read once at entry to select the immediate-return or longer path. |

The function also creates an addressable temporary region of 92 bytes. It passes the region’s address to dependencies but does not directly initialize, inspect, or modify its contents.

Two dependency-returned addresses are dereferenced during temporary preparation:

- One supplies a one-byte value at offset `0x2C`.
- A separately obtained address supplies a four-byte value at offset `0x28`.

These addresses are obtained by separate calls. They must not be assumed to identify the same object.

### Preconditions

For defined normal execution, each direct memory access must target appropriately accessible storage:

- The object’s selection byte must be readable at entry.
- The recorded-value field must be writable on either normal path.
- On the longer path, the stored-byte field must be writable.
- The conditional halfword field must be readable if its selection conditions are met.
- The two dependency-returned addresses used for field reads must support their respective accesses.

Alignment, byte order, address-space rules, and the accessible extent required by dependencies are not established. The supplied representation uses 32-bit integer carriers for addresses, but that alone does not establish a portable calling convention.

There are no explicit null checks, bounds checks, or access-validity checks for these addresses.

## Behavior

### 1. Entry selection and immediate return

The function first reads one byte at object offset `0xEC`.

If that byte is nonzero:

1. It writes the recorded value as four bytes at object offset `0xE4`.
2. It returns.

On this path, the function does not directly write the stored-byte field, read the conditional halfword, or call any dependency. The precise nonzero byte value does not affect this decision.

If the entry byte is zero, execution takes the longer path. This selection byte is not directly read again. Changes to it during subsequent calls do not cause this function to revisit the initial decision.

### 2. Early byte store and associated-value lookup

On the longer path, the function first writes the stored byte at object offset `0xE8`.

It then calls F-1f46730b1ced with:

1. The address at object base plus `0xB4`.
2. The recorded value.

The returned 32-bit value is tested for zero.

If it is zero, the function invokes F-a817d8d58c10. The supplied source marks this dependency as non-returning. Under that annotation, no subsequent normal-path operations occur: the later recorded-value write and temporary cleanup are not reached.

The implementation of this failure dependency is unavailable. Its diagnostic behavior, termination mechanism, and process-wide effects are unknown. If it returns contrary to the annotation, the supplied control flow has no explicit return or alternative recovery branch before the subsequent sequence.

### 3. Associated-value processing

For the ordinary nonzero lookup result, the function performs three calls in order:

1. F-d2f99870cc43 receives the lookup result and returns a 32-bit value.
2. F-bf557f07179f receives the original object address and that returned value.
3. F-fa8898fa75ae receives the original lookup result.

There is no explicit status check between these calls.

The lookup result remains retained by the caller. If the optional operation is later selected, that original result is passed to it even though F-fa8898fa75ae has already been called with it.

This ordering does not establish whether F-fa8898fa75ae releases, finalizes, validates, or merely inspects anything. In particular, the function cannot be specified as safely retaining ownership or safely releasing ownership without the missing dependency contracts.

### 4. Temporary preparation from separate observations

The function next obtains two fields through separate calls to F-fdd8589eca2d:

1. It calls F-fdd8589eca2d and reads one byte at offset `0x2C` from the returned address.
2. It calls F-fdd8589eca2d again and reads four bytes at offset `0x28` from this second returned address.
3. It calls F-2c5a9f9c1eb3 with the temporary region’s address, the four-byte value from the second observation, and the byte from the first observation, in that argument order.

The byte from the first observation is captured before the second call. If the second call changes the first object or returns a different address, the previously captured byte is still the value supplied to F-2c5a9f9c1eb3.

The caller supplies no explicit initialization of the 92-byte region before this call. Whether the dependency initializes all of it, constructs a resource in it, or uses only part of it is unknown.

### 5. Optional-operation selection

After temporary preparation, the function calls F-5c5958cfd5fe and compares its result with the integer value 8.

If the result equals 8:

- F-818c773e8c8a is not called.
- No third call to F-fdd8589eca2d occurs.
- The object halfword at offset `0xEA` is not directly read.
- Execution proceeds to the recorded-value write and temporary cleanup.

Every result other than 8 selects the optional operation.

For that operation, the function calls F-fdd8589eca2d once more. Its result becomes the operation’s first argument.

The operation’s fifth argument is a two-byte quantity selected as follows:

- If the suppression selector is nonzero, the quantity is zero.
- If the suppression selector is zero but the operation argument is zero, the quantity is zero.
- Only if the suppression selector is zero and the operation argument is nonzero does the function read the two bytes at object offset `0xEA`.

The third call to F-fdd8589eca2d occurs before any selected halfword read. Consequently, effects of that call may affect the halfword value subsequently observed.

F-818c773e8c8a then receives, in order:

1. The result of the third F-fdd8589eca2d call.
2. The temporary region’s address.
3. The original lookup result.
4. The original operation argument.
5. The selected two-byte quantity.

A nonzero suppression selector forces only the fifth argument to zero; it does not prevent this call.

### 6. Normal completion

After the optional operation returns, or immediately after the mode-equals-8 decision skips it, the function:

1. Writes the recorded value as four bytes at object offset `0xE4`.
2. Calls F-920630e53283 with the temporary region’s address.
3. Returns without an explicit result.

The recorded-value write precedes cleanup. Therefore, the write is established, but its persistence after cleanup is not: the cleanup dependency’s effects are unknown.

No exception handler or alternate unwinding cleanup is visible. If an earlier dependency fails to return normally, the later store and cleanup are not established.

## Side effects

The function’s direct object effects are limited to:

- Reading the selection byte at offset `0xEC`.
- Writing the recorded value at offset `0xE4` on the immediate-return path or near the end of normal longer-path execution.
- Writing the stored byte at offset `0xE8` near the beginning of the longer path.
- Reading the halfword at offset `0xEA` only under the specified conditions.

Dependencies may produce additional effects. Their global writes, object mutations, allocation, deallocation, I/O, and failure behavior are unknown.

In particular:

- The early stored-byte write occurs before lookup failure can be detected.
- On the longer path, the recorded-value write is delayed until after all selected processing except cleanup.
- No direct rollback of the early stored-byte write is present.
- The temporary region is passed to preparation, the optional operation when selected, and cleanup.
- The caller performs no explicit heap allocation or deallocation.
- No ownership transfer is proven for the object, lookup result, dependency-returned addresses, or temporary region.
- No synchronization or atomic-access guarantee is established.

The visible address arithmetic uses fixed offsets. There is no visible length calculation, multiplication, or narrowing conversion among the explicit input values. The selected optional argument is two bytes; whether its receiving ABI treats those bits as signed or unsigned is not established.

## Dependencies

| Opaque ID | Established role in this function |
|---|---|
| F-1f46730b1ced | Receives the embedded-region address and recorded value; returns the value tested for zero and retained for later calls. |
| F-a817d8d58c10 | Invoked on a zero lookup result; annotated as non-returning. |
| F-d2f99870cc43 | Receives the lookup result and supplies a 32-bit value to the next dependency. |
| F-bf557f07179f | Receives the object address and the preceding dependency result. |
| F-fa8898fa75ae | Receives the lookup result before temporary preparation and before any optional operation. |
| F-fdd8589eca2d | Called twice for separate field observations, and a third time only when the optional operation is selected. |
| F-2c5a9f9c1eb3 | Receives the temporary region and the captured four-byte and one-byte values. |
| F-5c5958cfd5fe | Returns the value whose equality to 8 skips the optional operation. |
| F-818c773e8c8a | Performs the selected five-argument operation. |
| F-920630e53283 | Receives the temporary region after the recorded-value write on normal longer-path completion. |

None of these role descriptions establishes that a dependency preserves globals or other reachable memory.

## Acceptance examples

These examples describe conditional acceptance criteria derived from static evidence. They are not runtime observations. Unless stated otherwise, dependencies return normally and supplied addresses are valid.

### Nonzero entry selector

Given a nonzero byte at object offset `0xEC`:

- The recorded value is written at offset `0xE4`.
- No dependency is invoked.
- No direct write is made at offset `0xE8`.
- No direct read is made at offset `0xEA`.

The operation argument and suppression selector do not affect this path.

### Zero lookup result

Given a zero entry selector and a zero result from F-1f46730b1ced:

- The stored byte is written before lookup.
- F-a817d8d58c10 is invoked.
- If its non-returning annotation holds, the normal completion store and temporary cleanup are not reached.

No claim is made about other effects of the failure dependency.

### Mode value 8

Given a zero entry selector, nonzero lookup result, and a result of 8 from F-5c5958cfd5fe:

- Associated-value processing and temporary preparation still occur.
- Exactly two calls to F-fdd8589eca2d occur in this function.
- The optional operation is skipped.
- The recorded-value write occurs before cleanup.

### Mode other than 8 with halfword forwarding

Given a mode result other than 8, a zero suppression selector, and a nonzero operation argument:

- A third F-fdd8589eca2d call occurs.
- The halfword is read after that call.
- F-818c773e8c8a receives that observed halfword as its fifth argument.

If the third call changes the halfword, the changed value is the one directly read.

### Forced-zero optional argument

Given a mode result other than 8:

- A nonzero suppression selector forces the fifth argument to zero regardless of the operation argument.
- A zero operation argument also forces the fifth argument to zero.
- Neither condition prevents the optional operation.
- The halfword is not directly read in either case.

### Different observation objects

If the first two F-fdd8589eca2d calls return different valid addresses, temporary preparation receives the byte from the first address and the four-byte value from the second. Combining the values into a single-object snapshot would not satisfy the established behavior.

## Uncertainties

The following are **documented_only**:

- Application-level meanings of all inputs and fields.
- Full object layout, alignment requirements, byte order, and external ABI.
- Full dependency contracts, including mutation, ownership, lifetime, and failure behavior.
- Whether the dependency-returned addresses are stable or shared.
- Whether F-fa8898fa75ae affects the later usability of the retained lookup result.
- The temporary region’s internal layout and initialized extent.
- Whether cleanup can change the object’s recorded-value field.
- Behavior after invalid addresses, failed memory accesses, or abnormal dependency exits.
- Whether the non-returning annotation accurately describes every execution of F-a817d8d58c10.

An inferred intent is that the entry selector enables a record-only path and that the longer path prepares and disposes of temporary state around an optional operation. This intent is not required to explain the established reads, writes, and call order.

## Evidence

The evidence is the supplied static function body and the supplied opaque dependency mapping.

It establishes the explicit field widths, offsets, branch conditions, argument forwarding, temporary-region size, and sequential call order described above. The non-returning behavior is supported by an annotation, not by an available dependency implementation.

No runtime tests have been performed. No dependency bodies, external declarations, or independent layout definitions were supplied. Acceptance examples are therefore static contract cases, not execution results.
