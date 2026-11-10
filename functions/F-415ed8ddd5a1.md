<!-- Trusted provenance: F-415ed8ddd5a1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-415ed8ddd5a1

## Purpose

F-415ed8ddd5a1 dispatches a signed 32-bit command using a caller-supplied object address, context value, and output-word address.

Its statically established responsibilities are:

- Perform conditional global pre-dispatch work before classifying any command.
- Route recognized commands to dependencies using object-relative addresses, global values, or both.
- Pass the address of local temporary storage to dependencies on two command paths.
- Write a dependency-produced 32-bit output and return 1 on locally handled paths.
- Otherwise delegate the request to F-c755715b271a and return that dependency’s result.

The temporary-address calls do not establish construction, allocation, destruction, or release. Dependency implementations are unavailable.

“Command,” “context,” “selector,” and “resource” describe structural roles. Application-level meanings are not established.

## Interface

### Arguments and return value

| Argument | Representation and use |
|---|---|
| Object address | A 32-bit address used for fixed-offset accesses, dependency arguments, and fallback dispatch. |
| Context | A 32-bit value compared for equality with a global discriminator on selected paths. |
| Command | A signed 32-bit value used in equality and ordered comparisons. Its original value also initializes local storage. |
| Output address | An address for a 32-bit output word; also passed to the fallback dependency. |

The function returns one 32-bit word. Local handling writes the output and returns exactly 1. Delegation returns the fallback result unchanged; the fallback result may itself equal 1.

There is no separate established error result. Dependency return values generally become arguments or output without local validation.

### Global roles

The following names identify global references or values, not application semantics:

| Role | Use |
|---|---|
| Gate descriptor | Supplies an address whose 32-bit word at +8 controls pre-dispatch work. |
| Gate object | Argument to the pre-dispatch query and later notification. |
| Shared target | Argument to several operation and final-result dependencies. |
| Transfer arguments A and B | Respectively the second and third arguments of F-804adc29bbfb. |
| Upper and lower boundaries | Signed command-comparison values. |
| Default selector | A 32-bit value captured after pre-dispatch work and before routing. |
| Accepted context | Equality discriminator for commands 0x5799–0x579C and 0x579E. |
| Initial temporary descriptor | Global storage whose address is passed on command 0x5792. |
| Sequence resource selector | Value used in the second portion of command 0x5792. |
| Zero-byte and nonzero-byte resource selectors | Alternative values used by command 0x579D. |
| Final selector | Value used by command 0x579E. |

Their numeric values and stability are unknown. Distinct role names do not prove distinct values. A global used after a dependency call must not be treated as an earlier snapshot unless this contract explicitly describes such a snapshot.

### Object-relative layout

| Offset | Established access or use |
|---|---|
| +0x6C | Address supplied to F-08e2860e8e52 when the word at +0xA0 is not 1. |
| +0x74 | Address supplied to F-ecd1af0519f9 for command 0x578F. |
| +0x84 | Address supplied to F-2c8da614aa9f and subsequently F-ecd1af0519f9 for command 0x5785. |
| +0x8C | 32-bit word compared with 0, 1, 2, or 3. |
| +0x9C | 32-bit word whose low eight bits determine a branch. |
| +0xA0 | 32-bit word compared with exactly 1. |
| +0xA4 | Alternative address supplied to F-08e2860e8e52. |

Passing an address does not establish the size, representation, or ownership of the addressed subobject.

### Preconditions

Each selected path requires valid storage for its explicit accesses and arguments acceptable to its dependencies. No explicit null, alignment, object-size, or address-overflow checks are shown.

No output/object/global disjointness requirement is established. Aliasing can make the timing of writes significant. Effects of dependency calls through aliased storage remain unknown.

## Behavior

### Initialization and pre-dispatch work

Before consulting the gate, the function initializes local storage with the original command and stores the output address in another local slot. The command-initialized storage is later exposed by address on commands 0x5792 and 0x579D. Its complete representation is not established.

The function reads the 32-bit word at gate descriptor +8 and tests bit 0. Other bits do not affect this decision.

If bit 0 is set, all pre-dispatch dependency calls are skipped. Otherwise, F-b255b26284e6 is called with the gate object. A zero result ends pre-dispatch work.

A nonzero query result causes this ordered interaction:

- F-5b24edfb22c0 receives the shared target.
- F-804adc29bbfb receives that result, transfer argument A, and transfer argument B.
- F-d08d3ca17b81 receives the gate object as obtained for this later call.

The latter two results do not determine routing. Their side effects remain possible. In particular, the later gate-object argument is not guaranteed to equal the query argument if intervening dependencies modify its global reference.

The default selector is loaded only after this pre-dispatch work. It is then retained for applicable context-qualified selection paths. Pre-dispatch work therefore also occurs for ordinary fallback and context rejection.

### Routing precedence

Command ordering comparisons are signed 32-bit comparisons. Routing gives the upper boundary first priority:

| Condition | Routing consequence |
|---|---|
| Command equals upper boundary | Indexed operation 3. |
| Command is less than upper boundary | Lower routing region. |
| Command is greater than upper boundary | Upper routing region. |

Within the lower region, equality with the lower boundary selects indexed operation 0. Commands below the lower boundary are eligible for cases 0x5785, 0x578F, and 0x5792. Commands above the lower boundary are eligible for cases 0x5796 and 0x5797. Unmatched values delegate.

Within the upper region, cases 0x5799 through 0x579E are recognized. Other values delegate.

Numeric case labels are therefore conditional on earlier boundary comparisons. Equality with the upper boundary overrides any identically valued named case. Equality with the lower boundary has its described effect only after entry into the lower region. No ordering between the two global boundaries is established.

### Indexed operations

The four indexed routes select:

| Route | Index |
|---|---:|
| Lower-boundary equality in the lower region | 0 |
| Reachable command 0x5796 | 1 |
| Reachable command 0x5797 | 2 |
| Upper-boundary equality | 3 |

The function reads the 32-bit word at object +0xA0. Exactly 1 selects object +0xA4; every other value selects object +0x6C.

F-08e2860e8e52 receives the selected address and index. Its result is passed to F-63e4b97ed5e4 together with the shared target read for that call. The return value of F-63e4b97ed5e4 is unused.

After that call returns, the shared target is obtained again for F-ecd1af0519f9. The latter result is written to the output address, and the function returns 1.

The shared target used for final processing need not equal the one used by F-63e4b97ed5e4.

### Direct object-relative operations

Reachable command 0x5785 calls F-2c8da614aa9f with the object address and object +0x84. After it returns, F-ecd1af0519f9 receives object +0x84. Its result becomes the output, followed by return value 1.

Reachable command 0x578F calls F-ecd1af0519f9 with object +0x74, writes its result, and returns 1.

Neither path locally validates the addressed storage or interprets a dependency result as an error.

### Command 0x5792: two temporary-address phases

When reachable, this command first calls F-e208c7bcfbce with the object address and shared target.

The first phase begins with F-398e0e9a318a receiving the local temporary address and the address of the initial temporary descriptor. F-44362605ab79 is then called without explicit arguments. Its result is supplied as the third argument of F-f0b79b581bd5, whose first two arguments are the shared target obtained after that result is produced and 0x7FFFFFFF. F-289ec4311c64 then receives the temporary address.

The second phase obtains a value from F-9d0ecc5f6adf. After that call returns, the sequence resource selector is read and passed with the obtained value to F-c0b020cd3882. F-7afe554bcf62 receives the temporary address and that lookup result; its return value is unused on this path.

A second, separate F-44362605ab79 call supplies another third argument to F-f0b79b581bd5. This call again receives the shared target obtained for it and 0x7FFFFFFF. F-289ec4311c64 then receives the temporary address a second time.

Finally, the shared target is read after the second F-289ec4311c64 call and passed to F-ecd1af0519f9. Its result becomes the output, and the function returns 1.

The two F-44362605ab79 results are not assumed equal. The value 0x7FFFFFFF is the maximum positive signed 32-bit integer, but its semantic role is unknown. The temporary-address calls establish neither ownership transfer nor release.

### Commands 0x5799–0x579C: context-qualified selection

For each reachable case, the context is first compared with the accepted-context global.

On mismatch, the function writes a 32-bit zero to the output address and then calls F-c755715b271a with the original four arguments. It returns the fallback result unchanged. The zero write precedes delegation; it does not guarantee a final output of zero.

On a match, the function reads the 32-bit word at object +0x8C and compares it with the command-specific expected value:

| Command | Expected value |
|---|---:|
| 0x5799 | 0 |
| 0x579A | 1 |
| 0x579B | 2 |
| 0x579C | 3 |

That comparison occurs before F-9d0ecc5f6adf is called. An equal value selects the default selector captured before routing; an unequal value selects zero.

F-ea9442e8ddbb receives the F-9d0ecc5f6adf result and the selected value. Its result becomes the output, followed by return value 1.

A mismatch in the object word does not directly produce output zero: zero is an argument to F-ea9442e8ddbb. Changes to object +0x8C or the default-selector global during F-9d0ecc5f6adf do not revise the earlier comparison or captured selector.

### Command 0x579D: early branch, late selector read

This reachable case performs no accepted-context comparison.

The function first reads the full 32-bit word at object +0x9C and tests its low eight bits. This establishes which resource-selector global will later be used:

- Low byte zero chooses the zero-byte selector branch.
- Low byte nonzero chooses the nonzero-byte selector branch.

**The branch decision precedes F-9d0ecc5f6adf, but the selected global’s value is read only after that dependency returns.** The function does not snapshot either resource-selector value before the call.

Consequently, F-9d0ecc5f6adf may change either selector, and the subsequent lookup uses the then-current value of the selector chosen by the earlier branch. Changing object +0x9C during the call does not cause the branch to be reconsidered.

F-c0b020cd3882 receives the dependency result and the late-read selector. Its result is passed with the temporary address to F-7afe554bcf62.

Unlike command 0x5792, this path uses the return value of F-7afe554bcf62: F-46f0b3cb316e receives it together with the shared target obtained after temporary processing. F-289ec4311c64 then receives the temporary address.

After that call, the shared target is read for F-ecd1af0519f9. Its result is written to the output, and the function returns 1.

### Command 0x579E: context-qualified final selector

A reachable context mismatch performs the same zero-write-before-fallback behavior as commands 0x5799–0x579C.

On a match, F-9d0ecc5f6adf is called first. The final-selector global is read after that call returns. F-ea9442e8ddbb receives the returned value and this late-read selector; its result becomes the output, followed by return value 1.

This accepted-context path performs no direct object-field read.

### Ordinary fallback

Any command not selected by the preceding routing rules is passed to F-c755715b271a with the original object address, context, command, and output address.

Ordinary fallback has no explicit output initialization by this function. Its return value is propagated unchanged.

## Side effects

Explicit caller-memory writes are limited to:

- One 32-bit result write on each locally handled path.
- One 32-bit zero write immediately before context-rejection fallback.

Local initialization also writes local storage. Dependencies receive the temporary address on commands 0x5792 and 0x579D and may modify that storage. No direct object-field write is shown.

Dependencies may modify objects, globals, output storage, or other reachable memory. No general preservation guarantee is available. Replacing later global reads with earlier snapshots can change behavior, notably for resource selectors, the final selector, and repeated shared-target uses.

F-289ec4311c64 is called twice on the normal 0x5792 path and once on the normal 0x579D path. The calls establish processing of a temporary address, not memory release or destruction.

No dependency failure is checked except the zero/nonzero gate query. There is no shown recovery path guaranteeing later calls or output writes if a dependency fails to return normally.

## Dependencies

These are call-site roles only.

| Opaque ID | Established role |
|---|---|
| F-b255b26284e6 | Gate query controlling additional pre-dispatch calls. |
| F-5b24edfb22c0 | Supplies the first transfer argument. |
| F-804adc29bbfb | Receives that result and two global transfer arguments. |
| F-d08d3ca17b81 | Receives the gate object after transfer. |
| F-08e2860e8e52 | Receives an object-relative address and index 0–3. |
| F-63e4b97ed5e4 | Receives the shared target and indexed-operation result. |
| F-ecd1af0519f9 | Supplies final output on several handled paths. |
| F-2c8da614aa9f | Receives the object and its +0x84 address. |
| F-e208c7bcfbce | Receives the object and shared target before the two-phase path. |
| F-398e0e9a318a | Receives temporary storage and a global descriptor address. |
| F-44362605ab79 | Supplies two separately obtained sequence values. |
| F-f0b79b581bd5 | Receives the shared target, 0x7FFFFFFF, and a sequence value. |
| F-289ec4311c64 | Receives temporary storage after preceding operations. |
| F-9d0ecc5f6adf | Supplies an input for lookup or selection. |
| F-c0b020cd3882 | Receives an obtained value and resource selector. |
| F-7afe554bcf62 | Receives temporary storage and a lookup result. |
| F-46f0b3cb316e | Receives the shared target and temporary-processing result. |
| F-ea9442e8ddbb | Supplies output from an obtained value and selector. |
| F-c755715b271a | Handles delegation and determines its return value. |

