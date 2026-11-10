<!-- Trusted provenance: F-c828afef6d06; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c828afef6d06

## Purpose

F-c828afef6d06 constructs a caller-supplied record from data obtained through a temporary resource. It performs an initial acquisition operation, transfers selected fields into the destination, and conditionally performs a second acquisition followed by a bounded marker search.

The function returns a 32-bit status: `1` on its explicit success paths and `0` on its explicit failure paths. A successful return does not require finding the marker. A failed return does not imply that the destination is unchanged.

The record’s application-level meaning, the acquisition mechanism, and the marker value are not established. Descriptions such as “initialization,” “transfer,” and “release” below identify apparent roles; they do not establish the internal behavior of unknown dependencies.

## Interface

The function accepts four arguments:

| Argument | Established representation | Use |
|---|---|---|
| A | 32-bit value | Passed unchanged as the first argument of both acquisition calls. Its interpretation is unknown. |
| D | Byte-addressed pointer | Destination record and argument to several dependencies. |
| C | 32-bit value | Not referenced by the visible function body. |
| X | 32-bit value | Passed twice to the initial dependency call. |

The return value is 32 bits and is explicitly either `0` or `1` on every visible normal return.

### Address notation and extent

All offsets in this contract are hexadecimal byte offsets:

- **D + offset** denotes a destination address.
- **R + offset** denotes an address relative to the nonzero resource value returned by F-2f4031632ae3.
- **B** denotes the address **R + 0x18**, used as the acquisition buffer and the start of the marker search.

The destination’s apparent size is `0x5c` bytes, or 92 bytes. The initial dependency receives that size, and the latest direct destination write occupies offsets `0x58` through `0x5b`. This is not evidence that the initial dependency clears all 92 bytes.

Valid writable destination storage and valid readable resource storage are preconditions for the visible accesses. No null check, capacity check, or alignment repair is visible for D. Native 16-bit and 32-bit accesses occur, so the execution environment must support the addresses and alignments used.

No source-level ownership transfer of D is established. Whether dependencies retain pointers, allocate storage, or release storage internally is unknown.

## Behavior

### 1. Initial preparation and resource acquisition

The first observable call is to F-4042d12a1376 with these five arguments, in order:

1. D
2. `0x5c`
3. `0`
4. X
5. X

Its result, if any, is not inspected. Its memory effects are unknown; specifically, this contract does not assume that it zeroes D.

The function next calls F-2f4031632ae3 with no explicit arguments.

- If the returned 32-bit resource value is zero, the function returns `0`.
- Otherwise, it forms B as R + `0x18` and calls F-a63889a8554d with `(A, 0, B, 1, 0)`.

A zero result from this acquisition call causes a call to F-2b1ba6d8de9d with `(R, 1)`, followed by return `0`. A nonzero result begins destination population.

The value of R is retained for subsequent resource-relative accesses and the eventual finalization call. The buffer pointer used for the later search initially remains B.

### 2. Initial destination population

The following actions occur in the listed order. “Direct” means a visible native-width read followed by a destination write. Dependency-mediated operations state the call arguments without assuming the callee’s implementation.

| Order | Destination | Source or action | Width |
|---:|---|---|---:|
| 1 | D + `0x00` | Direct read from R + `0x18` | 1 byte |
| 2 | D + `0x01` | F-2eceb10e9abc with destination, R + `0x1b`, and length `8` | Requested 8 bytes |
| 3 | D + `0x09` | Direct write of zero | 1 byte |
| 4 | D + `0x0c` | Direct read from R + `0x25` | 1 byte |
| 5 | D + `0x10` | Direct read from R + `0x28` | 1 byte |
| 6 | D + `0x16` | Direct read from R + `0x2d` | 1 byte |
| 7 | D + `0x2c` | Direct read from R + `0x3c` | 1 byte |
| 8 | D + `0x2d` | Direct read from R + `0x3d` | 1 byte |
| 9 | D + `0x2e` | Direct read from R + `0x3e` | 1 byte |
| 10 | D + `0x0a` | F-ext-01 with destination, R + `0x23`, and length `2` | Requested 2 bytes |
| 11 | D + `0x0e` | F-ext-01 with destination, R + `0x26`, and length `2` | Requested 2 bytes |
| 12 | D + `0x12` | F-ext-01 with destination, R + `0x29`, and length `2` | Requested 2 bytes |
| 13 | D + `0x14` | F-ext-01 with destination, R + `0x2b`, and length `2` | Requested 2 bytes |
| 14 | D + `0x18` | F-ext-01 with destination, R + `0x2e`, and length `2` | Requested 2 bytes |
| 15 | D + `0x1a` | Direct read from R + `0x30` | 2 bytes |
| 16 | D + `0x1c` | F-ext-01 with destination, R + `0x32`, and length `2` | Requested 2 bytes |

F-ext-01 is a document-local opaque ID for the external dependency not identified in the supplied dependency map.

