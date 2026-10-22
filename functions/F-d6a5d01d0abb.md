<!-- Trusted provenance: F-d6a5d01d0abb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d6a5d01d0abb

## Purpose

F-d6a5d01d0abb searches an indexed sequence by passing one four-byte value from each visited entry, together with a caller-supplied four-byte argument, to dependency F-e6db2171781a.

It returns the zero-based index of the first visited entry for which that dependency returns zero. It returns −1 when the current sequence bound excludes the next index before a zero result has been obtained.

This is a forward, short-circuiting search. The entry count and entry-storage address are read during traversal rather than captured as a single immutable snapshot. Consequently, changes made by the dependency can affect subsequent iterations.

The meaning of a zero dependency result is not established beyond its role as the acceptance condition. Equality comparison, key lookup, and predicate evaluation are possible interpretations, not established semantics.

## Interface

### Inputs

The function receives two arguments:

1. **Descriptor address.** An address-like value used to access a descriptor containing a signed count and an entry-storage address.
2. **Auxiliary value.** An opaque four-byte value passed unchanged to F-e6db2171781a on each invocation.

The supplied representation uses a 32-bit integer/address model. The descriptor fields and entry values relevant to this function have these layouts:

| Location | Width | Interpretation by this function |
|---|---:|---|
| Descriptor offset +4 bytes | 4 bytes | Signed entry bound |
| Descriptor offset +8 bytes | 4 bytes | Address of entry storage |
| Entry storage offset 20 × index | 4 bytes | Opaque value passed to the dependency |

Entries have a traversal stride of 20 bytes. This establishes the spacing between candidate values, not the meaning or ownership of a complete entry. The remaining 16 bytes in each stride are not directly read by this function.

No meaning is established for descriptor offset zero or for other descriptor storage. Byte order and alignment requirements are not specified by the supplied evidence.

### Output

The return value is a signed 32-bit integer:

- **−1:** the current bound is less than or equal to the next candidate index.
- **A nonnegative index:** the dependency returned zero for the value fetched at that index.

The return value is an index, not an entry value or address. A successful return does not guarantee that the corresponding entry remains present or unchanged when the caller receives the result.

### Preconditions

For defined execution, every memory access actually reached must be valid under the execution environment’s memory and alignment rules.

In particular:

- The descriptor count field must be readable, including for an empty sequence.
- Whenever the current bound admits an index, the descriptor’s entry-storage address field must be readable.
- The selected entry’s leading four bytes must be readable.
- Descriptor storage required after a dependency invocation must remain readable if traversal continues.
- Address and offset calculations must identify valid locations.

These are preconditions, not validations performed by the function. No explicit null-address, allocation-size, alignment, or bounds-consistency checks are established.

## Behavior

### Initial state and termination check

Traversal begins at index zero. Before any candidate is fetched, the function reads the signed count at descriptor offset +4 and compares it with the current index.

If the count is less than or equal to that index, the function returns −1 immediately.

Consequences include:

- A zero count produces −1 without reading the entry-storage address.
- A negative count also produces −1 without reading the entry-storage address.
- Neither case invokes the dependency.
- A descriptor with an unreadable count field is not made safe by an intended count of zero; the count read must still occur.

For a stable positive count, the admitted indices are zero through count minus one.

### Candidate acquisition and dependency invocation

When the bound admits the current index, the function performs the following observable sequence:

1. Reads the entry-storage address from descriptor offset +8.
2. Reads the four-byte value at that address plus 20 times the current index.
3. Invokes F-e6db2171781a with the fetched value as its first argument and the original auxiliary value as its second argument.
4. Examines the dependency’s integer result.

The candidate fetch occurs before the dependency invocation. If the dependency subsequently changes the entry or its storage, the current invocation has already received the fetched four-byte value.

No other entry fields are directly inspected. The function does not directly compare the candidate with the auxiliary value.

### Zero and nonzero dependency results

A dependency result of zero causes immediate return of the current index.

Every nonzero result, whether positive or negative, causes traversal to continue at the next index. Such results are not returned to the caller, and no nonzero value has a distinct failure or error interpretation within this function.

After a nonzero result, the index increases by one and the descriptor count is read again. After a zero result, there is no further count read, storage-address read, or candidate fetch.

Thus, if the dependency changes the count so that the current index would no longer be admitted but returns zero, the function still returns that index. Success depends on the dependency result for the already-fetched candidate, not on a post-call bounds check.

### Mutable descriptor behavior

The count is read at the start of each iteration. The entry-storage address is read separately for each admitted candidate.

Under mutations made by the dependency:

- Reducing the count can terminate traversal at the next bounds check.
- Increasing the count can admit additional indices.
- Replacing the entry-storage address can redirect later candidate reads.
- Changing an entry that has not yet been visited can change a later dependency argument.
- Changing a previously visited entry does not cause that index to be revisited.
- Reordering entries does not reset the index.

The search therefore does not provide snapshot semantics. Its successful result identifies a traversal index at the time that candidate was fetched; it does not certify a stable relationship to the descriptor’s final contents.

These observations describe the explicit access sequence. They do not establish thread safety, atomicity, synchronization, or valid behavior under concurrent unsynchronized access.

### Integer and address boundaries

The index starts at zero and advances in unit steps. The count comparison is signed, so a negative count is treated as excluding index zero rather than as a large unsigned length.

