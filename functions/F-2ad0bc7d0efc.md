<!-- Trusted provenance: F-2ad0bc7d0efc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2ad0bc7d0efc

## Purpose

F-2ad0bc7d0efc selects between a bulk operation followed by explicit structural reset and an iterative operation driven by caller-supplied traversal state.

The field layout, preliminary checks, and reset pattern suggest range removal with a whole-container shortcut. That purpose is **inferred intent**, not an established semantic contract. The supplied body establishes memory accesses, dependency calls, branch conditions, and return construction. It does not establish that any dependency removes nodes, destroys values, or releases memory.

The function always writes to the caller’s output location before its first dependency call. Its 64-bit return combines two 32-bit values; it is not an established success or failure indicator. On the iterative path, the return is constructed from local words after the terminating comparison. Those words need not still contain the values present at the last output write.

## Interface

### Inputs

| Input | Established role |
|---|---|
| Output location | Address receiving 32-bit writes at entry and subsequently according to the selected path. |
| Context | Original 32-bit address-valued argument used to access fields at byte offsets 0x10 and 0x14 and passed to processing dependencies. |
| Traversal state | Address of mutable state. Its first 32-bit word is read during each iteration. The address is also passed to comparison and advancement dependencies. |
| Boundary value | Opaque 32-bit argument passed unchanged to the second preliminary comparison and each loop comparison. Its representation and semantic type are unknown. |

The supplied representation uses 32-bit words for the relevant addresses and values, and a 64-bit concatenated result. Original public types, pointer qualifications, and calling convention are not established.

### Accessible layout

The context has these directly accessed fields:

| Byte offset | Width | Use |
|---|---:|---|
| 0x10 | 32 bits | Address of a referenced record, reloaded at multiple points. |
| 0x14 | 32 bits | Conditionally tested for nonzero; explicitly cleared on the bulk path. |

The referenced record has these directly accessed fields:

| Byte offset | Width | Use |
|---|---:|---|
| 0x04 | 32 bits | Passed by value to F-1a19ee74eb68; subsequently cleared on the bulk path. |
| 0x08 | 32 bits | Initially copied into a local comparison operand; assigned a record address during bulk reset. |
| 0x0c | 32 bits | Assigned a record address during bulk reset. |

The first record word and any other fields have no established role here. “Header,” “sentinel,” and “count” are possible interpretations, not required semantic types.

### Output and return

The output location receives:

- At entry: the record address read from context offset 0x10.
- On the bulk path: another record address read after the reset writes.
- After each completed iterative processing call: the then-current value of a local result word.

The return combines a high 32-bit component and a low 32-bit component:

| Exit | High component | Low component |
|---|---|---|
| Bulk path | Record address captured immediately before the final output write | Original output-location address |
| Iterative path with no iterations | Original context argument | Original output-location address |
| Iterative path with iterations | Local result word after the terminating comparison returns | Local saved word after the terminating comparison returns |

The two words are concatenated without sign extension or arithmetic addition. Their interpretation as addresses, integers, or an aggregate is not established. This logical high/low description does not establish a memory byte order.

**The iterative return high component need not equal the value last written by this function to the output.** F-c4998c4a64da receives addresses of both return-bearing local words. It may retain those addresses, and a later dependency—including the terminating comparison—may modify those still-live words. Such changes affect the return without requiring another output write by this function.

The final contents of output storage also depend on dependency effects and aliasing. A statement about this function’s last write is not an unconditional statement about the memory contents at return.

### Preconditions

Normal execution requires:

- Writable storage for every direct 32-bit output write.
- Valid context and record storage for every access at the time it occurs.
- Readable traversal-state storage for each direct first-word read.
- Satisfaction of all dependency preconditions.
- Valid field-address calculations under the target address representation.

No null, bounds, alignment, or address-overflow checks are visible. Field offsets are byte offsets; no wider address arithmetic or overflow recovery is established.

There are no aliasing checks. Output, context, record, and traversal storage must not be assumed distinct unless an external contract requires that separation.

## Behavior

### Entry and preliminary selection

The entry output write precedes every dependency call. Its value comes from context offset 0x10.

The function then reloads that context field and reads offset 0x08 of the resulting record into a local comparison word. This is a distinct record-address read after the output write. If the output aliases the context’s record-address field or related storage, the second read may observe a value changed by the first write.

Two return-bearing local words are initialized before dependency calls:

- The result word initially holds the original context argument.
- The saved word initially holds the original output-location address.

The first F-9ed8931accde call receives traversal state and the address of the local comparison word. It does not receive the address of the record’s 0x08 field. Writing through that argument therefore changes a local copy, not directly that record field.

A zero first result sends execution to the iterative path. Any nonzero first result permits the second preliminary comparison.

