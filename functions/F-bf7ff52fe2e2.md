<!-- Trusted provenance: F-bf7ff52fe2e2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bf7ff52fe2e2

## Purpose

F-bf7ff52fe2e2 performs ordered initialization or reconfiguration of a mutable target. It selects an initial association, invokes association and mode operations, invokes a target-specific virtual operation, clears selected state, and selects secondary state.

A descriptor-controlled branch additionally resolves a manager, conditionally initializes storage, updates a counter, and conditionally inserts an entry through a virtual operation.

These operations are established by static evidence. “Association,” “manager,” “counter,” and “insertion” are descriptive roles, not established application-level meanings.

## Interface

### Arguments and result

| Argument | Contract |
|---|---|
| Target | A 32-bit address of mutable object storage, required on every path. |
| Related object | A 32-bit value passed unchanged to the target virtual operation. On the inheritance path, a nonzero value is also used as an address. |
| Descriptor | A 32-bit address of configuration storage. An early branch tolerates zero, but later unconditional accesses prevent treating zero as a supported normal-completion input. |

There is no explicit return value. Observable outputs consist of memory changes and dependency calls.

Words and addresses are 32 bits; each word access covers four bytes. Byte accesses affect only the specified byte. All offsets below are byte offsets.

### Directly accessed layouts

| Object | Offset | Use |
|---|---|---|
| Target | `0x00` | Virtual-dispatch-table address. |
| Target | `0x14` | Zero-word write. |
| Target | `0x18` | Zero-byte write. |
| Target | `0x1C`, `0x20`, `0x24`, `0x30` | Zero-word writes. |
| Target | `0x34` | Zero-word write on entry. |
| Target | `0x38` | Initial association; read and conditionally written. |
| Target | `0x44` | Fallback value for target `0x98`. |
| Target | `0x48` | Separately read for two mode decisions. |
| Target | `0x80` | Embedded-region address passed to a dependency. |
| Target | `0x94` | Conditional byte write of one. |
| Target | `0x98` | Selected descriptor value, fallback, or zero. |
| Target | `0x9C` | Secondary association, manager address, or zero. |
| Related object | `0x38` | Inherited association, read only on the inheritance path. |
| Descriptor | `0x0C` | Selector distinguished as zero, `0xFFFFFFFF`, or another value. |
| Descriptor | `0x4C` | Byte whose bits `0x02` and `0x01` are tested at different times. |
| Descriptor | `0x50` | Secondary-state value. |
| Descriptor | `0x54` | Registration condition, tested only following nonzero `0x50`. |
| Manager | `0x00` | Virtual-dispatch-table address. |
| Manager | `0x04` | Primary-storage address. |
| Manager | `0x08` | Secondary-storage address. |

Registration also uses a four-byte local key, a 20-byte scan workspace, and an eight-byte scan-result area containing two four-byte words. The key initially contains the target address. The result area has branch-dependent direct initialization, specified below; it is not generally zero-initialized.

Every reached access requires appropriately readable or writable storage. Reached virtual calls require usable dispatch tables and entries. No general null-result, bounds, allocation-success, or type checks are established.

## Behavior

### Initial reset and association

The first direct effect is a zero-word write at target `0x34`, followed by a read of target `0x38`.

If the association is already nonzero, initial resolution is bypassed. F-8f9bd59e3ec5 receives the association from target `0x38` and the target address.

If the association is zero:

- A zero descriptor skips resolution and attachment.
- Otherwise, descriptor `0x0C` is read. Zero skips resolution and attachment.
- For `0xFFFFFFFF`, a zero related-object argument skips resolution and attachment. Otherwise, the word at related-object `0x38` is copied to target `0x38`.
- For any other nonzero selector, F-283eda65a14c is called without explicit arguments. Its result and `5` are passed to F-682adfd5a561. The latter result is stored at target `0x38`.

The ordinary selector chooses a branch; its value is not passed to the resolver pair.

After either association-producing path, a zero result suppresses attachment. A nonzero result leads to F-8f9bd59e3ec5, receiving the association read from target `0x38` and the target.

### Mode operations

The first decision reads target `0x48` and masks it with `0x1800`. F-e85a9e627a8f receives the target and:

- `0x1000` when the masked value is exactly `0x0800`;
- `0x1800` otherwise.

