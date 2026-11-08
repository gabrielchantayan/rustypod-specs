<!-- Trusted provenance: F-b2a2aac9c43d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b2a2aac9c43d

## Purpose

F-b2a2aac9c43d prepares eight candidate entries, considers eligible candidates in ascending index order, and stops attempting candidates when an application dependency returns zero. It then performs a cleanup pass over all eight candidate-offset entries.

On normal completion, its four-byte result is:

- **0:** an application returned zero.
- **2:** no application returned zero.

Preparation status, construction results, and application results influence processing differently. Their individual error values are not returned.

“Candidate,” “construction,” “application,” and “cleanup” describe call-site roles. They do not establish rule semantics, object identity, allocation, ownership, or actual resource release.

## Interface

### Arguments and widths

| Argument | Established use |
|---|---|
| First | Pointer whose first four-byte word is read before any dependency call. The pointer is subsequently passed to preparation, construction, and application. |
| Second | Opaque four-byte value passed to preparation. |
| Third | Opaque four-byte value passed to application. |
| Fourth | Opaque four-byte value passed to application. |
| Fifth | Address-bearing value used to read a four-byte field at byte offset 12. |

Candidate statuses and candidate offsets have signed 32-bit representations. Candidate application values and the construction output slot each occupy four bytes. The third and fourth arguments are forwarded without visible interpretation or indirect loading.

The representation is consistent with a 32-bit address model, but the precise ABI and endianness are not independently established.

### Storage requirements

For the visible accesses to be defined:

- The first argument must support the initial four-byte read.
- The fifth argument must support each required four-byte read at byte offset 12.
- The configuration reference obtained after preparation must support signed 32-bit reads at byte offset 52 whenever a diagnostic decision occurs.
- Dependency-written local storage must contain determinate values whenever those values are subsequently read.
- Values passed to dependencies must satisfy those dependencies’ requirements, which are not supplied here.

There are no visible null checks, bounds checks, or recoverable invalid-address paths.

### Candidate arrays

Preparation receives three separate local arrays, each containing eight four-byte entries:

1. Candidate offsets.
2. Candidate-specific application values.
3. Candidate statuses.

The function does not visibly initialize these arrays before preparation.

Every status entry reached by candidate traversal must be determinate. Every offset entry must be determinate when examined during final cleanup, even if early success prevented its candidate from being attempted. An application-value entry must be determinate whenever its candidate reaches application.

Statuses are tested only for zero versus nonzero. Offsets are likewise tested for zero versus nonzero; nonzero values are not range-checked.

### Construction storage and output-slot lifetime

Construction receives a pointer to input storage and the address of a separate four-byte output slot.

The output slot is shared across construction attempts within this invocation. It is not visibly initialized before the first attempt or reset between attempts.

A zero construction result does **not** establish that the current construction freshly wrote the slot. A prior construction, including a failed construction, may already have populated it. Defined execution requires a determinate value at each subsequent read, not a fresh write by every successful construction.

Construction receives the slot’s address and may retain it. Nothing in this function establishes that later dependencies preserve its contents.

### Return

The return occupies four bytes and is either 0 or 2 on normal completion. No candidate index, output-slot value, individual dependency error, or cleanup result is returned.

## Behavior

### Initial capture and preparation

Before any dependency call, the function reads and saves the first four-byte word addressed by the first argument. That captured value is used for every final cleanup call. Later changes to the caller’s first word do not replace the captured value.

The retained attempt result begins at 2.

The function reads the fifth argument’s field at byte offset 12 and calls F-2d50ac8b0359 with:

1. The first argument’s pointer.
2. The second argument.
3. The field value just read.
4. The offset-array address.
5. The application-value-array address.
6. The status-array address.

No preparation return value is used.

After preparation returns, the function obtains a global configuration reference. Subsequent diagnostic decisions use that retained reference rather than visibly reloading the global reference. The signed 32-bit field at byte offset 52 is nevertheless read separately at each diagnostic decision.

Changing the global reference alone therefore does not redirect later checks. Changing the field in the retained configuration can affect later checks. The referenced storage must remain accessible when read.

### Candidate selection and construction input

Candidates are considered in index order, from 0 through 7, unless an application returns zero.

A nonzero status makes a candidate ineligible:

- No construction or application is performed for that candidate.
- A diagnostic decision is made.
- The retained attempt result is unchanged.

A zero status makes a candidate eligible. For such a candidate, the function writes these fields in construction-input storage:

| Relative byte offset | Explicitly written value |
|---|---|
| 0 | Four-byte value 4 |
| 12 | The candidate offset, unless it is zero; otherwise a newly read fallback value |

A nonzero candidate offset is copied without arithmetic adjustment. Negative nonzero offsets receive the same treatment as positive nonzero offsets.

If the candidate offset is zero, the fifth argument’s field at byte offset 12 is read again. This is a fresh read for each eligible zero-offset candidate. It can differ from the field value supplied to preparation or from a fallback used by an earlier candidate.

Selecting a fallback does not visibly write that fallback into the offset array.

