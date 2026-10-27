<!-- Trusted provenance: F-0e4d2a00feeb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0e4d2a00feeb

## Purpose

F-0e4d2a00feeb performs a lookup using a 32-bit value read from a caller-supplied location and a separate 32-bit argument. If both a destination argument and the lookup result are nonzero, it invokes a second dependency. A nonzero result from that dependency causes two ordered updates to the destination: a 32-bit value store followed by setting bit 8 of a 16-bit field.

After that conditional processing, the function passes the original lookup result to a final dependency and returns the destination argument unchanged.

This description establishes control flow, argument forwarding, and direct memory effects. It does not establish the semantic meaning of the lookup result, the destination record, or the final dependency. In particular, the final call is not sufficient evidence of destruction, reference release, or ownership transfer.

## Interface

### Inputs

The function has three inputs:

| Input | Established use |
|---|---|
| Source-word address | An address from which the function reads one 32-bit value before invoking the lookup dependency. The address itself is also passed to the conditional dependency when that call occurs. |
| Destination | An integer-valued argument tested against zero, used as a base address for conditional writes, and returned unchanged. The supplied representation uses a 32-bit integer type. |
| Lookup argument | A 32-bit value forwarded unchanged to the lookup dependency. No interpretation or validation is shown. |

“Source-word address,” “destination,” and “lookup argument” are descriptive interface labels, not recovered identifiers.

The destination is address-like in its use, but the supplied representation declares it as an integer rather than a pointer. A portable pointer ABI, pointer provenance rules, and support for addresses wider than 32 bits are not established.

### Return value

On normal completion, the return value is the original destination argument, including zero or any nonzero bit pattern. It is not:

- The lookup result.
- The conditional dependency’s result.
- An indication that a destination update occurred.
- An indication that the final dependency succeeded.

Any return value produced by the final dependency is ignored.

### Directly accessed layout

The function uses the following destination-relative layout when the update condition succeeds:

| Byte offset | Width | Access |
|---|---:|---|
| 2 | 16 bits | Read, then write the value with mask `0x0100` set. |
| 4 | 32 bits | Write the conditional dependency’s result. |

These fields occupy distinct byte ranges: offsets 2–3 and 4–7. No direct access to offsets 0–1 is shown.

For a path that performs these updates, the destination must support the indicated 32-bit write at offset 4 and 16-bit read/write at offset 2. A record with at least eight accessible bytes is a conventional way to satisfy the extent requirement, but no allocation size or complete record definition is established.

Endianness, alignment requirements, and the target’s behavior on invalid or misaligned accesses remain unspecified.

## Behavior

### 1. Obtain the lookup result

The function first reads the 32-bit word at the source-word address.

It invokes F-e7c4e7dd076d with:

1. The word just read.
2. The lookup argument.

The returned value becomes the lookup result used throughout the remaining processing.

This read and call occur even when the destination is zero. A zero destination therefore does not make a null or otherwise invalid source-word address safe.

Only the initial 32-bit word is directly read through the source-word address by this function. Later access through that address may occur inside a dependency.

### 2. Decide whether to invoke the conditional dependency

F-039d16ce0621 is invoked only if both:

- The destination is nonzero.
- The lookup result is nonzero.

If either condition is false, the conditional dependency is skipped, and the function performs neither of its direct destination-field updates.

When invoked, F-039d16ce0621 receives:

1. The original source-word address.
2. The lookup result.
3. The 32-bit all-ones value, `0xFFFFFFFF`.

The constant’s bit pattern is established. Its interpretation as a signed value, sentinel, limit, flag combination, or other protocol value is not established.

The conditional dependency’s result is tested for zero. Every nonzero result takes the update branch; there is no visible positivity test, range check, address check, or error-code classification.

### 3. Perform the destination updates when enabled

If the conditional dependency returns nonzero, the function performs these direct accesses in order:

1. Store that result as a 32-bit value at destination byte offset 4.
2. Read the 16-bit value at destination byte offset 2.
3. Combine that 16-bit value with `0x0100` using bitwise OR.
4. Store the resulting 16-bit value back at destination byte offset 2.

Consequently, the offset-4 store precedes the offset-2 field read and write. The mask sets bit 8 of the 16-bit field. At the point of this write, every other bit has the same value as in the immediately preceding field read.

If the conditional dependency returns zero, none of these direct accesses occurs. In particular, the function does not clear either field to represent failure.

The field read occurs after the conditional dependency returns. If that dependency changes the field, the OR operation uses the changed value, not an entry-time snapshot.

### 4. Invoke the final dependency

After the conditional processing, F-94cb53877c15 is invoked once with the original lookup result.

This call occurs on every path that reaches it through normal execution, including when:

- The destination is zero.
- The lookup result is zero.
- The conditional dependency was skipped.
- The conditional dependency returned zero.
- The destination updates were performed.

There is no visible nonzero guard around this call. Thus, zero is passed to F-94cb53877c15 when the lookup returns zero. Whether that dependency accepts zero safely is a separate contract question.

The lookup result passed here is not replaced by the conditional dependency’s result.

### 5. Return the destination

After the final dependency returns normally, the function returns the original destination argument.

If an earlier dependency does not return normally, or an access faults, the supplied body establishes no completed return and no recovery path.

## Side effects

### Direct effects

The function’s direct memory operations are limited to:

- One initial 32-bit read through the source-word address.
- On the successful conditional branch, one 32-bit destination store followed by one 16-bit destination read and one 16-bit destination store.

No direct allocation, deallocation, global-variable access, loop, retry, or rollback operation is shown.

