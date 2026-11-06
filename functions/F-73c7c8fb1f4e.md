<!-- Trusted provenance: F-73c7c8fb1f4e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-73c7c8fb1f4e

## Purpose

F-73c7c8fb1f4e follows 32-bit links through a structure, maintains a candidate, submits temporary storage to a validation dependency, and writes either the candidate or a freshly read header value to a caller-provided destination. It also returns a 64-bit composite whose upper word is read from candidate storage and whose lower word records whether any comparison call occurred.

Two details distinguish the established behavior from a conventional search:

- A zero traversal-comparison result makes the selected **child link**, not the node just compared, the candidate. This can make the candidate zero.
- Validation receives the candidate’s address. A later comparison can therefore modify that same live storage through a pointer retained by validation. The value used to form the comparison argument need not be the value subsequently written or returned.

An ordered-container lookup is inferred intent, not an established abstract contract. The meanings of the links, header, comparison relation, and validation result remain unspecified.

## Interface

### Arguments and widths

The function accepts four arguments:

| Argument | Observable role |
|---|---|
| Output destination | Address of writable storage receiving one explicit 32-bit result. Its 32-bit address representation also initializes the return’s lower word. |
| Structure address | A 32-bit address-like value used for header access and comparison-context address formation. |
| Search operand | A 32-bit value passed unchanged to the comparison dependency. This function does not directly dereference it. |
| Auxiliary word | A 32-bit value initially copied into temporary storage. That storage is overwritten with a header value before being passed to validation; the original auxiliary value has no subsequent visible use. |

Links, candidate values, header values, and dependency results are 32 bits in the supplied representation. The function returns 64 bits. Dependency results are tested only for zero versus nonzero; positive and negative nonzero results do not select different branches.

### Referenced layout

| Base | Byte offset | Direct use |
|---|---:|---|
| Structure address | 16 | Read a 32-bit header address |
| Structure address | 25 | Form the comparison-context address |
| Initial header address | 4 | Read the initial traversal link |
| Current node address | 8 | Read the next link after a zero comparison result |
| Current node address | 12 | Read the next link after a nonzero comparison result |
| Current node or candidate value | 16 | Form a payload address for comparison |

These offsets describe accesses, not complete object layouts. The function does not directly read the payload at offset 16 or the context at offset 25. Their interpretation belongs to F-31a0b2b5649a.

The representation uses 32-bit address calculations. No overflow check, wider-address protocol, null check, or alignment repair is visible. Valid access to each directly dereferenced address is a precondition for normal execution. Arithmetic behavior outside the valid address domain is documented_only; the contract does not invent wrapping or recovery guarantees.

### Outputs

On normal completion, the function performs one explicit 32-bit write through the original output destination.

The 64-bit return contains:

- **Bits 63–32:** the candidate-storage value read when constructing the return, after the output write.
- **Bits 31–0:** a marker initialized from the output destination’s address representation and cleared before every comparison call.

Thus:

- If no comparison call occurs, the lower word retains the output destination’s address representation.
- If at least one comparison call occurs, the lower word is zero.
- The upper word is not necessarily the value written to the output destination.

The upper word must not be described as a snapshot taken immediately after validation. Candidate storage remains live through the final comparison and is read again afterward.

## Behavior

### Initial reads and traversal state

The initial accesses occur in this order:

1. Read the structure’s header field at offset 16.
2. Read the initial traversal link at offset 4 from that first header address.
3. Read the structure’s header field again, using this second result as the initial candidate.

The two header reads are distinct. Their equality requires an additional stable-memory assumption; the body does not enforce it.

The traversal link and candidate are separate state. A nonzero traversal link causes another iteration. A zero traversal link ends traversal.

### Traversal comparisons and link selection

Before each traversal comparison, the return marker is set to zero. F-31a0b2b5649a then receives, in order:

1. Structure address plus 25.
2. Current node address plus 16.
3. Original search operand.

After the dependency returns, its 32-bit result determines which link is read:

- **Zero:** read the current node’s offset-8 link. That value becomes both the next traversal link and the candidate.
- **Nonzero:** read the current node’s offset-12 link. That value becomes the next traversal link; the candidate is unchanged.

The selected link is read **after** the comparison returns. Dependency-induced changes to that link can therefore affect traversal.

A zero-result step that reads a zero offset-8 link ends traversal with candidate zero. There is no substitution of the previously visited node. A nonzero-result step that reads a zero offset-12 link ends traversal without updating the candidate.

No cycle detection, depth limit, or malformed-link handling is visible. Normal termination requires the selected links eventually to reach zero and all invoked dependencies to return.

### Validation and exposed storage

After traversal, the function reads the structure’s header field again. It prepares two distinct temporary 32-bit words:

- Candidate storage, initialized from the traversal candidate.
- Header storage, initialized from this new header read.

