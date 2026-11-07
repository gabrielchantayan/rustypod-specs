<!-- Trusted provenance: F-dc139e6efdaf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dc139e6efdaf

## Purpose

F-dc139e6efdaf initializes a caller-supplied output word from a context’s header reference, then selects between two behaviors:

- A conditional bulk-operation path that invokes a dependency, restores an empty-looking header configuration, clears a context field, and publishes the current header reference.
- An iterative path that captures a cursor word, advances the cursor through one dependency, passes the captured word to another dependency, and publishes that dependency’s resulting scratch value.

The inferred intent is to process a cursor-delimited portion of a linked container, with a shortcut for processing the entire container. Container type, element ownership, and the meaning of the cursor comparisons are not established. This contract therefore defines the observable word-level behavior without requiring that interpretation.

The displayed return is a 64-bit concatenation of two 32-bit words. It is not established to be an ordinary scalar result at the original calling interface.

## Interface

The function has four inputs:

| Input | Representation | Established use |
|---|---|---|
| Output destination | Address of a writable 32-bit word | Receives an initial header reference and subsequent path-dependent updates. |
| Context | Address-sized value represented in 32 bits | Provides a header-reference field at byte offset `0x10` and a control/count-like field at byte offset `0x14`. Also passed to dependencies. |
| Mutable cursor | Address of a 32-bit word | Passed to comparison and advancement dependencies; its first word is captured before each advancement. |
| Boundary operand | Opaque 32-bit value | Passed directly to a comparison dependency. Its required interpretation is unknown. |

The supplied representation uses 32-bit words for pointers, context references, fields, and scratch values. The reported result consists of a high 32-bit word and a low 32-bit word. No sign extension is involved in that concatenation.

### Referenced layout

The context has at least these accessible fields:

| Byte offset | Width | Role in this function |
|---|---:|---|
| `0x10` | 32 bits | Header reference, repeatedly reloaded |
| `0x14` | 32 bits | Nonzero gate for the bulk path; cleared by that path |

The object addressed by the header reference has at least these accessible fields:

| Byte offset | Width | Role in this function |
|---|---:|---|
| `0x04` | 32 bits | Supplied to the bulk dependency; later cleared |
| `0x08` | 32 bits | Initial comparison value; later assigned a header self-reference |
| `0x0c` | 32 bits | Later assigned a header self-reference |

No field at header offset zero is directly accessed by the supplied body.

### Preconditions

For ordinary completion, every address actually accessed on the selected path must be valid for the corresponding read or write. The function performs no explicit null, bounds, alignment, or lifetime checks.

In particular:

- The output destination must permit a 32-bit write immediately on entry.
- The context’s header-reference field must initially be readable.
- The header’s `0x08` field must be readable before any dependency is called.
- The mutable cursor must support the operations required by its dependencies and, on every iterative pass, a direct 32-bit read.
- The bulk path additionally requires the relevant context and header fields to remain valid after its dependency returns.

Requirements concerning aliasing, concurrent access, and dependency-specific operand validity are not established. Aliases can affect subsequent reads because the function does not preserve all context and header values in a stable snapshot.

## Behavior

### Entry initialization

Before calling any dependency, the function:

1. Reads the header reference from context offset `0x10` and writes that reference to the output destination.
2. Reads the context’s header reference again, then reads the word at header offset `0x08`.
3. Places that word in a local comparison slot.

The two header-reference reads are distinct in the supplied behavior. For example, if the output destination aliases the context’s header-reference field, the initial output write occurs before the second read.

The function also initializes scratch state containing:

- The output-destination address.
- The context address.
- The mutable-cursor address.

Some scratch slots are subsequently reused. Their later contents must not be assumed to retain their initial meanings.

### Bulk-path eligibility

The function calls **F-2b931655bb54** with:

- The mutable cursor as its first operand.
- The address of the local comparison slot as its second operand.

If the result is zero, it proceeds directly to the iterative path.

If the result is nonzero, the function:

1. Reloads the context’s header reference into another local slot.
2. Calls **F-2b931655bb54** with the boundary operand and the address of that slot.
3. Reads context offset `0x14` only if this second result is nonzero.

