<!-- Trusted provenance: F-010deee43e2f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-010deee43e2f

## Purpose

F-010deee43e2f prepares temporary configuration records and invokes two mandatory construction dependencies, followed by a third construction dependency when an object flag is set at a late read.

The mode argument controls both an early table selection and the second construction path. Exactly-one mode has a special effect on the table selection; all nonzero modes use the same second construction path.

**Established behavior:** the routine prepares stack-resident records, reads object and shared state at specific points, and forwards resource-request results into construction calls without checking them.

**Inferred intent:** the calls appear to create two related entities and optionally a third. Entity types, registration, resource ownership, and successful construction are not established.

The routine has no explicit return value or local success indication.

## Interface

### Arguments

| Position | Interpretation established at this boundary |
|---|---|
| First | A 32-bit address-like object reference. The routine reads a byte at offset `0x04` and a 32-bit field at offset `0x180`. |
| Second | A 32-bit integer mode. Zero and nonzero select different second construction paths. Exactly one additionally affects the early table selection. |
| Third | An opaque 32-bit value forwarded unchanged to every construction call. |
| Fourth | An opaque 32-bit value forwarded unchanged to every construction call. |

Only the least significant bit of the object byte at offset `0x04` participates in decisions. Other bits do not affect the shown behavior.

The object reference is not validated. Each object access requires the corresponding address to remain readable when that access occurs. In particular, dependencies invoked between the early and late flag reads are not established to preserve the object or its lifetime.

The representation establishes 32-bit word operations and one-byte flag reads. It does not establish a portable host-language calling convention, byte order, alignment requirements, or the semantic signedness of opaque words. The mode decisions require only equality comparisons with zero and one.

### Result and forwarding

F-8157a180d901 returns a 32-bit value that is forwarded directly as the first argument of the next construction call. No null check, status check, or transformation intervenes.

Each construction receives five arguments:

1. The result of the immediately preceding F-8157a180d901 call.
2. Zero.
3. The original third argument.
4. The original fourth argument.
5. The address of the selected temporary record.

Construction results, if any, are not inspected or retained.

### Temporary record layouts

The record names below describe roles, not recovered type names.

| Record | Prepared span | Use |
|---|---:|---|
| Auxiliary record | 48 bytes | Receives an initial transfer request and explicit overrides; subsequently becomes the source of an embedded-record transfer. |
| Second-base record | 104 bytes | Passed to the second construction in zero mode; otherwise used as the source of the extended record’s prefix. |
| First record | 104 bytes | Passed to the first construction. |
| Optional record | 104 bytes | Prepared unconditionally; passed to the optional construction only when the late flag test succeeds. |
| Extended record | 124 bytes | Contains a 104-byte prefix and five appended 32-bit fields; used in nonzero mode. |

These spans follow from stack-relative locations, transfer lengths, and explicit field writes. Displayed temporary-array boundaries do not independently define the semantic record boundaries.

The transfer dependency’s implementation is unavailable. A requested span does not, by itself, prove that the dependency reads or writes that span.

## Behavior

The sequence below applies when all invoked dependencies return normally. A dependency that does not return normally prevents the routine from reaching subsequent steps; no local recovery is shown.

### 1. Request auxiliary preparation and select a table word

The routine first invokes the transfer dependency with:

- Destination: the auxiliary record.
- Source: a shared template source argument.
- Requested length: `0x30`, or 48 bytes.

After that call returns, it reads the object byte at offset `0x04`.

The selected table index is:

- One if the byte’s least significant bit is set and the mode is exactly one.
- Zero otherwise.

It reads a 32-bit entry from the shared table using a four-byte stride and writes that word to auxiliary offset `0x2C`.

This selection is not equivalent to testing whether the mode is nonzero. With the flag set, mode one selects entry one, while mode two selects entry zero.

### 2. Apply conditional auxiliary overrides

Next, the routine reads the 32-bit object field at offset `0x180`.

If the value is zero, it makes no explicit writes to auxiliary offsets `0x0C`, `0x10`, or `0x14` at this stage.

If the value is nonzero, it performs the following operations in order:

1. Reads the first 32-bit word of a shared configuration source and writes it to auxiliary offset `0x0C`.
2. Reads the second 32-bit word of that source and writes it to auxiliary offset `0x10`.
3. Reads the object field at offset `0x180` again and writes that second result to auxiliary offset `0x14`.

The branch-test value is not reused as the stored value. A nonzero first read followed by a zero second read still causes the two shared-word writes and stores zero at offset `0x14`.

No atomic snapshot or synchronization mechanism is shown. If the branch is not taken, preservation of template bytes at these offsets is conditional on ordinary copy semantics for the initial transfer.

### 3. Request preparation of the remaining base records

The routine issues these transfer calls in order:

| Order | Destination | Source | Requested length |
|---|---|---|---:|
| 1 | Second-base record | A shared template source argument | `0x68` — 104 bytes |
| 2 | First record | Another shared template source argument | `0x68` — 104 bytes |
| 3 | First record, offset `0x1C` | Auxiliary record | `0x30` — 48 bytes |
| 4 | Optional record | Another shared template source argument | `0x68` — 104 bytes |

