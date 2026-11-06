<!-- Trusted provenance: F-a39eff8eec40; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a39eff8eec40

## Purpose

F-a39eff8eec40 selects one of two regions within an associated object, invokes F-5ef10d079522, and then writes a derived 16-bit pattern at the beginning of the caller’s output area.

Selection depends on a 32-bit descriptor field and, when that field is nonzero, a byte accessed through the associated object. The dependency receives the selected region address, an accompanying 32-bit descriptor value, an output address displaced by two bytes, the address of initialized 32-bit local storage, and a value read from a referenced global datum.

The meanings of the selected regions, descriptor values, global value, and output are not established. Interpreting them as buffers, lengths, counts, format selectors, or encoded records would be inferred intent rather than established behavior.

## Interface

The function accepts four arguments and supplies no explicit return value.

| Argument | Established use |
|---|---|
| Descriptor address | Identifies a structure read through 32-bit fields at byte offsets 0, 44, and 48. |
| Output address | Receives a final 16-bit store after the dependency returns normally. An address two bytes beyond it is passed to the dependency. |
| Third argument | Has no observable use in the supplied body. Its represented width is 32 bits; no semantic interpretation is established. |
| Initial auxiliary value | A 32-bit unsigned value used to initialize local in/out storage when the initial descriptor field at offset 48 is zero. Otherwise, the associated object address replaces it. |

### Widths and address model

The represented accesses use 8-bit bytes, 16-bit halfwords, and 32-bit words. Address-bearing values and the relevant address arithmetic use a 32-bit model. No portable interface for wider pointers is established.

The final calculation operates on an unsigned 32-bit value and produces a 16-bit stored pattern. The fifth dependency argument is a value read from a global datum; its width, signedness, and semantic type are not established by the supplied body.

### Accessed layouts

The descriptor has these relevant fields:

| Byte offset | Width | Role |
|---|---:|---|
| 0 | 32 bits | Associated object address. |
| 44 | 32 bits | Accompanying value for the region at associated-object offset 280. |
| 48 | 32 bits | Initial selector and accompanying value for the region at associated-object offset 368. |

The associated object has these relevant locations:

| Byte offset | Width or extent | Role |
|---|---:|---|
| 8 | 32 bits | Secondary-object address, read only when the initial selector is nonzero. |
| 280 | Unknown extent | First selectable region start. |
| 368 | Unknown extent | Second selectable region start. |

The secondary object is read for one unsigned byte at offset 2831. The supplied body establishes neither a region extent nor a payload read from either selected region before delegation.

### Preconditions and validation

Ordinary execution requires validity of every access taken on the selected path, the global-datum read, the dependency’s accesses, and the final halfword store.

The body performs no explicit validation of:

- Descriptor, associated-object, secondary-object, or output addresses.
- Alignment.
- Selected region extent.
- Descriptor values.
- Output capacity.
- Address-addition overflow.
- The global datum’s value.

A zero initial selector bypasses the secondary-object lookup, but not the later read of the associated object address.

Output capacity cannot be determined from this function. Its own final store occupies two bytes, but F-5ef10d079522 receives an address at output offset 2 and has an unknown access extent.

## Behavior

### Initial selection

The first descriptor read obtains the unsigned 32-bit field at offset 48. That saved value determines whether the secondary-object lookup occurs.

**If the saved selector is zero:**

1. The function skips the associated-object and secondary-object reads used to obtain the flag.
2. It reads the descriptor field at offset 44 as the accompanying value.
3. It then reads the associated object address from descriptor offset 0.
4. It forms the selected region address by adding 280 bytes.
5. The supplied fourth argument remains the value used to initialize local in/out storage.

**If the saved selector is nonzero:**

1. The function reads the associated object address from descriptor offset 0.
2. That captured address replaces the supplied fourth argument for subsequent local initialization.
3. It reads the secondary-object address at captured associated-object offset 8.
4. It reads an unsigned byte at secondary-object offset 2831.

A zero flag selects the offset-280 region:

- The function first reads the descriptor field at offset 44.
- It then rereads the associated object address from descriptor offset 0.
- The newly read address, plus 280 bytes, becomes the selected region address.
- Local in/out storage will nevertheless be initialized with the earlier object address captured before the flag lookup.

A nonzero flag selects the offset-368 region:

- The selected region address is the previously captured object address plus 368 bytes.
- The accompanying value is the original selector read from descriptor offset 48.
- Descriptor offset 44 is not read on this path.
- Local in/out storage will be initialized with the captured object address.

