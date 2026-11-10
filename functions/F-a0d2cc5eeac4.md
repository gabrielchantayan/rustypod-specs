<!-- Trusted provenance: F-a0d2cc5eeac4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a0d2cc5eeac4

## Purpose

F-a0d2cc5eeac4 dispatches a 32-bit numeric request against an object and, on handled paths, writes one 32-bit result to caller-provided storage. Its separate 32-bit return indicates whether the request was handled, not whether the result is nonzero.

Before dispatch, the function conditionally invokes a shared-state service sequence. This sequence can execute even when the eventual request is unhandled.

The contract below describes behavior established by the supplied static body. Terms such as “service,” “lookup key,” “payload,” and “scratch” identify apparent roles without establishing the dependencies’ underlying semantics. In particular, a reference loaded from shared state is not necessarily stable across dependency calls.

## Interface

### Arguments and result

| Input | Representation | Use |
|---|---|---|
| Object reference | 32-bit address-bearing argument | Supplies the base for object-field reads and object-relative dependency arguments. |
| Context | 32-bit word | Compared with a shared expected-context word for request `0x7694` only. |
| Request | 32-bit integer | Selects the dispatch path. |
| Output reference | Address of writable 32-bit storage | Receives one explicit 32-bit assignment on every normally completed handled path. |

The normal return is:

- **1:** handled; an explicit output assignment occurred.
- **0:** unhandled; no explicit output assignment occurred.

A handled result of zero still returns 1. The body does not translate invalid memory access, dependency failure, or abnormal control transfer into a defined error result.

The supplied representation supports a 32-bit address model. Exact ABI, endianness, alignment enforcement, and machine-level fault behavior are not established.

### Direct object accesses

| Object offset | Access | Role |
|---|---|---|
| `0x19` | One byte | Zero disables either indexed request family; every nonzero byte enables it. |
| `0x1c` | Address formation | Embedded-region argument for F-44362605ab79. |
| `0x20` | 32-bit pointer read; conditional one-byte dereference | Selects the content-present or content-absent path for `0x7691`. |
| `0x4c` | Address formation | Embedded-region argument for F-7a393ce8e214. |
| `0x6c` | Unsigned one-byte read | Direct result for `0x7681`. |

Address formation at `0x1c` or `0x4c` is not itself a direct read of that region by this function. Dependencies may access additional storage.

### Preconditions

Every executed direct read and final output write requires accessible storage appropriate to that access. Dependency-specific preconditions are unknown. No pointer, object extent, or output-address validation is visible.

Object-relative addresses must be meaningful in the target address model. The body does not provide checked address arithmetic or recovery from invalid addresses. No request-independent minimum object size can be established because different paths access different regions.

## Behavior

### Shared values and capture rules

The function uses distinct shared slots holding:

- A control-structure reference.
- A service reference.
- A scratch reference.
- Two additional operands, identified here as **G-2204** and **G-2200**, corresponding to the respective slot suffixes.
- A default lookup key.
- Separate empty-case, nonempty-case, content-present, and content-absent lookup keys.
- An expected-context word.

These names identify storage roles, not immutable values. “Read again” below means a fresh load from the relevant slot; intervening dependencies may have changed it.

An argument value already captured for a call is distinct from the current contents of its originating slot. Likewise, retained dependency results and the captured default key are not replaced merely because later calls alter shared storage.

### Preliminary service sequence

Every invocation first obtains the shared control-structure reference and reads the 32-bit word at offset `0x10` within that structure.

If bit 0 is set, no preliminary dependency is called.

If bit 0 is clear:

1. Read the service-reference slot and call **F-b255b26284e6** with that value.
2. If the result is zero, perform no further preliminary calls.
3. If the result is nonzero:
   - After F-b255b26284e6 returns, read the scratch-reference slot and call **F-5b24edfb22c0** with that value.
   - After F-5b24edfb22c0 returns, read G-2204 and G-2200. Call **F-804adc29bbfb** with the F-5b24edfb22c0 result first, G-2204 second, and G-2200 third.
   - After F-804adc29bbfb returns, read the service-reference slot again and call **F-d08d3ca17b81** with this newly read value.

