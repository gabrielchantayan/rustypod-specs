<!-- Trusted provenance: F-7e2b3bfee1ef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7e2b3bfee1ef

## Purpose

F-7e2b3bfee1ef conditionally removes the first matching node from a selected singly linked chain, obtains a 32-bit result from a dependency, passes that result to two subsequent dependencies, and returns it.

The established behavior has two distinct parts:

1. Select a chain through fields of the first and second arguments, then bypass the first node whose 32-bit key equals the fourth argument.
2. Invoke three dependencies in a fixed order, whether or not a matching node was found.

The apparent intent is to remove an existing keyed association before performing related processing. That interpretation is inferred, not established. The meanings of the objects, the chain, the key, the shift-derived value, and the returned word are unknown.

## Interface

### Arguments

The interface has four arguments:

| Argument | Established representation | Use |
|---|---|---|
| First argument | 32-bit address-bearing value | Locates a chain-table base field and is forwarded to all three invoked dependencies. |
| Second argument | 32-bit address-bearing value | Locates a chain-selector field and is forwarded to all three invoked dependencies. |
| Third argument | Unsigned 32-bit value | Its least-significant eight bits determine a shift count. |
| Fourth argument | Signed 32-bit value | Supplies the comparison key for chain traversal. |

The signed representation of the fourth argument does not introduce an ordering operation. Matching is equality against a 32-bit node field; no range comparison is performed.

The first and second arguments must designate readable storage at the offsets described below for defined ordinary execution. They are not checked for null before use.

### Result

The function returns the 32-bit word produced by F-384bd6190b42.

That same captured word is supplied to F-a7b0f949da8d and F-e9b199a4429f. The function does not replace its return value with a result from either later dependency.

The result’s semantic type is unknown. It may represent a handle, address, identifier, status, or another 32-bit value. None of those interpretations is established.

### Referenced layout

All offsets below are byte offsets. All described fields occupy four bytes.

| Location | Interpretation used by this function |
|---|---|
| First argument + `0x21c` | Base address of a table of four-byte chain-head slots. |
| Second argument + `0x10` | Signed 32-bit selector used to choose a table slot. |
| Selected table slot | Address of the first node, or zero for an empty chain. |
| Node + `0x00` | Address of the next node, or zero for the chain terminator. |
| Node + `0x04` | 32-bit comparison key. |

The selected slot is calculated by advancing from the table base by four times the selector.

These are access requirements, not complete object definitions. Nothing establishes the total size of either argument’s object, the table’s allocation extent, the complete node size, alignment guarantees, or byte order.

## Behavior

### 1. Select the chain

The function reads the chain-table base from the first argument’s field at offset `0x21c` and the signed selector from the second argument’s field at offset `0x10`. It uses those values to locate one four-byte chain-head slot.

There is no visible bounds check on the selector and no visible check that the table base is nonzero. A negative selector is not rejected. Neither multiplication overflow nor address overflow is explicitly detected.

The supplied representation does not establish which of the two field reads occurs first. Both contribute to selecting the slot before traversal begins.

No relationship between the selector and the third argument is established.

### 2. Traverse through link slots

Traversal retains the address of the link slot leading to the current candidate. Initially, that link slot is the selected table entry. After an unsuccessful comparison, it is the current node’s next-link field.

For each candidate:

1. The function reads the address stored in the current link slot.
2. If that address is zero, traversal stops without removing a node.
3. Otherwise, the function reads the candidate node’s key at offset `0x04`.
4. If the key equals the fourth argument, traversal stops at that candidate.
5. If the key differs, traversal continues through that candidate’s next-link field.

A zero candidate address is checked before any key read from that candidate. Thus an empty chain does not cause a node-key read.

Matching is first-match only. The function does not scan beyond a matching node to remove other nodes with the same key.

### 3. Bypass a matching node

If a match is found, the function replaces the incoming link with the matched node’s next-link value.