After that call returns, target `0x48` is read again and masked with `0x0600`. F-0f9e751358c3 receives the target and:

- `0x0400` when the masked value is exactly `0x0200`;
- `0x0600` otherwise.

F-b8e104d8a382 then receives the target and `0x4000`.

These rules specify call arguments, not final mode bits. The second decision observes the field after the first dependency has run.

### Descriptor flag, target operation, and clears

The descriptor byte at `0x4C` is read. If bit `0x02` is set, target byte `0x94` is written with one. If clear, there is no direct write to that target byte.

The target virtual operation at dispatch-table offset `0x9C` receives the target, related-object argument, and zero. F-01844383d2b2 then receives the target and the address of target `0x80`.

After those calls return normally, direct writes occur in this order:

1. Zero word at target `0x14`.
2. Zero byte at target `0x18`.
3. Zero word at target `0x20`.
4. Zero word at target `0x24`.
5. Zero word at target `0x30`.
6. Zero word at target `0x1C`.

The byte write does not directly clear target bytes `0x19` through `0x1B`. Neither preceding call is established to preserve target fields, descriptor contents, or globals.

### Secondary-state selection

Descriptor byte `0x4C` is read again.

If bit `0x01` is clear, zero words are written to target `0x98` and then target `0x9C`, and the function returns.

If bit `0x01` is set, descriptor `0x50` is tested:

- When zero, target `0x44` is copied to target `0x98`. Descriptor `0x54` is not tested.
- When nonzero, descriptor `0x50` is read again and stored at target `0x98`; descriptor `0x54` is then tested.

If the `0x50` test was zero, or the subsequent `0x54` test is zero, the current word at target `0x38` is copied to target `0x9C`, and the function returns. This later association read need not equal the value supplied to the earlier attachment call.

### Registration entry

Registration requires the later descriptor bit `0x01`, a nonzero `0x50` test, and a nonzero `0x54` test.

F-283eda65a14c and F-682adfd5a561 are called again, with `5` supplied to the latter. The resulting manager address is written to target `0x9C` and retained for subsequent operations. Subsequent manager use does not depend on rereading target `0x9C`.

An insertion-position value is initialized to `0x7FFFFFFF`. The local four-byte registration key is initialized with the target address. Both initializations precede F-f4c5ae40135d, whose 32-bit result is captured for unsigned scan comparisons.

The manager virtual operation at dispatch-table offset `0x08` then receives the retained manager address.

### Conditional storage and scan-result initialization

After that manager operation returns, manager `0x04` is tested. Nonzero skips this entire initialization sequence.

When manager `0x04` is zero:

1. F-8157a180d901 receives size `0x10`.
2. F-e194f3fbdfaa is called without explicit arguments, and its result is stored at manager `0x04`.
3. F-8157a180d901 receives size `0x1C`; its returned block is used directly.
4. **After this allocation returns and before any following block-initialization write, the second four-byte word of the local scan-result area, at result-area offset `0x04`, is written with zero.**
5. The block receives zero-word writes at `0x00`, `0x10`, and `0x14`; zero-byte writes at `0x18` and `0x19`; then zero-word writes at `0x0C`, `0x08`, and `0x04`, in that order.
6. F-2ca62cf8d9c9 receives the block. Its returned address is stored at block `0x10`.
7. A zero word is written at returned-address offset `0x04`. Through the address stored at block `0x10`, words at offsets `0x08` and `0x0C` are set to that address.
8. The block address is written to manager `0x08`.
9. F-39af4974f3d2 receives the manager.

The result-area write is separate from the allocated block’s initialization. No direct initialization of the result area’s first word is shown. When existing primary storage causes initialization to be skipped, neither result-area word is explicitly initialized.

The same two-word result area is later passed by address to F-6e4651b532dc if scanning occurs. Its branch-dependent initial state cannot be treated as irrelevant: that dependency’s read/write contract is unknown. The zero write establishes a direct initialization event, not a general guarantee that intervening dependencies preserve the value.

Block bytes `0x1A` and `0x1B` are not directly initialized. The relationship between the first allocation result and F-e194f3fbdfaa is not established.

### Counter update and eligibility

Manager `0x08` is read once, and that secondary-storage value is retained for both lookup calls.

F-f5d5922924e8 receives the retained storage and the registration-key address. The word at its returned address is read, incremented by one, and written back.

