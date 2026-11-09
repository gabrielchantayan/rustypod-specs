<!-- Trusted provenance: F-1b179da9ccad; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1b179da9ccad

## Purpose

F-1b179da9ccad searches a linked structure using a supplied 32-bit value and a comparison dependency. It then either selects an existing candidate or invokes an insertion dependency.

The public result consists of a 32-bit word followed by a one-byte indicator:

- **0:** the existing-candidate path was taken.
- **1:** an insertion dependency was called and returned normally.

The indicator does not independently establish successful insertion, allocation, or construction. On an insertion path, the result word is whatever occupies the selected temporary word when the function copies it after the dependency returns.

An ordered-tree insertion operation, with predecessor-like navigation and a header sentinel, is inferred intent. The established contract is the access sequence, dependency interactions, branch conditions, and writes described below. No runtime tests have been performed.

## Interface

The function accepts three inputs and has no explicit return value.

| Input | Established use |
|---|---|
| Result address | Destination for a 32-bit word at offset 0 and a byte at offset 4. |
| Container address | Base for structural references, a control byte, and a comparison-context address. |
| Supplied value | Opaque 32-bit quantity passed unchanged to search comparisons and insertion. |

The represented interface uses 32-bit address and word quantities. Comparison results are tested only for zero versus nonzero.

### Public result layout

| Offset | Width | Meaning |
|---|---:|---|
| 0 | 4 bytes | Selected candidate address, or contents copied from an insertion temporary. |
| 4 | 1 byte | Existing-candidate indicator 0 or insertion-call indicator 1. |

The function directly writes no additional public result bytes. The word write precedes the indicator write. No byte-order assumption is needed to specify these separate accesses.

### Container and structural layouts

| Container offset | Access | Role |
|---|---|---|
| 0x10 | Read 32-bit address | Header reference; read more than once on some paths. |
| 0x18 | Read one byte | Nonzero bypasses existing-candidate detection. |
| 0x19 | Form address | Comparison context passed to F-1455d60ba59a. |

The function does not directly inspect the comparison context’s contents. Its size, alignment requirements, and interpretation are dependency-defined.

| Header offset | Access | Role |
|---|---|---|
| 0x04 | Read 32-bit address | Initial traversal node. |
| 0x08 | Read 32-bit word | Boundary value copied into writable temporary storage. |

| Node offset | Access | Role |
|---|---|---|
| 0x00 | Read one byte | Controls a special navigation check. |
| 0x04 | Read 32-bit address | Parent-like navigation link. |
| 0x08 | Read 32-bit address | Search branch for nonzero comparison; also used in neighbor navigation. |
| 0x0c | Read 32-bit address | Search branch for zero comparison; also used in neighbor navigation. |
| 0x10 | Form address | Node payload location supplied to a search comparison. |

These are required access regions, not complete structure definitions.

### Temporary storage

Two distinct 32-bit words are important:

- **Early-insertion temporary:** passed to insertion on the immediate boundary-selected path.
- **Ordinary-insertion temporary:** passed to ordinary insertion and also reused as writable boundary storage.

Both words are explicitly zeroed at the beginning of every search iteration, before that iteration’s comparison call. Neither has an explicit entry-time initialization if the traversal visits no nodes.

On the boundary path, the ordinary-insertion temporary is overwritten with the header’s offset-0x08 word and its address is passed to F-1c076abb7b7f. It is not reset before subsequent ordinary insertion.

A separate writable candidate slot is initialized when existing-candidate detection is enabled. Its address is also passed to F-1c076abb7b7f.

### Preconditions and arithmetic

Every address actually dereferenced must support the specified width. Navigation must terminate, and each selected path must provide readable links and objects. Null links terminate only where explicitly checked; other null or invalid pointers are not handled.

Offsets are byte offsets. No size calculation, allocation-size arithmetic, overflow check, bounds check, cycle detection, or depth limit is present. Address calculations must designate valid accessible regions; no safe wrapping behavior is established.

