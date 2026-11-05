<!-- Trusted provenance: F-23eebec8340f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-23eebec8340f

## Purpose

F-23eebec8340f conditionally resolves an input record to another record pointer. It either:

- Returns the input record unchanged when neither of two control bits is set.
- Selects a pointer from a descriptor when a computed address is null.
- Obtains a signed lookup key through one of two dependencies and searches a strided table.
- Returns a descriptor-provided fallback pointer when the search does not match.
- Returns null, optionally invoking a reporting dependency first, when the selected fallback pointer is null.

The established behavior is pointer selection and table lookup. Interpreting the control bits as particular application states, or the lookup key as a particular kind of identifier, would be inferred intent rather than established behavior.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

### Inputs

The function has three inputs:

1. **Context pointer.** A pointer to storage whose first signed integer supplies a base value used in address calculation. This storage is read only on the resolution path.
2. **Input-record pointer.** A pointer to storage interpreted as unsigned integer words. The first word contains control bits. The word at byte offset 16 supplies the descriptor address on the resolution path.
3. **Reporting control.** A signed integer tested only for equality to zero. Any nonzero value enables a reporting call if the selected fallback pointer is null. Its magnitude and sign otherwise have no effect.

### Output

The result is a pointer interpreted as pointing to unsigned integer words. It can be:

- The original input-record pointer.
- An address four bytes into a matching table entry.
- One of two pointers loaded from the descriptor.
- Null.

There is no separate status result. A non-null result is not validated before return.

### Width and representation model

The supplied representation uses four-byte words, four-byte address-bearing fields, a 24-byte table-entry stride, and signed integer comparison for the table count and lookup key. The layout below therefore describes a 32-bit address model.

The supplied text does not independently establish every underlying language or ABI rule. In particular, exact behavior for signed arithmetic overflow, invalid pointer arithmetic, or invalid memory access is not established. Acceptance examples assume valid, aligned storage and representable arithmetic unless explicitly stated otherwise.

### Input-record layout

| Byte offset | Width | Meaning |
|---|---:|---|
| 0 | 4 bytes | Unsigned control word |
| 16 | 4 bytes | Descriptor address, used only when resolution is enabled |

Other input-record fields are not inspected directly.

### Descriptor layout

| Byte offset | Width | Meaning |
|---|---:|---|
| 4 | 4 bytes | Signed displacement added to the context base |
| 12 | 4 bytes | Pointer to the first table entry |
| 16 | 4 bytes | Signed table-entry count |
| 20 | 4 bytes | Fallback pointer used after an unsuccessful search |
| 24 | 4 bytes | Pointer used when the computed address is null |

The function does not establish a meaning for other descriptor bytes.

### Table-entry layout

Each entry occupies a 24-byte stride:

| Byte offset within entry | Width | Meaning |
|---|---:|---|
| 0 | 4 bytes | Signed key compared with the dependency result |
| 4 | Not established | Start of the object returned for a matching entry |

The function does not read the remaining 20 bytes of an entry. Their type and semantic layout are unknown. Returning an address into that region does not establish that the region is an independent allocation.

### Preconditions

For a defined successful execution, all storage actually accessed on the selected path must be readable and suitably aligned.

In particular:

- The input-record control word must be readable on every invocation.
- When either control bit is set, the input record’s descriptor field, the descriptor’s displacement field, and the context base must be readable.
- A non-null computed address must support a four-byte read only when the dependency taking that word is selected.
- A positive table count requires readable keys for every entry visited before a match or termination.
- The descriptor must remain readable at the points where the function accesses its fields after dependency calls.

No null checks protect the context pointer, input-record pointer, descriptor pointer, or table pointer generally. Nullness is tested only for the computed address and the selected fallback pointer.

## Behavior

### 1. Bypass decision

The function first reads the input record’s control word and tests mask `0x300`, comprising bits `0x100` and `0x200`.

If both bits are clear, it immediately returns the input-record pointer.

On this path:

- The context is not read.
- The descriptor address is not read.
- No dependency is called.
- No table search or reporting occurs.
- The reporting control has no effect.

Other control bits do not affect this initial decision.

### 2. Descriptor selection and computed address

When either tested bit is set, the function reads the descriptor address from input-record offset 16.

It then forms an address by adding:

- The signed base value read from the context’s first word.
- The signed displacement read from descriptor offset 4.

