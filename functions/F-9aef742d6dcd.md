<!-- Trusted provenance: F-9aef742d6dcd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9aef742d6dcd

## Purpose

F-9aef742d6dcd conditionally invokes a helper and coordinates callbacks obtained through a shared dispatch-table reference.

For a nonzero object input, the established static sequence is:

1. Read a 32-bit field at object offset `+0x04`.
2. Invoke F-5990a26eaaa3 if that field is nonzero.
3. Read the shared dispatch reference and retain this first observed table base.
4. Read the shared dispatch reference again and use this second observed base to locate the first optional callback at offset `+0x30`.
5. Invoke that optional callback if its observed pointer is non-null.
6. Read and invoke the mandatory callback at offset `+0x18` in the **retained first table**.
7. Read offset `+0x30` in the **retained first table** and invoke its callback if non-null.

The two shared-reference reads have different uses. No dependency invocation intervenes between them, but the supplied representation does not establish that they observe the same value. Consequently, the first optional callback can come from a different table than the mandatory and final optional callbacks.

The optional invocations might represent entry and exit notifications. That is **inferred intent**, not an established contract. No object type, ownership operation, synchronization protocol, or resource lifecycle is established.

## Interface

### Explicit inputs

The supplied representation exposes four explicit input positions:

| Position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | Zero sentinel or object address. |
| Second | 32 bits | No explicit use in this body. |
| Third | 32 bits | No explicit use in this body. |
| Fourth | 32 bits | Passed to the first optional callback, when invoked. |

The first input is represented as a signed integer but is tested only for equality with zero and otherwise used as an address. There is no signed ordering test.

The second and third inputs are not explicitly inspected or forwarded. Their lack of explicit use does not establish their original ABI roles or exclude their presence in machine state accessible to dependencies.

An additional incoming machine-state value appears as the fifth argument of the first optional invocation. Its origin and interpretation are unresolved. It is not established as a constant, an initialized value, or an ordinary fifth public input.

### Output and completion

No return value is established. An ordinarily completed path returns control without a specified result.

Dependency return values are not explicitly consumed. If a dependency faults, does not return, or transfers control elsewhere, subsequent operations are not guaranteed to occur. The representation supplies no handled-error result.

### Object layout

Only one object field is directly inspected:

| Byte offset | Width | Established interpretation |
|---|---:|---|
| `+0x04` | 32 bits | Zero skips F-5990a26eaaa3; every nonzero bit pattern invokes it. |

A zero first input causes no object-field read. No other object field is directly read by this body.

### Shared reference and dispatch layouts

The shared reference is observed twice on a nonzero-object path that reaches dispatch processing:

| Observation | Use |
|---|---|
| First | Retained base for the later `+0x18` and final `+0x30` reads. |
| Second | Base used to locate the first optional `+0x30` slot. |

Each observed base refers to storage with whichever slots are subsequently accessed through that base:

| Byte offset | Contents | Invocation rule |
|---|---|---|
| `+0x18` | Function pointer | Read through the retained first base and invoked without an explicit null check. |
| `+0x30` | Function pointer | First read through the second observed base; later read through the retained first base. Each observed pointer is independently checked against null. |

The representation is consistent with 32-bit addresses and function-pointer slots, but the executable architecture, pointer encoding, alignment requirements, and ABI are not independently established. The offsets are byte offsets, not slot indices. No complete table size is supplied.

### Preconditions

For valid execution, each access and invocation must be valid when reached:

- The nonzero object input must permit the 32-bit read at `+0x04`.
- The shared reference must be readable for both observations.
- The second observed table must permit the first `+0x30` slot read.
- A non-null first optional target must support its represented invocation.
- After that optional invocation returns or is skipped, the retained first table must permit the `+0x18` read and contain a callable target there.
- After the mandatory invocation returns, the retained first table must permit the final `+0x30` read.
- A non-null final optional target must support its represented invocation.

These are validity requirements, not checks performed by the function. In particular, neither observed table base is explicitly checked for null before its relevant slot access.

## Behavior

### Zero-object path

If the first input is zero, the function returns immediately.

This path performs no object-field read, no shared-reference read, and no dependency invocation. The remaining explicit inputs do not affect this branch.

### Object predicate and conditional helper

For a nonzero first input, the function reads the 32-bit field at object offset `+0x04`.

A zero field skips F-5990a26eaaa3. Any nonzero field invokes F-5990a26eaaa3, including values whose high bit is set.

The helper invocation shows no explicit arguments. Its calling convention and access to ambient state are unknown.

