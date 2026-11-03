<!-- Trusted provenance: F-71233cee1dae; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-71233cee1dae

## Purpose

F-71233cee1dae selects one of two dependency argument pairs from a caller-associated structure, invokes F-c45cb3f8e7fd exactly once on every normally completing path, and then conditionally halves a caller-supplied unsigned 32-bit word.

The selection depends on a 32-bit field in a control structure and, when that field is nonzero, a byte reached through two additional pointers. The fourth input does not affect the selected path or the dependency arguments.

The halving occurs **after** the dependency returns. Consequently, it applies to the word present at that time, not necessarily the word supplied on entry.

Established behavior is limited to the visible routine. The purpose of the selected objects, the meaning of the selector fields, and the dependency’s effects are unknown. Descriptions such as “primary” and “alternate” below identify branches, not application-level roles.

## Interface

The interface has four inputs and no explicit result value.

| Input | Width and interpretation | Contract |
|---|---|---|
| Associated-object address | 32-bit address-bearing value | Identifies an object whose field at byte offset `0x20` contains the control-structure address. |
| Opaque payload | 32 bits | Forwarded unchanged as the dependency’s third argument. Its signedness and meaning are not established. |
| Optional result-word address | Address of an unsigned 32-bit word, or null | Forwarded unchanged as the dependency’s fourth argument. If non-null and the dependency returns normally, its then-current word is read, logically shifted right by one, and written back. |
| Nominal selection input | Unsigned 32 bits | Has no observable influence in the supplied routine. It is not forwarded to the dependency. |

The address and integer widths above follow the supplied 32-bit representation. They do not establish portability to an environment with different pointer widths.

### Relevant memory layout

All offsets are in bytes. The names in this table are descriptive labels for this contract.

| Object | Offset | Accessed representation | Role |
|---|---:|---|---|
| Associated object | `0x20` | 32-bit pointer | Locates the control structure. |
| Control structure | `0x00` | 32-bit pointer | Locates the base object. |
| Control structure | `0x2C` | 32-bit value | Dependency second argument on the primary path. |
| Control structure | `0x30` | 32-bit value | Alternate selector; also the dependency second argument on the alternate path. |
| Base object | `0x08` | 32-bit pointer | Locates the flag-bearing object when the alternate selector is nonzero. |
| Flag-bearing object | `0xB0F` | Unsigned 8-bit value | Determines whether a nonzero alternate selector activates the alternate path. |
| Base object | `0x118` | Address only | Primary-path dependency first argument. |
| Base object | `0x170` | Address only | Alternate-path dependency first argument. |
| Result word | `0x00` | Unsigned 32-bit value | Post-dependency read and write, if its address is non-null. |

The routine computes addresses at `0x118` and `0x170`; it does not itself dereference those interior addresses. Their contents and the extent of access performed by the dependency are unknown.

### Preconditions

For the routine’s own visible accesses:

- The associated-object address must permit reading its pointer field at `0x20`.
- The resulting control-structure address must permit reading the alternate selector at `0x30`.
- The primary path additionally requires readable control fields at `0x00` and `0x2C`.
- A nonzero alternate selector requires a readable base pointer at control offset `0x00`, a readable pointer at base offset `0x08`, and a readable byte at flag-object offset `0xB0F`.
- If the result-word address is non-null, it must remain readable and writable for a 32-bit access after the dependency returns.

These are necessary conditions for the visible operations, not a complete validity contract for F-c45cb3f8e7fd. That dependency may impose additional requirements.

## Behavior

### Selection and dependency invocation

The routine first reads the control-structure pointer from associated-object offset `0x20`. It then reads the 32-bit alternate selector from control offset `0x30`.

That selector determines whether the extra flag lookup occurs:

- **Selector equals zero:** the flag-bearing pointer chain is not traversed. The primary path is selected.
- **Selector is nonzero:** the routine reads the base pointer from control offset `0x00`, reads the flag-object pointer from base offset `0x08`, and reads one unsigned byte from flag-object offset `0xB0F`.
  - A zero byte selects the primary path.
  - Any nonzero byte selects the alternate path.

