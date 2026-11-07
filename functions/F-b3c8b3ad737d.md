<!-- Trusted provenance: F-b3c8b3ad737d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b3c8b3ad737d

## Purpose

The supplied static representation describes a low-level routine that repeatedly writes through an output pointer and through addresses derived from hidden entry state. A condition on that hidden state determines whether execution remains in the write loop or proceeds to additional memory writes and two indirect calls. The represented continuation ends at an invalid-instruction boundary rather than a normal return.

No reliable application-level purpose is established. In particular, the evidence does not justify describing the routine as initialization, reset handling, allocation, or cleanup. Symbol names resembling those concepts are not behavioral proof.

The representation carries explicit warnings about invalid instruction data and overlapping global symbols. This contract therefore distinguishes:

- **Established static evidence:** operations and ordering represented in the supplied material.
- **Conditional interpretation:** consequences under a 32-bit machine-word interpretation.
- **documented_only unknowns:** facts that require trustworthy instruction decoding, declarations, execution context, or dependency contracts.

No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface has three arguments:

| Input | Represented form | Observable use |
|---|---|---|
| First argument | Opaque 32-bit value | No direct use is shown. |
| Second argument | Pointer-like value | Becomes a value written to the output on later loop iterations. It is not directly dereferenced. |
| Third argument | Pointer to 32-bit storage | Receives two consecutive 32-bit writes at offsets 0 and 4 on every loop iteration. Its value also participates in a later address calculation. |

The routine has no represented return value. No normal return is shown.

The unused first argument is not established to be irrelevant to the actual machine routine: the unreliable decoding limits that conclusion to the supplied representation.

### Hidden entry inputs

The represented behavior also depends on state outside the explicit interface:

- A 32-bit **condition word** controlling loop termination and contributing to an address.
- A 32-bit **address-base word** contributing to two computed addresses.
- A **saved data word** written repeatedly to the first output word.
- A **saved pointer-like word** initially written to the second output word.
- A **link-derived address value** used to select the first indirect call target.
- A four-byte **frame-resident pointer value**, represented at stack offset `0x140`.
- A shared **source object** read twice after loop termination.
- A memory-associated **call-argument source** identified with address `0x2FE75000`.

The representation does not explain how these inputs are initialized. The three explicit arguments alone are insufficient to determine behavior.

### Width and representation constraints

The explicit scalar argument and the output elements are represented as 32-bit quantities. Pointer arithmetic, pointer-to-integer conversions, and address masks are also expressed using a 32-bit model.

The arithmetic consequences below use that model conditionally. The actual architecture, instruction semantics, endianness, address-space rules, and calling convention are not independently established.

Stores through explicitly 16-bit pointers are two bytes wide; the store through a character pointer is one byte wide. Several direct assignments to global symbols lack declarations. For those assignments, the value being assigned may be known while the destination’s actual storage width remains **documented_only**.

### Preconditions for an uninterrupted represented path

Execution requires more than a writable eight-byte output region:

- Every computed destination reached on the selected path must permit the indicated access.
- The shared source and call-argument source must be readable.
- The indirect call targets must be executable under the actual calling convention.
- The first dependency must return an address that permits a two-byte zero write.
- The final frame-derived pointer must permit a four-byte zero write.

These are execution prerequisites, not checked conditions. No validation or recovery path is shown.

## Behavior

### 1. Repeated write phase

Each iteration begins with these operations in the represented order:

1. Write the saved data word to the first 32-bit output element.
2. Write the current saved pointer-like word to the second 32-bit output element.
3. Compute an address from the address-base word multiplied by `0x100`. Store the low 16 bits of that computed address through a 16-bit pointer at that address.
4. Test whether shifting the condition word left by four produces zero.

Under a wrapping 32-bit interpretation, the third operation uses:

- Address: the low 32 bits of the address-base word multiplied by `0x100`.
- Stored payload: the low 16 bits of that address.

The halfword write occurs before the termination test, including on an iteration that exits immediately.

If the test succeeds, execution leaves the loop. If it fails, the iteration additionally:

- Stores one byte at the address formed from the condition word plus the address-base word.
- Uses the low byte of the condition word shifted left by four as the byte payload.
- Replaces the current second-argument value with the fixed address `0x0897F360`.
- Makes the second-argument value captured at the start of this iteration the saved pointer-like word for the next iteration.

