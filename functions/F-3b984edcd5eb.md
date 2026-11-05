<!-- Trusted provenance: F-3b984edcd5eb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3b984edcd5eb

## Purpose

F-3b984edcd5eb conditionally performs a callback-and-dispatch sequence associated with a receiver, using a temporary four-word record. It returns the record’s first two words as a 64-bit result.

The sequence is gated by a pointer in the receiver and by the nonzero result of an indirect operation. If the receiver’s gating pointer is initially nonnull, the function clears a separate receiver field before returning normally, including when the indirect operation declines further processing.

The interpretation of this sequence as event processing, notification, or state synchronization is **inferred intent**, not an established semantic contract. The supplied evidence establishes the accesses, calls, branches, and result construction described below, but does not identify the application-level meaning of the fields or record.

## Interface

### Inputs

The function has four incoming 32-bit words:

| Input | Established use |
|---|---|
| Receiver | Used as an address for receiver-relative accesses; also initializes the first temporary-record word and is passed to two direct dependencies. |
| Second word | Initializes the second temporary-record word. |
| Third word | Initializes the third temporary-record word only when the receiver’s gating pointer is initially nonnull. |
| Fourth word | Initializes the fourth temporary-record word only when the receiver’s gating pointer is initially nonnull. |

The receiver is represented as a 32-bit integer in the supplied material but is used as a memory address. Its signedness does not establish meaningful signed arithmetic semantics for the accesses.

The second, third, and fourth inputs are opaque 32-bit values. No interpretation as pointers, counts, flags, or signed values is established.

### Return value

The result is a 64-bit bit pattern formed from the first two words of the temporary record at return:

- Bits 0–31 contain the first record word.
- Bits 32–63 contain the second record word.

This is bit concatenation, not signed extension or a separately calculated success code. If neither word is changed during the call, the result is the second input in the upper half and the receiver input in the lower half.

Dependencies receive access to the temporary record on the processing path, so the returned pair is not necessarily identical to the initial input pair.

### Receiver-relative layout

The function directly uses the following receiver-relative locations:

| Offset | Access width | Role established by use |
|---|---:|---|
| `0x34` | 32 bits | Pointer to an optional secondary object. |
| `0xEC` | 32 bits | Pointer used to gate processing and resolve two indirect operations. |
| `0x118` | 32 bits | Field cleared after the initially nonnull gating-pointer path completes. |

The indirect object layouts used are:

- The object reached through receiver offset `0xEC` begins with a 32-bit dispatch-table pointer.
- That table supplies callable entries at byte offsets `0x174` and `0x170`.
- The secondary object reached through receiver offset `0x34` begins with a 32-bit dispatch-table pointer.
- The secondary object has a 32-bit flag word at byte offset `0x48`.
- Its dispatch table supplies a callable entry at byte offset `0x148`.

These offsets establish required readable or writable locations, not complete object sizes or type definitions.

### Temporary-record layout

The temporary record occupies four consecutive 32-bit words:

| Byte offset | Initial contents |
|---|---|
| `0x00` | Receiver input |
| `0x04` | Second input |
| `0x08` | Third input, initialized only on the initially nonnull gating-pointer path |
| `0x0C` | Fourth input, initialized only on that same path |

The record has automatic, call-local storage. The function passes its address to dependencies but performs no visible heap allocation.

## Behavior

### 1. Establish the return pair and inspect the gating pointer

The function first initializes the temporary record’s first two words from the receiver and second input.

It then reads the 32-bit pointer at receiver offset `0xEC`.

If that pointer is null:

- The third and fourth record words are not initialized by this function.
- No dependency is called.
- The secondary-object field is not inspected.
- The field at receiver offset `0x118` is not cleared.
- The function returns the initialized first two record words as the 64-bit result.

A null gating pointer does not make a null receiver safe: accessing the gating-pointer field already requires a valid receiver-relative read.

### 2. Initialize the complete record and invoke the gate operation

If the initial gating-pointer read is nonnull, the function initializes the record’s third and fourth words from the corresponding inputs.

It resolves an indirect operation through the object at receiver offset `0xEC`, using dispatch-table byte offset `0x174`, and invokes that operation.

The supplied call representation establishes no explicit arguments for this indirect invocation. It does not establish a portable callback signature or prove that an object pointer is passed as an implicit argument.