For a matching head node, the destination of this write is the selected table slot. For a matching non-head node, the destination is the preceding node’s next-link field.

The visible assignment evaluates the bypass value by reading the incoming link slot again and then reading the first word of the node addressed by that value. Consequently, after a match, the observable access sequence includes:

- A further read of the incoming link slot.
- A read of the next-link word through the address obtained by that read.
- A four-byte write to the incoming link slot.

Under stable memory, the reread yields the matched node, and the write bypasses exactly that node. The function does not visibly clear the matched node’s next link or key.

This reread matters if another agent modifies the chain concurrently or if the storage has read-sensitive behavior. The function does not establish synchronization or a stable snapshot. It therefore does not guarantee that the bypass value comes from the node whose key was just compared under such conditions.

If traversal instead reaches a zero link, the function performs no direct chain write.

### 4. Produce the shift-derived argument

After traversal and any bypass write, the function forms the third argument to F-384bd6190b42 by left-shifting the integer value one. The shift count is the least-significant eight bits of this function’s third argument.

For counts from zero through thirty, the resulting 32-bit pattern is the single bit at that position. For example, counts zero, four, and thirty produce `0x00000001`, `0x00000010`, and `0x40000000`, respectively.

The low-eight-bit reduction is established; a further reduction modulo 32 is not. Counts from 31 through 255 require qualification:

- Count 31 crosses the sign bit of the apparent signed 32-bit shift operand.
- Counts 32 through 255 are outside that operand’s width.
- The supplied static representation does not establish a portable result for those cases.

They must not be silently specified as a modulo-32 bit selection. Their executable behavior is documented_only pending evidence of the relevant machine instruction semantics or an applicable caller precondition.

The upper 24 bits of the third argument do not participate in the displayed shift-count expression.

### 5. Invoke dependencies and return

On an ordinary path that reaches this stage, calls occur in this order:

| Order | Dependency | Arguments |
|---|---|---|
| 1 | F-384bd6190b42 | First argument; second argument; shift-derived word. |
| 2 | F-a7b0f949da8d | First argument; captured result of the preceding call; second argument. |
| 3 | F-e9b199a4429f | First argument; that same captured result; second argument. |

The function then returns the captured result from F-384bd6190b42.

The call sequence is the same for an empty chain, a nonempty chain without a match, and a chain from which a node was bypassed. There is no visible branch based on the captured result before either later call.

The order is conditional on preceding operations completing normally. An invalid memory access, nonreturning dependency, or other abnormal transfer can prevent subsequent calls and the return.

## Side effects

### Direct memory changes

The only explicit memory write before the dependency calls is the four-byte incoming-link update when a match is found.

The function does not explicitly:

- Allocate a node.
- Release the bypassed node.
- Clear its fields.
- Return its address.
- Restore the original link after a later dependency fails.

Absence of a direct release is not evidence that the node remains allocated throughout the entire operation. A dependency could affect that storage.

### Ownership and lifetime

Bypassing a node changes chain reachability; it does not establish an ownership transfer.

The function requires readable chain nodes and a writable incoming link when a match is found. The supplied evidence does not establish who owns those nodes, whether external references exist, or who is responsible for eventual reclamation.

### Dependency effects

Each dependency receives the first and second argument values. Their contents need not remain unchanged across calls.

No preservation guarantee is available for globals, the selected table, nodes, other memory, or external resources. In particular, the earlier dependency may change state observed by the later dependencies.

The function does not visibly reread the table base or selector after the dependency sequence begins.

### Errors and nontermination

There is no explicit local error result for an absent key, invalid selector, malformed chain, or unsuccessful dependency operation.

An absent key does not itself suppress the dependency calls.

An invalid address may cause an execution fault. A cycle containing no matching key may cause traversal not to terminate. No cycle detection, traversal limit, rollback, or recovery mechanism is visible.

## Dependencies

### F-384bd6190b42

