<!-- Trusted provenance: F-9e4ac145d6aa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9e4ac145d6aa

## Purpose

F-9e4ac145d6aa fills a five-field integer record using an optional source reference. The record contains a numeric representation of that reference, a result value, a classification indicator, and two cleared fields.

The result has three cases:

- An absent source produces value zero and indicator −1.
- A present source leads to an indirect lookup. A retrieved value of −1 produces value zero and indicator −1.
- Any other retrieved value is preserved, with indicator zero.

The indirect lookup uses an integer stored at the source to determine another read location. It is not a fixed-offset lookup relative to the source reference itself.

These field descriptions identify observable roles only. The evidence does not establish an application-level record type, a general error-reporting interface, or the meaning of the two cleared fields.

## Interface

### Inputs and output channel

The function accepts two references:

| Input | Requirement |
|---|---|
| Source | May be null. When nonnull, must support an integer read and the subsequent address derivation described below. |
| Destination | Must identify writable storage for five consecutive signed-integer fields, including when the source is null. |

There is no returned value. The destination record is the output.

The destination’s previous contents are not needed for a valid, disjoint invocation. Overlapping storage can make those previous contents relevant because destination initialization precedes the source-derived reads.

### Record layout

Fields are numbered by increasing storage position:

| Field | Meaning | Completed contents |
|---|---|---|
| 0 | Numeric source representation | The source reference converted to the destination integer type |
| 1 | Result value | Retrieved value, except that an absent source or retrieved −1 yields zero |
| 2 | Classification indicator | −1 for an absent source or retrieved −1; otherwise zero |
| 3 | Cleared field | Zero |
| 4 | Cleared field | Zero |

All five fields and both source-derived reads use the same signed-integer type in the supplied representation.

Its exact bit width and storage size are not established. If its storage size is \(S\) bytes, the field offsets are \(0\), \(S\), \(2S\), \(3S\), and \(4S\), and the required destination extent is \(5S\) bytes. A 20-byte record is conditional on four-byte integers; it is not an unconditional requirement supported by this evidence.

Pointer width, alignment, byte order, and signed-integer representation are also unresolved. Conversion of the source reference to an integer is not established as lossless. Field 0 must not be treated as a guaranteed recoverable reference or unique identity.

### Preconditions for normal completion

The destination must remain valid and writable for every field assignment.

A nonnull source additionally requires:

- A valid signed-integer read through the source reference.
- Addition of 4 within the supported signed arithmetic range.
- A supported conversion of that sum into a reference.
- A valid signed-integer read at the converted location.

The conversion used to populate field 0 must also be supported by the execution environment.

The function supplies no capacity, alignment, lifetime, address-validity, arithmetic-range, or overlap checks. Overlapping arrangements are covered only when the environment permits all resulting accesses.

### Ownership and lifetime

No allocation, release, retention, reference-count change, or explicit ownership transfer is present.

Storing a numeric source representation does not establish that the referenced storage remains alive. Neither result classification establishes that any storage has been released. Callers must arrange the lifetimes necessary for the actual reads and writes; any broader ownership policy is unknown.

## Behavior

### Result selection

Source absence is determined from the source reference, independently of its integer conversion and independently of any integer stored at its location.

A null source requires no source-derived read. Its completed value and indicator are zero and −1.

For a nonnull source, the first read supplies a signed integer used as a numeric address base. The constant 4 is added to that integer before conversion to the reference used for the second read. The second read supplies the candidate value.

Only candidate −1 receives special treatment:

| Candidate | Completed value | Completed indicator |
|---|---:|---:|
| −1 | 0 | −1 |
| Any other representable signed integer | Candidate unchanged | 0 |

Consequently, zero is an ordinary successful candidate. Negative candidates other than −1 are also preserved. There is no visible range clamp, rejection of negative values generally, or interpretation of the first-read integer as a sentinel.

A zero integer stored at a nonnull source does not select the null-source case.

### Address derivation and arithmetic boundaries

The integer obtained from the first read, not the source reference’s own address, controls the second read location.