Direct multibyte operations preserve the native-width value read from the source. The evidence does not establish native byte order. Neither transfer dependency’s name or implementation is available as trustworthy behavioral evidence; byte swapping, transformation, overlap handling, and exact write extent remain unspecified.

### 3. First discriminator check and optional extended fields

The function reads the native 16-bit value at D + `0x12` and compares it with zero. The visible comparison uses a signed 16-bit interpretation, but signedness does not affect a zero-versus-nonzero decision.

If this first check is zero, the following actions occur in order:

| Destination | Source or action | Width |
|---|---|---:|
| D + `0x28` | Direct read from R + `0x3c` | 4 bytes |
| D + `0x48` | Direct read from R + `0x40` | 2 bytes |
| D + `0x4a` | F-ext-01 from R + `0x42`, length `2` | Requested 2 bytes |
| D + `0x4c` | Direct read from R + `0x44` | 4 bytes |
| D + `0x50` | Direct read from R + `0x48` | 2 bytes |
| D + `0x52` | F-ext-01 from R + `0x4a`, length `2` | Requested 2 bytes |
| D + `0x2e` | Direct read from R + `0x5a` | 1 byte |

The final byte write overwrites the earlier value at D + `0x2e`.

If the first check is nonzero, this entire group is skipped. No local clearing of these destinations occurs in that branch.

### 4. Common trailing fields and repeated discriminator checks

The function next performs these actions regardless of the first discriminator result:

| Destination | Source or action | Width |
|---|---|---:|
| D + `0x1e` | Direct read from R + `0x34` | 2 bytes |
| D + `0x20` | F-ext-01 from R + `0x32`, length `2` | Requested 2 bytes |
| D + `0x24` | Direct read from R + `0x38` | 4 bytes |
| D + `0x30` | F-ext-01 from R + `0x3f`, length `4` | Requested 4 bytes |

Thus R + `0x32` supplies arguments to two distinct transfer calls: one targeting D + `0x1c`, the other targeting D + `0x20`. These are separate calls, not one cached result.

The function then rereads the 16-bit discriminator at D + `0x12`:

- Zero selects source address R + `0x5f`.
- Nonzero selects source address R + `0x43`.

It calls F-2eceb10e9abc with D + `0x34`, the selected source address, and length `0x0b`, requesting an eleven-byte transfer. No explicit terminator is written after this operation.

After that call, the function reads the discriminator a third time:

- If nonzero, it calls F-2b1ba6d8de9d with `(R, 1)` and returns `1`.
- If zero, it proceeds to the second acquisition.

These three discriminator reads are distinct. They must not be treated as one immutable decision unless additional dependency behavior guarantees stability. Intervening calls may affect reachable memory or global state; such effects are not excluded by the evidence.

### 5. Second acquisition and offset arithmetic

For the second acquisition, the function reads:

- An unsigned 16-bit value from D + `0x50`.
- An unsigned 16-bit value from D + `0x0a`, shifted right by nine bits.

The second argument to F-a63889a8554d is their unsigned 32-bit product. Its other arguments are A, B, `1`, and `0`, giving the argument sequence `(A, product, B, 1, 0)`.

The two destination reads used in the product have no established relative evaluation order.

The shifted factor ranges from 0 through 127. The product therefore ranges from 0 through 8,322,945 (`0x7eff81`), so this multiplication cannot overflow 32 unsigned bits. No additional scaling or unit conversion is visible.

If the second acquisition returns zero, the function calls F-2b1ba6d8de9d with `(R, 1)` and returns `0`. Previously populated destination fields are not rolled back.

### 6. Marker search and final fields

After a nonzero second acquisition result, the function searches native 32-bit words beginning at B. It uses an unsigned 32-bit byte-offset counter initialized to zero and advances the search pointer by four bytes per continuation.

Let **M** denote the value supplied by the referenced marker data object. Its numeric value, mutability, and storage properties are unknown.

At each candidate:

1. The visible condition first compares M with the current 32-bit word using unsigned less-than-or-equal.
2. It separately checks whether the current word differs from M.
3. When that inequality holds, the stopping decision instead becomes whether the counter exceeds `0x1ff`.

For stable reads, the effective stopping rule is: stop on equality with M, or stop on a mismatch once the counter is greater than 511.

On continuation, the counter becomes its previous value plus four, masked with `0xfffeffff`, and the pointer advances by one 32-bit word. The mask clears bit 16. On the reachable counter sequence `0, 4, …, 512`, that bit is already clear, so the mask does not change the value.

For stable memory, candidate offsets are B + `0x000` through B + `0x200`, inclusive: at most 129 candidates. A marker at offset `0x200` is accepted. The search is therefore not limited to the first 512 bytes.

After stopping, the function reads the current word again:

- If it equals M, it writes the following 32-bit word to D + `0x54`, then the next 32-bit word to D + `0x58`.
- Otherwise, it writes 32-bit zero to D + `0x54`, then 32-bit zero to D + `0x58`.

The first payload read and destination write precede the second payload read and destination write. Aliasing can therefore matter.

