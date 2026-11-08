<!-- Trusted provenance: F-0ebf7d7933df; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0ebf7d7933df

## Purpose

F-0ebf7d7933df performs an ordered operation on a caller-supplied object. Depending on a 32-bit mode and dependency results, it invokes object-associated callbacks, clears a flag byte, and replaces either or both of two destination bytes. It also performs an unconditional dependency pair and a final dependency call.

Every normal return produces the 32-bit value 1. This value does not report whether either destination was updated.

The established contract concerns accesses, conversions, call ordering, temporary-storage exposure, and conditional writes. Terms such as “query,” “descriptor,” and “finalization” identify observed roles, not verified domain semantics.

## Interface

### Arguments and result

| Argument | Representation | Established use |
|---|---|---|
| Object | 32-bit pointer to object storage | Supplies dispatch information and object fields; passed to dependencies |
| Mode | 32-bit integer | Tested against zero, copied into addressable temporary storage, and passed unchanged to the final dependency |
| Auxiliary A | Opaque 32-bit value | Copied into an addressable local word |
| Auxiliary B | Opaque 32-bit value | Initializes the first word of a 16-byte temporary region |

The function does not directly dereference either auxiliary value. Dependencies receive addresses of their local copies and may interpret or modify that storage.

The result is a 32-bit value equal to 1 whenever execution reaches the normal return.

### Object and dispatch layouts

Offsets below are byte offsets.

| Location | Source-level access | Role |
|---|---|---|
| Object + 0x00 | 32-bit word read | Dispatch-table address for the initial callback |
| Object + 0xB0 | 8-bit write | First optional destination |
| Object + 0xB1 | 8-bit write | Second optional destination |
| Object + 0xD0 | 32-bit word read, followed by conversion to an 8-bit character value | Controls flag-handling branch |
| Object + 0xD0 | 8-bit write | Clears the flag on the continuing flag-handling path |

The flag test is **not represented as a byte-sized dereference**. It accesses a 32-bit object element beginning at 0xD0 and converts that value to a character before comparing it with zero. The supplied representation does not establish the physical machine-load width.

The initial callback is selected through a 32-bit callable entry at dispatch-table offset 0x118. A separately obtained object supplies another dispatch-table address, with a callable entry at offset 0x90.

### Preconditions and arithmetic

There is no explicit validation of the object, global address values, descriptor addresses, or indirect-call targets. Defined normal execution requires validity for every access and call actually reached.

Under literal source-level access requirements, the flag-test word extends through object byte 0xD3. Storage ending at 0xD0 is insufficient for that access. This is distinct from the later clear, which explicitly writes only byte 0xD0. Alignment and object-access requirements also apply; the representation does not establish support for otherwise invalid or unaligned accesses.

Address calculations use the represented 32-bit pointer/address model. The contract does not establish safe wrapping, bounds recovery, or meaningful behavior for invalid address arithmetic.

## Behavior

### Initial callback and subsequent flag test

A nonzero original mode enables the initial callback. The function reads the object’s dispatch-table address, selects the entry at offset 0x118, and calls it with the original object pointer as an explicit argument. No returned value is consumed.

A zero original mode skips this callback.

After that callback returns, or immediately when it is skipped, the function accesses the 32-bit word at object offset 0xD0. It converts that word to a character value and tests the converted value against zero.

Consequently:

- Changes made by the initial callback can affect the flag test.
- The branch is not a test of the entire word for nonzero.
- It is also not evidence of a physical single-byte load.
- Character signedness and out-of-range conversion details are not established by this representation. The controlling condition is whether the conversion produces zero.

### Conditional flag clear and secondary callback

If the converted character value is zero, the flag-handling stage performs no calls or explicit flag write.

If it is nonzero, the function calls F-6a21f4f07f67 without explicit arguments and checks its returned pointer.

A null result leads to F-a817d8d58c10. That dependency is marked nonreturning in the supplied representation. Its termination mechanism is unknown, and no normal recovery path is shown. If it unexpectedly returned, the displayed continuation would clear the flag and then attempt dispatch through the null object.

On the continuing path with a valid nonnull result:

1. The function explicitly writes zero to object byte 0xD0.
2. It reads the returned object’s first word as a dispatch-table address.
3. It invokes the entry at table offset 0x90.

The flag clear precedes the secondary dispatch-table access and call. It does not explicitly overwrite bytes 0xD1 through 0xD3.

The secondary indirect call has no explicit arguments in the supplied representation. Its full calling convention, including any implicit receiver, is unresolved.

### Unconditional calls and saved descriptor base

After the flag-handling stage completes normally, the function calls F-bb50a072e902 with a 32-bit zero argument. It then calls F-9b5a149ab944 with the preceding result and another 32-bit zero argument. The latter call’s result is unused.

Only after both calls return does the function read a global 32-bit address value and retain it as the **saved descriptor base**.