The addition is signed-integer arithmetic performed before conversion to a reference. It is not pointer advancement by four integer elements. Under an environment where integer address values map directly to byte addresses, the operation would describe a four-byte displacement from the encoded base. That interpretation remains conditional on the target’s conversion and addressing rules.

The evidence supplies neither checked addition nor a guaranteed wider intermediate. An out-of-range sum has no guaranteed result under this contract. In particular, acceptance must not assume modular wraparound, truncation to pointer width, or a usable low-order address.

### Observable access order

Destination initialization precedes either source-derived read. The source representation is stored first, then field 3 is cleared, then field 4 is cleared. The value and indicator fields have not yet been assigned by this invocation at that point.

The remaining access relationships depend on the selected case:

- With a null source, the value is set to zero before the indicator is set to −1.
- With a nonnull source, both dependent reads occur before the candidate is stored in the value field.
- An ordinary candidate is followed by an indicator write of zero.
- Candidate −1 is first stored in the value field, then replaced there by zero, after which the indicator becomes −1.

Thus the candidate −1 case includes an intermediate destination value of −1 that is absent from the completed record. The null-source and candidate −1 cases have identical completed value/indicator pairs but different read and write histories.

These are access relationships established by the supplied representation. They are not a promise about individual machine instructions, volatile accesses, memory barriers, atomic publication, or safe concurrent observation.

### Overlap and dependence on prior contents

No disjointness check is present. Where overlap is valid, the initial destination writes can alter the data subsequently read through the source or through the derived reference.

Two consequences matter:

1. Initialization can change the first-read integer and therefore redirect the second read.
2. Initialization can change the candidate storage before the second read reaches it.

For example, if the first read remains unaffected and the resulting candidate location is field 3, the candidate is zero because that field has already been cleared.

Both dependent reads finish before the first value-field assignment. Classification uses the retrieved candidate; later destination writes do not trigger another lookup. If such a write also changes overlapping source storage, the classification does not follow that later change.

For valid disjoint storage without external mutation, the destination’s old contents do not affect the result. That independence is not established for every overlapping arrangement.

### Invalid inputs and abnormal completion

Indicator −1 is not a general memory-access failure result. It is assigned for an absent source or a successfully retrieved candidate equal to −1.

An invalid destination can prevent completion during initial population. With a nonnull source, an invalid first read, unsupported arithmetic or conversion, or invalid second read can prevent completion after the initial destination assignments.

There is no explicit rollback, retry, exception translation, or alternate error return. The access order describes which operations precede a problematic operation, but does not guarantee that partial writes remain recoverable or observable afterward. Fault delivery and the effects of invalid operations depend on the execution environment.

## Side effects

Normal completion assigns all five destination fields.

| Case | Destination assignments | Source-derived reads |
|---|---:|---:|
| Null source | 5 | 0 |
| Nonnull source, ordinary candidate | 5 | 2 |
| Nonnull source, candidate −1 | 6 | 2 |

The extra assignment in the last case replaces the value field’s intermediate −1 with zero.

The two source-derived reads each cover one signed integer. Their locations can coincide or overlap if the supplied data and environmental access rules permit it. No destination read is separately required, although a source-derived read may reach destination storage under overlap.

There is no explicit write through either readable reference. Destination writes can nevertheless modify the same underlying storage.

No function call, explicit named-global access, allocation, release, I/O operation, callback, or synchronization operation appears in the supplied fragment. This does not establish that the referenced memory is private, nonglobal, unshared, or free of external side effects associated with its memory mapping.

## Dependencies

No callee appears. There are no dependency function IDs to enumerate and no callee ownership or global-preservation guarantees to assume.

The contract depends on environmental definitions of:

- Signed-integer width, range, representation, storage size, and addition.
- Pointer width and both directions of integer/reference conversion.
- The relationship between numeric address values and readable storage.
- Alignment, lifetime, aliasing, and access permissions.
- Concurrent access and visibility.
- Faults and other consequences of invalid operations.

The constant 4 alone does not establish integer width, pointer width, record size, or byte-addressed pointer interpretation.

## Acceptance examples

