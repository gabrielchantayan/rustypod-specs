<!-- Trusted provenance: F-ad64f7eac2e3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ad64f7eac2e3

## Purpose

F-ad64f7eac2e3 performs two dependency-driven stages using a mutable context and an input object. Each stage publishes a 32-bit intermediate value through two ordered 16-bit writes, then calls a dependency whose zero result stops further processing.

After both checks succeed, the function writes a context marker. A nonzero unsigned dispatch value triggers flag-selected processing and a third publication. Whether or not that dispatch value is zero, the successful-check path ends by obtaining an external 32-bit value and storing it in the context.

The return value is a packed 64-bit quantity. Its low 32 bits contain either the initial argument or the captured dispatch value. Its upper 32 bits are constructed from two local 16-bit cells initially populated from the context pointer representation. Those cells can subsequently be exposed to dependencies by address.

The upper return word reflects the cells at return construction, not necessarily their values at the third publication or immediately after its associated dependency call. An external call and a context-field write still occur after that publication on successful-check paths.

These are structural behaviors established by the supplied static representation. Domain interpretations such as validation, resource management, or work scheduling are not established.

## Interface

### Inputs

| Input | Width and representation | Established use |
|---|---|---|
| Initial value | Unsigned 32-bit integer | First argument to F-603a53835f04; default low return word. |
| Context | 32-bit pointer representation | Supplies references and flag bytes; receives explicit marker and final-value writes. |
| Input object | 32-bit pointer representation | Supplies an indirect method; the original pointer is also passed to selected dispatch dependencies. |
| Initial temporary value | Opaque 32-bit value | Initializes an addressable temporary cell used in the first two stages. |

No explicit null, alignment, bounds, or object-type checks appear.

### Required layouts

The context must support every access reached on the selected path:

| Byte offset | Access |
|---|---|
| `0x000` | Read a 32-bit object reference on the threshold-based dispatch path. |
| `0x45E` | Write the byte value `1` after both checks succeed. |
| `0x464` | Write the final external dependency’s 32-bit result. |
| `0x73C` | Read the first dispatch flag when the dispatch value is nonzero. |
| `0x73D` | Conditionally read the second dispatch flag. |
| `0x73E` | Conditionally read the third dispatch flag. |

The first flag offset is `0x73C`: the source’s typed offset is scaled by the four-byte context element width. Likewise, the final field’s typed index denotes byte offset `0x464`.

The input object must provide a readable method-table reference, with an invocable method at table byte offset `0x08`. This invocation explicitly receives the original input-object pointer.

On the threshold-based path, the object referenced at context offset `0x000` must provide a method table with an invocable entry at byte offset `0x24`. No explicit arguments are shown for this invocation. Its actual calling convention and any implicit receiver remain unknown.

The destination returned by F-603a53835f04 must support 16-bit writes at byte offsets `0` and `2` for every publication reached. Its validity, capacity, and lifetime are not checked here.

These requirements are path-dependent. A first-check failure does not reach the input-object method, context marker, dispatch flags, or final context-field write.

### Return layout

The function returns a numerically assembled 64-bit value:

| Bits | Contents |
|---|---|
| `31:0` | Selected low return word. |
| `47:32` | First local 16-bit cell at return construction. |
| `63:48` | Second local 16-bit cell at return construction. |

Initially, the first cell holds the low 16 bits of the context pointer representation and the second holds its high 16 bits. If neither cell changes before return construction, the upper 32 return bits equal that original pointer representation.

The source establishes numeric bit placement, not an independent target byte-order contract. The two cells also form an addressable region used on selected paths; they must not be treated as an immutable saved pointer.

## Behavior

### Initial setup and first publication

Before the first dependency call, the function initializes:

- The two 16-bit cells from the context pointer representation.
- A pointer-sized cell containing the original input-object pointer.
- A 32-bit temporary cell containing the fourth argument.

F-603a53835f04 is then called with the initial value and three zero arguments. Its returned pointer is retained as the destination for all publications in this invocation.

F-5d7410a76c21 receives, in order, the temporary-cell address, the context pointer, and `1`.

After that call returns, the function performs these operations in order:

1. Writes the temporary cell’s low 16 bits to destination offset `0`.
2. Writes its high 16 bits to destination offset `2`.
3. Calls F-9b718979506d with the temporary-cell address.
4. Calls F-ec61551795ee with the destination.

The publication therefore precedes F-9b718979506d. A change to the temporary made by that dependency is not explicitly republished before the check.

If F-ec61551795ee returns zero, the function proceeds directly to return construction. The low return word is the initial value. No later-stage operations are explicitly performed.

Any nonzero check result permits continuation; its magnitude and sign have no further significance in this branch.

### Input-object processing and second publication

After the first check succeeds, the function invokes the input object’s method at table offset `0x08`, passing the original input-object pointer. The returned pointer replaces the pointer-sized local cell.

The address of that same cell is then passed sequentially to:

- F-70eba184e9ab, with additional argument `2`.
- F-65eb7f572de2, with additional argument `7`.

The first call supplies an opaque 32-bit auxiliary value. The second supplies an unsigned 32-bit dispatch value.

Because both receive the same cell address, the second call can observe changes made by the first. No intervening restoration or validation of that cell is shown. Later dispatch calls nevertheless receive the original input-object argument, not the current cell contents.

F-cd7148663427 receives the temporary-cell address, context pointer, auxiliary value, and dispatch value, in that order.

The function then repeats the publication sequence:

1. Write the temporary cell’s low 16 bits at destination offset `0`.
2. Write its high 16 bits at destination offset `2`.
3. Call F-9b718979506d with the temporary-cell address.
4. Call F-ec61551795ee with the destination.

The destination is not reacquired.

A zero second-check result leads directly to return construction with the initial value as the low return word. This remains true even if the captured dispatch value is nonzero. No explicit marker write, dispatch, or final-field update occurs on this path.

### Marker and zero-dispatch path

After both checks return nonzero, the function writes byte value `1` at context offset `0x45E`.

If the dispatch value is zero:

- No dispatch flag is read by this function.
- No dispatch dependency is selected.
- No third publication occurs.
- The low return word remains the initial value.

The function still performs the final external call and context-field write described below.

The marker write is an explicit event, not a guarantee that the byte remains `1` at return. Later dependencies or aliased writes may affect context memory.

### Flag-selected dispatch

For a nonzero dispatch value, flag reads occur after the marker write and all earlier dependency calls.

The read order is conditional:

- Read `0x73C`. If zero, take the threshold-based path without reading either later flag.
- Otherwise read `0x73D`. If nonzero, take the threshold-based path without reading `0x73E`.
- Otherwise read `0x73E`. If nonzero, take the threshold-based path.
- If the observed values are respectively nonzero, zero, and zero, call F-57c5776e9311.

F-57c5776e9311 receives:

1. The address of the first local halfword.
2. The original context pointer.
3. The original input-object pointer.
4. The captured auxiliary value.

The flags are not entry-time snapshots. Earlier calls and writes can affect the values subsequently observed.

### Threshold-based dispatch

This path reads the context’s first 32-bit reference, follows that object’s method table, and invokes the method at table byte offset `0x24`.

Its 32-bit result becomes the second argument to F-d7be9cfb4000. The first argument is `0x10000`, a 32-bit value equal to 65,536; it is not a 16-bit zero.

The result of F-d7be9cfb4000 is compared with the captured dispatch value as an unsigned 32-bit threshold:

- Dispatch value strictly below threshold: call F-e8e67147c5d5 with no explicit arguments.
- Dispatch value equal to or above threshold: call F-5e3b5e514705.

F-5e3b5e514705 receives the same four argument forms as F-57c5776e9311.

No subtraction, signed comparison, saturation, or threshold clamping is shown. A threshold of zero selects F-5e3b5e514705 for every dispatch value that reaches this comparison.

### Third publication

Every nonzero-dispatch path rejoins after its selected dependency returns. The function then:

1. Writes the current first halfword to destination offset `0`.
2. Writes the current second halfword to destination offset `2`.
3. Calls F-9b718979506d with the first halfword’s address.
4. Selects the captured dispatch value as the low return word.

There is no third call to F-ec61551795ee.

The halfword-region address is explicitly passed to F-57c5776e9311 or F-5e3b5e514705 when selected, and to F-9b718979506d after every third publication. F-e8e67147c5d5 receives no explicit address argument.

A publication is not a frozen snapshot for the eventual return. F-9b718979506d runs after the two writes and may alter the cells.

### Final external call, field write, and return

After both checks succeed, including when the dispatch value is zero, the function calls an unidentified external dependency with no explicit arguments.

Its 32-bit result is written at context offset `0x464`. Only after that write does the function construct the packed return value.

Consequently, any expected upper return word must account for possible modifications through this final interval. In particular, no contract establishes that the final external dependency cannot access a previously escaped halfword-region address. No non-aliasing contract establishes that the final context-field write cannot affect that region.

The return uses the halfword values present at return construction. Values observed immediately after F-9b718979506d determine the upper return word only if neither cell is subsequently modified.

## Side effects

The explicit writes are:

- Two ordered 16-bit destination writes for the first publication.
- Two more for the second publication, if reached.
- A byte write of `1` at context offset `0x45E` after both checks succeed.
- Two further 16-bit destination writes if the dispatch value is nonzero after both checks succeed.
- A 32-bit write at context offset `0x464` after the final external call returns.

The paired destination writes are separate operations, not an established atomic 32-bit update. Their ordering matters if memory aliases or external observation is possible.

The function neither reacquires nor explicitly releases the publication destination. It contains no explicit allocation, deallocation, rollback, exception handler, or error translation. F-9b718979506d’s repeated use does not establish a destruction or ownership-transfer rule.

