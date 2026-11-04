<!-- Trusted provenance: F-963cca84faed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-963cca84faed

## Purpose

F-963cca84faed selects one of two paths for populating a caller-supplied destination region. Selection depends on whether a 32-bit field reached through the first argument contains the all-ones value, interpreted as signed −1.

- **Sentinel path:** invoke F-46f0b3cb316e with destination-relative and shared-source-relative addresses, then transfer one additional 32-bit word from the shared source.
- **Non-sentinel path:** invoke an indirectly selected callback with access to a temporary argument record, invoke F-46f0b3cb316e using part of that record as its source argument, transfer the record’s final word to the destination, and invoke F-ee5919ca4be6 on the record.

Both paths return a 64-bit value assembled from the first two words of the temporary record.

This is a static behavioral contract. The supplied evidence establishes control flow, address calculations, explicit accesses, and call ordering. It does not establish the semantic contents copied or constructed by either named dependency, nor the ownership conventions of the callback or dependencies.

## Interface

### Arguments

The function accepts four machine-level arguments:

| Argument | Width and interpretation | Established use |
|---|---|---|
| Object address | 32-bit address of a word-addressable region | Used to locate the branch selector and, on the non-sentinel path, an indirect callback |
| Destination address | 32-bit integer used as an address | Supplies the base for dependency arguments and an explicit 32-bit write |
| Third input word | 32 bits, otherwise uninterpreted | Initializes the third word of a temporary record |
| Fourth input word | 32 bits, otherwise uninterpreted | Initializes the fourth word of that record |

The evidence uses a 32-bit address model and four-byte words. It does not establish a portable interface for wider pointers.

### Temporary record

Before selecting a path, the function initializes four consecutive 32-bit temporary words:

| Byte offset from record start | Initial contents |
|---|---|
| `0x00` | Object address |
| `0x04` | Destination address |
| `0x08` | Third input word |
| `0x0C` | Fourth input word |

The non-sentinel path passes the address of the first word to the indirect callback and to F-ee5919ca4be6. It passes the address of the second word to F-46f0b3cb316e.

These address-taking operations make mutations to the temporary words relevant to later behavior. In particular, the final return value must not be specified simply as the original first two arguments.

### Object-side layout

Two distinct pointer chains are used:

1. The 32-bit word at object offset `0x40` is treated as an address. The 32-bit word at offset `0x10` from that address is the branch selector.
2. On the non-sentinel path, the first 32-bit word of the object is treated as a dispatch-table address. The entry at dispatch-table byte offset `0x138` is treated as a callable address.

The evidence does not establish the size, type, or meaning of other object fields.

### Destination-side layout

The function explicitly uses these destination-relative addresses:

- `destination + 0x04`: first argument to F-46f0b3cb316e on either path.
- `destination + 0x0C`: location of an explicit four-byte write on either path.

The region required by F-46f0b3cb316e is unknown. Its destination argument being offset by four bytes does not prove that it accesses only one word or that the overall destination has any particular size.

### Return value

The return is 64 bits:

- Bits 31–0 contain the final bit pattern of temporary-record word `0x00`.
- Bits 63–32 contain the final bit pattern of temporary-record word `0x04`.

This is bit concatenation, not signed arithmetic. A sign bit in either component does not cause sign extension into the other component.

The result describes numerical high and low halves. It does not establish an in-memory byte order for a stored 64-bit result.

## Behavior

### Common entry and branch selection

The function first captures the address held in a shared address-valued cell. This captured address is subsequently used only on the sentinel path, but the capture precedes branch selection.

It then initializes the four-word temporary record from the four incoming arguments.

Next, it obtains the branch selector by:

1. Reading the address-valued word at object offset `0x40`.
2. Reading a 32-bit word at offset `0x10` from that address.
3. Comparing that word with signed −1, equivalently the bit pattern `0xFFFFFFFF`.

Only that exact bit pattern selects the sentinel path. Zero, other negative signed values, and every other 32-bit pattern select the non-sentinel path.

There is no explicit validation of the object address, the intermediate address, or the selector location.

### Sentinel path

The function invokes F-46f0b3cb316e with two arguments:

1. The incoming destination address plus `0x04`.
2. The address currently held in the shared address-valued cell, plus `0x04`.

The supplied evidence shows a shared-cell reference for this second argument, separately from the shared address captured at entry. Under ordinary stable-memory conditions they identify the same base. The contract does not assume that a mutable shared cell remains unchanged merely because no explicit assignment to it appears in this function.

