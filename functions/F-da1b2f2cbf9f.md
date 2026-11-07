<!-- Trusted provenance: F-da1b2f2cbf9f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-da1b2f2cbf9f

## Purpose

F-da1b2f2cbf9f selects between a bulk-reset path and an iterative processing path. It writes a 32-bit word through a caller-supplied output address and returns a separate 64-bit value composed from two 32-bit words.

The bulk path requires two nonzero dependency results and a nonzero state field. It calls F-afc46dab55b9, resets several fields, writes a reloaded header address to the output, and returns.

The iterative path calls F-cb011f129d08 until that dependency returns exactly `1`. Each nonterminating result causes a cursor-word capture, a call to F-a030ebc6bb58, a call to F-c4c63a93a1fb, and an output write.

**Inferred intent:** the operations resemble clearing a container or processing a range of elements. The supplied evidence does not establish container, iterator, removal, or destruction semantics. This contract therefore describes accesses, values, and call ordering without assigning those meanings to dependencies.

## Interface

### Arguments

| Argument | Established representation | Visible use |
|---|---|---|
| Output address | Address of a writable 32-bit word | Receives an initial header address and subsequent path-dependent writes. |
| State address | 32-bit address-bearing value | Identifies storage accessed at byte offsets `+16` and `+20`; also passed to dependencies. |
| Cursor address | Address of a 32-bit word | Passed to dependencies and directly read during each iterative body execution. |
| Boundary argument | Opaque 32-bit value | Passed to F-cb011f129d08. Not directly dereferenced by this function. |

Stored address-bearing values and directly accessed fields are 32 bits. This does not establish the host address width or permit treating the boundary argument as a particular pointer type.

No visible validation establishes that addresses are nonnull, aligned, accessible, nonoverlapping, or associated with valid objects.

### Visible storage layout

The **header address** is the 32-bit value read from state offset `+16`.

| Storage | Byte offset | Width | Visible role |
|---|---:|---:|---|
| State | `+16` | 32 bits | Supplies the header address through repeated reads. |
| State | `+20` | 32 bits | Tested for nonzero when deciding the bulk path; cleared on that path. |
| Header | `+4` | 32 bits | Supplies an argument to F-afc46dab55b9; subsequently cleared on the bulk path. |
| Header | `+8` | 32 bits | Supplies the initial comparison temporary; subsequently assigned a header address on the bulk path. |
| Header | `+12` | 32 bits | Assigned a header address on the bulk path. |

The total object sizes and meanings of other fields are unknown. Calling state offset `+20` a count, or the header fields links, would be inferred intent rather than an established layout contract.

### Output and return

The output word is written before the first dependency call.

The return is a 64-bit bit pattern with:

- A specified 32-bit word in bits 63–32.
- A specified 32-bit word in bits 31–0.

This composition does not specify memory byte order or an aggregate calling convention. There is no separately established success or error code.

Two mutable temporary words determine the iterative return:

- The **result word**, initialized to the state address.
- The **saved-address word**, initialized to the output address and overwritten during each iterative body.

Both words’ addresses are supplied to F-c4c63a93a1fb. Their values at return time—not necessarily their values immediately after the last such call—determine the iterative return.

## Behavior

### Initialization and first condition

Before invoking any dependency, the function performs these accesses in order:

1. Reads the header address from state offset `+16`.
2. Writes that address to the output word.
3. Reads state offset `+16` again.
4. Reads the second header’s `+8` field into a 32-bit comparison temporary.

The second header lookup is independent of the first. An overlapping output write can therefore affect the later lookup. No dependency call separates these accesses.

The function initializes the saved-address word to the output address and the result word to the state address. Another temporary initially receives the cursor address; if the first condition succeeds, that temporary is replaced with a freshly read header address before being passed by address.

F-cb011f129d08 is first called with:

- The cursor address.
- The address of the comparison temporary containing the header’s `+8` value.

The result is tested for **nonzero**. A zero result immediately selects the iterative path. A nonzero result permits evaluation of the remaining bulk conditions.

The comparison temporary is supplied by address. Its preservation, mutation, or retention by the dependency is not established.

### Remaining bulk conditions

Following a nonzero first result, the function:

1. Reloads the header address from state offset `+16`.
2. Places it in a temporary word.
3. Calls F-cb011f129d08 with the original boundary argument and that temporary’s address.
4. Tests the returned value for nonzero.
5. Only if that result is nonzero, reads state offset `+20`.

The bulk path is selected only if both dependency results and the tested state field are nonzero.

Consequences of this short-circuit behavior:

- A zero first result skips the second condition call and the decision-time read of state offset `+20`.
- A zero second result skips that field read.
- Any nonzero 32-bit pattern in the state field satisfies its condition, including `0xFFFFFFFF`.
- The header read before the second condition occurs after the first dependency call; it need not equal the entry header.

