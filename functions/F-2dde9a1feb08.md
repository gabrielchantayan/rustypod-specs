<!-- Trusted provenance: F-2dde9a1feb08; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2dde9a1feb08

## Purpose

F-2dde9a1feb08 performs two dependent lookups and, if both succeed, writes two signed differences derived from four fields in the resulting record.

The established behavior is:

- Read a 32-bit value through the first argument.
- Pass that value to F-b2834716058d.
- If its result is nonzero, pass that result and the third argument to F-69facf68ac4d.
- If the second result is also nonzero, interpret it as the location of a record, calculate two differences from signed 16-bit fields, and write those differences to the fourth and fifth arguments.
- Return a 32-bit status: zero for either lookup returning zero, or one after completing both output writes.

The field arrangement is consistent with calculating two dimensions from coordinate pairs. That interpretation is inferred intent, not an established semantic requirement. The contract does not assign names such as width, height, minimum, or maximum to the fields.

## Interface

The function has five arguments and a 32-bit return value.

| Position | Contract role | Representation and use |
|---|---|---|
| First | Input-value location | A pointer from which one 32-bit value is read before the first dependency call. |
| Second | Unused argument | A 32-bit value. The visible function does not inspect, forward, or otherwise use it. |
| Third | Lookup selector | A 32-bit value passed unchanged as the second argument to F-69facf68ac4d, if that call is reached. Its interpretation belongs to that dependency. |
| Fourth | First output location | A pointer receiving a signed 32-bit difference if both lookups return nonzero. |
| Fifth | Second output location | A pointer receiving a signed 32-bit difference after the first output write, if execution reaches it. |

The return value is a status indicator, not either calculated difference. On ordinary completion, its only possible values are zero and one.

The supplied representation establishes 32-bit words and integer results, and 16-bit signed record fields. It does not independently establish a complete platform ABI, byte order, or alignment policy.

### Memory-access preconditions

The first argument must designate storage valid for a 32-bit read whenever the function is invoked. There is no visible check before that read.

The fourth and fifth arguments need to designate valid writable 32-bit storage only when execution reaches their respective writes. A lookup failure bypasses both output writes.

A nonzero result from F-69facf68ac4d must designate storage from which the required 16-bit fields can be read. Nonzero alone does not establish that the location is valid.

These are access preconditions, not validations performed by the function. Misaligned, inaccessible, stale, or otherwise invalid locations have no defined recovery behavior in the supplied evidence.

## Behavior

### Initial read and first lookup

The function reads one 32-bit word through the first argument and passes that word as the sole argument to F-b2834716058d.

It passes the loaded value, not the address supplied as the first argument. No subsequent direct reread of that input location is visible.

If F-b2834716058d returns zero:

- F-69facf68ac4d is not called.
- No record fields are read by this function.
- Neither output location is written by this function.
- The function returns zero.

These statements do not imply that F-b2834716058d itself leaves memory or global state unchanged.

### Second lookup

If the first dependency returns nonzero, the function calls F-69facf68ac4d with:

1. The nonzero result from F-b2834716058d.
2. The third argument, unchanged as a 32-bit value.

If F-69facf68ac4d returns zero, the function returns zero without performing either output write or any of the record-field reads described below.

There is no retry, alternate lookup, or fallback path in the visible behavior.

### Record layout

A nonzero result from F-69facf68ac4d is treated as a record base. Four signed 16-bit fields participate in the calculations:

| Byte offset from record base | Interpretation within this contract |
|---|---|
| 0x10, decimal 16 | Subtrahend for the second difference |
| 0x12, decimal 18 | Subtrahend for the first difference |
| 0x14, decimal 20 | Minuend for the second difference |
| 0x16, decimal 22 | Minuend for the first difference |

Each field occupies two bytes. The highest accessed field therefore extends through byte offset 23.

The visible function does not inspect any other record fields. This layout does not establish a complete record size or the meaning of surrounding bytes.

### Calculations and output order

After the second lookup returns nonzero, the function performs the following observable stages:

1. Read the signed 16-bit fields at offsets 0x16 and 0x12.
2. Sign-extend both values to signed 32-bit integers.
3. Subtract the value at offset 0x12 from the value at offset 0x16.
4. Write the resulting signed 32-bit value through the fourth argument.
5. Read the signed 16-bit fields at offsets 0x14 and 0x10.
6. Sign-extend both values to signed 32-bit integers.
7. Subtract the value at offset 0x10 from the value at offset 0x14.
8. Write the resulting signed 32-bit value through the fifth argument.
9. Return one.

The supplied expression structure does not establish which of the two operand fields is read first within either subtraction. It does establish the first output assignment before evaluation of the second output assignment. In particular, the second pair of fields must not be treated as necessarily captured before the first output write.

Each signed 16-bit input lies between −32,768 and 32,767. Each mathematical difference therefore lies between −65,535 and 65,535, inclusive. Every possible difference fits in a signed 32-bit integer.

There is no visible unsigned reinterpretation, absolute value, clamping, sorting, or rejection of negative results. Equal paired fields produce zero, and zero-valued differences still constitute successful completion.

### Aliasing

The visible function does not check that its input, outputs, and returned record are disjoint.