No supported aliasing contract is supplied. Overlap between the public result and structural data is not rejected and can cause final writes to modify that data.

## Behavior

### Initial reads and search

The search state begins at nonzero value 1.

The function first reads the container’s header reference and then reads the initial node from that header at offset 0x04. It subsequently reads the container’s header reference again, saving this second reference as the insertion-position candidate. These are separate header-reference reads, not one shared snapshot.

For each non-null traversal node, the function:

- Writes zero to the early-insertion temporary, then zero to the ordinary-insertion temporary.
- Calls F-1455d60ba59a with the comparison-context address, the supplied value, and the address at current-node offset 0x10.
- Saves the current node as the insertion-position candidate.
- Reads the next link from that node: offset 0x0c for a zero comparison result, or offset 0x08 for any nonzero result.

The link read occurs after the comparison call. Dependency-mediated changes to the node can therefore affect traversal.

Only zero versus nonzero matters; negative results receive no separate treatment. Search stops when the selected link is null. The saved insertion position is then the last visited node.

If the initial node is null, no search comparison or temporary zeroing occurs. The insertion position remains the second header reference, and the search state remains 1.

### Control-byte decision

After traversal, the function reads the container byte at offset 0x18.

A nonzero byte sends execution directly to ordinary insertion. This bypasses the boundary call, neighbor navigation, candidate extraction, and final comparison.

A zero byte initializes the writable candidate slot with the saved insertion position and enables the remaining decision process. Because this byte is read after search dependency calls, its entry-time value is not controlling if those calls change it.

### Boundary decision

If the final search state is zero, the boundary decision and neighbor navigation are skipped.

Otherwise, the function reads the container’s header reference again, reads that header’s word at offset 0x08, and writes the word into the ordinary-insertion temporary. It calls F-1c076abb7b7f with:

1. The address of the writable candidate slot.
2. The address of the ordinary-insertion temporary containing the copied boundary word.

Both arguments expose storage, not merely values. The dependency can change the candidate slot and the boundary temporary; preservation is not established.

A nonzero dependency result selects immediate insertion using the early-insertion temporary. The insertion-position argument is the saved search position, not the possibly modified candidate slot.

A zero dependency result selects neighbor navigation, beginning with the candidate slot’s post-call value.

### Neighbor navigation: special case

Navigation first reads the byte at candidate offset 0x00.

If the byte is zero, it reads the candidate’s offset-0x04 link and then the referenced object’s offset-0x04 link. If the second link equals the candidate, it replaces the candidate with the link read at candidate offset 0x0c. This completes navigation.

A nonzero initial byte skips those two offset-0x04 reads and enters the ordinary route. A zero byte whose two-link check does not match also enters that route.

### Neighbor navigation: ordinary route

The repeated reads in this route have distinct uses. They must not be merged into a stable-memory snapshot.

First, the function reads candidate offset 0x08 and saves that value as a potential descent pointer. It then independently reads candidate offset 0x08 again:

- If the **second read is zero**, ascent is selected. The saved first value is not used for descent.
- If the **second read is nonzero**, descent is selected using the **first read’s value**, not the second.

Consequently, a nonzero second read does not establish that the saved descent pointer is non-null or otherwise valid.

For descent, each iteration first makes the saved descent pointer the candidate. It then reads that candidate’s offset-0x0c link twice, in order:

1. The first read saves the potential pointer for the next iteration.
2. The second read independently determines whether another iteration occurs.

A zero second read terminates descent with the current candidate. A nonzero second read continues using the first read’s saved pointer. The two values are not assumed equal. In particular, continuation can select an invalid or null saved pointer even when the second read was nonzero.

For ascent, the function initially reads candidate offset 0x04 as the parent. At each step it reads the parent’s offset-0x08 link and compares that value with the previous candidate. It then replaces the candidate with the parent regardless of the comparison result.

- If unequal, ascent ends with that parent as candidate.
- If equal, it reads that parent’s offset-0x04 link and repeats.