Thus, both offset-280 paths read descriptor offset 44 before the descriptor offset-0 read used to form the region address. On the nonzero-selector, zero-flag path, that offset-0 read is a reread; on the zero-selector path, it is the first associated-object-address read.

These are the orders represented by the supplied statements. They do not establish synchronization, atomicity, volatile-access semantics, or additional guarantees for concurrent or memory-mapped mutation.

### Local initialization and delegation

Once the region and accompanying value are selected, the function initializes a writable 32-bit local storage location:

| Selection path | Initial local value |
|---|---|
| Initial selector zero | Supplied fourth argument. |
| Initial selector nonzero, flag zero | Associated object address captured before the flag lookup. |
| Initial selector nonzero, flag nonzero | Associated object address captured before the flag lookup. |

After this initialization, and before entry into F-5ef10d079522, the function reads the referenced global datum to obtain the fifth dependency argument. The body does not establish that this datum is constant. Its value is not selected by the descriptor branch.

F-5ef10d079522 receives five arguments in this order:

1. Selected region address.
2. Selected 32-bit descriptor value.
3. Caller’s output address plus two bytes.
4. Address of the initialized 32-bit local in/out storage.
5. Value read from the referenced global datum.

The output displacement is two bytes, not two 32-bit elements. The accompanying descriptor value is passed at full 32-bit width without visible scaling, halving, truncation, or range checking.

There is one reachable dependency invocation. No dependency return value is inspected. Subsequent behavior uses the actual contents of the local storage after the dependency returns, whether or not the dependency changed them.

### Final output calculation

After a normal dependency return, the function reads the local 32-bit value. It shifts that unsigned value left by 15 bits within a 32-bit width, discarding bits shifted beyond that width, and then logically shifts the result right by 16 bits.

The resulting 16-bit pattern is stored at the original output address. Equivalently, for post-call local value \(V\):

\[
\text{stored pattern}
=
\left\lfloor \frac{V}{2} \right\rfloor \bmod 65{,}536.
\]

This selects bits 1 through 16 of \(V\):

- Bit 0 is discarded.
- Bit 16 becomes the stored pattern’s highest bit.
- Bits 17 through 31 do not affect the result.
- No sign extension contributes to the calculation.
- The final direct write occupies exactly two bytes.

The represented conversion through a signed 16-bit type does not alter the stored bit pattern. The function establishes no signed or unsigned interpretation for consumers.

Byte order is not established, so this contract specifies the halfword pattern rather than an ordered pair of bytes.

### Unreachable alternative

The supplied body contains a nested alternative involving F-c45cb3f8e7fd. Entry into its enclosing branch requires the saved selector and saved flag to be nonzero. Entry into the nested alternative would require at least one of those same saved values to be zero.

No intervening operation changes those saved values. The nested alternative is therefore unreachable under ordinary execution of the represented control flow.

Its apparent null-output checks, alternative dependency calls, and full-word output shifts do not establish reachable behavior. They provide neither null-output support nor an alternative output format.

## Side effects

The represented reachable order is:

1. Read descriptor offset 48.
2. If that value is nonzero, read descriptor offset 0, then associated-object offset 8, then secondary-object offset 2831.
3. Complete selection:
   - For either offset-280 path, read descriptor offset 44 before reading or rereading descriptor offset 0 to form the region address.
   - For the offset-368 path, use the already captured address and selector.
4. Initialize the 32-bit local in/out storage.
5. Read the referenced global datum for the fifth dependency argument.
6. Enter F-5ef10d079522.
7. After normal return, read the local storage.
8. Store the derived 16-bit pattern at the original output address.

This ordering describes the supplied representation, not a platform memory-ordering guarantee.

The function performs no explicit allocation, deallocation, ownership transfer, direct descriptor write, or direct global write. Those observations do not constrain the dependency’s behavior. In particular, the function does not establish that F-5ef10d079522 preserves globals, preserves objects, or refrains from allocating or releasing memory.

Aliasing is not checked. If the output overlaps a descriptor, associated object, or other writable storage, the final halfword store can modify that storage. This direct store occurs after selection, after the global-value read, and after the dependency returns. Dependency writes may occur earlier; their locations and ordering are unknown.

The local in/out address is available during the dependency call. Its passage does not establish permission for the dependency to retain it or safely use it after F-a39eff8eec40 returns.

There is no explicit error status or recovery path. If an access faults or the dependency does not return normally, occurrence of the final store is not established. If execution reaches the final operation normally, that store is unconditional.

