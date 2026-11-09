<!-- Trusted provenance: F-c41d0d56ef3c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c41d0d56ef3c

## Purpose

F-c41d0d56ef3c searches a pointer-linked ordered structure and either requests insertion through F-c8192756568e or reports an existing candidate. It writes a 32-bit result word and a separate one-byte branch indicator to caller-provided storage.

The established behavior is a comparison-directed traversal followed by one of two policies:

- If the context byte at offset 24 is nonzero, insertion is requested without the later candidate-validation comparison.
- If that byte is zero, a boundary check and possible predecessor adjustment determine whether to report an existing candidate or request insertion.

**Inferred intent:** the context represents a tree-like associative container, the dependency F-685cd10e4a2e supplies an ordering predicate, and the zero-byte policy avoids inserting an equivalent item. These interpretations are consistent with the control flow but are not independently established dependency contracts.

The output indicator records the path taken: 1 means the insertion dependency was called; 0 means an existing candidate was returned. It does not independently establish that allocation or insertion succeeded.

## Interface

### Inputs

The function receives three arguments:

1. **Output storage address.** Points to writable storage for:
   - A 32-bit word at offset 0.
   - A one-byte indicator at offset 4.

   Only these five bytes are explicitly written by this function. Any bytes after offset 4 are outside its established output writes.

2. **Context address.** Used as the base of a structure containing:
   - At offset 16: a 32-bit pointer to a header-like object.
   - At offset 24: a one-byte policy field.
   - At offset 25: an embedded address passed as the first argument to the comparison dependency.

   The offset-25 region is passed by address, not loaded as a 32-bit field. Its size, alignment requirements, and contents are unknown.

3. **Opaque 32-bit supplied value.** Forwarded to comparison and insertion dependencies. The function itself does not establish whether this word is a pointer, handle, key, or another representation.

There is no explicit return value. Results are communicated through output storage and dependency side effects.

### Referenced layout

The supplied representation uses four-byte pointer slots and 32-bit words. This is a contract for that representation, not a portable layout for arbitrary host pointer widths.

The header-like object is accessed at:

| Offset | Access |
|---|---|
| 4 | A pointer used as the initial traversal node |
| 8 | A 32-bit word copied into temporary storage for the boundary dependency |

Objects visited as nodes are accessed at:

| Offset | Access |
|---|---|
| 0 | A single metadata byte, during predecessor adjustment |
| 4 | A pointer used as a parent-like link |
| 8 | A pointer selected after a nonzero traversal comparison; also used as a left-like link during adjustment |
| 12 | A pointer selected after a zero traversal comparison; also used as a right-like link during adjustment |
| 16 | An address passed to the comparison dependency |

The meaning of the metadata byte is not established. In particular, no color, balance, or sentinel interpretation is required by the behavioral contract.

### Preconditions and ownership

The function performs no explicit argument validation.

For defined operation, the output address must support the specified writes, and every address actually dereferenced along the selected path must be valid. A null initial traversal node is supported, but the header pointer itself must still be usable for the initial access at offset 4.

Traversals must eventually terminate. Cyclic or malformed links can cause nontermination or invalid memory access; there is no cycle detector or structural repair.

The function does not directly allocate, release, copy, or take ownership of the supplied value or nodes. Any such behavior belongs to dependencies and is unknown here. The lifetime and ownership of the returned 32-bit word are likewise dependency-defined.

## Behavior

### 1. Initial state and traversal

The function initializes a remembered comparison result to 1.

It first obtains the initial node through the context’s offset-16 header pointer and the header’s offset-4 pointer. It then separately loads the context’s offset-16 header pointer as the initial remembered attachment candidate. Thus, the header pointer is loaded twice; the source does not establish one shared snapshot for those two uses.

While the current node is nonnull, the function:

1. Clears two internal 32-bit temporary words to zero.
2. Calls F-685cd10e4a2e with:
   - The context address plus 25.
   - The supplied 32-bit value.
   - The current node address plus 16.
3. Remembers the current node as the attachment candidate.
4. Selects the next node using the returned integer:
   - Exactly zero: read the pointer at current-node offset 12.
   - Any nonzero value: read the pointer at current-node offset 8.

The child-link read occurs after the comparison call. Consequently, it can observe a link modification made by that dependency.

Only zero versus nonzero matters. A negative comparison result takes the same branch as a positive nonzero result; this function does not implement a three-way sign comparison.

If no node is visited, the remembered attachment candidate remains the initially loaded header pointer, and the remembered comparison remains 1.

### 2. Policy selection

After traversal terminates, the function reads the context byte at offset 24.

If the byte is nonzero, it proceeds directly to the common insertion request described below.

If the byte is zero, it initializes an internal cursor to the remembered attachment candidate and continues with candidate validation.

This policy read happens after every traversal comparison. It therefore observes any change to that byte made by those calls; it is not a value captured at entry.

### 3. Boundary handling when the last comparison was nonzero