There is no null-parent guard. Validity and termination depend on the supplied structure.

The overall pattern resembles predecessor navigation with a sentinel exception. That interpretation does not override the separate-read behavior.

### Existing-candidate decision

After navigation, or immediately after a zero final search state, the function calls F-7a0a658e41aa with the current candidate value.

It then calls F-1455d60ba59a with:

1. The comparison-context address.
2. The 32-bit value returned by F-7a0a658e41aa.
3. The original supplied value.

The supplied value occupies a different argument position than in search comparisons. No symmetry or conventional comparator semantics are established.

If the comparison returns zero, the function reads the candidate slot for the public result word and selects indicator 0. It does not call insertion.

If the comparison returns nonzero, it selects ordinary insertion.

Where the candidate slot’s address previously escaped to F-1c076abb7b7f, its preservation across later dependency calls is not established. The value passed to extraction and the value subsequently copied to the public result must not be assumed identical without an additional dependency-preservation guarantee.

### Insertion and temporary contents

Both insertion paths call F-c5a1ac02ea74 with five arguments:

1. Address of the selected insertion temporary.
2. Original container address.
3. Literal zero.
4. Saved insertion position from the search.
5. Original supplied value.

The immediate boundary-selected path uses the early-insertion temporary. All other insertion paths use the ordinary-insertion temporary.

The established initialization history is:

| Selected temporary and path | Established direct initialization before insertion |
|---|---|
| Early insertion after nonempty traversal | Zeroed during every search iteration; no later direct full-word overwrite before insertion. |
| Early insertion after empty traversal | No explicit initialization. |
| Ordinary insertion, nonempty traversal, boundary skipped | Zeroed during every search iteration; no later direct full-word overwrite before insertion. |
| Ordinary insertion, empty traversal, nonzero control byte | No explicit initialization. |
| Ordinary insertion after boundary processing | Overwritten with the header boundary word, then exposed by writable address to F-1c076abb7b7f; not reset afterward. |

The boundary-exposed word is not guaranteed to retain either zero or the copied header value. It may be modified during the boundary call or through an escaped address during later dependency calls.

Insertion is not established to treat its first argument as output-only, nor to overwrite it completely. After insertion returns, the function reads the selected temporary and copies its then-current 32-bit contents into the public result. It does not validate those contents or inspect a separate status.

If insertion leaves explicitly uninitialized storage unreadied, the supplied evidence establishes no deterministic public result word or defined failure behavior.

### Completion writes

On every normally completing path, the function writes the public result word first.

It then stages the indicator in the byte at offset 0 of the early-insertion temporary and writes that byte to public result offset 4. Thus, on early insertion, the temporary whose full word was just copied is subsequently modified by a one-byte write of 1. Other completion paths likewise overwrite this staging byte with their indicator.

There is no atomic publication guarantee.

## Side effects

Direct effects include structural reads, temporary writes, dependency calls, and the two public result writes. The function does not directly update structural links or the container control byte, allocate storage, or release storage.

Insertion-related structural changes are delegated to F-c5a1ac02ea74. The source does not establish allocation, balancing, construction, ownership transfer, or failure behavior inside that dependency.

No dependency is established as pure or as preserving globals, container fields, node fields, or previously exposed temporary storage. Repeated reads retain their separate roles; stable structural memory is not assumed.

Temporary addresses are borrowed call-site storage, not evidence of ownership transfer. Their validity after this invocation is not established. Returning a candidate address likewise does not transfer ownership of that node.

There is no local error result, rollback, exception handler, retry, or recovery path. Dependency failures are not translated into a defined public error. A normally returning insertion call results in indicator 1 even when its copied word is zero.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-1455d60ba59a | Receives three arguments; returns a 32-bit integer tested only for zero versus nonzero. Search and final-decision argument order differ. |
| F-1c076abb7b7f | Receives writable candidate storage and writable ordinary-insertion/boundary storage. Nonzero selects immediate insertion; zero selects navigation. |
| F-7a0a658e41aa | Receives the candidate value and returns a 32-bit value used in the final comparison. |
| F-c5a1ac02ea74 | Receives selected temporary storage, container, zero, saved insertion position, and supplied value. The caller subsequently copies the temporary’s contents, without proving the dependency wrote them. |

