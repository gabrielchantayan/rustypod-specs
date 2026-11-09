<!-- Trusted provenance: F-9a3a1fbd933f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9a3a1fbd933f

## Purpose

F-9a3a1fbd933f updates a caller-supplied, mutable object containing four groups of fixed-size records and several control fields.

Its established behavior consists of three ordered phases:

1. Compute a one-byte flag and select a reduced control value.
2. Transform four source fields in every visited record into four destination fields.
3. For two ordered group pairs, replace a record’s destination fields with those of the first qualifying record in its paired group.

The arithmetic transformations depend on opaque functions. Their names, units, mathematical meanings, side effects, and failure behavior are not established by the supplied evidence. Interpreting the records as geometry, the second argument as a scale, or the final phase as proximity-based alignment is **inferred intent**, not a required semantic interpretation.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

### Arguments and result

The function receives:

- **Object address:** a pointer to writable, integer-addressed storage containing the layout below.
- **Transform parameter:** an integer passed to both dependencies and used in the initial flag calculation.
- **Additive adjustment:** an integer added to three of the four transformed fields.

The function has no explicit return value. Its normal result is the modified object.

There is no visible null-pointer check, allocation-size check, record-count validation, or error return.

### Width model

The layout below uses the apparent 32-bit integer model of the supplied representation:

- Integer fields occupy four bytes.
- Records occupy 32 bytes.
- The flag occupies one byte.
- The rounding mask is a 32-bit mask, `0xffffffc0`.

The underlying ABI and its complete signed-arithmetic rules are not independently supplied. Consequently, byte offsets and 32-bit bit-pattern behavior are supported by the representation, while signed-overflow behavior remains a documented uncertainty.

### Object layout

The object begins with four consecutive group regions:

| Group | Header byte offset | First record byte offset | Record stride |
|---|---:|---:|---:|
| G0 | 0 | 4 | 32 bytes |
| G1 | 516 | 520 | 32 bytes |
| G2 | 1032 | 1036 | 32 bytes |
| G3 | 1548 | 1552 | 32 bytes |

Each header contains an integer record count. The spacing provides room for 16 records per group before the next region begins.

Each record has this layout:

| Record-relative byte offset | Contract name | Direct use |
|---|---|---|
| 0 | S0 | Transformation input and matching input |
| 4 | S1 | Transformation input |
| 8 | S2 | Transformation input |
| 12 | S3 | Transformation input |
| 16 | D0 | Adjusted, rounded transformation result |
| 20 | D1 | Unadjusted transformation result |
| 24 | D2 | Adjusted transformation result |
| 28 | D3 | Adjusted transformation result |

Control fields follow the four nominal group regions:

| Byte offset | Width | Contract name |
|---|---:|---|
| 2064 | 4 bytes | Flag input |
| 2068 | 4 bytes | Initial selection value |
| 2072 | 4 bytes | Selected value output |
| 2080 | 1 byte | Flag output |

The function does not directly write the bytes at offsets 2076–2079 or the bytes following the flag byte. This is not a guarantee that dependencies leave those locations unchanged.

For the conventional bounded layout, the object must provide at least 2081 accessible bytes, with suitable alignment for integer accesses. Counts above 16 require accesses beyond the corresponding nominal record region and can overlap other groups or control fields.

### Preconditions for bounded, defined operation

A caller seeking the conventional behavior must provide:

- A non-null object with valid readable and writable storage for all accessed fields.
- Group counts in the range 0–16 whenever those counts are read.
- Dependency calls that return normally.
- Arithmetic inputs for which all relevant signed operations have defined target behavior.
- Any additional preconditions imposed by the dependencies.

The implementation does not enforce these conditions.

## Behavior

### 1. Flag calculation

The function first compares the transform parameter with a global threshold whose value is not supplied.

If the transform parameter is less than that threshold, the function compares:

- Eight times the flag input; and
- 125 times the transform parameter.

The flag becomes zero when the first quantity is less than or equal to the second, and one otherwise.

If the transform parameter is greater than or equal to the threshold, the function instead calls F-870c4fcf7295 with:

1. The flag input shifted left by three bits.
2. The integer constant 125.

It compares the returned integer with the transform parameter. The flag becomes zero when the returned value is less than or equal to the transform parameter, and one otherwise.

The comparison boundary is inclusive in both branches: equality produces zero.

Only after the selected calculation completes does the function write the flag byte at offset 2080. The direct write affects one byte, not a four-byte integer field.