The two additional operands are not established constants. Both are read after F-5b24edfb22c0 returns; their relative load order is not established here. The final service argument need not equal the argument previously passed to F-b255b26284e6.

The results of F-804adc29bbfb and F-d08d3ca17b81 are ignored. The control word is not reread to reconsider whether servicing should continue.

After servicing completes or is skipped, the function captures the **default lookup key**, before dispatch. This capture occurs even for requests that do not subsequently use that key.

### Direct and explicitly unhandled requests

**`0x7681`:** Read the unsigned byte at object offset `0x6c`, zero-extend it to 32 bits, write it to output, and return 1. Byte values `0x80`–`0xff` produce 128–255.

**`0x7682`:** Return 0 without an explicit output assignment.

**`0x7684`:** Call **F-44362605ab79** with the address at object offset `0x1c`. Write its 32-bit result and return 1.

For every request not otherwise listed, return 0 without a request-specific object read or dependency call. Preliminary servicing and the default-key capture have already occurred.

### Conditional lookup: `0x7683`

Call **F-7a393ce8e214** with the address at object offset `0x4c`.

If its result is zero:

- Call **F-9d0ecc5f6adf**.
- After that call returns, read the empty-case lookup key.
- Call **F-c0b020cd3882** with the provider result first and the newly read key second.
- Write the lookup result directly and return 1.

If the first result is nonzero:

- Call **F-7a393ce8e214** again with the same object-relative address and retain this second result as the payload.
- Call **F-9d0ecc5f6adf**.
- After that call returns, read the nonempty-case lookup key.
- Perform the three-argument scratch path described below, using this key, provider result, and retained payload.

The first query result selects the branch; only the second supplies the payload. A zero second result does not change the selected branch.

### Indexed request families

Both families first read the byte at object offset `0x19`. If it is zero, write zero and return 1 without calling the indexed dependency. Otherwise, use the following mapping:

| Requests | Dependency | Indices in request order |
|---|---|---|
| `0x7685`–`0x768a` | **F-50d28d8f6fed** | 0, 1, 2, 3, 4, 5 |
| `0x768b`–`0x7690` | **F-639d81b148f5** | 0, 1, 2, 3, 4, 5 |

The dependency receives the object reference first and index second. Its 32-bit result becomes the output, followed by return 1.

For each family, indices 0–4 come from subtracting the family’s first request value; index 5 is selected explicitly. These recognized inputs produce exact values within 0–5, without arithmetic overflow. No fallback between families occurs.

The enable byte is tested only for zero versus nonzero; its signed interpretation does not affect selection.

### Content-dependent request: `0x7691`

Read and retain the pointer stored at object offset `0x20`.

- If null, do not dereference it.
- If nonnull, read its first byte once.
- Select the content-present branch only if the retained pointer is nonnull and that byte is nonzero.

The function does not directly scan for a terminator or reread the pointer for branch selection.

For content present:

1. Call **F-44362605ab79** with the address at object offset `0x1c`; retain its result as payload.
2. Call **F-9d0ecc5f6adf**.
3. After the provider returns, read the content-present lookup key.
4. Perform the three-argument scratch path.

The payload comes from the embedded region at `0x1c`, not from passing the pointer read at `0x20`.

For a null pointer or zero first byte:

1. Call **F-9d0ecc5f6adf**.
2. After it returns, read the content-absent lookup key.
3. Call **F-c0b020cd3882** with the provider result first and that key second.
4. After lookup returns, read the scratch-reference slot and call **F-151ab4bfe553** with this reference first and the lookup result second.
5. After F-151ab4bfe553 returns, read the scratch-reference slot again.
6. Pass this newly read reference to **F-44362605ab79**, write its result, and return 1.

The scratch reference used by F-151ab4bfe553 need not equal the one subsequently used by F-44362605ab79.

### Default-key payload requests