## Acceptance examples

These are conditional static criteria, not executed tests. All mentioned dependencies are assumed to return normally.

1. **Gate suppression.** If gate bit 0 is set, none of the four pre-dispatch dependencies runs. Routing still proceeds.

2. **Gate activation.** If the bit is clear and the query returns nonzero, F-5b24edfb22c0 precedes F-804adc29bbfb, which precedes F-d08d3ca17b81. The default selector is captured afterward.

3. **Boundary precedence.** If the command equals the upper boundary, indexed operation 3 runs even when that number also names another case.

4. **Exact indexed selection.** On an indexed route, +0xA0 equal to 1 selects +0xA4. Values 0, 2, and 0xFFFFFFFF select +0x6C.

5. **Context rejection.** For reachable 0x579A with rejected context, output zero is written before fallback. If fallback subsequently writes another value and returns 7, that later output remains subject to fallback behavior and this function returns 7.

6. **Object-selection mismatch.** For reachable 0x579B with accepted context and +0x8C unequal to 2, F-ea9442e8ddbb receives zero as its second argument. Its own result becomes the output.

7. **Low-byte boundary.** For reachable 0x579D, 0x00000100 at +0x9C chooses the zero-byte branch; 0x00000101 chooses the nonzero-byte branch.

8. **Late resource-selector read.** Suppose 0x579D chooses the zero-byte branch, and F-9d0ecc5f6adf changes that selector from value A to value B. F-c0b020cd3882 receives B, not A. A concurrent change by that dependency to the object word does not select the other branch. The corresponding late-read rule applies to the nonzero-byte branch.

9. **Captured default versus late final selector.** Changes to the default-selector global during F-9d0ecc5f6adf do not replace the captured value on a matching 0x5799–0x579C path. Changes to the final-selector global during that dependency are reflected on accepted 0x579E.

10. **Two-phase ordering.** Reachable 0x5792 obtains two separate F-44362605ab79 results. Each F-f0b79b581bd5 call is followed by F-289ec4311c64 before final-result processing. No release semantics are required by this criterion.

11. **Ordinary delegation.** An unmatched command reaches fallback without an explicit output-zeroing write by this function.

## Uncertainties

### Established static evidence

The supplied body establishes widths, offsets, signed command comparisons, branch precedence, explicit global-read placement relative to calls, dependency arguments, call order, output writes, and return handling.

### Inferred intent

The structure resembles command dispatch with indexed operations, resource selection, and temporary-state processing. These descriptions do not establish application meaning or ownership protocols.

### documented_only unknowns

- Global values, address targets, stability, and boundary ordering.
- Reachability of every numeric case under actual boundary values.
- Complete object and temporary-storage layouts.
- Byte order and precise alignment requirements.
- Dependency allocation, retention, release, and ownership behavior.
- Whether temporary addresses escape or remain valid for retained use.
- Dependency mutation of globals, arguments, or aliased memory.
- Error, exception, reentrancy, and concurrency behavior.
- Semantic interpretation of selectors, output words, and 0x7FFFFFFF.
- Address validity and arithmetic behavior near the 32-bit address-space limit.

Fixed-offset address formation reaches object +0xA4; direct 32-bit reads include object +0xA0. No bounds or overflow handling is established. The low-byte branch is defined by the low eight bits of the read 32-bit value and does not require an assumed byte order.

## Evidence

This contract derives from static inspection of the supplied function body, using the supplied opaque mapping to identify dependencies. Prior prose, comments, and feedback are not independent evidence of behavior.

The source places the 0x579D object-word test before F-9d0ecc5f6adf and the selected resource-selector load after that call in either branch. It also shows temporary-address dependency calls without supplying implementations that establish construction or release.

No runtime tests have been performed. Acceptance examples describe conditional obligations, not observed execution results.
