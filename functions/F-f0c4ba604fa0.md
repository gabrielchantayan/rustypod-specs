<!-- Trusted provenance: F-f0c4ba604fa0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f0c4ba604fa0

## Purpose

F-f0c4ba604fa0 looks up an encoded record through F-e81ed1edd673, checks two header bytes and a payload-length constraint, and decodes the accepted payload into caller-provided storage.

Each payload byte is multiplied by 23, with only the low eight bits written. Copy direction depends on the unsigned ordering of the destination and payload-source addresses. This supports ordinary overlapping source and destination ranges, subject to valid addressing.

The normal completion return is the 32-bit value zero. Rejection returns a value read from one of four external words. Their numeric values are unknown; they are not established to be distinct or nonzero.

The apparent intent is record extraction with either exact-length or maximum-length acceptance. That interpretation is inferred. The checks, transformations, access ordering, and writes described below are established by the supplied static representation.

## Interface

The function accepts six arguments in the following order.

| Argument | Representation | Meaning |
|---|---|---|
| Lookup token | Opaque 32-bit word | Passed unchanged as the second argument to F-e81ed1edd673. |
| Source handle | Pointer to a 32-bit word | Must be non-null and initially contain a nonzero word. After a nonzero lookup result, its contents are read again and used as the source base address. |
| Result location | Optional pointer to a 32-bit word | If non-null, receives the scaled payload length after payload copying finishes. |
| Destination | Unsigned 32-bit address value | Beginning of the output region. Zero is rejected at entry unless the length argument is zero. |
| Length argument | Unsigned 32-bit value | Required payload length in zero mode; maximum accepted payload length in nonzero mode. |
| Length mode | 32-bit integer | Zero requires exact length. Every nonzero value permits a payload no longer than the length argument. |

The return is a 32-bit word from one of five sources:

- Literal zero after successful processing.
- External entry-rejection word.
- External lookup-failure word.
- External header-rejection word.
- External length-rejection word.

These names distinguish return sources, not known numeric codes. A zero return alone does not establish success unless the external rejection words are independently known to be nonzero.

### Memory and ownership preconditions

Defined completion requires valid memory for every access reached on the selected path:

- A non-null source handle must permit a 32-bit read.
- The dependency must accept the handle and lookup token.
- Following a nonzero dependency result, the reread base and returned offset must identify the header bytes actually accessed.
- An accepted nonempty payload must have the required readable source range and writable destination range.
- A non-null result location must permit a 32-bit write on the success path.
- Address arithmetic, alignment, object lifetime, and access permissions must satisfy the execution environment.

The routine does not visibly check allocation extents or address validity. An invalid memory access has no specified conversion into a rejection return.

No direct allocation, release, or ownership transfer is established. F-e81ed1edd673 has an unknown ownership contract; this routine’s use of the handle does not establish that the dependency preserves or releases any particular allocation.

## Behavior

### Entry validation

Entry validation is short-circuited in this order:

1. A null source handle causes entry rejection without dereferencing the handle.
2. Otherwise, the handle word is read. Zero causes entry rejection.
3. With a nonzero handle word, processing continues only if the length argument is zero or the destination is nonzero.

Consequently:

- Destination zero and length zero pass the destination check.
- Destination zero and any positive unsigned length fail it.
- A nonzero destination passes that check for every length value.

This is only a zero-address check. It does not establish that the destination is writable.

Entry rejection returns the current entry-rejection word. The dependency is not called, and this routine performs no payload or result write.

### Lookup and source selection

After entry validation, F-e81ed1edd673 is called once with the source handle first and lookup token second.

A zero dependency result causes lookup-failure rejection. No record-header access, payload copying, or direct result write follows.

Every nonzero result is used as an offset. There is no explicit rejection of a negative signed result or of an offset outside an allocation.

After a nonzero result, the source handle is read again. This post-call value supplies the source base. The initial handle value is not retained as the authoritative base: dependency mutation of the handle can redirect subsequent reads.

For this contract, the **record address** is the post-call source base plus the returned offset. The offset is used in byte addressing. Portable behavior for signed address overflow or invalid pointer formation is not established; valid address arithmetic remains a precondition.

### Record layout and header checks

The directly inspected record fields are:

| Record offset | Width | Interpretation |
|---|---:|---|
| 0–3 | Four bytes | Encoded, big-endian total length |
| 4–13 | Ten bytes | Not inspected as header fields |
| 14 | One byte | Must decode to zero |
| 15 | One byte | Must decode to zero; checked before offset 14 |
| 16–19 | Four bytes | Not inspected as header fields |
| 20 onward | Computed payload length | Encoded payload |

For an encoded byte \(b\), define its decoded value as

\[
D(b)=(23b)\bmod 256.
\]

Offset 15 is read and checked first. If its transformed byte is nonzero, the function returns the header-rejection word without reading offset 14 for its header check.

