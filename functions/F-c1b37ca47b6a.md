<!-- Trusted provenance: F-c1b37ca47b6a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c1b37ca47b6a

## Purpose

F-c1b37ca47b6a conditionally removes a designated node from a linked tree, updates header links, performs color-based repair, invokes a node-disposition dependency, decrements a 32-bit count, and returns a 32-bit token through caller-provided storage.

The structure and repair operations are consistent with an ordered red-black tree using a header sentinel. That interpretation is inferred intent, not proof that every execution preserves red-black invariants. In particular, the repair state used when a replacement child is absent must not be replaced by textbook deletion behavior.

A preliminary dependency determines whether removal is bypassed. A nonzero result bypasses structural mutation and returns the header token. A zero result proceeds with removal.

## Interface

The function has three arguments and no explicit return value:

| Argument | Required role |
|---|---|
| Output storage | Writable storage for one 32-bit result word. Written at normal completion. |
| Container address | Address of an object containing a header pointer at byte offset 16 and a 32-bit count at byte offset 20. |
| Input-token storage | Readable storage for one 32-bit token. On the removal path, its subsequently reread value is interpreted as a node address. |

The supplied representation uses 32-bit pointer-bearing words. A compatible implementation must preserve that representation rather than substitute native pointer width without an explicit adaptation.

### Container and linked-record layout

The following fields are directly accessed:

| Object | Byte offset | Width | Observed use |
|---|---:|---:|---|
| Container | 16 | 32 bits | Header address |
| Container | 20 | 32 bits | Count |
| Header | 4 | 32 bits | Root address |
| Header | 8 | 32 bits | Cached lower-extreme address |
| Header | 12 | 32 bits | Cached upper-extreme address |
| Node | 0 | 8 bits | Color or repair-state byte |
| Node | 4 | 32 bits | Parent address |
| Node | 8 | 32 bits | Left-child address |
| Node | 12 | 32 bits | Right-child address |

“Left,” “right,” “root,” and “extreme” describe the inferred structural roles of these offsets. No key comparison or payload inspection is performed here.

A zero child word means an absent child. The header is a distinct, non-null structural sentinel. In the explicit empty-tree transition, the header’s root becomes zero and both extreme links become the header address.

Color byte zero is treated as red, and byte one as black, in the repair branches. Some tests distinguish only zero from nonzero, while others require exactly one. Other byte values therefore do not have a uniform “black” interpretation.

### Preconditions

Normal completion requires:

- Valid output, input-token, container, header, and traversed-record storage.
- A removable node with structurally consistent parent and child links when removal is not bypassed.
- Acyclic child chains wherever an extreme node is sought.
- Dependency behavior compatible with continued use of the container and retained node addresses.
- A usable input token after the preparation dependency returns.

These conditions are not validated locally. Malformed links, invalid tokens, inaccessible memory, or dependency-side invalidation can cause invalid accesses or nontermination rather than a reported error.

## Behavior

### 1. Preliminary comparison and result preparation

The function first reads the input token into a local 32-bit result word. It then reads the container’s header word into a separate local word.

It calls F-75663d4eabfa with addresses of those two local words.

If the dependency returns nonzero:

1. The function rereads the container’s header word.
2. That reread value becomes the result.
3. The result is written to output storage.
4. No local structural-removal, disposition, or count-decrement operation occurs.

The dependency may itself have side effects; the bypass is not an unconditional promise of globally unchanged state.

If the dependency returns zero, the function calls F-d7d008b8733c with the address of the local result word. The eventual successful-removal result is whatever that local word contains after this call and subsequent execution. Advancing an iterator is a plausible intent, but is not established by the supplied body.

After this call, the function rereads the input-token storage and uses that value as the node to remove. It does not simply reuse the initial copied token. This distinction is observable if a dependency changes the input storage indirectly.

### 2. Structural removal

For this section, “target” means the node selected by that reread. “Replacement” means the child, parent, or relocated node carried into later repair, as specified below.

#### Target with no children

The target’s parent is read.

If that parent equals the current header address, the function performs the empty-tree transition, in this order:

1. Set the header’s upper-extreme link to the header address.
2. Set the header’s lower-extreme link to the header address.
3. Set the header’s root link to zero.

Otherwise, it reads the parent’s left-child link and compares it with the target:

- If equal, it clears the parent’s left-child word.
- If unequal, it clears the parent’s right-child word.

These clears are expressed as four individual byte stores, in increasing offset order, rather than as a single word store. Atomicity must not be inferred.

The corresponding cached extreme link is then selected: lower extreme for a left-child removal, upper extreme otherwise. If that selected header link equals the target, it is changed to the parent address.

The parent becomes the repair candidate. The target remains the node supplied to the disposition dependency.