The nominal selection input cannot override this decision. When the selector is zero, the selector condition alone prevents the alternate path. When the selector is nonzero, the fetched byte supplies the flag value instead of the nominal selection input.

The dependency arguments are:

| Argument position | Primary path | Alternate path |
|---|---|---|
| First | Base-object address plus `0x118` | Base-object address plus `0x170` |
| Second | Value read from control offset `0x2C` | Previously read value from control offset `0x30` |
| Third | Opaque payload, unchanged | Opaque payload, unchanged |
| Fourth | Optional result-word address, unchanged | Optional result-word address, unchanged |

The alternate offset is `0x170` bytes: it is formed from an offset of `0x5C` elements, each four bytes wide. It is not a byte offset of `0x5C`.

On the primary path, the base pointer is obtained from control offset `0x00` for the call. If a nonzero selector led to a zero flag, the supplied expression reads that field again rather than explicitly reusing the base pointer fetched for flag inspection. When the selector is zero, no earlier base-pointer read for flag inspection occurs.

The alternate path uses the base pointer already obtained during flag inspection and the selector value already read before that inspection.

The supplied representation does not establish a separate ordering between the primary path’s base-pointer read and its read of the second argument at control offset `0x2C`. Both supply the dependency invocation. A contract requiring one of those two reads to precede the other would exceed the evidence.

### Post-dependency result adjustment

After F-c45cb3f8e7fd returns normally:

- If the result-word address is null, the routine performs no result-word read or write.
- Otherwise, the routine reads the unsigned 32-bit word currently at that address, shifts it logically right by one bit, and writes the 32-bit result back to the same address.

The mathematical result is the floor of the current unsigned word divided by two. The high bit is filled with zero, and the original low bit is discarded.

This operation uses a fresh post-call read. The routine does not visibly cache the word before invoking the dependency. If the dependency changes the word, the changed value is halved. If it invalidates the storage, the subsequent access is not protected.

There is no visible retry, range check, saturation, rollback, or special handling for zero or maximum-width values. No dependency return value is used.

### Normal and abnormal completion

On every normally completing path, exactly one dependency invocation occurs, followed by the conditional result adjustment.

The routine contains no explicit error result or recovery path. An invalid pointer, inaccessible field, or failure inside the dependency has no visible local handling. If the dependency does not return normally, the post-call adjustment is not established to occur.

## Side effects

The only direct write established for this routine is the post-call replacement of the optional result word. No direct write to the associated object, control structure, base object, or flag-bearing object is visible.

F-c45cb3f8e7fd may have additional effects. Its implementation is unavailable, so this contract does not assume that it:

- Preserves global state.
- Leaves the supplied structures unchanged.
- Preserves the result word.
- Retains or releases any passed storage.
- Returns normally.
- Restricts its accesses to the visible offsets described here.

The routine performs no visible allocation or release operation of its own. Passing an interior address to the dependency does not establish ownership transfer.

Aliasing is not checked. The optional result word may overlap storage involved in selection. If it does, the final write modifies that overlapping storage after the dependency returns. Selection is not repeated afterward, so that final write cannot change which dependency invocation has already occurred.

The evidence provides no synchronization, atomicity, or concurrent-mutation guarantee. The access distinctions above describe the supplied expression structure, not a supported concurrent-memory protocol.

## Dependencies

### F-c45cb3f8e7fd

This is the sole visible dependency.

It receives four arguments:

1. An interior address selected from one of two offsets in the base object.
2. A 32-bit value selected from the control structure.
3. The unchanged opaque payload.
4. The unchanged optional result-word address.

It is invoked even when the result-word address is null. Whether null is valid for the dependency is **documented_only**: the wrapper forwards it, but the dependency’s contract is unavailable.

The routine does not use a return value from this dependency. Its normal return is the sequencing boundary before the optional result-word transformation.

The dependency’s ownership rules, global effects, error behavior, and interpretation of the first two arguments are **documented_only**.