Under the zero-byte policy, a nonzero remembered comparison triggers a boundary check before the final candidate comparison.

The function reloads the header pointer from context offset 16, reads the 32-bit word at header offset 8, and copies that word into temporary storage. It calls F-2f569f4b2914 with:

- The address of the internal cursor storage.
- The address of the temporary 32-bit word.

This passes the address of a copy of the header word, not the address of the header field itself.

The dependency may modify the cursor through the provided address. Its second argument is also writable temporary storage. The exact dependency contract is unknown; the function subsequently uses the resulting cursor value when the dependency returns zero.

If F-2f569f4b2914 returns nonzero, the function:

1. Calls F-c8192756568e with a temporary result-word address, the context address, a literal zero, the remembered attachment candidate, and the supplied value.
2. Copies that temporary result word to output offset 0.
3. Writes the byte value 1 to output offset 4.
4. Finishes.

This insertion request uses the remembered attachment candidate, not the potentially modified cursor.

If F-2f569f4b2914 returns zero, the function adjusts the cursor as follows.

#### Special metadata-and-parent case

It reads the metadata byte at cursor offset 0.

- If the byte is nonzero, it does not perform the parent-chain check in this case.
- If the byte is zero, it reads the pointer at cursor offset 4 and then the pointer at offset 4 of that referenced object.

If the metadata byte was zero and that twice-followed pointer equals the current cursor, the cursor is replaced by its offset-12 pointer. No additional predecessor walk occurs on this branch.

#### Ordinary predecessor-like adjustment

Otherwise, the function reads the cursor’s offset-8 pointer.

If that pointer is nonnull, the cursor moves to that object and then repeatedly follows offset-12 pointers until the current object’s offset-12 pointer is null. The final cursor is the last object before that null link.

If the initial offset-8 pointer is null, the function follows offset-4 parent-like links. At each step, it compares the parent’s offset-8 pointer with the preceding cursor and moves the cursor to the parent. It continues upward while that equality holds and stops at the first parent for which it does not.

The cursor is moved to that stopping parent as well. This branch does not check for a null parent before dereferencing it; a valid terminating structure is a precondition.

**Inferred intent:** these operations implement predecessor movement with a header/sentinel special case. The contract does not depend on naming the structure a particular kind of tree.

### 4. Final candidate validation

Under the zero-byte policy, final validation is reached either:

- Directly, when the last traversal comparison was zero; or
- After a zero boundary result and cursor adjustment.

The function calls F-e54eaadf2e54 with the current cursor and obtains a 32-bit value. It then calls F-685cd10e4a2e with:

1. The context address plus 25.
2. The value returned by F-e54eaadf2e54.
3. The original supplied value.

The argument order differs from traversal: traversal compares the supplied value against a node-associated address, whereas final validation compares the extracted candidate value against the supplied value.

If this comparison returns exactly zero:

- Output offset 0 receives the current cursor as a 32-bit word.
- Output offset 4 receives byte value 0.
- No insertion dependency is called by this invocation.

If it returns nonzero, execution proceeds to the common insertion request.

### 5. Common insertion request

The common insertion path calls F-c8192756568e with:

1. An address for a temporary 32-bit result.
2. The context address.
3. A literal zero.
4. The remembered attachment candidate from traversal.
5. The original supplied 32-bit value.

It then copies the temporary result word to output offset 0 and writes byte value 1 to output offset 4.

The remembered attachment candidate is retained even when final validation used a different cursor. The adjusted cursor does not replace the insertion attachment argument.

The insertion dependency’s return value is not used to select success or failure behavior. There is no explicit error check before setting the indicator to 1.

## Side effects

### Explicit memory writes

On every normally completing path, this function writes:

1. A 32-bit result at output offset 0.
2. A one-byte indicator at output offset 4.

The word write precedes the indicator write. No atomic publication, synchronization, or concurrent-reader guarantee is established.

Other explicit writes are to internal temporary storage. This function does not directly update node links, the header fields, or the policy byte.

If output storage aliases the context or structure, its writes may consequently modify those objects. No aliasing check prevents that. The ordinary container interpretation assumes caller-selected output storage that does not corrupt required state.

### Dependency-mediated effects

Every dependency may have effects not visible in this function. In particular:

- Comparisons may change globals or reachable objects.
- F-2f569f4b2914 receives writable temporary addresses.
- F-c8192756568e may modify the context or structure, but that behavior is not established here.
- No ownership or memory-release behavior can be inferred merely from a dependency call.

Because structure reads occur between dependency calls, mutation can affect later control flow. A stable ordered-container interpretation requires suitable dependency and concurrency guarantees that are not supplied.

### Errors and abnormal completion

There are no explicit error codes, null checks for required structures, exception handlers, rollback operations, or allocation-failure branches.

A dependency that does not complete normally prevents the later output writes from being guaranteed. A dependency that returns normally without producing an expected result word does not cause this function to detect that condition.