For that second comparison, the function reloads context offset 0x10 into another local operand. F-9ed8931accde receives the original boundary value and the address of that local operand.

The bulk path requires all three conditions:

1. The first preliminary result is nonzero.
2. The second preliminary result is nonzero.
3. Context offset 0x14 is nonzero when subsequently read.

If the second result is zero, the function does not read context offset 0x14 for this decision. A zero first result skips both the second preliminary call and that decision read.

These preliminary conditions distinguish zero from nonzero. They do not establish that the dependency implements equality or returns only Boolean values.

### Bulk operation and reset

After bulk selection, the function reads the current record address from context offset 0x10, reads that record’s 0x04 field, and calls F-1a19ee74eb68 with the original context argument and the field value.

After normal return from that dependency, the function performs these direct writes in order:

1. A record’s 0x08 field receives a record address obtained from context offset 0x10.
2. A record’s 0x04 field receives zero.
3. A record’s 0x0c field receives a record address obtained from context offset 0x10.
4. Context offset 0x14 receives zero.
5. The output location receives a freshly read context record address.

The context record address is reloaded for the reset operations; the pre-call address is not consistently reused. If F-1a19ee74eb68 changes context offset 0x10, the reset can target a different record from the one whose 0x04 field supplied the call argument.

Each self-address assignment uses separate context-field reads for its destination record and its stored value. The supplied representation does not establish the relative evaluation order of those reads within one assignment. The order of the writes listed above is established.

Aliasing can affect the reset itself. For example, an earlier reset write can change storage later used as an address. Likewise, the final output write can overwrite a field that was just cleared. Therefore, “the count-like field remains zero” is not an unconditional postcondition.

The bulk return uses the address captured for the final output write as its high component and the original output-location address as its low component. The loop comparison, advancement dependency, and per-element dependency are not called on this path.

### Iterative operation

Every non-bulk case reaches a comparison between traversal state and the original boundary value through F-9ed8931accde.

The loop terminates only when this call returns **exactly 1**. Zero, 2, negative 32-bit values, and every other result continue iteration. This differs from the preliminary nonzero tests.

Each continuing iteration has the following observable ordering:

- The function reads and retains the first 32-bit word of traversal state.
- F-029238addcdd is called with traversal state.
- The retained pre-advancement word replaces the local saved word.
- F-c4998c4a64da is called with the address of the local result word, the original context argument, and the address of the local saved word.
- The function reads the local result word and writes it to the original output location.
- The next loop comparison is called.

The value presented through the saved-word address initially comes from before advancement. Changes to traversal state by F-029238addcdd do not replace that retained value.

The result word is initialized only once. It is not restored to the original context before each processing call. Its current value can carry between iterations, subject to dependency mutations. The saved word is overwritten with the newly retained traversal word before every F-c4998c4a64da call.

The original context argument remains the processing dependency’s second argument even if the result word changes.

### Termination and retained local addresses

Both local words remain live through the terminating comparison and return construction. Once their addresses have been passed to F-c4998c4a64da, the body provides no guarantee that subsequent dependencies cannot access them through retained addresses.

Consequently:

- A comparison can change the result word after its previous value was written to output.
- A comparison can change the saved word after the last per-element call.
- Those changed values become the packed return if that comparison returns exactly 1.
- The comparison can separately change output storage; preservation of the locals does not establish preservation of the output.

If the loop terminates before any iteration, neither return-bearing local has been exposed through the per-element call. Their initialized values supply the no-iteration return.

There is no explicit iteration limit, progress check, or direct count decrement. Termination depends on a future comparison returning exactly 1.

## Side effects

### Direct effects

The function directly performs:

- The unconditional entry output write.
- Context and record reads described above.
- Bulk reset writes and a bulk output write when selected.
- One traversal first-word read per iteration.
- One output write after each normally returning per-element call.
- Local-word updates, including updates observable to dependencies given their addresses.

No explicit allocation, deallocation, payload destruction, or traversal arithmetic appears in the body.

### Dependency-mediated effects and ownership

Dependency contracts are unknown. The body does not establish preservation of globals, context, traversal storage, output storage, or previously exposed local words.

Passing a record field or an element-like value does not establish ownership transfer. Neither processing dependency is proved to release memory, preserve the record, or avoid retaining pointers.

Local addresses supplied to dependencies remain usable during their lifetime, including subsequent calls before this function returns. No policy preventing retention or indirect mutation is visible. Their later use after this function returns is not established as valid.

### Errors and partial progress

There is no explicit error result, rollback, exception handling, or recovery path.

If an access faults or a dependency does not return normally, earlier writes and dependency effects may already have occurred. Output is not protected from partial progress: it is written before the first dependency call.

