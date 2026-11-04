<!-- Trusted provenance: F-67fb1a8bbd79; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-67fb1a8bbd79

## Purpose

F-67fb1a8bbd79 combines two dependency-adjusted input values with a shared enable value, publishes a normalized 32-bit status, and passes the complementary status to another dependency. It then reloads the published storage and, when an optional callback is present, delivers that reloaded value through a writable temporary payload.

The established behavior is a state update followed by a conditional notification. The domain meaning of the inputs, shared enable value, status, and notification selector is unknown. Descriptions such as “enabled,” “active,” or “notification” are interpretive labels, not established application semantics.

A significant distinction is that the optional callback receives a value read **after** the complementary-status dependency returns. It does not necessarily receive the normalized value originally written by this function.

## Interface

### Explicit inputs

The supplied representation has four arguments:

| Position | Represented width and kind | Direct use |
|---|---|---|
| First | 32-bit opaque value | No direct use in the visible body. |
| Second | 32-bit unsigned value | Initially copied into the temporary cell later used as callback payload. That cell is explicitly overwritten before the callback. |
| Third | 32-bit signed integer | Copied into a writable temporary cell whose address is passed to F-f23023f237bd. |
| Fourth | 32-bit signed integer | Copied into a writable temporary cell whose address is passed to F-ef97309f9a12. |

All bit patterns are accepted by the visible body. There is no visible range check or explicit rejection path.

The third and fourth values are ultimately tested only for equality or inequality with zero. Negative values therefore count as nonzero; their signs and magnitudes do not otherwise affect the visible decision.

The incoming second argument has no direct contribution to the final status or callback payload: its temporary copy is replaced by a later shared-storage read. This observation does not establish the absence of undocumented effects in callees.

### Output

There is no explicit return value. Observable results consist of:

- Calls to dependencies, in the order specified below.
- A 32-bit write to shared status storage.
- Possible further shared-state changes performed by dependencies.
- An optional indirect callback invocation with a writable 32-bit payload.

No explicit error value or success indicator is returned.

### Referenced storage and layout

The visible behavior uses three externally supplied references:

1. **Enable-storage reference:** points to a 32-bit word consulted only when at least one adjusted input is nonzero.
2. **Status-storage reference:** points to a writable 32-bit word. Its target is captured before the status decision and later used for the post-dependency read.
3. **Callback-container reference:** identifies storage containing a callable pointer at byte offset **4** from the container base.

These are descriptive role names, not identifiers from the source.

The callback slot has the represented target’s function-pointer type. The supplied material does not independently establish the complete container layout, its total size, or fields at other offsets. The offset is four **bytes**, not four elements.

Three distinct 32-bit temporary cells hold the second, third, and fourth argument copies. No relative placement or contiguity of those cells is required by this contract.

### Preconditions for defined completion

The following requirements apply when their respective operations are reached:

- Both initial dependencies must accept the writable temporary addresses supplied to them.
- The status destination used for publication must be valid for a 32-bit write.
- The captured status target must remain valid for a 32-bit read after F-fa1e7c5a0d15 returns.
- If either adjusted input is nonzero, the enable-storage reference must support a 32-bit read.
- The callback container must permit access to the function-pointer slot at byte offset 4, even when that slot contains a null pointer.
- A non-null callback must be callable with selector 15 and a pointer to a writable 32-bit temporary.

The visible body does not enforce these requirements or define recovery from their violation.

## Behavior

### 1. Prepare temporary argument storage

The function copies the second, third, and fourth arguments into separate 32-bit temporary cells.

The first argument is not directly read by the visible body.

### 2. Invoke the two adjustment dependencies

The first dependency call is to **F-ef97309f9a12**, with the address of the fourth-argument temporary.

After that call returns, the function invokes **F-f23023f237bd**, with the address of the third-argument temporary.

This order is observable and required. The calls are unconditional: neither depends on either incoming argument being nonzero.

Both addresses designate writable storage. The later decision uses the values present after these calls, not necessarily the original argument values. The dependencies may leave their supplied words unchanged or modify them; their actual contracts are unavailable.