All four calls precede the first F-8157a180d901 invocation. The optional record is therefore prepared regardless of whether the optional construction will later occur.

Under ordinary byte-copy semantics, the third transfer replaces first-record bytes `0x1C` through `0x4B`, inclusive. Its field correspondence is:

| Auxiliary offset | First-record offset |
|---|---:|
| `0x0C` | `0x28` |
| `0x10` | `0x2C` |
| `0x14` | `0x30` |
| `0x2C` | `0x48` |

These byte-content conclusions are conditional on the transfer dependency honoring the requested copy operation. Supplying a source argument and length is established; actual source-byte reads and destination-byte copying are not independently established.

### 4. Invoke the first construction

The routine calls F-8157a180d901 with `0x220`, or 544.

It then invokes F-7f6becd152e6 using the common five-argument convention and the first record.

The 104-byte preparation span does not establish how many bytes the construction dependency reads or writes. Nor does the caller establish that earlier dependencies leave the prepared record unchanged.

### 5. Invoke the mode-dependent second construction

#### Zero mode

For mode zero, the routine:

1. Calls F-8157a180d901 with `0x220`.
2. Calls F-7f6becd152e6 with the returned value and the second-base record.

The first and second resource requests are separate calls. The first result is not intentionally reused.

#### Nonzero mode

For every nonzero mode, the routine prepares the extended record’s appended fields in this order:

| Order | Offset | Width | Value |
|---|---:|---:|---|
| 1 | `0x68` | 32 bits | `0x1E`, or 30 |
| 2 | `0x6C` | 32 bits | An opaque 32-bit value read from the extension-word global at this point |
| 3 | `0x70` | 32 bits | 2 |
| 4 | `0x74` | 32 bits | `0x19`, or 25 |
| 5 | `0x78` | 32 bits | 5 |

The extension-word global is not established to be immutable or address-valued. The routine reads its word value and stores that value. It does not dereference an address represented by that word.

The routine then requests a `0x68`-byte transfer from the second-base record to the start of the extended record.

Under ordinary byte-copy semantics, that request fills offsets `0x00` through `0x67` without replacing the appended fields. The external dependency’s unavailable implementation prevents an unconditional assertion that it leaves those fields or other state unchanged.

After the transfer returns, the routine writes a 32-bit zero at extended-record offset `0x30`.

It then reads an opaque 32-bit value from the nonzero-request global and passes that value to F-8157a180d901. This global read occurs **after** the prefix-transfer call and the explicit zero write. Its value is not established to be fixed, a valid size, or an address. The routine passes the read word itself; it does not dereference that word.

Finally, it invokes F-4a73b9be0f3a with the returned resource value and the extended record.

Consequently, the extension-word read and the resource-request read are distinct observations separated by an external call. There is no established preservation guarantee for either global.

### 6. Evaluate and optionally invoke the final construction

After the second construction dependency returns, the routine rereads the object byte at offset `0x04`.

If its least significant bit is clear, no final construction occurs.

If the bit is set, the routine:

1. Reads a 32-bit word at byte offset four within another shared source.
2. Writes that word to optional-record offset `0x64`.
3. Calls F-8157a180d901 with `0x230`, or 560.
4. Calls F-60fe53f7ab39 with the returned value and the optional record.

Offset `0x64` occupies bytes `0x64` through `0x67`, the last four bytes of the 104-byte prepared span.

The late flag read is independent of the early flag read. An early set bit does not guarantee the optional call; an early clear bit does not exclude it.

## Side effects

### Direct reads and writes

The routine directly writes temporary stack storage. No explicit write to the input object or shared configuration sources is shown.

Established direct reads include:

- The object flag byte, once after initial auxiliary preparation and once after the second construction.
- The object field at offset `0x180`, once for the branch and conditionally again for the stored value.
- The selected 32-bit table entry.
- Two shared configuration words when the object-field branch is taken.
- The extension-word global in nonzero mode, before the extended-prefix transfer.
- The nonzero-request global in nonzero mode, after that transfer.
- The shared word used for the optional record, only when the late flag test succeeds.
- Shared values used to supply the transfer-source and shared-source addresses.

The transfer calls are established effects: destination, source, and byte-count arguments are supplied in the stated order. Reads of shared template bytes, and writes of corresponding destination bytes, are conditional on the transfer dependency’s semantics. They are not established direct reads and writes of this routine.

Neither the direct reads nor the dependency calls form an atomic snapshot.

### Dependency effects, ownership, and lifetime

No dependency is established to preserve globals, object fields, or previously prepared records.

The routine performs no explicit release and retains no returned construction handle. F-8157a180d901 has a resource-request-like interface, but allocation, initialization, failure encoding, and ownership are unknown.

Construction dependencies receive addresses of stack-resident records. Whether they copy, mutate, or retain those records is unknown. Retention beyond the invocation would require a lifetime arrangement not established here; the routine supplies no persistent copy of its own.

