<!-- Trusted provenance: F-c30df85ef8a0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c30df85ef8a0

## Purpose

F-c30df85ef8a0 coordinates two resource requests, initializes selected fields in the first returned object, invokes an indirect initialization callback, and performs a finalization or validation step.

The established behavior has two distinct successful exits:

- A finalization result of zero returns zero **without writing the caller’s output slot**.
- A finalization result of decimal 38 (`0x26`) normally publishes the first object through the output slot and returns zero.

All other ordinary paths enter an ordered cleanup sequence, clear the output slot, and return a status.

“Resource request,” “initialization,” and “finalization” describe inferred roles. The dependencies’ allocation mechanisms, resource ownership rules, and detailed semantics are not established by the supplied static evidence.

## Interface

The function accepts six arguments and returns an integer status.

| Argument | Role established by the function |
|---|---|
| Context address | Supplies a resource-domain value and a descriptor address through fixed-offset reads. Also stored in the first object. |
| First payload word | Stored in the first object and passed as the first argument to the initialization callback. |
| Second payload word | Passed as the third argument to the initialization callback. |
| Third payload word | Passed as the fourth argument to the initialization callback. |
| Fourth payload word | Passed as the fifth argument to the initialization callback. |
| Output-slot address | Receives the first object on the qualifying decimal-38 path, or zero after cleanup. It is not written on the direct-zero success path. |

The interface is represented using 32-bit integer and opaque-word operations. Addresses are carried in integer-like values in the supplied representation. This contract describes that represented 32-bit interface; it does not establish portability to another pointer width.

Payload words have no established signed interpretation. The return value is represented as an integer, but this function primarily distinguishes zero, decimal 38, and all other bit patterns. No general positive-versus-negative error convention is established.

### Required memory layouts

The context provides these readable fields:

| Byte offset | Width | Use |
|---|---:|---|
| `0x08` | 4 bytes | Resource-domain word passed to resource and cleanup dependencies. |
| `0x14` | 4 bytes | Descriptor address used for a request field and indirect callback lookup. |

The descriptor provides:

| Byte offset | Width | Use |
|---|---:|---|
| `0x24` | 4 bytes | Second argument to the first resource request. |
| `0x30` | Address-sized word in the represented interface | Initialization callback address. |
| `0x34` | Address-sized word in the represented interface | Cleanup callback address. |

After both resource requests report zero status, the first returned object must permit these writes:

| Byte offset | Width | Stored value |
|---|---:|---|
| `0x60` | 4 bytes | Context address. |
| `0x64` | 4 bytes | Resource-domain word. |
| `0x68` | 4 bytes | First payload word. |
| `0x80` | 4 bytes | Second resource-request result. |

The highest explicit object access occupies bytes `0x80` through `0x83`. These accesses require suitable storage; the function does not prove that the first request supplied it.

The output slot must support a 4-byte write on every path that publishes or clears it. The function does not read its previous contents.

### Preconditions

For defined ordinary execution:

- Every performed memory access must address valid storage.
- Each indirect callback selected at its actual lookup time must be callable with the stated arguments.
- A zero status from both resource requests must make the first object suitable for the four field writes.
- Failure-path dependencies must accept the values passed to them, including a first request result obtained with nonzero status and a zero second-resource value when no second request occurred.

The function performs no explicit null, range, alignment, object-size, or callback-validity checks. A null output-slot address can escape use on direct-zero success, but is not generally a valid interface argument.

## Behavior

### Initial reads and first request

The function first reads the resource-domain word from the context, then reads the descriptor address. Those two values are retained for later use; subsequent mutation of the context’s corresponding fields does not cause them to be reloaded.

It then reads the descriptor’s field at offset `0x24` and calls F-3ce007b692cc with:

1. The retained resource-domain word.
2. The descriptor field.
3. The address of a local 32-bit status slot.

The returned value becomes the first object value.

The local status slot is not explicitly initialized before this call. The caller subsequently reads it, so this use requires F-3ce007b692cc to establish a readable status value. The function does not independently validate the returned object.

If the first reported status is nonzero, the second request and initialization are skipped. That status is selected for cleanup.

### Second request

Only when the first status is zero does the function call F-3ce007b692cc again, using:

1. The same retained resource-domain word.
2. The literal value decimal 48 (`0x30`).
3. The same local status-slot address.

The return value becomes the second resource value. Before this call is reached, that value is represented as zero for failure-cleanup purposes.