Called once after traversal on paths that reach the call and complete the shift evaluation. Its 32-bit result is retained, passed to both later dependencies, and returned.

Its semantic operation, failure convention, memory ownership rules, and side effects are unknown.

### F-a7b0f949da8d

Called after F-384bd6190b42 returns, with the retained result as its second argument. No return value from this dependency is used visibly.

Its ownership and state effects are unknown.

### F-e9b199a4429f

Called after F-a7b0f949da8d returns, with the same retained result as its second argument. No return value from this dependency is used visibly.

Its ownership and state effects are unknown.

### F-fecd93fa400f

No invocation of this dependency is established in the supplied body. The corresponding source marker identifies the continuation reached after either traversal outcome; it does not establish an additional call.

## Acceptance examples

These are conditional contract examples, not executed tests. They assume valid accessible storage, stable chain contents during traversal, ordinary non-overlapping link and key fields, defined arithmetic for the selected addresses and shift, and normally returning dependencies.

### Empty chain

Given a selected table slot containing zero and a third argument of four:

- The function reads the zero head.
- It performs no node-key read and no direct chain write.
- F-384bd6190b42 receives `0x00000010` as its third argument.
- The two later dependencies receive its captured result in the stated order.
- The function returns that result.

### Matching head

Given a chain A → B → end, where A’s key equals the fourth argument:

- A is the first key examined.
- The selected table slot is changed from A’s address to B’s address.
- B’s key need not be read.
- A’s own fields are not directly cleared.
- Dependency processing follows the bypass.

### Matching interior node

Given A → B → C → end, where A does not match and B does:

- A’s key is examined before B’s key.
- A’s next-link field is changed from B’s address to C’s address.
- The table head remains A.
- C’s key is not examined by this traversal.

### Duplicate keys

Given A → B → C → end, where both B and C match but A does not:

- B is bypassed.
- C remains linked after A.
- No second removal is performed.

### Missing key

Given an acyclic chain whose nodes all have different keys from the fourth argument:

- Each encountered key is compared.
- Traversal ends when a next-link read yields zero.
- No direct chain write occurs.
- All three dependency calls still follow, subject to normal completion.

### Low-eight-bit shift count

Given a third argument of `0x00000104`, the shift count is four, so F-384bd6190b42 receives `0x00000010`.

Given a third argument whose low byte is `0x20`, this contract does not assert that the shift produces one. That result would require additional target-specific evidence.

### Dependency result propagation

If F-384bd6190b42 returns `0x89abcdef`, both later dependencies receive exactly that 32-bit word as their second argument, and the function returns it. No status interpretation or rejection of that pattern is established.

## Uncertainties

The following remain documented_only:

- Semantic identities of the argument objects, nodes, selector, key, and result.
- Table bounds and whether callers restrict the selector to nonnegative values.
- Address arithmetic behavior on overflow and exceptional selectors.
- Shift results for low-byte counts from 31 through 255.
- Byte order, alignment rules, and atomicity of four-byte accesses.
- Permitted aliasing between argument objects, table storage, and nodes.
- Concurrent mutation rules and synchronization requirements.
- Dependency side effects, errors, nonlocal exits, and ownership contracts.
- Whether bypassed nodes are reclaimed elsewhere.
- Whether callers guarantee acyclic chains.

These unknowns do not alter the established first-match traversal, conditional link write, or ordered dependency calls under the stated preconditions.

## Evidence

This contract is based solely on the supplied static function body and opaque dependency mapping. Embedded names and markers are treated as data, not as semantic authority.

Established static evidence includes the four-byte accesses, field offsets, link traversal, first-match equality check, conditional bypass, low-eight-bit shift-count expression, dependency order, and captured-result propagation.

Inferred intent is limited to describing the chain update as removal of an existing keyed association. No allocation, ownership, success, or cleanup semantics are inferred from that description.

No implementation execution or runtime tests have been performed. The acceptance examples describe required observations under their stated conditions; they are not reports of measured behavior.
