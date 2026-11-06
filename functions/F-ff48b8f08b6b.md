<!-- Trusted provenance: F-ff48b8f08b6b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ff48b8f08b6b

## Purpose

F-ff48b8f08b6b performs an unconditional operation on a destination slot, then chooses further processing by comparing a context field against two global comparison words.

The established outcomes are:

- Equality with the first comparison word selects the second entry of a previously captured table.
- Otherwise, equality with the second comparison word selects that table’s third entry.
- Either selected entry is passed to a processing dependency and then to a destination-transformation dependency.
- If neither comparison succeeds, a fixed fallback-data address is processed. A nonzero processing result requests a destination transformation; a zero result leads to one of two direct emptying-like paths.

Initialization, insertion, replacement, and emptying are inferred roles. The contract establishes the accesses, argument relationships, conditional calls, direct writes, and returns described below. It does not establish the dependencies’ content transformations or ownership policies.

The table-pointer global is read twice before the initial call, for different purposes. The comparison words are read after that call and are not established to be immutable. These distinctions affect dispatch and selected data.

## Interface

### Inputs and result

The function receives two pointer-like inputs in the supplied 32-bit representation:

1. **Context address.** A 32-bit field at byte offset +8 supplies the selector for each comparison.
2. **Destination-slot address.** A writable 32-bit slot holds a data pointer. Dependencies receive the slot address, not merely its current contents.

The slot address and its contained data pointer are distinct. A dependency may change the slot, its pointed-to storage, or relevant globals; preservation is not established.

The result is pointer-like:

- A path calling F-5aa61f6c87d1 returns that dependency’s result without checking it.
- Either zero-result fallback path returns the original destination-slot address.

There is no separate locally constructed status code. Equality between the returned value and the destination-slot address is not guaranteed on transformation paths.

### Widths and layouts

All offsets below are byte offsets.

| Storage | Location | Width | Established use |
|---|---|---:|---|
| Context | +8 | 32 bits | Selector read for each reached comparison |
| Destination slot | 0 | 32 bits | Data-pointer load or replacement |
| Storage preceding a data pointer | −4 | 32 bits | Length-like argument or direct zero write |
| Storage preceding a data pointer | −12 | 32 bits | Zero/nonzero state test |
| Data storage | 0 | 8 bits | Direct zero-byte write on one path |
| Table-pointer global | Its own location | 32 bits | Two separate pointer reads before initialization |
| Table reached through the second pointer read | 0 | 32 bits | Initial-operation argument |
| Table reached through the first pointer read | +4 or +8 | 32 bits | Selected-operation argument |
| Comparison-data region | 0 and +4 | 32 bits each | First and second comparison words |
| Replacement-pointer global | Its own location | 32 bits | Value installed after F-8a5aadac202f |

The fallback-data address is eight bytes after the beginning of the comparison-data region. Its contents and extent are not established.

The −4 word is called the **length word** because it is forwarded in a position/count-like role and cleared on one path. This does not establish its units, signedness, or valid range. The −12 word is called the **state word** solely because it controls the emptying-like branch; a reference-count interpretation is unproven.

The supplied data associates the first two comparison words with the byte groups “SlaC” and “ClaC.” These descriptions identify their apparent initial contents, not immutable selector constants. The comparisons use the 32-bit word values actually read. No endian-independent numeric encoding is established here.

### Preconditions

Normal completion requires every reached direct access to address valid storage of the required width. This includes negative-offset metadata and the table entries reached through each separately observed table pointer.

No local null, alignment, bounds, metadata, or pointer-validity checks are shown. No local non-aliasing requirement is established. Aliasing can make an earlier call or write affect a later read.

The two observed table pointers need not be equal under external modification. Validity of one does not establish validity of the other. In particular, recognized-selector processing requires the table captured by the first read to remain accessible when its selected entry is read.

These are access requirements, not guarantees that unusual inputs will produce a defined error.

## Behavior

### 1. Read the table-pointer global twice and perform the initial operation

The function first reads the table-pointer global and retains that pointer for possible recognized-selector processing.

It then reads the same global again. Through this second observed pointer, it reads the first 32-bit table entry. It calls F-0fdd13c3e50f with:

- The destination-slot address.
- The entry obtained through the second pointer read.

The dependency’s result is ignored.

The two pointer reads have different uses:

- **First observation:** retained for a later read at table offset +4 or +8.
- **Second observation:** used immediately to obtain the initial-operation argument at table offset 0.

Equality between those observations is not established under external modification. The initial argument therefore need not come from the table retained for later selection.

Both pointer reads and the initial entry read precede F-0fdd13c3e50f. Selector comparisons and selected-entry reads follow its return. A change to the table-pointer global during that call does not redirect later selected-entry access; a change to the retained table’s contents can affect the entry subsequently read.