The meaning of decimal 48 is not established. It may describe a size or another request parameter, but this function alone does not prove either interpretation.

If the second status is nonzero, initialization is skipped and that status is selected for cleanup.

Neither request has its return value checked independently of its status. In particular, a zero return value combined with zero status does not trigger a local rejection.

### Object initialization and callback

When both request statuses are zero, the function performs the four object writes in the order listed below:

1. Context address at offset `0x60`.
2. Resource-domain word at offset `0x64`.
3. First payload word at offset `0x68`.
4. Second resource value at offset `0x80`.

These writes occur before the initialization callback is looked up and invoked.

The callback address is read from offset `0x30` of the retained descriptor address. It is called with:

1. First payload word.
2. First object value.
3. Second payload word.
4. Third payload word.
5. Fourth payload word.

Its integer return value replaces the local status.

A nonzero callback result selects cleanup and prevents the finalization dependency from being called. A zero callback result permits finalization.

No other object bytes are explicitly initialized by this function. Any initialization of other fields must come from dependencies or pre-existing storage state.

### Finalization and success selection

After a zero initialization result, the function calls F-8a9775a593fb with the first object value.

Its result determines the next action:

| Finalization result | Observable action |
|---|---|
| Zero | Return zero immediately. Do not write the output slot. Do not run local cleanup. |
| Decimal 38 (`0x26`) | Write the first object value to the output slot, then reread the local status. Return zero if that status remains zero. |
| Any other value | Select that result as the cleanup status. |

The decimal-38 path deserves a separate distinction: publication occurs **before** the local status is reread. If that status is nonzero at the reread, execution proceeds to cleanup using the reread value, and cleanup eventually clears the output slot.

Under ordinary dependency behavior, the status still contains the initialization callback’s zero result, so decimal 38 produces publication and a zero return. However, the local status-slot address was previously exposed to F-3ce007b692cc. The supplied evidence does not establish that it cannot be retained or later modified.

The direct-zero finalization path does not perform that status reread. It returns zero even though no output publication occurs.

### Cleanup

Every ordinary path that does not take one of the successful exits performs the same cleanup sequence.

First, the selected status is written into the local status slot. The function then performs these calls in order:

1. F-6db2cf2b7ce6 with the first object value and retained resource-domain word.
2. The callback currently found at descriptor offset `0x34`, with the first object value.
3. F-bf7242c98d06 with the retained resource-domain word and second resource value.
4. F-bf7242c98d06 with the retained resource-domain word and first object value.

After those calls return, the function writes zero to the output slot and returns the current local status.

The cleanup callback address is read after F-6db2cf2b7ce6 returns. It is not cached at entry. Consequently, a change to the descriptor’s callback field before that lookup can affect which callback runs.

Cleanup is not conditional on a resource value being nonzero, on initialization having run, or on a dependency’s return value. Return values from the cleanup operations are not used.

If any dependency faults, fails to return, or transfers control elsewhere, the evidence does not establish that later cleanup calls or the output clear occur.

## Side effects

### Explicit memory effects

The function can:

- Read two context fields and three descriptor fields, with callback fields read only on applicable paths.
- Write four fields in the first object after both resource requests report zero.
- Write the output slot once on ordinary decimal-38 success.
- Write the output slot once after cleanup.
- Write the output slot twice on the exceptional decimal-38 path where publication is followed by a nonzero status reread.

There is no entry-time clearing of the output slot. Its previous value remains untouched throughout request and initialization processing unless a dependency changes it independently.

An early first-request failure still reaches cleanup with that request’s returned object value. The function does not substitute zero for it. When the second request was never made, its cleanup argument is zero.

### Ownership and lifetime

Successful paths do not invoke the function’s cleanup sequence. That establishes the absence of those calls, not ownership transfer or continued resource validity: dependencies may already have affected either resource.

Failure paths pass both resource values to F-bf7242c98d06, after other cleanup activity involving the first object. Actual deallocation, release ordering requirements, and tolerance for zero or failed-request results are documented_only.

No dependency is assumed to preserve globals, object contents, descriptor fields, or resources. The retained resource-domain word, descriptor address, and returned resource values remain the values used by this function unless an explicit operation changes them.

Aliasing can make the listed writes visible through other arguments or structures. No non-aliasing guarantee is supplied. In particular, an output slot overlapping object or descriptor storage can affect state beyond a standalone result variable.

