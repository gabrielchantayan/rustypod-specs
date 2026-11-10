<!-- Trusted provenance: F-618b1fec9f4a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-618b1fec9f4a

## Purpose

F-618b1fec9f4a conditionally configures shared state, submits a local record, and creates or reuses a stored resource before passing that resource to a final dependency.

**Established behavior:** Entry to the main body requires a clear exclusion bit and a zero-valued byte latch. Once admitted, the function may perform preliminary preparation, writes several shared fields, sets the latch, and continues through configuration and resource operations. Every visible normal-return path returns the 32-bit value zero.

**Inferred intent:** The sequence resembles initialization followed by activation. Neither successful completion nor exclusive execution is established by the latch: it is written before most configuration and resource operations, and no atomic guard-and-latch operation is visible.

One resource-creation result leads to a dependency marked non-returning. The function does not otherwise expose a success or failure status to its caller.

## Interface

**Explicit inputs:** None.

**Normal output:** A 32-bit zero.

**Implicit inputs:** A context accessor, a resource-context accessor, a global control-object reference, two global 32-bit configuration values, a global address, a static string argument, and state accessible to dependencies.

The observed representation uses four-byte scalar and pointer-bearing fields, plus individual byte accesses. Offsets below are byte offsets.

| Object | Offset | Width | Observed role |
|---|---:|---:|---|
| Context returned by the context accessor | +4 | Address only | Embedded interface passed to dependencies |
| Context returned by the context accessor | +217 | 1 byte | Entry latch |
| Control object | +4 | 4 bytes | Pointer-bearing reference to subordinate state |
| Control object | +8 | 4 bytes | Stored resource identifier |
| Subordinate state | +720 | 4 bytes | Receives a dependency result |
| Subordinate state | +724 | 4 bytes | Selected-entity value, initially written as zero |
| Subordinate state | +775 | 1 byte | Written as zero |
| Object returned for selector 12 | +0 | 4 bytes | Dispatch-table pointer |
| Dispatch table | +12 | 4 bytes | Indirect callable entry |

The complete object layouts, byte order, scalar domain meanings, and portability to other pointer widths are not established. Offsets describe address displacements, not scaled array indices. No arithmetic on the selected-entity or configuration values is visible.

**Preconditions for defined operation:**

- Each accessor result must support the accesses made through that particular result.
- Control and subordinate references must designate valid storage at every use.
- The selector-12 result must support the dispatch-table load and indirect call.
- Local addresses passed to dependencies must satisfy those dependencies’ requirements.
- Address calculations must designate the required accessible storage.

The function provides no visible null checks, bounds checks, synchronization, or recovery for violations of these preconditions. Repeated accessor results and global reads must not be assumed identical merely because their source is the same.

## Behavior

### Entry guards

The function obtains a context, forms the address at context offset +4, and calls F-62d1a359629f with that address and selector 1. It tests bit mask `0x200` in the returned 32-bit value.

If the bit is set, the function returns zero without evaluating the latch guard or entering the body.

If the bit is clear, the context accessor is called again. The byte at offset +217 of this new result is read. Any nonzero byte skips the body and leads to a zero return; only zero permits entry.

The accessor calls are distinct. The source does not establish that the bit-field query and latch read concern the same context. Skipping the body prevents its explicit writes, but does not imply that guard dependencies had no side effects.

### Conditional preparation

After admission, F-f87d423afa3b is called. A nonzero result bypasses preparation. A zero result selects this ordered sequence:

1. F-bfade2068003 receives 54 and 0.
2. F-8ff0260f6859 receives the address of a 128-byte local buffer, length 128, and constant 93.
3. F-ed9d1d84e9d1 receives a separate 32-byte local work area.
4. If that result is zero, F-219b4ea81e9a receives the work area, the 128-byte buffer, and length 128.
5. If both zero-tested results are zero, F-77f59129a16d receives the work area and a static string argument.
6. F-85722f62e20a receives 54 and 1000, regardless of the two tested results, provided execution reaches it normally.

The function does not explicitly initialize either local area before its first dependency use. Their resulting contents and initialization obligations are **documented_only**.

