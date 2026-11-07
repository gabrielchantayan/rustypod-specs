<!-- Trusted provenance: F-13f92cfed3c1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-13f92cfed3c1

## Purpose

F-13f92cfed3c1 conditionally performs a bulk state reset or processes a cursor range one element at a time. It writes a caller-provided output word and returns a separate 64-bit packed value.

The visible accesses are consistent with a container holding a header or sentinel pointer and a count-like field. That interpretation is inferred, not established. The contract below therefore describes fields by offset and dependencies by opaque function ID.

Two distinctions are essential:

- The initial shortcut checks accept **any nonzero** dependency result. The processing loop stops only when its comparison result is **exactly 1**.
- The returned 64-bit value is not simply the output word. Its upper and lower 32-bit halves come from separate working slots, whose contents depend on the path taken.

## Interface

### Arguments

| Argument | Observed representation | Role |
|---|---|---|
| Output destination | Address of a writable 32-bit word | Receives an initial header value, a replacement value after each per-element operation, and a final header value on the reset path. |
| Context | 32-bit address-valued argument | Identifies a structure with readable fields at offsets `0x10` and `0x14`; those fields can also be written or indirectly affected. |
| Cursor | Address of a 32-bit word, also passed as an opaque dependency argument | Used in comparisons, read directly before each per-element operation, and passed to a cursor-update dependency. |
| Endpoint | Opaque 32-bit value | Passed to comparison dependencies. This function does not directly dereference it. |

The representation shown uses 32-bit address values and 32-bit memory words. A portable implementation must preserve these widths rather than silently substituting host-sized pointers or integers.

### Visible structure fields

The context has these observed fields:

| Offset | Width | Observed use |
|---|---:|---|
| `0x10` | 32 bits | Address of a header-like object; repeatedly reloaded. |
| `0x14` | 32 bits | Tested for zero and cleared on the reset path. |

The object addressed by the field at context offset `0x10` has these observed fields:

| Offset | Width | Observed use |
|---|---:|---|
| `0x04` | 32 bits | Passed to F-2ef485ee6933, then cleared on the reset path. |
| `0x08` | 32 bits | Copied for the first comparison; overwritten with a header address on reset. |
| `0x0c` | 32 bits | Overwritten with a header address on reset. |

No complete structure size, other field meanings, or object ownership is established.

### Return value

The function returns 64 bits assembled from two 32-bit working slots:

- The **upper half** contains the final context/result working slot.
- The **lower half** contains the final output/element working slot.

Assembly preserves each half’s 32-bit pattern. There is no evidenced sign extension of the lower half, and the return value should not be interpreted as a status code without an external contract.

On the reset path, the upper half is explicitly set to the final header address. On the processing path, both halves depend on initialization and on any writes made through the slots passed to F-4b95025d68e8.

### Preconditions

Every memory access actually reached must address valid memory of the required readability or writability. In particular:

- The output destination must support the initial 32-bit store.
- The context must support the header-field reads.
- The initially selected header must support a read at offset `0x08`.
- The reset path requires additional header and context writes.
- Each entered loop iteration requires a readable 32-bit cursor word.

Dependency-specific argument requirements are unknown. There are no visible null checks, bounds checks, allocation checks, or alignment checks.

## Behavior

### Initial publication and working state

The first observable memory operation is a read of the context field at offset `0x10`, followed by a store of that 32-bit value to the output destination.

The function then reads the context field at offset `0x10` again and copies the selected header’s word at offset `0x08` into a local comparison slot. The second header lookup is not specified as reuse of the first lookup.

This matters when the output destination aliases context storage: the first output write can affect the later read.

Before the first dependency call, the function also initializes working slots with:

- The output destination’s address.
- The original context argument.
- The cursor argument.

These slots are distinct from the caller’s output word.

### Shortcut eligibility checks

The function calls F-e1c8edf49b9a with:

1. The cursor argument.
2. The address of the local slot containing the copied header word from offset `0x08`.

This is a comparison against a copied word, not a direct pointer to the header field. Writes through that slot address do not, by themselves, update the original header field.

If the result is zero, the shortcut is abandoned and execution proceeds to the processing loop.

If the result is nonzero:

1. The current context field at offset `0x10` is loaded into another local slot.
2. F-e1c8edf49b9a is called with the endpoint value and the address of that slot.
3. Only if this second result is nonzero is the context field at offset `0x14` read and tested.

