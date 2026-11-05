<!-- Trusted provenance: F-be1db7c73ba5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-be1db7c73ba5

## Purpose

F-be1db7c73ba5 updates a signed control value and derives two output values from a shared unsigned base value. It stores the normalized control, constructs two intermediate gain words, and then either copies or scales the base independently for each output.

The structure is consistent with a two-channel balance or attenuation operation: a negative control changes the second gain, a positive control changes the first, and zero assigns the same global default word to both. That interpretation is inferred intent, not an established interface name or guarantee about the surrounding system.

The routine has an early-return optimization. When the requested control already equals the stored control and the third argument is zero, it performs no object writes. This comparison occurs **before** normalization, so the optimization can preserve a stored value that the update path would otherwise replace with zero.

The default gain word and comparison threshold are read from global storage. Their actual values are not supplied. In particular, this contract does not assume either is the conventional fixed-point unity value.

## Interface

### Inputs

The routine takes three inputs:

1. **State address.** An address-valued input identifying a writable state object. The supplied representation performs address arithmetic through a 32-bit integer-shaped value. The relevant fields occupy offsets `0x6c` through `0x83`, inclusive.
2. **Requested control.** A signed 32-bit integer.
3. **Recompute selector.** A 32-bit integer tested only for equality with zero. Zero permits the early return; every nonzero value prevents it. Positive and negative nonzero values have identical significance.

There is no explicit return value.

### State layout

All listed fields are four bytes wide.

| Offset | Role | Interpretation |
|---|---|---|
| `0x6c` | Shared base | Unsigned 32-bit value |
| `0x70` | First output | Unsigned 32-bit result |
| `0x74` | Second output | Unsigned 32-bit result |
| `0x78` | Stored control | Signed 32-bit value |
| `0x7c` | First gain | Stored as a 32-bit word; compared as signed |
| `0x80` | Second gain | Stored as a 32-bit word; compared as signed |

Only the five fields at `0x70` through `0x80` are explicitly written. The base field at `0x6c` is read but not written.

The state storage must remain valid throughout the operation. Its accessed fields must support the target platform’s four-byte reads and writes. The supplied evidence does not establish byte order, exact alignment requirements, or the enclosing object’s full size.

### Global inputs

Two global storage locations contribute to the result:

- **Default gain word:** a 32-bit word read at entry and retained for subsequent gain assignments.
- **Signed threshold:** a 32-bit value used in signed gain comparisons. The routine obtains one threshold value for the second output and separately evaluates the threshold for the first output.

These are data dependencies, not function dependencies. Their initialization, mutability, and numeric values are unknown.

### Preconditions and failure reporting

The routine provides no explicit validation of the state address, no status result, and no error-reporting path. Invalid or inaccessible storage is outside the established normal-operation contract.

There is no allocation, ownership transfer, or release operation in the supplied body. The caller retains responsibility for the state storage.

## Behavior

### Entry read and early return

The default gain word is read before the early-return decision.

The stored control at `0x78` is then compared with the original, unnormalized requested control. If they match and the recompute selector is zero, the routine returns immediately.

On that path:

- No state field is written.
- The base value is not read.
- Neither output is recalculated.
- Existing gains remain untouched.
- The threshold is not read by the subsequent update logic.

Thus, “no change requested” does not imply that the gains or outputs are consistent with the current base or globals. The routine does not check that consistency.

### Control normalization

If the routine does not return early, it accepts precisely the signed interval:

**−32767 through +32767, inclusive.**

Any other signed 32-bit request becomes zero.

The supplied checks use unsigned 32-bit addition and subtraction. Their combined accepted regions are:

- −32767 through 32512;
- 32513 through 32767.

The union is the single interval above. Consequently, −32768 is rejected, while −32767 is accepted. There is no behavioral discontinuity between 32512 and 32513.

An out-of-range request is reset to zero rather than saturated to a nearest endpoint. This normalization does not signal an error.

The normalized control is written to `0x78` before any gain or output write.

### Gain construction

The entry-time snapshot of the default gain word is reused wherever a default gain is required.

For a normalized control of zero:

1. The first gain receives the default word.
2. The second gain receives the default word.

For a negative normalized control:

1. The second gain receives the control plus 32767.
2. The first gain receives the default word.

For a positive normalized control:

1. The first gain receives 32767 minus the control.
2. The second gain receives the default word.

The explicitly calculated gain is always in the inclusive interval 0 through 32766. These additions and subtractions do not overflow signed 32-bit arithmetic after normalization.

The default word has no established numeric range. Although stored as a raw 32-bit word, it is subsequently interpreted as signed for comparison and as an unsigned multiplication operand when scaling occurs.

The order of gain writes differs by branch. In particular, the negative branch writes the second gain before the first gain. This is relevant to observers of intermediate state; the routine does not provide an atomic multi-field update.

### Output derivation

After both gain writes, the routine reads a threshold value and retains it for the second-output comparison. It then reads the unsigned base at `0x6c`.

The first output is determined next:

- The first gain is read and compared, as signed 32-bit, against a separately evaluated signed threshold value.
- If the gain is strictly less than that threshold, the base is multiplied by the first gain’s 32-bit bit pattern using unsigned 32-bit arithmetic. The low 32 bits of the product are retained, then shifted logically right by 15 bits.
- Otherwise, the first output receives the unchanged base.
- The first output is written to `0x70`.

The second output is then determined:

- The second gain is read and compared, as signed 32-bit, against the earlier retained threshold.
- If the gain is strictly less, it is multiplied by the same captured base using unsigned 32-bit arithmetic, retaining the low 32 bits before a logical right shift by 15.
- Otherwise, the second output receives the unchanged base.
- The second output is written to `0x74`.