Only when offset 15 passes is offset 14 checked. A nonzero transformed value there also causes header rejection.

Because 23 is relatively prime to 256, \(D(b)=0\) exactly when the raw byte is zero. Both header bytes therefore must be raw zero for processing to reach the length computation. This arithmetic equivalence does not remove the short-circuit read order.

After those checks pass, the four length bytes are individually decoded and interpreted as

\[
T=D(b_0)2^{24}+D(b_1)2^{16}+D(b_2)2^8+D(b_3).
\]

Here \(T\) is a 32-bit quantity. The expression establishes the significance of each byte, but not a guaranteed order among the four individual length-byte loads.

The payload length is

\[
L=(T-20)\bmod 2^{32}.
\]

There is no separate requirement that \(T\geq20\). A smaller decoded total produces a large unsigned payload length rather than an immediate header rejection.

### Length acceptance

The function accepts the computed payload length exactly when:

\[
L\leq N
\quad\text{and}\quad
(M\ne0\ \text{or}\ N\leq L),
\]

where \(N\) is the unsigned length argument and \(M\) is the mode.

Thus:

- **Zero mode:** accept only \(L=N\).
- **Nonzero mode:** accept any \(L\) from zero through \(N\), inclusive.

A failed comparison returns the length-rejection word. This routine performs no payload or result-location write on that path.

In nonzero mode, the payload-copy operation writes only the first \(L\) destination bytes. It does not pad, clear, terminate, or otherwise write the spare capacity from \(L\) through \(N-1\). This is not a guarantee that those bytes remain unchanged across the whole invocation: the dependency may modify them, and an aliased result-location write may overwrite them afterward.

### Scaled result and payload copying

Following length acceptance, the payload source begins at record offset 20.

Before any payload byte is copied, the function reads an external scaling word \(S\) and computes a 32-bit result word corresponding to

\[
R=(L\times S)\bmod 2^{32}.
\]

The scale read and product computation occur even when the result location is null or the payload length is zero. The scale’s value, signed interpretation, and intended unit are unknown. The representation supports the low 32-bit product; it does not establish a portable signed-overflow contract.

Copy direction is selected by an unsigned address comparison:

- Destination below the payload source: process indices from zero upward.
- Destination equal to or above the payload source: process indices from \(L-1\) downward.

For each processed index, the corresponding source byte is read, transformed by \(D\), and written to the corresponding destination byte. The source read precedes its destination write.

For \(L=0\), there are no payload reads or writes. Equal source and destination addresses select descending processing and permit in-place decoding.

For ordinary valid ranges without address wraparound, the direction preserves unread payload bytes when source and destination overlap. No separate overlap rejection or temporary payload buffer is visible.

After copying finishes:

1. A non-null result location receives the previously computed 32-bit word \(R\).
2. A null result location receives no write.
3. The function returns literal zero.

The optional result write occurs after every payload write. Consequently, final destination contents are not necessarily just the decoded payload when the result location overlaps the destination.

## Side effects

### Direct accesses and writes

Depending on the selected path, the function directly performs:

- One initial source-handle read.
- One dependency call.
- One post-call source-handle read after a nonzero lookup result.
- Ordered header checks at offsets 15 and then 14.
- Four encoded length-byte reads after successful header checks.
- One scaling-word read after length acceptance.
- Exactly \(L\) payload source-byte reads and \(L\) payload destination-byte writes on a completed success path.
- One optional 32-bit result write after copying.
- A read of the applicable external rejection word when returning a rejection value.

The function does not directly write the source handle as a handle operation. Aliased destination or result writes may nevertheless change its storage.

### Dependency effects and rejection

F-e81ed1edd673 may modify the handle, destination, result storage, other memory, or global state. No preservation guarantee is supplied.

Entry rejection precedes the dependency call. Lookup, header, and length rejection occur after it. Therefore, those later rejections do not imply an invocation without side effects, even though this routine performs no payload or result write on them.

External rejection words and the scaling word must not be assumed to retain their pre-call values across the dependency call.

### Aliasing and partial execution

No general non-aliasing restriction is checked.

- A result location overlapping the destination can overwrite decoded payload bytes or spare capacity after copying.
- A destination overlapping the source handle can change the handle word during copying. The function uses the already obtained source base rather than rereading the handle for each byte.
- Destination writes may overwrite header storage or other writable data identified by the supplied addresses.
- A destination write to scaling-word storage does not change the already computed result.
- An aliased result write may also change source, handle, or external-word storage after payload processing.

There is no established rollback. If an access fails after earlier writes, neither restoration nor a clean rejection return is specified.

## Dependencies

### F-e81ed1edd673

**Inputs:** source handle, followed by the unchanged 32-bit lookup token.

