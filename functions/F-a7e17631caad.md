<!-- Trusted provenance: F-a7e17631caad; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a7e17631caad

## Purpose

F-a7e17631caad dispatches between two behaviors according to an integer discriminator in its second argument.

- When the discriminator equals `0x16`, it obtains a related object through F-682adfd5a561, examines a nested record, and conditionally invokes F-3871f5a8cc5e. Every normally completing execution of this path returns `1`, including executions that do not invoke F-3871f5a8cc5e.
- For every other discriminator, it delegates to F-2972f8322b82 and normalizes that dependency’s result: zero becomes `0`; any nonzero value becomes `1`.

The result therefore does not generally indicate whether the nested record matched or whether the conditional action occurred. In particular, a result of `1` on the special path establishes only normal completion of that path.

The application meanings of the discriminators, the related objects, and the conditional action are not established by the supplied static representation.

## Interface

### Arguments

| Argument | Representation | Established use |
|---|---|---|
| First argument | Opaque four-byte value | Forwarded unchanged to F-2972f8322b82 on the delegated path or F-3871f5a8cc5e when the special-path conditions hold |
| Second argument | Integer-represented address-bearing value | Used as the base for the initial discriminator read; forwarded unchanged to F-682adfd5a561 or F-2972f8322b82 |

The wrapper does not dereference or perform arithmetic on the first argument. Its interpretation as a handle, address, identifier, or other value is unknown.

The second argument must support the initial read at byte offset `+4`. Further storage requirements depend on the selected path. Its integer representation does not establish that arbitrary integers are valid inputs.

### Result

The result is a four-byte value. On normal completion, it is exactly one of:

| Result | Condition |
|---|---|
| `0` | The initial discriminator differs from `0x16`, and F-2972f8322b82 returns zero |
| `1` | The special path completes normally, or F-2972f8322b82 returns a nonzero value |

A nonzero delegated result is not propagated unchanged. For example, a delegated result of `7` produces `1`, not `7`.

No normal error result is separately defined for invalid addresses, failed dependencies, or unsuccessful special-path comparisons.

### Widths and layouts

The supplied representation is consistent with four-byte integers and four-byte address-bearing fields. The first argument and return value are explicitly four-byte quantities. The field widths below use the recovered 32-bit integer/address model; an independently verified target ABI is not supplied.

All offsets are measured in bytes.

| Base | Offset | Accessed content |
|---|---:|---|
| Second argument | `+4` | Integer discriminator compared with `0x16` |
| Address returned by F-682adfd5a561 | `+8` | Address of a nested record |
| Nested record | `+4` | Integer discriminator compared with `0x2503` |
| Nested record | `+12` | Address of integer storage |
| Integer storage addressed through the nested record | `+0` | Integer used as the first final-comparison operand |
| Global comparison base | `+4` | Integer used as the second final-comparison operand |

The nested record’s `+12` field contains an address, not the comparison integer itself. The wrapper reads that field and then dereferences the resulting address.

The global dependency supplies a base address; the comparison integer is read at offset `+4` from that base. It is not a comparison against the base address.

No complete structure definitions, total allocation sizes, alignment guarantees, or byte order are established.

## Behavior

### Initial dispatch

The first wrapper-level memory read obtains the integer at byte offset `+4` from the second argument.

An exact match with `0x16` selects the special path. Any other value selects the delegated path. There is no preceding wrapper-level null check, bounds check, validation call, or fallback.

The discriminator is not masked, interpreted as a Boolean, or compared against a range.

### Special path

The wrapper invokes F-682adfd5a561 with:

- The original second argument.
- The constant `0xF00`.

After that dependency returns, the wrapper treats its result as an address. It reads the nested-record address at offset `+8` from that result, then reads the nested-record discriminator at offset `+4`.

If the nested discriminator differs from `0x2503`, the wrapper performs no further special-path data reads shown in the source and returns `1`. In particular:

- It does not read the nested record’s `+12` address field.
- It does not dereference the address stored there.
- It does not read the global comparison field.
- It does not invoke F-3871f5a8cc5e.

If the nested discriminator equals `0x2503`, the wrapper continues with the following data-dependent accesses:

1. Read the address at offset `+12` from the nested record.
2. Read the integer at that address.
3. Read the integer at offset `+4` through the global comparison base.
4. Compare those two loaded integers for equality.

When the integers are equal, the wrapper invokes F-3871f5a8cc5e with the original first argument. When they differ, it omits that invocation.

Both outcomes return `1` after the selected operations complete normally. Any result produced by F-3871f5a8cc5e is unused.

Neither zero nor a high-bit-set comparison value receives special treatment. The final predicate compares integer values, not their storage addresses.

### Delegated path

When the initial discriminator differs from `0x16`, the wrapper invokes F-2972f8322b82 with the original first argument followed by the original second argument.

It tests the returned value against zero:

- A zero result causes the wrapper to return `0`.
- Any nonzero result causes the wrapper to return `1`.

Under the represented signed-integer model, a negative nonzero result is normalized to `1` just like a positive nonzero result. The wrapper does not classify negative results as a separate error category.

This path performs none of the special-path nested-record or global-comparison reads. F-2972f8322b82 may itself access memory or invoke other functions; its internal behavior is not supplied.

### Ordering and arithmetic

The relevant wrapper-level order is established by the static representation:

- The input discriminator is read before either dispatch dependency is invoked.
- F-682adfd5a561 returns before the wrapper reads its result’s `+8` field.
- The nested discriminator is read before any access through the nested record’s `+12` field.
- On a matching nested discriminator, the indirect comparison integer is read before the global comparison integer.
- Both comparison integers are obtained before any conditional invocation of F-3871f5a8cc5e.

The wrapper does not capture the global comparison value at entry. F-682adfd5a561 may change the global base, its referenced storage, or other relevant state before the later reads; no preservation guarantee is established.

All branch predicates use equality tests; only the delegated result is tested against zero. No ordered or sign-based comparison is shown.

Address arithmetic consists of the fixed byte offsets `+4`, `+8`, and `+12`. No explicit overflow or bounds checks are shown. Validity requires each selected address calculation and read to identify accessible storage under the actual execution environment. The source does not establish a portable contract for wrapped address arithmetic.

### Preconditions and abnormal completion

The initial discriminator read must be valid on every execution. Additional requirements are conditional:

- On the delegated path, requirements beyond the initial read depend on F-2972f8322b82.
- On the special path, the returned object’s `+8` field and the nested record’s `+4` field must be readable.
- Only when the nested discriminator matches must the nested `+12` field, its referenced integer, and the global comparison field be readable.

The wrapper performs no explicit validation of these addresses. There is no demonstrated recovery path for an invalid read, an abnormal dependency exit, or a dependency result that cannot safely serve as the required address.

## Side effects

The wrapper contains no explicit writes to the input object, returned object, nested record, indirect integer storage, or global comparison field.

It also contains no explicit allocation, deallocation, reference-count update, ownership transfer, or resource release. Those absences do not establish that dependencies are side-effect-free or that argument-reachable storage remains unchanged.

Direct dependency calls occur as follows:

| Path or condition | Direct invocation count |
|---|---:|
| Special path: F-682adfd5a561 | Once |
| Special path with matching nested discriminator and equal comparison integers: F-3871f5a8cc5e | Once |
| Special path without both matches: F-3871f5a8cc5e | Zero |
| Delegated path: F-2972f8322b82 | Once |

These counts apply to normally progressing wrapper execution and do not count calls made internally by dependencies.

No ownership conclusion follows from the wrapper’s use of an address. In particular, the absence of a release operation does not establish that a returned object is borrowed, permanently allocated, or safe to retain.

## Dependencies

| Opaque function ID | Arguments supplied | Use of result | Unestablished behavior |
|---|---|---|---|
| F-682adfd5a561 | Original second argument, then `0xF00` | Used as a base address for the `+8` read | Result validity, lifetime, ownership, failure representation, state changes, and meaning of the constant |
| F-3871f5a8cc5e | Original first argument | No result is inspected | Purpose, argument requirements, memory effects, ownership effects, and abnormal-exit behavior |
| F-2972f8322b82 | Original first argument, then original second argument | Tested for zero and normalized to `0` or `1` | Result semantics, argument requirements, memory effects, ownership effects, and abnormal-exit behavior |

