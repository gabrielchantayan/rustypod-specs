<!-- Trusted provenance: F-e504f72ffcc9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e504f72ffcc9

## Purpose

F-e504f72ffcc9 attempts to construct an object from a descriptor and a three-word source record, optionally initialize it through a descriptor-supplied callback, and append its address to a context-owned table.

The visible success path initializes a four-word object header, obtains space for one additional table entry, increments the context’s entry count, and stores the object address in the new slot. Failure after successful construction invokes a cleanup dependency.

This is a source-facing contract. It distinguishes directly visible behavior from inferred purpose and unresolved dependency behavior. No runtime tests have been performed.

## Interface

### Inputs

The function accepts four arguments in a 32-bit address-and-word model:

| Argument | Visible interpretation |
|---|---|
| Descriptor address | Nullable address of a record containing at least two 32-bit words. The first word is passed to the construction dependency. The second word is interpreted as a nullable callback address. |
| Auxiliary value | An opaque 32-bit value passed unchanged to the construction dependency and, if invoked, the callback. It also initially supplies the upper half of the return value. |
| Source-record address | Nullable address of a record containing at least three 32-bit words. Its first word initially identifies the context. |
| Output address | Optional address of a writable 32-bit location. It is passed to the construction dependency and may later receive the candidate object address or zero. |

The descriptor and source-record addresses are checked for null. Their memory accessibility, alignment, and record sizes are not checked.

The output address is checked for null only before the function’s own final output store. The construction dependency receives it even when it is null.

### Relevant layouts

Offsets below are byte offsets. Every listed field occupies four bytes.

**Descriptor**

| Offset | Meaning established by use |
|---|---|
| 0 | Opaque construction parameter. |
| 4 | Optional callback address. |

**Source record**

| Offset | Meaning established by use |
|---|---|
| 0 | Context address initially; later copied again into the candidate header. |
| 4 | Header value copied into the candidate. |
| 8 | Header value copied into the candidate. |

**Context**

| Offset | Meaning established by use |
|---|---|
| 36 | Entry count used when requesting table growth and later selecting the insertion slot. |
| 40 | Current table address; replaced with the table dependency’s return value. |
| 100 | Opaque dependency parameter captured before construction. |

**Candidate object header**

| Offset | Value written by this function |
|---|---|
| 0 | Source record’s first word, reread after construction succeeds. |
| 4 | Source record’s second word. |
| 8 | Source record’s third word. |
| 12 | Descriptor address. |

The candidate must therefore support at least 16 writable bytes when construction reports success. Table entries are four-byte object addresses.

### Return value

The function returns a 64-bit composite value:

- Bits 0–31 contain the final 32-bit status value.
- Bits 32–63 contain an auxiliary word whose meaning depends on the path taken.

The status starts at zero. Invalid required inputs set it to 6. Dependencies and the optional callback can supply other status values.

The upper word is:

- The original auxiliary argument if execution never reaches the table-growth operation.
- The value read from context offset 40 immediately before the table-growth call if that operation is reached.

The upper word is therefore **not generally the candidate address or the returned table address**. No broader public error enumeration or return-value convention is established.

## Behavior

### 1. Validate required inputs and capture the context

The status is initialized to zero.

If both the descriptor address and source-record address are non-null, the first source word is read and retained as the context address.

Execution rejects any of these conditions:

- Null descriptor address.
- Null source-record address.
- Zero context address in the first source word.

Rejection sets status to 6 and returns with the auxiliary argument in the upper return word.

On this path, no dependency is invoked and the function performs no output-address store. A null source-record address is not dereferenced. If both required addresses are non-null, the first source word is read even if it contains zero.

Validation does not check the descriptor’s contents, the remaining source words, the context fields, or the output location.

### 2. Invoke construction

Using the retained context address, the function reads the word at context offset 100 and retains that word for later dependency calls.

It invokes F-3ce007b692cc with these six arguments:

1. The captured context word from offset 100.
2. The descriptor’s first word.
3. The address of the function’s writable 32-bit status storage.
4. The original output address.
5. The original auxiliary value.
6. The original source-record address.

The return value is treated as a candidate object address.

This function decides whether construction succeeded by checking the status storage, not by checking the candidate address.

If status is nonzero after construction:

- No candidate header is initialized.
- No callback is invoked.
- No table-growth operation is attempted.
- No cleanup call is made by this function.
- If the output address is non-null, the candidate value returned by the dependency is written there, even if that value is nonzero.
- The upper return word remains the original auxiliary argument.

