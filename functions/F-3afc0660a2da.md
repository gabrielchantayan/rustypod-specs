<!-- Trusted provenance: F-3afc0660a2da; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3afc0660a2da

## Purpose

F-3afc0660a2da traverses records selected by a trailing offset table and invokes dependencies on particular record fields. It supports two record formats, selected by values 3 and 4. A separate direction argument determines whether certain length, count, and discriminator fields are processed before or after they are used.

The ordering is consistent with an in-place representation conversion: fields needed for navigation are processed early in one direction and late in the other. That purpose is **inferred intent**, not an established dependency contract. The supplied material does not establish what the dependencies actually read, write, allocate, release, or return.

On every normal completion, this function returns the 32-bit value zero. It does not validate the structure or report a recoverable format error.

## Interface

The interface has four arguments:

| Argument | Established use |
|---|---|
| Descriptor address | Points to a structure from which a 32-bit base address and a 32-bit span are read. The base address is at descriptor byte offset 0; the span is at byte offset 12. |
| Format selector | A 32-bit value. Exactly 3 and 4 select traversal behavior. All other values skip record traversal, but not the initial descriptor and header reads. |
| Direction | A 32-bit value tested only for zero versus nonzero. All nonzero values select the same ordering. |
| Initial auxiliary value | A 32-bit value copied into local mutable storage. Format 4 can pass the address of that storage to F-697f02051d96. |

The return value is a 32-bit zero if execution reaches the normal return.

### Address and field notation

All offsets below are **byte offsets**.

- **B**: base address read once from descriptor offset 0 at entry.
- **S**: span read from descriptor offset 12 during initial setup.
- **N**: unsigned 16-bit record count stored at B + 10.
- **A**: offset-table address, initially calculated as B + S − 2 × N.
- **R**: address of the current record.
- **L**: unsigned 16-bit value read from the beginning of the current record at the specified point in processing.
- **T**: address R + L + 2.
- **D16(X)**: a call to F-df70a2b0a7d8 with address X.
- **D32(X)**: a call to F-bb2eb79124b3 with address X.

The D16 and D32 labels identify recurring call sites associated with apparent 16-bit and 32-bit fields. They do **not** establish dependency access widths or mutation semantics.

The supplied representation uses a 32-bit address model. Counts, offsets, and lengths loaded as unsigned 16-bit values are zero-extended before address arithmetic. Loop indices are unsigned 32-bit values.

### Preconditions for defined, successful operation

The caller must provide readable storage for the initial descriptor and header accesses, even for an unsupported format selector. For traversed records, all subsequently read fields and all dependency targets must be valid for their actual uses.

No buffer length, offset validity, alignment, or object-lifetime check is performed. If dependencies write through their arguments, the corresponding storage must be writable. The required bounds and alignment cannot be fully specified without the dependency contracts.

## Behavior

### Initial setup and record selection

Before checking the format selector, the function:

1. Reads and retains B.
2. Obtains S and the unsigned 16-bit count at B + 10.
3. Computes and retains A.
4. Initializes one local 32-bit auxiliary value from the fourth argument.

The supplied expression establishes the values used to compute A, but not a portable ordering between the individual reads contributing to that expression.

For formats 3 and 4, records are visited by ascending unsigned index, starting at zero. The loop condition rereads the unsigned 16-bit count at **the retained B + 10** on every condition evaluation. The initial count is not a fixed traversal limit.

For each selected index:

- A 16-bit unsigned offset is read from A + 2 × index.
- The descriptor’s base-address field is read again.
- R is formed from that freshly read base address plus the selected offset.

Thus A and the header address used for loop bounds remain based on the entry-time B, while each record address uses a fresh descriptor-base read. The evaluation order of the offset-table read and fresh descriptor-base read within this address expression is not established.

Changes made by dependencies can affect later count evaluations, table entries, record addresses, and record contents. The function does not snapshot the table or the records.

### Format 3

Each record is processed in this order:

1. If direction is zero, invoke D16(R).
2. Read the unsigned 16-bit length at R and calculate T = R + L + 2.
3. If direction is nonzero, invoke D16(R).
4. Invoke D32(R + 4).
5. Invoke D32(R + 8).
6. Read the single-byte flag at retained B + 8.
7. Process the tail according to that flag.

If the flag is zero, invoke D32(T) once.

If the flag is nonzero, process eight consecutive eight-byte groups. For each group, invoke D32 on its first four-byte position and then on its second four-byte position. The resulting call order is T, T + 4, T + 8, continuing in four-byte increments through T + 60.

The flag is read after both fixed-header D32 calls. A dependency-induced flag change before that read can therefore select a different tail layout. Once selected, the nonzero-flag branch makes its fixed sixteen calls without rereading the flag.

T remains the address already calculated from the length read in step 2. Later changes to the record’s length field do not recalculate T.