This global read occurs even when the original mode is zero. On the zero-mode path, no descriptor is subsequently read through the saved address.

### Addressable temporary storage

The function next initializes local storage with copies of the original object pointer, original mode, auxiliary A, and auxiliary B.

The supplied local layout places:

- The copied object-pointer word before the copied-mode word.
- The auxiliary-A word after the copied-mode word.
- A 16-byte temporary region after those words.

Only the first word of the 16-byte region is explicitly initialized, using auxiliary B. The remaining twelve bytes have no shown initialization before the first dependency call receiving that region.

Two temporary starting addresses are reused:

- The start of the 16-byte region.
- The address of the copied-mode word.

F-9b9c1c7483f4 also receives either the address of the auxiliary-A word or the address of the copied object-pointer word. Its permitted or actual write extent is unknown. The represented adjacency identifies potentially relevant neighboring storage; it does not establish that all writes spanning those locations are valid.

The original object and mode arguments remain the values used by the surrounding control flow and final call. Modifying an exposed local copy is not represented as replacing either original argument.

### First optional destination

The entire optional-update block is enabled only by a nonzero original mode.

For the first query, the function rereads the descriptor-base global rather than using the saved descriptor base. It reads a 32-bit descriptor at offset 0x0C from this freshly obtained base.

F-9b9c1c7483f4 receives:

- The start of the 16-byte temporary region.
- That descriptor.
- The address of the auxiliary-A local word.

After this call returns, the function reads a second global 32-bit value and retains it as the **saved context value**. The supplied representation then reads that context global again for the first argument to F-8094c3d72052. Its other argument is the result of F-9b9c1c7483f4.

There is no intervening explicit call between those two context-global reads, but they are distinct source-level reads. Without an external stability assumption, their equality is not an established guarantee.

After F-8094c3d72052 returns, the function calls F-8a5aadac202f with the 16-byte region. Only then does it test the previously returned 32-bit query result against zero.

A zero result skips the first extraction and explicit destination write.

For a nonzero result, the function:

- Reads the descriptor again at offset 0x0C, this time from the **saved descriptor base**.
- Calls F-9b9c1c7483f4 with the copied-mode starting address, that descriptor, and the address of the copied object-pointer word.
- Calls F-cf946f91e683 with the saved context value and the preceding result.
- Writes the returned byte to object offset 0xB0.
- Calls F-8a5aadac202f with the copied-mode starting address.

The destination write precedes that final dependency call. The byte is stored without Boolean normalization.

### Second optional destination

Still inside the nonzero-mode block, the second query follows regardless of whether the first query result was zero.

The function reads a 32-bit descriptor at offset 0x10 from the saved descriptor base. It calls F-9b9c1c7483f4 with the reused 16-byte region, that descriptor, and the address of the auxiliary-A local word.

F-8094c3d72052 then receives the saved context value and the preceding result. Unlike the first query’s context argument, this argument does not require another context-global read.

The function calls F-8a5aadac202f with the 16-byte region before testing the second query result.

A zero result skips the second extraction and explicit write. A nonzero result causes another descriptor read at saved-base offset 0x10, followed by:

- F-9b9c1c7483f4 using the copied-mode starting address and the copied object-pointer word’s address.
- F-cf946f91e683 using the saved context value.
- An 8-bit write of its result to object offset 0xB1.
- F-8a5aadac202f using the copied-mode starting address.

There is no explicit restoration of the exposed temporary words between uses. Later calls may therefore encounter changes left by earlier calls.

### Final call and return

After the optional-update block, or directly after temporary preparation for zero mode, the function calls F-876f4b90d114 with the original object pointer and original 32-bit mode.

Its result is ignored. If it returns normally, F-0ebf7d7933df returns 1.

## Side effects

### Direct writes and partial progress

The shown direct object writes are:

- Zero to byte 0xD0 on the continuing flag-handling path.
- A dependency-produced byte to 0xB0 when the first query result is nonzero.
- A dependency-produced byte to 0xB1 when the second query result is nonzero.

Local initialization also writes the copied input values into temporary storage.

There is no shown rollback. A flag clear or first-destination write can remain observable even if a later dependency fails, terminates execution, or does not return.

A zero query result guarantees only that this function skips the corresponding explicit extraction/store sequence. It does not guarantee that the destination remains unchanged overall: other dependencies may modify it.

### Global reads and dependency interference

Dependencies are not assumed to preserve the object, globals, descriptor storage, or exposed local storage.

The descriptor-base global is read once for the saved base and again for the first query. Later descriptor reads use the saved address, but reread the descriptor contents when needed. Capturing an address does not snapshot the storage it designates.

The context global is read twice after the first F-9b9c1c7483f4 call: once for the retained value and once for the first query argument. Later queries and extractions use the retained value. Changes to the global do not replace that retained value.

No synchronization or concurrency guarantee is shown.

