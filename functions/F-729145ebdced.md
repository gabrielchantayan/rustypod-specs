<!-- Trusted provenance: F-729145ebdced; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-729145ebdced

## Purpose

F-729145ebdced constructs a record in a requested 72-byte storage region or, if the storage request returns null, in a temporary region. The intended record contains:

- A caller-supplied or resolved 32-bit primary value.
- Seven additional 32-bit input values, Fields A through G.
- A one-byte discriminator with value 110.
- A caller-supplied auxiliary byte.
- Five 32-bit sentinel values of −1.

The seven additional inputs occupy intended byte offsets 12, 16, 20, 24, 28, 44, and 48.

The auxiliary-byte destination requires special qualification: its address is formed by converting the record pointer to a 32-bit integer, adding 5, and converting the result to a byte-addressable pointer. Interpreting that operation as a write at record offset 5 requires address-preserving conversions and valid integer addition. The supplied material does not independently establish those platform properties.

On the ordinary successful-storage path, the populated storage pointer is returned. On the temporary-storage path, an initialization dependency is invoked before population, the populated region is passed to F-8d9399c13232, and null is returned.

**Inferred intent:** the function constructs a typed operation or object record, with default-value resolution and an alternative processing path when storage is unavailable. Domain meanings, ownership rules, and equivalence between the two completion paths are not established.

## Interface

### Inputs

The function has ten positional inputs.

| Position | Input | Representation and use |
|---|---|---|
| 1 | Context reference | Pointer supporting a 4-byte read. The pointer itself is also passed to F-a9dcd3e71f90 when the primary input is zero. |
| 2 | Primary value | 32-bit integer. Used directly when nonzero; otherwise replaced by a dependency result. |
| 3 | Field A | 32-bit integer, intended offset 12. |
| 4 | Field B | 32-bit integer, intended offset 16. |
| 5 | Field C | 32-bit integer, intended offset 20. |
| 6 | Field D | 32-bit integer, intended offset 24. |
| 7 | Field E | 32-bit integer, intended offset 28. |
| 8 | Auxiliary byte | 8-bit value, written through the integer-derived address described below. |
| 9 | Field F | 32-bit integer, intended offset 44. |
| 10 | Field G | 32-bit integer, intended offset 48. |

The field labels identify positions, not application meanings.

The supplied representation establishes 4-byte record words and one-byte accesses. Pointer width and multi-byte byte order are not independently established. The primary input is tested only for equality to zero; no range checks are shown for any payload input.

### Output

Assuming normal completion and valid memory operations:

- If the selected record address differs from the temporary-region address, that selected address is returned.
- If the addresses are equal, F-8d9399c13232 is called with the selected address, and null is returned.

A nonnull storage-request result ordinarily follows the first rule. A null storage-request result selects the temporary region and follows the second.

The actual completion condition is address equality with the temporary region, not a separately retained allocation-success flag. Consequently, a nonnull storage-request result equal to that region would also select processing and a null return. Whether such a result is possible is not established.

There is no separate status output. A null return does not establish that no record was constructed or processed.

### Preconditions

For the intended, valid record-construction behavior:

- The context reference must support the initial 4-byte read.
- Selected storage must support the listed 32-bit and byte writes, including suitable alignment for word accesses.
- The auxiliary-byte address calculation must produce a valid writable address.
- To obtain the intended layout, that calculation must specifically produce the selected record address plus five bytes.
- Each dependency must satisfy its own invocation and lifetime requirements.

The function does not check the context reference for null, validate a nonnull storage result’s capacity, or verify that the integer-derived auxiliary address remains inside the record.

Normal-completion claims also require dependencies to return normally and not invalidate memory needed by subsequent operations.

## Behavior

### Initial read and storage selection

The first relevant memory operation is a 4-byte read through the context reference. Its value is captured before the storage request.

F-f053f140662a receives that captured value and the size 72. This request occurs regardless of whether primary-value resolution will be needed.

If the request returns nonnull, its result becomes the selected record address. The temporary initialization dependency is not invoked on that branch.

If the request returns null:

1. A temporary region containing eighteen 4-byte words becomes the selected record.
2. An unmapped initialization dependency receives the temporary address and 72.
3. Any required primary-value resolution follows.
4. Record population follows resolution.

The initializer’s name and implementation are not supplied through the opaque dependency mapping. Its call does not establish zero-filling, copying, allocation, or any particular initialization effect.

### Primary-value resolution

A nonzero primary input is retained. This includes any negative value under a signed interpretation.

A zero primary input causes two calls, in this order:

1. F-855c51f3aa90 receives the captured 32-bit context value, 107, zero, zero, and zero.
2. F-a9dcd3e71f90 receives the original context-reference pointer, zero, the preceding call’s result, and zero.

The second result becomes the primary value used in record population.

The context value supplied to F-855c51f3aa90 is the value captured before storage selection. This function does not reread the context location to obtain that argument. A dependency changing the context location does not, by itself, change the captured argument.

