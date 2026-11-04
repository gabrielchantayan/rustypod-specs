<!-- Trusted provenance: F-d3c2a8b5a34b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d3c2a8b5a34b

## Purpose

F-d3c2a8b5a34b advances through linked records from an initial record toward a caller-supplied stopping value, incrementing a caller-owned counter once for each visit that does not match that stopping value.

The links and traversal order are consistent with an in-order walk of a parent-linked binary tree. That interpretation is inferred intent, not a validated input requirement enforced by the function. The established behavior is a sequence of address comparisons, counter updates, and link reads. In particular, the function does not validate tree structure, guarantee forward progress, or detect cycles.

The stopping record is excluded from counting. The counter is accumulated into rather than initialized. The function returns no traversal position or status.

## Interface

The function takes three addresses:

| Input | Required storage | Role |
|---|---|---|
| Initial-position address | One readable integer word | Contains the initial record address or stopping value. Read once on entry. |
| Stopping-value address | One readable integer word | Contains the value against which the current traversal position is compared. Read again at each visit. |
| Counter address | One readable and writable integer word | Contains the running count. Incremented once per non-stopping visit. |

The supplied representation uses four-byte integer values and four-byte link slots. Values obtained from the initial-position word and link slots are subsequently used as addresses. This supports a 32-bit address-bearing representation; it does not independently establish a native pointer ABI, byte order, or behavior when adapting the function to a wider-address environment.

Each traversed record has the following accessed layout:

| Byte offset from record address | Accessed width | Traversal role |
|---|---:|---|
| +4 | 4 bytes | Parent link |
| +8 | 4 bytes | Left link |
| +12 | 4 bytes | Right link |

The function does not access the word at offset +0 or any payload beyond these links. Zero is treated as an absent child when inspecting right and left links. Zero is not treated as a universally safe traversal position or parent terminator.

There is no return value. The externally useful result is the modified counter.

### Preconditions for defined, terminating use

The following are caller obligations, not checks performed by the function:

- The initial-position and stopping-value addresses must support their required reads.
- Whenever a visit is not the stopping value, the counter address must support a read and write.
- Every record link actually inspected must reside in valid readable storage.
- Every nonzero child address followed during descent must identify a record with the required left-link storage.
- Every parent address examined during ascent must identify a record with the required right-link storage. Further ascent also requires readable parent links.
- The reached topology and stopping convention must eventually produce a visit equal to the stopping value if return is required.
- Counter arithmetic must remain within the supported integer domain unless wrapping behavior is independently established.

The stopping value need not identify a readable record if it is encountered directly at the visit comparison. However, a record used as a structural sentinel may be read during ascent before it becomes the current position.

## Behavior

### Initial position and termination check

The function reads the initial-position word once and retains that value as its initial current position. It does not reread that input to restart or redirect traversal.

At the beginning of every visit, it reads the stopping-value word and compares that value with the current position.

If they are equal, the function returns immediately. For that visit:

- The counter is not read or written.
- No record links are inspected.
- The stopping position is not counted.

Consequently, equal initial and stopping values permit immediate return even when that value would not be a valid record address. The two input words themselves must still be readable.

If the current position differs from the freshly read stopping value, the function reads the counter, adds one, and writes the result back before reading any links for that visit.

This ordering matters for aliasing and invalid topology: a visit is counted before its outgoing traversal links are processed.

### Right-child descent

After incrementing the counter, the function obtains the current record’s right-link value. The supplied representation then reads that right-link slot again to decide whether it is zero.

When the tested right link is nonzero, traversal descends into the right-side subtree:

- The previously obtained right-link value supplies the first descent position.
- At each descent position, the left-link value is obtained.
- The left-link slot is read again for the zero test.
- A nonzero tested left link causes descent to continue using the previously obtained left-link value.
- A zero tested left link ends descent.

The resulting position becomes the next visited position. The stopping comparison occurs there before its counter increment.

