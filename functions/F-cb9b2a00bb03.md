<!-- Trusted provenance: F-cb9b2a00bb03; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cb9b2a00bb03

## Purpose

F-cb9b2a00bb03 coordinates operations on a mutable object using a mutable position argument and a boundary argument. It initializes caller-provided output storage, conditionally performs a reset, and otherwise repeats an advance-like call followed by a per-item call until a comparison-like dependency returns exactly one.

The established contract concerns 32-bit memory accesses, dependency calls, branch conditions, and a packed 64-bit return. Interpreting the object as a container, the positions as iterators, or the repeated operation as removal is inferred intent, not established behavior.

Dependencies receive addresses of invocation-local words. Once exposed, those words cannot be assumed unchanged across subsequent dependency calls: a dependency may retain an address and another call may use it during the same invocation. The contract therefore distinguishes explicit assignments from values observed at later reads.

## Interface

The function takes four arguments:

| Argument | Established representation and use |
|---|---|
| Output destination | Address of a writable 32-bit word. Written before the first dependency call and potentially written again. |
| Object address | A 32-bit address-valued argument. Direct accesses use byte offsets +0x10 and +0x14. |
| Mutable position address | Address whose first 32-bit word is read during each loop iteration. Also passed directly to dependencies. |
| Boundary argument | A 32-bit value passed unchanged to the comparison-like dependency. Whether it represents an address, handle, or other value is undocumented. |

The return is a **64-bit concatenation of two 32-bit bit patterns**:

- The first component occupies bits 63–32.
- The second component occupies bits 31–0.

This is not established as a conventional success/failure result. Concatenation preserves the component bit patterns; it does not sign-extend either component into the other half.

On the reset path, the upper component is the record address captured before the final output write, and the lower component is the original output-destination address.

On the loop path, the components are the values of two invocation-local scratch words when read for return construction. Their initialization, explicit replacement, and exposure to dependencies are specified below.

### Directly accessed layout

“Object,” “record,” and “position” are descriptive labels, not recovered semantic types.

| Storage | Byte offset | Width | Direct use |
|---|---:|---:|---|
| Object | +0x10 | 32 bits | Address of a referenced record |
| Object | +0x14 | 32 bits | Tested for nonzero during reset selection; cleared on reset |
| Referenced record | +0x04 | 32 bits | Supplied to the reset dependency; subsequently cleared |
| Referenced record | +0x08 | 32 bits | Used to initialize a comparison temporary; assigned a record address on reset |
| Referenced record | +0x0C | 32 bits | Assigned a record address on reset |
| Mutable position | +0x00 | 32 bits | Captured before each advance-like call |
| Output destination | +0x00 | 32 bits | Receives output writes |

All offsets are byte offsets. The supplied representation uses 32-bit address-valued words. No explicit bounds, null, alignment, or address-overflow checks are visible. The evidence does not establish portable behavior for invalid address arithmetic.

### Preconditions

Every memory access actually reached must refer to storage valid for that access. Dependencies may impose additional requirements not established here.

No non-aliasing requirement is visible. Overlap between caller-provided storage and object or record fields can change subsequent reads. Such cases must follow the access order below rather than an assumed entry-time snapshot.

Invocation-local addresses supplied to dependencies designate live storage during this invocation. Their exposure establishes neither permission for use after return nor a prohibition against retention and use by later calls before return.

## Behavior

### Initial output and temporary state

The function first reads the object’s +0x10 word and writes that value to the output destination.

It then reads the object’s +0x10 word again and reads the referenced record’s +0x08 word. That value initializes a 32-bit comparison temporary.

The second object-field read is significant. For example, an initial output write overlapping object storage can affect the address used to obtain the comparison temporary. The function does not reuse the first captured record address for this purpose.

Before calling a dependency, it also initializes:

- An **output-related scratch word** with the original output-destination address.
- An **object-related scratch word** with the original object address.
- A separate **record-check temporary** with the mutable-position address.

These are distinct words. In particular, the record-check temporary is not either scratch word used to construct the normal return.

### First reset check

The function calls **F-4ff56f8ab90f** with:

1. The original mutable-position address.
2. The address of the comparison temporary initialized from record +0x08.

A zero result bypasses all remaining reset checks and proceeds to the loop condition. Any nonzero result permits the second reset check.

The comparison temporary is passed by address. Its value after the call is not constrained by this function and is not subsequently used in a visible direct computation.

### Second reset check

If the first result was nonzero, the function reads the object’s current +0x10 word and assigns it to the record-check temporary, replacing that temporary’s initial mutable-position address.

It calls **F-4ff56f8ab90f** with:

1. The original boundary argument.
2. The address of the record-check temporary.

If this call returns zero, the function proceeds to the loop without reading object +0x14 for reset selection.

If it returns nonzero, the function reads object +0x14. A nonzero word selects the reset path; zero selects the loop.

Thus, reset requires all three conditions:

- First comparison result is nonzero.
- Second comparison result is nonzero.
- The subsequently read object +0x14 word is nonzero.

Neither comparison must return exactly one for these checks. The object word also has no positivity requirement: any nonzero 32-bit pattern satisfies the visible condition.

Object reads separated by dependency calls are not interchangeable. Either call may change storage that later reads consult.

### Reset path

The function reads the object’s current +0x10 word, reads +0x04 from that referenced record, and calls **F-6d526a7d872e** with:

1. The original object address.
2. The 32-bit value obtained from record +0x04.

After that call returns normally, the direct reset writes occur in this order:

1. Assign a current object +0x10 value to the currently referenced record’s +0x08 field.
2. Assign zero to the currently referenced record’s +0x04 field.
3. Assign a current object +0x10 value to the currently referenced record’s +0x0C field.
4. Assign zero to object +0x14.

The reset expressions consult object +0x10 after the dependency returns; they do not retain the record address used to obtain the dependency’s second argument. A dependency-induced record replacement therefore affects the reset targets.

These writes also must not be treated as an atomic update to one immutable record address. If a reset write aliases object +0x10, a later reset expression can consult the changed value. The evidence establishes the sequence of assignments, but not an additional ordering between separate address reads within an individual assignment.

The function next reads object +0x10, captures its value, and writes that captured word to the output destination. It returns:

- Upper 32 bits: the captured record-address word.
- Lower 32 bits: the original output-destination address.

The upper component is not refreshed after the final output write. If that write changes object storage through aliasing, the captured return component remains unchanged.

No loop condition, advance-like call, or per-item call occurs on this path.

### Loop condition

If reset is not selected, the function calls **F-4ff56f8ab90f** with the original mutable-position address and original boundary argument.

Its result is interpreted differently from the reset checks:

- **Exactly one:** finish the loop.
- **Any other value:** execute the body.

Zero, two, and other non-one patterns therefore all enter the body, unless an independent dependency contract excludes them.

The comparison is repeated after every completed body execution. There is no independent iteration bound.

### Loop body and scratch-word exposure

Each body execution has this order:

1. Read and capture the first 32-bit word at the mutable-position address.
2. Call **F-af78fbc288e3** with the original mutable-position address.
3. Assign the captured pre-call word to the output-related scratch word.
4. Call **F-36e5260b6de6** with:
   - The address of the object-related scratch word.
   - The original object address.
   - The address of the output-related scratch word.
5. Read the object-related scratch word and write that value to the original output destination.
6. Evaluate the next loop condition.

The snapshot assignment occurs **after** the advance-like call. Therefore, if that call changes the mutable position, the per-item call still receives the earlier captured word through its third argument.

After a previous per-item call has exposed the scratch addresses, an advance-like call could also modify the output-related scratch word through a retained address. The explicit assignment in step 3 overwrites such a modification with the current iteration’s pre-advance snapshot before the next per-item call.

The object-related scratch word has no corresponding per-iteration reinitialization. It starts as the original object address, but the value present when a later per-item call begins is not necessarily what the preceding per-item call left. An intervening comparison or advance-like call may modify it through an address retained from an earlier per-item call.

Both scratch words remain live until return. The function does not revoke access to their addresses after a per-item call. Whether dependencies actually retain or share those addresses is unknown, but their preservation across later calls is not guaranteed.

The output write following a per-item call uses the object-related scratch word as read at that point. There is no intervening dependency call between that call’s return and this direct read and write.

### Normal return

When a loop comparison returns exactly one, the function constructs the return from:

- Upper 32 bits: the object-related scratch word as read for return construction.
- Lower 32 bits: the output-related scratch word as read for return construction.

A terminating comparison may modify either word through an address retained from a prior per-item call. The returned pair therefore need not equal the pair left by the last per-item call.

There is no direct output write after the terminating comparison. Consequently, that comparison can change the return components without the function subsequently copying the changed upper component to the output destination.

If the body executes zero times, neither scratch address has been supplied to the per-item dependency during this invocation. Under the ordinary execution model in which dependencies access invocation-local storage only through available references, the components retain their initial values: original object address above original output-destination address. This does not imply that comparison calls preserve caller-visible storage or globals.

## Side effects

### Direct writes

The function directly writes:

- The output destination once at entry.
- The output destination after every normally returning per-item call.
- The output destination once more on the reset path.
- Record fields +0x08, +0x04, and +0x0C, in that order, on reset.
- Object +0x14 after those record writes on reset.

It also initializes and updates invocation-local words. Two of those words become externally reachable through the per-item dependency’s arguments.

There is no visible direct write through the mutable-position argument. Changes to that position are delegated.

### Dependency effects, lifetime, and ownership

No allocation or release operation is directly visible in this function. This does not establish allocation-free execution or preservation of any object’s lifetime.