### Errors

There is no visible input validation, failure branch, rollback, or error-reporting mechanism.

A failure-shaped value returned normally by F-8157a180d901 is forwarded unchanged. The routine does not independently determine whether construction succeeded.

Invalid memory accesses and dependencies that terminate abnormally have no locally specified recovery behavior.

## Dependencies

| Opaque function ID | Established call-site role | Unresolved contract |
|---|---|---|
| F-8157a180d901 | Accepts one 32-bit request value and returns a 32-bit value forwarded into construction. | Resource meaning, valid request domain, failure encoding, ownership, and state effects. |
| F-7f6becd152e6 | Receives the first record and, in zero mode, the second-base record. | Entity semantics, record access extent, retention, mutation, and failure behavior. |
| F-4a73b9be0f3a | Receives the extended record in nonzero mode. | Field meanings, resource ownership, record lifetime, and failure behavior. |
| F-60fe53f7ab39 | Receives the optional record when the late flag test succeeds. | Entity semantics, ownership, record lifetime, and failure behavior. |

The external transfer dependency has no supplied opaque function ID. Its visible interface accepts a destination, source, and requested byte count. Ordinary copying is an interpretation of its usage, not an independently verified implementation contract.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. They assume valid required memory accesses and normally returning dependencies. Record-byte expectations additionally assume ordinary copy semantics and no intervening mutation of the relevant record.

### Zero mode, flag consistently clear

For mode zero and a clear least significant flag bit at both reads:

- Table entry zero is selected.
- F-8157a180d901 receives `0x220` twice.
- Both results are forwarded to separate F-7f6becd152e6 calls.
- Neither F-4a73b9be0f3a nor F-60fe53f7ab39 is invoked.
- Neither nonzero-mode global is read by that branch.

### Exactly-one mode, flag consistently set

For mode one with the flag bit set at both reads:

- Table entry one is selected.
- Construction calls occur in the order F-7f6becd152e6, F-4a73b9be0f3a, F-60fe53f7ab39.
- Resource-request arguments occur in the order `0x220`, the nonzero-request global’s value when read after the prefix transfer, and `0x230`.
- Extended offset `0x6C` receives the extension-word global’s value read before that transfer.

### Other nonzero modes

For mode two, or any other representable nonzero mode except one:

- Table entry zero is selected even when the early flag bit is set.
- The second construction still uses F-4a73b9be0f3a.
- The final flag read independently controls F-60fe53f7ab39.

A negative nonzero mode follows the same branch structure; no positive-range requirement is shown.

### Global mutation during the nonzero path

Suppose the extension-word global reads as `0x11111111` when its appended field is prepared. The routine writes that word to extended offset `0x6C`.

If the prefix-transfer dependency changes the nonzero-request global to `0x33333333` and that value remains until the subsequent read, F-8157a180d901 receives `0x33333333`.

The earlier value of the request global is irrelevant to this request. Changing the extension-word global later does not itself reread or rewrite offset `0x6C`; a dependency could nevertheless mutate the record directly.

### Flag changes between observations

An initially clear flag followed by a set late flag enables the optional construction. An initially set flag followed by a clear late flag suppresses it.

The table choice remains based on the early observation.

### Object-field changes between reads

A nonzero first read at object offset `0x180` followed by a zero second read causes both shared configuration words to be written and stores zero at auxiliary offset `0x14`.

A zero first read skips those shared-word reads and the second object-field read.

### Failure-shaped resource result

If F-8157a180d901 returns zero normally, the next construction receives zero as its first argument. That value alone causes no local fallback or early return.

## Uncertainties

The following remain **documented_only**:

- Shared template contents and semantic record types.
- Meanings and possible values of the extension-word and nonzero-request globals.
- Whether any opaque global word represents an address, size, handle, or another quantity.
- Exact transfer behavior, including actual memory access and unrelated side effects.
- Dependency preservation of globals, input-object state, and prepared records.
- Resource ownership, construction success criteria, and retained-record lifetimes.
- Required alignment, full object extent, and behavior on invalid addresses.

No variable-length record-size calculation is shown. The table index is bounded to zero or one, giving a byte displacement of zero or four. Transfer lengths and record offsets are fixed. The first resource request, the zero-mode second request, and the optional request are fixed numeric values; the nonzero-mode second request is a 32-bit global value read at its stated point.

These bounded offsets do not establish address validity or rule out address wraparound for an invalid base. No explicit overflow or bounds check is shown.

## Evidence

Established static evidence consists of the supplied routine’s access widths, stack-relative spans, constants, comparisons, branch conditions, explicit writes, argument forwarding, and dependency-call order.

In particular, the nonzero path contains two separate global-value reads: one for extended offset `0x6C` before the prefix-transfer call, and one for the resource request after that call. Neither read establishes immutability or an address-valued interpretation.

Transfer arguments establish requests, not the transfer dependency’s internal memory accesses. Copy-derived byte layouts and construction or allocation intent remain conditional interpretations.

No runtime tests have been performed.
