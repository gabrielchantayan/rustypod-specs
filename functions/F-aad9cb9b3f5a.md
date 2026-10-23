<!-- Trusted provenance: F-aad9cb9b3f5a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aad9cb9b3f5a

## Purpose

F-aad9cb9b3f5a performs a stateful memory operation using three evaluations of a global root. It retains the first root value as a final-writeback address, obtains a selector through the second root value, and obtains an indirect-storage address through the third root value. It then clears a two-byte location, reads a two-byte value, calls F-d7be9cfb4000, and writes a secondary 32-bit call output through the initially retained address.

The three root evaluations can identify the same object if the root remains stable. Root stability is not established by the supplied representation and is not a precondition imposed by this contract. The three address roles must therefore remain distinct.

The established behavior concerns access widths, address relationships, arithmetic, and operation order. No domain meaning is established for the selector, indirect storage, cleared location, or final value. Interpretations such as advancing a cursor or consuming a record are inferred intent, not contractual behavior.

This contract rests on static evidence only. No runtime tests have been performed.

## Interface

### Inputs and outputs

The function has no explicit parameters and no explicit return value.

Its effective inputs are:

- The global root’s values at three separate evaluations.
- A 32-bit selector read through the second root value.
- A 32-bit address-bearing integer read through the third root value.
- A 16-bit memory value read after the clear.
- The behavior and secondary 32-bit output of F-d7be9cfb4000.

The directly established outputs are:

- A two-byte zero store to a computed indirect address.
- One invocation of F-d7be9cfb4000.
- A four-byte store of the secondary call output through the first root value.

There is no explicit status result or error-reporting interface.

### Address roles and layout

The following labels describe roles, not source identifiers:

| Role | Origin or location | Width or use |
|---|---|---|
| Retained writeback base | First evaluation of the global root | Captured address-bearing 32-bit integer |
| Selector-source base | Second evaluation of the global root | Used to address the selector field |
| Storage-source base | Third evaluation of the global root | Used to address the storage-address field |
| Captured selector | Selector-source base +20 bytes | 32-bit integer |
| Captured storage address | Storage-source base +12 bytes | 32-bit address-bearing integer |
| Clear destination | Captured storage address + four times the captured selector +10 bytes | Two-byte write |
| Metadata location | Captured storage address +2 bytes | Two-byte read |
| Final writeback field | Retained writeback base +20 bytes | Four-byte write |

The selector-source field and final writeback field occupy the same offset, but they are not necessarily the same location. They coincide only when their respective bases identify the same address.

Likewise, the storage-address field need not belong to either the writeback object or selector-source object. The function does not compare the three root values.

The multiplier of four establishes address spacing. It does not establish an array type, a complete record layout, allocation bounds, or a valid selector range.

### Preconditions for ordinary completion

An ordinary completed execution requires:

- Each root evaluation and subsequent field read to be valid.
- The computed clear destination to permit a two-byte write.
- The metadata location to permit a two-byte read after that write.
- All relevant integer operations to be valid under the represented arithmetic model.
- F-d7be9cfb4000 to return normally.
- The retained writeback destination to remain valid for a four-byte store after the dependency returns.

The function visibly checks none of these conditions. It has no explicit handling for null addresses, inaccessible storage, negative selectors, arithmetic overflow, or invalidated objects.

## Behavior

### Three root evaluations

The represented sequence begins with three distinct uses of the global root:

1. Evaluate the root and retain that value as the base for final writeback.
2. Evaluate the root again, add 20 bytes, and read the 32-bit selector.
3. Evaluate the root a third time, add 12 bytes, and read the 32-bit storage-address integer.

The selector read occurs before the third root evaluation and storage-address read.

If the root changes between evaluations, the retained writeback base, selector-source base, and storage-source base can differ. The selector and storage address are nevertheless captured before the indirect clear and dependency call.

After the dependency returns, final writeback uses the first retained root value. There is no visible post-call root evaluation to select a new writeback destination.

These statements describe the represented access sequence. They do not establish volatile semantics, inter-thread synchronization, or a required number of physical memory-bus transactions.

### Clear-address arithmetic

The clear destination is formed from the captured storage-address integer, four times the captured selector, and a displacement of 10 bytes.

The multiplication and additions are represented as 32-bit integer arithmetic, not unbounded mathematical address arithmetic. The relevant intermediate results are:

- The captured selector multiplied by four.
- The captured storage-address integer plus that product.
- That sum plus 10.

Defined evaluation requires every relevant intermediate operation to be valid under the represented arithmetic model. A mathematically acceptable final address does not establish that an overflowing intermediate operation is acceptable.

Under a signed 32-bit interpretation, multiplication by four alone fits only for selectors from −536,870,912 through 536,870,911, inclusive. This is an arithmetic boundary, not an established supported-input range. Values inside that interval can still overflow during the additions, identify invalid memory, or violate unknown dependency requirements.

The supplied representation does not establish whether actual execution wraps, traps, or exhibits another consequence for overflow. The contract therefore promises neither modular addressing nor rejection at these boundaries.

Other represented address additions, including field offsets and the metadata displacement, also require a valid arithmetic and addressing interpretation.

### Two-byte clear, then metadata read

After capturing the selector and storage address, the function writes zero to the two-byte clear destination.

This is a two-byte store, not a four-byte record clear.

It then reads two bytes at the captured storage address plus two. The dependency’s second argument originates from this read.

The order is significant: the metadata read observes memory after the clear. If the clear destination coincides with the metadata location, the read obtains the zero just written, assuming no intervening modification.

There is no alias check. The clear may also overlap object fields or other storage, provided those accesses are valid. Changes to previously read selector or storage-address fields do not change the captured values used by this invocation. A change to the global root after the third evaluation likewise does not replace the captured storage address.

The metadata has a 16-bit source width. Its signed interpretation and extension into the dependency’s argument representation are not established by the supplied evidence. Zero remains zero regardless of that unresolved extension behavior.

### Dependency invocation

The function next calls F-d7be9cfb4000 once on the ordinary path.

The supplied arguments are:

- The captured selector incremented by one.
- A value originating from the post-clear 16-bit metadata read.

The increment is represented as 32-bit integer arithmetic. Under a signed 32-bit interpretation, incrementing the maximum signed value exceeds the representable range. There is no visible checked, saturating, or explicitly wrapping increment.

The clear and metadata read precede the call. The evidence does not establish an additional ordering guarantee between the two call-argument evaluations, nor is one needed to establish that the metadata is read after the clear.

The dependency’s internal behavior is not visible.

### Final writeback and return

After normal dependency return, the function writes a secondary 32-bit call output to the retained writeback base plus 20 bytes.

The representation exposes this secondary output separately from an ordinary explicit return expression. It establishes the dataflow into the final store, but not the output’s semantic meaning or the dependency’s complete calling convention.

The stored word is not established to equal:

- The captured selector.
- The selector plus one.
- The metadata.
- A status code.
- A pointer or count.

If the dependency modifies the final writeback field and returns normally while leaving that destination valid, this function subsequently overwrites that field with the secondary output.

The function then returns without an explicit value.

### Failure boundaries

No branch, retry, rollback, explicit error return, or cleanup operation is visible.

Effects are not transactional. If execution fails after the clear, the evidence provides no rollback. If the dependency does not return normally, final writeback is not established. If the final destination becomes invalid during the dependency call, the earlier clear and dependency effects may already have occurred.

Exact failure behavior for invalid memory or invalid arithmetic is documented_only.

## Side effects

The represented ordinary-path sequence is:

1. Evaluate the global root and retain the writeback base.
2. Evaluate the global root again.
3. Read the selector at the second root value plus 20 bytes.
4. Evaluate the global root a third time.
5. Read the storage-address integer at the third root value plus 12 bytes.
6. Write a two-byte zero at the computed clear destination.
7. Read two bytes at the captured storage address plus two.
8. Invoke F-d7be9cfb4000.
9. Write the secondary 32-bit output at the retained first root value plus 20 bytes.

This sequence does not promise atomicity, memory barriers, or synchronization.

No allocation, release, ownership transfer, or reference-count operation is directly visible in F-aad9cb9b3f5a. That absence does not establish anything about resource operations inside F-d7be9cfb4000.

The dependency is not assumed to preserve the global root, any of the objects involved, indirect storage, or allocation lifetimes. The caller still attempts final writeback through the retained address after normal dependency return. Safe completion therefore depends on that address remaining writable.

## Dependencies

### F-d7be9cfb4000

Established caller-side behavior:

- Called once after the two-byte clear and metadata read.
- Receives the captured selector plus one.
- Receives a value originating from a two-byte memory read.
- Has a secondary 32-bit call output consumed by final writeback.
- Must return normally for the caller to reach that writeback.