The two branches are not established as mathematically equivalent. In particular, treating F-870c4fcf7295 as division would require evidence not present here.

### 2. Selection reduction

After storing the flag, the function reads the initial selection value at offset 2068.

While its current selection value is positive, it calls F-74efb2674226 with:

1. The current selection value.
2. The transform parameter.

A result greater than 32 causes the current selection value to decrease by one. A result less than or equal to 32 ends the reduction immediately.

When the current selection value is zero or negative, no selection-related dependency call is made.

The function writes the final value to offset 2072. In the absence of interference from a dependency, a positive initial value can be reduced to zero but not below zero. An initially negative value is written unchanged.

The initial selection value is read once for this reduction. Changes to its stored field during dependency calls do not replace the current local selection value.

### 3. Record transformation

The function visits groups in this exact order:

**G0, G1, G2, G3.**

At the start of each group, it reads that group’s count. It then processes records in increasing address order, advancing 32 bytes per record.

The loop terminates when its count reaches zero, not when it becomes nonpositive. A negative count therefore does not mean “empty”; it can cause extensive invalid traversal or encounter arithmetic-limit behavior.

For each visited record, the following dependency calls and direct writes occur in order:

1. Transform S2 with F-74efb2674226, using the transform parameter as its second argument. Add the additive adjustment and store the result in D3.
2. Transform S3 in the same manner. Add the additive adjustment and store the result in D2.
3. Transform S0 in the same manner. Add the additive adjustment and store the result in D0.
4. Transform S1 in the same manner. Store the result directly in D1, without the additive adjustment.
5. Read D0 and replace it with the 32-bit result obtained by adding 32 and clearing its low six bits.

Thus, under ordinary 32-bit bit-pattern arithmetic, the final D0 operation maps to a multiple of 64. For nonnegative values without arithmetic-limit complications, it rounds to the nearest multiple of 64, with midpoint values moving upward. That informal rounding description must not replace the specified add-and-mask behavior for negative or boundary values.

The final D0 rounding reads the stored D0 after the S1 dependency call. If that call modifies D0 through an external alias, the rounded value is the value then present, not necessarily the earlier S0-derived value.

No direct writes to S0–S3 occur.

### 4. Ordered matching and replacement

After all four groups have been transformed, the function processes these pairs in order:

1. Source group G0 against candidate group G2.
2. Source group G1 against candidate group G3.

For each pair, the source group count is read once. Source records are visited in increasing address order.

For each source record, the candidate group count is read again, and candidates are scanned from the first record onward. Candidate traversal therefore restarts for every source record.

For each candidate:

1. The function reads the source S0 and candidate S0.
2. It subtracts candidate S0 from source S0.
3. If the difference is negative, it negates the difference.
4. It passes the resulting integer and the transform parameter to F-74efb2674226.
5. It accepts the candidate if the returned integer is strictly less than 64.

The predicate has no explicit lower bound. A negative dependency result qualifies.

On the first qualifying candidate, the function copies the candidate’s destination fields into the source record in this order:

1. D3
2. D2
3. D0
4. D1

It then stops searching candidates for that source record and advances to the next source record.

It does not choose the smallest distance or the lowest dependency result. The earliest qualifying candidate in storage order wins.

If no candidate qualifies, the source record’s destination fields receive no direct writes during this phase. If the candidate group count is zero, no matching dependency call is made for that source record.

G2 and G3 are candidate groups only in this phase. They receive no direct matching-phase writes under the conventional, nonoverlapping layout.

## Side effects

The function directly writes:

- The flag byte.
- The selected value output.
- D0–D3 in every transformed record.
- D0–D3 again in source records for which matching finds a candidate.

These writes are incremental. There is no rollback or transactional boundary. If a dependency does not return normally, earlier stores may already be visible.

The function directly reads:

- The global threshold.
- Relevant control fields.
- Group counts.
- Record source fields.
- D0 during rounding.
- Candidate destination fields during replacement.

Dependency calls are interleaved with reads and writes; source fields are not snapshotted as a complete record. Likewise, each replacement field is read and written in sequence rather than copied from an established four-field snapshot.

No direct allocation, deallocation, ownership transfer, or pointer retention is visible. The caller supplies the object. Whether either dependency allocates, releases memory, retains state, mutates globals, or reaches the object through aliases is unknown.

Claims that unmentioned bytes remain unchanged apply only to this function’s direct accesses under a valid, nonoverlapping layout. They do not constrain undocumented dependency effects.

## Dependencies