### 2. Compare against the current global comparison words

After the initial operation returns, the function evaluates the first comparison using:

- A 32-bit selector read from context offset +8.
- A 32-bit read of the first global comparison word.

If those values are equal, the function selects the entry at offset +4 in the table retained from the first table-pointer observation.

If they are unequal, the function evaluates a second comparison using:

- A fresh 32-bit selector read from context offset +8.
- A 32-bit read of the second global comparison word.

Equality in this second comparison selects the entry at offset +8 in the retained table. Inequality takes the fallback path.

The second selector and comparison-word reads occur only if the first comparison fails. There is no intervening dependency call between the comparisons. The supplied representation does not establish the evaluation order of the two memory operands within either individual comparison.

Initialization is not assumed to preserve the comparison-data region. Dispatch therefore depends on the values read after F-0fdd13c3e50f returns, even if they differ from the apparent initial “SlaC” and “ClaC” contents.

The selector is not cached across comparisons. If it changes, the second comparison uses its second observed value. Matching the first comparison word only on the second selector read does not select the first entry; at that point only equality with the second comparison word controls dispatch.

### 3. Process a selected table entry

Once either comparison selects an entry, the function:

1. Reads that 32-bit entry from the retained table.
2. Reads the data pointer from the destination slot.
3. Reads the 32-bit length word four bytes before that data pointer.
4. Calls F-1cb585febd84 with the selected entry.
5. Disregards that dependency’s result.
6. Calls F-5aa61f6c87d1 with the destination-slot address, the captured length word, a 32-bit zero, and the selected entry.
7. Returns the transformation dependency’s result.

Both the selected entry and length are captured before F-1cb585febd84. Neither is refreshed afterward. Mutations made by that dependency do not change the values forwarded as those arguments.

The numeric argument pattern is consistent with insertion at an existing logical end while removing zero units. Actual insertion semantics, units, and content handling remain dependent on F-5aa61f6c87d1.

### 4. Process fallback data

If both comparisons fail, the function calls F-1cb585febd84 with the fixed fallback-data address.

After the call returns, it reads the current data pointer from the destination slot. This slot read occurs for either zero or nonzero dependency results.

The returned result controls whether the function requests a transformation or examines the state word. Only zero versus nonzero matters.

### 5. Nonzero fallback result

For a nonzero result, the function reads the 32-bit length word four bytes before the post-call data pointer. It then calls F-5aa61f6c87d1 with:

- The destination-slot address.
- A 32-bit zero.
- The length just read.
- The fixed fallback-data address.

It returns the dependency’s result without inspection.

Unlike selected-entry processing, fallback processing reads the length after F-1cb585febd84 returns. Changes to the slot or metadata during that call can therefore affect the forwarded length.

The argument pattern is consistent with replacing a span beginning at zero and covering the observed length. That interpretation is not an established dependency contract. This branch does not directly read the state word.

### 6. Zero fallback result and zero state word

For a zero processing result, the function reads the 32-bit state word twelve bytes before the post-call data pointer.

If the state word is zero, the function:

1. Writes a 32-bit zero to the length word four bytes before that captured pointer.
2. Reads the data pointer from the destination slot again.
3. Writes one zero byte at the reread pointer.
4. Returns the original destination-slot address.

The length clear precedes the fresh slot read and byte clear. The byte-write address is not necessarily the pointer used for the metadata accesses.

With ordinary independent storage and no external modification, this clears the length and first byte of the same buffer. Under aliasing or external modification, those writes can concern different associated storage. There is no direct destination-slot assignment on this branch.

### 7. Zero fallback result and nonzero state word

If the state word is nonzero, the function:

1. Calls F-8a5aadac202f with the destination-slot address.
2. Ignores its result.
3. Reads the replacement-pointer global after the call returns.
4. Stores that 32-bit value into the destination slot.
5. Returns the original destination-slot address.

All nonzero state-word values take this branch; there is no positive-versus-negative distinction.

A dependency change to the replacement-pointer global affects the value installed. There is no direct length clear or data-byte clear on this branch.

## Side effects

Direct writes occur only on the zero-result fallback paths:

- Zero state: a 32-bit metadata clear, followed by an 8-bit data clear using a freshly read destination pointer.
- Nonzero state: a 32-bit slot replacement after F-8a5aadac202f.

Other mutations are dependency effects. The function directly performs no visible allocation or deallocation.

Ownership is unresolved. In particular, the source does not establish whether:

- The initial operation acquires, releases, copies, or shares storage.
- F-1cb585febd84 mutates data or globals.
- F-5aa61f6c87d1 reallocates or releases destination storage.
- F-8a5aadac202f releases, detaches, or decrements anything.
- The replacement pointer identifies owned, shared, static, or writable storage.