If the fourth and fifth arguments designate the same valid 32-bit location, the second write overwrites the first. On normal completion, that location contains the second difference.

If the first output overlaps a field used by the second calculation, its write can change the bytes that the second calculation subsequently reads. Consequently, the second result is not unconditionally the difference between two values captured before either output was written.

If an output overlaps the first argument’s storage, the output may replace that storage after the initial input value has already been consumed. The visible function does not reread the first argument afterward.

These are consequences of the visible access order, not a guarantee that every overlapping arrangement is valid under the original language’s object-access rules or the execution environment.

## Side effects

The function directly performs:

- One 32-bit input read.
- Up to two dependency calls, in the stated order.
- On the successful path, four signed 16-bit record-field reads and two 32-bit output writes.

There are no visible direct accesses to global variables, allocation operations, release operations, logging operations, or I/O operations. The dependencies may have such effects; their implementations are not supplied.

The function does not visibly claim ownership of the input location, output locations, lookup results, or record. It does not explicitly release any of them. Neither observation establishes ownership or lifetime rules for the dependencies.

The two output updates are not atomic as a pair. If execution fails during the second calculation or second write, the first output may already have changed. No rollback or output restoration is visible.

Lookup failure leaves the outputs untouched **by this function**. It does not guarantee that their contents remain unchanged across the complete call: a dependency could modify aliased memory or shared state.

## Dependencies

### F-b2834716058d

Called exactly once on every execution that completes the initial input read and reaches the call.

Its argument is the 32-bit value loaded through the first argument. A zero result stops processing. A nonzero result is forwarded to F-69facf68ac4d.

Its argument interpretation, ownership behavior, memory effects, failure mechanism, and result lifetime are documented_only unknowns.

### F-69facf68ac4d

Called exactly once if F-b2834716058d returns nonzero.

It receives the first dependency’s result and the original third argument. A zero result stops processing. A nonzero result is used as the record location for the field accesses.

Its lookup semantics, record ownership, lifetime guarantees, mutation behavior, and effects on other memory are documented_only unknowns.

Neither dependency is assumed to preserve globals, preserve caller-provided storage, or retain or release memory.

## Acceptance examples

These examples are conditional behavioral requirements, not reports of executed tests. They assume valid accesses, normal dependency returns, and no unmentioned concurrent mutation.

### First lookup fails

Given a readable first argument and F-b2834716058d returning zero:

- The function returns zero.
- F-69facf68ac4d is not called.
- The function performs no output writes.

If the dependency itself leaves both output locations unchanged, their original contents remain intact.

### Second lookup fails

Given F-b2834716058d returning a nonzero value and F-69facf68ac4d returning zero:

- The second dependency receives that nonzero value and the unchanged third argument.
- The function returns zero.
- The function reads none of the four record fields and writes neither output.

### Positive differences

Given a valid, stable record with:

- Offset 0x16 containing 30.
- Offset 0x12 containing 10.
- Offset 0x14 containing 19.
- Offset 0x10 containing 4.

And given nonoverlapping valid output locations, the function writes 20 through the fourth argument, then 15 through the fifth, and returns one.

### Signed and boundary behavior

Given nonoverlapping outputs and stable fields:

- A first minuend of −32,768 and first subtrahend of 32,767 produce −65,535.
- A second minuend of 32,767 and second subtrahend of −32,768 produce 65,535.

The function writes those signed 32-bit results without saturation and returns one.

### Identical output locations

Given record values producing a first difference of 20 and a second difference of 15, and both output arguments designating the same valid location outside the record:

- The location is first written with 20.
- It is then overwritten with 15.
- The function returns one.

### Output overlapping later record reads

If the first output write changes bytes belonging to the fields at offsets 0x10 or 0x14, the second difference uses the field values present when those later reads occur. A fixed numeric expectation requires the actual overlap, byte order, and validity rules; those are not established here.

### Unused second argument

Changing only the second argument does not change any explicit operation performed by the visible function. It is not forwarded to either dependency. This is not a broader claim about dependencies observing ambient process state.

## Uncertainties

The following remain documented_only:

- The semantic meanings of the initial input value, lookup selector, intermediate result, and record fields.
- Whether the two calculated differences represent geometric dimensions.
- The complete record layout and size.
- Byte order and alignment requirements.
- Dependency side effects, ownership rules, result lifetimes, and error behavior beyond their zero/nonzero results.
- Whether overlapping storage is supported by the original execution environment.
- Behavior under concurrent mutation.
- Any handling of access faults, dependency exceptions, nonreturning calls, or other abnormal execution.

No validation, recovery, synchronization, or ownership guarantee should be inferred from the absence of visible mechanisms.

## Evidence

**Established static evidence:** The supplied function representation shows the initial 32-bit read, conditional dependency calls, zero/nonzero branching, four signed 16-bit field accesses at the stated byte offsets, signed 32-bit subtractions, ordered output assignments, unused second argument, and zero-or-one normal return.

**Inferred intent:** The offset pairing resembles calculating two extents from endpoints. This interpretation is not required for acceptance.

**Unknown behavior:** Dependency implementations and the surrounding ABI, memory model, and ownership conventions are unavailable.

No runtime tests have been performed. All acceptance examples are conditional deductions from the supplied static evidence.
