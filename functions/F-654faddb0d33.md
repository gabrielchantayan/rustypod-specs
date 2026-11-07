<!-- Trusted provenance: F-654faddb0d33; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-654faddb0d33

## Purpose

F-654faddb0d33 performs a boundary-controlled operation on mutable state. It has two paths:

- A bulk path calls one dependency, resets selected structure fields, publishes a distinguished-node address, and returns.
- A general path repeatedly captures a cursor value, calls two operation dependencies, and publishes a working value until a comparison returns exactly one.

**Inferred intent:** The access pattern is consistent with removing a range from a node-based container, with a special case for the entire container. The supplied body does not establish the container type, the meaning of its fields, or any memory-reclamation policy.

The function returns a composed 64-bit value. It is not established as a status code. In the general path, its components are two working words read at return, potentially after changes made through addresses retained by dependencies.

## Interface

### Inputs

| Input | Observed representation | Use |
|---|---|---|
| Output address | Address of a writable 32-bit word | Receives an initial node address and later path-dependent publications |
| Context address | 32-bit address-bearing value | Identifies structure fields and is passed to operation dependencies |
| Cursor address | Address of a readable 32-bit word | Passed to dependencies and directly read during each iteration |
| Boundary value | 32-bit value | Passed to the second qualification call and every general-path comparison |

Address-bearing values occupy 32 bits in the supplied representation. Compatibility with a wider address model is not established.

### Directly accessed layout

| Base | Byte offset | Width | Established use |
|---|---:|---:|---|
| Context | +0x10 | 32 bits | Holds an address used to reach a distinguished node |
| Context | +0x14 | 32 bits | Tested for nonzero during bulk qualification; cleared during bulk reset |
| Distinguished node | +0x04 | 32 bits | Supplies the bulk dependency’s second argument; subsequently cleared |
| Distinguished node | +0x08 | 32 bits | Supplies the first qualification’s temporary value; overwritten during bulk reset |
| Distinguished node | +0x0c | 32 bits | Overwritten during bulk reset |

“Distinguished node” identifies the role of the address obtained from context +0x10. Successive reads of that field need not produce the same address.

Context +0x14 is count-like only by inference. Its qualification test accepts every nonzero 32-bit pattern, not just positive values.

### Working storage and result

Two 32-bit working words participate in the general-path result:

- The **context working word** initially contains the original context address.
- The **item working word** initially contains the original output address.

The per-item dependency receives both working-word addresses. Later dependency calls may change their contents through retained references; no preservation contract is supplied.

The return preserves two 32-bit component patterns:

| Path | High 32 bits | Low 32 bits |
|---|---|---|
| Bulk | Node address captured immediately before final publication | Original output address |
| General, no iterations | Original context address | Original output address |
| General, one or more iterations | Context working word at return | Item working word at return |

This is bit-pattern composition, not arithmetic addition or a sign-extension operation.

### Preconditions

Safe execution requires valid storage for every access actually reached, including fields reached through context +0x10. Dependency-specific preconditions also apply.

There is no explicit validation of null addresses, alignment, cursor or boundary representation, structure consistency, aliasing, or address arithmetic. Offsets are byte offsets. Checked address-overflow behavior is not established.

No blanket nonaliasing precondition is visible. Consequently, unconditional behavior is described as ordered accesses rather than as a single final-state invariant.

## Behavior

### Initial publication and preparation

The first direct write places the 32-bit value read from context +0x10 at the output address.

The function then reloads context +0x10 and reads the reached node’s +0x08 field into a temporary word. The initial output write therefore precedes this node-field read. If the output overlaps relevant structure storage, that write can affect subsequent reads.

Before the first dependency call, the function initializes the two result-related working words and a separate temporary containing the original cursor address. The separate temporary is reused for the second qualification call if that call is reached.

The initial publication precedes every dependency call. A later fault or nonlocal exit does not reverse it.

### Bulk qualification

The first call to F-f063d8ade9f7 receives:

1. The original cursor address.
2. The address of the temporary populated from node +0x08.

If the result is zero, the function skips the remaining bulk qualification and enters general-path control.

If the result is nonzero, the function reloads context +0x10 into the separate temporary. A second call to F-f063d8ade9f7 receives:

1. The original boundary value.
2. The address of that temporary.

If the second result is zero, the function enters general-path control without reading context +0x14 for qualification.

If the second result is nonzero, context +0x14 is read. Bulk dispatch occurs only if that field is also nonzero.

Thus both qualification results are tested for **nonzero**, not exact equality to one. Dependency mutations can affect subsequent structure reads. The temporary values passed by address are not subsequently used as saved node addresses for the bulk operation or reset.

### Bulk operation and ordered reset

F-6924b383e664 receives:

1. The original context address.
2. The 32-bit value at +0x04 of the node then reached through context +0x10.