These facts do not bound the effects of dependencies. Any called dependency may have additional memory, global-state, allocation, ownership, or external effects not established here.

### State preservation limits

The direct 16-bit update preserves all bits other than setting bit 8 relative to the value read at that step. This is not a guarantee that all other destination state is preserved across the entire function.

In particular:

- Earlier dependencies may modify the destination through aliases or other reachable state.
- F-94cb53877c15 runs after the direct writes and may modify them.
- Dependencies may modify globals.
- Dependencies may invalidate storage whose later use this function assumes remains valid.

Accordingly, the direct writes are established events, but their persistence until return requires additional dependency guarantees.

### Aliasing and concurrency

No non-aliasing condition is shown between the source-word address and the destination.

If they alias, the initial lookup still uses the source word read before any direct destination updates. The conditional dependency receives the original address, so any later reads it performs may observe a different value.

The 16-bit field update consists of a read and a write. No atomic operation or synchronization is shown. Atomicity of the field update or of the two-field update must not be assumed. Concurrent observation and mutation require an external synchronization contract.

### Ownership

No ownership transfer is established for any input or result. The final call’s placement may suggest post-lookup housekeeping, but it does not prove that the lookup result is owned, borrowed, retained, released, or destroyed.

Safe execution requires whatever lifetime guarantees are necessary for subsequent accesses and calls. Those guarantees cannot be inferred solely from this function.

## Dependencies

| Opaque ID | Invocation and use | Unknown contract details |
|---|---|---|
| F-e7c4e7dd076d | Called once after the initial word read, with that word and the lookup argument. Its result gates further processing and is later passed to F-94cb53877c15. | Result meaning, failure conventions beyond the caller’s zero test, ownership, global effects, and memory effects. |
| F-039d16ce0621 | Called at most once, only when destination and lookup result are nonzero. Receives the source-word address, lookup result, and `0xFFFFFFFF`. A nonzero result is stored at destination offset 4. | Result meaning, interpretation of the all-ones argument, validation responsibilities, ownership, and mutation effects. |
| F-94cb53877c15 | Called once on every normally progressing path, after any direct destination updates. Receives the lookup result, including zero. | Zero acceptance, ownership or release behavior, memory effects, global effects, and abnormal termination behavior. |

The caller does not visibly inspect dependency status beyond the two zero/nonzero tests described above.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests. Each assumes valid accesses and normal dependency returns unless stated otherwise.

### Zero destination

Given a valid source-word address and destination zero:

- The source word is still read.
- F-e7c4e7dd076d is invoked.
- F-039d16ce0621 is not invoked, regardless of the lookup result.
- No direct destination-field access occurs.
- F-94cb53877c15 receives the lookup result.
- The return value is zero.

### Zero lookup result

Given a nonzero destination and a lookup result of zero:

- F-039d16ce0621 is skipped.
- No direct access to destination offsets 2 or 4 occurs.
- F-94cb53877c15 is called with zero.
- The original nonzero destination is returned.

This path does not itself establish that the destination changed or remained unchanged through dependency effects.

### Zero conditional result

Given a nonzero destination, nonzero lookup result, and conditional result zero:

- F-039d16ce0621 receives the original source-word address, lookup result, and `0xFFFFFFFF`.
- Neither direct destination-field update occurs.
- F-94cb53877c15 receives the lookup result, not the conditional result.
- The destination is returned unchanged as a value.

### Nonzero conditional result

Given a valid destination, nonzero lookup result, and conditional result `0x12345678`:

- A 32-bit store of `0x12345678` occurs at offset 4.
- The offset-2 field is then read.
- If that read yields `0x0025`, the subsequent 16-bit write is `0x0125`.
- The final dependency runs after those accesses.

The field values at function return are guaranteed only if the final dependency does not subsequently alter them.

### Already-set flag and high-bit result

If the offset-2 field read yields `0xA155`, the direct field write is also `0xA155`, because bit 8 is already set.

If the conditional result has the 32-bit pattern `0x80000000`, it is nonzero and takes the update branch. The visible condition does not reject it because of its high bit.

### Invalid source with zero destination

A zero destination does not bypass the initial source-word read. No successful return is established for an invalid source-word address.

## Uncertainties

The following remain **documented_only**:

- Exact machine ABI, pointer width, byte order, alignment rules, and invalid-access behavior. The supplied representation supports the stated 16-bit and 32-bit access widths, but does not independently establish the full platform ABI.
- Semantic roles of the source word, lookup argument, dependency results, destination fields, and bit 8.
- Whether nonzero dependency results can encode errors that this caller nevertheless stores.
- Meaning of `0xFFFFFFFF` to F-039d16ce0621.
- Whether dependencies preserve globals, destination storage, source storage, or object lifetimes.
- Whether F-94cb53877c15 releases anything, accepts zero, or changes the destination after the direct updates.
- Any synchronization, ownership, allocation, or validation obligations imposed outside this body.

An inferred intent is “lookup, conditionally attach a result, mark a field, then perform housekeeping.” That interpretation is not required by the established behavior and must not replace the explicit call and access contract.

## Evidence

The supplied static function body establishes:

- The initial 32-bit dereference and lookup call.
- The two nonzero guards controlling the conditional dependency.
- The all-ones 32-bit third argument.
- The conditional result test.
- The ordered 32-bit store and 16-bit read/OR/write.
- The subsequent final call using the original lookup result.
- The unchanged destination return value.

No dependency bodies, complete data-layout definitions, ABI documentation, or runtime observations were supplied. No runtime tests have been performed.