Invalid-input behavior, nonlocal exits, concurrent mutation, and reentrant effects beyond the visible access sequence remain documented_only.

## Dependencies

| Opaque ID | Established use | Unknown contract |
|---|---|---|
| F-9ed8931accde | Preliminary comparisons use nonzero results; repeated loop comparison terminates only on exactly 1. Preliminary second operands are addresses of local copies. Loop operands are traversal state and the original boundary value. | Comparison meaning, complete return domain, mutations, retained-address access, and preservation guarantees. |
| F-1a19ee74eb68 | Called once on the bulk path with the original context and the current record’s 0x04 field value. Direct reset follows normal return. | Resource effects, ownership, traversal, and preservation of context or record addresses. |
| F-029238addcdd | Called once per iteration after retaining the traversal first word and before populating the saved word for processing. | Advancement rule, progress, mutations, and failure behavior. |
| F-c4998c4a64da | Receives writable addresses of the result and saved words, with the original context between them. | Processing semantics, guarantees for either word, address retention, resource release, and other effects. |

Return values from the bulk, advancement, and per-element dependencies are not used. Their memory effects can still affect subsequent behavior.

## Acceptance examples

These are conditional static contract examples, not executed tests. They assume valid accesses and normally returning dependencies.

### Immediate termination

Suppose the first preliminary comparison returns zero and the first loop comparison returns exactly 1.

Required observations:

- The entry output write occurs.
- The second preliminary comparison is skipped.
- No bulk, advancement, or per-element call occurs.
- The return high component is the original context argument.
- The return low component is the original output-location address.

If the comparisons preserve output storage, it still contains the entry-time record address.

### Noncanonical comparison results

Suppose the preliminary results are 2 and zero, followed by loop results 2 and 1.

Then:

- The first result permits the second preliminary comparison.
- The second result prevents bulk selection without a decision read of context offset 0x14.
- The first loop result does not terminate.
- Exactly one advancement and one per-element call occur before termination.

An implementation that terminates the loop on every nonzero result would violate this behavior.

### Stable-record bulk reset

Suppose both preliminary results are nonzero, context offset 0x14 is nonzero, the bulk dependency preserves the record address, and the relevant context, record, and output fields do not overlap.

After the bulk call:

- Record offset 0x08 receives the record address.
- Record offset 0x04 receives zero.
- Record offset 0x0c receives the record address.
- Context offset 0x14 receives zero.
- Output receives the record address.
- The return contains that address in its high component and the original output-location address in its low component.

No loop comparison occurs.

### Zero count-like field

Suppose both preliminary results are nonzero but context offset 0x14 is zero.

The bulk call and reset are skipped. Execution reaches the ordinary loop comparison. The zero field does not itself cause return.

### Advancement and preserved locals

Suppose a continuing iteration reads an element-like value, and advancement replaces traversal state’s first word with a successor value.

The per-element dependency initially receives the retained earlier value through its third argument, not the successor.

Suppose that dependency then sets the result word to A and the saved word to B. The function writes A to output. If the next comparison returns exactly 1 **and preserves both local words**, the return high component is A and the low component is B. If that comparison also preserves output storage, output contains A at return.

### Terminating comparison changes the return

Suppose the per-element dependency sets the local words to A and B and retains their addresses. The function writes A to output.

Suppose the next comparison uses those retained addresses to change the local words to C and D, then returns exactly 1.

The required return has high component C and low component D. The function performs no additional output write. If the terminating comparison preserves output storage, that storage still contains A.

This behavior is consistent with the supplied body; equality between the return high component and the last output write is not required.

## Uncertainties

**Inferred intent:** range processing or range removal, with a whole-structure shortcut and a reset resembling an empty header.

**documented_only:**

- Original public types, ABI, byte order, and alignment requirements.
- Semantic meanings of the fields, traversal values, and comparisons.
- Supported aliasing configurations and empty-structure representations.
- Dependency ownership, allocation, destruction, address retention, and global-state behavior.
- Whether traversal makes progress or termination is guaranteed.
- Failure, concurrency, and reentrancy semantics.
- Consumer interpretation of the packed return.

Unknown dependency preservation must not be replaced with an assumption that only the most recent per-element call determines the final local values.

## Evidence

Established static evidence consists of the supplied body’s 32-bit accesses, field offsets, short-circuit decisions, exact loop termination condition, dependency arguments, ordered direct writes, exposed local addresses, and concatenated 64-bit returns.

The return/output distinction follows from the call order: the iterative output write precedes the next comparison, while return construction follows the terminating comparison. No intervening restoration of the two return-bearing local words is visible.

Container and removal terminology is inferred. Actual dependency effects and supported runtime conditions remain unknown.

No runtime tests have been performed.
