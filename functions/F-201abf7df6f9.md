<!-- Trusted provenance: F-201abf7df6f9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-201abf7df6f9

## Purpose

F-201abf7df6f9 derives a 32-bit value from an input object, conditionally installs that value into one of two eligible field groups in an attached object, marks that attached object, and invokes downstream processing. It may then issue a notification through a related object. Finally, it obtains another object through an indirect call and sets a flag in that returned object.

The arithmetic resembles a mode-dependent conversion, and the paired field groups resemble alternative update targets. Those interpretations are inferred intent, not established domain semantics. No units, object types, or application-level meanings are established.

A missing attached object suppresses all work after the initial conversion call. Failure to find an eligible field group does **not** suppress downstream processing or the final flag update.

## Interface

### Input and result

The function accepts one input-object address and has no explicit return value.

The supplied representation uses 32-bit integer words and a 32-bit pointer model. This contract describes that model; it does not establish behavior for a different ABI.

All offsets below are byte offsets from the indicated object address. Hexadecimal values denote offsets, masks, or bit patterns unless otherwise stated.

### Input-object layout

| Offset | Access | Meaning established by use |
|---|---|---|
| `0x20` | Read as a signed 32-bit integer | Arithmetic input |
| `0x2D` | Read as one byte | Selects the arithmetic branch when equal to `0x03` |
| `0x94` | Read as a pointer | Attached object; may be null |

The mode byte and arithmetic input are read before the first dependency call. The attached-object pointer is read **after** that call. Consequently, the pointer used is not necessarily the one that existed on entry.

### Attached-object layout

| Offset | Access | Meaning established by use |
|---|---|---|
| `0x00` | Read as a pointer | Dispatch-table address for the final indirect call |
| `0x34` | Read as a pointer | Optional related object |
| `0x48` | Read and written as 32 bits | Flag word; bit `0x20` is set |
| `0x54` | Read as 32 bits | First eligibility tag for the preferred group |
| `0x58` | Conditionally written as 32 bits | Preferred-group copied value |
| `0x60` | Conditionally read as 32 bits | Second eligibility tag for the preferred group |
| `0x64` | Conditionally written as 32 bits | Preferred-group derived value |
| `0x6C` | Conditionally read as 32 bits | First eligibility tag for the fallback group |
| `0x70` | Conditionally written as 32 bits | Fallback-group copied value |
| `0x78` | Conditionally read as 32 bits | Second eligibility tag for the fallback group |
| `0x7C` | Conditionally written as 32 bits | Fallback-group derived value |
| `0x80` | Conditionally read as 32 bits; its address is passed onward | Source value and embedded-region start |
| `0xA0` | Conditionally read as a word and narrowed to a character | Only its low eight bits control notification |

The copied value at `0x80` has no established signed numeric meaning. Its 32-bit contents are copied unchanged.

### Other accessed layouts

The optional related object contains a dispatch-table pointer at offset `0x00`. A callable entry is obtained from byte offset `0x128` in that table.

The attached object’s dispatch table contains a callable entry at byte offset `0x5C`. That call receives the attached-object address and returns an address.

The returned address must permit a 32-bit read and write at offset `0x48`. The function sets bit `0x40` in that word.

### Preconditions

Every address actually dereferenced must be valid for the represented access width and execution environment. Relevant dispatch entries must be callable with the observed calling conventions.

The function checks only:

- Whether the attached-object pointer is null.
- Whether the optional related-object pointer is null.
- The low-eight-bit notification condition.

It does not validate the input-object pointer, dispatch tables, dispatch entries, or the address returned by the final indirect call.

## Behavior

### 1. Derive a value before checking for an attachment

The mode byte at input offset `0x2D` selects one of two calculations:

| Mode condition | First argument to F-870c4fcf7295 | Second argument | Adjustment to returned value |
|---|---|---|---|
| Equal to `0x03` | `(input value − 76,000) × 20` | `1,000` | Add `18` |
| Any other byte | `(input value − 88,000) × 14` | `1,000` | Add `20` |

The input value comes from signed 32-bit offset `0x20`.

F-870c4fcf7295 is called exactly once if execution reaches this stage normally. Its return value, after the selected adjustment, is the derived value used later.

The second argument being `1,000` does not establish division, scaling, rounding, saturation, or any other implementation of that dependency. Those semantics remain unknown.