**Consumed output:** a 32-bit integer-like result. Zero selects lookup failure. Every nonzero value is used as a record offset without a separate sign or range check.

**Call ordering:** called once after entry validation; a nonzero result is followed by a fresh handle read.

**Unknown contract:** lookup rules, admissible tokens, offset guarantees, handle mutation, allocation lifetime, ownership effects, and other memory or global-state changes.

### External storage

Four words provide rejection returns, and one word provides the result scale.

Their numeric values, initialization, mutability, lifetime, and addresses are not established by the supplied evidence. In particular, no distinctness or nonzero guarantee exists for the rejection words.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless an example describes an earlier rejection, assume the dependency returns a usable nonzero offset and every reached access and address calculation is valid. Assume no concurrent mutation.

For examples that reach length acceptance, the post-lookup record bytes at offsets 15 and 14 must both be raw zero. Any example asserting final decoded destination bytes additionally requires a null result location or a valid result location whose four-byte write is disjoint from the bytes asserted.

1. **Null handle.**  
   A null source handle returns the entry-rejection word. The handle is not dereferenced, and F-e81ed1edd673 is not called.

2. **Zero destination boundary.**  
   With a readable, nonzero handle word, destination zero and length one cause entry rejection. Destination zero and length zero proceed to lookup; they do not themselves establish later acceptance.

3. **Lookup failure.**  
   If the dependency returns zero, return the lookup-failure word without header inspection or a direct result write. Dependency side effects remain possible.

4. **Short-circuited header failure.**  
   If post-lookup record offset 15 contains raw byte `01`, return the header-rejection word. Offset 14 is not read for its header check, regardless of its value.

5. **Exact-length decoding.**  
   Require post-lookup offsets 15 and 14 to be zero. Encoded length bytes `00 00 00 01` decode to total 23, so \(L=3\). With zero mode and length three, the length check passes. Encoded payload `01 02 FF` produces final destination bytes `17 2E E9`, provided the result location is null or its write is disjoint from those three bytes.

6. **Exact mismatch and spare capacity.**  
   Require the same successful header checks and three-byte payload as example 5, but use length five. Zero mode returns the length-rejection word. Nonzero mode accepts and copies three bytes. For the accepting case, a null result location or a result write disjoint from the five-byte destination range ensures that destination bytes three and four retain their values from immediately before copying. To claim preservation relative to function entry, the dependency must also preserve those bytes.

7. **Empty payload with destination zero.**  
   Require both checked header bytes to be zero. Encoded length bytes `00 00 00 0C` decode to total 20 because \(12\times23\bmod256=20\). With destination zero and length zero, either mode can pass. No payload access occurs. The scaling word is still read, and a valid non-null result location receives zero.

8. **Unsigned subtraction boundary.**  
   Require both checked header bytes to be zero. Encoded length bytes `00 00 00 65` decode to total 19 because \(101\times23\bmod256=19\). The computed payload length is 4,294,967,295. Length 100 rejects it. Length 4,294,967,295 passes the length comparison even in zero mode, but completed copying additionally requires valid addressing for the entire range. No safe completion is implied merely by acceptance of the comparison.

9. **Backward overlap.**  
   Require successful header and length checks for a three-byte payload. Placing the destination one byte above the payload source selects descending processing: source index two is consumed before index one, then index zero. With valid ordinary overlapping ranges, writes do not destroy lower-index source bytes before they are read.

10. **Result aliasing output.**  
    Require successful header and length checks and a valid result location overlapping the destination. Payload copying completes first. The scaled-length word is then written over the overlapping bytes. Final destination contents reflect that later write, whose byte order is not established here.

## Uncertainties

The following remain **documented_only**:

- Numeric rejection values, including whether any equal zero or each other.
- The scaling word’s value, intended unit, and signed interpretation.
- The dependency’s lookup, ownership, mutation, and lifetime guarantees.
- Formal source-language types and portable signed-overflow semantics.
- Behavior for invalid addresses, alignment faults, address wraparound, and concurrent mutation.
- Byte order of the optional 32-bit result write.
- Wider record-format requirements, including allocation size, checksums, and meanings of header bytes not inspected here.

Overlap-safe extraction and an encoded total-record-length field are inferred intent. Neither interpretation supplies missing validation or dependency guarantees.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. The prior draft and reviewer feedback are not treated as independent evidence.

Established behavior includes short-circuited entry validation, dependency argument order, the post-call handle reread, header-check order, byte transformation, 32-bit unsigned payload-length subtraction and comparisons, pre-copy scale computation, direction-dependent decoding, post-copy result writing, and storage-backed rejection returns.

The examples’ encoded length bytes follow the same byte transformation as the implementation: `0C` decodes to 20, and `65` decodes to 19.

No runtime execution or tests have been performed. All acceptance examples are conditional consequences of static evidence, not observed results.
