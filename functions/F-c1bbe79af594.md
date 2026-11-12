<!-- Trusted provenance: F-c1bbe79af594; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c1bbe79af594

## Purpose

F-c1bbe79af594 selects one of two construction paths, obtains and prepares storage through dependencies, initializes path-specific fields and four-element collections, and returns a 32-bit address-like result.

The branch is determined exclusively by whether F-30953398b064 returns zero. The first input is unused in the supplied representation. The second input is stored only on the nonzero-result path. The third input is stored and passed to initialization dependencies on both paths.

**Established static evidence:** the branch condition, dependency calls, explicit reads and writes, constants, iteration counts, and return-address calculations described below.

**Inferred intent:** the function resembles a constructor for two related object layouts. Values of all-one bits resemble unset sentinels, and several byte fields resemble flags. These interpretations do not establish field types, lifecycle rules, or domain semantics.

## Interface

### Inputs

The function accepts three 32-bit values:

| Input | Established use |
|---|---|
| First input | No observable use in the supplied function body. |
| Second input | Stored as a 32-bit value on the nonzero-result path; unused on the zero-result path. |
| Third input | Stored as a 32-bit value and supplied to initialization dependencies on both paths. |

No pointer, ownership, enum, or signedness contract for these inputs is established. In particular, the third input must not be described as a valid pointer or resource handle solely from its uses here.

### Result

The function returns a 32-bit address-like value calculated from the result of F-469501ccc769:

- Nonzero-result path: that dependency’s result minus `0x13C6`.
- Zero-result path: that dependency’s result minus `0x61A`.

There is no explicit error-result branch, null check, exception handler, or cleanup path in the supplied body.

### Layout notation

The following notation describes externally observable accesses without assigning undocumented field meanings:

- **R**: the address returned by this function.
- **A**: the result of F-469501ccc769.
- **H**: the result of F-bedebbf8780c.
- **P**: the result of F-8157a180d901.

On the nonzero-result path, `A = R + 0x13C6`. On the zero-result path, `A = R + 0x61A`.

Offsets are byte offsets. A **byte write** writes 8 bits. A **word write** writes 32 bits. `0xFFFFFFFF` denotes the exact 32-bit all-one pattern; its semantic interpretation as signed −1 is not established.

### Preconditions

For normal completion:

- Each dependency must return normally.
- Every explicitly accessed address must be accessible for the stated width.
- Storage returned or derived through dependencies must support all listed accesses.
- The execution environment must support the observed address arithmetic and potentially unaligned 32-bit accesses.

The supplied body does not validate these conditions. Allocation sizes, alignment guarantees, overflow behavior, and dependency failure conventions are documented_only unknowns.

## Behavior

### 1. Branch selection

F-30953398b064 is called once, without explicit arguments.

- A nonzero result selects the larger-layout path.
- A zero result selects the smaller-layout path.

No particular nonzero value is required. The result is not otherwise used.

### 2. Common preparation sequence

Each path uses distinct global values for its allocation argument and two descriptor-like values. Their actual contents and meanings are unavailable.

For the selected path, preparation occurs in this order:

1. Read the path-specific allocation argument and call F-8157a180d901 with it, obtaining P.
2. Read the path-specific first descriptor value and write it as a word at `P + 0`.
3. Write zero words at `P + 8`, then `P + 12`.
4. Call F-bedebbf8780c with `P + 0x20`, obtaining H.
5. Read the path-specific second descriptor value.
6. Write a zero byte at `H + 0x30`.
7. Write the previously read second descriptor value as a word at `H − 0x20`.
8. Call F-469501ccc769:
   - With `H + 0x13A6` on the larger-layout path.
   - With `H + 0x5FA` on the smaller-layout path.
9. Use its result as A and derive R using the path-specific subtraction above.

There is no explicit write to `P + 4`. No equality or fixed-offset relationship between P, H, and A is established beyond the arguments and arithmetic shown.

The second descriptor is read after F-bedebbf8780c returns. Its value must not be modeled as a snapshot taken before that call.

### 3. Larger-layout path

#### Initial fields and dependency calls

The following occur in order:

| Operation | Address or arguments |
|---|---|
| Store second input as a word | `A − 0x366` |
| Store zero byte | `A − 0x342` |
| Store third input as a word | `A − 0x46` |
| Call F-6c42998e42ae | Third input, `A − 0x2A` |
| Call F-46c92be38753 | Word freshly read from `A − 0x46`, `A − 0x26` |
| Call F-46c92be38753 again | Word freshly read from `A − 0x46`, `A − 0x1E` |
| Store zero bytes, in order | `A − 0x35`, `A − 0x42` |
| Store word value 8000 | `A − 0x3E` |

The reads from `A − 0x46` occur separately after intervening calls. They are not guaranteed to reproduce the original third input if a dependency has modified that storage.

#### Four larger records

Exactly four iterations occur, in ascending index order from 0 through 3.