No stronger semantic, memory-preservation, ownership, or error contracts are established.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Successful completion assumes valid accessed memory, terminating navigation, and normally returning dependencies.

1. **Control-byte bypass.** After a nonempty traversal, a nonzero control byte causes ordinary insertion without boundary processing or candidate extraction. The ordinary temporary has been explicitly zeroed during traversal. Whatever word is present after insertion is copied, followed by indicator 1.

2. **Direct candidate selection.** With control byte zero and final search state zero, boundary processing and navigation are skipped. A zero final comparison selects the candidate word and indicator 0 without insertion.

3. **Boundary mutation and early insertion.** A nonzero boundary result selects early insertion even if the boundary dependency changed the candidate slot or boundary word. Insertion still receives the saved search position. After a nonempty traversal, the early temporary has the explicit zero-initialization history described above.

4. **Distinct branch reads select descent.** Suppose the first ordinary-route offset-0x08 read yields address A and the second yields a different nonzero address B. Descent begins at A, not B. If A is invalid, B’s validity does not rescue the access.

5. **Distinct branch reads select ascent.** If the first offset-0x08 read yields A but the second yields zero, ascent is selected. A is not followed.

6. **Distinct descent reads control continuation.** At a descending candidate, suppose the first offset-0x0c read yields C. A zero second read terminates at the current candidate without following C. A nonzero second read continues using C, regardless of the second read’s actual pointer value.

7. **Reused boundary storage reaches ordinary insertion.** If the boundary dependency returns zero after replacing its boundary word with W, navigation completes, and the final comparison is nonzero, ordinary insertion receives that same storage without a reset. If later dependencies preserve it and insertion does not write it, W is copied to the public result.

8. **Empty traversal distinguishes initialization paths.** With an initially null node and nonzero control byte, ordinary insertion receives a word lacking explicit initialization. With control byte zero, boundary processing initializes the ordinary word from the header; however, immediate insertion uses the separate early word, which still lacks explicit initialization.

9. **Zero insertion word and subsequent staging.** If insertion returns with its selected word equal to zero, the public word becomes zero and the indicator becomes 1. On early insertion, the subsequent indicator staging also changes byte offset 0 of that temporary to 1 after the zero word has been copied.

10. **Negative comparison results.** A negative search result selects offset 0x08. A negative final comparison selects ordinary insertion.

## Uncertainties

The following remain **documented_only**:

- Exact container type, ordering relation, and meaning of the control byte.
- Whether comparisons implement ordering, equality, or another predicate.
- Semantic meaning of the boundary dependency, node byte, and special navigation case.
- Interpretation and ownership of the supplied 32-bit value.
- Dependency allocation, release, balancing, construction, and failure behavior.
- Whether dependencies read, fully overwrite, partially overwrite, or retain access to temporary storage.
- Preservation of escaped candidate and boundary storage across later calls.
- Defined behavior when an insertion temporary lacks explicit initialization and is not fully written.
- Supported aliasing, concurrency, reentrancy, and synchronization guarantees.
- Behavior for invalid pointers, cycles, malformed structures, or nonreturning dependencies.

Repeated reads are explicitly distinguished; whether valid callers ensure their values agree is unknown.

## Evidence

Established static evidence includes input uses, access widths and offsets, repeated header and navigation reads, per-iteration temporary zeroing, boundary-storage reuse, dependency argument order, branch predicates, and public write order.

Ordered-tree insertion, predecessor selection, sentinel use, and existing-element detection are inferred intent only.

No dependency implementations, runtime observations, allocation traces, or executed tests establish stronger guarantees.