For **`0x7692`**, obtain the payload from **F-d53862ada031** with the object reference.

For **`0x7693`**, obtain the payload from **F-7a393ce8e214** with the address at object offset `0x4c`.

Then call **F-9d0ecc5f6adf** and perform the three-argument scratch path using the default key captured before dispatch.

Neither the payload call nor the provider call causes the default key to be reread. Changes those calls make to its shared slot do not replace the saved key. A zero payload does not select a different path.

### Three-argument scratch path

This path is used by nonempty-case `0x7683`, content-present `0x7691`, `0x7692`, and `0x7693`.

With the selected key, provider result, and payload retained:

1. Call **F-c0b020cd3882** with the provider result first and selected key second.
2. After lookup returns, read the scratch-reference slot.
3. Call **F-1f62160dfd52** with that scratch reference first, lookup result second, and payload third.
4. After F-1f62160dfd52 returns, read the scratch-reference slot again.
5. Call **F-44362605ab79** with this freshly read reference.
6. Write its 32-bit result and return 1.

The F-1f62160dfd52 result is ignored. Its scratch argument and the subsequent conversion argument need not be equal. In particular, the function does not guarantee conversion of the same referenced resource that received the preceding operation.

### Context-gated request: `0x7694`

Read the expected-context word during this request’s dispatch, after preliminary servicing and default-key capture.

If it differs from the context argument, return 0 without an explicit output assignment or request-specific object query.

If equal, call **F-7a393ce8e214** with the address at object offset `0x4c`, write its result directly, and return 1.

The comparison uses the expected-context value at this read, not a value captured on entry.

## Side effects

Every normally completed handled path performs exactly one explicit 32-bit output assignment after its relevant reads and calls. Unhandled paths perform none. Dependencies may independently write to output storage or aliases of it; “no explicit assignment” is not an unchanged-memory guarantee.

Dependencies may alter object storage, shared slots, referenced resources, or other state. This possibility matters directly to the fresh-load schedules above. Neither a query-like role nor an ignored result establishes purity.

No explicit allocation, deallocation, reference acquisition, or reference release appears in the body. Dependency ownership and lifetime behavior remain unknown. Returned words are not established to be owned objects, borrowed pointers, strings, or stable handles. Passing a reference to a dependency does not establish ownership transfer.

No aliasing restriction is checked. Output storage may overlap object or shared storage, in which case the final assignment can overwrite it. Earlier dependency effects can also affect aliased locations.

No locking, rollback, or reentrancy protection is visible. Concurrent-access guarantees are not established.

## Dependencies

Only the listed argument/result roles are established. Additional semantics are `documented_only`.

| Opaque ID | Observed contract at this boundary |
|---|---|
| **F-b255b26284e6** | Takes the currently read service reference; zero/nonzero result controls further preliminary calls. |
| **F-5b24edfb22c0** | Takes a freshly read scratch reference; result becomes the first F-804adc29bbfb argument. |
| **F-804adc29bbfb** | Takes the preceding result, G-2204, and G-2200, in that order; result ignored. |
| **F-d08d3ca17b81** | Takes the service reference reread after F-804adc29bbfb; result ignored. |
| **F-9d0ecc5f6adf** | Takes no arguments; supplies the first lookup argument. |
| **F-c0b020cd3882** | Takes provider result and selected key; supplies a 32-bit result. |
| **F-44362605ab79** | Takes an embedded-region address or scratch reference; supplies a 32-bit result. |
| **F-7a393ce8e214** | Takes the object-relative address at `0x4c`; supplies a zero-tested result or payload/output word. |
| **F-1f62160dfd52** | Takes scratch reference, lookup result, and payload; result ignored. |
| **F-151ab4bfe553** | Takes scratch reference and lookup result; result ignored. |
| **F-50d28d8f6fed** | Takes object reference and first-family index 0–5. |
| **F-639d81b148f5** | Takes object reference and second-family index 0–5. |
| **F-d53862ada031** | Takes object reference; supplies the `0x7692` payload. |