| Opaque ID | Visible inputs | Visible use of result | Established call circumstances |
|---|---|---|---|
| F-870c4fcf7295 | Shifted flag input; constant 125 | Signed comparison against the transform parameter | Once only when the transform parameter is at least the global threshold |
| F-74efb2674226 | Integer value; transform parameter | Selection predicate, transformed field value, or matching predicate | Selection reduction, four calls per transformed record, and candidate scanning |

No semantic equivalence between the dependencies is established.

For F-74efb2674226, the first argument has three distinct origins:

- A positive current selection value.
- One of a record’s four source fields.
- A magnitude-like difference between two S0 fields.

The last origin is not guaranteed to be a representable nonnegative integer: subtraction or negation can reach signed arithmetic limits.

If all counts remain valid and all dependencies return normally, transformation makes exactly four dependency calls per visited record. Matching makes one call per examined candidate, including the winning candidate, and none after the first winner for that source record.

## Acceptance examples

These are conditional acceptance scenarios derived from static behavior, not observed executions. Unless stated otherwise, they assume valid storage, representable arithmetic, and dependencies that do not mutate the object or relevant globals.

### Empty groups and nonpositive selection

Given four zero group counts and an initial selection value of −3:

- The selected value output becomes −3.
- No selection, transformation, or matching call to F-74efb2674226 occurs.
- The flag is still computed and written.
- F-870c4fcf7295 is called only if the threshold branch requires it.

### Selection boundary

Given an initial selection value of 5, suppose F-74efb2674226 returns:

- 33 for selection value 5.
- 32 for selection value 4.

The selected value output must be 4. The equality result of 32 stops the reduction; selection value 3 is not examined.

### Transformation ordering and adjustment

For one record, suppose the dependency results for S2, S3, S0, and S1 are respectively 10, 20, 70, and 40, and the additive adjustment is 5.

The transformation phase must produce:

- D3 = 15.
- D2 = 25.
- D0 = 64, from applying the add-32-and-mask operation to 75.
- D1 = 40.

The call order must follow S2, S3, S0, S1. D1 must not receive the additive adjustment.

### Rounding midpoint

If the adjusted S0 transformation result is 96, D0 becomes 128. If it is 95, D0 becomes 64.

These examples concern ordinary nonnegative values, not overflow boundaries.

### First candidate wins

Suppose a source record has three candidates whose matching dependency results, in storage order, are 64, 63, and 0.

- The first candidate is rejected.
- The second candidate supplies all four destination fields.
- The third candidate is not examined.

A result of exactly 64 does not qualify.

### Negative matching result

If the first examined candidate produces a matching dependency result of −1, that candidate qualifies because −1 is less than 64. No additional nonnegative-result check is performed.

### No candidate match

If every examined candidate returns at least 64, the source record retains its transformation-phase destination fields, subject to any dependency side effects.

### Flag equality

In the direct-arithmetic branch, equality between eight times the flag input and 125 times the transform parameter produces flag zero.

In the dependency branch, a dependency result equal to the transform parameter also produces flag zero.

## Uncertainties

The following are **documented_only**:

- The global threshold’s value, width confirmation, mutability, and intended role.
- The mathematical contracts and side effects of both dependencies.
- The complete ABI and machine-level behavior of signed overflow, signed left shift, and minimum-negative-value negation.
- Valid application-level ranges and units for all integer inputs.
- Whether malformed counts are possible in legitimate callers.
- Whether object aliases, concurrent mutation, or reentrant dependency calls can occur.
- Ownership behavior internal to either dependency.
- Any external error mechanism, including nonlocal control transfer or hardware faults.

The nominal capacity of 16 records per group follows from region spacing. No visible validation establishes that callers actually honor it.

The apparent use of a threshold to select a safer arithmetic path is inferred intent. It is not evidence that either branch is overflow-safe for all accepted inputs.

## Evidence

Established static evidence supports:

- A one-byte flag write followed by selection reduction.
- Four ordered transformation groups with 32-byte records.
- Four transformation calls per visited record, with adjustment omitted from D1.
- D0 normalization through addition of 32 and a 32-bit low-six-bit-clearing mask.
- Ordered matching from G0 to G2 and G1 to G3.
- Strict matching acceptance below 64, first-match precedence, and ordered replacement writes.
- No visible argument validation, explicit error result, allocation, release, or rollback.

The supplied representation is the sole evidence for these claims. Dependency semantics and runtime behavior have not been independently established. No runtime tests have been performed.