For index \(i\), the record begins at `R + 0x1240 + i × 0x50`. The following operations occur in the listed order, with offsets relative to that record:

| Order | Operation |
|---|---|
| 1 | Write all-one words at `+0x00`, `+0x04`, `+0x08`. |
| 2 | Write zero word at `+0x10`. |
| 3 | Write all-one word at `+0x0C`. |
| 4 | Write zero words at `+0x14`, `+0x18`. |
| 5 | Write all-one words at `+0x1C`, `+0x24`, `+0x40`. |
| 6 | Read the word at `A − 0x46`; call F-46c92be38753 with that value and the record address `+0x30`. |
| 7 | Write zero words at `+0x28`, `+0x2C`. |
| 8 | Write zero bytes at `+0x44`, `+0x45`, `+0x46`. |
| 9 | Write all-one words at `+0x20`, `+0x48`. |
| 10 | Write zero byte at `+0x4C`. |
| 11 | Write zero word at `R + 0x10C4 + i × 0x60`. |

Each iteration finishes before the next begins. In particular, the dependency call precedes the later writes in its own record and the separate `0x60`-stride collection write.

The function does not directly initialize every byte of each `0x50`-byte record. Effects of F-46c92be38753 on the destination and other storage remain unknown.

#### Four additional slots

After all four records, write zero words at these addresses, in order:

- `R + 0x1210`
- `R + 0x1218`
- `R + 0x1220`
- `R + 0x1228`

These are four 32-bit writes at an 8-byte stride, not four 8-byte clears.

#### Final larger-layout writes

The following writes occur in order, all relative to A:

| Offset | Width | Value |
|---|---:|---:|
| `−0x359` | 8 | 2 |
| `−0x41` | 8 | 0 |
| `−0x33` | 8 | 0 |
| `−0x189` | 8 | 0 |
| `−0x18A` | 8 | 1 |
| `−0x34` | 8 | 0 |
| `−0x3A` | 32 | 0 |
| `−0x32` | 32 | `0xFFFFFFFF` |
| `−0x346` | 32 | 0 |
| `−0x362` | 32 | `0xFFFFFFFF` |
| `−0x35E` | 32 | `0xFFFFFFFF` |
| `−0x36` | 8 | 0 |
| `−0x16` | 8 | 0 |
| `−0x15` | 8 | 0 |
| `−0x0E` | 8 | 0 |
| `−0x2E` | 8 | 0 |
| `−0x2D` | 8 | 0 |
| `−0x0A` | 32 | 0 |
| `−0x06` | 32 | 0 |
| `−0x02` | 8 | 0 |
| `−0x34A` | 32 | 0 |
| `−0x13B6` | 8 | 0 |
| `−0x13AA` | 32 | 0 |
| `−0x13B2` | 32 | 0 |
| `−0x13AE` | 32 | 0 |
| `+0x01` | 8 | 1 |
| `−0x01` | 8 | 0 |
| `−0x1372` | 32 | 0 |

The function then returns R.

### 4. Smaller-layout path

#### Initial fields

The following occur in order:

1. Store the third input as a word at `A + 0x0A`.
2. Call F-46c92be38753 with the third input and `A + 0x92`.
3. Store byte value 1 at `A + 0x0E`.
4. Store zero byte at `A + 0x2E`.

Unlike the larger-layout path’s subsequent calls, this first dependency call receives the original third input directly, not a reread of its stored copy.

#### Four smaller records

Exactly four iterations occur, in ascending index order from 0 through 3.

For index \(i\), the record begins at `R + 0x62C + i × 0x20`. Operations occur in this order:

| Order | Operation |
|---|---|
| 1 | Read the word at `A + 0x0A`; call F-46c92be38753 with that value and the record start. |
| 2 | Write zero word at record `+0x08`. |
| 3 | Write all-one word at record `+0x0C`. |
| 4 | Write zero word at record `+0x14`. |
| 5 | Write zero word at record `+0x10`. |
| 6 | Write zero byte at record `+0x18`. |
| 7 | Write zero word at `R + 0x4A0 + i × 0x60`. |

The stored third-input value is reread for each iteration. Dependencies may affect the values obtained by later reads.

After all four iterations, call F-d3d627eaec9e with `A − 0x1B6`.

#### Additional slots and final fields

After F-d3d627eaec9e returns, write zero words at:

- `R + 0x5EC`
- `R + 0x5F4`
- `R + 0x5FC`
- `R + 0x604`

Then perform these writes in order, relative to A:

| Offset | Width | Value |
|---|---:|---:|
| `+0x8E` | 8 | 0 |
| `+0x8F` | 8 | 0 |
| `−0x1BA` | 8 | 0 |
| `+0x02` | 32 | 0 |
| `+0x06` | 8 | 0 |
| `−0x60A` | 8 | 0 |
| `−0x5FE` | 32 | 0 |
| `−0x606` | 32 | 0 |
| `−0x602` | 32 | 0 |
| `+0x07` | 8 | 0 |
| `−0x01` | 8 | 0 |
| `−0x02` | 8 | 1 |
| `−0x5C6` | 32 | 0 |