The operation produces a 32-bit result. Zero suppresses the processing sequence described next; any nonzero bit pattern enables it. There is no requirement that the result equal one.

If the result is zero, execution proceeds directly to the final receiver-field clear.

### 3. Obtain a word from the current gating object

When the gate result is nonzero, the function resolves another indirect operation using dispatch-table byte offset `0x170`.

The receiver’s pointer at offset `0xEC` is read again for this operation. The function does not require that it still designate the object used for the preceding gate operation.

This second indirect operation returns an opaque 32-bit word. Its domain, validity rules, and ownership meaning are unknown.

As with the gate operation, the supplied representation does not establish explicit arguments for this invocation.

### 4. Call the two direct dependencies in order

The function calls F-4cad27b84ae7 with:

1. The address of the temporary four-word record.
2. The original receiver input.
3. The opaque 32-bit word returned by the indirect operation at slot `0x170`.

It then calls F-6d16652f2d03 with:

1. The original receiver input.
2. The address of that same temporary record.

No return value from either direct dependency is used as an explicit success or failure decision by this function.

These calls may alter the record or receiver-associated state. Their bodies are unavailable, so this contract does not assume that either dependency preserves any record word, receiver field, global state, or reachable allocation.

The original receiver input remains the base used by this function for subsequent receiver-field accesses and the final clear. A dependency changing the record’s first word does not redirect those accesses.

### 5. Inspect the current secondary object

After both direct dependencies return, the function reads the pointer at receiver offset `0x34`.

If this pointer is null, it does not read a flag word or invoke the optional secondary callback.

If it is nonnull, the function reads the 32-bit word at secondary-object offset `0x48` and tests the bit represented by mask `0x00000002`.

- If that bit is clear, there is no secondary callback.
- If that bit is set, other bits do not prevent the callback.

For an eligible secondary object, the function resolves the callable entry at dispatch-table byte offset `0x148` and invokes it with two explicit arguments:

1. The secondary-object pointer obtained from receiver offset `0x34`.
2. The temporary-record address.

The callback’s return value is not used. Its possible mutations of the temporary record can affect the final 64-bit result.

The secondary-object pointer and flag are obtained after the direct dependencies execute. Consequently, changes those dependencies make to the secondary-object association or its flag can affect callback eligibility during the same invocation.

### 6. Clear the receiver field and return

On every normally completing path whose initial gating-pointer read was nonnull, the function writes a 32-bit zero to receiver offset `0x118`.

This write occurs:

- After a zero gate result, without running the processing sequence.
- After both direct dependencies when no secondary callback is eligible.
- After the secondary callback when it is eligible and returns normally.

The function then constructs its 64-bit return value from the current first two temporary-record words.

There is no visible failure-status branch around the final clear. If a dependency faults, fails to return, or transfers control elsewhere, the evidence does not establish that the clear or return occurs.

## Side effects

### Direct effects

The directly established effects are:

- Initialization of the call-local record.
- Reads of the receiver, object, dispatch-table, and flag locations required by the selected path.
- The conditional dependency invocations described above.
- A 32-bit zero write to receiver offset `0x118` when the initial gating pointer was nonnull and execution reaches that point.

No direct heap allocation, deallocation, reference-count update, or input-buffer copy is visible.

### Indirect effects and ownership

The function exposes a writable address to its temporary record. The dependencies’ actual write behavior is unknown; preservation of any record word must not be assumed.

The record’s storage belongs to this invocation and ceases to be available after return. The evidence does not establish whether dependencies retain its address or whether such retention is permitted by a broader interface. A consumer must not infer a valid post-return record lifetime.

The opaque word obtained from slot `0x170` is passed to F-4cad27b84ae7 without a visible interpretation or release operation in this function. Whether that word represents owned, borrowed, transferred, or non-memory data is **documented_only**.

Likewise, the absence of a direct release does not establish that dependencies leave allocations or ownership unchanged.

### Preconditions and invalid accesses

For defined completion, all memory and callable entries actually traversed must be valid for their respective accesses. In particular:

- The receiver-relative fields used on the selected path must be accessible.
- A nonnull gating object must provide the required dispatch-table accesses.
- After the gate operation, the current gating-object pointer must still support slot `0x170` if processing continues.
- A nonnull secondary object must support the flag read and, when eligible, the callback dispatch.
- Receiver offset `0x118` must be writable on the initially nonnull path.