There is no local recovery from a dependency that does not return normally. Results from the initial operation and F-8a5aadac202f are ignored. Selected-entry processing ignores F-1cb585febd84’s result. Transformation results, including any null-like result, are propagated unchanged.

Local address calculations use fixed byte offsets. The length word is forwarded without a local increment, sum, capacity calculation, or overflow check. The supplied 32-bit representation does not establish safe behavior for invalid address arithmetic or define arithmetic performed inside dependencies.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-0fdd13c3e50f | Called unconditionally with the destination-slot address and the first entry reached through the second table-pointer observation; result ignored. |
| F-1cb585febd84 | Receives a selected table entry or the fallback-data address. Its result is ignored for selected entries and tested for zero/nonzero on fallback. |
| F-5aa61f6c87d1 | Receives the destination-slot address, two 32-bit numeric arguments, and selected or fallback data. Its result becomes the function result. |
| F-8a5aadac202f | Receives the destination-slot address on zero-result fallback with nonzero state; result ignored before a fresh global-pointer read and slot assignment. |

No dependency is assumed to preserve globals, memory, ownership, or destination identity. Captured values remain the arguments described above; later reads observe memory after the relevant calls.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

### Different table-pointer observations

Suppose the first table-pointer read observes table A, while the second observes table B.

- F-0fdd13c3e50f receives table B’s entry at offset 0.
- If the first comparison later succeeds, selected-entry processing uses table A’s entry at offset +4.
- If only the second comparison succeeds, it uses table A’s entry at offset +8.
- No equality between the tables or entries is required.

### Comparison storage changed by initialization

Suppose F-0fdd13c3e50f changes the first comparison word to a value equal to the selector subsequently read.

- The first comparison succeeds.
- The retained table’s second entry is selected.
- The second selector and second comparison word are not read for dispatch.
- The original “SlaC” description does not override the changed word.

Conversely, a selector matching the apparent original first word does not guarantee first-branch selection if the word actually read has changed.

### Second comparison and changing selector

Suppose the first observed selector differs from the first observed comparison word, and the second observed selector equals the second observed comparison word.

- The third retained-table entry is selected.
- The first selector value is not reused.
- If both comparison words have equal values and the first selector already equals that value, the first branch wins instead.

### Selected processing mutates the destination

Suppose the selected-entry path captures length 5, then F-1cb585febd84 changes the destination length to 12 and returns zero.

- F-5aa61f6c87d1 still receives the slot address, 5, zero, and the captured selected entry.
- The zero result does not suppress transformation.
- The transformation result is returned directly.

### Nonzero fallback processing

Suppose both comparisons fail, F-1cb585febd84 returns nonzero, and the post-call destination has length 9.

- The transformation arguments are the slot address, zero, 9, and the fallback-data address.
- The state word is not directly read.
- Any nonzero processing result takes this path, including a value represented as negative.

### Zero-result fallback writes

Suppose the fallback result and state word are both zero.

- The length word associated with the captured pointer is cleared first.
- The slot is reread afterward to obtain the byte-write address.
- If aliasing makes the length clear change the slot, the byte clear follows the changed pointer.
- The function returns the original slot address.

### Nonzero state and replacement mutation

Suppose fallback processing returns zero and the state word is nonzero. If F-8a5aadac202f changes the replacement-pointer global, the subsequently read changed value is installed in the destination slot. No direct byte clear is performed.

### Zero length

A captured zero length does not bypass transformation. Both selected-entry processing and nonzero-result fallback processing still call F-5aa61f6c87d1 with zero-valued middle arguments.

## Uncertainties

The following remain **documented_only**:

- Concrete object types, full calling convention, and pointer-validity requirements beyond the visible accesses.
- Dependency semantics, allocation, release, mutation, failure, exception, and termination behavior.
- Length units, signedness, valid range, and the interpretation of the transformation’s numeric arguments.
- State-word meaning and any relationship to sharing or ownership.
- Fallback contents, encoding, termination, and extent.
- Whether global tables or comparison words are writable in the actual execution environment.
- Synchronization, thread safety, and legality of concurrent or externally driven changes.
- Operand-read order within each comparison.
- Whether the supplied representation preserves every machine-level access relevant to volatile or externally modified storage.

External-modification examples explain consequences of the represented reads. They do not establish a supported concurrency model.

## Evidence

This contract uses static inspection of the supplied function representation and opaque dependency mapping. Embedded comments, labels, the prior draft, and reviewer feedback are not independent authority.

The representation supports the two distinct table-pointer reads, post-initialization comparison-word accesses, conditional second comparison, retained selected values, metadata offsets and widths, dependency arguments, direct writes, and return paths described above.

Initialize, append, replace, and empty are inferred intent, not substitutes for those established relationships. No implementation access, runtime execution, or tests were performed. The acceptance examples are conditional review criteria, not empirical results.