## Acceptance examples

These examples are conditional contracts, not reports of executed tests. Unless stated otherwise, all required accesses are valid, address additions identify valid dependency inputs, and F-c45cb3f8e7fd returns normally.

### Zero selector bypasses flag inspection

Given an alternate selector of zero:

- The primary path is selected.
- The dependency receives the base address plus `0x118` and the value at control offset `0x2C`.
- The pointer at base offset `0x08` and the byte at flag-object offset `0xB0F` are not read by this routine.
- A nonzero nominal selection input does not change the outcome.

If the dependency leaves a non-null result word equal to `9`, the final word is `4`.

### Nonzero selector and zero flag select the primary path

Given an alternate selector of `7` and a flag byte of zero:

- The flag lookup occurs.
- The primary path is selected.
- The dependency’s second argument comes from control offset `0x2C`, not from the selector value `7`.
- The primary-path base pointer is read from control offset `0x00` for that invocation.

Changing only the nominal selection input does not change this behavior.

### Any nonzero flag enables the alternate path

Given an alternate selector of `7` and a flag byte of `0x80`:

- The alternate path is selected.
- The dependency’s first argument is the flag-lookup base address plus `0x170`.
- Its second argument is `7`.
- Control offset `0x2C` is not read by this routine.

The byte need not equal `1`; `0x80` is nonzero and therefore sufficient.

### Unsigned result transformation

If the post-dependency result word has one of the following values, the final stored value is:

| Post-dependency word | Final word |
|---:|---:|
| `0x00000000` | `0x00000000` |
| `0x00000001` | `0x00000000` |
| `0x00000003` | `0x00000001` |
| `0x80000000` | `0x40000000` |
| `0xFFFFFFFF` | `0x7FFFFFFF` |

These outcomes require a logical unsigned shift, not sign-preserving division or an arithmetic right shift.

### Dependency changes the result word

Given an entry word of `100`, if the dependency replaces it with `11`, the routine stores `5`. It does not store `50`.

### Null result-word address

Given a null result-word address, the dependency still receives null as its fourth argument. If it returns normally, this routine performs no result-word access.

This example does not establish that the dependency accepts null safely.

### Invalid flag chain cannot be bypassed by the fourth input

Given a nonzero alternate selector, an unreadable flag-object pointer is encountered during flag lookup regardless of the nominal selection input. The routine has no visible fallback that skips that lookup based on the fourth input.

## Uncertainties

### Established static evidence

The supplied routine establishes:

- The field offsets and widths listed above.
- Conditional flag lookup for a nonzero selector.
- Primary selection for a zero selector or zero flag.
- Alternate selection for a nonzero selector and nonzero flag.
- One dependency invocation on each normally completing path.
- Unchanged forwarding of the payload and result-word address.
- A post-call unsigned 32-bit right shift for a non-null result-word address.
- No influence from the nominal selection input.

### Inferred intent

Selecting between two related embedded regions is a plausible interpretation of the two interior addresses. The selector may represent availability or activation, and the final shift may represent scaling. These interpretations are not required behavior and must not be used to infer additional validation or ownership rules.

### documented_only

The following remain unknown:

- Application-level meanings of all structure fields.
- Required sizes and contents of the regions passed to the dependency.
- Dependency behavior, including memory ownership and global-state effects.
- Supported aliasing arrangements and concurrent access.
- Endianness and platform-specific unaligned-access behavior.
- Behavior for overflowing or invalid address arithmetic.
- Any intended semantic role for the fourth input outside this routine.

No portable wraparound, exception type, or recovery guarantee is asserted for invalid address calculations or accesses.

## Evidence

This contract is derived solely from static inspection of the supplied routine representation and its mapped dependency identity. No dependency body, caller behavior, runtime trace, or external type definition was available.

The supplied representation supports the selection rules, access widths, offsets, forwarding behavior, and post-call transformation described here. It does not establish broader application semantics or dependency guarantees.

No runtime tests have been performed. The acceptance examples are conditional predictions from static evidence, not observed execution results.