After that dependency returns normally, the function performs these direct writes in order:

1. Write a value read from context +0x10 to +0x08 of a node reached through context +0x10.
2. Write zero to +0x04 of a node reached through context +0x10.
3. Write a value read from context +0x10 to +0x0c of a node reached through context +0x10.
4. Write zero to context +0x14.
5. Read context +0x10, capture that value, and write it to the output address.

It then returns the captured value as the high word and the original output address as the low word. No general-path comparison or iteration follows.

These operations reload context +0x10 rather than consistently using a saved pre-call node address. The bulk dependency can replace that address, and aliasing can make earlier writes affect later reads or destinations. The supplied representation does not establish the relative evaluation order of multiple reads within an individual assignment.

**Reset-state qualification:** If the distinguished-node address remains stable and the reset destinations do not overlap one another, immediately after step 4 and before final publication:

- Node +0x08 and node +0x0c contain the node address.
- Node +0x04 contains zero.
- Context +0x14 contains zero.

Those values are not unconditional postconditions at return. Step 5 can overwrite a reset field through the output address. For example, output at context +0x14 replaces its zero with the captured node address; output at node +0x04 similarly replaces that field’s zero.

To assert that the reset values remain at return additionally requires that final publication and any other permitted mutation do not alter those fields. A stable distinguished-node address alone is insufficient.

Without the stated stability and nonoverlap assumptions, the established contract is the ordered access sequence, not a claim that all reset values coexist in one node or remain present at return.

### General-path control

When bulk dispatch is not selected, F-f063d8ade9f7 receives the original cursor address and original boundary value.

A result of **exactly one** ends the loop. Every other result causes an iteration, including zero, two, and a negative signed result.

This differs from qualification, where any nonzero result permits the next qualification step.

### General-path iteration

Each iteration performs the following sequence:

1. Read and capture the 32-bit value at the original cursor address.
2. Call F-34e6a1a966cb with the original cursor address.
3. Replace the item working word with the captured value.
4. Call F-b3fff165a321 with the context-working-word address, original context address, and item-working-word address, in that order.
5. Read the context working word and write that value to the output address.
6. Perform the next general-path comparison.

The item supplied to F-b3fff165a321 comes from the cursor read **before** F-34e6a1a966cb. A change to cursor storage during that dependency does not replace the already captured value.

The context working word is not reset between iterations. Its current contents carry forward, subject to dependency effects.

The item working word is overwritten before every per-item call, after the cursor-operation call. Consequently, mutations to that word from an earlier iteration may be replaced by the next captured cursor value.

After F-b3fff165a321 has received the working-word addresses, later dependency calls may affect them through retained references. In particular:

- A nonterminating comparison can change a value carried into the next iteration.
- A cursor-operation call can change the context working word before the next per-item call.
- The terminating comparison can change either return component after the last explicit output publication.

Output aliasing also matters: a publication can alter cursor or structure storage used by later operations.

There is no explicit progress check, iteration limit, or rollback.

### General-path completion

When a comparison returns exactly one, the function forms its result from the current working words.

If no iteration occurred, the working-word addresses have not been supplied to F-b3fff165a321. The result contains the original context address above the original output address. The initial publication is the only direct output write on this path.

After one or more iterations, three observations must remain distinct:

- The last explicit output write uses the context working word read after the last per-item call.
- The high return component uses that working word at return.
- The low return component uses the item working word at return.

The terminating comparison can therefore make the high return component differ from the last explicitly published value. Dependencies can also alter output storage through aliases or other reachable references, so final output contents need not equal the last direct write.

Equality between the last per-item values, final published contents, and return components requires the corresponding preservation and nonaliasing assumptions.

## Side effects

### Direct effects

The body directly writes:

- The output word at entry.
- The output word after each general-path iteration.
- On the bulk path, node +0x08, node +0x04, node +0x0c, context +0x14, and then the output word.
- Frame-local temporary and working storage.

No direct allocation, deallocation, external I/O, locking, or explicit global assignment appears in the supplied body.

The final bulk output write is a separate mutation after the reset. It must be included when deriving final-state claims, even when the node address is stable.

### Ownership and dependency effects

Dependency implementations are unavailable. Their global effects, ownership conventions, allocation or release behavior, and failure mechanisms are not established.

The body does not itself visibly release nodes or transfer ownership. Such effects may occur within dependencies, but cannot be inferred as contractual behavior.

Temporary and working-word addresses refer to storage available during this invocation. Retention and later use while the invocation remains active cannot be excluded. No validity guarantee is established for use after return.

### Errors and partial progress

There is no explicit validation error, recovery branch, exception handler, or dedicated error return.

Dependency comparison results are consumed as control values. Zero is not established as an error. A general-path result other than one causes an iteration regardless of any undocumented dependency convention.