Assuming distinct, ordinary writable memory and no interruption, successive second-output-word writes are:

| Iteration | Value written |
|---|---|
| First | Hidden saved pointer-like word supplied at entry |
| Second | Original explicit second argument |
| Third and later | `0x0897F360` |

The first output word receives the same saved data word on every iteration.

Neither the condition word nor the address-base word is explicitly updated in this phase. Consequently, under the represented state model:

- If the condition succeeds initially, exactly one iteration executes.
- If it fails initially, there is no represented internal transition that later makes it succeed.

The latter path remains in the loop unless execution is interrupted, faults, or behaves outside this representation.

For a 32-bit logical interpretation of the shift, termination requires the low 28 bits of the condition word to be zero. The full condition word need not be zero.

### 2. Post-loop assignments and derived write

After loop termination, the representation shows:

1. Assign the condition word narrowed to 16 bits to storage associated with address `0x0897F362`.
2. Read the shared source object and replace the low 16 bits of the frame-resident pointer value with the source’s low 16 bits, preserving the pointer value’s upper 16 bits.
3. Read the shared source object again and write its low 16 bits through a 16-bit pointer at the address derived by shifting the output pointer’s 32-bit representation left by 20.
4. Assign the address-base word, interpreted unsigned and shifted right by 20, to storage associated with address `0x000897F5`.
5. Assign `0x28` to storage associated with address `0x00000028`.
6. Assign zero to a further shared control destination whose address and width are not provided.

The two source reads are distinct in the representation. They must not be assumed to produce the same value. The first determines the substituted pointer bits; the second determines the halfword payload.

Under the conditional 32-bit model:

- A condition word satisfying the loop-exit test has zero low 16 bits. Its narrowed value in the first assignment is therefore zero.
- The shifted output-pointer address depends only on the original pointer’s low 12 bits and has its low 20 bits cleared.
- The unsigned right shift of the address-base word produces a value between `0` and `0xFFF`.

The output-pointer-derived write is not a write at a conventional offset into the output buffer. It targets a separately computed address.

### 3. Dependency calls

The first indirect call, designated **F-000000000001**, targets an address computed from the hidden link-derived value plus `0xA84`.

It receives:

- An integer conversion of the value associated with the source at `0x2FE75000`.
- A second argument equal to zero.

The source declaration is unavailable. Its read width and any sign extension before conversion are **documented_only**.

The returned value is treated as a pointer to 16-bit storage. The routine writes a two-byte zero at that address without a represented null check.

The second indirect call, designated **F-000000000002**, receives that returned pointer. Its target is computed by clearing the low two bits of `0x0897F3D5` and adding `0x604`. Under ordinary 32-bit arithmetic, the resulting address is `0x0897F9D8`.

The routine writes through the returned pointer before passing it to the second dependency. The second dependency’s return value is not used in the shown continuation.

Neither call is established to preserve global state or to return normally.

### 4. Final represented writes and terminal boundary

If both calls return and preceding accesses complete, the representation continues with:

1. Assign `0xF484` to storage associated with address `0x0000897F`.
2. Assign zero to storage associated with address `0x44BF9D40`.
3. Assign zero to another shared control destination whose address and width are unspecified.
4. Write a four-byte zero through the frame-resident pointer whose low half was replaced earlier.
5. Assign zero to storage associated with address `0x00000034`.
6. Reach the reported invalid-instruction boundary.

The frame-resident pointer modification precedes both dependency calls. The evidence does not establish that the calls leave its backing storage unchanged.

The terminal boundary is not evidence of a deliberate application-level halt, a particular exception, or a normal return. Actual behavior beyond that point is **documented_only**.

## Side effects

### Memory visibility and aliasing

The represented writes are ordered as described above. That order matters when destinations overlap.

For example, the loop’s computed halfword write follows both output-word writes. If it overlaps the output region, it can alter bytes just written there. On a nonterminating iteration, the computed byte write occurs later still.

The fixed address used as the later second-output value, `0x0897F360`, is two bytes below the post-loop destination `0x0897F362`. No dereference of that fixed pointer value is shown.

No non-aliasing guarantee is supplied among:

- The output region.
- Computed byte and halfword destinations.
- The frame-resident pointer storage.
- Shared sources and control destinations.
- Memory accessed by either dependency.