The visible body does not use an explicit return value from either call.

### 3. Determine and publish normalized status

After both adjustment calls return, the function captures the target designated by the status-storage reference.

It then evaluates whether either adjusted temporary is nonzero.

- If both are zero, the enable word is not read, and the published status is zero.
- If either is nonzero, the function reads the 32-bit enable word. The published status is one when that word is nonzero and zero when it is zero.

The value written to shared status storage is therefore exactly **0 or 1**. It is not a bitwise combination of the adjusted inputs and enable word.

The enable word is treated as a zero/nonzero condition. Its value is not copied into the status destination. For example, an enable word of 2 produces status 1 when an adjusted input is nonzero.

The visible write uses the status-storage reference. Under ordinary stable-reference execution, this is the same target captured immediately before the decision. There is no intervening dependency call between capture and publication. No stronger synchronization guarantee is established for externally changing references.

### 4. Pass the complementary value to F-fa1e7c5a0d15

After the status write, the function calls **F-fa1e7c5a0d15** with one integer argument:

| Published status | Argument |
|---|---|
| 0 | 1 |
| 1 | 0 |

This complement is representable without overflow in a 32-bit integer. Although the supplied representation includes an additional range-related branch, the normalized status cannot exceed one, so that branch does not introduce another reachable argument value in the visible logic.

Publication precedes this call. Consequently, the dependency can potentially observe or alter the status storage, subject to its unknown implementation.

The function does not visibly consume a return value from this dependency.

### 5. Reload the captured status target

After F-fa1e7c5a0d15 returns, the function reads one 32-bit word through the status target captured before the decision.

That word replaces the second-argument temporary and becomes the callback payload.

This reload has two important consequences:

- The callback payload need not equal the normalized status originally published. If F-fa1e7c5a0d15 changes that storage, the reloaded word reflects the change.
- If F-fa1e7c5a0d15 redirects the shared status-storage reference, the reload still uses the previously captured target. It does not automatically follow the new reference.

The payload can therefore contain any 32-bit value present at the captured target, not merely zero or one.

### 6. Check and invoke the optional callback

Only after the payload reload does the function inspect the function-pointer slot at byte offset 4 in the callback container.

- If the slot is null, no callback is invoked.
- If it is non-null, the function invokes the callback with:
  - selector **15**;
  - the address of the 32-bit payload temporary.

The supplied representation checks the slot and then accesses the callback expression for invocation. It does not establish an immutable callback registration snapshot. Behavior under concurrent replacement of the container or slot is therefore not specified here.

The callback can synchronously read or modify the temporary payload. The visible body performs no subsequent read of that payload and does not explicitly copy callback changes back to shared status storage.

The function then returns without an explicit result.

## Side effects

### Effects directly established

The visible body:

- Initializes three temporary 32-bit cells.
- Exposes the fourth-argument temporary to F-ef97309f9a12.
- Exposes the third-argument temporary to F-f23023f237bd.
- Conditionally reads the shared enable word.
- Writes exactly one normalized 32-bit status value.
- Calls F-fa1e7c5a0d15 after that write.
- Reads the captured status target after that call.
- Reads the callback slot.
- Optionally exposes the payload temporary to the callback.

The enable word is not directly written. The callback slot is not directly written.

### Ownership and lifetime

No allocation, deallocation, ownership-transfer operation, or explicit resource release appears in the visible body.

The pointers passed to the initial dependencies and callback refer to invocation-local temporary storage. The visible function keeps those cells available during the synchronous calls. Their addresses do not establish a right to release the storage or use it after the invocation ends.

Whether any dependency retains pointers, changes global references, releases shared storage, or performs other resource operations is unknown. In particular, successful completion requires the captured status target to remain readable across F-fa1e7c5a0d15; preservation is a precondition, not an established dependency guarantee.

### Ordering and failures

The required visible ordering is:

1. Fourth-argument adjustment call.
2. Third-argument adjustment call.
3. Status-target capture and status decision.
4. Status write.
5. Complementary-status dependency call.
6. Captured-target reload.
7. Callback-slot check.
8. Optional callback invocation.