The subtraction, multiplication, and return-value adjustment operate in the displayed signed 32-bit model. No wider intermediate or overflow check is visible. For values whose intermediate results are not representable, the supplied evidence does not establish whether execution wraps, traps, or has another target-specific outcome. Acceptance of an ordinary mathematical result is therefore conditional on representable arithmetic.

### 2. Read the attachment after conversion

After the conversion call and adjustment, the function reads the pointer at input offset `0x94`.

If that pointer is null, the function returns immediately. It performs no direct attached-object writes, eligibility reads, downstream-processing call, notification, or final dispatch.

This is not a side-effect-free early return: F-870c4fcf7295 has already run and may have effects of its own.

For a nonnull pointer, the function retains that attached-object address for the remainder of its direct processing. It does not visibly reload the input object’s attachment field later.

### 3. Select at most one eligible field group

A tag is eligible when clearing bit `0x20` from its 32-bit representation produces exactly `0x80000002`.

Equivalently, the accepted tag patterns are:

- `0x80000002`
- `0x80000022`

All other bits must match the required pattern. This is a bit-pattern test, not a signed numeric comparison.

The preferred group is examined first:

1. Read the tag at `0x54`.
2. Only if that tag is eligible, read the tag at `0x60`.
3. Select the preferred group only if both tags are eligible.

When the preferred group is selected:

- Write the derived value to `0x64`.
- Then read the word at `0x80` and write that word to `0x58`.
- Do not inspect the fallback-group tags.

If the preferred group is not selected, examine the fallback group:

1. Read the tag at `0x6C`.
2. Only if that tag is eligible, read the tag at `0x78`.
3. Select the fallback group only if both tags are eligible.

When the fallback group is selected:

- Write the derived value to `0x7C`.
- Then read the word at `0x80` and write that word to `0x70`.

If neither group is selected, none of these four destination fields is directly written.

This ordering establishes both priority and short-circuit access behavior. If both groups would qualify, only the preferred group is updated. A failed first tag prevents reading the second tag in that same group.

### 4. Mark the attachment and invoke processing

For every nonnull attachment, regardless of group eligibility:

1. Read the 32-bit word at attached offset `0x48`.
2. Write it back with bit `0x20` set, preserving its other bits at that operation.
3. Call F-6d16652f2d03 with:
   - The retained attached-object address.
   - The address of its embedded region beginning at `0x80`.

The second argument is an interior address, not the word stored at `0x80`. It is passed even when no eligible group was found.

The visible flag operation is an ordinary read-modify-write, not an established atomic operation. Effects of F-6d16652f2d03 are unknown; in particular, the function does not visibly restore fields that dependency changes.

### 5. Conditionally notify through the related object

After F-6d16652f2d03 returns, the function examines the attached object’s current state:

- First, read the related-object pointer at `0x34`.
- If it is null, skip notification without reading the condition at `0xA0`.
- Otherwise, inspect the low eight bits of the word at `0xA0`.
- Notify only when those low eight bits are zero.

Higher bits of the word at `0xA0` do not affect this condition. For example, `0x00000100` permits notification, while `0x00000001` does not.

For notification, the function obtains an indirect target through the related object’s dispatch table at slot offset `0x128` and calls it. The supplied expression shows no explicit arguments. It does not establish a hidden receiver, register arguments, or any complete ABI-level signature.

The related-object field is fetched for the null test and again when resolving the call target. No snapshot guarantee is established for external mutation between those accesses.

### 6. Obtain a result object and mark it

Whether notification was skipped or completed, the function next:

1. Resolves an indirect target through the retained attached object’s dispatch table at slot offset `0x5C`.
2. Calls that target with the attached-object address as an explicit argument.
3. Treats its returned value as an address.
4. Reads the 32-bit word at returned-address offset `0x48`.
5. Writes that word back with bit `0x40` set.

There is no null check on the returned address. The supplied evidence establishes no recovery path for an invalid return value.

The dispatch table used here is read after the processing call and any notification. Changes made by those calls can therefore affect target resolution.

## Side effects

The function’s direct writes are limited to:

- The two fields of at most one selected group.
- The attached object’s flag word at `0x48`.
- The final returned object’s flag word at `0x48`.

These are direct-write locations, not a complete inventory of transitive effects. Any dependency may have additional effects that are not described by the supplied body.

The observable call order on a completed nonnull-attachment path is:

1. F-870c4fcf7295.
2. F-6d16652f2d03.
3. Optional notification call.
4. Final result-producing indirect call.