The reset path is selected only when all three conditions hold:

- First comparison result is nonzero.
- Second comparison result is nonzero.
- Context word at offset `0x14` is nonzero.

Negative comparison results qualify as nonzero. There is no requirement that either shortcut comparison return exactly 1.

If any condition fails, execution proceeds to the processing loop. The shortcut results are not reused as the loop’s termination decision.

### Reset path

The reset path first calls F-2ef485ee6933 with:

1. The original context argument.
2. The word at offset `0x04` of the header selected by the context’s current field at offset `0x10`.

The helper’s return value is not used.

After that call returns, the function performs the following stores in order:

1. Store a current header address into offset `0x08` of a currently selected header.
2. Store zero into offset `0x04` of the currently selected header.
3. Store a current header address into offset `0x0c` of a currently selected header.
4. Store zero into context offset `0x14`.
5. Reload the header address from context offset `0x10` and store it to the output destination.

The header field is re-read for these operations. The function does not visibly retain one immutable header pointer across F-2ef485ee6933 and all subsequent writes.

Consequently, a dependency-induced change to context offset `0x10` can redirect later writes. Aliasing can also make an earlier store affect a later lookup. Within a single self-linking store, the exact order of the address-forming reads is not established by the supplied material.

After the final output write, the function returns a 64-bit value with:

- Upper 32 bits: the header address loaded for that final output write.
- Lower 32 bits: the output-destination address held in the output/element working slot.

The function does not subsequently enter the processing loop.

### Processing loop

On every loop condition check, including the first, the function calls F-e1c8edf49b9a with the original cursor argument and the endpoint value.

The loop terminates only when this call returns **exactly 1**.

For any other result—including zero, negative values, or positive values other than 1—the function performs one iteration:

1. Read the 32-bit word at the cursor argument and retain that word as the current element value.
2. Call F-4193c824164f with the cursor argument.
3. Put the retained element value into the output/element working slot.
4. Call F-4b95025d68e8 with:
   - The address of the context/result working slot.
   - The original context argument.
   - The address of the output/element working slot.
5. Store the current contents of the context/result working slot to the caller’s output destination.

The cursor-word read occurs **before** F-4193c824164f. Therefore, the value initially supplied through the third argument to F-4b95025d68e8 is the pre-update cursor word, not a word re-read after cursor advancement.

The context/result working slot is initialized to the original context argument once before the comparisons. It is not visibly reinitialized before each per-element call. The output/element working slot, by contrast, is replaced with the newly retained cursor word on each iteration.

F-4b95025d68e8 receives writable addresses for both working slots. Its modifications to the context/result slot determine the following output write. Its modifications to the output/element slot remain available for return unless a later iteration replaces that slot.

After the terminating comparison returns 1, the function returns the two working slots packed into 64 bits. It performs no additional output write at loop exit.

### Zero-iteration case

If the first loop comparison returns 1, neither per-element dependency is called.

Absent other side effects:

- The output word remains the header value written at entry.
- The return value’s upper half is the original context argument.
- The return value’s lower half is the output-destination address.

Thus, the returned upper half and the output word can differ even on an ordinary terminating path.

## Side effects

### Direct writes

The function directly writes:

- The output destination at entry.
- Header offsets `0x08`, `0x04`, and `0x0c`, plus context offset `0x14`, on the reset path.
- The output destination after the reset.
- The output destination after every completed per-element call.

These effects are not transactional. Earlier writes are not rolled back if a later access or dependency fails.

### Dependency-visible state

Dependencies receive original arguments, copied values, and addresses of local slots. Some receive the context directly. They may therefore observe or change additional state according to their own contracts.

No guarantee is established that any dependency:

- Preserves globals or context fields.
- Preserves memory reachable through its arguments.
- Leaves cursor or endpoint state unchanged.
- Allocates or releases memory.
- Terminates normally.

The reset helper’s placement before the reset stores is established. Whether it destroys a subtree, releases resources, or performs another operation is not.

### Ownership and lifetime

No direct allocation or release is visible. Ownership transfer cannot be inferred merely from passing a header field or element value to a dependency.

Addresses of local slots are supplied during dependency calls. Retention of those addresses is not established; they cannot be treated as persistent storage valid after this invocation returns.

### Errors and termination