After F-46f0b3cb316e returns, the function:

- Reads a 32-bit word at offset `0x0C` from the address captured at entry.
- Writes that word to the incoming destination address plus `0x0C`.
- Returns the concatenation of the first two temporary-record words.

The additional source-word read occurs **after** the dependency call. It is not an entry-time snapshot of that word. Consequently, mutations performed by F-46f0b3cb316e can affect the transferred word if they reach that source location.

The source base for this final transfer is the entry-captured address. A dependency changing the shared address-valued cell does not, by itself, redirect that transfer to the newly published base.

This path does not explicitly invoke the indirect callback or F-ee5919ca4be6.

### Non-sentinel path

The function reads the object’s dispatch-table address and obtains the callable entry at byte offset `0x138`.

It invokes that entry with:

1. The address of temporary-record word `0x00`.
2. The incoming object address.

The second callback argument is the object address supplied on entry, not a replacement value that might subsequently appear in the temporary record.

The callback’s return value is not used by this function. Its effects through memory remain relevant. The visible interface permits the callback to access the record through the first argument; the extent of its authorized access and any additional effects are not supplied.

After the callback returns, the function invokes F-46f0b3cb316e with:

1. The incoming destination address plus `0x04`.
2. The address of temporary-record word `0x04`.

The first argument continues to use the incoming destination. Changing temporary-record word `0x04` does not redirect that argument. Instead, such a change alters the contents beginning at the source address supplied as the second argument.

After F-46f0b3cb316e returns, the function reads temporary-record word `0x0C` and writes its current 32-bit value to the incoming destination plus `0x0C`.

It then invokes F-ee5919ca4be6 with the address of temporary-record word `0x00`.

Finally, it assembles and returns the 64-bit result from the first two temporary-record words, as they stand **after** F-ee5919ca4be6 returns.

This ordering has two important consequences:

- The explicit destination-word write uses the record’s fourth word after the callback and F-46f0b3cb316e have run, but before F-ee5919ca4be6 runs.
- The return uses the record’s first two words after all three calls have run. The evidence does not justify assuming that F-ee5919ca4be6 preserves them.

### Arithmetic and access preconditions

The offsets are byte offsets except for the initial object indexing that resolves to byte offset `0x40`.

Address computations occur in the supplied 32-bit model. No widening, overflow check, bounds check, alignment repair, or null check is visible. Valid use therefore requires that each executed address calculation identify a location suitable for the ensuing access or call.

The function does not define a recovery result for invalid pointers, address-space wraparound, unreadable fields, unwritable destinations, or invalid callback targets. The platform’s behavior for such operations is outside the supplied evidence.

## Side effects

### Explicit effects

On either normally completing path, the function explicitly performs one four-byte write at the incoming destination plus `0x0C`.

The sentinel path obtains the value from the entry-captured shared-source base plus `0x0C`, after F-46f0b3cb316e returns.

The non-sentinel path obtains the value from temporary-record word `0x0C`, after the callback and F-46f0b3cb316e return.

No explicit write to destination offsets `0x00`, `0x04`, or `0x08` is established outside the dependency call. In particular, the evidence does not support describing the function as copying a complete fixed-size object.

### Dependency-mediated effects

F-46f0b3cb316e may modify memory or other state according to its own contract, which is unavailable here. Its first argument identifies a destination-relative location, but its write extent is not established.

The callback and F-ee5919ca4be6 receive the temporary-record address. Their effects may therefore influence subsequent reads of that record. Neither callee is assumed to preserve globals, pointed-to objects, or record words.

The function contains no explicit assignment to the shared address-valued cell. That does not establish preservation across calls.

### Ownership and lifetime

The temporary record has invocation-local lifetime. The evidence shows its address being passed to callees but does not establish permission for a callee to retain or use that address after this function returns.

No allocation or deallocation is explicit in this function. The name or position of F-ee5919ca4be6 is insufficient evidence that it destroys an object or releases memory.

Ownership of the object, destination, shared-source storage, and any values placed in the temporary record is unspecified. Any allocation, release, reference-counting, or ownership transfer performed by a callee is **documented_only**.

### Completion and failures

There is no explicit error status, error branch, retry, or rollback.

Normal return requires each selected call to return normally. If a dependency does not return normally, the supplied evidence does not establish further execution.

In particular, on the non-sentinel path the destination word is written before F-ee5919ca4be6 is invoked. Failure or nonlocal exit from that final call does not imply rollback of the earlier write.

