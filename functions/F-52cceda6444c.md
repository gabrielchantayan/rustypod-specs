<!-- Trusted provenance: F-52cceda6444c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-52cceda6444c

## Purpose

F-52cceda6444c conditionally updates two externally managed objects from four candidate values and a context containing dimensions, object handles, and attachment values.

The update is gated by up to four ordered calls to F-1d1b465375c3. If every call returns zero, this function performs no object-update calls and does not write its four cached values. If any call returns nonzero, it calculates geometry and complementary 8-bit control values, updates the objects, performs two attachment operations, and stores the candidate values in the context.

The function returns the 32-bit value 1 on every explicit return path. This return value does not distinguish an update from a skipped update.

**Inferred intent:** the calculations resemble a fixed-point geometry and crossfade update. That interpretation does not establish the meaning of the external objects, the control setter, or the geometry operation.

## Interface

### Input

One argument identifies a writable context in a 32-bit address space. The observed layout is:

| Context byte offset | Observed interpretation |
|---|---|
| +0x18 | 32-bit value passed as the second argument of the second attachment operation |
| +0x1c | 32-bit value passed as the second argument of the first attachment operation |
| +0x20 | 32-bit handle for the first object |
| +0x24 | 32-bit handle for the second object |
| +0x28 | Signed 32-bit dimension, called **W** below |
| +0x2c | Signed 32-bit dimension, called **H** below |
| +0x3c | One-byte mode flag; zero and nonzero select opposite control-value assignments |
| +0x44 | Pointer to a record containing the first candidate at record offset +4 |
| +0x48 | Pointer to a record containing the second candidate at record offset +4 |
| +0x4c | Pointer to a record containing the third candidate at record offset +4 |
| +0x50 | Pointer to a record containing the fourth candidate at record offset +4 |
| +0x54 | Writable 32-bit cache for the first candidate |
| +0x58 | Writable 32-bit cache for the second candidate |
| +0x5c | Writable 32-bit cache for the third candidate |
| +0x60 | Writable 32-bit cache for the fourth candidate |

The four candidate values are called **A**, **B**, **C**, and **D**, respectively. These names identify contract values, not source identifiers.

### Output

The explicit return value is the 32-bit word 1.

There is no explicit error return, null check, dimension validation, or dependency-failure handling.

### Preconditions and ownership

The context, the four pointer fields, and the four referenced record values must be accessible for their observed reads. The cache fields must be writable when an update occurs. Dependency-specific validity requirements for object handles and attachment values are unknown.

All four referenced records must be readable even if the first gate call would trigger an update: all four candidates are loaded before any gate call.

The function contains no explicit allocation or release operation. It passes object handles, attachment values, and addresses of temporary candidate storage to dependencies. Whether those dependencies retain pointers, change ownership, allocate, release, or mutate other state is not established.

## Behavior

### 1. Snapshot the candidate values

The function reads the candidate records in this order:

1. Read the pointer at context +0x44, then the 32-bit value at that record’s +4.
2. Read the pointer at context +0x48, then the 32-bit value at that record’s +4.
3. Read the pointer at context +0x4c, then the 32-bit value at that record’s +4.
4. Read the pointer at context +0x50, then the 32-bit value at that record’s +4.

These values initially become A, B, C, and D in temporary storage. Subsequent operations do not explicitly reload them from their records.

### 2. Evaluate the update gate

F-1d1b465375c3 receives two addresses on each invocation: a temporary candidate address and its corresponding cache-field address.

Calls occur in candidate order:

| Call | Candidate storage | Cache address |
|---|---|---|
| First | A | Context +0x54 |
| Second | B | Context +0x58 |
| Third | C | Context +0x5c |
| Fourth | D | Context +0x60 |

Evaluation stops at the first nonzero result. A nonzero result enters the update path. Four zero results skip that path.

The dependency is not established to be a pure comparison. Both addresses are passed without an established read-only contract. Consequently:

- A zero result does not prove that the call had no effects.
- An invoked call may change its candidate or cache storage.
- Later calculations and final cache writes use the candidate storage as it exists after the gate calls.
- Candidates whose gate calls are skipped still retain their initially loaded values, subject to any otherwise undocumented side effects.

If all four results are zero, the function returns 1 without its own cache stores or object-update calls.

### 3. Establish working dimensions

On the update path, H is read from context +0x2c and W from context +0x28.

The initial horizontal working extent is obtained by arithmetic-right-shifting signed A by 16 bits and clearing its lowest two bits. Thus, for ordinary nonnegative A, the integer portion is rounded downward to a multiple of four.

F-870c4fcf7295 is then called with:

- First argument: the product of that initial horizontal extent and H.
- Second argument: W.

Its 32-bit result supplies the initial vertical working extent. The dependency’s arithmetic meaning is unknown; it must not be replaced by division merely because the surrounding calculation resembles aspect-ratio scaling.

The horizontal extent is adjusted using signed comparisons:

- If four times W is smaller than the initial horizontal extent, use four times W.
- Otherwise, use the larger of the initial horizontal extent and W divided by four with truncation toward zero.

The vertical extent is adjusted similarly:

- If four times H is smaller than the helper’s result, use four times H.
- Otherwise, use the larger of the helper’s result and H divided by four with truncation toward zero.

For nonnegative W and H, and arithmetic within range, these operations constrain the respective extents between one quarter and four times their associated dimensions. That interpretation is not a separate validation rule: zero, negative, and overflowing inputs are not rejected.

The products passed to F-870c4fcf7295 can involve unsigned 32-bit arithmetic because the horizontal extent is held as an unsigned word. Such unsigned arithmetic wraps modulo 2³². Later comparisons interpret relevant extent words as signed 32-bit values.

### 4. Calculate placement and leading-edge clipping

The horizontal anchor is signed C plus 65,535, followed by an arithmetic right shift of 16 bits. The vertical anchor is calculated identically from D.

When the signed additions do not overflow, these expressions implement mathematical ceiling of signed 16.16 values, including negative values.

The function divides each adjusted extent by two using signed division with truncation toward zero.

For the horizontal axis:

- If the horizontal anchor is at least half the horizontal extent, destination X is the anchor minus that half extent, and the horizontal source offset is zero.
- Otherwise, destination X is zero.
- In the latter case, the clipped horizontal amount is half the extent minus the anchor.
- F-870c4fcf7295 is called with W multiplied by that clipped amount, and the current horizontal extent.
- Its result becomes the horizontal source offset.
- The clipped amount is subtracted from the horizontal destination extent.

For the vertical axis:

- If the vertical anchor is at least half the vertical extent, destination Y is the anchor minus that half extent, and the vertical source offset is zero.
- Otherwise, destination Y is zero.
- In the latter case, the clipped vertical amount is half the extent minus the anchor.
- F-870c4fcf7295 is called with that amount multiplied by H, and the current vertical extent.
- Its result becomes the vertical source offset.
- The clipped amount is subtracted from the vertical destination extent.

The horizontal clipping call, when needed, precedes the vertical clipping call.

Only leading-edge clipping is explicit here. There is no independent right-edge or bottom-edge clipping operation in this function. A sufficiently negative anchor can reduce an extent to zero or below; no explicit positivity check follows.

### 5. Assign complementary control values

A control value **T** is calculated from B by:

1. Adding 32,767 using unsigned 32-bit arithmetic.
2. Retaining the lowest 24 bits.
3. Shifting right by 16 bits.

T is therefore always between 0 and 255 inclusive. This is a masked, wrapping calculation, not saturation of arbitrary signed B into an 8-bit range.

The byte at context +0x3c determines the assignment:

| Mode byte | First-object value | Initial second-object value |
|---|---:|---:|
| Zero | T | 255 − T |
| Nonzero | 255 − T | T |

F-460b2abbbbbe is called first for the handle read from context +0x20, then for the handle read from context +0x24.

The second-object handle is loaded after the first setter returns. The function does not establish that the first setter preserves context fields.

### 6. Suppress or configure the second object

After both initial control-setting calls, the function rereads W from context +0x28.

It suppresses the second object if either of these signed comparisons holds:

- The vertical source offset is greater than or equal to the reread W.
- Otherwise, the horizontal source offset is greater than or equal to H reread from context +0x2c.

H is reread only if the first comparison permits continuing. The subtraction and overflow-flag expressions in the supplied material implement signed comparisons; the suppression decision should not be reduced to an unchecked test of a wrapped subtraction’s sign.

If suppression is selected, F-460b2abbbbbe is called again for the current second-object handle with value zero. This overrides the second object’s initial complementary value.

Otherwise, F-a05963f65233 receives nine arguments, in this order:

| Argument | Value |
|---|---|
| 1 | Current second-object handle |
| 2 | Destination X |
| 3 | Destination Y |
| 4 | Vertical source offset |
| 5 | Horizontal source offset |
| 6 | Remaining horizontal destination extent |
| 7 | Remaining vertical destination extent |
| 8 | Reread W minus horizontal source offset |
| 9 | Reread H minus vertical source offset |

The source-offset ordering and suppression comparisons are asymmetric: the fourth argument is the vertical offset, but the first bounds comparison checks that offset against W. This contract preserves the observed ordering rather than normalizing it to a presumed geometry API.

### 7. Perform attachments and commit caches

After either suppression or geometry configuration, the function calls F-1d891874186d twice:

1. Current first-object handle from +0x20, with the value at +0x1c.
2. Current second-object handle from +0x24, with the value at +0x18.

Only after both attachment calls return does the function explicitly store:

1. A at context +0x54.
2. B at context +0x58.
3. C at context +0x5c.
4. D at context +0x60.

It then returns 1.

## Side effects

The directly established effects are:

- Reads of the context and four referenced records.
- Passing writable addresses to the gate dependency.
- Conditional calls that set controls on two objects.
- Either a further zero-valued control call or a geometry call for the second object.
- Two attachment calls on every completed update path.
- Four ordered cache writes after the attachment calls.