Dependencies may affect caller-visible memory, globals, object references, and reachable invocation-local words. No global-preservation guarantee is established.

In particular, retaining scratch addresses for use by later dependency calls during the same invocation is not excluded by the evidence. Assuming that only the per-item dependency can mutate those words would overstate the contract.

Ownership transfer, record destruction, allocation, pointer retention rules, and post-return use of temporary addresses remain **documented_only**. Subsequent accesses require that any dependency-induced state changes leave the reached storage valid.

### Errors and termination

There is no visible validation failure, error-specific return, rollback, or recovery branch.

The loop can run indefinitely if comparisons never return exactly one and dependencies continue returning normally. No progress guarantee follows from the advance-like call’s presence.

Behavior following invalid accesses, abnormal dependency termination, or unsynchronized concurrent mutation is not established.

## Dependencies

| Opaque ID | Established use | Unresolved behavior |
|---|---|---|
| **F-4ff56f8ab90f** | Performs up to two reset checks and every loop condition. Reset checks accept nonzero; loop termination requires exactly one. | Comparison meaning, argument types, result range, mutations, retained-address access, and failure behavior. |
| **F-af78fbc288e3** | Called once per body, after the position snapshot and before its assignment to the output-related scratch word. | Whether it advances a position; traversal, mutation, lifetime, and retained-address effects. |
| **F-6d526a7d872e** | Called on reset with the object and a word read from its then-referenced record +0x04. | Whether it clears, traverses, destroys, releases, replaces records, or changes other state. |
| **F-36e5260b6de6** | Receives writable addresses of both return-related scratch words and the original object address once per body. | Item-operation meaning, output interpretation, ownership effects, address retention, and failures. |

Every subsequent action described above is conditional on the preceding dependency returning normally.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless an example specifies otherwise, addresses are valid, relevant storage does not overlap, dependencies make no additional mutations, and there is no concurrent access.

1. **Immediate termination.**  
   The first reset check returns zero, and the first loop comparison returns one. The only direct output write is the entry record-address write. No advance-like or per-item call occurs. The return contains the original object address above the original output-destination address.

2. **Reset without record replacement.**  
   Both reset checks return nonzero, object +0x14 is nonzero, and the reset dependency leaves object +0x10 unchanged. Record +0x08 and +0x0C become the record address; +0x04 becomes zero; object +0x14 becomes zero. The final output and upper return component contain that record address.

3. **Reset with record replacement.**  
   The reset dependency replaces object +0x10 with another valid record address. With non-overlapping fields, subsequent reset writes target the replacement record. The final capture uses object +0x10 after those writes, not the address used to obtain the dependency’s second argument.

4. **Failed reset-state check.**  
   Both reset comparisons return one, but object +0x14 is zero. The reset dependency is not called. A separate comparison invocation evaluates the loop condition.

5. **Noncanonical comparison result.**  
   A reset comparison returning two satisfies its nonzero check. A loop comparison returning two enters the body rather than terminating.

6. **Pre-advance snapshot.**  
   The position word initially contains value A. The advance-like call changes it to B. The subsequent explicit scratch assignment supplies A to the per-item dependency, not B.

7. **Terminating comparison changes return words.**  
   A per-item call retains both scratch addresses, leaves upper value U and lower value L, and returns. The direct output write stores U. The next comparison uses the retained addresses to set the words to U′ and L′, then returns one. The packed return contains U′ above L′. There is no subsequent direct output write of U′.

8. **Intervening changes before another iteration.**  
   After scratch addresses have been exposed, a comparison changes the object-related scratch word and returns a non-one result. The advance-like call changes it again and also changes the output-related scratch word through retained addresses. Before the per-item call, the object-related word retains its then-current value, while the explicit post-advance assignment replaces the output-related word with that iteration’s pre-advance position snapshot.

## Uncertainties

The following remain **documented_only**:

- Semantic types of the object, record, position, boundary, and packed result.
- Whether object +0x14 is a count, flag, or other state.
- Whether the record is a sentinel and reset denotes an empty container.
- Whether the body removes items.
- Dependency result ranges, side effects, and abnormal termination.
- Whether exposed addresses are retained or shared between dependencies.
- Ownership, allocation, release, synchronization, and supported aliasing rules.
- Architecture-specific validity, alignment, and address-overflow behavior.

The layout and advance-before-operation order support a container-range interpretation, but do not establish it.

## Evidence

The established behavior derives solely from the supplied static function body and opaque dependency mapping. It supports the stated access widths, field offsets, call order, branch predicates, explicit scratch assignments, output writes, and packed return construction.

The source exposes both return-related scratch addresses to the per-item dependency. It contains no mechanism establishing that subsequent calls cannot use retained addresses. Accordingly, later reads are specified by their actual timing, not by an assumed persistence of the last per-item results.

Dependency implementations and broader data-structure invariants are unavailable. No runtime tests have been performed. All acceptance examples are conditional static consequences, not observed executions.