F-356e3995b10d receives the addresses of these two words, candidate first and header second. It does not receive their values by value.

Both words may be modified by the dependency. The caller reads candidate storage later; it does not subsequently use the validation header word to select fallback output.

The source establishes no rule preventing validation from retaining the candidate-storage address or making it available to another dependency. That storage remains live during the subsequent comparison. Its value must therefore be treated as mutable across that call, not frozen at validation’s return.

A nonzero validation result selects fallback immediately. A zero validation result enables the final comparison.

### Final comparison

Before the final comparison, the return marker is cleared to zero. The function reads the then-current candidate value to form the third argument to F-31a0b2b5649a.

The argument order is:

1. Structure address plus 25.
2. Original search operand.
3. Candidate value read before the call, plus 16.

The search operand and payload-address positions are reversed relative to traversal comparisons.

The third argument is a value computed before the call. If the dependency subsequently changes candidate storage, that change does not retroactively change the argument it received. It does affect later reads of candidate storage.

The final result controls selection:

- **Zero:** select candidate storage as the source of the output word.
- **Nonzero:** select fallback.

Selecting candidate storage is not equivalent to preserving the value used in the comparison argument.

### Fallback, output, and return

Fallback reads the structure’s header field again and places that newly read value into separate output-source storage. It does not reuse either the initial header value or the header word passed to validation.

The function then:

1. Reads the selected output-source word.
2. Writes it through the original output destination.
3. Reads candidate storage for the return’s upper word.
4. Returns the 64-bit composite.

For successful candidate selection, the output-source read occurs after the final comparison. For fallback, the output-source word is the fresh fallback header value.

Consequently, validation may leave candidate storage containing one value, while the final comparison changes it to another. A successful comparison then causes the later candidate value to be written. The return also reads candidate storage after that call, rather than using the earlier comparison-argument value.

Under ordinary sequential execution with no further mutation between those reads, successful selection writes the same candidate value that appears in the return’s upper word. The source nevertheless establishes the read order, not a general guarantee against asynchronous mutation.

Fallback output and the returned upper word have different sources: fallback uses the header reread, while the upper word uses candidate storage.

## Side effects

### Direct effects

The function directly:

- Reads the structure’s header field repeatedly.
- Reads the initial traversal link.
- Reads one selected link for each visited node.
- Creates and updates temporary state.
- Exposes addresses of two temporary words to validation.
- Writes one 32-bit word through the output destination on normal completion.

No direct allocation, deallocation, object destruction, or explicit ownership-transfer operation is visible. This does not establish that dependencies preserve memory, avoid allocation, or refrain from releasing objects.

### Ordering and aliasing

Comparison calls precede the traversal-link reads they control. Validation precedes both the optional final comparison and any fallback header reread. The final comparison, when present, precedes successful candidate output selection and the return’s candidate read.

This ordering permits dependency side effects to change:

- A subsequently followed link.
- A subsequently reread header.
- Candidate storage exposed by validation.
- The output value, even after the final comparison’s argument has already been formed.

The final output write follows all direct structure and node reads, but precedes the candidate read used for the return. There is no explicit readback through the output destination.

No general non-aliasing rule is established for the output destination. If it aliases a structure field or node link, the final write can alter that object. Any effect of aliasing other live storage must follow the stated access order; the function provides no protective copy or alias check.

### Lifetime and failure constraints

The temporary words passed to validation remain live during the final comparison and completion sequence. Their validity after this function returns is not established. Retaining an address does not establish ownership or extend the lifetime of its storage.

No explicit error result, exception recovery, invalid-address handling, or dependency-failure translation is visible.

A zero initial traversal link skips traversal comparisons, but still requires valid structure and header accesses. A zero candidate is not rejected before the final comparison. If validation returns zero and leaves the candidate zero, the final comparison’s third argument is address value 16.

Whether any dependency accepts such an argument, returns normally, changes shared state, or terminates execution is documented_only.

## Dependencies

### F-31a0b2b5649a

A three-argument comparison-like dependency whose return is tested only for zero versus nonzero.

During traversal, it receives context address, current-node payload address, and search operand. During final checking, it receives context address, search operand, and the payload address computed from the candidate value available before that call.

Its interpretation of these arguments is unknown. No strict ordering, equality, purity, ownership, or global-state-preservation contract is supplied.

In particular, the final call may access candidate storage through shared dependency state established by F-356e3995b10d. It may change that storage while returning zero. Later output and return reads must observe the resulting storage state under the execution’s applicable memory rules.

### F-356e3995b10d

Called once after traversal with addresses of candidate and header temporary words.

Its zero return permits the final comparison. Its nonzero return selects fallback. Neither result establishes that either temporary is preserved.

The dependency may modify candidate storage and may retain its address for use while that storage is still live. The supplied body does not establish whether it actually does so, how shared dependency state is managed, or whether retained addresses are used after return.