The results of F-8ff0260f6859 and F-77f59129a16d are not tested. Nonzero results from either zero-tested dependency suppress later preparation calls as described, but do not abort shared-state setup. The static string’s exact contents are not established by the supplied material.

### Shared-state setup

The global control-object reference is captured for later use. The initial zero store nevertheless obtains its subordinate pointer through a separate read of the global control reference. The visible order is:

1. Capture the global control reference.
2. Separately read that global reference, load its pointer-bearing field at +4, and write a 32-bit zero at subordinate offset +724.
3. Obtain a context and call F-b37968c9f644 with context offset +4 and zero.
4. Load the subordinate pointer from offset +4 of the captured control object.
5. Store the returned 32-bit value at subordinate offset +720.
6. Store a zero byte at offset +775 of that same subordinate object.
7. Obtain another context and write byte value one at its offset +217.
8. Invoke the unmapped external dependency with the global address and constant 36.

No dependency call intervenes between capture of the control reference and the separate global read used for the initial zero store. They remain distinct reads; equality is not an unconditional contract.

Subsequent control accesses use the captured reference. Dependencies may change its contents or the subordinate pointer it contains. The stores at +720 and +775 use the same subordinate pointer loaded after F-b37968c9f644 returns.

The external operation’s argument 36 does not establish clearing, copying, or a 36-byte memory extent. Its semantics are **documented_only**.

The latch is already one before the following configuration, record, and resource operations. No rollback is visible.

### Configuration and distinct configuration reads

Let **configuration A** and **configuration B** denote the two global 32-bit configuration locations. Each has a saved value for later use, distinct from a separate read supplied to its initial configuration call.

Each of the following five rows first obtains a fresh context. It passes that context’s embedded interface at +4 and freshly reads the subordinate pointer from the captured control object, then the selected-entity field at subordinate offset +724.

| Order | Dependency | Configuration argument |
|---:|---|---|
| 1 | F-1cd96d01ddae | Separate argument read of global configuration A |
| 2 | F-9ef7726bf517 | Saved configuration A |
| 3 | F-da2fe84eecde | Separate argument read of global configuration B |
| 4 | F-67b16ca5ec28 | Constant 19 |
| 5 | F-56ee0cebf586 | Constant 2 |

The configuration capture points are significant:

- **Row 1:** Obtain the context first. Then read configuration A into its saved value. Separately read configuration A again as the argument to F-1cd96d01ddae. No dependency call intervenes between these two configuration reads.
- **Row 2:** Obtain another context. F-9ef7726bf517 receives the saved configuration A, not another read of that global.
- **Row 3:** Obtain the row’s context first. Then read configuration B into its saved value. Separately read configuration B again as the argument to F-da2fe84eecde. No dependency call intervenes between these two configuration reads.
- **Later record construction:** Use saved configuration B and saved configuration A.

Thus, each initial configuration call and its later consumers do not necessarily share one snapshot. If the global remains stable between its two reads, the argument value equals the saved value. Without that condition, equality is not established. Conversely, there is no intervening dependency call to explain a change between either pair of reads; asynchronous or other externally changing state would be required.

Context-accessor effects occur before the corresponding saved read. Later accessor or configuration calls cannot retroactively change either saved value.

Although subordinate offset +724 was initially written as zero, subsequent arguments are not unconditionally zero. Intervening dependencies may modify the field or replace the subordinate pointer.

After the five rows, another context is obtained. F-96b09b4480d9 receives:

- The context itself, rather than its +4 embedded interface.
- A freshly read selected-entity value.
- Constants 1 and 0.
- The address of a separate four-byte local scratch area.

The function does not initialize this scratch area before passing it and does not inspect it afterward.

### Related-object acquisition and indirect dispatch

The function obtains a context and calls F-9266831caef7 with its embedded interface and selector zero. The returned 32-bit value is retained for two later calls.

It then obtains another context and calls F-9266831caef7 with its embedded interface and selector 12. This result is treated as an object pointer. The function reads the object’s first pointer-bearing word, reads the callable entry at byte offset 12 in the resulting dispatch table, and invokes it with:

- The selector-12 object as receiver.
- Constant 4.
- A freshly read selected-entity value.

No object-validity check, dispatch-table check, or dispatch-result check is visible. The indirect target’s identity and semantics are **documented_only**.

### Record construction and submission

The function prepares a 20-byte local record with this layout:

| Record offset | Width | Content |
|---|---:|---|
| 0 | 1 byte | Zero |
| 1–3 | 3 bytes | Not explicitly initialized by this function |
| 4 | 4 bytes | Integer 16 |
| 8 | 4 bytes | Selected-entity value read during construction |
| 12 | 4 bytes | Saved configuration B |
| 16 | 4 bytes | Saved configuration A |

The visible write order is: integer 16 at +4, zero byte at +0, selected-entity value at +8, saved configuration B at +12, and saved configuration A at +16.

F-c6c589453fa8 then receives zero, zero, the record address, and length 20.

The three unwritten bytes are not specified as zero or stable. Whether the dependency reads them is unknown. The internal integer 16 and submission length 20 are separate observed values; no semantic relationship between them is established.

### Ordered operations using the retained value

F-2c4fc5b4a966 is called twice using the retained selector-zero result:

1. Retained value, freshly read selected-entity value, selector zero.
2. Retained value, freshly read selected-entity value, selector 6.

The subordinate pointer and selected-entity field are reread for the second call. Changes made by the first call may therefore affect the second call’s selected-entity argument.

### Resource creation or reuse

The function reads the 32-bit resource field at offset +8 of the captured control object and compares it with signed −1, the all-ones 32-bit representation.

If equal:

1. Obtain a value from the resource-context accessor.
2. Call F-c930571cdede with that value, 1000, the subordinate pointer freshly loaded from the captured control object, and zero.
3. Store the 32-bit result at control offset +8.
4. Compare that returned result with −1. If equal, invoke F-a817d8d58c10, marked non-returning.

The failure decision uses the creation result, not a separate reread of the stored field.

If the initial resource field is anything other than −1, creation is skipped. Zero and other negative signed interpretations are not separately rejected.

On a continuing path, the resource-context accessor is called again. F-8becc2a0337c receives its result and the resource field freshly read from control offset +8. This read occurs after the accessor call; the argument is not unconditionally the earlier existing value or creation result.

After that call returns normally, F-618b1fec9f4a returns zero.

## Side effects

**Direct shared writes:** A 32-bit zero at subordinate +724; a 32-bit dependency result at subordinate +720; a zero byte at subordinate +775; byte one at context +217; and, conditionally, a 32-bit creation result at control +8. Distinct reads may select distinct objects as described above.

**Dependency effects:** Calls may modify shared objects, globals, references, or external state. No preservation guarantee is supplied. Return values that are ignored do not imply absence of effects or success.

**Ownership:** The visible body performs no explicit allocation, release, or destruction. It passes local-buffer addresses, a local record, scratch storage, shared-state pointers, and object references to dependencies. None of this establishes transfer, borrowing rules, copying, retention, or release responsibilities. Local storage is available during this invocation; safe retention beyond it is not established.

**Errors:** Preparation statuses only control the nested preparation calls. Creation result −1 is stored before the non-returning failure call. Other returned values are ignored or consumed without explicit validation. No transaction, cleanup sequence, or rollback is visible.

**Concurrency:** No visible synchronization makes the exclusion query, latch test, and latch write atomic. Concurrent or reentrant invocations are not established to exclude one another.

## Dependencies

Opaque IDs identify supplied dependencies; roles below describe call boundaries, not implementation guarantees.