The two intervening four-byte positions at relative offsets 4 and 8 are not visibly initialized. Their interpretation and required contents are unknown. The supplied representation does not independently establish a complete formal descriptor type.

After writing the two fields, the function makes an attempt-diagnostic decision, then invokes construction. These are explicit writes before the diagnostic call, not a guarantee that retained pointers cannot be used by a dependency to alter the storage before construction consumes it.

### Construction result and output-slot reads

F-352a6facb968 receives:

1. The first argument’s pointer.
2. The construction-input address.
3. The output-slot address.

Its integer result replaces the retained attempt result.

A nonzero construction result:

- Prevents application for this candidate.
- Does not cause an invocation of F-22f99af8431f.
- Leads to a construction-failure diagnostic decision.
- Allows traversal to continue.

The output slot is neither read for application nor cleared by this function on that failure path. A value written during failed construction can therefore remain available to later attempts.

A zero construction result leads to application. At that point, the function reads the output slot for the application argument. Defined execution requires this read to obtain a determinate value. The visible body does not prove when that value was written, whether it is newly produced, or whether it denotes a valid object.

### Application and the following release-dependency call

After zero-result construction, F-b829b770e451 receives:

1. The first argument’s pointer.
2. The output-slot value read for this call.
3. The current candidate’s application value.
4. The third argument.
5. The fourth argument.

The application result replaces the retained attempt result.

After application returns normally, the function **reads the output slot again** and passes that newly read value to F-22f99af8431f. There is exactly one such invocation on this application path, whether application returned zero or nonzero.

The two output-slot reads are distinct. Construction may have retained the slot’s address, and application may modify the slot through such an alias. Consequently:

- Application and F-22f99af8431f need not receive equal values.
- The later call is not established to receive the object or value used by application.
- The later read independently requires determinate contents.
- Invoking F-22f99af8431f does not, by itself, establish that any resource was actually released.

The result of F-22f99af8431f is not used to select the public result.

Only after that call returns does the function make the application-result diagnostic decision. If the application result was zero, candidate traversal then stops. Otherwise, traversal continues.

Thus a zero application result does not bypass the later slot read, F-22f99af8431f, or the result-diagnostic decision.

### Diagnostic conditions and ordering

Every diagnostic decision reads the retained configuration’s signed 32-bit field at byte offset 52 and compares it with 2.

- Values greater than 2 enable that diagnostic call.
- Values of 2 or less, including negative values, suppress it.

The distinct diagnostic sites are:

| Situation | Placement and explicit information |
|---|---|
| Nonzero candidate status | Before advancing past the skipped candidate |
| Eligible candidate | After construction-input writes and before construction; explicitly supplies the candidate index |
| Nonzero construction result | After construction and before advancing |
| Application result | After F-22f99af8431f; supplies an indication selected according to application success or failure |

Each enabled diagnostic invokes F-4457a9a44a68.

Dependencies are not assumed to preserve shared state. Preparation can affect which configuration reference is obtained; later dependencies can affect the field read at later diagnostic decisions. Dependencies holding retained pointers may also affect exposed local storage.

Exact diagnostic rendering is not established. Some apparent formatting requirements are not matched by explicit arguments at the visible call sites. No contract is asserted for complete text, formatting safety, destination, or diagnostic side effects.

### Final cleanup and result normalization

After traversal finishes or stops on zero application result, the function examines all eight offset entries in ascending index order.

For each entry:

- Zero causes no cleanup call.
- Nonzero causes a call to F-bf7242c98d06 with the captured original first word.
- After that call returns, the corresponding offset entry is written to zero.

The offset’s numeric value is not passed to this dependency. Every cleanup invocation receives the same captured first-word value.

Cleanup eligibility depends on the entry’s value when visited, not on its preparation-time value or candidate status. Dependencies may have retained the array address and changed entries before those reads.

Duplicate nonzero offsets cause separate calls. Entries for skipped or unattempted candidates still participate. A zero entry used with a nonzero construction fallback does not thereby become cleanup-eligible.

After cleanup, a retained attempt result of zero produces public result 0; every other result produces 2. Because the initial result is 2, an all-skipped candidate set returns 2. A zero construction result alone cannot establish public success because application immediately replaces that result.

If a dependency faults, does not terminate, or exits nonlocally, the supplied body establishes no completed cleanup or normal-return guarantee.

## Side effects

### Explicit storage changes

The function visibly writes local control state, the two described construction-input fields, and each nonzero cleanup entry after its corresponding cleanup call returns.

Preparation can write the candidate arrays. Construction can write the output slot through its supplied address. Dependencies may retain supplied addresses; the function provides no general preservation or non-aliasing guarantee.

No direct write through the first or fifth argument is visible. Indirect writes by dependencies are unresolved.

### Resource and lifetime limits

The established lifecycle is a call sequence, not an ownership proof:

- Every normally returning application is followed by one invocation of F-22f99af8431f using the slot value read afterward.
- Failed construction does not trigger that invocation.
- The slot is not cleared between attempts.
- The final cleanup calls are separate and use the captured original first word.

Whether these calls allocate, retain, transfer, decrement references, destroy objects, or release memory is unknown. In particular, no same-object relationship is established between application and the following F-22f99af8431f invocation.