The global comparison base is a separate data dependency. Its initialization, lifetime, mutability, and relationship to the nested record are unknown. No dependency is assumed to preserve it.

## Acceptance examples

These examples are conditional contract cases, not executed tests. Each assumes that every selected read is valid and every invoked dependency returns normally.

### Delegated zero result

With an initial discriminator of `0x15` and a zero result from F-2972f8322b82:

- F-2972f8322b82 receives both original arguments in their original order.
- Neither special-path dependency is invoked directly by the wrapper.
- The wrapper returns `0`.

### Delegated negative result

With an initial discriminator of `0x17` and a delegated result of `-1` under the represented integer model:

- The wrapper returns `1`.
- It neither propagates `-1` nor selects a separate negative-result branch.

### Nested discriminator mismatch

With an initial discriminator of `0x16` and a nested discriminator of `0x2502` after F-682adfd5a561 returns:

- F-682adfd5a561 receives the original second argument and `0xF00`.
- The nested `+12` field and global comparison field are not read by the wrapper.
- F-3871f5a8cc5e is not invoked.
- The wrapper returns `1`.

An invalid address value stored in the unused `+12` field does not itself cause a wrapper dereference on this branch.

### Matching discriminators, unequal integers

With discriminators `0x16` and `0x2503`, an indirect integer of `7`, and a global comparison integer of `8`:

- The indirect integer is read before the global comparison integer.
- F-3871f5a8cc5e is not invoked.
- The wrapper returns `1`.

### Matching discriminators, equal zero integers

With both discriminators matching and both comparison integers equal to zero:

- Equality succeeds.
- F-3871f5a8cc5e receives the original first argument.
- The wrapper returns `1` after that invocation completes normally.

A zero integer value is distinct from a zero address used to locate an integer. This example does not establish that a zero address is valid.

### Matching high-bit-set integers

Under the recovered 32-bit model, if both comparison integers have the bit pattern `0x80000000`:

- Equality succeeds despite the high bit being set.
- F-3871f5a8cc5e is invoked.
- The wrapper returns `1` after normal completion.

This follows from equality comparison, not an ordered or sign-based predicate.

### Global state changed during lookup

If F-682adfd5a561 changes the global comparison state before returning, and the nested discriminator subsequently matches:

- The wrapper obtains the comparison integer through the global base after that call.
- The conditional action depends on the later loaded value, not an assumed entry-time snapshot.

## Uncertainties

The following remain **documented_only**:

- Application meanings of `0x16`, `0x2503`, and `0xF00`.
- Exact target ABI, byte order, alignment requirements, and independent confirmation of the recovered integer/address widths.
- Full object layouts and allocation sizes.
- Ownership and lifetime of all arguments, returned objects, nested records, and referenced storage.
- Dependency mutations, allocation or release behavior, failure conventions, exceptions, and nonlocal control flow.
- Global-base initialization and update rules.
- Aliasing among input, returned, nested, indirect, and global storage.
- Synchronization requirements and behavior under concurrent mutation.
- Whether relevant memory accesses have volatile or atomic semantics.

The described order is the logical order in the supplied static representation. It is not a guarantee of hardware instruction order, atomicity, or a coherent concurrent snapshot.

An **inferred intent** is that the special path recognizes a particular nested-record association before performing an action. That interpretation is not required for conformance and does not establish application-level success semantics.

## Evidence

The supplied static representation establishes:

- The initial `+4` discriminator read and equality test against `0x16`.
- The arguments supplied to each opaque dependency.
- The returned-object `+8` access and nested-record `+4` discriminator test against `0x2503`.
- Conditional access through the nested record’s `+12` address field.
- The subsequent global-base `+4` comparison read.
- Equality comparison between the two loaded integers.
- Conditional invocation of F-3871f5a8cc5e.
- Zero/nonzero normalization of F-2972f8322b82’s result.
- Return of `1` on every normally completing special-path execution.
- Absence of explicit wrapper-level memory writes and resource-management operations.

No runtime tests have been performed. Dependency implementations, external type definitions, and runtime memory observations were not supplied; they cannot establish guarantees beyond this static contract.