If either dependency result is zero, or the tested state field is zero, execution enters the iterative path.

### Bulk path

The bulk path performs the following ordered operations:

1. Reloads the header address from state offset `+16` and reads that header’s `+4` field.
2. Calls F-afc46dab55b9 with the state address and the field value just read.
3. Assigns a header address read from state offset `+16` to offset `+8` of a header selected through state offset `+16`.
4. Reloads the header through state offset `+16` and clears its `+4` field.
5. Assigns a header address read from state offset `+16` to offset `+12` of a header selected through state offset `+16`.
6. Clears state offset `+20`.
7. Reloads the header address from state offset `+16` into a captured return word.
8. Writes that captured word to the output.
9. Returns the captured word in the high 32 bits and the saved-address word in the low 32 bits.

Each reset obtains its header through the state rather than reusing the header that supplied the dependency argument. F-afc46dab55b9 can therefore affect which storage receives the resets.

For the assignments to header offsets `+8` and `+12`, the destination header and stored address are both obtained through state offset `+16`. The supplied representation does not establish the relative evaluation order of those operand reads.

With stable, nonoverlapping storage and no relevant dependency mutations, both fields become self-references, header offset `+4` becomes zero, and state offset `+20` becomes zero. With overlap or intervening changes, those simple postconditions need not hold for one fixed header object.

The final high return word is the address captured before the final output write. It is not a further reload after that write.

Neither F-a030ebc6bb58 nor F-c4c63a93a1fb is called on this path.

### Iterative path

Each iteration begins by calling F-cb011f129d08 with the original cursor address and original boundary argument.

The termination predicate is **equality to `1`**, unlike the nonzero bulk predicates:

- Exactly `1` terminates iteration.
- Every other 32-bit result executes the body, including zero, `2`, and `0xFFFFFFFF`.

For each body execution, the function:

1. Reads the 32-bit word at the cursor address and captures its value.
2. Calls F-a030ebc6bb58 with the cursor address.
3. Overwrites the saved-address word with the captured value.
4. Calls F-c4c63a93a1fb with the result word’s address, the original state address, and the saved-address word’s address.
5. Reads the result word and writes that value to the output.
6. Begins the next condition call.

The capture precedes F-a030ebc6bb58. A cursor change made by that dependency does not replace the captured value used to initialize the saved-address word for the subsequent call.

The result word is initialized once, not once per body. Its existing value is supplied to later calls unless changed through accessible storage. The saved-address word, by contrast, is explicitly overwritten from the current capture on every body execution.

F-c4c63a93a1fb can modify either temporary through its arguments. It may also retain either address; no non-retention guarantee is supplied. Both temporary words remain relevant during subsequent condition calls and until return.

### Iterative return and intervening mutations

When a condition call returns exactly `1`, the function returns:

- High 32 bits: the result word’s value at return time.
- Low 32 bits: the saved-address word’s value at return time.

There is no explicit output write after that terminating call.

The terminating F-cb011f129d08 call occurs after the final body’s output write. If F-c4c63a93a1fb retained temporary addresses and the terminating dependency can access them, that dependency could modify either temporary before return. Consequently:

- The returned pair need not equal the values immediately after the final F-c4c63a93a1fb call.
- The high return word need not equal the last value explicitly written to the output.
- Equality with those earlier values requires intervening dependencies not to modify the relevant storage.

If no body executes, the initialized result and saved-address words provide the return, provided dependencies have not modified their storage indirectly. Under that condition, the high word is the state address and the low word is the output address. The function’s only explicit output write is then the initial header-address write.

## Side effects

### Direct memory effects

The function explicitly writes:

- The output word during initialization.
- The output word after every iterative F-c4c63a93a1fb call.
- On the bulk path, header offset `+8`, header offset `+4`, header offset `+12`, state offset `+20`, and finally the output word, in that order.

These destinations need not represent independent storage when arguments or objects overlap.

The function also writes local temporary storage, including storage whose address is exposed to dependencies. Later dependency-mediated changes to those temporaries can affect output or return values.

### Ownership and dependency effects

No explicit allocation or release operation is visible in this function. Calls to F-afc46dab55b9 and F-c4c63a93a1fb do not establish deallocation, destruction, detachment, or ownership transfer.

Dependencies may affect reachable memory, globals, allocation state, or retained addresses. No preservation guarantee is established.

Temporary-address retention is relevant while the function is active, particularly across the final iterative condition call. This function establishes no valid post-return lifetime for those temporary addresses.

### Errors and termination

No explicit error handling, rollback, or failure result is visible.