The bulk path is selected only when both comparison results are nonzero and the subsequently read field at context offset `0x14` is nonzero.

These are truth-value checks, not exact-one checks. Negative results and nonzero results other than one satisfy either eligibility comparison.

The comparison dependency receives addresses of local words in these calls. Whether it only inspects those words or also modifies them is unknown. Changes to those particular comparison slots are not directly used as replacement context references by the function after the comparisons.

### Bulk path

On selection of the bulk path, the function calls **F-657b7c1c9d1e** with:

- The context address.
- The word read from offset `0x04` of the header currently referenced by context offset `0x10`.

After that dependency returns normally, the function performs the following writes in order:

1. Assigns a header reference to header offset `0x08`.
2. Writes zero to header offset `0x04`.
3. Assigns a header reference to header offset `0x0c`.
4. Writes zero to context offset `0x14`.

The header-reference field is reloaded for these operations; it is not necessarily the header that was read before the dependency call. For the self-reference assignments, both the target object and the reference value are obtained through the context’s header-reference field. The supplied representation does not establish a finer evaluation order between the source and destination address calculations within an individual assignment.

After those writes, the function reloads the context’s header reference, writes it to the output destination, and returns:

- **High 32 bits:** the final reloaded header reference.
- **Low 32 bits:** the scratch word initialized with the output-destination address.

There is no direct bulk-path assignment changing that low scratch word. Unusual alias-mediated dependency effects are not ruled out by the available evidence.

No iterative comparison or cursor advancement is performed after the bulk path is selected.

### Iterative path

If the bulk path is not selected, the function repeatedly calls **F-2b931655bb54** with the mutable cursor and the boundary operand.

This loop terminates only when the comparison result is **exactly one**. Zero, negative values, and every other positive value cause another iteration.

For each nonterminating comparison result, the function performs these operations in order:

1. Reads the first 32-bit word addressed by the mutable cursor and captures it.
2. Calls **F-2f0be07eb2b5** with the mutable-cursor address.
3. Copies the captured pre-advancement word into a scratch slot.
4. Calls **F-58b2da07e615** with:
   - The address of the context/result scratch slot.
   - The original context argument.
   - The address of the slot holding the captured word.
5. Writes the current context/result scratch value to the output destination.

The context/result scratch slot initially contains the context address. It is not explicitly reinitialized between iterations. The captured-word slot is replaced from the current cursor word on every iteration, after the advancement dependency returns.

Both scratch slots are passed by address to F-58b2da07e615. Its modifications to them therefore affect subsequent output writes and the final packed result.

The first argument to that dependency is a writable local slot, not the context object itself. The second argument remains the original context argument even if the first slot has been changed by an earlier iteration.

When a comparison finally returns exactly one, the function returns:

- **High 32 bits:** the current context/result scratch value.
- **Low 32 bits:** the current captured-word scratch value.

It performs no additional output write at termination.

If the iterative path executes zero iterations, the output retains the initial header-reference write, while the returned high word is the original context address and the returned low word is the output-destination address, subject to indirect alias effects.

## Side effects

### Direct writes

The function directly writes:

- The output destination on entry.
- The output destination after every completed iterative dependency call.
- On the bulk path, header fields at offsets `0x08`, `0x04`, and `0x0c`, followed by context offset `0x14`, followed by the output destination.

The initial output write is observable even if a later operation fails or never returns.

### Dependency effects and ownership

Dependencies may mutate their operands, the context, referenced objects, or other reachable state. The supplied body does not establish purity, allocation behavior, ownership transfer, deallocation, or preservation of global state for any dependency.

No allocation or release operation is directly visible in this function. That does not establish that the operation as a whole allocates or releases nothing.

### Errors and termination

There is no explicit error return, error flag, rollback, or exception handler.

Invalid accesses, dependency failures, nonlocal exits, and concurrent mutation behavior are outside the established contract. If a dependency does not return normally, later writes described here are not guaranteed.

