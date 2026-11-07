<!-- Trusted provenance: F-74cfcbeb5ec0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-74cfcbeb5ec0

## Purpose

F-74cfcbeb5ec0 derives position- and extent-related quantities from a descriptor, prepares caller-supplied scratch storage through dependencies, and makes one or two submission calls.

A nonzero starting-boundary result enables an initial submission. If that submission returns a nonzero result, the function returns immediately. Otherwise, execution reaches a final preparation and submission stage. The final submission’s result is ignored.

Established behavior consists of field accesses, arithmetic, dependency-call order, conditional control flow, and direct marker-byte writes. The meanings of the submissions, the scratch-buffer representation, and the arithmetic dependency are not established.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

The function accepts four inputs and has no explicit return value.

| Input | Representation | Established use |
|---|---|---|
| Context | Address represented by a 32-bit integer-shaped argument | Supplies an adjustment flag and, conditionally, an adjustment operand. |
| State | Pointer to a signed 32-bit word | Its initial word is captured; the original pointer is passed to submissions. |
| Descriptor | Address represented by a 32-bit integer-shaped argument | Supplies a mode, unsigned position, and five signed extent contributions. |
| Scratch storage | Byte pointer | Passed to dependencies and used for a direct write at byte offset zero. |

The supplied representation distinguishes signed 32-bit integers, unsigned 32-bit integers, and 8-bit bytes. Physical ABI details, endianness, alignment rules, and address-space constraints are not independently established.

### Descriptor fields

Offsets are byte offsets from the descriptor address.

| Offset | Width and interpretation | Use |
|---|---|---|
| `0x28` | Signed 32-bit | Captured mode and arithmetic operand. |
| `0x80` | Unsigned 32-bit | Captured position. |
| `0x84` | Signed 32-bit | Extent contribution. |
| `0xd4` | Signed 32-bit | Extent contribution. |
| `0x124` | Signed 32-bit | Extent contribution. |
| `0x174` | Signed 32-bit | Extent contribution. |
| `0x1c4` | Signed 32-bit | Extent contribution. |

### Context fields

| Offset | Width and interpretation | Use |
|---|---|---|
| `0x13c` | Signed 32-bit | Zero disables adjustment; every nonzero value enables it. |
| `0x140` | 32-bit quantity | Read only on the enabled adjustment path and passed to the arithmetic dependency. |

### Preconditions and outputs

There is no visible pointer, alignment, capacity, or arithmetic-range validation. Defined normal execution requires valid accesses to every field reached, valid scratch storage for every direct or dependency-mediated access, and arguments accepted by each dependency.

The function does not allocate scratch storage. Its minimum required capacity cannot be derived without dependency contracts. A zero or negative length-like argument does not prevent the function’s direct marker write when the corresponding branch reaches that write.

There is no returned status. Observable outputs are dependency effects and scratch-memory changes. The meaning of a nonzero initial-submission result is limited to its control-flow effect: immediate return.

## Behavior

### Contract notation

The following symbols describe captured values and results:

- **M**: descriptor mode.
- **B**: signed first word captured from State.
- **P**: unsigned descriptor position.
- **E₀**: signed sum of the five extent contributions.
- **E**: extent after the optional adjustment.
- **K**: signed primary result of F-d7be9cfb4000 called with **M**, **B**.
- **S**: signed primary result of F-d7be9cfb4000 called with **P >> 3**, **B**.
- **T**: signed primary result of F-d7be9cfb4000 called with the shifted endpoint, **B**.
- **L**: signed final length-like quantity, **B × ((T − S) + 1)**.
- **U(x)**: conversion to an unsigned 32-bit value.

These symbols do not imply that F-d7be9cfb4000 performs division, is pure, or returns the same result for repeated operands. Every stated invocation is a separate call.

### Initial capture and extent construction

The function first reads **M**, then **B**, then calls F-d7be9cfb4000 with those captured operands to obtain **K**.

Only after this call does it read **P** and the five extent contributions. Effects of that dependency can therefore influence these later reads. The mode and state word are not reloaded.

The extent expression groups signed additions in this order:

1. Combine contributions at `0x1c4` and `0x174`.
2. Add the contribution at `0x124`.
3. Add the contribution at `0xd4`.
4. Add the contribution at `0x84`.

This establishes the arithmetic grouping, not an observable evaluation order among the five field reads. Each signed intermediate must be representable for the expression to have a defined result under the supplied representation.

The context flag is then read:

- If zero, **E = E₀**, and context offset `0x140` is not read.
- If nonzero, the adjustment operand is read from `0x140`. F-d7be9cfb4000 receives that operand first and captured **M** second. If its signed primary result is **J**, the adjusted extent is **(J + E₀) + 1**.

Both additions in the adjustment are signed operations. A flag of −1 enables the same branch as a flag of 1.

### Marker selection

Captured **M** determines one marker byte:

| Mode | Marker |
|---|---|
| `0x200` | `0xe0` |
| `0x400` | `0xc0` |
| Every other value | `0x80` |