Invalid accesses, dependency nontermination, exceptions or other nonlocal control transfers, and dependency-specific errors have no established outcome here. If iterative condition calls never return exactly `1`, no visible iteration bound forces termination.

There is no explicit iterative count decrement or range arithmetic. Address accesses use the stated byte offsets without visible checks. Validity of wrapped or inaccessible addresses is not established.

## Dependencies

| Opaque function ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-cb011f129d08 | Performs the first bulk condition, optionally the second, and every iterative condition. Bulk results use nonzero; iterative termination uses exactly `1`. | Comparison semantics, normalization, argument mutation, retained-address access, global effects. |
| F-afc46dab55b9 | Called only on the bulk path with the state address and a freshly read header `+4` value. Reset writes follow if it returns normally. | Traversal, release behavior, ownership effects, preservation of state and header storage. |
| F-a030ebc6bb58 | Called once per iterative body, after the cursor-word capture and before the saved-address overwrite. | Cursor advancement, element access, effects on other reachable storage. |
| F-c4c63a93a1fb | Receives addresses of both return-related temporary words and the original state address once per body. | Whether it writes or retains either address, removal semantics, ownership effects, state mutations. |

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless specified otherwise, assume valid, nonoverlapping storage and no dependency effects beyond those stated.

### Immediate iterative termination

Given output address `0x1000`, state address `0x2000`, header address `0x3000`, a zero first bulk-condition result, and a first iterative-condition result of `1`:

- The output is written with `0x3000`.
- The second bulk-condition call is skipped.
- No iterative body executes.
- The packed return is `0x0000200000001000`.

### Bulk reset with stable storage

Given two nonzero bulk-condition results, state offset `+20` equal to `3`, and no dependency changes to relevant storage:

- F-afc46dab55b9 receives the state address and current header `+4` value.
- Header offsets `+8` and `+12` become the header address.
- Header offset `+4` and state offset `+20` become zero.
- The final output is the header address.
- The return’s high word is that address; its low word is the original output address.

Replacing `3` with `0xFFFFFFFF` selects the same path.

### Zero state field after matching conditions

If both bulk-condition results are nonzero but state offset `+20` is zero, no bulk dependency or explicit reset occurs. Execution proceeds to a new F-cb011f129d08 call for the iterative condition.

### One body with unchanged return temporaries afterward

Suppose the first bulk condition returns zero, then iterative conditions return `2` and `1`. The cursor word contains `0x5000` when captured.

F-c4c63a93a1fb receives a saved-address word initialized to `0x5000`, even if F-a030ebc6bb58 changes the cursor. If F-c4c63a93a1fb leaves the result word as `0x6000` and the saved-address word as `0x7000`:

- The next explicit output write stores `0x6000`.
- **If the terminating condition call does not modify either temporary**, the return is `0x0000600000007000`.

### Terminating dependency changes retained temporaries

Use the preceding example, but suppose F-c4c63a93a1fb retains both temporary addresses. The terminating F-cb011f129d08 call writes `0x8000` to the result word and `0x9000` to the saved-address word before returning `1`.

Then:

- The body’s output write remains `0x6000`, assuming no dependency changes the output itself.
- No further explicit output write occurs.
- The packed return is `0x0000800000009000`.

This distinguishes return-time values from values immediately following the last body call.

### Header replacement during the bulk dependency

If F-afc46dab55b9 replaces the state’s header address with another valid header address, subsequent reset accesses use fresh state lookups. They need not modify the original header. The final output and high return word use the address captured after the reset sequence.

## Uncertainties

The following are **documented_only**:

- The actual data structure and field meanings.
- Cursor and boundary semantics.
- Dependency memory effects, retention behavior, and global preservation.
- Allocation, release, and ownership rules.
- Any semantic meaning of the packed return beyond its word composition.
- Supported aliasing, alignment, address-space, and concurrency conditions.
- Operand-read ordering within individual expressions where the supplied representation does not establish it.
- Behavior on invalid memory, nonlocal control transfer, or dependency failure.

Retention is not asserted to occur. Its possibility prevents an unconditional claim that iterative return words equal the values immediately after the final F-c4c63a93a1fb call.

## Evidence

Established behavior derives from the supplied static representation: 32-bit accesses and values, byte offsets, dependency arguments, short-circuit conditions, ordered explicit writes, and 64-bit concatenated returns.

The iterative return reads the two temporary words after the terminating condition call. Their addresses were previously exposed to F-c4c63a93a1fb if a body executed; no supplied dependency contract excludes retention or subsequent mutation.

Container clearing, cursor advancement, and element removal remain inferred intent. Dependency implementations and external ownership contracts were not supplied.

No runtime tests have been performed. Acceptance examples are conditional static deductions, not execution evidence.