The iterative path has no iteration limit and no independent progress check. If its comparison never returns exactly one, normal termination is not guaranteed.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| **F-2b931655bb54** | Returns a comparison-like integer. Called with cursor/local-slot, boundary/local-slot, or cursor/boundary operand combinations. Initial gates accept any nonzero result; loop termination requires exactly one. |
| **F-657b7c1c9d1e** | Called once on the bulk path with the context and current header field at offset `0x04`. Its returned value is not used. |
| **F-2f0be07eb2b5** | Called once per iterative pass with the mutable-cursor address, after capturing its current first word. Its returned value is not used. |
| **F-58b2da07e615** | Called once per iterative pass with two writable scratch-slot addresses and the original context argument. Scratch values after this call determine the output write and contribute to the final result. |

The roles “comparison,” “advancement,” and “bulk operation” describe observed usage or inferred intent, not independently established dependency contracts.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. They assume valid, nonoverlapping storage and dependencies that make only the changes explicitly stated.

### Bulk operation accepted by non-one results

Suppose the first two comparison calls return two and negative one, respectively, and context offset `0x14` contains a nonzero word.

The bulk path must be selected. Both comparison results satisfy the nonzero gates. After the bulk dependency returns, the function must perform the header reset writes, clear context offset `0x14`, and publish the current header reference.

The result’s high word must be that final header reference; its low word must be the output-destination address.

### Second comparison rejects the bulk path

Suppose the first comparison returns nonzero and the second returns zero.

The context field at offset `0x14` must not be read as part of the bulk eligibility check. The function must enter the iterative comparison path regardless of that field’s stored value.

### Nonzero field required for bulk selection

Suppose both eligibility comparisons return nonzero, but context offset `0x14` is zero.

The bulk dependency must not be called. The next selection operation is the iterative cursor/boundary comparison.

### Immediate iterative termination

Suppose the first eligibility comparison returns zero and the first iterative comparison returns exactly one.

No cursor-word capture, cursor advancement, or per-item dependency call occurs. The output remains the initially written header reference. The returned high word is the context address, not that header reference; the returned low word is the output-destination address.

### Comparison result two does not terminate iteration

Suppose an iterative comparison returns two.

The function must execute an iteration. Treating all nonzero results as loop termination would violate the supplied behavior.

### Capture precedes advancement

Suppose the cursor initially contains reference A. The advancement dependency changes it to reference B. The per-item dependency leaves its captured-word argument unchanged and writes result R into its context/result argument. The next comparison returns one.

The per-item dependency must receive A through the captured-word slot, not B. The output must become R, and the final result must contain R in its high word and A in its low word.

### Bulk dependency changes the header reference

Suppose the bulk dependency changes context offset `0x10` from header A to header B and leaves B valid.

The subsequent reset operations use the reloaded reference to B. An implementation that unconditionally resets a cached A instead would not match the established behavior.

## Uncertainties

The following are **documented_only**:

- The semantic type and purpose of the context and header.
- Whether the field at context offset `0x14` is a count, flag, or another nonzero indicator. Its value is not decremented or otherwise arithmetically processed here.
- The exact comparison relation implemented by F-2b931655bb54.
- The boundary operand’s representation and validity requirements.
- Whether the bulk dependency destroys elements, detaches them, or performs another operation.
- Ownership and lifetime consequences of every dependency.
- The intended consumer interpretation of the packed 64-bit result.
- Whether the original calling convention represents that result as a scalar, register pair, or another ABI-level construct.
- Valid aliasing configurations, thread-safety guarantees, and behavior under asynchronous mutation.
- Behavior after invalid memory access or abnormal dependency termination.

The apparent empty-header pattern is inferred from the two self-reference writes and two zero writes. It is not proof of a particular container invariant.

## Evidence

Established facts come solely from the supplied static function representation: 32-bit accesses, fixed offsets, branch conditions, dependency argument forms, operation sequencing, local-slot reuse, and the two-word return construction.

No dependency implementation, external caller contract, or independent layout definition was supplied. No runtime tests have been performed. The acceptance examples specify conditional consequences of the visible behavior; they are not execution results.