The function then returns R.

## Side effects

The direct side effects are the writes and dependency calls listed above. They include header-like preparation, scalar initialization, four-record initialization, and separate strided word clears.

No explicit release, deallocation, reference decrement, logging, or global assignment appears in the supplied body. This does not constrain indirect effects of dependencies.

Ownership is **documented_only**:

- The allocation-like role of F-8157a180d901 does not establish who owns its result.
- The returned R is not proven to be the allocation base.
- Passing the third input to dependencies does not establish copying, borrowing, retention, or ownership transfer.
- Neither initialization dependency is known to preserve globals or unrelated memory.
- There is no visible recovery if a dependency fails after earlier writes have occurred.

Final-memory assertions require care: a write made before a dependency call is not necessarily still present afterward. The contract establishes the explicit access sequence, not an undocumented preservation guarantee.

## Dependencies

| Opaque ID | Explicit use |
|---|---|
| F-30953398b064 | Called once without arguments; zero versus nonzero selects the layout. |
| F-8157a180d901 | Called once with a path-specific global value; returns P used for direct writes. |
| F-bedebbf8780c | Called once with `P + 0x20`; returns H used for direct writes and subsequent argument formation. |
| F-469501ccc769 | Called once with a path-specific displacement from H; returns A, from which R is derived. |
| F-6c42998e42ae | Larger path only; one call with the third input and a destination at `A − 0x2A`. |
| F-46c92be38753 | Larger path: six calls. Smaller path: five calls. Argument order and rereads are specified above. |
| F-d3d627eaec9e | Smaller path only; one call after the four records, with `A − 0x1B6`. |

Return values from the last three initialization dependencies are not used explicitly. Their memory effects, failure behavior, and ownership behavior are unknown.

## Acceptance examples

These are conditional acceptance criteria, not executed tests.

1. **Nonzero selection.** If F-30953398b064 returns any nonzero value and all dependencies complete normally, the larger path is selected. If F-469501ccc769 returns address Q, the function returns `Q − 0x13C6`.

2. **Zero selection.** If the selector returns zero, the second input is not explicitly stored or forwarded, F-6c42998e42ae is not called, and F-d3d627eaec9e is called once.

3. **First-input independence.** Changing only the first input does not change any explicit branch, argument, or stored value in this body. This is not a guarantee about dependencies that could independently inspect calling context.

4. **Reread sensitivity.** On the larger path, if F-6c42998e42ae changes the word at `A − 0x46`, the next F-46c92be38753 call receives the changed value. If later calls modify it again, later rereads observe the then-current value.

5. **Exact-width sentinel.** Each listed all-one word write affects four bytes. It must not be widened into a pointer-sized store on a wider host.

6. **Sparse clearing.** With suitably accessible storage initially containing nonzero bytes and dependencies that leave the relevant locations untouched, the additional-slot loops clear only four bytes per 8-byte slot. The other four bytes in each slot are not cleared by those loops.

7. **Call-before-overwrite ordering.** On the smaller path, each record dependency call occurs before the record’s explicit zero and sentinel writes. Any dependency-written bytes overlapping those later writes are replaced by the listed values.

8. **No invented failure result.** If F-8157a180d901 produces an unusable address, the body does not explicitly convert that condition into a null return or error code; its next operation attempts the specified storage access.

## Uncertainties

The following remain **documented_only**, not implementation requirements inferred from naming or familiar patterns:

- Actual allocation sizes and global constant values.
- Object types, field names, record semantics, and meanings of byte values 1 and 2.
- Whether descriptor-like values identify types, dispatch tables, or something else.
- Dependency preservation guarantees and indirect side effects.
- Dependency failure signaling, nonlocal exits, and partial-construction cleanup.
- Ownership and eventual destruction requirements.
- Byte order, alignment rules, and exact address-overflow semantics.
- Whether unwritten bytes are initialized elsewhere.
- Whether any dependency aliases the regions involved in later accesses.
- Behavior omitted or distorted by the supplied source representation.

The displayed storage and address quantities are 32-bit. The small loop indices range only from 0 through 3, so their displayed stride products do not approach 32-bit overflow. Address additions and subtractions still require valid address relationships; this contract does not invent portable wraparound or fault semantics.

## Evidence

This contract is based solely on static inspection of the supplied function representation.

The representation establishes two branch bodies, explicit 8-bit and 32-bit stores, fixed four-iteration loops, ordered dependency calls, fresh reads of stored input values, and path-specific return-address subtraction.

The supplied material also reports unreachable-block removal and delayed elimination of dead material. Those notices limit confidence about omitted behavior; they do not establish an additional executable path.

No dependency bodies, allocation metadata, type definitions, caller contracts, or runtime observations were supplied. No runtime tests have been performed.