These examples are conditional static consequences, not executed tests. Except where overlap is expressly described, assume valid, suitably aligned, disjoint storage, supported conversions, defined arithmetic, and no external mutation.

### Absent source

For a null source:

- The completed value is zero.
- The completed indicator is −1.
- Fields 3 and 4 are zero.
- Field 0 contains the environment’s integer conversion of the null reference.
- Neither source-derived read occurs.

Acceptance must not independently require field 0 to be numerically zero without evidence about the target’s conversion rules.

### Ordinary values and exact sentinel matching

Suppose the first read and address derivation select valid candidate storage.

| Retrieved candidate | Required value | Required indicator |
|---|---:|---:|
| 37 | 37 | 0 |
| 0 | 0 | 0 |
| −2 | −2 | 0 |
| −1 | 0 | −1 |

Fields 3 and 4 must be zero in every case. Field 0 must contain the source conversion.

For candidate −1, the expressed access history includes storing −1 in the value field before replacing it with zero. Observing that intermediate state requires a valid observation mechanism not supplied by this contract.

### Nonnull source containing zero

Suppose the source reference is nonnull but its stored integer is zero. If adding 4 and converting the result selects readable storage containing 12, the completed value is 12 and the indicator is zero.

Selecting null-source behavior merely because the first-read integer is zero would violate the established behavior. If the derived location is unusable, no completed result is guaranteed.

### Encoded address versus source-relative storage

Suppose the first-read integer encodes a base unrelated to the source’s location. The second read must use the location obtained by adding 4 to that integer and converting the result.

A different value positioned four integer elements after the source does not control the result merely because of that relative position.

### Candidate storage overlapping a cleared field

Suppose valid overlap makes field 3 the second read location, while initial destination writes leave the first-read integer suitable for deriving that location.

The candidate is zero, regardless of field 3’s previous contents. The completed value and indicator are both zero. Using the field’s pre-initialization value would not match the established access order.

### Numeric source representations that collide

If the environment permits two distinct source references to convert to the same integer, field 0 need not distinguish them. In particular, a nonnull reference remains nonnull for control selection even if its conversion matches the conversion of a null reference.

This is a conditional limitation, not evidence that such a collision occurs on the target.

### Invalid derived access or out-of-range addition

A readable first source does not by itself guarantee completion. An inaccessible derived location need not produce indicator −1. An addition outside the signed arithmetic range must not be accepted on the assumption that it wraps to a readable location.

## Uncertainties

- **documented_only — Exact layout:** Integer and pointer widths, integer storage size, alignment, byte order, and signed representation remain unresolved.
- **documented_only — Conversion semantics:** Null encoding, truncation, conversion collisions, supported numeric addresses, and reference round-trip guarantees are unknown.
- **documented_only — Address interpretation:** Addition of 4 is established; a four-byte address displacement is conditional on the environment.
- **documented_only — Arithmetic failure:** No usable result is established for an out-of-range sum.
- **documented_only — Application semantics:** The record’s application type and the broader meanings of its indicator and cleared fields are unknown.
- **documented_only — Ownership policy:** The fragment establishes no transfer, retention, release, or lifetime extension.
- **documented_only — Concurrency:** Atomicity, coherent publication, and safe observation of intermediate writes are not established.
- **documented_only — Abnormal execution:** Fault handling, recovery, and survival of earlier writes after an invalid operation remain environment-dependent.

## Evidence

**Established static evidence:** The supplied fragment shows five consecutive integer destination fields; initial source-representation and clearing writes; a source-nullness distinction; two dependent integer reads for a nonnull source; integer addition by 4 before address conversion; exact classification of candidate −1; and replacement of the intermediate sentinel value before the final indicator assignment.

**Inferred intent:** “Value,” “classification indicator,” and “numeric source representation” are descriptive roles. They do not establish a domain-specific object model or a general success/failure protocol.

**Unresolved:** Target representation, conversion behavior, ownership beyond the visible operations, concurrency guarantees, and abnormal-execution semantics require evidence not supplied here.

No runtime execution or tests have been performed. The acceptance examples are conditional requirements derived from static evidence, not observed outcomes.