## Dependencies

| Opaque function ID | Established role |
|---|---|
| F-2d50ac8b0359 | Receives preparation inputs and three eight-entry array addresses. Its return is unused. |
| F-352a6facb968 | Receives construction input and a shared four-byte output-slot address. Zero permits application. |
| F-b829b770e451 | Receives the output-slot value read before application, candidate-specific data, and the third and fourth arguments. Zero stops later candidate attempts. |
| F-22f99af8431f | Receives the output-slot value read after application returns. Its result is unused. |
| F-bf7242c98d06 | Receives the captured original first word once per nonzero offset encountered during final cleanup. |
| F-4457a9a44a68 | Receives diagnostic requests when the relevant verbosity read exceeds 2. |

No dependency’s implementation, ownership rules, pointer-retention policy, global-preservation behavior, or exceptional behavior is established here.

## Acceptance examples

These are conditional expectations from static evidence, not executed tests. Assume accessible storage, determinate reads, and normally returning dependencies. Where particular prepared values matter, assume dependencies leave them unchanged until the described reads.

### All candidates are ineligible

If all eight statuses are nonzero:

- No construction, application, or F-22f99af8431f call occurs.
- Each candidate has a diagnostic decision.
- Cleanup examines all eight offsets.
- The return is 2.

### First application succeeds

If candidate 0 is eligible, its construction returns zero, and its application returns zero:

- Application receives the slot value read before its call.
- After application returns, F-22f99af8431f receives the slot value read then.
- Those values are equal only if the slot contents remain equal.
- The result-diagnostic decision precedes stopping traversal.
- Candidates 1 through 7 are not attempted, but their offsets are still examined during cleanup.
- The return is 0.

### A failed construction supplies a later usable slot value

Suppose candidate 0’s construction writes a determinate value into the slot but returns nonzero. Candidate 1’s construction returns zero without writing the slot.

If the slot remains unchanged and the value satisfies the application dependency’s requirements:

- Candidate 0 causes no application or F-22f99af8431f call.
- Candidate 1’s application receives the value left by candidate 0’s construction.
- A fresh write by candidate 1’s successful construction is not required by the visible control flow.

### Application changes the output slot

Suppose construction writes value A and retains the output-slot address. Application receives A, then uses an available alias to write value B into that slot.

If the write is effective before application returns:

- F-22f99af8431f receives B.
- There is one invocation of that dependency.
- No conclusion follows that A was released.
- Candidate continuation depends on the application result, not on A or B.

### Uninitialized slot after successful construction

Suppose the first attempted construction returns zero without writing the slot, and no earlier action supplied determinate contents.

The ensuing application-argument read lacks the required determinate value. This contract assigns no defined application argument or normal public result to that execution.

### Application failure followed by skipped candidates

If an application returns nonzero and all later candidates are ineligible:

- The post-application slot read and F-22f99af8431f call still occur.
- Later skipped candidates do not overwrite the retained failure.
- Cleanup occurs.
- The return is 2.

### Fresh fallback and independent cleanup

Suppose preparation receives fallback value 100 and changes the fifth argument’s offset-12 field to 200. An eligible candidate with zero offset uses 200 unless another dependency changes the field before its fallback read.

That fallback selection does not change its offset-array entry. If only entries 0, 3, and 7 remain nonzero at cleanup, cleanup makes three calls, each using the captured original first word, even after early application success.

An offset of −1 is nonzero: it is used directly for construction input and qualifies for cleanup if still nonzero when visited.

## Uncertainties

The following remain **documented_only**:

- Semantic meaning of offsets, statuses, application values, and output-slot values.
- Complete construction-input layout and requirements for unwritten fields.
- Whether preparation supplies every subsequently read array entry.
- Which construction attempts write the output slot.
- Whether retained aliases modify that slot between its two reads.
- Validity and ownership requirements for values passed to application and F-22f99af8431f.
- Actual resource effects of all dependencies.
- Retention and lifetime handling of pointers to local storage.
- Dependency mutations of caller storage, candidate arrays, construction storage, or shared state.
- Invalid-address, indeterminate-value, fault, nontermination, and nonlocal-exit behavior.
- Exact diagnostics, ABI, and endianness.

The visible loop indices remain within 0 through 8, so their increments do not approach signed 32-bit overflow. Candidate offsets are selected and forwarded, not arithmetically adjusted. Valid address formation for the fixed-offset reads remains a storage precondition.

## Evidence

**Established static evidence:** The supplied body shows eight-entry candidate arrays, ascending traversal, zero-status eligibility, per-candidate zero-offset fallback reads, separate construction and application results, an uninitialized and unreset shared output slot, distinct slot reads before application and after its return, all-entry final cleanup, and return normalization to 0 or 2.

**Inferred intent:** The call structure resembles ordered candidate selection with temporary construction and later cleanup. These labels do not establish object identity or memory-management semantics.

**Verification limit:** This contract is based solely on the supplied static representation. No implementation access, dependency inspection, or runtime tests have been performed. Acceptance examples are conditional specifications, not observed execution results.
