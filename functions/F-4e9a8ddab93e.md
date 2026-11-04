<!-- Trusted provenance: F-4e9a8ddab93e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4e9a8ddab93e

## Purpose

F-4e9a8ddab93e conditionally transforms information from a source object into a destination object. It recognizes two selector conditions, optionally invokes a source-to-scratch transformation followed by an external destination operation, and, for the second selector condition only, transfers two four-byte fields from source to destination.

A destination-preparation dependency is invoked before selector recognition. Consequently, an unsupported selector is not a no-operation: the preparation call has already occurred before the function returns its unsupported-selector result.

The established behavior is the sequence of calls, conditional reads, explicit field transfers, and return-value selection described below. The meanings of the selector constants, source flag, external operation, and destination format are not established. In particular, the preparation call is not proven to clear memory, and the external operation is not proven to copy or format a string.

## Interface

### Inputs

The function takes three arguments:

| Argument | Established use |
|---|---|
| Selector | A 32-bit integer participating in selector arithmetic with values from global storage. |
| Source address | An address represented through a 32-bit argument. Used as the base for byte and four-byte reads and passed to a dependency on selected paths. |
| Destination address | An address represented through a 32-bit argument. Passed to two dependencies and used as the base for four-byte writes on one selected path. |

The available representation uses integer-typed arguments for addresses. That representation does not establish a portable pointer interface, address-space rules, alignment guarantees, or a higher-level object type.

There is no supplied source length or destination capacity.

### Return value

The function returns a 32-bit value:

- `0x00000000` after either recognized selector path completes normally.
- `0xFFFFFFCE` after the unsupported-selector path completes normally.

The latter bit pattern represents −50 when interpreted as a signed 32-bit two’s-complement integer. No particular named error category is established.

The function does not inspect dependency return values. A zero return therefore establishes only that the function selected a recognized path and reached its return; it does not establish that dependencies successfully produced valid output.

### Observed layout

All offsets below are byte offsets from the applicable argument address.

| Object | Offset | Access |
|---|---:|---|
| Source | `0x25` / 37 | One-byte condition read on recognized paths. |
| Source | `0x28` / 40 | Four-byte read on the second recognized path. |
| Source | `0x2C` / 44 | Four-byte read on the second recognized path. |
| Destination | `0x10C` / 268 | Four-byte write on the second recognized path. |
| Destination | `0x110` / 272 | Four-byte write on the second recognized path. |

The two explicit destination fields occupy bytes 268–275. The highest directly accessed source byte is byte 47.

These spans describe direct accesses only. Dependencies may access additional memory. The preparation dependency receives the numeric argument `0x204`, or 516, but its meaning as a byte count, capacity, operation code, or other parameter is not established.

The function also provides a dependency with a 260-byte automatic scratch region. No initialization of that region is visible before the dependency receives it.

## Behavior

### Entry and preparation

The function first establishes a zero return result and invokes F-c67cb20988a1 with:

1. The destination address.
2. The numeric value 516.

This call precedes all selector-dependent source reads and all explicit destination-field writes.

After that call returns, the function reads a four-byte global value used as the third argument to the later external operation. This value is retained for that operation if the operation occurs. The global read occurs even when the selector will subsequently be rejected or the source condition byte will be zero.

Because F-c67cb20988a1 runs first, any global changes it makes can affect the captured external-operation argument and the subsequent selector evaluation. The available evidence does not justify assuming this dependency preserves globals.

### Selector recognition

Two global selector-related values participate in recognition. They are described here as the **base adjustment** and **secondary adjustment**; these names express their arithmetic roles, not established semantic meanings.

The first recognized condition is that the selector plus the base adjustment equals zero.

Only if that condition is false does the function evaluate the second condition: the selector plus the base adjustment plus the secondary adjustment equals zero.

The base adjustment appears in both condition evaluations. It must not be treated as a guaranteed single snapshot across them. The secondary adjustment is needed only for the second condition.