No dependency is assumed to preserve globals or object contents, complete normally, or allocate or release memory in any particular way.

## Acceptance examples

These are conditional static expectations, not executed tests. Dependencies are assumed to complete normally and not independently modify output storage. **Every stated object-field or shared-slot value applies when the dispatcher actually reads that location**, not merely at function entry.

1. **Unsigned property:** For `0x7681`, a byte of `0xff` at the `0x6c` read produces output `0x000000ff` and return 1.

2. **Servicing before rejection:** For an unknown request, control bit 0 clear and a nonzero F-b255b26284e6 result cause the four preliminary calls in their specified order. Default-key capture follows; dispatch returns 0 without an explicit output assignment.

3. **Service suppression:** If control bit 0 is set when read, F-b255b26284e6 is not called.

4. **Fresh preliminary operands:** If F-5b24edfb22c0 changes G-2204 and G-2200 before returning, F-804adc29bbfb receives their changed values as its second and third arguments.

5. **Service-reference replacement:** If F-804adc29bbfb changes the service-reference slot, F-d08d3ca17b81 receives the replacement, not necessarily the earlier query argument.

6. **Disabled index:** For `0x768a`, a zero enable byte at its read produces output zero and return 1 without F-50d28d8f6fed.

7. **Enabled boundary index:** For `0x7690`, a nonzero enable byte at its read causes F-639d81b148f5 to receive index 5; its result becomes output.

8. **Repeated query:** For `0x7683`, query results 9 then 0 select the nonempty-case scratch path with payload zero. The direct empty-case path is not taken.

9. **Fresh branch key:** If F-9d0ecc5f6adf changes the empty-case key before returning during the zero branch of `0x7683`, lookup receives the changed key.

10. **Saved default key:** For `0x7692`, if the captured default key is A and the payload or provider call changes its shared slot to B, lookup still receives A.

11. **Scratch replacement:** On a three-argument scratch path, if F-1f62160dfd52 changes the scratch slot from reference A to B, the subsequent F-44362605ab79 receives B. The same reload rule applies after F-151ab4bfe553 on the content-absent path.

12. **Absent content:** For `0x7691`, a null pointer at the pointer read causes no dereference. A valid nonnull pointer whose first byte is zero when read selects the same content-absent branch.

13. **Context rejection:** For `0x7694`, an expected-context word unequal to the context at comparison time produces return 0 without the request-specific F-7a393ce8e214 call.

14. **Handled zero:** For matching-context `0x7694`, a query result of zero is written to output and return remains 1.

## Uncertainties

- **`documented_only` — semantic purpose:** Property meanings, key contents, resource formats, and actual subsystem purpose are unknown.
- **`documented_only` — dependency behavior:** Failure reporting, side effects, allocation, ownership, reference retention, and lifetime guarantees are unavailable.
- **`documented_only` — representation details:** Exact ABI, alignment, endianness, and fault behavior are not independently established.
- **`documented_only` — invalid inputs:** There is no visible validation or defined recovery for inaccessible object, shared, pointed-to, or output storage.
- **`documented_only` — operand load ordering:** Both G-2204 and G-2200 are read after F-5b24edfb22c0 returns and before F-804adc29bbfb; their relative read order is unresolved.
- **`documented_only` — concurrency:** The explicit capture/reload schedule does not establish atomicity, synchronization, or behavior under concurrent mutation.
- **Inferred intent:** Lookup-and-scratch paths resemble presentation construction, and preliminary calls resemble servicing followed by acknowledgement. Neither interpretation establishes formatting, encoding, notification, clearing, or ownership semantics.

## Evidence

The supplied static body establishes request selection, fixed object offsets, byte and word accesses, zero/nonzero conditions, dependency identities and argument positions, retained results, shared-value capture and reload points, output assignments, and normal return values.

The dependency map establishes the opaque function identities used here. Comments, prior descriptions, and naming implications do not establish additional behavior.

No dependency implementation, independent ABI documentation, or runtime trace was supplied. No runtime tests have been performed.
