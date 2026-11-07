<!-- Trusted provenance: F-eedd60b3bafd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eedd60b3bafd

## Purpose

F-eedd60b3bafd selects one of four processing routes using a byte in a context object and, when that byte does not force direct dispatch, a shared 32-bit selector.

Three routes forward work to another operation. The remaining route obtains an intermediate result through an indirect operation, copies and transforms parts of that result using temporary storage, writes a derived result through the fourth argument, and performs explicitly ordered cleanup calls.

This is a static behavioral contract. It establishes dispatch conditions, argument forwarding, visible storage operations, and dependency-call ordering. It does not establish the application-level meaning of the selector, the result, or the temporary objects.

## Interface

The function has four arguments and no defined return value.

| Argument | Observed representation | Role |
|---|---|---|
| Context | 32-bit address-bearing value | Supplies a byte at offset `0x2EC`; also forwarded to F-39d61164a730 on one route. |
| Target | Pointer to a target object whose first word is a table address | Supplies indirect operation entries; forwarded as the receiver to those operations. |
| Input | Opaque 32-bit value | Forwarded unchanged to the selected dependency. Its interpretation is dependency-specific. |
| Result argument | Opaque 32-bit value | Forwarded unchanged on direct routes; used as a destination address in the transformation route. |

“Unchanged” describes the argument’s bit pattern at the call boundary. It does not imply that the object or memory denoted by that argument remains unchanged.

### Relevant storage layout

The following offsets and widths are established by the supplied static representation:

- Context offset `0x2EC`: one byte tested for equality with `1`.
- Shared selector object offset `0x1C`: a 32-bit selector.
- Target offset `0`: a table address.
- Table offsets `0x150`, `0x168`, and `0x19C`: indirect operation entries used by different routes.

The table offsets are byte offsets. No named class, inheritance model, or complete table layout is established.

The transformation route uses three distinct areas of automatic storage:

| Region | Represented layout | Observed use |
|---|---|---|
| Accumulator | Four-byte leading region, followed by eight bytes, followed by a four-byte trailing word | Initialized by F-b02b795a8319; receives an operation on its eight-byte interior region and a direct trailing-word assignment; passed to F-ee5919ca4be6 at the end. |
| Intermediate result | Four-byte leading region, followed by eight bytes, followed by a four-byte trailing word | Supplied as the destination of the indirect operation at table offset `0x150`; its interior and trailing word feed the accumulator; passed to F-ee5919ca4be6 before transformation. |
| Transformation temporary | Eight bytes | Receives the result of F-b9acea4b06ad, supplies input to F-46f0b3cb316e, and is then passed to F-289ec4311c64. |

These are represented storage extents, not independently verified type definitions. In particular, the eight-byte interior region and adjacent four-byte trailing word are not proven to be independent application-level objects.

### Preconditions

For defined execution through the selected route:

- The context must support the one-byte read at offset `0x2EC`.
- Unless that byte equals `1`, the shared selector location must support a 32-bit read.
- Any route using an indirect operation requires a readable target table address and a callable entry at the selected table offset.
- Arguments forwarded to dependencies must satisfy those dependencies’ contracts.
- On the transformation route, the result argument must be usable as the destination expected by F-46f0b3cb316e.

No null checks, bounds checks, destination-capacity checks, or indirect-entry validation are visible.

## Behavior

### Dispatch precedence

The context-byte test has precedence over the shared selector.

1. The function reads the byte at context offset `0x2EC`.
2. If the byte equals exactly `1`, it selects the indirect operation at table offset `0x168`.
3. Otherwise, it reads the shared 32-bit selector and chooses a route:

| Selector | Route |
|---|---|
| `0` | Call F-39d61164a730 |
| `1` | Call indirect operation at table offset `0x168` |
| `2` | Execute the transformation route |
| `3` | Call indirect operation at table offset `0x19C` |
| `4` | Call F-39d61164a730 |
| Any other 32-bit value | Call indirect operation at table offset `0x168` |

The context-byte comparison is equality, not a general truth test. A byte of `2` or `255` does not itself force the `0x168` route.

When the byte equals `1`, the function does not explicitly read the shared selector. This is an observable distinction if that selector is volatile, externally monitored, or otherwise access-sensitive.

The selector uses exact comparisons against the listed values. There is no range normalization or arithmetic transformation. Both selector values `0` and `4` take the same route; unlisted values take the default indirect route.

### Direct indirect-operation routes

For the operation at table offset `0x168`, the function obtains the target’s table address, obtains that entry, and invokes it with:

1. The target.
2. The opaque input.
3. The result argument.