Within the represented signed 32-bit count model, index increment does not itself need to overflow. The greatest index that can pass the bounds check is one less than the greatest signed count. After an unsuccessful invocation at that index, the increment reaches the greatest signed value, which no signed count can exceed.

The 20-byte stride creates a separate arithmetic concern. A mathematical offset of 20 times the index exceeds the positive signed 32-bit range beginning at index 107,374,183. Adding an entry-storage address may encounter an address-space boundary earlier.

The supplied evidence contains no guard against these conditions. It does not establish whether out-of-range address calculations wrap, fault, or have some other execution-specific consequence. Such cases are outside the established valid-memory contract.

## Side effects

### Direct effects

The function directly reads:

- The descriptor count at each reached bounds check.
- The descriptor entry-storage address for each admitted index.
- The leading four-byte value of each visited entry.

No direct writes to descriptor storage, entry storage, or other caller-provided memory are established. No direct allocation, deallocation, reference-count adjustment, or ownership transfer is established.

### Dependency-mediated effects

F-e6db2171781a is invoked once for each visited candidate, in increasing index order, until it returns zero or the next bounds check terminates traversal.

Its side effects are unknown. The caller must not assume that it preserves:

- The descriptor or entry storage.
- Global state.
- Memory reachable through either argument.
- The lifetime of objects represented by the fetched value or auxiliary value.

The wrapper contains no cleanup path after dependency calls. If the dependency changes ownership or releases memory, this function neither documents nor compensates for that behavior. If traversal continues, later required reads must still be valid.

### Errors and abnormal completion

The only explicit unsuccessful-search return is −1. Nonzero dependency results are continuation signals, not separately propagated error codes.

No recovery behavior is established for invalid memory access, dependency non-return, abnormal dependency termination, or other execution failures. Such events are not shown as being converted to −1.

## Dependencies

### F-e6db2171781a

**Invocation inputs:**

- First argument: the four-byte value fetched from the current entry.
- Second argument: the unchanged four-byte auxiliary input.

**Result use:**

- Zero: accept the current candidate and return its index.
- Any nonzero value: advance to the next index and check the current bound again.

**Established ordering:** the current candidate is fetched before invocation; the next count read occurs only after a nonzero result and index advancement.

**Unknown contract:** argument interpretation, signedness of the opaque argument values, external reads and writes, ownership effects, allocation or release behavior, global-state changes, reentrancy, and abnormal termination behavior.

No additional function dependency is established by the supplied evidence.

## Acceptance examples

These are conditional contract examples derived from static evidence. They are not reports of executed tests.

### Empty or negative count

Given a readable descriptor count of zero or −3, the function returns −1.

It does not read the entry-storage address or invoke F-e6db2171781a. Entry-storage readability is therefore unnecessary for these executions; descriptor-count readability remains necessary.

### Acceptance at the first entry

Given a count of at least one, valid first-entry storage, and a dependency result of zero for that candidate, the function returns zero.

There is one dependency invocation and no subsequent count check.

### Acceptance after two rejections

Given a stable count of at least three and valid entry storage, suppose the dependency results at indices zero, one, and two are respectively 4, −7, and 0.

The function visits those indices in order and returns 2. The positive and negative nonzero results both mean continuation. No fourth candidate is fetched.

### Exhaustion with a stable count

Given a stable count of three and a nonzero dependency result for each visited entry, the function visits indices zero, one, and two.

It then reads the count at index three and returns −1 without fetching an entry at index three.

### Count reduction during traversal

Given an initial count of five, suppose the dependency invoked at index zero changes the count to one and returns nonzero.

The next bounds check compares index one with count one. The function returns −1 without visiting index one.

If that same invocation instead returns zero, the function returns index zero without rereading the count.

### Storage redirection during traversal

Given an admitted first candidate, suppose its dependency invocation changes the descriptor’s entry-storage address and returns nonzero.

If the next count check admits index one, the function fetches the next candidate at offset 20 from the replacement storage address. It does not retain the original storage address for that read.

### Later invalidation of a successful position

Suppose the dependency for index two removes or rearranges entries and returns zero.

The function still returns 2. This contract does not guarantee that the caller can subsequently dereference entry two or recover the same candidate from that index.

## Uncertainties

The following are **documented_only**:

- The domain meaning of the descriptor, entry values, auxiliary input, and acceptance result.
- Whether either four-byte argument represents an address, handle, key, integer, or other encoding.
- The complete descriptor and entry layouts.
- Byte order, alignment requirements, and exact address-overflow behavior.
- Whether the dependency mutates descriptor state or entry contents in actual use.
- Ownership, lifetime, and release obligations associated with either dependency argument.
- Thread-safety, reentrancy, and synchronization requirements.
- Fault behavior and recovery for invalid memory or abnormal dependency completion.

An inferred intent is “find the first matching entry.” The established behavior is narrower: return the first visited index whose dependency result is zero. No stronger definition of matching is justified.

## Evidence

The supplied static representation establishes the initial zero index, signed bounds check, descriptor-field offsets, 20-byte traversal stride, four-byte candidate fetch, dependency argument order, zero-result short circuit, nonzero continuation, and −1 termination result.

It also establishes the explicit rereads of the count and storage address, the absence of a post-success bounds check, and the absence of direct writes or cleanup operations in this function.

No runtime tests have been performed. Dynamic dependency behavior, platform-specific memory semantics, and cases outside valid arithmetic and memory preconditions remain unverified.