### Format 4: common record prefix

Each record begins with the following sequence:

1. If direction is zero, invoke D16(R).
2. Read and retain the unsigned 16-bit length L at R; calculate T = R + L + 2.
3. If direction is nonzero, invoke D16(R).
4. Invoke D32(R + 2).
5. If direction is zero, invoke D16(R + 6).
6. Traverse the unsigned 16-bit count at R + 6. For each index starting at zero, invoke D16(R + 8 + 2 × index).
7. If direction is nonzero, invoke D16(R + 6).
8. Read the single-byte flag at retained B + 8.

The count at R + 6 is reread for every loop condition. It is not cached. Consequently, changes to that count during element processing affect subsequent iterations.

If the flag is zero, invoke D32(T). No discriminator lookup or discriminator-dependent processing occurs for that record.

If the flag is nonzero, use the extended-tail behavior below.

### Format 4: extended-tail discriminator

For a nonzero flag:

1. If direction is zero, invoke D16(T).
2. Invoke F-697f02051d96 with the address of the local auxiliary value as its first argument and T as its second argument.
3. Inspect the low 16 bits of the auxiliary value.
4. Perform the matching subtype processing.
5. If direction is nonzero, invoke D16(T), after all subtype-specific processing.

The subtype comparison is expressed through a signed 16-bit interpretation, but the only recognized values are 1, 2, 3, and 4. Every other low-16-bit pattern takes the unrecognized-subtype path. The upper 16 bits do not affect selection.

The local auxiliary value is initialized only once, not once per record. If F-697f02051d96 leaves some or all of it unchanged, values from the initial argument or an earlier record can affect later subtype selection.

#### Subtype 1

Invoke dependencies in this exact order:

| Order | Dependency | Target |
|---|---|---|
| 1 | D16 | T + 2 |
| 2–10 | D32 | T + 4, T + 8, T + 12, T + 16, T + 20, T + 24, T + 28, T + 32, T + 36 |
| 11 | D16 | T + 42 |
| 12 | D32 | T + 44 |
| 13 | D32 | T + 80 |

There are no direct calls targeting T + 40 or T + 48 through T + 76 in this subtype. This does not establish that those bytes remain unchanged: dependency access ranges are unknown.

#### Subtype 2

Invoke dependencies in this exact order:

| Order | Dependency | Target |
|---|---|---|
| 1 | D16 | T + 2 |
| 2–9 | D32 | T + 8, T + 12, T + 16, T + 20, T + 24, T + 28, T + 32, T + 36 |
| 10 | D16 | T + 42 |
| 11 | D32 | T + 44 |
| 12 | D32 | T + 80 |
| 13 | F-68c3320a2751 | T + 88 |
| 14 | F-68c3320a2751 | T + 168 |

Unlike subtype 1, subtype 2 has no D32 call at T + 4.

#### Subtypes 3 and 4

Both recognized values select the same behavior:

1. Invoke D32(T + 4).
2. If direction is zero, invoke D16(T + 8).
3. Traverse the unsigned 16-bit count at T + 8. For each index starting at zero, invoke D16(T + 10 + 2 × index).
4. If direction is nonzero, invoke D16(T + 8).

The count is reread on each loop-condition evaluation. These element calls are separate from the common-prefix element calls based on the count at R + 6.

For nonzero direction, the final D16(T) occurs after the D16(T + 8) in step 4.

#### Unrecognized subtype

No subtype-specific field calls occur. The discriminator lookup still occurs. D16(T) still occurs before that lookup for zero direction or after it for nonzero direction.

An unrecognized subtype does not directly cause an error return.

### Arithmetic and malformed structures

The multiplication 2 × N cannot overflow a 32-bit unsigned quantity because N is at most 65,535. The other directly loaded counts, offsets, and lengths have the same unsigned 16-bit range.

The function performs no checked addition or subtraction for address calculations. It does not reject:

- A span smaller than twice the record count.
- An offset outside an enclosing allocation.
- A length whose resulting tail falls outside the record.
- A count inconsistent with available element storage.
- Overlapping records or fields.
- A table overlapping records.
- An unsupported format selector.
- An unrecognized extended-tail subtype.

The supplied 32-bit address model permits wraparound at the machine-address level; the representation alone is not a portable guarantee that overflowing signed expressions are valid. Acceptance requires valid resulting addresses rather than an invented overflow-recovery behavior.

## Side effects

The function’s visible work consists primarily of ordered dependency calls and the reads that select their targets.

There are no explicit writes to caller-provided storage outside dependencies. The local auxiliary value is explicitly initialized and is subsequently exposed by address to F-697f02051d96.

The visible function contains no direct allocation, release, ownership-transfer, or global-variable operation. That does not establish that its dependencies lack those effects. In particular:

- No dependency is known to preserve the descriptor, header, table, record fields, or globals.
- No dependency is known to preserve the lifetime of storage passed to it.
- No dependency is known to restrict its accesses to the nominal field width suggested by its call sites.
- The lifetime requirements of the local auxiliary address after F-697f02051d96 returns are unknown. This function provides only local storage.

Dependency return values are not used to report failure. Whether a dependency can fail, terminate execution, or transfer control abnormally is unknown. Zero is guaranteed only on normal completion.

## Dependencies

| Opaque ID | Role established by call sites | Unknown contract details |
|---|---|---|
| F-df70a2b0a7d8 | Receives addresses of apparent 16-bit lengths, counts, discriminators, and elements. Its placement depends on direction for navigation fields. | Actual access width, transformation, return value, global effects, ownership effects, failure behavior. |
| F-bb2eb79124b3 | Receives addresses at apparent four-byte field positions. | Actual access width, transformation, return value, global effects, ownership effects, failure behavior. |
| F-697f02051d96 | Receives a pointer to persistent local 32-bit auxiliary storage and the current extended-tail address. Subsequent subtype selection uses the auxiliary value’s low 16 bits. | Whether it writes that value, how many bytes it writes, how it interprets the tail, and its other effects. |
| F-68c3320a2751 | Receives two addresses, in separate ordered calls, for subtype 2 only. | Layouts, accessed extents, transformations, ownership effects, and failure behavior. |

The direction argument is not forwarded to these dependencies. Directional behavior is achieved by changing call placement relative to reads.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Except where stated otherwise, assume all accesses are valid, dependencies return normally, and they do not change loop bounds, table entries, descriptor fields, or flags.

1. **Unsupported format selector.**  
   With a selector other than 3 or 4, initial setup reads still occur. No dependency is called for record processing, and the function returns zero.

2. **Zero record count.**  
   With a supported format and zero at retained B + 10, no record is selected and no dependency is called. Initial setup still occurs.

3. **Format 3, zero direction, zero flag.**  
   For one record, the call sequence is D16(R), D32(R + 4), D32(R + 8), D32(T). T uses the length read after the first call.

4. **Format 3, nonzero direction, nonzero flag.**  
   T uses the length read before D16(R). The two fixed-header calls precede sixteen tail calls at T through T + 60 in increasing four-byte increments.

5. **Format 4 common-prefix count conversion.**  
   Suppose direction is zero and D16(R + 6) changes the count from one representation into the usable value 2. The element calls target R + 8 and R + 10, provided that count remains 2. With nonzero direction, the usable count is read before the final count-processing call.

6. **Subtype 2 versus subtype 1.**  
   If the discriminator lookup leaves low 16 bits equal to 2, the subtype omits the call at T + 4 and includes calls to F-68c3320a2751 at T + 88 and T + 168. A resulting discriminator of 1 selects the different sequence specified for subtype 1.

7. **Subtype 3, nonzero direction, count 2.**  
   After the common prefix and discriminator lookup, subtype processing calls D32(T + 4), D16(T + 10), D16(T + 12), and D16(T + 8), followed by the final D16(T).

8. **Unrecognized discriminator.**  
   If the resulting low 16 bits are 5, no recognized-subtype sequence runs. Normal completion still returns zero. The direction-dependent D16(T) is not omitted.

9. **Auxiliary value persists across records.**  
   If the lookup writes subtype 1 for one record and leaves the auxiliary value untouched for the next, the next extended tail also selects subtype 1.

10. **Observable bound mutation.**  
    If a dependency changes the count at retained B + 10 to a value no greater than the next record index, traversal stops at the next loop-condition check. This demonstrates the live bound; it is not an assurance that arbitrary mutations are safe.

## Uncertainties

**Established static evidence:** the supplied representation establishes the caller-side field reads, retained addresses, dependency targets, relative call order, branch conditions, loop-bound rereads, and normal zero return described above.

**Inferred intent:** the paired early-versus-late handling of 16-bit navigation fields suggests directional byte-order or representation conversion. The apparent widths suggest corresponding field layouts. Neither interpretation establishes dependency behavior.

**documented_only unknowns:**

- The external format name, header schema, and semantic meaning of each field.
- Whether S denotes allocation size, serialized size, or another boundary.
- Byte order and alignment requirements.
- All dependency implementations and their actual memory footprints.
- Input/output aliasing restrictions.
- Whether mutation of traversal metadata is supported usage.
- Failure behavior for malformed or inaccessible storage.
- Complete minimum record sizes, especially the extents accessed by F-68c3320a2751.
- Whether the fourth argument is intended as a default discriminator, context, or another value.

## Evidence

This contract is derived solely from the supplied static function representation and opaque dependency mapping. Dependency implementations were not provided.

No runtime tests, dependency executions, memory-access traces, or independent implementation inspection have been performed. The acceptance examples specify conditional observable behavior; they are not reports of measured results.