The function visibly performs no allocation, deallocation, reference-count adjustment, or ownership transfer. This absence does not prove that its callees do none of those things.

Safe completion requires that the retained attached-object address remain usable through its later accesses. No callee preservation or lifetime guarantee is established here.

Aliasing is not prohibited by visible checks. For example, the final returned object may be the attached object itself. In that case, the last operation sets `0x40` in the word currently at attached offset `0x48`; whether the earlier `0x20` remains set depends on intervening effects.

No explicit error result, rollback, or transactional behavior is present. An interrupted execution may leave earlier writes committed without completing later calls or writes.

## Dependencies

| Opaque ID | Established invocation contract | Not established |
|---|---|---|
| F-870c4fcf7295 | Receives the selected arithmetic expression and `1,000`; returns the value subsequently adjusted | Numeric algorithm, rounding, overflow handling, global effects, ownership effects |
| F-6d16652f2d03 | Receives the attached address and its interior address at `0x80`; runs after the attached flag update | Interpretation of either argument, mutations, lifetime effects, global preservation |
| F-INDIRECT-01 | Resolved through the related object’s dispatch slot `0x128`; called only under the notification condition | Concrete identity, full signature, hidden arguments, ownership and state effects |
| F-INDIRECT-02 | Resolved through the attached object’s dispatch slot `0x5C`; receives that object and returns an address | Concrete identity, allocation or ownership policy, validity guarantee, other effects |

The indirect IDs label call sites, not known fixed target implementations.

## Acceptance examples

These are conditional static examples, not executed tests. They assume valid memory, callable targets, normal returns, and representable arithmetic.

### Null attachment still permits conversion effects

Given mode `0x03` and arithmetic input `76,000`, F-870c4fcf7295 receives `0` and `1,000`. If it returns `7`, the derived value is `25`.

If the attachment field is null when read afterward, the function returns without any attached-object processing. The conversion call must not be omitted merely because the attachment is null.

### Preferred group wins

Given:

- A derived value of `25`.
- Preferred tags `0x80000002` and `0x80000022`.
- A source word of `0x12345678` at `0x80`.

The function first writes `25` to `0x64`, then copies `0x12345678` to `0x58`. It does not inspect fallback tags, even if that group would also qualify.

It subsequently performs the attached flag update and processing call.

### Fallback selection after a short-circuit failure

If the tag at `0x54` is `0x80000000`, the preferred group fails without reading `0x60`.

If fallback tags at `0x6C` and `0x78` are both eligible, the function writes the derived value to `0x7C`, then copies the current word from `0x80` to `0x70`.

### No eligible group still triggers processing

If both groups fail their tag checks, the group destination fields remain directly untouched. Nevertheless, bit `0x20` is set at attached offset `0x48`, F-6d16652f2d03 is called, and the later notification and final-dispatch logic remains applicable.

### Post-processing state controls notification

Suppose F-6d16652f2d03 leaves a nonnull related-object pointer and the word `0x00000100` at `0xA0`. Notification occurs because its low eight bits are zero.

If it instead leaves `0x00000101`, notification is skipped. If it leaves the related-object pointer null, the `0xA0` condition is not read.

### Final flag update preserves current other bits

If F-INDIRECT-02 returns a valid object whose word at `0x48` is `0x00000005`, the final direct write stores `0x00000045`.

If the word is already `0x00000045`, its value remains unchanged, although the read-modify-write still occurs.

## Uncertainties

The following are **documented_only**:

- The conversion dependency’s actual numerical behavior.
- Signed-overflow behavior outside representable arithmetic.
- Domain meanings and units of all fields.
- Hidden arguments and complete signatures of indirect calls.
- Callee mutation of memory or globals.
- Ownership, allocation, release, and lifetime behavior of callees.
- Synchronization requirements and concurrent-mutation behavior.
- Fault handling and nonlocal control transfers.
- Additional aliasing constraints imposed by the wider system.

The sequential ordering described here does not establish a cross-thread memory-order guarantee or exact machine-instruction scheduling.

## Evidence

Established static evidence consists solely of the supplied function representation and dependency-ID map. It supports the branch conditions, access widths, field offsets, direct writes, explicit argument relationships, short-circuit reads, and call ordering described above.

“Inferred intent” is limited to the characterization of a conversion, alternative update groups, processing, and notification. Those labels do not establish application semantics.

No implementation access, external dependency inspection, or runtime testing has been performed. The acceptance examples are deductions from the supplied static evidence under their stated conditions.
