<!-- Trusted provenance: F-ddfd1cf1eed7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ddfd1cf1eed7

## Purpose

F-ddfd1cf1eed7 performs staged initialization, publishes an acquired value through shared storage, registers a derived address, updates a retained state record, invokes an indirect operation, and issues a final notification.

The successful return is the address supplied by F-72538fc0941d minus 720 bytes. It is not necessarily the initial address supplied by F-ff29814bf4cc.

Two explicit conditions lead to the designated non-returning dependency F-a817d8d58c10:

- F-3d00a9979740 returns zero.
- F-c930571cdede returns the 32-bit pattern `0xFFFFFFFF`.

“Initialization,” “registration,” and “notification” describe the apparent roles of these operations. Their application-level meanings are inferred. The contract below instead rests on the supplied static representation’s accesses, widths, arguments, ordering, and branch conditions.

## Interface

### Inputs and result

There are no explicit parameters. Inputs nevertheless include global storage, memory reached through returned addresses, and dependency results.

On normal completion, the routine returns a 32-bit address-like result formed by subtracting `0x2D0` from the result of F-72538fc0941d. No ordinary failure result is shown.

The supplied representation uses 32-bit address-like values and words, 16-bit halfwords, and 8-bit bytes. All offsets below are in bytes. Signed interpretation matters only where explicitly described.

### Contract notation

The following labels describe values, not additional program objects:

| Label | Meaning |
|---|---|
| **A** | Initial address returned by F-ff29814bf4cc. |
| **R** | Address returned by F-72538fc0941d. |
| **B** | Address-like result R minus `0x2D0`, used for registration and the successful return. |
| **S** | Shared-state base captured by the first global-base read after F-3d00a9979740 returns. Retained for subsequent state accesses. |
| **P** | Shared-state base obtained by a separate, subsequent read of the same global-base storage. Used to publish the acquisition result. |
| **Header value** | A 32-bit value read from its global storage after F-ff29814bf4cc returns. |
| **Registration value** | A 32-bit argument value read from its global storage after the first F-unresolved-01 invocation returns. |
| **Notification value** | A 32-bit argument value read from its global storage after F-unresolved-02 returns. |

The three named values are global reads, not established constants. Their concrete values and immutability are unknown. In particular, preceding dependencies must not be assumed to preserve them.

S and P come from distinct reads. Equality between them requires stability of the global-base storage across those reads; that stability is not established.

### Operational preconditions

No visible null, capacity, range, or alignment validation precedes the accesses. Normal completion requires:

- A permits the four-byte header write.
- A plus 720 bytes is suitable for F-72538fc0941d.
- R permits every listed initialization write, ending with the four bytes at offsets `0x48–0x4B`.
- P permits the four-byte publication at offset `0x28`.
- S permits all retained-record reads and writes described below.
- The value later read from S plus `0x28` leads to readable dispatch storage and a callable target.
- Dependencies return normally wherever continuation requires it.

These are requirements for execution, not checks performed by this routine. Address arithmetic outside valid storage, overflow, access faults, and alignment failures have no established recovery contract.

## Behavior

### 1. Obtain the initial address and read the header value

The routine calls F-ff29814bf4cc without explicit arguments and retains its result as A.

After that call returns, it reads the header value from global storage and writes that value as a 32-bit word at A. A value observed before the call would not be a valid substitute: the dependency may change that global storage.

It then calls F-72538fc0941d with A plus `0x2D0`, or 720 bytes. The displacement reflects 180 four-byte units, not 180 bytes.

The dependency’s result becomes R. No comparison establishes that R equals the supplied interior address. B is derived from R, not independently from A.

### 2. Initialize selected fields relative to R

The following direct writes occur in order:

| Order | Offset from R | Width | Value |
|---:|---:|---:|---:|
| 1 | `0x1D` | 8 bits | Zero |
| 2 | `0x20` | 32 bits | Zero |
| 3 | `0x24` | 32 bits | Zero |
| 4 | `0x28` | 32 bits | `0xFFFFFFFF` |
| 5 | `0x2C` | 8 bits | Zero |
| 6 | `0x2D` | 8 bits | Zero |
| 7 | `0x2E` | 8 bits | Zero |
| 8 | `0x30` | 32 bits | Zero |
| 9 | `0x34` | 32 bits | Zero |
| 10 | `0x38` | 8 bits | Zero |
| 11 | `0x3A` | 16 bits | Zero |
| 12 | `0x3C` | 16 bits | Zero |
| 13 | `0x40` | 32 bits | Zero |
| 14 | `0x44` | 32 bits | Zero |
| 15 | `0x48` | 32 bits | Zero |

This is selective initialization, not a contiguous clear. There are no direct initialization writes to the preceding region or to gaps such as `0x1E–0x1F`, `0x2F`, `0x39`, and `0x3E–0x3F`.

The all-ones field at R plus `0x28` is established only as a 32-bit pattern. Its semantic role is unknown.

### 3. Acquire a value, capture S, and publish through P

After the R-relative writes, the routine calls F-3d00a9979740 without explicit arguments.

Following its return, the supplied representation establishes this order:

1. Read the global shared-state base and retain that value as S.
2. Read the global shared-state base again, obtaining P.
3. Store the 32-bit acquisition result at P plus `0x28`.
4. Test the acquisition result for zero.

The publication is therefore not unconditionally a write to S plus `0x28`. If the global base changes between its two reads, S and P can identify different records.

The branch uses the dependency’s returned result, not a reload of the publication field:

- Zero invokes F-a817d8d58c10.
- Any nonzero 32-bit pattern continues to registration.

The zero-result path still performs both base reads and publishes zero through P before invoking failure. It does not guarantee a write to S plus `0x28`.

No rollback of preceding writes is visible.

### 4. Read the registration value and register B

On the nonzero acquisition path, the routine calls F-unresolved-01 without explicit arguments.

After that dependency returns, it reads the registration value from global storage. F-c930571cdede then receives four arguments in positional order:

1. The result of this first F-unresolved-01 invocation.
2. The registration value read after that invocation.
3. B, equal to R minus 720 bytes.
4. The integer one.

The first F-unresolved-01 invocation may change the registration-value storage. The argument is not specified to equal any earlier observation of that storage.

The routine writes the 32-bit registration result at S plus `0x10`, using retained S rather than a fresh global-base read. It then tests the returned result.

Only `0xFFFFFFFF`, representing minus one in the shown 32-bit comparison, selects failure. Zero and every other 32-bit pattern continue.

On failure, the sentinel has already been stored at S plus `0x10`. F-a817d8d58c10 is invoked without visible rollback.

### 5. Update the retained record after successful registration

The routine writes zero to the byte at S plus zero.

It calls F-unresolved-01 a second time; the first result is not reused. After this second invocation returns, it reads the current 32-bit word at S plus `0x10`.

F-23aadb16b673 receives:

1. The second F-unresolved-01 result.
2. The freshly read word from S plus `0x10`.

Consequently, the second argument need not equal the registration result. The intervening dependency may have changed that field. A change to the global shared-state base does not redirect this read away from retained S.

After F-23aadb16b673 returns, the routine performs these writes in order:

| Offset from S | Width | Value |
|---:|---:|---:|
| `0x14` | 32 bits | Zero |
| `0x18` | 32 bits | Zero |
| `0x02` | 8 bits | Zero |

### 6. Dispatch through the current S-relative field

The routine reads the current 32-bit value at S plus `0x28`. It uses that value as an address, reads the address-like word at offset zero there, and obtains an indirect call target from offset `0x2C` in the resulting table-like storage.

The selected target is F-unresolved-03 for purposes of this contract. The supplied representation shows no explicit arguments. It does not establish hidden arguments or receiver conventions.

This dispatch uses a fresh field read from retained S. It does not necessarily use the original acquisition result:

- Publication may have targeted P distinct from S.
- Intervening dependencies may have modified S plus `0x28`.
- Overlapping storage may affect the field.

The returned 32-bit result controls a one-byte assignment:

- Result two or three: write one at S plus `0x01`.
- Any other result: perform no direct write at S plus `0x01`.

There is no unconditional zeroing of that byte. A nonmatching result means “leave unwritten by this step,” not “set false.”

### 7. Read the notification value, notify, and return

The routine calls F-unresolved-02 without explicit arguments.

After that call returns, it reads the notification value from global storage. F-379efa3ca9d1 receives:

1. The F-unresolved-02 result advanced by four bytes.
2. The integer four.
3. The notification value read after F-unresolved-02 returns.

The dependency may change that global value before the read. No earlier snapshot is specified.

The return value of F-379efa3ca9d1 is not used. If it returns normally, F-ddfd1cf1eed7 returns B.

## Side effects

### Direct accesses and global observations

Direct writes comprise the header at A, the fifteen R-relative fields, publication through P, registration and successful-path state changes through S, and the conditional flag assignment.

The significant global observations are:

- Header-value read after F-ff29814bf4cc.
- Two separate shared-base reads after F-3d00a9979740.
- Registration-value read after the first F-unresolved-01.
- Notification-value read after F-unresolved-02.

No dependency is assumed to preserve these global locations. Once captured, S remains the base for later direct shared-record accesses even if the global base changes.

The two later S-relative field reloads are also significant: S plus `0x10` after the second F-unresolved-01, and S plus `0x28` before dispatch.

### Aliasing and final contents

Disjointness of A, R, S, P, global storage, and indirectly reached memory is not established. Where locations overlap, the ordered reads and writes determine the observed behavior.

A direct write does not guarantee the same final contents after later dependency calls. Dependencies may modify the same storage, and later reads may observe those modifications.

### Ownership and partial failure

No explicit release, deallocation, reference decrement, deregistration, or rollback operation appears.

That absence does not establish that dependencies perform no such operations. Allocation, retention, ownership transfer, lifetime, and cleanup responsibilities remain unresolved.

Failure may occur after initialization and publication, or after registration-result storage. The routine provides no visible transactional guarantee.

## Dependencies

