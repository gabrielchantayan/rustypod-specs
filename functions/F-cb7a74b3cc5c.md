<!-- Trusted provenance: F-cb7a74b3cc5c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cb7a74b3cc5c

## Purpose

F-cb7a74b3cc5c conditionally performs a bulk operation or an iterative sequence on caller-supplied state. It writes a 32-bit result through an output address and separately returns a packed 64-bit value.

The bulk path calls F-c0d1b13ad3d8, resets three fields of an object selected through the context, clears a context field, writes the selected-object address to the output, and returns.

The iterative path repeatedly captures a word through a cursor argument, calls F-ee5c701f2f6f and F-e329dab7fb6c, and copies an internal result word to the output. A comparison result of exactly one ends this path.

The field arrangement and bulk reset suggest a sentinel-based structure. That interpretation is **inferred intent**, not an established semantic requirement. The available evidence does not establish that either path destroys objects, releases memory, or transfers ownership.

The output word and packed return are distinct results. In particular, immediate iterative termination can leave the output containing a selected-object address while returning the context address and output address as its two halves.

## Interface

### Arguments

The function accepts four arguments:

| Argument | Established role |
|---|---|
| Output address | Destination of an initial 32-bit write and subsequent path-specific 32-bit writes. |
| Context address | Base address for fields at byte offsets +16 and +20; also passed to structural dependencies. |
| Cursor argument | Passed directly to F-50be1c0375de and F-ee5c701f2f6f. Dereferenced for a 32-bit word at the beginning of each iterative body. |
| Boundary argument | A 32-bit value passed directly to F-50be1c0375de. The wrapper does not directly dereference it. |

The supplied representation uses 32-bit words for the relevant values, including stored addresses and the words used in the return packing. The returned value is 64 bits. No signed ordering comparison is present: branch decisions test dependency results against zero or exactly one.

### Directly accessed storage

All offsets below are byte offsets.

| Location | Access width | Established use |
|---|---:|---|
| Context +16 | 32 bits | Repeatedly read as an address-valued field. |
| Context +20 | 32 bits | Conditionally read for a nonzero test; cleared on the bulk path. |
| Selected object +4 | 32 bits | Read for the bulk dependency’s second argument; later cleared. |
| Selected object +8 | 32 bits | Read during initialization; assigned an address-valued word on the bulk path. |
| Selected object +12 | 32 bits | Assigned an address-valued word on the bulk path. |
| Output address | 32 bits | Written initially and at path-specific points. |
| Cursor argument | 32 bits | Read once per entered iterative body. |

“Selected object” means the object addressed by the relevant load from context +16. It is not necessarily one fixed object throughout the invocation.

The wrapper also maintains two 32-bit result cells:

- A **high-result cell**, initially containing the context address.
- A **low-result cell**, initially containing the output address.

F-e329dab7fb6c receives addresses of both cells. Separate internal comparison cells are supplied to eligibility comparisons.

### Return value

The return is a bitwise concatenation:

- Bits 63–32: the high word.
- Bits 31–0: the low word.

On the bulk path, the high word is the context +16 word loaded immediately before the final output write. The low word is the low-result cell, which the wrapper has initialized to the output address and has not explicitly replaced on this path.

On the iterative path, the high and low words are the result cells’ values when the loop terminates.

This packing does not establish a numerical meaning, success convention, pointer interpretation, or language-level pair type. It is not an arithmetic addition or a sign-extending conversion of either component.

### Preconditions

Normal execution requires every reached memory access to address suitable storage for its 32-bit operation. This includes accesses performed after dependencies return or earlier writes change aliased state.

The cursor argument needs to support the wrapper’s direct 32-bit read only when an iterative body is entered, but dependencies may impose additional requirements even when the body is skipped. The boundary argument likewise has dependency-defined requirements despite not being directly dereferenced by the wrapper.

No null check, bounds check, checked address arithmetic, or memory-access recovery is visible. Exact alignment requirements and behavior when offset calculations do not identify valid storage remain unresolved.

No argument-disjointness requirement is established. Aliasing can make earlier writes affect later reads. The access order described below must therefore be preserved.

## Behavior

### Initialization and unconditional first comparison

Before any dependency call, the wrapper reads context +16 and writes that 32-bit word to the output address. It then reads context +16 again and reads offset +8 of the object selected by this second load. That +8 word initializes an internal comparison cell.

The two context +16 reads are distinct in the supplied representation. The initial output write occurs between them. If the output aliases storage subsequently read, initialization observes the resulting memory state rather than a cached pre-write value.

Before the first dependency call, the high-result cell contains the context address, the low-result cell contains the output address, and another comparison cell contains the cursor argument.

**The first F-50be1c0375de call is unconditional on normal execution after initialization.** It receives:

- The cursor argument.
- The address of the comparison cell initialized from the selected object’s +8 word.

A zero result bypasses the remaining bulk-eligibility checks and proceeds to the iterative condition. Any nonzero result enables the second eligibility comparison. The first result need not equal one.

### Conditional second comparison and bulk eligibility

Only when the first comparison returns nonzero does the wrapper reload context +16 into the other internal comparison cell and call F-50be1c0375de again. This second eligibility call receives:

- The boundary argument.
- The address of that comparison cell.

If the second result is zero, context +20 is not read for eligibility, and execution proceeds to the iterative condition.

If the second result is nonzero, context +20 is read. The bulk path is selected only when that word is also nonzero.

Thus, bulk selection requires both comparison results and the context +20 word to be nonzero. Failing any applicable requirement leads to the iterative path, not directly to return.

### Bulk operation

The wrapper reloads context +16, reads offset +4 of the selected object, and calls F-c0d1b13ad3d8 with the context address and that +4 word. Its direct return value is unused.

After that call returns, the wrapper completes these writes in order:

1. Stores a current context +16 word at selected-object offset +8.
2. Stores zero at selected-object offset +4.
3. Stores a current context +16 word at selected-object offset +12.
4. Stores zero at context +20.
5. Reloads context +16 and stores that word at the output address.

The object-selection reads after the dependency call are not replaced by a single address cached before the call. A dependency change to context +16 can therefore redirect the reset operations.

Likewise, the assignments are not one indivisible reset. Earlier writes can affect later selections if storage aliases. For either assignment that uses context +16 both as a stored value and to select a destination, the supplied representation does not establish a finer machine-level ordering between those reads. The order of the completed assignments is established.

The packed return’s high word is the word loaded for the final output write. If that output write itself changes context +16 through aliasing, there is no subsequent context reload for the return. The low word comes from the low-result cell.

The wrapper returns immediately afterward. It does not evaluate an iterative condition on this path.

### Iterative condition

Whenever execution reaches the iterative path, F-50be1c0375de receives the original cursor argument and the original boundary argument directly.

A result of **exactly one** terminates the loop. Every other result enters the body, including zero and nonzero values other than one.

This predicate differs from the eligibility predicates, where any nonzero value satisfies the comparison requirement. Replacing all three uses with one normalized Boolean interpretation would change behavior.

The condition runs before the first body and after each completed body. No iteration limit or progress check is visible.

### Iterative body and completion

Each entered body first reads and saves the 32-bit word addressed by the cursor argument. It then calls F-ee5c701f2f6f with the cursor argument.

After that call returns, the wrapper replaces the low-result cell with the saved word. Consequently, a change to the cursor’s pointed-to word during F-ee5c701f2f6f does not change the word supplied for this body’s next call.

F-e329dab7fb6c then receives:

- The address of the high-result cell.
- The context address.
- The address of the low-result cell.

Its direct return value is ignored. The wrapper subsequently copies the current high-result cell to the output address.

Both result cells are supplied by address. F-e329dab7fb6c can therefore affect the output and packed return through writes to those cells. The wrapper does not reinitialize the high-result cell between bodies. It does replace the low-result cell with a newly captured cursor word in every body.

When a condition call returns one, the wrapper returns the current result cells packed into 64 bits. There is no additional output write at termination.

If the first iterative condition returns one, no body executes. In the absence of indirect dependency effects, the high-result cell still contains the context address, the low-result cell still contains the output address, and the output retains the word written during initialization.

## Side effects

Established direct external writes are:

- One initial 32-bit output write before any dependency call.
- On the bulk path, writes to selected-object offsets +8, +4, and +12, then context +20, then the output.
- On the iterative path, one output write after each completed body’s F-e329dab7fb6c call.

The wrapper also writes its internal cells and exposes several cell addresses to dependencies.

No explicit allocation or memory release is established in the wrapper. Passing an address does not establish ownership transfer. Dependency allocation, release, retention of addresses, mutation of globals, or other effects require separate contracts.

The wrapper does not prove that dependencies preserve context, cursor storage, global state, or memory reachable through their arguments. Subsequent wrapper accesses use the state that exists when those accesses occur.

No rollback or explicit error-reporting branch is visible. If a dependency does not return normally or a memory access fails, prior writes and dependency effects have already occurred. The supplied evidence does not establish exception, signal, or fault semantics.