F-a9dcd3e71f90 receives the original pointer rather than the captured context value. Whether it reads or modifies the referent is unknown.

The replacement primary value is not checked. Zero, negative values, or dependency-specific error representations receive no special handling here. There is no retry.

### Auxiliary-byte address formation

The auxiliary-byte write is not expressed using width-independent pointer displacement. Its destination is obtained through a pointer-to-32-bit-integer conversion, integer addition of 5, and conversion back to a pointer suitable for a one-byte write.

The intended offset-5 interpretation therefore requires all of the following:

- The initial pointer conversion preserves the address information needed for this access.
- Adding 5 has defined, usable behavior for the resulting integer value.
- Converting the result back to a pointer yields the address five bytes after the selected record base.

A pointer representation wider than 32 bits could lose information in the conversion. Pointer width alone does not settle the issue: even on a 32-bit-address platform, the conversion rules, treatment of address bit patterns, and signed-integer boundary behavior matter.

The supplied material does not specify whether problematic cases truncate, wrap under target rules, fault, or lack defined behavior. No portable wraparound or recovery rule can be promised.

If the derived address is valid but differs from the intended offset, the auxiliary byte is written there at its specified point in the operation sequence. That write may affect unrelated memory or overlap another record field. If it overlaps an earlier write, it can change that earlier value; if it overlaps a later write, the later write may overwrite it. Invalid address formation or access provides no established normal-completion behavior.

### Intended record layout

The table describes the completed layout under the address-preservation precondition above, assuming dependencies leave the selected storage valid and no concurrent actor modifies it.

Offsets are byte offsets from the selected record address. Ranges are inclusive.

| Byte range | Width | Directly assigned value |
|---|---:|---|
| 0–3 | 32 bits | Retained or resolved primary value |
| 4 | 8 bits | Discriminator 110, hexadecimal 6E |
| 5 | 8 bits | Auxiliary input byte, subject to the address-formation condition |
| 6–11 | 6 bytes | No direct assignment |
| 12–15 | 32 bits | Field A |
| 16–19 | 32 bits | Field B |
| 20–23 | 32 bits | Field C |
| 24–27 | 32 bits | Field D |
| 28–31 | 32 bits | Field E |
| 32–43 | 12 bytes | No direct assignment |
| 44–47 | 32 bits | Field F |
| 48–51 | 32 bits | Field G |
| 52–55 | 32 bits | Sentinel −1 |
| 56–59 | 32 bits | Sentinel −1 |
| 60–63 | 32 bits | Sentinel −1 |
| 64–67 | 32 bits | Sentinel −1 |
| 68–71 | 32 bits | Sentinel −1 |

The sentinel assignments are 32-bit assignments of −1. Their conventional two’s-complement representation is all ones; the supplied material does not separately establish the complete target representation.

Under the intended auxiliary-address interpretation, direct assignments cover 54 distinct record bytes. The other 18 bytes receive no direct assignment from this function. Their contents depend on prior storage contents and dependency behavior, not on an established zeroing guarantee.

Without that interpretation, the 54-byte coverage claim does not apply. The auxiliary write could land outside the record, in an otherwise untouched range, or on another assigned byte.

Multi-byte byte order remains unspecified. The discriminator is an explicit byte write; the auxiliary value is also written as one byte, wherever its address calculation directs it.

### Order of direct writes

After storage selection, any temporary initialization, and any primary resolution, the writes occur in this order:

1. Primary value at offset 0.
2. Fields A through E at offsets 12, 16, 20, 24, and 28.
3. Auxiliary byte at the integer-derived address.
4. Discriminator 110 at offset 4.
5. Sentinel −1 at offset 52.
6. Field F at offset 44.
7. Field G at offset 48.
8. Sentinels −1 at offsets 56, 60, 64, and 68, in that order.

This is the order represented by the supplied body. It does not establish atomic publication, memory barriers, or a concurrency protocol.

### Completion

After the listed writes, the selected address is compared with the temporary-region address.

If unequal, the selected pointer is returned without calling F-8d9399c13232.

If equal, F-8d9399c13232 receives that address. Its result is not used to select the function’s return value, which is null. No subsequent access to record contents is shown.

The layout describes the record at the end of direct population, before this final dependency call. F-8d9399c13232 might modify or otherwise affect the record; its behavior is unavailable.

## Side effects

Established direct effects comprise:

- The initial 4-byte context read.
- The storage-request call.
- The temporary initialization call on a null storage result.
- Two resolution calls when the primary input is zero.
- The ordered record writes and integer-derived auxiliary-byte write.
- The final processing call when the selected address equals the temporary-region address.

There is no dedicated store through the context-reference pointer. This does not guarantee that its referent remains unchanged: a dependency may modify it, or selected storage or the auxiliary destination may alias it.

No direct release operation is shown. The function does not establish:

- Ownership of returned storage.
- A required release API.
- Whether a dependency copies, retains, releases, or transfers memory.
- Whether dependencies preserve globals, referenced memory, or external resources.