Both shared-reference observations occur after this conditional helper step. If the helper changes the shared reference and returns, those observations occur in the resulting state. The function does not visibly reread the object field or obtain a replacement object from the helper’s return value.

The original first input is the object value explicitly supplied to the subsequent first optional and mandatory invocations.

### Two shared-reference observations

The function first reads the shared reference and retains that observed base.

It then reads the shared reference again. This second observed base is used to read the first optional function pointer at `+0x30`.

There is no dependency invocation between these two shared-reference reads. Nevertheless, they are distinct observations in the supplied representation. No synchronization, immutability, atomicity, or other evidence permits a contract to collapse them into one.

For clarity:

- The **retained table** means the table selected by the first shared-reference observation.
- The **initial optional table** means the table selected by the second shared-reference observation.

These names denote roles, not necessarily distinct storage. The observed bases may be equal or different. The initial optional table’s base is not subsequently used for the mandatory or final optional slot reads unless it equals the retained base.

### First optional invocation

The function reads the pointer at `+0x30` in the initial optional table.

If that pointer is null, this invocation is skipped. Otherwise, the observed target is invoked with the following visible argument sequence:

| Argument position | Visible value |
|---|---|
| First | Original object input |
| Second | Integer zero |
| Third | The same function-pointer value selected as this invocation’s target |
| Fourth | Original fourth explicit input |
| Fifth | Unresolved incoming machine-state value |

The third argument is the pointer already obtained by this slot read. It is not obtained by rereading either the shared reference or the slot.

The supplied static representation establishes this argument sequence. Its full machine-level interpretation, especially the fifth argument, remains dependent on unresolved ABI information.

### Mandatory invocation

After the first optional target returns, or immediately if it was skipped, the function reads the function pointer at `+0x18` in the **retained table**.

It invokes that target with the original object input as the sole explicitly shown argument.

There is no explicit null check on this target. A null or otherwise invalid pointer does not produce an established safe no-op or handled-error result.

The mandatory slot is read after the first optional invocation. A first optional callback can therefore affect target selection by modifying this slot in the retained table, even when that callback was selected from a different table.

Replacing only the shared reference during the callback does not redirect this read.

### Final optional invocation

After the mandatory invocation returns, the function reads the function pointer at `+0x30` in the **retained table**.

If it is null, the function returns without a final optional invocation. Otherwise, the target is invoked or transferred to with two explicitly shown arguments: integer zero followed by integer one.

This final slot access is not necessarily a reread of the same memory location used for the first optional selection. It accesses the same offset, but the first optional selection used the second observed base, while this selection uses the retained first base.

Even when both bases are equal, the two target values need not match: intervening dependencies may change the slot.

The static representation explicitly leaves the final indirect transfer uncertain. Target selection, the null check, and the visible argument pair are established. Whether the machine-level operation is an ordinary call or a tail transfer is **documented_only**.

### Mutation boundaries

There are two shared-reference reads before the first optional invocation. No later shared-reference reload is shown.

Accordingly:

- A change before either observation can affect that observation.
- If the observations differ, initial optional selection and subsequent selections use different table bases.
- A callback that changes only the shared reference does not redirect the mandatory or final slot reads.
- A callback that modifies the retained table can affect a later slot read through that table.
- After the first optional target has been selected, changing its source slot does not change the already selected target or its self-pointer argument.

Only a base value is retained; table contents are not snapshotted. No dependency is established to preserve either table, the object, or the shared reference.

### Widths and arithmetic

The object-field predicate is a 32-bit comparison against zero. Signedness does not distinguish among nonzero values for this branch.

The visible address calculations use fixed byte offsets `+0x04`, `+0x18`, and `+0x30`. There is no caller-controlled index, multiplication, or length computation.

No bounds, alignment, overflow, or address-wrap check is shown. The representation does not establish portable behavior for arithmetic that fails to designate valid storage. A negative represented first input is not explicitly rejected; as with any nonzero input, valid access remains a precondition.

## Side effects

For a nonzero object input, the direct observable sequence represented by the body is:

1. Read the object’s 32-bit field at `+0x04`.
2. Conditionally invoke F-5990a26eaaa3.
3. Read the shared dispatch reference and retain its observed base.
4. Read the shared dispatch reference a second time.
5. Read `+0x30` through the second observed base.
6. Invoke that target if non-null.
7. Read `+0x18` through the retained first base.
8. Invoke the mandatory target.
9. Read `+0x30` through the retained first base.
10. Invoke or transfer to that target if non-null.

This is the explicit access and invocation order in the supplied representation, not a hardware memory-ordering guarantee.

No direct object, table, or shared-reference write is shown. No direct allocation, release, retention, reference-count update, logging, or error-state write is established.