This is not an atomic transaction. Object calls occur before cache commitment, and no rollback is visible. If a dependency fails to return normally, later calls and cache stores are not reached.

Dimensions and handles are not all snapshotted at entry. Geometry calculations use dimensions read near the start of the update path; suppression and final geometry arguments use later dimension reads. Handles are also read for later calls. If dependencies mutate these fields, one invocation can use different values at different stages.

No preservation guarantee for globals, context state, or externally managed objects can be attributed to an unknown dependency.

## Dependencies

| Opaque function ID | Established role at this call site | Unestablished behavior |
|---|---|---|
| F-1d1b465375c3 | Receives candidate and cache addresses; zero continues the gate sequence, nonzero triggers the update | Comparison semantics, mutations, retention, other effects |
| F-870c4fcf7295 | Maps two 32-bit arguments to a 32-bit result used in extent or source-offset calculations | Arithmetic operation, signedness contract, zero-divisor behavior, errors |
| F-460b2abbbbbe | Receives an object handle and a value in 0–255 | Meaning of the value, object mutation, ownership effects |
| F-a05963f65233 | Receives the second-object handle and eight geometry-related values | Argument semantics beyond observed construction, validation, rendering effects |
| F-1d891874186d | Receives an object handle and a context-supplied attachment value | Attachment meaning, ownership transfer, lifetime effects |

No return value from the three object-operation dependencies controls subsequent execution.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, dependencies return normally and do not mutate context fields or candidate storage.

### All gate results are zero

Given four readable candidate records and four zero gate results:

- All four gate calls occur in order.
- No F-870c4fcf7295, control, geometry, or attachment call occurs.
- The function itself performs no cache writes.
- The return value is 1.

Any gate-internal mutations remain outside that no-write statement.

### An early gate result triggers the update

If the first gate call returns nonzero:

- The other three gate calls do not occur.
- B, C, and D have nevertheless already been loaded.
- The update uses the current temporary candidates.
- Completed attachment calls precede all four explicit cache stores.

### Centered, unclipped configuration

Suppose W is 100, H is 80, A represents 100 in signed 16.16 form, C represents 50, and D represents 40. Suppose the initial F-870c4fcf7295 call returns 80.

Then the horizontal and vertical extents are 100 and 80. Both destination coordinates and both source offsets are zero. No clipping-related arithmetic-helper calls occur.

The suppression checks pass, and F-a05963f65233 receives, after the handle:

0, 0, 0, 0, 100, 80, 100, 80.

### Leading horizontal clipping

Use the same dimensions and initial extents, but let C represent 40 and D represent 40.

The horizontal clipped amount is 10. The horizontal clipping helper receives 1,000 and 100. If it returns 10, destination X is zero, the horizontal source offset is 10, and the remaining horizontal extent is 90.

The geometry arguments after the handle are:

0, 0, 0, 10, 90, 80, 90, 80.

### Control polarity and wraparound

If B is zero, T is zero. Zero mode assigns 0 to the first object and 255 to the second; nonzero mode reverses those values, before any suppression override.

If B is 0x01000000, T is also zero because the calculation discards bits above bit 23. An implementation that saturates this input to 255 would not match the observed calculation.

### Suppression preserves update completion

If the calculated vertical source offset equals the later W value, the second object receives a further control value of zero. No geometry call occurs, and the later H read for the second bounds comparison is unnecessary.

Both attachment calls and all four cache stores still occur if the dependencies return normally.

## Uncertainties

- **documented_only — dependency semantics:** names such as “control,” “attachment,” and “geometry” describe call-site roles, not verified external API behavior.
- **documented_only — arithmetic helper:** F-870c4fcf7295 is not established to implement division or any other particular operation.
- **documented_only — signed overflow:** several signed additions, products, shifts, and subtractions can exceed their ordinary signed ranges. The supplied representation does not independently establish a portable overflow contract. Numeric examples above avoid those cases.
- **documented_only — invalid inputs:** null pointers, inaccessible records, invalid handles, zero dimensions, and negative dimensions have no explicit local rejection path. Their ultimate effects depend on memory validity and dependency behavior.
- **documented_only — extra register artifact:** an unexplained secondary result appears in the supplied representation after a setter call. On the path that consumes the corresponding dimension in the geometry call, that value is replaced by an explicit H read. No additional setter-output contract is established.
- **documented_only — concurrency and aliasing:** no synchronization is visible. Aliased records, overlapping context fields, or dependency mutations may affect later reads. No race-free or snapshot-consistent behavior is promised.

## Evidence

The established contract comes from static inspection of the supplied function body: field offsets, 32-bit accesses, byte-flag access, ordered candidate loads, short-circuit gate calls, arithmetic expressions, external-call arguments, later dimension rereads, final cache stores, and the constant return.

Fixed-point interpretation, complementary blending, and geometry intent are inferences from those operations. Dependency implementations and runtime observations were not supplied. No runtime tests have been performed.