No mode is rejected. Selection alone does not write scratch memory.

### Boundary calculations

The function next calls F-d7be9cfb4000 twice, in order:

1. First operand **P >> 3**, second operand **B**; capture **S**.
2. First operand **((P + U(E)) modulo 2³²) >> 3**, second operand **B**; capture **T**.

Both shifts are logical unsigned shifts. The second shift applies to the entire wrapped position-plus-extent sum. A negative **E** is converted to unsigned before this addition; the endpoint calculation therefore has defined modulo-2³² behavior once **E** itself has been validly computed.

Both boundary calls occur before deciding whether to make an initial submission. No check requires **B** to be positive or nonzero, **S** or **T** to be nonnegative, or **T ≥ S**.

### Conditional initial submission

When **S ≠ 0**, the function performs these observable operations in order:

1. Calls F-000000000001 with Scratch and **B**.
2. Writes the selected marker to Scratch byte zero.
3. Calls F-3d3ba69a50c8 with six arguments:

| Position | Argument |
|---:|---|
| 1 | Original State pointer |
| 2 | Unresolved 32-bit auxiliary value |
| 3 | **(U(K) × P) modulo 2³²** |
| 4 | Zero |
| 5 | Captured **B** |
| 6 | Original Scratch pointer |

The producer and meaning of the second argument are not established.

Because **P** is unsigned, the third argument is unsigned 32-bit multiplication. Negative **K** is converted to unsigned, and overflow wraps modulo 2³². This expression is not an unresolved signed-overflow case.

If this submission returns a nonzero primary result, the function immediately returns. It performs no visible rollback, restoration, or release. The final length calculation and all final-stage calls are skipped.

If the submission returns zero, execution continues. When **S = 0**, the entire initial-submission sequence is skipped.

### Final preparation and submission

Every execution that continues beyond the preceding branch performs the following sequence:

1. Computes signed **T − S**.
2. Adds signed one.
3. Multiplies the result by signed **B**, obtaining **L**.
4. Calls F-000000000001 with Scratch and **L**.
5. If **S = 0**, writes the marker to Scratch byte zero.
6. Calls F-6a9ae7071ae3 with a residual position, **E**, and Scratch.
7. Calls F-3d3ba69a50c8 with the final six arguments.
8. Returns without inspecting the final submission result.

The residual position requires particular arithmetic treatment. The expression first multiplies signed **B × S**, then multiplies that signed result by **−8**. Only afterward is the result added to unsigned **P**.

When both signed multiplication intermediates are representable, the residual is:

**(P − 8 × B × S) modulo 2³²**

If either signed multiplication overflows, the supplied representation does not establish the residual result. The final unsigned addition does not make an earlier signed overflow defined.

The final submission arguments are:

| Position | Argument |
|---:|---|
| 1 | Original State pointer |
| 2 | Unresolved 32-bit auxiliary value |
| 3 | **(U(K) × P + U(S)) modulo 2³²** |
| 4 | Zero |
| 5 | **L** |
| 6 | Original Scratch pointer |

Both multiplication and addition in the third argument use unsigned 32-bit arithmetic. Neither requires signed representability of the corresponding mathematical result.

The final auxiliary argument’s producer is unresolved. In particular, it is not established as an output of F-000000000001. F-6a9ae7071ae3 runs between that dependency and the final submission, and its register-preservation behavior is unknown.

### Arithmetic boundaries

The following operations have established unsigned 32-bit behavior:

- Position-plus-extent endpoint addition.
- Both position shifts.
- Initial submission’s **K × P** expression.
- Final submission’s **K × P + S** expression.
- Addition of the validly computed signed residual term to **P**.

The following require representable signed intermediates:

- Each grouped addition forming **E₀**.
- Adjustment-result plus **E₀**, then plus one.
- **T − S**, then plus one.
- Multiplication producing **L**.
- **B × S**, then multiplication by **−8** for the residual.

No widening or overflow checks are visible. Overflow cases for these signed operations are `documented_only`; this contract does not replace them with assumed wraparound.

## Side effects

The function directly reads the listed fields and writes only Scratch byte zero. It contains no direct write to Context, Descriptor, or the State word.

For paths whose dependencies return normally and whose arithmetic is defined, exactly one direct marker write occurs:

- When **S ≠ 0**, it occurs before the initial submission.
- When **S = 0**, it occurs during final preparation.

If **S ≠ 0** and the initial submission returns zero, the second F-000000000001 call is not followed by another direct marker write. The function therefore does not guarantee marker survival across that call. F-6a9ae7071ae3 also runs after any final-stage marker write and may alter that byte under its unknown contract.

Dependencies may affect input objects through aliases, globals, retained references, or other mechanisms. Their preservation of memory or machine state is not established. Captured scalar values remain the values used by this function, but submissions receive the original State pointer and may observe changed pointed-to contents.

No ownership transfer, allocation, deallocation, copying guarantee, or pointer-retention rule is established. Aliasing between Scratch and other inputs is not explicitly checked or prohibited; safe overlapping use remains `documented_only`.