### Arithmetic

The function uses fixed byte offsets, zero comparisons, and an equality comparison against decimal 38. It does not explicitly add the two request parameters, calculate an allocation extent, or perform payload arithmetic.

The memory accesses require valid effective addresses at the stated offsets. The evidence supplies no overflow detection or bounds enforcement.

## Dependencies

| Opaque dependency | Established interaction | Unestablished semantics |
|---|---|---|
| F-3ce007b692cc | Called once or twice with a resource-domain word, a request parameter, and the same status-slot address; each call returns an object/resource value. | Whether it allocates, what its request parameter means, status codes, ownership, initialization, and retention of the status address. |
| F-8a9775a593fb | Called once with the first object after zero initialization status; its result controls success or cleanup. | Validation versus registration versus another role, side effects, and meaning of decimal 38 outside this caller’s branching. |
| F-6db2cf2b7ce6 | First cleanup call; receives first object and resource-domain word. | Whether it releases subordinate state, mutates descriptor data, or changes resource lifetime. |
| F-bf7242c98d06 | Called twice during cleanup, first for the second resource and then for the first. | Release behavior, zero tolerance, failed-result tolerance, and memory ownership. |

Two additional dependencies are indirect descriptor callbacks. No opaque function IDs for their targets are supplied. Their argument lists and invocation order are established above; their implementation behavior is documented_only.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless stated otherwise, dependencies return normally, write the stated statuses, do not subsequently modify the exposed status slot, and use non-overlapping valid storage.

1. **First request reports a nonzero status.**  
   No second request, object initialization, initialization callback, or finalization call occurs. All four cleanup calls occur in order. The second-resource cleanup argument is zero; the first-object arguments are the first request’s return value. The output becomes zero, and the reported status is returned.

2. **First request succeeds; second request fails.**  
   No explicit object-field initialization occurs. Cleanup receives both returned resource values, including the second request’s value despite its failure status. The output becomes zero, and the second status is returned.

3. **Both requests succeed; initialization reports a nonzero status.**  
   All four field writes precede the callback. Finalization is not called. Cleanup runs, the output becomes zero, and the callback status is returned.

4. **Initialization succeeds; finalization returns zero.**  
   The function returns zero immediately. A pre-existing output value remains unchanged by this function. None of the cleanup calls occurs.

5. **Initialization succeeds; finalization returns decimal 38.**  
   The output receives the first object value. With the local status still zero, the function returns zero without cleanup.

6. **Initialization succeeds; finalization returns another value.**  
   A result such as decimal 39 selects cleanup. After normal cleanup, the output is zero and decimal 39 is returned.

7. **Decimal-38 publication is followed by a nonzero status reread.**  
   If the previously exposed local status has become nonzero, the first object is written to the output before cleanup begins. Cleanup then clears the output. With no later status modification, the reread status is returned rather than decimal 38.

8. **Finalization returns zero after modifying the exposed status indirectly.**  
   The function still takes the direct-zero exit: it neither checks that status again nor writes the output slot.

9. **The first cleanup dependency changes the cleanup callback field.**  
   The subsequent indirect call uses the callback value present at lookup time, provided the descriptor remains readable and the new target is valid.

## Uncertainties

The following remain documented_only:

- Dependency implementations, concrete callback identities, and global side effects.
- Whether either resource request actually allocates memory.
- The semantic meaning of the descriptor request field and decimal 48.
- Resource ownership on either successful exit.
- Why zero finalization success omits output publication.
- The external meaning of decimal 38.
- Whether request failures produce values acceptable to every cleanup dependency.
- Whether dependencies retain the local status-slot address or later write through it.
- Whether cleanup can alter the status ultimately returned.
- Object contents outside the four explicit writes.
- Endianness, alignment guarantees, and portability beyond the represented 32-bit interface.
- Behavior after invalid accesses, invalid callback targets, nonlocal exits, or incompatible dependency contracts.

The apparent lifecycle is acquisition, initialization, finalization, and ordered release on failure. That is inferred intent, not proof of allocation or ownership semantics.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. Established facts are its explicit accesses, argument forwarding, branch conditions, write ordering, call ordering, and return expressions.

No dependency bodies, runtime traces, external API contracts, or platform ABI documentation were supplied. No runtime tests have been performed. Acceptance examples describe consequences of the visible control flow under their stated conditions; they are not observations of execution.