## Dependencies

### F-5ef10d079522

The sole reachable processing dependency receives:

- A selected region address.
- An accompanying 32-bit descriptor value.
- An output address displaced by two bytes.
- A pointer to initialized, writable 32-bit local storage.
- A value read from the referenced global datum after local initialization and before dependency entry.

The fifth argument is not established as a fixed constant. Its value, interpretation, width, and signedness remain unknown.

Other unknowns include the dependency’s return convention, failure behavior, reads, writes, output extent, allocation or release behavior, retained references, and effects on objects or global state.

Initialization of local storage does not imply preservation. The final prefix depends on its actual post-call value.

### F-c45cb3f8e7fd

This dependency appears only in the unreachable nested alternative. No reachable invocation is established for F-a39eff8eec40.

Its apparent usage does not establish an operational fallback, ownership rule, null-output case, or second output representation.

## Acceptance examples

These are conditional consequences of static evidence, not runtime results. Except for the invalid-output example, they assume valid accesses and normal dependency return.

### Zero initial selector

Given zero at descriptor offset 48:

- No secondary-object pointer or flag-byte read occurs.
- Descriptor offset 44 is read before descriptor offset 0.
- The dependency receives the offset-44 value and the subsequently read object address plus 280.
- Local storage initially contains the supplied fourth argument.
- The global datum is read after that initialization and supplies the fifth dependency argument.

If the dependency leaves the local value equal to 10, the final output halfword is 5.

### Nonzero selector and zero flag

Given a nonzero initial selector and a zero flag:

- An associated object address is captured before the flag lookup.
- Descriptor offset 44 is subsequently read before descriptor offset 0 is reread.
- The reread object address plus 280 supplies the region address.
- The earlier captured address initializes local storage.

If the earlier capture and the later read produce different addresses, the local initializer and region base differ accordingly. This describes the represented read dependence; it does not promise defined concurrent-mutation behavior.

### Nonzero selector and nonzero flag

Given an initial selector of 7 and any nonzero flag:

- The captured associated object address plus 368 supplies the region address.
- The accompanying value is 7.
- Descriptor offset 44 is not read.
- Local storage initially contains the captured object address.

Flag values 1 and 255 have the same selection effect.

### Global argument value

If the global-datum read produces value \(G\), the dependency receives \(G\) as its fifth argument. No particular numeric value is required by the supplied body.

A later mutation of that datum does not create another explicit read in this function before the final store. The dependency’s own accesses remain unknown.

### Width-sensitive output

| Post-call 32-bit local value | Final 16-bit pattern |
|---:|---:|
| 0 | 0 |
| 1 | 0 |
| 2 | 1 |
| 65,536 | 32,768 |
| 131,070 | 65,535 |
| 131,072 | 0 |
| 4,294,967,295 | 65,535 |

These cases distinguish extraction of bits 1–16 from saturation or a full-width divided-value store.

### Null output

There is no reachable null-output early return. Under the represented 32-bit address model, a zero output address produces dependency argument address 2. If execution returns normally from the dependency, the function attempts a halfword store at address zero.

Whether either address causes a fault is environment-dependent; null output is not established as supported.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings, extents, validity constraints, and ownership of selected regions.
- Units and accepted ranges of descriptor values.
- Value, interpretation, width, signedness, and mutability rules of the referenced global datum.
- Dependency reads, writes, errors, allocation, release, retained references, and global effects.
- Required output capacity beyond the visible two-byte store.
- Byte order, alignment behavior, and invalid-address consequences.
- Language-level portability and consequences of address arithmetic outside valid storage.
- Concurrent-mutation, memory-mapped-access, and synchronization semantics.
- Why the contradictory nested alternative appears in the supplied representation.

An **inferred intent** is that the output prefix records a reduced form of a value produced or updated during delegated processing. No particular format or application meaning follows from that inference.

## Evidence

Established behavior comes from the supplied function body: access widths and offsets, saved branch conditions, descriptor-read order, local initialization, the global-datum read, dependency argument positions, unsigned shift arithmetic, and the final halfword store.

In particular, both offset-280 paths obtain descriptor offset 44 before the offset-0 read used to form the region address. The global datum supplies the fifth argument after local initialization and before dependency entry; the body does not establish a constant value or additional type details.

Unreachability follows from contradictory conditions on saved values, without assuming any dependency preserves globals.

No dependency implementation, runtime trace, external interface definition, or test result was supplied. No runtime tests have been performed.
