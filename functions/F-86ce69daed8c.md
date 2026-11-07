<!-- Trusted provenance: F-86ce69daed8c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-86ce69daed8c

## Purpose

F-86ce69daed8c conditionally dispatches one of five operations against a record selected from an eight-element identifier range. Each recognized operation prepares a temporary representation of caller-provided data and passes that representation to an operation-specific dependency.

The function also has a distinct fallback behavior: when a context exists but the identifier is outside the supported range, it changes the context’s first 32-bit word from zero to `0x00000500`. A nonzero value is left unchanged.

These are established control-flow and data-flow properties. Interpreting the records as graphical objects, the payloads as geometric values, or the fallback word as an error code would be inferred intent, not an established contract.

## Interface

### Inputs

The function receives three arguments:

| Argument | Width and interpretation | Use |
|---|---|---|
| Record identifier | 32-bit integer bit pattern | Determines whether dispatch is permitted and contributes to the selected-record address. |
| Operation selector | 32-bit value | Recognized values are `0x1200` through `0x1204`. |
| Payload address | Pointer to 32-bit words | Supplies operation-dependent input. Its required readable extent depends on the selector and execution path. |

The supported record identifiers are exactly `0x4000` through `0x4007`, inclusive.

The payload’s established direct-read requirements are:

- Selectors `0x1200`, `0x1201`, and `0x1202`: the payload address is passed to F-fa3878f0b526. That dependency’s input requirements are not established here.
- Selector `0x1203`: four consecutive 32-bit words, occupying 16 bytes, are copied directly by this function.
- Selector `0x1204`: three consecutive 32-bit words, occupying 12 bytes, are copied directly by this function.
- An unrecognized selector: no direct payload access by this function.
- An unsupported record identifier: no direct payload access by this function.
- An absent context: no direct payload access by this function.

Passing a null payload is therefore not uniformly valid or invalid. Its safety depends on the selected path and, for the first three selectors, the behavior of F-fa3878f0b526.

### Context and record layout

F-7fba0909c452 supplies a context address or a null address. The visible function uses the following context fields:

| Context-relative offset | Width | Established use |
|---|---|---|
| `+0` | 32 bits | Tested and conditionally written on the unsupported-identifier path. |
| `+4` | 32 bits | Address-like component used by selectors `0x1203` and `0x1204`. |
| `+8` | 32 bits | Index-like component multiplied by `0x44` for those selectors. |

The selected-record address is formed from:

- the returned context address;
- an externally stored offset whose value is not supplied;
- the original record identifier multiplied by `0xA8`.

The identifier is **not** reduced by `0x4000` for this address calculation. The range check uses that subtraction, but record addressing uses the original identifier.

The `0xA8` multiplier establishes a 168-byte stride in the address calculation. It does not establish the layout, readable extent, writable extent, or ownership of an individual record.

### Output

There is no explicit return value and no returned success or failure status.

Observable results consist of:

- calls to dependencies;
- reads and writes performed directly by this function;
- any additional effects of the dependencies.

Normal return does not distinguish successful processing, absent context, fallback handling, or default-selector handling.

### Preconditions

For defined execution along a chosen path:

- A nonnull context must support the accesses actually made on that path.
- A directly copied payload must provide the required 12 or 16 readable bytes.
- Addresses passed to dependencies must meet those dependencies’ requirements.
- Address arithmetic must produce usable addresses for the target environment.

This function visibly performs no general validation of payload addresses, selected-record addresses, context-derived addresses, alignment, or accessible memory extents.

## Behavior

### 1. Obtain the context

F-7fba0909c452 is called before the identifier or selector affects any branch.

If it returns null, the function returns immediately. In this case there is no direct context-field access, no direct payload access, no selected-record address calculation, and no operation-specific call.

The initial dependency call may itself have side effects. An early return does not establish that the overall invocation was side-effect-free.

### 2. Classify the identifier

With a nonnull context, the function tests whether the identifier belongs to the eight-value range beginning at `0x4000`.

The visible test is an unsigned 32-bit subtraction followed by comparison with eight. Its effect is precisely to accept `0x4000` through `0x4007`. Values below the lower bound do not become valid through signed interpretation.

For an unsupported identifier:

1. Read the first 32-bit context word.
2. If that word is zero, write `0x00000500` to the same location.
3. Return.

This path does not dispatch on the selector and does not directly access the payload. The default-selector dependency is not called merely because the identifier is unsupported.

For a supported identifier, the function forms the selected-record address before selector dispatch. This address computation is therefore also present for an unrecognized selector, although no direct access through that address is visible on the default path.

### 3. Selectors `0x1200`, `0x1201`, and `0x1202`

These selectors share the following preparation:

1. Supply an eight-byte temporary region and the original payload address to F-fa3878f0b526.
2. After that dependency returns, supply the previously calculated selected-record address and the same temporary region to the selector-specific consumer.
3. Return after the consumer returns.

The consumer differs by selector:

| Selector | Consumer |
|---|---|
| `0x1200` | Unresolved consumer; no opaque dependency ID was supplied. |
| `0x1201` | F-2b03d01d2062 |
| `0x1202` | F-067cbfe20f51 |

The visible function does not directly copy a fixed number of bytes from the payload on these paths. It delegates payload interpretation to F-fa3878f0b526.

No initialization of the eight-byte temporary region is visible before that call. The evidence does not establish which bytes the dependency writes, nor which bytes the subsequent consumer reads. In particular, the temporary region’s size does not establish an eight-byte input payload requirement.

The selected-record address is calculated before F-fa3878f0b526 runs. The visible function does not recalculate it after that call.

### 4. Selector `0x1203`

This path uses a 16-byte input snapshot and a separate 16-byte temporary output region.

Its visible sequence is:

1. Copy the payload word at byte offset `0` into the first snapshot word.
2. Copy the payload word at byte offset `4` into the second snapshot word.
3. Copy the payload word at byte offset `8` into the third snapshot word.
4. Copy the payload word at byte offset `12` into the fourth snapshot word.
5. Obtain an auxiliary address from the context words at offsets `+4` and `+8`, using a `0x44` multiplier on the latter.
6. Call F-24e81f016961 with the temporary output region, the auxiliary address, and the snapshot address.
7. After that call returns, call F-3574ebf61794 with the previously calculated selected-record address and the temporary output region.
8. Return after the consumer returns.

The snapshot preserves four 32-bit word values in their original order. No numeric conversion, component interpretation, or byte-order conversion is visible in the copy itself.

The auxiliary address is formed by adding the context’s `+4` word to the context’s `+8` word multiplied by `0x44`, or 68 decimal. The supplied evidence does not establish a record layout for memory at that auxiliary address.

The converter receives the local snapshot address rather than the original payload address. Thus, its explicit input pointer refers to the values copied before the converter call. This does not establish that the converter cannot access the original payload through another alias or global state.

### 5. Selector `0x1204`

This path follows the same broad structure as `0x1203`, but its directly copied input and temporary output regions are each 12 bytes.

The function:

1. Copies payload words at byte offsets `0`, `4`, and `8`, in that order, into a local snapshot.
2. Forms the auxiliary address from the context words at offsets `+4` and `+8`, with the same `0x44` multiplier.
3. Calls F-9390324a38c0 with the temporary output region, auxiliary address, and snapshot address.
4. Calls F-8eedcc51cba5 with the previously calculated selected-record address and the temporary output region.
5. Returns after that consumer returns.

There is no fourth direct payload-word read on this path.

### 6. Unrecognized selectors

For a supported identifier and a selector other than `0x1200` through `0x1204`, the function calls F-80de2f8a602f and then returns if that call returns normally.

The visible function does not directly read the payload, inspect the fallback word, or store a fallback value on this path.

Whether F-80de2f8a602f logs, changes state, signals an error, or does not return normally is unknown.

## Side effects

### Direct memory effects

The only explicit write to nonlocal memory in the supplied function is the conditional write of `0x00000500` to the first context word for an unsupported identifier.

The function also writes its temporary regions while copying the 12-byte or 16-byte payload snapshots. Further temporary-region writes may occur inside dependencies.

There is no explicit write through the caller’s payload address. This is not a guarantee that dependencies leave caller memory unchanged.

### Ordering and aliasing

The established call and direct-copy ordering is:

- Context acquisition precedes all path-specific work.
- Selected-record address formation precedes any selector-specific helper call.
- For `0x1203` and `0x1204`, the direct payload copies precede the converter call.
- The auxiliary-address expression occurs after those copies and before the converter call.
- Each converter returns before its corresponding consumer is called.

The supplied expression does not establish the relative evaluation order of the two context-field reads used to form the auxiliary address.

The contract does not require the context and payload to be disjoint. Nor does it assume that dependencies preserve context fields, external state, or caller memory. If any dependency changes such state, the effect must be analyzed against the reads that have already occurred and those that remain.

The direct-read sequence described here follows the supplied static representation; it is not a claim about atomicity or physical bus transaction order.

### Ownership and lifetime

Temporary regions are local to this invocation. No explicit allocation, deallocation, ownership transfer, or payload release is visible.

Passing a temporary address to a dependency does not establish that the dependency may retain it. Conversely, this function alone does not prove that dependencies refrain from retaining addresses. Any later use of such an address would require a lifetime guarantee not supplied here.

### Failure behavior

There are no visible checks of converter or consumer results and no local recovery after a dependency call fails to return normally.