It then returns if the operation returns normally.

The route for table offset `0x19C` has the same argument sequence and return behavior, differing only in the selected table entry.

The context is not forwarded on either route. The function performs no explicit result construction, post-call copying, or cleanup around either call.

Any output writes, target mutations, allocation, ownership changes, or failures on these routes belong to the selected indirect operation’s behavior and are not established here.

### F-39d61164a730 route

For selector `0` or `4`, provided the context byte did not force indirect dispatch, the function invokes F-39d61164a730 with all four original arguments in their original order:

1. Context.
2. Target.
3. Opaque input.
4. Result argument.

The function then returns if that dependency returns normally. There is no visible local postprocessing or cleanup on this route.

### Transformation route

For selector `2`, provided the context byte is not `1`, the function performs the following ordered operations.

1. **Initialize the accumulator.**  
   F-b02b795a8319 receives the base address of the accumulator region.

2. **Request an intermediate result.**  
   After the initializer returns, the function reads the target’s table address and the indirect operation entry at offset `0x150`. It invokes that entry with:
   - The base address of the intermediate-result region.
   - The target.
   - The opaque input.

   The result argument supplied to F-eedd60b3bafd is not passed to this operation.

3. **Apply an operation between the interior regions.**  
   F-46f0b3cb316e receives:
   - The accumulator address four bytes after its base.
   - The intermediate-result address four bytes after its base.

   This call precedes the explicit trailing-word read described next.

4. **Transfer the trailing word.**  
   The function reads the 32-bit word twelve bytes after the intermediate-result base and writes that value twelve bytes after the accumulator base.

   This is an explicit four-byte transfer. The leading four-byte region is not explicitly copied. The contract of F-46f0b3cb316e may account for other effects, but those effects are not known.

5. **Clean up the intermediate result.**  
   F-ee5919ca4be6 receives the intermediate-result base address.

6. **Produce the transformation temporary.**  
   F-b9acea4b06ad receives:
   - The base address of the eight-byte transformation temporary.
   - The accumulator address four bytes after its base.

   This call occurs after intermediate-result cleanup and before accumulator cleanup.

7. **Write through the result argument.**  
   F-46f0b3cb316e receives:
   - The original result argument as destination.
   - The transformation-temporary base address as source.

8. **Clean up the transformation temporary.**  
   F-289ec4311c64 receives the transformation-temporary base address.

9. **Clean up the accumulator.**  
   F-ee5919ca4be6 receives the accumulator base address.

The function then returns normally.

This ordering matters. In particular:

- The target’s indirect entry is resolved after accumulator initialization.
- The intermediate trailing word is read after the first F-46f0b3cb316e call.
- Intermediate cleanup completes before F-b9acea4b06ad is called.
- The destination operation completes before transformation-temporary cleanup.
- Accumulator cleanup is last.

The function does not reread the selector after choosing this route.

## Side effects

### Explicit reads and writes

The function explicitly reads the context byte, conditionally reads the selector, and reads the target table and the selected indirect entry when required.

On the transformation route, it additionally performs the explicit four-byte trailing-word transfer between the intermediate result and accumulator.

Other writes to temporary storage or caller-visible memory occur through dependencies. In particular, the result argument is used as a destination only by the second F-46f0b3cb316e call on that route.

There is no explicit write to the shared selector or context field in the supplied body. This does not establish that dependencies leave either location unchanged.

### Ownership and lifetime

The three temporary regions have automatic-storage lifetime within this invocation. Their addresses are passed to dependencies, but the function contains no explicit heap allocation or deallocation operation of its own.

The call pattern is consistent with initialization, value transfer, transformation, and destruction. Those meanings remain inferred intent unless dependency contracts establish them.

The observable contract is the call sequence, not a promise that any dependency:

- Performs a deep copy.
- Retains or releases a reference.
- Allocates or frees memory.
- Leaves its source unchanged.
- Avoids retaining a temporary’s address.
- Preserves global state.

If a dependency retains an address into automatic storage, that storage does not remain valid after the invocation returns.

### Errors and abnormal completion

There is no explicit error return, status inspection, retry, or recovery branch.

If a dependency fails to return normally, later operations in the sequence are not established to execute. In particular, the supplied representation does not establish cleanup during exceptions, nonlocal exits, faults, or process termination.

Invalid addresses or non-callable indirect entries have no defined handling in this contract. No specific fault type or diagnostic is promised.

## Dependencies

