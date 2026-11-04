<!-- Trusted provenance: F-b951efde8a28; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b951efde8a28

## Purpose

F-b951efde8a28 performs an in-place structural rotation between a selected node and the node referenced by its child slot at byte offset 8. It promotes that child into the selected node’s former position, makes the selected node the promoted node’s child at byte offset 12, and transfers the promoted node’s former offset-12 child to the selected node’s offset-8 child slot.

This structural effect is established by the supplied static evidence. Interpreting offset 8 as a right-child link and offset 12 as a left-child link makes the operation a conventional left rotation. Those directional names, the use of a binary search tree, and any ordering or balancing purpose are inferred intent, not established requirements.

The operation changes links only. It does not inspect payloads, compare keys, update heights or colors, allocate nodes, or release storage.

## Interface

### Inputs and return

The function accepts two address-bearing arguments:

| Argument role | Required use |
|---|---|
| Context | Identifies an object containing a root-holder reference at byte offset 16. |
| Selected node | Identifies the node whose offset-8 child will be promoted. |

There is no explicit return value. The observable result consists of memory mutations.

The supplied representation uses 32-bit integer values and four-byte field accesses for the relevant addresses and links. This contract therefore describes four-byte address-bearing slots. It does not establish a portable native-pointer interface on machines with wider pointers.

All offsets below are byte offsets.

### Referenced layouts

Only the following fields are relevant:

| Object role | Offset | Meaning in this contract |
|---|---:|---|
| Context | 16 | Address of the root holder |
| Root holder | 4 | Root-node address |
| Node | 4 | Parent address |
| Node | 8 | Child-A address |
| Node | 12 | Child-B address |

“Child A” and “child B” are neutral names for the two child slots. Under the conventional rotation interpretation, child A is the right child and child B is the left child.

Four node roles describe the operation:

- **Selected node:** the second argument.
- **Promoted node:** the selected node’s initial child A.
- **Transferred subtree:** the promoted node’s child B, when nonzero.
- **Former parent:** the selected node’s former parent, used when the selected node is not the root.

The context and root holder are separate access roles. The function reads the holder address through the context rather than treating the context itself as the root holder.

### Preconditions for the ordinary rotation contract

The normal structural result requires:

- The context, root holder, selected node, and promoted node to provide valid storage for every accessed field.
- The selected node’s child-A address to identify a usable promoted node. No null check protects this access.
- A nonzero transferred-subtree address to identify a node with a writable parent field.
- If the selected node is not the recorded root, its parent field to identify a node with accessible child slots.
- Relevant address additions to resolve to valid fields.
- No concurrent mutation or unintended storage overlap that changes the meaning of intermediate reads and writes.

A well-formed tree additionally requires the selected node to be linked from its former parent, or to be the recorded root, and requires consistent parent-child relationships. The function does not validate these invariants.

## Behavior

### Ordinary structural result

For a valid, stable, nonoverlapping structure, the operation produces these changes:

| Location | Value after completion |
|---|---|
| Selected node, child A | Former child B of the promoted node |
| Transferred subtree, parent, if the subtree exists | Selected node |
| Promoted node, parent | Former parent of the selected node |
| Root-holder root, if selected node was root | Promoted node |
| Former parent’s applicable child slot, otherwise | Promoted node |
| Promoted node, child B | Selected node |
| Selected node, parent | Promoted node |

The selected node’s child B and the promoted node’s child A are not directly written. No other payload fields are directly written.

The root-holder root changes only when its value compares equal to the selected-node address. A null parent value alone does not identify the root.

When the selected node is not the root, the former parent’s child-B slot is checked first. If that slot equals the selected-node address, child B is replaced. Otherwise, child A is replaced without checking whether child A actually refers to the selected node.

### Observable access order

The order matters because the operation is not atomic, and several values are read again rather than captured once for the whole operation.

**Subtree transfer occurs first.** The function obtains the promoted-node address from the selected node’s child-A slot. It then reads the promoted node’s child-B slot and writes that value into the selected node’s child-A slot.

The promoted node’s child-B slot is read again to determine whether a transferred subtree exists. If this read is nonzero, that same slot is read once more to obtain the address whose parent field is written with the selected-node address.

Consequently, in an ordinary stable structure, the transferred subtree’s parent is repaired immediately after its new downward link is installed. Under concurrent changes or overlapping storage, the repeated reads need not yield the same value.

**The promoted node’s parent is installed next.** The selected node’s parent field is read and copied into the promoted node’s parent field. This occurs before the function determines whether the selected node is the root.

**The incoming link is then redirected.** The function reads the root-holder address from the context and reads the holder’s root field.

- If that root field equals the selected-node address, it writes the promoted-node address to the root field.
- Otherwise, it reads the selected node’s parent field again. It checks that parent’s child-B field against the selected-node address. A match causes child B to be replaced; a nonmatch causes child A to be replaced.

The root-holder address is obtained after the earlier subtree and parent writes. It is not captured at entry. Likewise, the parent used for non-root reconnection is obtained by a later read, distinct from the earlier read copied into the promoted node.

**The final node-to-node links are installed last.** The promoted node’s child-B field is written with the selected-node address. The selected node’s parent field is then written with the promoted-node address.

These are the ordered accesses represented by the supplied static material. No stronger guarantee about processor-level ordering, compiler transformations, or visibility to other threads is established.

### Branch selection and malformed structures

There are two explicit structural decisions:

1. Whether the promoted node’s child B is nonzero.
2. Whether the root-holder root equals the selected node, followed on the non-root path by the former parent’s child-B equality check.

The function does not establish that the promoted node is distinct from the selected node, that the structure is acyclic, or that any parent-child pair is mutually consistent.