Invalid memory, arithmetic-related address failures, and dependency-specific error behavior have no established recovery contract in the supplied evidence.

## Dependencies

| Opaque ID | Established role |
|---|---|
| F-7fba0909c452 | Called once at entry to obtain the context address. |
| F-fa3878f0b526 | Receives an eight-byte temporary region and the original payload address for selectors `0x1200`–`0x1202`. |
| F-2b03d01d2062 | Consumes the selected-record address and eight-byte temporary region for `0x1201`. |
| F-067cbfe20f51 | Consumes the selected-record address and eight-byte temporary region for `0x1202`. |
| F-24e81f016961 | Receives a 16-byte temporary output region, auxiliary address, and four-word snapshot for `0x1203`. |
| F-3574ebf61794 | Consumes the selected-record address and temporary output region for `0x1203`. |
| F-9390324a38c0 | Receives a 12-byte temporary output region, auxiliary address, and three-word snapshot for `0x1204`. |
| F-8eedcc51cba5 | Consumes the selected-record address and temporary output region for `0x1204`. |
| F-80de2f8a602f | Called for an unrecognized selector when the identifier is supported. |

One additional consumer is visible on the `0x1200` path, but its opaque dependency ID is unavailable. Its established interface is limited to the selected-record address and the eight-byte temporary region.

These roles do not establish the dependencies’ full contracts, memory ownership policies, global-state effects, or return guarantees.

## Acceptance examples

These are conditional static acceptance criteria, not executed test results.

1. **No context.**  
   If F-7fba0909c452 returns null, execution returns without dispatch or direct payload access, regardless of identifier and selector.

2. **Lower identifier boundary.**  
   With a valid context, identifier `0x4000` takes the dispatch path. Its selected-record address contribution is `0x4000 × 0xA8 = 0x002A0000`, before adding the context address and external offset.

3. **Upper identifier boundary.**  
   Identifier `0x4007` also takes the dispatch path. Its identifier-derived contribution is `0x002A0498`.

4. **Identifiers just outside the range.**  
   Identifiers `0x3FFF` and `0x4008` take the fallback path. If the first context word is zero, it becomes `0x00000500`; otherwise it remains unchanged.

5. **Unsupported identifier with a recognized selector.**  
   Identifier `0x4008` and selector `0x1203` still take only the fallback path. No four-word payload snapshot is made.

6. **Supported identifier with an unrecognized selector.**  
   Identifier `0x4000` and selector `0x1205` lead to F-80de2f8a602f, not to the fallback-word update.

7. **Four-word input preservation.**  
   For selector `0x1203`, if the payload contains four readable 32-bit values, the snapshot passed to F-24e81f016961 contains those values in the same order. No interpretation as signed numbers or floating-point values is established.

8. **Three-word input boundary.**  
   For selector `0x1204`, this function directly reads only the first three payload words. Any additional memory requirement would need evidence from dependencies.

9. **Context-derived auxiliary address.**  
   If the context’s `+4` word represents address `B` and its `+8` word is `3`, with no arithmetic overflow, the auxiliary address passed on either conversion path is `B + 204`.

10. **Converter-to-consumer order.**  
    For selector `0x1201`, F-fa3878f0b526 must return before F-2b03d01d2062 is called. Both receive the same eight-byte temporary region in their respective argument positions.

## Uncertainties

The following remain **documented_only**:

- The meaning, value, and mutability of the external record-address offset.
- The opaque ID and full behavior of the `0x1200` consumer.
- Dependency input and output layouts beyond the visible argument passing and local region sizes.
- Whether conversion dependencies initialize every byte later consumed.
- Ownership, address retention, allocation, release, and global-state effects inside dependencies.
- Alignment requirements, byte order, and support for concurrent access.
- The semantics of the fallback value `0x00000500`.
- Whether the default-selector dependency returns normally or signals an error.
- Overflow behavior for the recovered address expressions.

The identifier range test explicitly uses unsigned 32-bit arithmetic. The remaining address calculations appear in a 32-bit address-and-word setting, but the supplied representation is insufficient to establish a portable signed-overflow contract. Acceptance examples involving addresses therefore assume arithmetic remains representable and the resulting addresses are valid.

The arrangement of converters and consumers suggests a prepare-then-apply operation. That is inferred intent only; it does not establish the domain meaning of any payload or record.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes the initial context lookup, unsigned eight-value identifier check, two address formulas, selector dispatch, temporary-region sizes, direct payload-word copies, dependency-call ordering, conditional fallback write, and absence of an explicit return value.

The supplied material also notes removal of an unreachable block. No behavior for that omitted block is available or incorporated.

No implementation access, dependency inspection, runtime execution, smoke test, or acceptance test was performed. The examples specify expected behavior conditional on the stated memory and dependency assumptions; they are not observations of a running system.