There is no explicit error return or error-recovery branch. Comparison results control flow rather than report a documented failure category.

The loop has no visible iteration bound. Termination depends on F-e1c8edf49b9a eventually returning exactly 1, or on an external interruption or failure. A dependency returning 2 indefinitely does not terminate the loop.

## Dependencies

| Opaque ID | Observed role | Contract limits |
|---|---|---|
| F-e1c8edf49b9a | Performs both shortcut checks and every loop condition check. | Argument interpretation and mutation behavior are unknown. Shortcut checks use nonzero truth; loop termination uses exact equality to 1. |
| F-2ef485ee6933 | Called once on the reset path with the context and current header offset-`0x04` word. | Return value is ignored. Resource effects and context mutations are unknown. |
| F-4193c824164f | Called once per entered iteration after saving the cursor word. | Cursor advancement is inferred intent, not a proven complete contract. |
| F-4b95025d68e8 | Called once per iteration with two writable working slots and the original context. | Can modify both slots. Removal, ownership, and resource effects remain unspecified. |

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, assume valid, nonoverlapping memory and no dependency side effects beyond those explicitly described.

### Reset selected by non-Boolean results

Suppose:

- Context offset `0x10` contains header address **H**.
- Context offset `0x14` contains 7.
- The first shortcut comparison returns −1.
- The second shortcut comparison returns 2.
- F-2ef485ee6933 leaves the context and header selection unchanged.

Required behavior:

- Both comparison results qualify.
- F-2ef485ee6933 receives the context and the initial word at **H + `0x04`**.
- The final header words at offsets `0x08` and `0x0c` contain **H**.
- The header word at offset `0x04` and context word at offset `0x14` contain zero.
- The output word contains **H**.
- The return has **H** in its upper half and the output-destination address in its lower half.
- No processing-loop comparison occurs.

### Empty count prevents reset

Suppose both shortcut comparisons return nonzero, but context offset `0x14` contains zero.

Required behavior:

- F-2ef485ee6933 is not called.
- The reset stores do not occur.
- A separate loop comparison is still performed.
- If that comparison returns 1, no per-element operation occurs.

### Loop result 2 does not mean termination

Suppose the shortcut is bypassed and the next two loop comparisons return 2 and 1.

Required behavior:

- Exactly one iteration occurs.
- The cursor word is captured before F-4193c824164f.
- F-4b95025d68e8 receives that captured word through its third argument.
- One per-iteration output write occurs.
- The second comparison terminates the loop.

### Per-element slot modifications affect output and return

Suppose one iteration occurs. F-4b95025d68e8 writes **R** to its first referenced slot and **E** to its third referenced slot. The following comparison returns 1.

Required behavior:

- The iteration writes **R** to the caller’s output word.
- The return’s upper half contains **R**.
- The return’s lower half contains **E**, not necessarily the originally captured cursor word.

### Reset helper changes the selected header

Suppose the reset helper changes context offset `0x10` from **H** to **J**, and subsequent stores do not alter that selection.

Required behavior:

- The helper’s second argument was obtained from the selected header before its call.
- Post-call header stores target **J**.
- The final output word and return upper half contain **J**.

## Uncertainties

The following are **documented_only**:

- Complete dependency contracts, including failures, resource ownership, and retained references.
- Whether the context represents a tree, another container, or a different structure.
- Whether offset `0x14` is an element count rather than another nonzero state indicator.
- The precise logical relationship implemented by F-e1c8edf49b9a.
- Supported aliasing arrangements and concurrency guarantees.
- Exact evaluation order of multiple address-forming reads within one assignment.
- Runtime validity, alignment requirements, fault behavior, and exception or unwinding behavior.

Inferred intent is a full-range reset optimization followed by a cursor-driven per-element removal fallback. That interpretation does not establish equality semantics, deallocation behavior, or ownership transfer.

## Evidence

Established behavior comes solely from static inspection of the supplied function representation:

- Explicit 32-bit field accesses and a 64-bit packed return.
- Direct comparison against zero in shortcut selection.
- Direct comparison against 1 for loop termination.
- Visible dependency arguments and call order.
- Cursor capture before the per-iteration update call.
- Repeated header-field loads around reset writes.
- Writable working-slot addresses passed to the per-element dependency.

No runtime tests have been performed. The acceptance examples specify conditional consequences of the visible behavior; they are not execution reports.