A match at B + `0x200` requires payload reads through B + `0x20b`, equivalent to R + `0x223`. Adequate readable storage is required; the function does not check this extent.

Finally, it calls F-2b1ba6d8de9d with `(R, 1)` and returns `1`, whether or not the marker was found.

## Side effects

Established local effects include destination writes, native-width resource reads, dependency calls, and an advancing search pointer.

There is no transaction or rollback:

- The initial dependency runs even if resource acquisition subsequently fails.
- Initial acquisition failure occurs before the visible field-population sequence.
- Second acquisition failure occurs after substantial destination population.
- The nonzero-discriminator success path does not locally write D + `0x54` or D + `0x58`.
- Marker absence after a successful second acquisition explicitly zeroes both final 32-bit fields.

For every nonzero resource result, exactly one visible call to F-2b1ba6d8de9d occurs on each normal return path. A zero resource result causes no such call.

This call pattern does not prove that storage is released. No resource dereference follows that call, and its result is not inspected.

No guarantees are established concerning dependency-induced global changes, retained pointers, callbacks, synchronization, or exception behavior.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-4042d12a1376 | Called first with D, `0x5c`, zero, X, X. Result not inspected. |
| F-2f4031632ae3 | Returns the resource value; zero selects immediate failure. |
| F-a63889a8554d | Called once initially and conditionally a second time. Zero indicates failure to this caller; every nonzero result is accepted. |
| F-2eceb10e9abc | Receives destination, source, and requested lengths of eight and eleven bytes. |
| F-ext-01 | Document-local ID for the unmapped external dependency receiving destination, source, and requested lengths of two or four bytes. |
| F-2b1ba6d8de9d | Called with the nonzero resource value and `1` before every subsequent normal return. |

Any more specific callee behavior is **documented_only** unless established independently.

## Acceptance examples

These examples are conditional specifications, not runtime results. Memory examples assume valid storage, stable marker data, and no unmentioned dependency side effects.

1. **Resource unavailable.**  
   If F-2f4031632ae3 returns zero, the result is `0`. The initial preparation call has already occurred. Neither acquisition nor finalization is called.

2. **Initial acquisition fails.**  
   With a nonzero resource and a zero first acquisition result, finalization receives `(R, 1)` once and the result is `0`. The visible field-population sequence does not execute.

3. **Stable nonzero discriminator.**  
   If the discriminator remains nonzero at all three checks, the extended-field group is skipped, the eleven-byte transfer uses R + `0x43`, no second acquisition occurs, and the result is `1` after finalization.

4. **Stable zero discriminator and arithmetic boundary.**  
   If the discriminator remains zero, D + `0x50` contains unsigned `0xffff`, and D + `0x0a` contains unsigned `0xffff`, the second acquisition receives `0x7eff81` as its second argument. The extended-field group executes, and the eleven-byte transfer uses R + `0x5f`.

5. **Second acquisition fails.**  
   If the final discriminator check is zero and the second acquisition returns zero, the result is `0` after finalization. Earlier destination updates remain; the function performs no marker search or local final-pair writes.

6. **Marker at the first candidate.**  
   If the word at B equals M, the words at B + `0x04` and B + `0x08` become the final pair at D + `0x54` and D + `0x58`. The result is `1`.

7. **Marker at the boundary candidate.**  
   If all earlier candidates differ from M and B + `0x200` equals M, the words at B + `0x204` and B + `0x208` supply the final pair. This candidate is accepted despite its counter exceeding 511.

8. **Marker absent.**  
   If all 129 candidates differ from M, the final pair is zeroed. No candidate at B + `0x204` is searched. The result remains `1`.

9. **Discriminator changes between checks.**  
   If the first check is nonzero but an intervening dependency changes the discriminator to zero, later behavior follows the later reads. The function does not retroactively populate the skipped extended fields before computing the second-acquisition argument.

## Uncertainties

The following remain **documented_only**:

- Field meanings, record format, acquisition units, and the marker’s numeric value.
- Whether initial preparation clears, validates, or transforms the destination.
- Whether either transfer dependency copies bytes unchanged.
- Resource allocation size, pointer-retention rules, and actual ownership or release semantics.
- Native byte order and platform rules for unaligned multibyte accesses.
- Permitted overlap between D, resource storage, and dependency-accessible memory.
- Dependency mutation of globals, source storage, discriminator bytes, or other destination fields.
- Behavior under concurrent mutation, faults, invalid addresses, or nonlocal exits.

The apparent intent is to decode one of two record variants and optionally retrieve marker-associated metadata. That interpretation is inferred, not an established semantic contract.

## Evidence

This contract is based solely on the supplied static function representation and dependency map. Established observations are limited to visible argument passing, access widths and offsets, branch conditions, arithmetic, call order, and explicit returns.

No runtime tests have been performed. Acceptance examples describe conditions and expected consequences; they are not observed executions. No snapshot hash or independently verified provenance is asserted.