The first condition has precedence. If both conditions would be true, only the first path executes; the field transfers belonging to the second path do not occur.

The arithmetic is represented as 32-bit integer arithmetic. Its exact overflow semantics are not independently established by the supplied representation. For values whose additions remain within signed 32-bit range, the conditions above are unambiguous. If the target implements wrapping 32-bit additions, the conditions instead apply modulo \(2^{32}\). A portable signed-overflow rule must not be invented.

### First recognized path

The function reads the source byte at offset 37.

If that byte is zero, the function performs no source-to-scratch call and no external destination call.

If that byte is nonzero, the function:

1. Invokes F-32b90d7f6546 with the source address and the address of the 260-byte scratch region.
2. After that call returns, invokes the external dependency with the destination address, the scratch-region address, and the previously captured four-byte global value.

The condition is zero versus nonzero. It is not restricted to a particular nonzero flag value, and character signedness does not change this condition.

This path performs neither of the explicit four-byte source-to-destination transfers. If execution reaches the return normally, the return value is zero.

### Second recognized path

The function first performs the same source-byte check and optional pair of dependency calls as the first recognized path.

After that optional work completes—or immediately after the byte check when the byte is zero—the function performs two ordered transfers:

1. Read four bytes from source offset 40 and write that value to destination offset 268.
2. Read four bytes from source offset 44 and write that value to destination offset 272.

These are four-byte value transfers, not an established aggregate-object copy. No numeric conversion, byte swapping, or field interpretation is visible.

The ordering is significant. The second source-field read occurs after the first destination-field write. The function does not visibly preload both source fields before writing either destination field.

If execution reaches the return normally, the return value is zero.

### Unsupported-selector path

If neither selector condition holds, the function selects return value `0xFFFFFFCE`.

On this path, it performs no direct source-byte read, no direct source-field read, no explicit destination-field write, and neither of the optional dependency calls.

The earlier preparation call and four-byte global capture have nevertheless occurred. Destination state is therefore not guaranteed unchanged.

## Side effects

### Destination changes and partial effects

Direct destination writes occur only on the second recognized path. They affect the two four-byte fields at offsets 268 and 272, in that order.

Other destination effects are dependency-defined:

- F-c67cb20988a1 receives the destination on every path.
- The external dependency receives it when a recognized path observes a nonzero condition byte.

There is no visible rollback. A dependency failure, abnormal termination, or invalid memory access can leave earlier effects in place. For example, on the second recognized path, the first explicit destination write may complete before a fault on the second source-field read.

### Source and destination overlap

No non-overlap check is visible. Overlap can affect results:

- The initial preparation call may affect source data before the condition byte is read.
- The optional dependency calls may affect source fields before their explicit reads.
- The first explicit destination write may overlap and change the source bytes used by the second transfer.

The contract therefore preserves the stated access order rather than promising an original-source snapshot. Whether any overlapping arrangement is supported by the dependencies remains unknown.

### Scratch storage and ownership

The function supplies temporary automatic storage of 260 bytes. Its lifetime extends through the calls made by this invocation, but not beyond the function’s return.

There is no visible allocation, ownership transfer, or explicit release by this function. This does not establish that dependencies neither allocate nor release other memory. It also does not establish that they refrain from retaining supplied addresses.

Defined use of the scratch region requires dependency behavior compatible with its capacity and lifetime. There is no visible guard, capacity argument to F-32b90d7f6546, or validation of the scratch contents before the external call.

### Preconditions and error handling

No explicit null, range, alignment, capacity, or access-permission validation is visible.

Required memory validity depends on the path:

- Every invocation must satisfy F-c67cb20988a1’s destination requirements.
- Recognized paths require a readable source byte at offset 37.
- A nonzero condition byte additionally requires valid calls to the two optional dependencies.
- The second recognized path requires the two four-byte source reads and destination writes to be valid when reached.

The unsupported-selector result is the only explicit error result. Dependency failures are not translated into a different return value.

## Dependencies