There is no visible rollback, retry, exception handler, or explicit failure branch. If a dependency does not return normally, subsequent steps are not established to occur.

No locks, atomic-access operations, memory barriers, or reentrancy guards are visible. The ordering above is not a claim of inter-thread synchronization.

## Dependencies

| Opaque function ID | Supplied input | Required position | Unknown behavior |
|---|---|---|---|
| F-ef97309f9a12 | Address of the writable 32-bit fourth-argument temporary | First dependency call | Resulting word, global changes, pointer retention, errors, and resource behavior |
| F-f23023f237bd | Address of the writable 32-bit third-argument temporary | After F-ef97309f9a12 returns | Resulting word, global changes, pointer retention, errors, and resource behavior |
| F-fa1e7c5a0d15 | Integer 0 or 1, complementary to the published status | After publication, before payload reload | Shared-storage changes, reference changes, callback-registration changes, errors, and resource behavior |

The indirect callback has no supplied opaque function ID. Its established call shape is selector 15 plus a writable 32-bit payload address. Its calling convention, return contract, application meaning, and broader side effects are undocumented.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, shared references remain stable, referenced storage is valid, and all invoked dependencies return normally.

### Both adjusted inputs are zero

If the two adjustment calls leave both relevant temporary words at zero:

- The enable word is not read.
- Shared status receives 0.
- F-fa1e7c5a0d15 receives 1.
- The captured status target is subsequently reloaded.
- Callback delivery depends only on the callback slot, not on the status being zero.

### Nonzero adjusted input with zero enable word

If either adjusted input is nonzero and the enable word is zero:

- The enable word is read.
- Shared status receives 0.
- F-fa1e7c5a0d15 receives 1.

A nonzero input alone is insufficient to publish status 1.

### Negative adjusted input with nonzero enable word

If one adjusted input is −1 and the enable word is 2:

- Both values satisfy the relevant nonzero tests.
- Shared status receives 1, not −1 or 2.
- F-fa1e7c5a0d15 receives 0.

### Adjustments override incoming values

If both incoming third and fourth arguments are zero, but an adjustment dependency changes its temporary to a nonzero value, the decision uses that changed value.

Conversely, nonzero incoming arguments do not force status 1 if both temporary values have become zero before the decision.

### Post-publication mutation changes callback payload

If status 1 is published and F-fa1e7c5a0d15 then writes 7 to the captured status target, a present callback receives selector 15 and a payload word containing 7.

The function does not renormalize the reloaded word.

### Redirecting the status reference does not redirect the reload

If the captured target is storage A, and F-fa1e7c5a0d15 redirects the shared reference to storage B while A remains valid, the callback payload is read from A.

### Null callback

If the callback slot is null, all preceding work still occurs, including the status publication, complementary dependency call, and captured-target reload.

### Callback edits its payload

If the callback replaces its payload word with another value, the visible body does not subsequently publish that replacement or return it to the caller.

## Uncertainties

The following remain **documented_only**:

- Application-level meaning of all arguments, shared words, and selector 15.
- The exact ABI, calling conventions, alignment requirements, byte order, and function-pointer representation.
- Contracts and side effects of all three identified dependencies.
- Initialization and lifetime management of shared references and callback storage.
- Whether shared references or the callback slot can change concurrently.
- Reentrancy behavior and any external serialization requirements.
- Whether callees retain supplied temporary addresses.
- Behavior after invalid memory access, an incompatible callback, or abnormal dependency termination.

An inferred intent is to coordinate a normalized status with a complementary action and then notify an observer. This intent does not override the established sequencing, post-call reload, or unknown dependency effects.

## Evidence

This contract is based solely on static inspection of the supplied function representation and the supplied opaque dependency mapping.

Established evidence covers argument-copy widths, dependency order, zero/nonzero decision logic, conditional enable access, normalized publication, complementary argument values, captured-target reload, callback-slot offset, selector 15, and the lack of an explicit returned result.

No dependency implementations, complete data layouts, external callers, or runtime observations were supplied. **No runtime tests have been performed.**