The internal result words are cleared at the start of each visited-node iteration, but no such iteration occurs for an empty initial traversal. Their initialization therefore does not establish a universal fallback result if the insertion dependency fails to write its output.

## Dependencies

| Opaque ID | Established use | Not established |
|---|---|---|
| F-685cd10e4a2e | Returns an integer used only as zero/nonzero. Called during traversal and candidate validation with the argument orders described above. | Ordering laws, purity, accepted operand representations, global preservation, failure behavior. |
| F-2f569f4b2914 | Receives addresses of cursor storage and a copied 32-bit header word. Nonzero requests immediate insertion; zero leads to cursor adjustment. | Whether it is an iterator comparison, whether either temporary is modified, mutation limits, error conventions. |
| F-e54eaadf2e54 | Receives the candidate cursor and returns a 32-bit operand for final comparison. | Whether it extracts a key, returns a reference-like address, allocates, releases, or changes state. |
| F-c8192756568e | Receives a temporary result address, context, zero, remembered attachment candidate, and supplied value. Its output word is copied to caller storage. | Actual insertion semantics, allocation, ownership transfer, duplicate handling, failure reporting, validity of the returned word. |

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, assume valid terminating links and dependencies that return normally.

1. **Nonzero policy after traversal.**  
   If the context byte at offset 24 is nonzero when read, the function must call F-c8192756568e once, using the last visited node as the attachment argument, or the initially loaded header if no node was visited. The output indicator must be 1. Neither boundary handling nor final candidate validation occurs.

2. **Zero traversal result followed by an existing-candidate result.**  
   Suppose the last traversal comparison returns zero and its offset-12 link is null. With a zero policy byte, the last visited node becomes the validation cursor without a boundary call. If final validation also returns zero, output must contain that cursor and indicator 0. F-c8192756568e must not be called.

3. **Nonzero boundary result.**  
   Suppose the policy byte is zero and the last traversal comparison is nonzero. If F-2f569f4b2914 returns nonzero, insertion must be requested immediately. F-e54eaadf2e54 and the final comparison must not be called.

4. **Candidate rejection after adjustment.**  
   Suppose boundary handling returns zero and cursor adjustment reaches a different node. If final comparison returns nonzero, insertion must use the original remembered attachment candidate, not the adjusted node. The output indicator must be 1.

5. **Rightmost descent within an offset-8 subtree.**  
   Suppose ordinary adjustment applies, the cursor’s offset-8 link points to node A, A’s offset-12 link points to node B, and B’s offset-12 link is null. Final extraction must receive node B.

6. **Parent ascent.**  
   Suppose ordinary adjustment applies and the cursor’s offset-8 link is null. If the first parent’s offset-8 link equals the starting cursor, ascent continues. At the first parent whose offset-8 link differs from the preceding cursor, that parent becomes the validation cursor.

7. **Special metadata case.**  
   If the metadata byte is zero and following offset 4 twice returns to the cursor, the adjusted cursor must be its offset-12 pointer. The ordinary offset-8 descent or parent ascent must not occur.

8. **Empty initial traversal.**  
   A null initial node causes no traversal comparison. The remembered comparison remains nonzero. A zero policy byte therefore invokes the boundary dependency; it does not independently guarantee insertion. A nonzero policy byte goes directly to insertion.

9. **Comparison sign does not matter.**  
   A traversal return of −1 selects offset 8, just as a return of 1 does. Only a return of zero selects offset 12.

10. **Output extent and order.**  
    For nonoverlapping output storage with observable surrounding bytes, normal completion writes the result word first and the indicator byte second. This function does not explicitly overwrite bytes after offset 4.

## Uncertainties

The following are **documented_only**:

- The concrete container type, ordering relation, and intended meaning of the policy byte.
- The meaning of header offset 8 and the metadata byte.
- Whether the result word is a node pointer, iterator representation, handle, or other value.
- Whether indicator 1 corresponds to a completed insertion under the dependency contract.
- Allocation failure, exception, cancellation, and ownership behavior of dependencies.
- Required alignment beyond what the supplied target representation permits.
- Behavior under concurrent mutation or dependency-driven structural changes that violate traversal invariants.
- The value copied from insertion temporary storage if the dependency returns without writing it, especially on an empty traversal.
- Structural conditions guaranteeing that predecessor adjustment reaches a valid candidate rather than a null or invalid address.

No stronger behavior is supplied to resolve these uncertainties.

## Evidence

**Established static evidence:** the supplied function body supports the offsets, four-byte data slots, one-byte accesses, dependency call order, zero/nonzero branches, traversal directions, cursor adjustments, and output-write order described above.

**Inferred intent:** ordered-tree search, predecessor selection, duplicate avoidance, and insertion are interpretations of that structure and call pattern. They are not substitutes for missing dependency specifications.

**Runtime evidence:** none. No runtime tests, dependency execution, or behavioral reproduction have been performed.