F-f5d5922924e8 is called again with the same storage value and local-key address. The word at this second returned address is read as unsigned:

- Values at least two skip scanning and insertion.
- Values below two enter scanning and insertion.

The lookups need not return the same counter address. They are not established to preserve the key, so the second lookup may observe changed key contents.

The increment is represented as signed 32-bit addition. Signed-overflow behavior is unresolved; unconditional modulo-\(2^{32}\) wraparound is not established.

### Scan and insertion

On the insertion path, manager `0x04` is read for F-538044b67754. That dependency receives the 20-byte workspace, this primary-storage value, and the 32-bit value `0xFFFFFFFE`.

F-6e4651b532dc is repeatedly called with the workspace and the address of the same two-word result area described above:

- A zero return ends scanning without changing the insertion-position value from `0x7FFFFFFF`.
- A nonzero return causes the result area’s first word to be used as an entry address. The word at entry offset `0x10` is compared unsigned with the captured F-f4c5ae40135d value.
- An entry value less than or equal to the captured value continues scanning.
- The first strictly greater entry causes F-999d5c439412 to receive the workspace. Its result replaces the insertion-position value.

There is no direct validation of the result-area first word before dereferencing it after a nonzero return. The second word’s meaning, whether either word is read before being written by the dependency, and whether results depend on previous iterations remain unknown.

F-944b595e25e9 then receives the workspace.

Only after these intervening operations is manager `0x04` read again for insertion. The primary-storage virtual operation at dispatch-table offset `0x20` receives the later-read storage address, selected insertion position, and registration-key address.

Insertion storage need not equal workspace-initialization storage. No stronger ordering among field reads within this dispatch expression is established. The key address identifies the original mutable local word; no direct assignment restores its initial target-address contents before insertion.

Whether insertion occurred or was skipped by the counter threshold, the manager virtual operation at dispatch-table offset `0x0C` receives the retained manager address on normal progression, and the function returns.

## Side effects

Direct effects include target-field writes, conditional storage construction, manager-field writes, a counter increment, and conditional initialization of the scan-result area’s second word. Dependencies may produce additional effects.

The function does not directly free storage, restore prior state, report an error, or roll back partial updates. Failures or nonlocal exits can leave earlier effects visible. The final manager operation is not established as guaranteed cleanup on abnormal exits.

The registration key, workspace, and result area have local lifetime. Passing their addresses does not establish whether callees read, mutate, copy, retain, or consume their contents. Retention beyond that lifetime requires a separate contract.

Ownership of resolver results, allocated blocks, linked storage, and inserted entries is unknown. Allocation or insertion alone does not establish release obligations.

The paired manager operations resemble synchronization boundaries. Locking and thread safety are inferred possibilities, not established guarantees.

## Dependencies

| Dependency | Observed interface or role |
|---|---|
| F-283eda65a14c | No explicit arguments; supplies resolver input. |
| F-682adfd5a561 | Receives resolver input and `5`; supplies association or manager address. |
| F-8f9bd59e3ec5 | Receives association and target. |
| F-e85a9e627a8f | Receives target and first selected mode argument. |
| F-0f9e751358c3 | Receives target and second selected mode argument. |
| F-b8e104d8a382 | Receives target and `0x4000`. |
| Target dispatch `0x9C` | Receives target, related-object argument, and zero. |
| F-01844383d2b2 | Receives target and embedded-region address. |
| F-f4c5ae40135d | Supplies captured 32-bit comparison value. |
| Manager dispatches `0x08`, `0x0C` | Surround registration work on normal execution. |
| F-8157a180d901 | Receives allocation sizes `0x10` and `0x1C`. |
| F-e194f3fbdfaa | Supplies primary-storage value. |
| F-2ca62cf8d9c9 | Receives the second block; supplies linked address. |
| F-39af4974f3d2 | Receives manager after storage setup. |
| F-f5d5922924e8 | Receives retained secondary storage and mutable key address; returns counter address. |
| F-538044b67754 | Receives workspace, earlier primary-storage value, and `0xFFFFFFFE`. |
| F-6e4651b532dc | Receives workspace and the branch-dependently initialized two-word result area; returns scan status. |
| F-999d5c439412 | Receives workspace; supplies insertion-position value. |
| F-944b595e25e9 | Receives workspace before insertion’s later storage read. |
| Primary-storage dispatch `0x20` | Receives later-read storage, position, and mutable key address. |

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless specified otherwise, assume normal dependency returns and preservation of fields relevant to each example.