Intermediate records passed while following left links are not themselves counted merely because they were inspected. Counting occurs only at visit boundaries.

Under stable links, this selects the leftmost record reachable from the current record’s right child. If repeated reads of a link disagree, the value used as the next address and the value used for the branch decision can differ. The function contains no reconciliation or retry mechanism.

### Parent ascent

When the tested right link is zero, the function reads the current record’s parent link.

It then examines that parent candidate’s right link:

- If the candidate’s right link equals the current position, traversal moves the working position up to that candidate, reads its parent link, and repeats the comparison against the next parent candidate.
- If the candidate’s right link does not equal the working position, the ascent loop ends.

After ascent ends, the function reads the working position’s right link and compares it with the remaining parent candidate.

There are two outcomes:

| Final comparison | Next visited position |
|---|---|
| Working position’s right link differs from the parent candidate | The parent candidate |
| Working position’s right link equals the parent candidate | The working position remains current |

The second outcome is material. The function does not invariably replace the current position with the parent candidate after ascent. Under that relationship, the retained working position reaches another visit comparison and can be counted again.

For ordinary tree links and a compatible sentinel arrangement, this structure is consistent with in-order successor selection. The exact handling of the root and endpoint depends on the caller’s parent and sentinel links. A null parent is not automatically recognized as traversal completion.

### Repetition and stopping scope

After selecting or retaining a position, the function repeats the visit comparison against a fresh read of the stopping-value word.

The function does not compare the stopping value against every record whose links it reads. A record can be examined during descent or ascent without triggering termination at that moment.

There is no independent bound, node count, cycle check, or structural-error exit. If the stopping value is never reached at a visit boundary, the function may continue indefinitely, encounter invalid storage, or overflow the counter.

## Side effects

The only explicit stores are updates to the counter word. There are no explicit stores to the initial-position word, stopping-value word, or record links.

That statement does not imply that those storage locations remain unchanged: the counter may alias them. If it does, a counter update can also alter an input word or a link used by subsequent traversal.

Important aliasing consequences include:

- If the counter aliases the initial-position word, the stored initial value changes, but traversal does not reread that word as an initial position.
- If the counter aliases the stopping-value word, a non-stopping visit changes the value used by later termination checks.
- If the counter aliases a record link, the increment occurs before the links for that visit are inspected and can therefore change traversal.
- If both input addresses identify the same word, the initial read and first stopping-value read agree in the absence of an intervening modification, producing immediate return.

No disjointness checks are present. Acceptance examples using ordinary tree behavior assume stable, non-aliasing inputs.

The function performs no allocation, deallocation, ownership transfer, callback, or external function call in the supplied body. All referenced storage remains caller-provided. The function supplies no synchronization or atomicity guarantee.

There is no explicit error reporting. Invalid addresses are used directly. The supplied representation does not establish whether an invalid access would cause a process fault, another runtime failure, or some platform-specific outcome. If failure occurs after an increment, there is no rollback of earlier counter writes.

## Dependencies

No callable dependencies are present in the supplied body. F-d3c2a8b5a34b performs its work through direct input, counter, and link accesses.

Its environmental dependencies are:

- Valid storage for every access actually reached.
- A consistent interpretation of four-byte address-bearing words.
- Appropriate integer arithmetic semantics.
- A caller-supplied topology and endpoint convention that permit the requested traversal.
- Sufficient memory stability for any expected tree interpretation.

No behavior of an unknown callee is assumed. In particular, the contract does not rely on another function preserving globals or managing memory.

## Acceptance examples

These examples are conditional consequences of the supplied static representation, not results of runtime testing. Unless stated otherwise, links and input words remain stable, storage does not alias, all reached accesses are valid, and counter additions are representable.

### Equal initial and stopping values

Given an initial-position word and stopping-value word containing the same value, and a counter initially containing 17:

- The function returns on the first visit comparison.
- The counter remains 17.
- No record link or counter storage is accessed.

The shared value may be zero because it is compared, not dereferenced.

### Stop at the right-subtree minimum

Suppose record A has a nonzero right link to B, B has a left link to C, and C has a zero left link. Let the stopping value be C and the initial position be A.

- A is counted once.
- The right link of A and the left-link chain through B and C are inspected.
- C becomes the next visited position.
- The function returns at C without counting C.
- B is not counted merely because its left link was inspected.

With an initial counter of 9, the resulting counter is 10.

### Ascend from a left child to its parent

Suppose A has no right child, A’s parent is P, P’s right link is not A, and A’s right link differs from P. Let the stopping value be P.

- A is counted once.
- The ascent comparison stops at P.
- P becomes the next visited position.
- The function returns without counting P.

This example depends on the stated link relationships, not simply on a label such as “left child.”

### Skip ancestors already completed on their right side

Suppose A has no right child and its parent P has A as its right child. Suppose P’s parent Q does not have P as its right child, and P’s right link differs from Q. Let the stopping value be Q.

- A is counted once.
- Ascent passes through P.
- Q becomes the next visited position.
- P is not counted during that ascent.
- The function returns at Q.

### Retained-position branch

Suppose ascent ends with working position W and parent candidate P, where P’s right link is not W but W’s right link is P.

The next visit uses W rather than P. If W does not equal the freshly read stopping value, the counter is incremented again before traversal continues.

A specification that always selects P after ascent would fail to describe this branch.

### Zero initial position

If both initial and stopping values are zero, the function returns without accessing a record.

If the initial position is zero and the stopping value is nonzero, the function first increments the counter and then attempts record-link access relative to zero. There is no explicit null-position error path.

### Counter arithmetic boundary

If the counter is one below the largest supported signed value and exactly one non-stopping visit precedes termination, the final value is that largest signed value.

If a further increment exceeds the signed range, the supplied representation does not justify accepting modular wraparound as the contract.

## Uncertainties

### Inferred intent

The parent, left, and right offsets and traversal pattern suggest an in-order range-counting operation over a binary tree, potentially with a header or sentinel record.

No tree invariants, sentinel construction rules, or endpoint-reachability guarantees are supplied. Those interpretations must not replace the explicit branch behavior.

### documented_only: arithmetic and address model

The representation uses integer-valued addresses and four-byte slots. The exact machine ABI, address arithmetic rules, alignment requirements, byte order, and integer-overflow behavior are not established.

In particular, ordinary signed arithmetic does not establish defined wraparound on overflow. No saturation or overflow reporting appears.

### documented_only: repeated reads and external mutation

The supplied representation contains repeated reads of right and left links. It does not establish whether the original executable performs separate physical loads for each represented read, whether storage is volatile, or what concurrent modification rules apply.

The logical access sequence is described above. Instruction-level memory ordering and concurrency guarantees remain unknown.

### documented_only: termination and malformed structures

The function has no structural validation. Whether a particular sentinel topology terminates, whether a cycle repeats visits, and which invalid access occurs first depend on the concrete links and any aliasing.

No universal behavior for malformed trees is established.

## Evidence

Established static evidence consists of:

- One initial-position read before traversal.
- A freshly read stopping value at every visit comparison.
- Immediate return on equality, before counter or link access for that visit.
- One counter increment before link traversal on every non-stopping visit.
- Four-byte link slots at offsets +4, +8, and +12.
- Right-child descent followed by left-link descent when the tested right link is nonzero.
- Parent ascent when the tested right link is zero.
- A final comparison that can retain the working position instead of selecting its parent candidate.
- No explicit calls, allocation, release, structural validation, error return, or input-position writeback.

The binary-tree purpose is inferred from those relationships. Platform behavior, sentinel invariants, and overflow semantics remain documented unknowns.

No runtime tests have been performed.