Consequently, construction failure does not imply that the final output is zero. The dependency’s failure-return convention is unknown.

### 3. Initialize the candidate header

If construction leaves status zero, the function reads all three source-record words and copies them into the first three candidate words. It then stores the descriptor address in the fourth candidate word.

The visible read sequence obtains source words at offsets 4 and 8 before rereading offset 0. The header stores proceed in increasing offset order: 0, 4, 8, then 12.

These values are read **after** the construction dependency returns. Changes that dependency makes to the source record can therefore affect the header.

The retained context address is not refreshed. If construction changes the source record’s first word, subsequent context operations still target the original context, while the candidate header receives the changed first word.

There is no candidate-null check before these stores. A zero candidate address with zero construction status does not produce a handled failure; it leads to attempted memory writes through that address.

### 4. Invoke the optional callback

After header initialization, the descriptor’s second word is examined as a callback address.

- If it is zero, initialization proceeds without a callback.
- If it is nonzero, the callback is invoked with the candidate address and original auxiliary value.

The callback’s 32-bit return value replaces the status.

A zero callback result permits table insertion. A nonzero result skips table insertion and enters cleanup.

The callback runs after all four header stores but before this function updates the context’s table or count. It receives no explicit status-storage address.

No preservation guarantee is established for the candidate, descriptor, source record, context, globals, or memory reachable by the callback.

### 5. Request a table with one additional entry

If the callback is absent or returns zero, the function reads the original context’s table-address field at offset 40. That value becomes the upper word of the eventual return value.

It then invokes F-6dd003795db7 with:

1. The context word previously captured from offset 100.
2. An element width of 4.
3. A count read from context offset 36.
4. A count read from context offset 36 and increased by one.
5. The table-address value read before this call.
6. The address of the writable status storage.

The visible expression contains two count reads. Their relative argument-evaluation order is not established. It is unsafe to infer a coherent snapshot if other activity can change the count during argument preparation.

The dependency’s returned value is stored into context offset 40 **before** the function checks status.

Thus, a table-growth failure does not preserve the old table field through this function’s own logic. The dependency may return the old address, a replacement address, zero, or another value according to its unknown contract; this function stores that result unconditionally.

### 6. Append the candidate

If status is zero after table growth:

1. The context count at offset 36 is read again.
2. That newly read count plus one is stored back to offset 36.
3. The candidate address is stored into the returned table at byte offset four times the newly read count.

The count update precedes the table-entry store.

The insertion index is therefore not necessarily the count supplied to the table dependency. A dependency-induced change to the count can change the eventual insertion slot.

The function does not check that the returned table address is nonzero, that the insertion slot is within allocated storage, or that the count arithmetic is representable.

After the entry store, execution proceeds to the optional final output store.

### 7. Clean up a rejected candidate

Cleanup occurs when construction reported success but either:

- The optional callback returned nonzero; or
- The table-growth dependency left status nonzero.

F-ee708a7c6ea9 is called with the candidate address. Its return value is not used.

After that call returns, the function sets its candidate value to zero. If the output address is non-null, the final output store therefore writes zero.

There is no explicit rollback of:

- Candidate header changes.
- Callback effects.
- Dependency effects.
- A table-address replacement already written at context offset 40.

On table-growth failure, this function does not perform its own count increment or insertion store.

### 8. Publish the output

All non-validation paths reach an optional final store:

- If the output address is non-null, write the current candidate value there.
- If it is null, omit this store.

This store follows insertion or cleanup. Because the same output address was passed to construction earlier, a dependency’s earlier write may be overwritten.

## Side effects

Directly visible writes include:

- Four candidate header words after zero construction status.
- Context offset 40 after every table-growth call, irrespective of status.
- Context offset 36 and one table entry after zero table-growth status.
- The optional output location on every path that passes initial validation.

Dependencies and the callback may cause additional effects. Their memory ownership, allocation, destruction, and global-state behavior are not established here.

The visible structure suggests that an accepted candidate becomes registered in a context-owned table and that rejected initialized candidates require cleanup. That is inferred intent, not proof of ownership transfer or deallocation semantics.

Aliasing is not checked. If the output location overlaps the context, candidate, descriptor, source record, or table, the final output store can alter those structures. Header stores and dependency effects can likewise affect later reads when inputs overlap. Conventional behavior therefore requires suitable valid, noninterfering storage unless a particular alias arrangement is independently justified.