1. **Existing association:** Nonzero target `0x38` causes attachment without initial resolver calls. Registration can independently invoke the resolver pair later.

2. **Inheritance:** With target `0x38` zero and descriptor `0x0C` equal to `0xFFFFFFFF`, related-object association `A` is copied to the target. Attachment occurs only for nonzero `A`. A zero related-object argument skips both actions.

3. **Ordinary selector:** Selector values `1` and `2` select the same resolver-call pattern; neither value is forwarded.

4. **Mode reread:** A first masked value of `0x0800` selects `0x1000`. If that dependency changes the later masked value to `0x0200`, the second mode call receives `0x0400`.

5. **Byte scope:** Clear descriptor bit `0x02` causes no direct clear of target `0x94`. Clearing target `0x18` affects one byte only.

6. **Secondary-state branches:** A later clear bit `0x01` produces zero words at target `0x98` and `0x9C`. With that bit set and descriptor `0x50` zero, target `0x44` supplies `0x98`, and descriptor `0x54` does not select registration.

7. **New-storage result state:** If manager `0x04` is zero, the `0x1C` allocation returns before the result area’s second word is zeroed, and that zeroing precedes the block’s first initialization write. The first result word is not directly initialized. If scanning follows and intervening dependencies preserve the area, its second word is zero before the first scan call.

8. **Existing-storage result state:** If manager `0x04` is nonzero, neither result word receives explicit initialization. Scanning still passes the same area to F-6e4651b532dc; zero contents cannot be required. A hypothetical dependency that reads the second word before replacing it may therefore distinguish the storage branches.

9. **Count threshold:** If both lookups return the same counter and preserve the key, zero becoming one permits insertion; one becoming two suppresses it.

10. **Mutable key:** If the first lookup changes the key to `K`, the second receives the same address containing `K`, absent another write. If the second changes it to `L`, insertion receives that address containing `L` when later operations preserve it.

11. **Storage replacement:** If workspace setup uses storage `A` and a later scan dependency changes manager `0x04` to valid storage `B`, insertion uses `B`, assuming no subsequent change.

12. **Unsigned scan boundary:** An entry equal to the captured comparison value is passed over. The first unsigned-greater entry selects F-999d5c439412’s result. Exhaustion retains `0x7FFFFFFF`.

13. **Zero descriptor:** Avoiding its early dereference does not establish normal completion; the later access at descriptor `0x4C` is unconditional.

## Uncertainties

- **documented_only — Dependency effects:** Preservation of globals, object fields, descriptor contents, and exposed local storage is unknown.
- **documented_only — Scan-result contract:** The second result word’s meaning and the dependency’s input/output requirements are unknown. Its conditional initialization does not establish that it is output-only or irrelevant. The first word is not directly initialized, and neither word is explicitly initialized on the existing-storage path.
- **documented_only — Counter identity and overflow:** Lookup results need not identify the same counter. Signed increment overflow has no established contract.
- **documented_only — Storage stability:** Primary storage may change between workspace setup and insertion. No consistency repair is visible.
- **documented_only — Allocation and ownership:** Failure behavior, first-allocation linkage to primary-storage creation, and release responsibilities are unknown.
- **documented_only — Scan semantics:** Collection ordering, result-field meanings beyond the observed first-word use, position-token meaning, and workspace lifetime rules are unknown.
- **documented_only — Synchronization and failure:** Locking, reentrancy, invalid-address behavior, exception handling, and recovery are not established.

## Evidence

Established claims derive from the supplied static representation’s access widths, offsets, explicit arguments, branch conditions, comparisons, local initialization, dispatch sites, and sequencing.

In particular, the scan-result area contains two four-byte words. Its second word receives a zero write only in the missing-primary-storage branch, after the `0x1C` allocation returns and before block initialization begins. Its first word has no shown direct initialization. Neither word is explicitly initialized on the existing-storage path. The same area is subsequently supplied to F-6e4651b532dc.

Both counter lookups and insertion receive the same mutable key address. Primary storage is read separately for workspace setup and insertion. These observations establish neither immutable key contents nor stable storage identity.

Application roles and synchronization intent remain inferences. No dependency implementations, ownership contracts, or execution traces establish stronger guarantees. No runtime tests have been performed.