Dependency effects remain additional to these explicit writes. Dependencies may mutate memory accessible to them or retain supplied addresses; their contracts are not supplied. Retention is not established, but preservation cannot be assumed.

Early returns retain already-performed writes and dependency effects. “No explicit write” on an early-return path does not mean dependencies could not have changed the same memory.

## Dependencies

| Dependency | Established call-site role |
|---|---|
| F-603a53835f04 | Supplies the publication destination from the initial value and three zeros. |
| F-5d7410a76c21 | Receives temporary-cell address, context, and `1` before the first publication. |
| F-9b718979506d | Receives the corresponding local-storage address after each publication. |
| F-ec61551795ee | Receives the destination after the first two publications; zero stops further processing. |
| F-70eba184e9ab | Receives mutable pointer-cell address and `2`; supplies the auxiliary value. |
| F-65eb7f572de2 | Receives the same cell address and `7`; supplies the unsigned dispatch value. |
| F-cd7148663427 | Receives temporary-cell address, context, auxiliary value, and dispatch value. |
| F-57c5776e9311 | Handles the nonzero/zero/zero flag combination. |
| F-d7be9cfb4000 | Receives `0x10000` and the context-associated method result; supplies the threshold. |
| F-e8e67147c5d5 | Handles the strict unsigned-below-threshold branch without explicit arguments. |
| F-5e3b5e514705 | Handles equality and unsigned-above-threshold cases. |

The two indirect methods and the final external dependency have no supplied opaque IDs. Their identities, full signatures, ownership rules, and broader effects remain documented_only.

## Acceptance examples

These are conditional static expectations, not executed tests. All assume valid reached memory and normally returning dependencies.

1. **First check is zero.**  
   One publication, its F-9b718979506d call, and the first check occur. The input-object method is not invoked. The low return word equals the initial value. This function performs no explicit marker or final-field write.

2. **Second check is zero.**  
   Both publications and both pointer-cell consumers occur. The low return word remains the initial value regardless of the captured dispatch value. No dispatch or final external call occurs.

3. **Both checks succeed; dispatch value is zero.**  
   The marker write occurs, but no flag read or third publication occurs. The external result is stored at `0x464`. The low return word remains the initial value, not necessarily zero.

4. **Observed flags are nonzero, zero, zero.**  
   With a nonzero dispatch value, F-57c5776e9311 runs. The threshold-producing method and F-d7be9cfb4000 do not. The third publication follows, and the low return word becomes the captured dispatch value.

5. **Threshold equality or unsigned boundary.**  
   Equality selects F-5e3b5e514705. Dispatch `0xFFFFFFFF` with threshold `1` also selects it. Dispatch `1` with threshold `0xFFFFFFFF` selects F-e8e67147c5d5.

6. **Short-circuited flag reads.**  
   An observed zero at `0x73C` selects threshold processing without reading `0x73D` or `0x73E`. Nonzero values at both `0x73C` and `0x73D` likewise avoid reading `0x73E`.

7. **Publication differs from return.**  
   Suppose the first and second halfwords are `0x1234` and `0x5678` when the third publication reads them, with no intervening modification between those writes. Those values are published in that order. If F-9b718979506d then changes them to `0xABCD` and `0xEF01`, the upper return word is `0xEF01ABCD` **only if neither halfword changes afterward**, including during the final external call and the write at `0x464`. Otherwise, the upper word uses their values at return construction. For example, if that later interval leaves them as `0x1111` and `0x2222`, with no further change before construction, the upper word is `0x22221111`.

## Uncertainties

The following remain documented_only:

- Domain meanings of the fields, auxiliary value, dispatch value, threshold, and check results.
- Complete dependency signatures and calling conventions.
- Whether destination storage is borrowed, allocated, shared, or globally backed.
- Ownership, release, and address-retention behavior.
- Whether escaped local addresses remain accessible to later dependencies.
- Whether dependencies preserve globals, context contents, or local storage accessible through supplied or retained addresses.
- Non-aliasing relationships among destination, context, objects, and addressable local regions.
- Meaning of the final external result.
- Higher-level interpretation of the packed return value.
- Behavior under invalid pointers, exceptions, nonreturning calls, or concurrent mutation.

The lack of explicit recovery does not establish how external failures manifest. Unknown dependency behavior must not be replaced with preservation, cleanup, or ownership assumptions.

## Evidence

This contract is based on the supplied static representation and opaque dependency mapping. Established evidence includes the operand widths, scaled field offsets, ordered writes, short-circuited flag reads, two zero-result gates, unsigned threshold comparison, dependency argument forms, and packed return construction.

The final external call and context-field write precede return construction. This ordering establishes why post-publication halfword values are not unconditional return predictions.

“Publication,” “check,” and “dispatch” describe observable structure. They do not establish application intent. No runtime tests have been performed; all acceptance examples are conditional static expectations.