Partial writes therefore cannot be treated as isolated updates to independent objects. Endianness-dependent byte effects require additional evidence.

### Ownership and lifetime

No allocation, ownership transfer, or release is established.

The first dependency’s result might have several storage origins; none is proven. Passing it to the second dependency does not establish whether that dependency retains, consumes, modifies, or releases it.

The explicit second argument is copied as a value, not directly dereferenced. The third argument supplies writable storage, but the routine is not shown to acquire ownership of it.

### Errors and partial progress

No error return, explicit status value, rollback, bounds check, or null check is represented.

An access failure or dependency that does not return may prevent all later operations. Earlier writes are not shown to be undone. In particular, an invalid first dependency result is used for a halfword store before the second dependency is invoked.

## Dependencies

The following opaque IDs are contract-local labels for unresolved call sites, not recovered identities.

| ID | Established interaction | Unknown contract |
|---|---|---|
| **F-000000000001** | Indirect target derived from hidden link state; two arguments; result used as a 16-bit-storage pointer | Argument interpretation, target validity, return validity, memory effects, ownership, failure behavior, register and global preservation |
| **F-000000000002** | Computed target `0x0897F9D8` under the stated model; receives the first result after a zero halfword write | Pointer lifetime effects, ownership, additional writes, failure behavior, state preservation |

The invalid-instruction termination notation is not treated as a separately established callable dependency.

## Acceptance examples

These are conditional acceptance cases for a future implementation or validation effort, not executed tests.

### Immediate exit

Given a zero condition word, valid hidden state, and valid non-overlapping destinations:

- Exactly one loop iteration occurs.
- Both output-word writes and the computed halfword write precede loop exit.
- No loop-body byte write occurs.
- Post-loop operations are reached, subject to accesses completing.

### Nonzero condition that still exits

Given condition word `0x10000000` and 32-bit shift semantics:

- Shifting left by four produces zero.
- The loop exits after its first iteration.
- The condition-derived 16-bit assignment carries zero.

An implementation that tests only whether the unshifted condition word is zero would not match this behavior.

### Persistent loop

Given condition word `1`, with accesses completing:

- The exit test remains false under the represented state model.
- The computed byte payload is `0x10`.
- The second output word receives the entry saved pointer-like value, then the original second argument, then `0x0897F360` repeatedly.
- Neither dependency call is reached.

### Computed halfword payload

Given address-base word `0x12345678` and wrapping 32-bit arithmetic:

- The loop halfword destination is `0x34567800`.
- The stored halfword is `0x7800`.

On a terminating path, the later unsigned-right-shift assignment carries `0x123`.

### Separate source reads

Given frame-resident pointer value `0xABCD1234`, with source reads producing low halves `0x5678` and `0x9ABC` respectively:

- The pointer becomes `0xABCD5678`.
- The output-pointer-derived halfword write carries `0x9ABC`.

Using the first read for both effects would not match the represented sequence.

### Invalid dependency result

If F-000000000001 returns zero, the next represented operation attempts a two-byte zero write at address zero. No graceful failure or invocation of F-000000000002 is guaranteed.

## Uncertainties

The following remain **documented_only**:

- Whether the decoded control flow corresponds to executable instructions.
- Actual architecture, endianness, ABI, and overflow or shift semantics.
- Initialization and provenance of hidden entry state.
- Exact widths and volatility of incompletely declared global objects.
- Whether numeric addresses designate ordinary memory, device registers, executable storage, or unmapped space.
- Whether overlapping symbols represent intended shared storage or decoding artifacts.
- Source-object read widths and sign-extension rules.
- Dependency ownership, preservation, failure, and return contracts.
- Effects of concurrent or device-driven changes.
- Actual behavior at the invalid-instruction boundary.

Intentional infinite looping, reset behavior, and memory-management intent are not established.

## Evidence

This contract is based solely on the supplied static representation. It exposes a three-argument interface, hidden entry-state dependencies, repeated memory writes, a shift-based loop condition, post-loop assignments, two indirect calls, and a terminal invalid-instruction warning.

The supplied material also explicitly warns that global symbols overlap. Those warnings constrain confidence in the representation and in inferred storage layouts.

Arithmetic examples and address simplifications are conditional deductions under the stated 32-bit interpretation. They are not runtime observations. No implementation access, dependency inspection, or runtime testing was used.