No synchronization or atomic transaction is visible. In particular, the count is incremented before the candidate entry is written.

Arithmetic operates on 32-bit fields and addresses. Count increment and four-byte slot scaling have no explicit overflow checks. The supplied representation does not independently settle signed-overflow semantics. A normal-operation precondition is that the relevant counts, increments, address calculations, and allocated extent are valid without overflow.

## Dependencies

| Opaque ID | Visible role | Unresolved contract |
|---|---|---|
| F-3ce007b692cc | Produces a candidate address and can update status; receives descriptor data, the output address, auxiliary value, and source record. | Allocation mechanism, failure pointer, memory ownership, output writes, and mutations of reachable state. |
| F-6dd003795db7 | Produces a table address from element width, old and requested counts, an existing table value, and writable status. | Resizing semantics, allocation extent, failure preservation, movement or release of storage, and count/context mutations. |
| F-ee708a7c6ea9 | Called for a candidate rejected after construction success. | Whether it frees memory, runs destructors, changes globals, or affects other reachable state. |

The descriptor callback has no supplied opaque dependency ID. It is identified only by its interface: candidate address and auxiliary word in, 32-bit status out.

The status address is exposed to both construction and table dependencies. No contract establishes whether either retains that address or makes it reachable to later calls. Accordingly, this function performs no explicit status assignment during cleanup, but preservation of the prior status across cleanup cannot be guaranteed independently of dependency behavior.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless stated otherwise, assume valid nonoverlapping storage, representable arithmetic, returning dependencies, and no unmentioned mutations.

1. **Required argument missing.**  
   With a null descriptor or source-record address, the low return word is 6, the upper word is the auxiliary argument, no dependency is called, and the output location is untouched.

2. **Context absent.**  
   With non-null required addresses but a zero first source word, the result is the same validation failure. The source record’s first word is read.

3. **Construction reports failure with a nonzero candidate value.**  
   If construction supplies status 9 and returns address P, the function does not initialize P or clean it up. A non-null output receives P, not zero. The upper return word remains the auxiliary argument.

4. **No callback and successful insertion.**  
   Suppose construction returns writable candidate P with zero status, the callback field is zero, and the context count is 3. If table growth returns T with zero status and leaves the count unchanged, the count becomes 4 and P is written at T plus 12 bytes. A non-null output receives P. The upper return word is the table value read before growth, not necessarily T.

5. **Callback rejects the candidate.**  
   If the callback returns 12, table growth is skipped, cleanup receives P, and the final output is zero. The upper return word remains the auxiliary argument. The low word is 12 provided cleanup does not indirectly alter exposed status storage.

6. **Table growth reports failure.**  
   If table growth returns Q and sets status 5, context offset 40 is first set to Q. Cleanup then receives P, and the final output is zero. This function performs no count increment or insertion store. The upper return word contains the pre-growth table value.

7. **Construction changes the source context word.**  
   If construction changes that word from C to D, the candidate header receives D. Table and count operations still use C.

8. **Table growth changes the count.**  
   If the count supplied during argument preparation is 3 but the dependency changes it to 7 before returning zero status, insertion uses index 7 and the function stores count 8. Correct storage capacity for that behavior is an external requirement.

## Uncertainties

- **documented_only:** Dependency allocation, ownership-transfer, and release contracts are unavailable.
- **documented_only:** No complete meaning for status values other than the visible invalid-input value 6 is established.
- **documented_only:** The semantic purpose of the upper return word is unknown; only its path-dependent origin is established.
- **documented_only:** Argument-evaluation ordering does not establish an order between the two count reads used for table growth.
- **documented_only:** Overflow behavior, invalid-access behavior, callback calling-convention details, and concurrency guarantees are not established.
- **Inferred intent:** Construction, optional initialization, registration, and rejection cleanup appear to form an object-creation workflow. These names do not add guarantees to the dependency interfaces.

## Evidence

The contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes the null checks, 32-bit field accesses, byte offsets, dependency arguments, status-controlled branches, four ordered header writes, unconditional table-field replacement after growth, count-before-entry write ordering, cleanup conditions, optional output store, and two-part return construction.

No runtime execution, tests, dependency implementations, or external ownership documentation were available. All acceptance examples are conditional consequences of the visible behavior, not runtime verification.