The temporary region is local to the invocation. Any later use of its address requires a lifetime mechanism not evidenced here; no safe retention contract can be inferred merely from passing it to F-8d9399c13232.

The storage size and ordinary word offsets are fixed. There is no input-dependent size calculation. However, the auxiliary address calculation explicitly involves pointer conversion and integer addition, so truncation and integer-boundary concerns cannot be categorically excluded.

## Dependencies

| Opaque dependency | Established invocation | Unresolved boundary |
|---|---|---|
| F-f053f140662a | Captured context value; size 72 | Supplies candidate storage. Allocation, initialization, ownership, lifetime, and other effects are unknown. |
| F-855c51f3aa90 | Captured context value; 107; three zeros | Called only for a zero primary input. Its result is forwarded; meaning and effects are unknown. |
| F-a9dcd3e71f90 | Original context pointer; zero; preceding result; zero | Supplies the replacement 32-bit primary value. Mutation, error, and ownership semantics are unknown. |
| F-8d9399c13232 | Selected address when equal to the temporary region | Called after direct population. Its result does not affect the null return; processing and retention semantics are unknown. |

An additional initialization dependency receives the temporary-region address and 72 immediately after null-selected fallback. No opaque ID for it was supplied. Its identity and behavior remain **documented_only**; no standard initialization API is substituted.

## Acceptance examples

These are conditional specifications derived from static evidence, not executed tests. Normal-completion examples assume valid accesses, normally returning dependencies, and no interfering concurrent writes. Intended-layout examples additionally assume address-preserving auxiliary-byte formation.

### Nonzero primary and successful storage

Given primary value 25 and a nonnull writable 72-byte storage result distinct from the temporary region:

- F-f053f140662a receives the initially captured context value and 72.
- Neither resolution dependency is called.
- Offset 0 receives 25.
- All seven additional inputs occupy their listed offsets.
- The temporary initializer and F-8d9399c13232 are not called.
- The storage-result pointer is returned.

The 18 untouched bytes have no required value.

### Zero primary resolved to a nonzero value

Given primary zero and a replacement result of 91:

- Storage selection precedes resolution.
- F-855c51f3aa90 receives the captured context value and fixed arguments.
- F-a9dcd3e71f90 receives the original context pointer and the preceding result.
- The primary write stores 91.

Completion still depends on selected-address equality with the temporary region.

### Resolution returns zero

Given primary zero and a zero result from F-a9dcd3e71f90:

- The primary write stores zero.
- No second resolution attempt occurs.
- No special error return is introduced.

### Storage request returns null

Given a null storage result:

- The temporary initialization call precedes resolution and direct population.
- F-8d9399c13232 receives the populated temporary region.
- The function returns null after that call returns.

This null return is compatible with processing having occurred.

### Context changes after capture

Given initial context value 7, primary zero, and a storage dependency that changes the referenced location to 9 without invalidating required memory:

- F-855c51f3aa90 still receives 7.
- F-a9dcd3e71f90 receives the original pointer.
- Whether the latter reads or uses 9 is **documented_only**.

### Auxiliary-byte boundary and address preservation

Given auxiliary byte FF and an address-preserving conversion-and-addition sequence:

- Offset 5 receives FF.
- Offset 4 receives 6E.
- The byte write does not itself clear offsets 6 or 7.

If address preservation is not established, requiring FF at offset 5 is not justified. If the derived address instead validly aliases the low-address byte of the primary field, that byte is overwritten after the primary assignment, subject to the target’s byte order. This is a conditional consequence of write order, not an assertion that such an alias occurs.

### Integer-boundary case

If the pointer-derived integer cannot undergo the addition of 5 with established valid semantics, neither a successful offset-5 write nor a normal return is required by this specification. The source supplies no overflow check, alternative address calculation, or recovery path.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of the record, fields, discriminator 110, and selector 107.
- Pointer width, ABI details, integer-address conversion rules, and multi-byte byte order.
- Address preservation and boundary behavior of the auxiliary-byte calculation.
- Effects of the unmapped initialization dependency.
- Contents of bytes without direct assignments under the intended layout.
- Allocation, release, retention, copying, and ownership rules.
- Dependency effects on globals, memory, and external resources.
- Dependency error encodings and nonlocal-control-flow behavior.
- Equivalence, if any, between returned-record and temporary-processing paths.
- Thread safety and reentrancy.

## Evidence

Established static evidence consists of the supplied function body and opaque dependency mapping. It supports the initial context capture, 72-byte storage request, null-selected temporary path, conditional two-call resolution, seven additional 32-bit input fields, write ordering, five sentinel assignments, and final address-dependent pointer-or-null return.

The auxiliary-byte operation specifically includes pointer-to-32-bit-integer conversion and integer addition. Its interpretation as record offset 5 is conditional, not a width-independent fact.

No dependency implementations, platform conversion guarantees, ownership contracts, or runtime traces were supplied. No runtime tests have been performed. Acceptance examples describe consequences under stated conditions, not observed executions.