#### Target with exactly one child

The existing child becomes the replacement.

The function:

1. Writes the target’s parent address into the child’s parent field.
2. Replaces the target in the header’s root link if the target is the root.
3. Otherwise replaces the matching parent link: left if it equals the target, right otherwise.
4. Updates the header’s lower-extreme link if it names the target.
5. Updates the header’s upper-extreme link if it names the target.

An affected lower extreme is computed from the replacement’s leftmost descendant when the target had a right child; otherwise it becomes the target’s parent. An affected upper extreme is computed symmetrically, using the replacement’s rightmost descendant when the target had a left child.

The child becomes the repair candidate. The target remains the node supplied to disposition.

#### Target with two children

The function finds the leftmost node in the target’s right subtree. This node is the successor for structural replacement. Its original right child, possibly zero, is retained as a potential repair candidate.

The visible structural operations occur in this order:

1. Change the target’s left child’s parent to the successor.
2. Change the successor’s left-child link to the target’s left child.
3. If the successor is the target’s immediate right child, change its right child’s parent to the successor when that child exists.
4. Otherwise:
   - Change the successor’s right child’s parent to the successor’s old parent, when that child exists.
   - Replace the successor’s old parent’s left-child link with that right child.
   - Change the successor’s right-child link to the target’s original right child.
   - Change that original right child’s parent to the successor.
5. Replace the target in the header’s root link or in its parent’s selected child link.
6. Change the successor’s parent to the target’s parent.
7. Exchange the target’s and successor’s color bytes.

If the successor’s original right child exists, it becomes the repair candidate. If it is absent, the relocated successor itself becomes the repair candidate. This latter behavior applies even when the successor was extracted from deeper in the right subtree.

The original target, not the relocated successor, is supplied to disposition. Its post-exchange color controls whether repair begins.

No explicit header-extreme updates occur in this two-child branch.

### 3. Color-based repair

Repair is skipped if either:

- The repair candidate equals the current header address; or
- The detached target’s current color byte is zero.

Otherwise, the function enters repair using the candidate described above. In particular, absent-child cases do not uniformly enter repair as null children: a removed leaf uses its parent, and a two-child removal with no successor right child uses the successor.

At each iteration, repair stops if the candidate is the current root or its color is not exactly one. On this normal repair exit, the candidate’s color is set to one.

For a black, non-root candidate, the function reads its parent and determines whether it is the parent’s left child.

#### Candidate is the left child

The sibling is the parent’s right child.

- **Sibling absent:** Set the candidate’s color to zero and continue with the parent as candidate.
- **Sibling color zero:** Set the sibling to one, set the parent to zero, and call F-8c0e4b5be301. Then reload the candidate’s parent and right sibling. If that sibling is absent, use the preceding absent-sibling transition.
- **Both sibling children absent or exactly color one:** Set the sibling’s color to zero and continue with the candidate’s current parent.
- **Otherwise, sibling’s right child absent or exactly color one:** Set its left child to one if present, set the sibling to zero, and call F-8114a1306b55. Reload the right sibling through the candidate’s current parent. If it is absent, exit repair.
- **Final transformation:** Copy the candidate parent’s color into the sibling; set the parent to one; set the sibling’s right child to one if present; call F-8c0e4b5be301 with the container and candidate’s current parent; then exit repair.

#### Candidate is not the left child

The sibling is the parent’s left child. The branch is structurally mirrored:

- An absent sibling makes the candidate zero and moves repair to the parent.
- A zero-colored sibling becomes one; the parent becomes zero; F-8114a1306b55 is called; parent and sibling are reloaded.
- Two absent-or-one sibling children cause the sibling to become zero and repair to ascend.
- An absent-or-one left child triggers recoloring of the right child, recoloring of the sibling, and F-8c0e4b5be301.
- The final transformation copies the parent’s color into the sibling, makes the parent and any left child one, and calls F-8114a1306b55.

Every exit through the repair completion point writes one to the current candidate’s color byte. No such write occurs when repair was skipped entirely.

The helper calls resemble rotations, but their actual mutations are external dependencies. Some displayed calls supply only the container; others also supply a node address. The incomplete argument evidence must not be silently repaired into an assumed uniform API.

### 4. Disposition, count, and output

After structural mutation and any repair, the function calls F-390d930ffa98 with:

- The container address.
- The original target address.
- The value one.

After that call returns, it reads the current 32-bit count, subtracts one, and writes the result back. Consequently, the decremented value is the count observed after disposition, not a value captured at entry.

Finally, it writes the prepared local result word to output storage.

There is no explicit success status, exception conversion, rollback, or count-underflow check.

## Side effects

Established direct effects include:

- Parent and child link changes in the affected structure.
- Root and cached-extreme header changes in the specified branches.
- Color-byte changes and, for two-child removal, a color-byte exchange.
- A post-disposition count decrement.
- One final 32-bit output write on normal completion.

The count operation is 32 bits wide. For a machine-level contract, subtraction is interpreted modulo \(2^{32}\); zero therefore becomes `0xffffffff`. The supplied body does not establish a source-language signed-overflow guarantee.

Node payloads beyond the listed structural fields are not directly accessed. The function does not locally clear all links of the detached target.

Ownership remains dependency-defined. Passing a node to F-390d930ffa98 does not, by itself, prove destruction, deallocation, ownership transfer, or retention. No subsequent local dereference of that target is shown after disposition, but the container is accessed afterward.

No aliasing restrictions are explicitly checked. If output storage aliases a container field or live node field, the final result write may overwrite earlier state changes. If dependencies alter shared storage or header links, later rereads can observe those changes. Snapshot semantics and concurrency safety are not established.

## Dependencies

| Opaque ID | Established interaction | Unknown contract |
|---|---|---|
| F-75663d4eabfa | Receives addresses of two local 32-bit words; zero selects removal, nonzero selects bypass. | Comparison semantics, local-word changes, indirect effects. |
| F-d7d008b8733c | Receives the address of the result word before structural removal. | Whether it advances a token, what values it produces, other effects. |
| F-8c0e4b5be301 | Called in left-candidate repair and mirrored preparatory repair. | Exact rotation or other transformation; complete argument requirements. |
| F-8114a1306b55 | Called in right-candidate repair and mirrored preparatory repair. | Exact rotation or other transformation; complete argument requirements. |
| F-390d930ffa98 | Receives container, original target, and one before the count decrement. | Destruction, release, allocator behavior, container mutations, failure behavior. |

No dependency is assumed to preserve globals, retain all reachable allocations, or preserve the header or count.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

1. **Bypassed removal.**  
   If F-75663d4eabfa returns nonzero, the output must receive the container’s header word as reread after that call. No local call to preparation or disposition, tree mutation, or count decrement occurs.

2. **Single-node tree.**  
   Given a target whose parent is the header and whose children are zero, the header’s two extreme links become the header and its root becomes zero. The repair candidate equals the header, so repair is skipped. Disposition receives the target, then the current count is decremented.

3. **Leaf at the cached lower extreme.**  
   Given a non-root target that is its parent’s left child, with the header’s lower extreme equal to the target, the parent’s left word is cleared byte by byte and the lower extreme becomes the parent. The parent, not a null child, is carried into repair.

4. **One-child root removal.**  
   Given a root with exactly one child, that child receives the old root’s parent address and becomes the header’s root. Applicable extreme links are updated before repair and disposition.

5. **Deep successor without a right child.**  
   Given a two-child target whose successor lies below the immediate right child and has no right child, the successor’s former parent’s left link becomes zero. The successor moves into the target’s position, exchanges colors with the target, and itself becomes the repair candidate.

6. **Prepared output differs from the target.**  
   If preparation replaces the local result with another token, but leaves input-token storage unchanged, structural removal still uses the reread input target. Output receives the prepared token, not necessarily the removed node’s address.

7. **Disposition changes the count.**  
   If disposition returns with the count equal to 12, the local decrement stores 11, regardless of the entry count.

8. **Nonstandard color value.**  
   If repair is entered with a non-root candidate whose byte is two, the exact-one continuation test fails and repair completion sets it to one. This does not establish general support for arbitrary color encodings.

## Uncertainties

- **documented_only — Result semantics:** The successful-path token’s meaning depends on F-d7d008b8733c.
- **documented_only — Eligibility semantics:** The initial dependency’s nonzero result is a bypass condition; equality with the header is only inferred intent.
- **documented_only — Repair helpers:** Rotation direction, complete argument lists, and helper-side invariants are not independently established.
- **documented_only — Ownership:** No allocation-release guarantee follows from the disposition call alone.
- **documented_only — Failure behavior:** Dependency failures, invalid memory, and interrupted execution have no local recovery contract.
- **documented_only — Global invariant preservation:** Red-black balance and ordered-container correctness require valid input and suitable dependencies; they are not proven here.
- **documented_only — Representation details:** Alignment requirements, byte order, payload layout, and any threading model are not supplied.

## Evidence

Established static evidence consists of the supplied function body’s field accesses, branch conditions, local-word handling, call order, direct stores, and final output assignment.

The red-black-tree interpretation, extreme-link terminology, and likely iterator-preparation and rotation roles are inferred from those operations.

No dependency bodies, independent interface declarations, runtime observations, or runtime tests were provided. No runtime tests have been performed.