The null test applies to the resulting address, not separately to either operand. A zero base does not by itself select the null-address path, and a nonzero base does not rule it out.

The descriptor address selected here is retained for subsequent descriptor accesses. Later mutation of the input record’s descriptor field does not, by itself, redirect those accesses.

The supplied expression does not establish the relative evaluation order of the context-base read and descriptor-displacement read. Both contribute to the address calculation before its nullness is tested.

### 3. Null computed address

If the computed address is null, the function reads the pointer at descriptor offset 24.

It does not:

- Invoke either key-producing dependency.
- Read the table pointer or count.
- Search the table.
- Use the fallback field at descriptor offset 20.

The selected pointer then follows the common fallback-result handling described below.

### 4. Non-null computed address and key selection

If the computed address is non-null, the function reads the input record’s control word again and tests bit `0x100`.

This is a distinct read from the initial mask check.

- If bit `0x100` is clear, the function reads one four-byte word from the computed address and passes that value to F-e26fb769d30e. Its signed integer result becomes the lookup key.
- If bit `0x100` is set, the function invokes F-2eb743f270d1 without explicit arguments. Its signed integer result becomes the lookup key. The computed address is not dereferenced by F-23eebec8340f on this branch.

Consequently, with stable control storage:

- `0x200` without `0x100` selects F-e26fb769d30e.
- `0x100`, whether alone or combined with `0x200`, selects F-2eb743f270d1.

The function does not establish that either dependency leaves shared state unchanged.

### 5. Table search

After the selected dependency returns, the function reads the table pointer from descriptor offset 12.

It searches entries in increasing address order, using a 24-byte stride. The first entry’s key is at the table pointer; each subsequent key is 24 bytes later.

The loop uses:

- A signed index beginning at zero.
- A signed count read from descriptor offset 16.
- A strict index-less-than-count condition.
- Signed equality between the entry key and the dependency’s result.

The count is expressed as a read at each loop-condition evaluation, rather than an explicit one-time snapshot. This does not establish concurrency safety or prescribe compiler treatment of unsynchronized external changes.

For a stable count:

- A zero or negative count visits no entries.
- A positive count permits at most that many comparisons.
- Duplicate matching keys select the earliest matching entry.

On a match, the function immediately returns the address four bytes after the matching key. It does not read a pointer from that location. The result is an interior address within the table entry.

A match bypasses both descriptor fallback fields and the reporting dependency.

### 6. Unsuccessful search

If the search terminates without a match, the function reads the pointer at descriptor offset 20.

This read occurs after the key-producing dependency and after the search. The function therefore uses the field’s value at that later access, not a value explicitly saved before the dependency call.

### 7. Common fallback-result handling

The pointer selected from descriptor offset 24 or offset 20 is tested for null.

- If non-null, it is returned unchanged.
- If null and reporting control is zero, the function returns null without calling the reporting dependency.
- If null and reporting control is nonzero, the function invokes F-4fc89ec41a6d with five integer arguments, in order: `0x0D`, `0x6E`, `0xA4`, zero, zero. If that call returns normally, F-23eebec8340f returns null.

Any result produced by the reporting dependency is ignored. The function does not reload the selected fallback pointer after reporting, retry the search, or substitute a pointer installed by that dependency.

## Side effects

### Direct effects

F-23eebec8340f contains no explicit writes to caller-provided storage, descriptor fields, table entries, or global storage.

Its direct observable activity consists of memory reads, dependency calls, and returning a pointer or null.

It performs no explicit allocation, copying, deallocation, reference-count adjustment, or ownership transfer operation.

### Dependency effects

Dependency side effects are unknown. In particular, this contract does not assume that dependencies:

- Preserve globals or other shared storage.
- Preserve descriptor or table contents.
- Preserve the lifetime of any referenced allocation.
- Avoid allocation or deallocation.
- Return normally.

The table pointer and table count are accessed after the key-producing dependency returns. The unsuccessful-search fallback is also read after that call. Mutations made by the dependency can therefore affect subsequent behavior if the resulting storage remains valid.

If a dependency invalidates storage subsequently used by this function, the supplied evidence establishes no safe recovery behavior.

### Ownership and lifetime

The function itself neither creates nor releases the storage it returns. No new ownership right is established by the return operation.

An interior table-entry result depends on the lifetime of the underlying table storage. The ownership and lifetime of descriptor-provided fallback pointers remain external requirements. The supplied evidence does not establish whether either key-producing dependency changes those lifetimes.