A fault or nonlocal exit may leave initial publication, completed iterations, or a partial bulk reset visible. No visible mechanism reverses earlier writes.

## Dependencies

| Opaque ID | Observed arguments | Result handling |
|---|---|---|
| F-f063d8ade9f7 | Qualification arguments involving temporary addresses; general-path arguments consisting of the original cursor address and boundary value | 32-bit result tested for nonzero during qualification and exact equality to one during loop control |
| F-6924b383e664 | Original context address and current node +0x04 value | Return value unused |
| F-34e6a1a966cb | Original cursor address, after cursor-value capture | Return value unused |
| F-b3fff165a321 | Context-working-word address, original context address, item-working-word address | Return value unused; working-word contents subsequently observed |

“Comparison,” “bulk,” “cursor,” and “per-item” describe observed placement or inferred roles, not independently established dependency semantics.

## Acceptance examples

These are conditional static acceptance scenarios, not executed tests. Unless an example specifies aliasing or mutation, assume valid nonoverlapping storage, normal dependency returns, and no additional effects.

1. **Noncanonical qualification results.**  
   Qualification returns two and minus one; context +0x14 contains three. Bulk dispatch occurs because all three tested values are nonzero. With a stable node and disjoint output, the reset values remain present at return, and the high return word is the published node address.

2. **First qualification fails.**  
   The first result is zero. The second qualification is omitted. The next dependency call is the general-path comparison using the original cursor address and boundary value.

3. **Second qualification fails.**  
   The first result is nonzero and the second is zero. Context +0x14 is not read for qualification. General-path comparison follows.

4. **State prevents bulk dispatch.**  
   Both qualification results are nonzero, but context +0x14 is zero. No bulk operation or reset occurs.

5. **Output overlaps context +0x14.**  
   Let context +0x10 hold a stable nonzero node address, with reset destinations otherwise disjoint. The initial publication writes that address into context +0x14. If both qualification calls return nonzero and leave it unchanged, bulk dispatch occurs. The reset clears context +0x14, but final publication writes the node address back there. Its value at return is nonzero, not zero.

6. **Output overlaps node +0x04.**  
   With a stable nonzero node address and successful qualification, initial publication writes that address to node +0x04. If dependencies leave it unchanged, that value becomes the bulk dependency’s second argument. The reset later clears the field, but final publication overwrites the zero with the node address. The field is not zero at return.

7. **Immediate general-path completion.**  
   Bulk dispatch is not selected and the first general comparison returns one. Neither iteration dependency is called. The return contains the original context address above the original output address.

8. **Non-one loop result.**  
   A general comparison returns two. An iteration occurs; nonzero alone does not terminate the loop.

9. **Capture precedes cursor operation.**  
   Cursor storage contains A, and F-34e6a1a966cb changes it to B. The item working word initially supplied to F-b3fff165a321 contains A.

10. **Terminating comparison changes the return.**  
    The per-item dependency retains both working-word addresses and leaves X and Y in them. The function publishes X. The next comparison changes the words to U and V and returns one. The return contains U above V. If output storage is otherwise unchanged, it still contains X.

11. **Mutation carries forward.**  
    After an iteration, a comparison changes the context working word to U and returns zero. If the cursor dependency preserves it, the next per-item call receives that word containing U.

12. **Bulk dependency replaces the node.**  
    The bulk dependency changes context +0x10 from A to B. If B remains valid and stable, subsequent reset accesses use B. Final publication and the high return word use B; final reset-field contents still depend on output aliasing.

## Uncertainties

The following remain **documented_only**:

- Concrete structure type and semantic field names.
- Whether context +0x14 represents cardinality.
- Cursor and boundary representations and comparison meaning.
- Dependency ownership, release, allocation, and failure conventions.
- Dependency retention of temporary or working-word addresses.
- Dependency effects on globals, output storage, and structure fields.
- Validity and intended use of unusual aliasing arrangements.
- Concurrency, synchronization, and reentrancy requirements.
- Relative evaluation order of multiple reads within individual assignments.
- Address-overflow behavior outside valid-access preconditions.
- Intended consumer interpretation of the composed return.

Unknown effects are not preservation guarantees. Stable node identity does not imply stable reset-field contents after final publication.

## Evidence

The supplied body establishes the access widths and offsets, dependency argument relationships, short-circuited qualification, ordered writes, exact-one loop termination, and high/low result composition.

The bulk sequence places final output publication after all reset writes. Consequently, output overlap can undo a reset value without changing the distinguished-node address.

The general sequence exposes both working-word addresses before later comparison calls. No supplied dependency contract guarantees their preservation, so return components are specified from their values at return.

Dependency identifiers establish identity only; no dependency implementation or independent caller contract is available.

This contract is based on static evidence only. No runtime tests or execution traces have been performed.