The header temporary is not the fallback source. Mutating only that temporary does not replace the later fallback header reread.

## Acceptance examples

These are conditional deductions from static evidence, not runtime observations. Unless an example states otherwise, assume valid memory, terminating dependencies, no asynchronous mutation, and no aliasing that changes the described temporary values.

### Empty traversal with immediate fallback

Assume the header remains equal to H, its initial traversal link is zero, and validation preserves candidate storage and returns nonzero.

Required behavior:

- No comparison call occurs.
- Validation initially receives H in both temporary words.
- Fallback rereads H and writes it to the output.
- The return’s upper word is H.
- The lower word is the output destination’s address representation.

### Empty traversal with final selection

Assume the initial traversal link is zero. Validation leaves candidate storage containing H and returns zero. The final comparison preserves candidate storage and returns zero.

Required behavior:

- Exactly one comparison call occurs.
- It receives the search operand as its second argument and H plus 16 as its third.
- The output and return’s upper word contain H.
- The lower word is zero.

Skipping traversal does not imply that the lower word remains the output address.

### Child link becomes the candidate

Assume the initial node’s comparison returns zero and its offset-8 link is node C. At C, comparison returns nonzero and the offset-12 link is zero. Validation preserves the candidate and returns zero; the final comparison also preserves candidate storage and returns zero.

Required behavior:

- The first step makes C the candidate.
- The second step leaves it unchanged.
- The output and return’s upper word contain C.
- The lower word is zero.

The initial node must not replace C as the candidate.

### Zero candidate reaches final comparison

Assume a visited node’s comparison returns zero and its offset-8 link is zero. Validation preserves the resulting zero candidate and returns zero.

Required behavior:

- Traversal ends with candidate zero.
- The final comparison receives address value 16 as its third argument.
- No implicit null-candidate fallback occurs.

The eventual output depends on that call’s result and any candidate-storage mutation it performs.

### Validation replacement preserved by comparison

Assume validation sets candidate storage to A and returns zero. The final comparison preserves that storage and returns zero.

Required behavior:

- The final comparison receives A plus 16 as its third argument.
- The output and return’s upper word contain A.
- The lower word is zero.

The preservation assumption is necessary; validation’s assignment alone does not fix the later output.

### Final comparison changes the live candidate

Assume validation retains the candidate-storage address, sets its value to A, and returns zero. The final comparison uses shared dependency state to write B through that retained address, then returns zero.

Required behavior:

- The final comparison receives A plus 16 as its third argument.
- Successful output selection reads B after the call.
- The output contains B.
- The return’s upper word contains B.
- The lower word is zero.

The function must not return or write A merely because A was used to form the comparison argument.

### Fallback after candidate mutation

Assume validation sets the candidate to A and returns zero. The final comparison changes candidate storage to B, changes the structure’s header to H, and returns nonzero.

Required behavior:

- The comparison receives A plus 16.
- Fallback rereads H and writes H.
- The return’s upper word contains B, not A or H.
- The lower word is zero.

### Validation forces fallback after replacement

Assume no traversal comparison occurs. Validation sets candidate storage to B, changes the structure’s header to H, and returns nonzero.

Required behavior:

- No comparison call occurs.
- Fallback writes H.
- The return’s upper word contains B.
- The lower word retains the output destination’s address representation.

## Uncertainties

The following remain documented_only:

- Intended container type, ordering invariant, and meaning of the header.
- Whether the two node links represent conventional left and right children.
- Comparison semantics beyond zero-versus-nonzero branching.
- Validation’s semantic purpose and actual mutation behavior.
- Whether dependencies retain temporary addresses or share them with each other.
- Dependency effects on globals, allocation, ownership, and object lifetimes.
- Alignment requirements and behavior for invalid or overflowing addresses.
- Concurrency and asynchronous-mutation rules.
- Exceptional or nonreturning dependency behavior.
- How callers interpret the composite return relative to the output write.

Possible ordered-search intent does not justify replacing the child-based candidate update, suppressing a zero candidate, caching a header reread, or freezing candidate storage across the final comparison.

## Evidence

Established static evidence consists of the supplied function body: 32-bit accesses and arguments, byte offsets, ordered header reads, branch-dependent link reads, temporary-address arguments, dependency-call order, output-source selection, final output write, and 64-bit return composition.

The successful path selects candidate storage after the final comparison. The return separately reads that storage after the output write. These accesses establish why a candidate mutation during the final comparison can affect both output and return without changing the argument already supplied to that comparison.

Dependency implementations and independent dependency contracts were not supplied. Retained-pointer mutation is therefore a permitted unresolved dependency behavior, not a claim that such mutation actually occurs.

All acceptance examples are conditional deductions. No runtime tests have been performed.