| Dependency | Established use | Not established |
|---|---|---|
| F-b02b795a8319 | Called once with the accumulator base on the transformation route, before the intermediate-result operation. | Initialization values, allocation, failure behavior, global effects. |
| F-46f0b3cb316e | Called first between interior temporary regions, then from the transformation temporary to the caller’s result destination. | Exact write extent, copy versus assignment semantics, source mutation, ownership effects, aliasing rules. |
| F-ee5919ca4be6 | Called on the intermediate-result base and later on the accumulator base. | Exact cleanup semantics, resource release, failure behavior. |
| F-b9acea4b06ad | Receives the transformation-temporary destination and the accumulator interior address. | Transformation meaning, output representation, allocation or ownership effects. |
| F-289ec4311c64 | Called on the transformation temporary after its use as the final source. | Exact cleanup semantics and resource release. |
| F-39d61164a730 | Receives all four original arguments for selector `0` or `4`. | Its outputs, mutations, ownership rules, and errors. |

Three additional dependencies are indirect and have no supplied opaque function IDs. They are identified solely by target-table offsets:

- `0x150`: receives intermediate destination, target, and input.
- `0x168`: receives target, input, and result argument.
- `0x19C`: receives target, input, and result argument.

No dependency is assumed to preserve globals or unrelated reachable memory. Where a read occurs after a dependency call, its value is the value present at that later point, not necessarily the entry value.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. They assume valid storage and dependencies that return normally unless stated otherwise.

### Context override

Given a context byte of `1`, the function must invoke the target operation at table offset `0x168` with target, input, and result argument. It must not explicitly read the shared selector or perform the transformation sequence.

This applies regardless of what selector value would otherwise be present.

### Non-Boolean context byte

Given a context byte of `2` and selector `3`, the function must select table offset `0x19C`, not the context-override route.

### Shared forwarding route

Given a context byte other than `1` and selector `0` or `4`, the function must call F-39d61164a730 with all original arguments. Neither selector may enter the transformation route.

### Unlisted selector

Given a context byte other than `1` and selector bit pattern `0xFFFFFFFF`, the function must select table offset `0x168`. No signedness interpretation changes that default outcome.

### Transformation ordering

Given a context byte other than `1` and selector `2`, the dependency order must be:

1. F-b02b795a8319.
2. Indirect operation at `0x150`.
3. F-46f0b3cb316e on the interior regions.
4. Explicit trailing-word transfer.
5. F-ee5919ca4be6 on the intermediate result.
6. F-b9acea4b06ad.
7. F-46f0b3cb316e into the result destination.
8. F-289ec4311c64.
9. F-ee5919ca4be6 on the accumulator.

### Mutation before a later read

If the first F-46f0b3cb316e call changes the intermediate trailing word through reachable memory, the subsequent explicit transfer must use the changed word. A contract that snapshots that word before the call would contradict the supplied ordering.

Likewise, if F-b02b795a8319 changes the target’s table address, the subsequent `0x150` lookup uses the table address read after that call.

### Interrupted transformation

If the indirect operation at `0x150` does not return normally, this body does not establish that either F-ee5919ca4be6 call occurs. Cleanup-on-failure must not be accepted as a guarantee without additional evidence.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of the context byte, selector values, input, and result.
- Exact native types and signedness beyond the represented widths.
- Complete object layouts, alignment requirements, and destination capacity.
- Whether the temporary layouts represent composite objects, hidden return storage, or another ABI convention.
- Exact effects and read/write extents of every dependency.
- Aliasing restrictions among caller-provided objects and dependency-reachable storage.
- Thread safety, synchronization, and reentrancy.
- Exception propagation and cleanup during abnormal control transfer.
- Whether indirect dispatch is implemented as an ordinary call, tail transfer, or another equivalent machine-level mechanism.

The operation at `0x150` appears to produce a result into caller-provided temporary storage. The paired initialization and cleanup calls appear to manage object lifetimes. Both are inferred intent, not independently established semantic contracts.

## Evidence

Established static evidence consists of the supplied function representation and dependency-ID mapping:

- The context-byte equality test precedes selector access.
- The selector dispatch distinguishes `0`, `1`, `2`, `3`, and `4`, with an explicit default route.
- Indirect operations use target-table offsets `0x150`, `0x168`, and `0x19C`.
- The transformation route has the argument relationships and call ordering specified above.
- The explicit trailing transfer is 32 bits.
- No explicit validation, status handling, or recovery branch is present.

No runtime tests have been performed. No independent dependency implementations, ABI documentation, or memory-layout definitions were supplied. Accordingly, dispatch and sequencing are established; application semantics, resource ownership, and dependency failure behavior remain limited as documented above.