| Identity | Observed role |
|---|---|
| F-62d1a359629f | Supplies the exclusion bit field |
| F-f87d423afa3b | Selects whether preparation runs |
| F-bfade2068003; F-85722f62e20a | Receive preparation-boundary constants |
| F-8ff0260f6859 | Receives the 128-byte buffer |
| F-ed9d1d84e9d1; F-219b4ea81e9a | Return zero-tested preparation statuses |
| F-77f59129a16d | Receives the work area and static string |
| F-b37968c9f644 | Supplies the value stored at subordinate +720 |
| F-1cd96d01ddae; F-9ef7726bf517 | Receive, respectively, separately read and saved configuration A |
| F-da2fe84eecde | Receives separately read configuration B |
| F-67b16ca5ec28; F-56ee0cebf586 | Receive constants 19 and 2 |
| F-96b09b4480d9 | Receives the whole context and scratch address |
| F-9266831caef7 | Supplies selector-zero and selector-12 results |
| F-c6c589453fa8 | Receives the 20-byte record |
| F-2c4fc5b4a966 | Receives ordered selectors zero and 6 |
| F-c930571cdede | Supplies the resource-creation result |
| F-a817d8d58c10 | Marked non-returning failure dependency |
| F-8becc2a0337c | Receives resource context and stored resource value |

No opaque IDs are supplied for either accessor, the address-and-36 operation, or the indirect target. Those boundaries remain **documented_only** rather than receiving invented identities.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Exclusion:** A guard result containing `0x200` produces a zero return without the second context access or body execution.

2. **Nonzero latch:** With the exclusion bit clear, latch value 2 skips the body exactly as latch value 1 does.

3. **Preparation failure:** If preparation runs and F-ed9d1d84e9d1 returns nonzero, both subsequent nested preparation calls are skipped. F-85722f62e20a still runs if preceding calls return normally, followed by shared-state setup.

4. **Second preparation failure:** If the first status is zero and F-219b4ea81e9a returns nonzero, the static-string call is skipped without aborting the body.

5. **Accessor before capture:** If row 1’s accessor changes configuration A to 40, and that global remains stable through both following reads, saved configuration A and the F-1cd96d01ddae argument are both 40. The analogous rule applies to configuration B after row 3’s accessor.

6. **Distinct A reads:** If configuration A is 40 at its saved read and 41 at its separate argument read, F-1cd96d01ddae receives 41, while F-9ef7726bf517 and record offset +16 use 40. No dependency call occurs between those reads.

7. **Distinct B reads:** If configuration B is 70 at capture and 71 at the argument read, F-da2fe84eecde receives 71 and record offset +12 uses 70.

8. **Selected-entity mutation:** If a dependency changes subordinate +724 or replaces the subordinate pointer, subsequent selected-entity reads observe the resulting state. The record captures its own construction-time value.

9. **Record contents:** Submission length is 20. Record offset +4 contains integer 16. Acceptance must not require bytes 1–3 to be zero.

10. **Resource reuse:** If the initial resource value is zero, creation is skipped. If the final resource-context accessor changes the stored field, the final dependency receives the subsequently read value.

11. **Creation failure:** If creation returns −1, that result is stored before F-a817d8d58c10 is invoked. Normal return is not required on this branch.

## Uncertainties

- **Inferred intent:** Initialization, configuration, and activation are descriptive interpretations, not established domain semantics.
- **documented_only:** Dependency behavior beyond observed arguments, tested results, and uses.
- **documented_only:** Accessor stability, reentrancy, concurrency guarantees, and preservation of globals.
- **documented_only:** Mechanisms that could change a configuration global between its consecutive saved and argument reads.
- **documented_only:** Exact string contents and meanings of selectors, configuration values, and resource identifiers.
- **documented_only:** The address-and-36 operation’s memory extent and effects.
- **documented_only:** Ownership, copying, retention, and release obligations.
- **documented_only:** Recipient treatment of uninitialized local bytes and scratch storage.
- **documented_only:** Indirect-call semantics and abnormal dependency outcomes.

## Evidence

Established statements derive from the supplied static body and opaque dependency mapping: access widths and offsets, short-circuit conditions, call ordering, distinct configuration reads, capture points relative to accessor calls, repeated pointer and field reads, record layout and write order, resource-result handling, and normal return value.

The non-returning designation is supplied static evidence, not an independently verified dependency implementation. No dependency implementations or runtime observations were supplied. No runtime tests have been performed.