## Dependencies

| Dependency | Invocation conditions | Visible arguments and role |
|---|---|---|
| F-46f0b3cb316e | Once on either normally progressing path | First argument is incoming destination plus `0x04`. Second argument is shared-source base plus `0x04` on the sentinel path, or temporary-record address plus `0x04` otherwise. Its return value is unused. |
| F-ee5919ca4be6 | Non-sentinel path only, after the explicit destination write | Receives the temporary-record base. Its return value is unused; subsequent return assembly reads the record. |
| Indirect callback; opaque function ID unavailable | Non-sentinel path only | Selected through dispatch-table byte offset `0x138`. Receives the temporary-record base and incoming object address. Its return value is unused. |

No unique opaque ID for the indirect target is supplied. Assigning it one of the known dependency IDs would invent an identity.

## Acceptance examples

These examples are conditional specifications, not reported test results. Each assumes valid addresses and normally returning callees unless stated otherwise.

### Exact sentinel selection

If the selector is `0xFFFFFFFF`, the shared-source path is selected. The function invokes F-46f0b3cb316e, performs the shared-source word transfer, and does not explicitly invoke either the indirect callback or F-ee5919ca4be6.

If the selector is `0xFFFFFFFE`, the non-sentinel path is selected despite that pattern representing a negative signed value.

### Return with unchanged record

Suppose the object address is `0x10203040`, the destination address is `0x50607080`, and all invoked dependencies leave the first two temporary-record words unchanged.

The returned value is `0x5060708010203040`.

This example is conditional on preservation; preservation itself is not established.

### Record destination differs from actual destination

On the non-sentinel path, suppose the callback changes temporary-record word `0x04` to another address.

The subsequent F-46f0b3cb316e call still receives the **incoming** destination plus `0x04` as its first argument. Its second argument remains the address of temporary-record word `0x04`, whose contents now hold the replacement value.

The explicit final-word write also remains directed to the incoming destination plus `0x0C`.

### Fourth-word mutation before the write

Suppose the callback changes temporary-record word `0x0C` to `0xA1B2C3D4`, and F-46f0b3cb316e leaves that word unchanged.

The function writes `0xA1B2C3D4` to destination plus `0x0C` before invoking F-ee5919ca4be6.

If F-46f0b3cb316e instead changes that record word, the later value is used.

### Final dependency affects return

Suppose F-ee5919ca4be6 changes the first two temporary-record words to `0x11223344` and `0x55667788`.

If it returns normally, the function returns `0x5566778811223344`, regardless of the original first two arguments.

### Shared-source mutation and redirection

On the sentinel path, suppose F-46f0b3cb316e changes the word at offset `0x0C` from the entry-captured source base. The explicit transfer reads the changed word.

If the dependency only replaces the shared address-valued cell with a different base, the final transfer still reads from the entry-captured base. That old storage must remain accessible for normal completion.

## Uncertainties

The following remain **documented_only**:

- Semantic types of the object, destination, shared source, and temporary-record contents.
- Exact read and write extents of F-46f0b3cb316e.
- Callback identity, dispatch-table ABI, and callback ownership rules.
- Whether F-ee5919ca4be6 performs cleanup, release, normalization, or another operation.
- Callee preservation of globals, record words, and aliased storage.
- Permitted aliasing among the object, destination, shared source, and other reachable memory.
- Alignment requirements and platform behavior for invalid accesses.
- Concurrent mutation rules and memory-ordering guarantees.
- Any meaningful status encoded in the returned pair of words.

An inferred interpretation is that the sentinel path supplies a shared default value and the non-sentinel path obtains a value through object dispatch. That interpretation is not required by the established behavior and must not replace the concrete access-and-call contract.

## Evidence

The supplied static representation establishes:

- Four 32-bit inputs and a 64-bit concatenated return.
- A four-word temporary record initialized from those inputs.
- A selector reached through object byte offset `0x40`, followed by byte offset `0x10`.
- Exact comparison with the all-ones 32-bit value.
- A shared address captured before branch selection.
- Sentinel-path dependency invocation followed by a four-byte transfer from the captured source base.
- Non-sentinel dispatch through table byte offset `0x138`.
- Non-sentinel ordering: callback, F-46f0b3cb316e, destination-word write, F-ee5919ca4be6, return assembly.
- Return assembly from record contents after the selected path’s calls.

No dependency implementation, independent ABI documentation, or runtime observation was supplied. No runtime tests have been performed.