## Dependencies

| Opaque ID | Invocation condition | Explicit inputs | Use of result |
|---|---|---|---|
| F-e26fb769d30e | Computed address is non-null and the second control-word read has `0x100` clear | One four-byte word read from the computed address | Signed lookup key |
| F-2eb743f270d1 | Computed address is non-null and the second control-word read has `0x100` set | None | Signed lookup key |
| F-4fc89ec41a6d | Selected fallback pointer is null and reporting control is nonzero | `0x0D`, `0x6E`, `0xA4`, zero, zero | Ignored |

For any normal execution, at most one key-producing dependency is invoked. Reporting, when reached, occurs after fallback selection and its null test.

The identities, application roles, side effects, and internal failure modes of these dependencies are not established here.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Each assumes valid storage, stable values except for explicitly described dependency effects, and normal dependency return.

1. **Bypass with unrelated flags.**  
   The input control word is `0x400`. The function returns the input-record pointer. Neither context nor descriptor storage needs to be accessed on this path, and reporting control is irrelevant.

2. **Null address with a usable special fallback.**  
   Resolution is enabled, the context base and displacement sum to zero, and descriptor offset 24 contains a non-null pointer. That pointer is returned. Neither key-producing dependency is called, even if bit `0x100` is set.

3. **Null special fallback with reporting disabled.**  
   The computed address is null, descriptor offset 24 contains null, and reporting control is zero. The result is null without a reporting call.

4. **Null special fallback with reporting enabled.**  
   The computed address is null, descriptor offset 24 contains null, and reporting control is negative. F-4fc89ec41a6d receives the five specified constants. If it returns normally, the function returns null.

5. **Word-based key generation.**  
   The control word is `0x200`, and the computed address is non-null. Its first four-byte word is passed to F-e26fb769d30e. If that dependency returns key 17 and the second table entry is the first entry with key 17, the result is the table start plus 28 bytes: one 24-byte stride plus the four-byte key offset.

6. **No-argument key generation takes precedence.**  
   The control word is `0x300`, and the computed address is non-null. F-2eb743f270d1 is called; F-e26fb769d30e is not. The computed address itself is not dereferenced by this function.

7. **Duplicate keys select the first entry.**  
   The generated key is −7, and both the first and third entries contain −7. The function returns the address four bytes into the first entry. It does not inspect later entries after that match.

8. **Nonpositive count skips entry reads.**  
   The computed address is non-null, the selected key dependency returns normally, and the signed count is zero or negative. No table key is read. The pointer at descriptor offset 20 determines the result or reporting behavior.

9. **Dependency mutation affects fallback selection.**  
   A key-producing dependency changes descriptor offset 20 to a new non-null pointer and leaves the descriptor valid. If the subsequent search has no match, the function returns that new pointer.

10. **Reporting does not repair the result.**  
    The selected fallback is null. The reporting dependency changes the descriptor to contain a non-null fallback and returns normally. The function still returns null because it does not reload that field.

## Uncertainties

- **documented_only — Address arithmetic:** Exact overflow, wrapping, and invalid-address behavior is not established. The null-address branch must not be interpreted as permission to rely on overflowing arithmetic.
- **documented_only — Dependency contracts:** Side effects, ownership effects, failure behavior, and non-returning behavior are unknown.
- **documented_only — Concurrency:** Repeated reads do not establish atomicity, synchronization, or support for concurrent mutation.
- **documented_only — Returned payload:** The structure and usable extent beginning four bytes into a matching entry are unknown.
- **documented_only — Fault handling:** No explicit recovery is present for unreadable storage, invalid descriptors, or invalid table ranges. Platform-level consequences are unspecified.
- **Inferred intent:** The control bits appear to enable resolution and select a key source. The descriptor appears to provide lookup metadata and two fallback choices. Application-specific names or meanings are not established.

## Evidence

The supplied function body establishes the control masks, fixed offsets, dependency-call branches, signed lookup comparisons, 24-byte stride, interior-pointer return, two fallback selections, and conditional reporting call.

The four-byte layout and address interpretation are supported by the supplied word types and fixed-offset accesses. Broader ABI and arithmetic guarantees are not independently supplied.

No dependency implementations or external contracts were supplied. No runtime tests, fault-injection checks, or ownership observations have been performed.