The second output uses the original captured base, not the first output.

Each scaling branch reads its gain again for the multiplication. With stable storage, that read sees the same word as its comparison. The routine itself does not establish synchronization that would guarantee this in the presence of external mutation.

### Arithmetic boundaries

For stable inputs, the scaling operation is mathematically described as:

**the low 32 bits of the unsigned product, divided by 32768 with truncation downward.**

The truncation to 32 bits happens **before** division. It is not equivalent in general to forming a full-width product and then dividing.

Important consequences:

- Multiplication overflow wraps modulo 2³²; it is not saturated or reported.
- Right shifting is logical, not sign-extending.
- A scaled result lies between 0 and 131071, inclusive.
- An unscaled result can be any unsigned 32-bit base value.
- A gain equal to the threshold takes the unscaled branch.
- A gain greater than the threshold also takes the unscaled branch.
- A zero gain produces zero only when it actually enters the scaling branch.
- A default word with its high bit set compares as negative, but participates in multiplication through its unsigned bit pattern.

No clipping, rounding-to-nearest, or post-scaling range correction is present.

## Side effects

On an update path, the established state-write order is:

1. Stored normalized control.
2. First gain, then second gain for zero or positive controls; second gain, then first gain for negative controls.
3. First output.
4. Second output.

The default-word read precedes all these writes. The retained threshold read and base read occur after gain construction and before the first output write.

The routine does not explicitly modify either global input. It has no visible function calls, allocation, deallocation, logging, or error reporting.

No transaction or rollback mechanism is shown. The evidence does not establish atomicity, thread safety, reentrancy guarantees, or recovery from a failed memory access. Stable-state examples below assume no concurrent modification and no aliasing between the state fields and the global input locations.

## Dependencies

No other opaque function ID is invoked by F-be1db7c73ba5 in the supplied body.

Its external dependencies are:

- Caller-provided readable and writable state storage.
- A readable 32-bit default gain word.
- Readable signed 32-bit threshold storage.
- The represented 32-bit integer and memory-access semantics.

There is no basis for assigning memory ownership, global-preservation guarantees, or other behavioral obligations to unknown callees: no such callees appear here.

## Acceptance examples

These examples are conditional specifications, not runtime observations.

### Matching request skips recomputation

Given stored control 123, requested control 123, and a zero recompute selector:

- The routine reads the default gain word and stored control.
- It leaves all state fields unchanged.
- It does not refresh outputs even if the base has changed since an earlier invocation.

### Early return can preserve an out-of-range control

Given stored control −32768, requested control −32768, and a zero selector, the state remains unchanged.

With the same stored control and request but a nonzero selector, the update path instead stores control zero and assigns the captured default word to both gains.

### Normalization boundaries

For any invocation that reaches the update path:

| Request | Stored normalized control |
|---|---:|
| −32769 | 0 |
| −32768 | 0 |
| −32767 | −32767 |
| 32512 | 32512 |
| 32513 | 32513 |
| 32767 | 32767 |
| 32768 | 0 |

The outputs depend additionally on the base, default word, and threshold.

### Conventional unity configuration

Assume stable default word 32768, stable threshold 32768, and base 1000:

| Updated control | First gain | Second gain | First output | Second output |
|---|---:|---:|---:|---:|
| 0 | 32768 | 32768 | 1000 | 1000 |
| 32767 | 0 | 32768 | 0 | 1000 |
| −32767 | 32768 | 0 | 1000 | 0 |
| 16383 | 16384 | 32768 | 500 | 1000 |
| −16383 | 32768 | 16384 | 1000 | 500 |

The default gains pass the base through because they equal the threshold, not because their multiplication is evaluated.

### Equality bypasses multiplication

Assume the update constructs a first gain of 100, the first comparison threshold is 100, and the base is 777.

The first output is 777. Equality does not select scaling.

### A zero gain need not mute

Assume control 32767 and a stable threshold of zero. The first gain becomes zero, but zero is not less than zero. The first output therefore equals the base.

### Product truncation precedes shifting

Assume control 16383, producing first gain 16384; threshold 32768; and base 262144.

The mathematical product is 4294967296. Its low 32 bits are zero, so the first output is zero. A full-width multiply followed by division would produce 131072 and would not match this behavior.

## Uncertainties

### Established static evidence

The supplied body establishes field offsets, four-byte accesses, the early-return placement, the accepted control interval, branch-dependent write ordering, unsigned multiplication, and strict signed threshold comparisons.

### Inferred intent

The paired gains and outputs suggest balance or attenuation control. The shift by 15 suggests a fixed-point scaling convention. Neither inference establishes channel naming, units, permissible base range, or the actual global values.

### Documented-only unknowns

The following remain **documented_only**:

- Actual values and initialization rules for both global inputs.
- Whether those globals can change during an invocation.
- The surrounding object type and full allocation size.
- Target byte order and exact alignment requirements.
- Whether state storage can alias global input storage.
- Permitted concurrency and memory-observation rules.
- Platform behavior for invalid addresses or interrupted writes.
- Whether callers enforce a base range that prevents product wraparound.
- Whether a broader API intends nonzero recompute selectors to have additional meaning.

The ordering stated here follows the supplied static representation. It does not establish machine-level barriers or exact hardware access behavior.

## Evidence

This contract is based solely on static inspection of the supplied body for F-be1db7c73ba5. No implementation files, external documentation, dependency bodies, or runtime observations were available.

The normalization interval and arithmetic examples were derived from the displayed 32-bit operations. The global values used in acceptance examples are explicit conditions, not claims about deployed configuration.

No runtime tests have been performed.