Termination is conditional. The iterative path can continue indefinitely if condition calls never return exactly one.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-50be1c0375de | Called unconditionally after initialization for the first eligibility comparison. Called for the second eligibility comparison only if the first returns nonzero. Also called for every reached iterative condition. Eligibility calls receive addresses of internal comparison cells as their second arguments and use nonzero predicates. Iterative calls receive the original boundary argument directly and terminate only on exactly one. |
| F-c0d1b13ad3d8 | Called only after bulk eligibility succeeds, with context and the word read from selected-object offset +4. Its direct return value is ignored. Subsequent object selections and field reads occur after its effects. |
| F-ee5c701f2f6f | Called once per entered iterative body, after the cursor word is captured and before that captured word replaces the low-result cell. Receives the cursor argument; its direct return value is ignored. |
| F-e329dab7fb6c | Called once per entered iterative body with writable addresses of both result cells and the context address. Its direct return value is ignored. The high-result cell is copied to the output afterward. |

These interactions do not establish comparison purity, cursor advancement, structural removal, memory ownership, or release behavior. Unknown callees must not be assumed to preserve globals or unrelated reachable state.

## Acceptance examples

These are conditional consequences of the static representation, not executed tests. Unless an example states otherwise, assume valid, nonaliasing storage and no dependency effects beyond those specified.

### First comparison rejects bulk eligibility

Suppose the first F-50be1c0375de call returns zero and the first iterative condition returns one.

The initial output write occurs before either call. The second eligibility comparison does not occur, and context +20 is not read for eligibility. Exactly two comparison calls occur. No iterative body runs.

The output contains the initial context +16 word. The return’s high word is the context address; its low word is the output address.

### Second comparison rejects bulk eligibility

Suppose the first comparison returns two and the second returns zero.

The second comparison occurs because two is nonzero. Context +20 is not read for eligibility. No bulk operation occurs; execution reaches the iterative condition.

### Bulk reset with a stable selected object

Suppose both eligibility comparisons return nonzero, context +20 is nonzero, and context +16 remains the address **S** throughout the bulk operation.

F-c0d1b13ad3d8 receives context and the word from **S** +4. After it returns, the wrapper leaves **S** +8 and **S** +12 equal to **S**, **S** +4 equal to zero, and context +20 equal to zero.

The final output and return high word are **S**. The return low word is the output address. No iterative condition or body call occurs.

### Bulk dependency replaces the selected object

Suppose bulk eligibility succeeds, context +16 initially contains **S**, and F-c0d1b13ad3d8 replaces that field with **T**, which remains stable afterward.

The dependency receives the +4 word read from **S**. The subsequent reset accesses select **T**. The final output and return high word are **T**, not **S**.

### Zero context field

Suppose both eligibility comparisons return nonzero but context +20 is zero.

The field is read, but F-c0d1b13ad3d8 and the bulk reset writes do not occur. Execution proceeds to the iterative condition.

### Noncanonical iterative comparison result

Suppose an iterative condition returns two.

The body executes. Two would satisfy an eligibility comparison’s nonzero requirement, but it does not terminate the loop.

### Capture before cursor mutation

Suppose the cursor’s pointed-to word is **X**, and F-ee5c701f2f6f changes it to **Y**.

The wrapper places **X** in the low-result cell before calling F-e329dab7fb6c. If that dependency replaces the high-result cell with **H** and the low-result cell with **L**, the body writes **H** to the output. If the next condition returns one without further relevant effects, the packed return contains **H** above **L**.

### Output aliases a later initialization read

Suppose the output address equals **S** +8, context +16 contains **S**, and these locations are otherwise disjoint.

The initial output write stores **S** at **S** +8. The subsequent initialization read of selected-object +8 therefore obtains **S**, not the previous contents of that field.

## Uncertainties

The following remain **documented_only**:

- The abstract structure and semantic meanings of its fields.
- Dependency implementations, error conventions, ownership rules, and global effects.
- Whether comparison results are conventionally restricted to zero and one.
- Calling-convention details beyond the represented widths and packed return.
- Required alignment and supported aliasing arrangements.
- Address-overflow, invalid-access, and abnormal-termination behavior.
- Concurrency, atomicity, volatile-access semantics, and synchronization requirements.
- Whether dependencies retain internal-cell addresses. No lifetime extension or ownership transfer for those cells is established.
- Any finer ordering of multiple reads within a single bulk assignment.

A sentinel reset and an advance-then-remove sequence are plausible **inferred intent**. Neither interpretation is an acceptance requirement.

## Evidence

Established claims derive from the supplied static function representation and opaque dependency mapping: argument uses, access widths, field offsets, branch predicates, call placement, ordered writes, and 64-bit concatenation.

The unconditional first comparison and conditional second comparison follow from their respective positions outside and inside the first nonzero-result branch. The context +20 read is separately guarded by the second comparison’s nonzero result.

No dependency bodies, caller contracts, ABI documentation, or runtime traces were supplied. No runtime tests have been performed. Acceptance examples are conditional static deductions, not observed executions.