| Opaque ID | Invocation and established role |
|---|---|
| F-c67cb20988a1 | Called once before selector evaluation with the destination address and 516. Its concrete operation and memory footprint are unknown. |
| F-32b90d7f6546 | Called only when a recognized path observes a nonzero byte at source offset 37. Receives the source address and scratch address. Its output format, bounds behavior, and side effects are unknown. |
| F-unmapped-1 | Document-local opaque label for the external dependency whose mapped function ID was not supplied. Called after F-32b90d7f6546 returns, with destination address, scratch address, and the previously captured four-byte global value. Its operation and argument semantics are unknown. |

No dependency return value controls subsequent behavior.

If F-32b90d7f6546 changes the global from which the external-operation argument was captured, that change does not replace the already retained argument. It may still have other effects.

## Acceptance examples

These are conditional static acceptance examples, not reports of executed tests. Normal dependency return and valid required memory accesses are assumed unless stated otherwise.

1. **First selector condition, zero condition byte.**  
   Choose selector-related values whose first sum is zero without overflow. Let source offset 37 contain zero. The preparation call occurs, the external-operation argument is captured, and the source byte is read. Neither optional dependency is called, neither explicit destination field is written, and the return is zero. No claim is made about destination changes caused by preparation.

2. **First selector condition, nonzero condition byte.**  
   With the first condition satisfied and source offset 37 containing `0x80`, both optional dependencies are invoked in order. The external dependency receives the retained global argument, not a fresh read after the scratch-producing call. No explicit field transfer occurs. The return is zero.

3. **Second condition, zero condition byte.**  
   Let the selector be 10, the base adjustment −3, and the secondary adjustment −7. The first sum is 7; the second is zero. With a zero condition byte, the optional calls are skipped. The four-byte value at source offset 40 is transferred first, followed by the value at offset 44, to destination offsets 268 and 272 respectively. The return is zero.

4. **Both conditions mathematically true.**  
   Let the selector and both adjustments be zero. The first path wins. Even though the second condition would also be true, its explicit field transfers do not occur.

5. **Unsupported selector.**  
   Let the selector be 10, the base adjustment −3, and the secondary adjustment −6. Neither sum is zero. Preparation and global capture still occur. There are no direct source reads or explicit field writes, and the return bits are `0xFFFFFFCE`.

6. **Observable overlapping transfers.**  
   On the second recognized path, suppose the address at destination offset 268 equals the address at source offset 44. With the optional calls skipped, the first transfer overwrites the bytes that the second transfer will subsequently read. Acceptance requires respecting that order, not copying both original source values.

7. **Preparation changes selector-related globals.**  
   If F-c67cb20988a1 changes the relevant globals before returning, recognition and argument capture use the post-call state. Evaluating the selector before preparation would not preserve the observed ordering.

## Uncertainties

The following remain **documented_only**:

- Concrete values, meanings, and mutability of the three global values.
- Exact overflow behavior of selector arithmetic and address calculations.
- Target byte order and alignment requirements.
- The preparation dependency’s operation and interpretation of 516.
- The scratch-producing dependency’s required source layout, maximum output size, and termination conventions.
- The external dependency’s mapped identity, semantics, memory footprint, and failure behavior.
- Whether dependencies mutate globals, source storage, destination storage, or other reachable state.
- Whether dependencies retain pointers or manage memory ownership.
- Supported concurrency, aliasing, and reentrancy conditions.
- Any semantic interpretation of the source condition byte or transferred fields.

An initialization-and-conversion workflow is a plausible intent, but is not an established contract. Names such as “clear,” “format,” “string,” or “copy” would assert behavior not demonstrated for the unknown dependencies.

## Evidence

This contract is based solely on static inspection of the supplied function representation and dependency mapping. Established observations are the argument uses, access widths and offsets, branch precedence, call ordering, scratch capacity, explicit transfers, and return bit patterns.

No dependency implementation was supplied. No runtime tests have been performed. The acceptance examples state conditional consequences of the visible control flow; they do not claim observed execution.