The function provides no visible null recheck before resolving slot `0x170`, no validation of dispatch-table entries, and no recovery for invalid addresses.

No integer overflow check is present. Offsets are byte displacements in the represented 32-bit address model; the evidence does not justify treating address wraparound as a supported input case.

## Dependencies

| Dependency | Invocation condition | Established interaction |
|---|---|---|
| F-4cad27b84ae7 | Initial gating pointer nonnull and gate result nonzero | Receives the record address, original receiver, and opaque word from slot `0x170`. |
| F-6d16652f2d03 | After F-4cad27b84ae7 returns on the processing path | Receives the original receiver and record address. |
| Unresolved indirect target at slot `0x174` | Initial gating pointer nonnull | Returns the 32-bit zero/nonzero gate result. Explicit arguments are not established. |
| Unresolved indirect target at slot `0x170` | Gate result nonzero | Returns the opaque 32-bit word supplied to F-4cad27b84ae7. Explicit arguments are not established. |
| Unresolved indirect target at slot `0x148` | Current secondary pointer nonnull and flag bit `0x2` set | Receives the secondary pointer and record address. Return value unused. |

The three indirect targets have no supplied opaque function IDs. Their dispatch slots identify their roles here, not their identities.

## Acceptance examples

These are conditional contract examples derived from static evidence, not executed tests.

1. **Null gating pointer.**  
   Given a valid receiver whose field at `0xEC` is null, the function calls no dependency and leaves the field at `0x118` untouched. With receiver word `0x10002000` and second input `0x89ABCDEF`, the return is `0x89ABCDEF10002000`.

2. **Gate operation declines processing.**  
   Given a nonnull valid gating object whose slot `0x174` returns zero, the function initializes all four record words, does not invoke slot `0x170` or either direct dependency, and writes zero to receiver offset `0x118`. Assuming no indirect alteration of the first two record words, the return retains the initial pair.

3. **Noncanonical true result.**  
   A gate result of `0x80000000` enables processing just as a result of one does. The contract tests nonzero, not a particular Boolean encoding.

4. **Gating object changes during the first operation.**  
   If the gate operation changes receiver offset `0xEC` to another valid object and returns nonzero, slot `0x170` is resolved through the newly current object. The function does not deliberately preserve the original gating object for this lookup.

5. **Direct dependency changes the secondary association.**  
   If either direct dependency installs a valid secondary object at receiver offset `0x34`, the later eligibility check uses that association. If its flag word is `0x00000002`, the secondary callback runs.

6. **Unrelated flag bits.**  
   A secondary flag word of `0xFFFFFFFF` enables the callback. A flag word of `0x00000004` does not. Only bit `0x2` determines this branch.

7. **Record mutation affects the result.**  
   If processing leaves record word zero equal to `0x01234567` and record word one equal to `0x89ABCDEF`, the returned value is `0x89ABCDEF01234567`, regardless of the initial receiver and second input. The final receiver-field clear still uses the original receiver.

8. **A dependency writes the field later cleared.**  
   If a normally returning dependency writes a nonzero value to original receiver offset `0x118`, this function subsequently overwrites it with zero on the initially nonnull path.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of the receiver fields, record words, gate result, and retrieved opaque word.
- Exact calling conventions and implicit arguments for the first two indirect operations.
- Internal behavior, memory ownership rules, and global-state effects of every dependency.
- Whether dependencies retain the temporary-record pointer.
- Synchronization, reentrancy, and concurrent-mutation guarantees.
- Exception, fault, or nonlocal-control-transfer behavior.
- Any broader validity requirements beyond the accesses visible here.

The evidence supports ordered reloads around dependency calls. It does not establish atomicity, memory barriers, or a coherent snapshot across those reads.

## Evidence

This contract is based solely on the supplied static representation of F-3b984edcd5eb and the supplied mapping for its two direct dependencies.

Established static evidence includes the four 32-bit inputs, receiver-relative offsets, four-word temporary layout, indirect-dispatch offsets, zero/nonzero gating, dependency order, secondary flag mask, conditional final zero write, and 64-bit concatenated return.

Descriptive terms such as “gating object,” “secondary object,” and “temporary record” identify observable roles without asserting application-specific types.

No runtime tests have been performed. The acceptance examples state consequences of the visible control flow under their stated preconditions; they are not observations from execution.