If the former parent refers to the selected node in both child slots, the child-B match takes precedence: child B is replaced, while child A is not directly changed by that branch.

If neither parent child slot refers to the selected node, child A is still overwritten. This is the established fallback behavior, not an error-reporting mechanism.

### Arithmetic and field widths

The operation performs fixed-offset address calculations and four-byte loads and stores. It does not perform key arithmetic, calculate sizes, increment counts, or update balancing metadata.

The supplied representation does not establish signed-overflow semantics, address-wrap behavior, alignment tolerance, or whether every possible 32-bit slot value is a valid address. Inputs must keep all effective accesses valid for the actual execution environment. Address truncation or wraparound is not a supported recovery mechanism.

## Side effects

All intended side effects are writes to existing objects. On an ordinary nonoverlapping execution, there are five structural field writes, plus one additional parent-field write when a transferred subtree exists:

- Selected node’s child A.
- Transferred subtree’s parent, conditionally.
- Promoted node’s parent.
- Root-holder root or one former-parent child slot.
- Promoted node’s child B.
- Selected node’s parent.

The function neither allocates nor frees storage. It does not explicitly acquire ownership of the context, root holder, nodes, or subtree. Their storage must remain accessible throughout the operation; ownership policy beyond that lifetime requirement is not established.

No lock acquisition, callback, external call, explicit global-variable access, logging, or status reporting is present in the supplied body. Caller-supplied addresses may nevertheless designate shared or globally reachable memory.

The mutation is not transactional. Intermediate states can contain incomplete or inconsistent reciprocal links. If an access fails after earlier writes have occurred, no rollback behavior is provided by this function. The exact fault or exception behavior belongs to the execution environment and is not established here.

## Dependencies

No called-function dependencies are present in the supplied static evidence. There are therefore no additional opaque function IDs to list.

Execution depends on valid memory access and on the caller providing an appropriate representation and synchronization regime. There is no visible dependency that validates the tree, repairs unrelated links, or maintains auxiliary metadata.

## Acceptance examples

The following are conditional static acceptance examples, not executed tests. Each assumes valid four-byte slots, stable memory, and no unintended aliasing. Unmentioned fields retain their values unless they overlap a written location.

### Root rotation with a transferred subtree

Initial relationships:

- The root holder identifies the selected node as root.
- The selected node’s parent is zero.
- The selected node’s child A is the promoted node.
- The promoted node’s child B is a nonzero subtree node.

Required result:

- The root holder identifies the promoted node as root.
- The promoted node’s parent is zero.
- The selected node’s child A identifies the transferred subtree.
- The transferred subtree’s parent identifies the selected node.
- The promoted node’s child B identifies the selected node.
- The selected node’s parent identifies the promoted node.

The selected node’s former child B and the promoted node’s former child A remain untouched by direct writes.

### Non-root rotation through the former parent’s child B

Initially, the root holder identifies some other node, the selected node’s parent identifies the former parent, and that parent’s child B identifies the selected node.

After completion, the former parent’s child B identifies the promoted node, and the promoted node’s parent identifies the former parent. The root-holder root remains unchanged. The subtree transfer and final reciprocal links follow the ordinary result above.

### Non-root rotation through child A

Initially, the selected node is not the recorded root. Its former parent’s child B does not identify it, while child A does.

After completion, child A identifies the promoted node and child B remains unchanged. The function chooses this path because the child-B comparison fails; it does not separately validate the child-A relationship.

### No transferred subtree

Initially, the promoted node’s child B is zero.

The selected node’s child A becomes zero. No transferred-subtree parent write occurs. The promoted node is still reconnected to the former parent or root holder, and its child B becomes the selected node.

### Inconsistent former-parent relationship

Initially, the selected node is not the recorded root, and neither child slot of its recorded parent identifies it.

Provided all accesses remain valid, the parent’s child-A slot is overwritten with the promoted-node address. No rejection or error return is expected.

### Missing promoted node

If the selected node’s child A is zero or otherwise unusable, there is no defined graceful no-op contract. The function proceeds toward accessing the promoted node’s child-B field without validation. The resulting runtime outcome is documented_only.

## Uncertainties

- **Inferred intent:** The field arrangement is consistent with a left rotation when child A means right and child B means left. Ordering and balancing semantics are not established.
- **documented_only — ABI:** Exact calling convention, pointer ABI, and portability beyond the displayed four-byte representation are unknown.
- **documented_only — memory faults:** The environment’s response to invalid, misaligned, protected, or wrapped addresses is unknown.
- **documented_only — aliasing:** Ordinary tree results assume distinct, appropriately laid-out objects. Arbitrary overlapping objects can change later reads and branch choices; no alias-safe tree transformation is promised.
- **documented_only — concurrency:** No synchronization contract or atomic visibility guarantee is established.
- **documented_only — ownership:** Allocation provenance, lifetime management outside this call, and permission to mutate shared structures are unknown.
- **documented_only — root-parent convention:** A zero root parent is plausible but not required by the observed operations. The selected node’s existing parent value is copied even on the root path.
- **documented_only — metadata:** No evidence establishes whether another operation must update counts, heights, colors, or other auxiliary state.

## Evidence

The contract is based solely on the supplied static function body for F-b951efde8a28. That evidence establishes the accessed offsets, four-byte field operations, comparisons, branch precedence, repeated reads, write order, absence of called functions, and absence of an explicit return value.

The rotation interpretation is a structural inference from those link changes. It does not establish the surrounding data structure’s type or broader invariants.

No implementation access, tool inspection, runtime execution, or tests were performed. The acceptance examples express conditional consequences of the supplied static evidence, not observed execution results.