### Ownership and failures

No direct heap allocation or memory release is shown. F-8a5aadac202f receives each temporary starting address after its normal query or extraction use, but its ownership and release semantics are unknown.

Dependencies receive addresses into local storage. Whether they retain those addresses is unknown; valid retention beyond the invocation is not established.

The function propagates no explicit error status. No recovery is shown for invalid accesses, invalid dispatch targets, dependency failures, or nonlocal control transfer.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-6a21f4f07f67 | Returns the pointer checked before the flag clear and secondary dispatch |
| F-a817d8d58c10 | Invoked for a null returned pointer; annotated as nonreturning |
| F-bb50a072e902 | Called with zero after flag handling |
| F-9b5a149ab944 | Called with the preceding result and zero |
| F-9b9c1c7483f4 | Receives temporary addresses and a 32-bit descriptor; returns an opaque value |
| F-8094c3d72052 | Receives context and an opaque value; returns a 32-bit value tested for zero |
| F-cf946f91e683 | Receives saved context and an opaque value; returns a byte |
| F-8a5aadac202f | Receives a temporary starting address after normal use |
| F-876f4b90d114 | Final call with the original object and mode |

The two indirect targets have no supplied opaque function IDs. Their identities, full interfaces, and side effects remain unresolved.

## Acceptance examples

These are conditional static acceptance scenarios, not executed tests. Unless stated otherwise, they assume valid reached accesses and normally returning dependencies.

1. **Zero mode, converted flag zero.** The initial callback is skipped. The function still performs the source-level flag-word access and character conversion. It skips the secondary callback, performs the unconditional dependency pair, reads the descriptor-base global, initializes local copies, calls F-876f4b90d114 with mode zero, and returns 1. No optional query runs.

2. **Zero mode, converted flag nonzero.** The initial callback remains skipped. If F-6a21f4f07f67 returns a valid object, byte 0xD0 is cleared before the secondary dispatch. This explicit clear does not overwrite bytes 0xD1–0xD3. Optional queries remain skipped.

3. **Initial callback changes the flag word.** With nonzero mode, if the initial callback leaves the word at 0xD0 equal to zero, the subsequent conversion yields zero and flag handling is skipped. The function does not use a pre-callback flag snapshot.

4. **Word access versus narrowed condition.** If a particular implementation converts a nonzero flag word to a zero character value, the flag-handling branch is skipped. This scenario does not eliminate the source-level requirement to access the entire word through byte 0xD3.

5. **Both query results zero.** With nonzero mode, both query temporaries are passed to F-8a5aadac202f before their saved results are tested. Neither extraction runs, and this function makes no explicit write to 0xB0 or 0xB1.

6. **First query succeeds, second does not.** If the first query result is nonzero and its byte-producing dependency returns 0xA5, byte 0xB0 receives 0xA5 before the extraction temporary is passed to F-8a5aadac202f. A zero second query skips the explicit 0xB1 write.

7. **Different nonzero modes.** Modes 1 and 0xFFFFFFFF both satisfy the direct mode-nonzero tests, enabling the initial callback and optional-update block. Identical behavior elsewhere is not guaranteed: the copied mode is exposed to dependencies, and dependency results control other branches. Each execution forwards its own original 32-bit mode unchanged to F-876f4b90d114.

8. **Context changes after capture.** Later queries and extractions receive the saved context value even if a dependency subsequently changes the context global. The first query instead uses the separate global read specified for its argument.

9. **Null secondary object.** A null result from F-6a21f4f07f67 invokes F-a817d8d58c10 before any explicit flag clear. Normal return from the enclosing function is not established for this path.

## Uncertainties

The following remain **documented_only**:

- Domain meanings and valid values of descriptors, context, auxiliary inputs, and destination bytes.
- Character signedness and conversion behavior for flag-word values outside the character type’s representable range.
- Actual physical load width used to implement the flag test.
- Dependency allocation, release, retention, aliasing, preservation, and failure behavior.
- Complete temporary-object sizes, initialization requirements, and permitted write extents.
- The secondary indirect call’s implicit receiver or other calling-convention details.
- F-a817d8d58c10’s actual nonreturning mechanism.
- Synchronization and behavior under concurrent mutation.
- Whether observed query, byte-production, and finalization roles correspond to those dependencies’ intended semantics.

## Evidence

Established claims derive from the supplied static representation: argument widths, object-element access followed by character conversion, explicitly byte-sized stores, dispatch offsets, call arguments, global-read placement, temporary reuse, conditional branches, and the constant normal return.

The flag-test word access and subsequent byte clear are separate operations with different source-level widths. The mode comparisons establish only direct zero/nonzero gating, not equivalence of dependency-controlled behavior for different nonzero modes.

Domain interpretations remain inferred intent. Dependency internals and physical instruction selection are not established. No runtime tests have been performed.