Dependencies may perform such actions. Passing an object address does not establish borrowing, ownership transfer, retention, destruction, or continued validity. The function has no visible cleanup or rollback path if a dependency invalidates required storage or fails to return.

## Dependencies

### F-5990a26eaaa3

Invoked when the single observed object field is nonzero.

No explicit arguments are shown. Its implementation, ABI, result, ownership effects, and global mutations are unknown. In particular, it must not be assumed to preserve the shared dispatch reference or object storage.

### Dynamically selected targets

No opaque function IDs are supplied for the dynamic targets. Their unresolved dependency roles are:

- Initial optional target: read from `+0x30` through the second shared-reference observation.
- Mandatory target: read from `+0x18` through the retained first observation.
- Final optional target: read from `+0x30` through the retained first observation.

These roles do not establish distinct function identities. Any two callable observations may resolve to the same function. The optional targets may independently be null.

The two optional invocations have different visible argument lists. Use of the same offset does not prove signature compatibility or a common semantic protocol.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Except where invalidity is the subject, required storage and targets are assumed valid and invoked dependencies return as needed.

1. **Zero object input.**  
   A zero first input returns without reading object memory, observing the shared reference, or invoking dependencies.

2. **No helper and no optional invocations.**  
   With a zero object field, a null `+0x30` pointer in the second observed table at the first slot read, and a null `+0x30` pointer in the retained table at the final slot read, only the mandatory target is invoked. It receives the original object input.

3. **High-bit object field.**  
   A field value of `0x80000000` invokes F-5990a26eaaa3. Its signed interpretation does not suppress the call.

4. **Helper changes the shared reference.**  
   If F-5990a26eaaa3 changes the shared reference before returning, both reference observations occur afterward. If nothing changes the reference between those observations, both select the helper-established table. No table base was captured before the helper.

5. **Different shared-reference observations.**  
   If the first observation selects table A and the second selects table B, the first optional target comes from B at `+0x30`. The mandatory target comes from A at `+0x18`, and the final optional target comes from A at `+0x30`. Acceptance must not substitute either table for the other.

6. **First optional invocation receives its selected pointer.**  
   If the first optional slot yields target P, the represented invocation supplies the original object, zero, P, the fourth explicit input, and the unresolved incoming value, in that order. The third argument remains P even if the source slot later changes.

7. **Optional callback changes the retained mandatory slot.**  
   A first optional target selected from table B may replace table A’s `+0x18` target. If A is the retained table, the mandatory invocation uses the subsequently read replacement, regardless of whether A and B are equal.

8. **Callback changes only the shared reference.**  
   If the first optional callback replaces the shared reference but leaves the retained table valid and unchanged, mandatory and final selection still use the retained table. There is no third shared-reference observation.

9. **Mandatory callback installs the final target.**  
   Even if the first optional slot observation was null, a mandatory callback that installs a callable target in the retained table at `+0x30` causes the final optional invocation with zero and one.

10. **Mandatory callback clears the final target.**  
    If the mandatory callback clears the retained table’s `+0x30` slot before the final read, the final optional invocation is skipped, whether or not the first optional invocation occurred.

11. **Invalid mandatory target.**  
    A null mandatory target has no established successful or handled-error outcome. Acceptance must not require a safe skip.

## Uncertainties

The following remain **documented_only**:

- The object’s domain meaning and the meaning of its `+0x04` field.
- Exact architecture, ABI, pointer widths, encoding, alignment, and calling conventions.
- The source and interpretation of the extra incoming argument value.
- The full signatures and implementations of dynamic targets.
- Whether the final indirect operation is a call or tail transfer.
- Any dependency’s memory ownership, allocation, release, or preservation behavior.
- Any synchronization, concurrency safety, or guarantee that the two shared-reference observations agree.
- Exception handling, fault behavior, and nonlocal control transfer.
- Any semantic relationship between the optional invocations beyond their represented selection, arguments, and order.

No mechanism causing unequal shared-reference observations is established. Equality is not established either; the contract preserves both observations without prescribing how a difference could arise.

“Initialization,” “locking,” “cleanup,” and “destruction” are not established descriptions of this function.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent evidence.

The representation establishes the zero-input branch, 32-bit object-field predicate, conditional helper invocation, two shared-reference observations, their distinct uses, later retained-base accesses, explicit argument expressions, null checks, and absence of direct writes. It also records uncertainty about the final indirect transfer.

No dependency implementation, executable, runtime trace, ABI specification, or complete memory-layout definition was supplied. No runtime tests have been performed. The acceptance examples state conditional consequences of static evidence, not observed execution results.