## Dependencies

| Opaque ID | Established interaction | Unknown contract |
|---|---|---|
| F-d7be9cfb4000 | Two operands; signed primary result used by later arithmetic. Three calls without adjustment, four with adjustment, if all calls return normally. | Mathematical operation, operand interpretation at the call boundary, exceptional inputs, side effects, determinism, preservation. |
| F-000000000001 | Scratch pointer and a signed 32-bit length-like argument; called once or twice. | Whether it initializes, fills, validates, or otherwise processes storage; capacity requirements; errors; ownership. |
| F-6a9ae7071ae3 | Residual position, signed extent, Scratch; called once on the final path. | Layout produced, write range, errors, register and memory preservation. |
| F-3d3ba69a50c8 | Six arguments; one or two submissions. Only the initial result controls continuation. | Submission semantics, auxiliary values, memory effects, ownership, and status meanings beyond the observed branch. |

F-000000000001 is a specification-local opaque ID for the unresolved external dependency. Its identifier asserts no implementation identity.

## Acceptance examples

These are conditional static examples, not executed tests. Unless an example specifically addresses a boundary, all signed intermediates are assumed representable and dependencies return normally.

### Zero starting boundary

Given **M = 0x200**, **B = 16**, **S = 0**, and **T = 2**:

- No initial submission occurs.
- **L = 48**.
- F-000000000001 receives Scratch and 48.
- Scratch byte zero is then written as `0xe0`.
- F-6a9ae7071ae3 receives residual **P**, extent **E**, and Scratch.
- The final submission receives **(U(K) × P) modulo 2³²** as its third argument and 48 as its fifth.

Final scratch contents remain dependent on F-6a9ae7071ae3 and the submission dependency.

### Nonzero boundary with early return

Given **S = 3**, **M = 0x400**, and a nonzero initial-submission result:

- F-000000000001 receives **B**.
- Scratch byte zero is written as `0xc0`.
- The initial submission receives the unsigned wrapped product **U(K) × P**.
- No final length calculation, second buffer call, residual preparation, or final submission follows.

### Nonzero boundary with continuation

Given **S = 2**, **T = 4**, **B = 8**, and a zero initial-submission result:

- Both submission stages occur.
- **L = 24**.
- Signed residual intermediates are 16 and −128.
- The residual passed to F-6a9ae7071ae3 is **(P − 128) modulo 2³²**.
- The final submission’s third argument is **(U(K) × P + 2) modulo 2³²**.
- No direct marker rewrite follows the second F-000000000001 call.

### Optional adjustment and unsigned endpoint wrap

If the context flag is nonzero and the adjustment dependency returns 6, **E = (6 + E₀) + 1**, provided both signed additions are representable.

Separately, if **P = 0xfffffff8** and **E = 16**, the endpoint wraps to 8, so the second boundary call receives 1 as its first operand.

### Unsigned submission arithmetic

If **K = −1**, **P = 2**, and **S = 3**:

- The initial submission’s third argument, when that submission occurs, is `0xfffffffe`.
- The final submission’s third argument is 1.

These results follow from unsigned conversion and modulo-2³² arithmetic, not assumed signed wraparound.

### Residual signed-overflow boundary

If **B = 268435456** and **S = 1**, their first product is representable, and multiplication by −8 yields the representable signed minimum, −2147483648. The modular residual formula applies.

If instead **B = 268435457** and **S = 1**, multiplication by −8 is outside the signed 32-bit range. If execution reaches that expression, its result is `documented_only`; no wrapped residual is promised.

### Equal boundaries and ignored final result

If **S = T**, the length expression produces **B**, not zero.

If the final submission returns a nonzero result normally, the function still returns without testing it. This does not establish that the submission succeeded or that its effects were rolled back.

## Uncertainties

The following remain `documented_only`:

- The operation performed by F-d7be9cfb4000.
- Both auxiliary submission arguments, including their producers.
- Scratch layout, required capacity, and final contents.
- Signed-overflow outcomes.
- Full ABI, alignment, address arithmetic, and dependency-boundary interpretation.
- Dependency errors, traps, exceptions, nonlocal exits, and asynchronous effects.
- Global-state preservation, reentrancy, concurrency, alias safety, and ownership.
- Any externally stored representation or higher-level meaning of a submission.

Inferred intent: the shifted positions, extent aggregation, mode-dependent markers, and residual suggest preparation of a segmented position/extent representation. That interpretation is not required for conformance and does not supply missing dependency semantics.

## Evidence

Established claims derive from the supplied static representation: input types, field offsets, expression grouping, signed and unsigned operand combinations, dependency argument order, branch conditions, direct byte writes, and return paths.

Arithmetic grouping is distinguished from unspecified operand-read order. Unresolved auxiliary values are not assigned unsupported producers. Dependency names and apparent arithmetic patterns are not treated as semantic evidence.

No dependency bodies, ABI documentation, runtime observations, or independent layout definitions were supplied. No runtime tests have been performed.