The following dependency properties are documented_only:

- Argument meanings and accepted ranges.
- Argument-extension rules for the 16-bit source value.
- Whether the secondary output is an intentional result of the dependency’s interface.
- The meaning and validity of that output.
- Error and non-return behavior.
- Effects on globals and memory.
- Ownership, retention, and release behavior.
- Reentrancy and lifetime guarantees.

No preservation or resource-management obligation is inferred merely from the caller’s subsequent store.

## Acceptance examples

These examples are conditional static consequences, not executed tests. Unless stated otherwise, accesses are valid, all relevant intermediate arithmetic is defined, no external modification intervenes, and the dependency returns normally.

### Stable root and selector three

If all three root evaluations identify the same object and the captured selector is 3:

- The clear destination is the captured storage address plus 22 bytes.
- Two bytes there are zeroed before metadata is read.
- The first dependency argument is 4.
- The second argument originates from the halfword at storage address plus two.
- Final writeback replaces the field from which the selector was read with the secondary output, not necessarily 4.

### Three different root values

If the first, second, and third root evaluations identify three different objects:

- Final writeback targets the first object at offset +20.
- The selector comes from the second object at offset +20.
- The storage address comes from the third object at offset +12.
- No visible equality check or reconciliation occurs.

This example does not assert that changing roots are supported by a concurrency protocol.

### Zero selector

With a captured selector of zero:

- The clear destination is storage address plus 10.
- The first dependency argument is 1.
- No zero-selector rejection or alternate path is visible.

### Clear destination equals metadata location

With a captured selector of −2, valid intermediate arithmetic, and valid addresses:

- Multiplication produces −8.
- The clear destination resolves to storage address plus two.
- The clear zeroes the metadata location before it is read.
- The metadata argument is zero.
- The first dependency argument is −1.

This establishes an aliasing consequence, not intended support for negative selectors.

### Dependency changes the global root

If the dependency replaces the global root while the retained writeback destination remains valid:

- Final writeback still uses the first root value retained before the call.
- The replacement root is not evaluated for that store.

### Dependency changes the destination field

If the dependency writes a value to the retained writeback base plus 20 and returns normally:

- This function subsequently overwrites that value with the secondary call output.
- The already captured selector and storage address remain the caller’s inputs.

### Clear-address intermediate overflow

With a selector of 536,870,912:

- Incrementing the selector by one would fit in signed 32-bit arithmetic.
- Multiplying it by four would not.
- The clear-address computation therefore crosses a represented intermediate boundary before the dependency call.
- Acceptance must not require a particular clear address, dependency invocation, wraparound result, or error response.

### Dependency does not return

If execution reaches the dependency and it does not return normally:

- The clear and metadata read have already occurred.
- Final writeback is not established.
- No rollback is visible.

## Uncertainties

The following remain documented_only:

- The global root’s semantic role and whether it remains stable between evaluations.
- The object types, allocation extents, and external selector constraints.
- Intended record layout or interpretation of indirect storage.
- Target byte order and alignment requirements.
- Actual machine-level overflow behavior.
- Behavior for invalid addresses and unsupported arithmetic.
- Metadata signedness and argument extension.
- The dependency’s full interface and secondary-output convention.
- Concurrency, interrupt, reentrancy, and memory-lifetime guarantees.
- Access qualifiers or low-level execution details absent from the representation.

The represented integer operations are 32-bit, the clear and metadata access are 16-bit, and final writeback is 32-bit. Those established widths do not resolve the unknown execution rules above.

## Evidence

The supplied static representation supports:

- Three global-root evaluations in source order, with distinct writeback, selector-source, and storage-source roles.
- A selector read at the second root value plus 20 bytes.
- A storage-address read at the third root value plus 12 bytes.
- A two-byte zero store using a 32-bit selector multiplication and successive 32-bit additions.
- A subsequent two-byte metadata read at storage address plus two.
- One call to F-d7be9cfb4000 with the incremented selector and metadata-derived value.
- A four-byte post-call store through the first retained root value, without a post-call root evaluation.
- No visible conditional validation, explicit error handling, or rollback.

Domain semantics are not established. Dependency behavior and unsupported-input outcomes remain unresolved.

No runtime execution, memory trace, dependency inspection, or acceptance test has been performed.