| Opaque ID | Established boundary |
|---|---|
| F-ff29814bf4cc | No explicit arguments; supplies A. Header global is read after it returns. |
| F-72538fc0941d | Receives A plus 720 bytes; supplies R. Identity between argument and result is not established. |
| F-3d00a9979740 | Supplies the 32-bit acquisition result. Followed by separate S and P captures, publication through P, and a zero-result check. |
| F-c930571cdede | Receives the first F-unresolved-01 result, a freshly read global value, B, and one. Its 32-bit result is stored through S; only all ones selects failure. |
| F-23aadb16b673 | Receives the second F-unresolved-01 result and the reloaded S-plus-`0x10` word. No returned value is used. |
| F-379efa3ca9d1 | Receives the F-unresolved-02 result plus four bytes, four, and a freshly read global value. No returned value is used. |
| F-a817d8d58c10 | Common failure dependency, identified by the supplied representation as non-returning. Mechanism and external effects are unknown. |
| F-unresolved-01 | Local opaque label for the unresolved dependency invoked twice. Each result is independently obtained and used. |
| F-unresolved-02 | Local opaque label for the dependency supplying the final address-like value. |
| F-unresolved-03 | Runtime-selected target reached through S plus `0x28` and table offset `0x2C`. Its result controls the flag assignment. |

Unresolved labels do not establish dependency internals, global preservation, ownership, or calling-convention details.

## Acceptance examples

These are conditional static examples, not executed tests. Assume valid accesses and normally returning dependencies except where failure is specified. Stability, non-aliasing, and absence of additional mutations apply only when explicitly assumed.

1. **Stable-base successful path.**  
   Both global-base reads yield the same record. Acquisition returns a valid nonzero dispatch object, registration returns seven, and the indirect operation returns two. With no interfering mutations, publication and later accesses use that record; the success resets occur, its byte at `0x01` becomes one, notification runs, and the return is R minus 720.

2. **Different bases for capture and publication.**  
   The first base read yields one valid record, while the second yields another. Acquisition is stored at the second record’s `0x28`. Registration-result storage and subsequent state operations use the first record. Dispatch reads the first record’s `0x28`, which must independently contain a valid dispatch value. The routine does not copy publication into the retained record.

3. **Zero acquisition with different bases.**  
   The acquisition result is zero and the two base reads differ. Zero is stored at P plus `0x28`, then failure is invoked. No write to S plus `0x28` is guaranteed. Registration and all later successful-path calls are not reached.

4. **Registration failure sentinel.**  
   Acquisition is nonzero and registration returns `0xFFFFFFFF`. That pattern is stored at S plus `0x10` before failure. The byte reset at S plus zero and subsequent success operations are not reached.

5. **Other registration patterns succeed.**  
   Registration returns zero or `0xFFFFFFFE`. Neither matches the failure sentinel, so execution continues. Rejecting all signed-negative results would change the contract.

6. **Nonmatching indirect result.**  
   The indirect operation returns four. If S plus `0x01` previously contained nine and no other operation changes it, it remains nine. This routine does not clear it in response to four.

7. **Reloaded registration field.**  
   Registration stores seven at S plus `0x10`; the second F-unresolved-01 changes it to eleven. F-23aadb16b673 receives eleven as its second argument.

8. **Global argument changes during dependencies.**  
   F-ff29814bf4cc changes the header global before returning; the changed value is written at A. The first F-unresolved-01 changes the registration global; that changed value becomes registration argument two. F-unresolved-02 changes the notification global; that changed value becomes notification argument three.

9. **Returned region differs from the requested interior address.**  
   F-72538fc0941d returns a valid R distinct from A plus 720. Initialization uses R, and both registration and the final return use R minus 720, not A.

## Uncertainties

### Inferred intent

The header write resembles installation of an object marker. The address adjustments resemble movement between enclosing and interior regions. Other operations resemble service acquisition, registration, state reset, capability inspection, and notification.

These interpretations do not establish semantic types or replace the offset-and-width contract.

### documented_only

The following remain unresolved:

- Concrete global values and their application-level meanings or immutability.
- Whether the two shared-base reads yield the same address.
- Whether global storage changes concurrently or asynchronously.
- Thread safety, atomicity, synchronization, and memory-ordering guarantees.
- Endianness, alignment requirements, and access-fault behavior.
- Exact overflow and invalid-address behavior for the 720-byte and four-byte adjustments; no explicit arithmetic checks are shown.
- Implicit arguments and ABI requirements of the indirect call.
- Dependency side effects, ownership, retention, releases, and failure modes.
- Whether the failure dependency terminates, throws, or transfers control by another mechanism.
- Allocation identity and aliasing relationships among the participating addresses.

Unknown callees are not assumed to preserve globals or memory, or to acquire or release ownership.

## Evidence

This contract is based solely on the supplied static representation and opaque dependency mapping. Prior prose and reviewer feedback are not treated as independent behavioral proof.

Established static evidence includes the field widths and offsets, selective write order, call sequence, post-call global-value reads, separate shared-base reads, publication through the second base, retained-base accesses, field reloads, exact failure comparisons, indirect dispatch structure, and successful return adjustment.

Semantic intent and environmental behavior remain separated from those observations. No runtime tests have been performed.
