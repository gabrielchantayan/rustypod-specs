<!-- Trusted provenance: F-efabac4fa79c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-efabac4fa79c

## Purpose

F-efabac4fa79c conditionally constructs and publishes results in two slots of a caller-supplied object. It considers the higher-offset slot first, then the lower-offset slot. Each slot has a completion byte, a result field, and an embedded context address available to one construction route.

The visible operations are not transactional: each slot has its own lookup and publication paths, and there is no explicit rollback. This does **not** establish that a published slot remains unchanged during subsequent dependency calls. Dependencies may modify either slot or other state; their preservation behavior is unknown.

“Lookup,” “allocation,” “construction,” and “completion” describe inferred roles. Established behavior consists of field accesses, selector comparisons, opaque dependency calls, zero tests, and ordered publication writes.

## Interface

### Arguments and result

The displayed interface has four arguments:

| Argument | Width | Established use |
|---|---:|---|
| Object base | 32 bits | Addresses object fields and embedded contexts. |
| Second argument | 32 bits | No explicit use in the supplied representation. |
| Third argument | 32 bits | No explicit use in the supplied representation. |
| Fourth argument | 32 bits | Appears among the displayed arguments to both lower-slot lookup forms. Its actual ABI role is unresolved. |

There is no explicit return value. Observable outputs are dependency calls and direct memory writes.

The object must support every access reached on the selected execution path. Supporting all paths requires field storage through byte offset `0x22B`, inclusive. The total object size and embedded-context extents are not established.

### Object layout

Offsets are byte offsets from the object base.

| Offset | Access width | Established role |
|---|---:|---|
| `0x38` | 4 bytes | Shared lookup input. |
| `0x40` | 4 bytes | Selector used for dispatch. |
| `0x44` | 4 bytes | Additional lower-slot lookup input. |
| `0xA1` | 1 byte | Lower completion flag. |
| `0xA4` | Address formation only | Lower embedded context passed to contextual construction. |
| `0x15C` | 4 bytes | Lower result destination. |
| `0x164` | 1 byte | Higher completion flag. |
| `0x168` | Address formation only | Higher embedded context passed to contextual construction. |
| `0x220` | 4 bytes | Higher result destination. |
| `0x228` | 4 bytes | Higher eligibility value and contextual lookup input. |

A four-byte field is not thereby established to be a pointer, signed quantity, enumeration, or owned resource.

Completion flags are tested for zero versus nonzero. Any nonzero byte suppresses that slot’s gated processing. Publication writes exactly the byte value one.

### Selector storage and saved values

Four four-byte values are read from static storage. This contract names their storage roles:

- **H-primary:** higher primary selector storage.
- **H-alternate:** higher alternate selector storage.
- **L-primary:** lower primary selector storage.
- **L-alternate:** lower alternate selector storage.

These names do not denote immutable constants. Numeric values, equality relationships, and mutability are unknown.

Two saved values must be distinguished from later reads:

- The **saved higher-primary value** is captured before the initial read of `0x228`.
- The **saved lower-primary value** is captured after higher processing and before the lower completion check.

Primary comparisons read their respective global storage again. Contextual lookup arguments use the saved values, not those later comparison reads.

### Preconditions and arithmetic

The representation uses 32-bit address-like values and fixed byte-offset additions. The only visible arithmetic is address formation; `0x44` supplied to the allocation-role dependency is the literal decimal value 68, not the object field at offset `0x44`.

There are no visible bounds, null, alignment, lifetime, or overflow checks. Valid object storage and valid dependency arguments are preconditions for meaningful execution. Behavior for address wraparound, invalid storage, or violated dependency preconditions is documented_only; this contract does not define modular-address recovery behavior.

All visible value tests are equality or inequality tests. No signed ordering is used.

## Behavior

### Higher-slot entry and eligibility

The first relevant read captures H-primary into the saved higher-primary value. The function then reads the four-byte field at `0x228`.

If that field is zero, higher processing is skipped. The higher completion byte is not read by this eligibility condition.

If the field is nonzero, the function reads `0x164`. A nonzero completion byte skips higher processing; zero permits dispatch.

Every normally returning higher path proceeds to lower processing. Neither ineligibility nor lookup failure itself ends the function.

### Higher primary dispatch

For an eligible higher slot, the selector at `0x40` is compared with a **new read of H-primary**. The comparison does not use the saved higher-primary value.

If the comparison matches, the function reads `0x38` and invokes F-5710a8628cba with three displayed inputs:

- The shared input read from `0x38`.
- The saved higher-primary value.
- The initial higher input read from `0x228`.

Consequently, the value establishing a primary selector match and the selector-like value supplied to this lookup need not be equal if the two H-primary reads differ.

A zero lookup result produces no direct higher publication. A nonzero result selects higher contextual construction.

The primary lookup does not reload `0x228`.

### Higher alternate dispatch and fallback

If the primary comparison fails, the function reads `0x40` again and compares it with a read of H-alternate. The selector is therefore not a single shared snapshot across both comparisons.

If this comparison also fails, higher processing ends without lookup or direct higher publication.

If it matches, F-5710a8628cba is invoked with only one explicitly displayed argument: the shared input read from `0x38`. The representation does not establish that this call has a reliable one-argument source-level prototype.

A nonzero result selects higher direct construction.

A zero result selects contextual fallback. After the alternate lookup returns, the function reads `0x38` again, then reads `0x228` again. F-5710a8628cba receives those refreshed field values together with the **original saved higher-primary value**.

This fallback does not reload H-primary. In particular, mutation of H-primary by the alternate lookup does not change the saved value supplied to fallback.

The fallback also does not recheck the higher completion byte, selector, or initial nonzero eligibility condition. A newly read zero at `0x228` is still supplied to fallback. A zero fallback result ends higher processing without direct publication; a nonzero result selects contextual construction.

### Higher construction and publication

Each successful higher lookup route requests 68 bytes through F-8157a180d901. Its returned four-byte value is passed to a construction dependency without a visible zero check.

The selected construction form is:

| Successful lookup route | Construction dependency and displayed inputs |
|---|---|
| Alternate lookup | F-1ade66241a82 receives the allocation-call result and the successful lookup result. |
| Primary or fallback lookup | F-f8beff1830ad receives the allocation-call result, object base plus `0x168`, and the successful lookup result. |

After construction returns, the function writes its four-byte return value to `0x220`, then writes one to the byte at `0x164`.

Construction return values are not checked. Zero is published just like any other four-byte value.

This order describes the direct writes. It establishes neither atomic publication nor visibility guarantees for concurrent observers.

### Lower entry and selector dispatch

After all normally completed higher processing, the function captures L-primary into the saved lower-primary value. Only then does it read the lower completion byte at `0xA1`.

A nonzero lower completion byte causes immediate return. The L-primary snapshot has nevertheless already occurred.

With a zero lower completion byte, the function compares `0x40` with a **new read of L-primary**, not the saved lower-primary value.

A match selects lower contextual lookup. A mismatch leads to another read of `0x40`, compared with a read of L-alternate. If this second comparison also fails, the function returns.

Higher dependencies may have changed the selector, lower fields, or selector storage before these lower reads. Entry-time values are not assumed to survive higher processing.

### Lower alternate lookup

An alternate match invokes F-5710a8628cba with ten displayed operands:

- The four-byte shared input from `0x38`.
- A **separate new read of L-alternate**.
- The four-byte lower input from `0x44`.
- The fourth function argument.
- Six additional machine-state-derived operands.

Thus the L-alternate read used as a lookup argument is distinct from the L-alternate read used for comparison. The contract does not equate their values.

The representation does not establish a total order among reads appearing within this call’s argument list. It does establish that this lookup follows the alternate comparison.

A nonzero alternate lookup result leads to a 68-byte request, direct construction through F-1ade66241a82, and lower publication.

A zero result falls through to lower contextual lookup. The function does not recheck the lower completion byte or either selector comparison.

### Lower contextual lookup

This lookup is reached directly after a primary match or following a zero alternate lookup result.

F-5710a8628cba receives:

- A read of the shared input at `0x38`.
- The saved lower-primary value.
- A read of the lower input at `0x44`.
- The fourth function argument.
- The same category of six unresolved machine-state-derived operands.

After alternate failure, the object-field reads occur after that dependency returns and can observe its changes. The selector-like argument remains the saved lower-primary value captured before the completion check. It is not refreshed from L-primary, even if the alternate lookup modifies that storage.

A zero contextual lookup result causes return without direct lower publication.

A nonzero result causes a 68-byte request to F-8157a180d901, followed by F-f8beff1830ad with the allocation-call result, object base plus `0xA4`, and the successful lookup result.

### Lower publication

Both successful lower construction routes write the four-byte construction return value to `0x15C`, then write one to `0xA1`, then return.

Neither the allocation-call result nor the construction return is validated before use or publication.

## Side effects

### Direct writes and skipped paths

The only direct object writes shown are:

- Higher result at `0x220`, followed by higher completion at `0x164`.
- Lower result at `0x15C`, followed by lower completion at `0xA1`.

Existing result values are overwritten without being read or explicitly released. A zero completion byte does not establish that its result field is empty.

Skipped paths and zero-result lookup paths perform no direct publication for the affected slot. Dependencies may still modify those same fields.

After higher publication, the function performs no subsequent **direct** higher result or completion write and has no visible rollback operation. Lower dependencies may nevertheless change or undo higher publication through their own effects. No final preservation guarantee for either slot follows from the visible write sequence.

### Ownership and dependency effects

There can be at most four lookup calls, two 68-byte requests, and two construction calls during one normally returning invocation.

No explicit release, rollback, exception handler, or error-reporting operation is shown. The following are not established:

- Whether F-8157a180d901 returns fresh storage or how failure is represented.
- Whether construction retains, consumes, replaces, or releases its inputs.
- Whether a published result aliases the allocation-call result.
- Whether lookup results are borrowed or owned.
- Whether dependencies modify embedded contexts, object fields, selector storage, or unrelated state.
- Whether overwritten values require cleanup or receive cleanup inside dependencies.

If a dependency does not return normally, subsequent calls and writes on that path are not established. No recovery path is visible.

## Dependencies

| Opaque function ID | Established call-site behavior |
|---|---|
| F-5710a8628cba | Returns a four-byte value tested for zero. Appears with one, three, and ten displayed operands. |
| F-8157a180d901 | Receives the literal four-byte value 68; its result becomes a construction input. |
| F-1ade66241a82 | Receives the allocation-call result and a nonzero alternate lookup result; its return is published. |
| F-f8beff1830ad | Receives the allocation-call result, an embedded-context address, and a nonzero contextual lookup result; its return is published. |

The differing lookup call shapes do not establish optional arguments, a common source-level prototype, or meanings for omitted and machine-state-derived operands. Dependency internals and preservation guarantees remain unknown.

## Acceptance examples

These are conditional static scenarios, not executed tests. Unless a scenario states otherwise, assume valid storage, normally returning dependencies, stable reads, and no dependency mutation.

### Higher primary success

With a nonzero higher input, zero higher completion, and a matching primary comparison:

- Lookup receives the saved higher-primary value and the initially read higher input.
- A nonzero result selects contextual construction using object base plus `0x168`.
- Construction return is written to `0x220` before one is written to `0x164`.
- Lower processing follows.

### Different higher-primary observations

Suppose the initial H-primary read yields value A, but the later comparison reads value B, and the selector matches B.

The primary lookup receives A, not B. This scenario specifies observed read values; it does not establish a supported concurrency mechanism or explain how storage changed.

### Higher fallback after dependency mutation

Suppose higher alternate dispatch succeeds, but its lookup returns zero after changing H-primary, the shared input, and `0x228`.

Fallback uses the changed shared and higher inputs, including a zero higher input if that is now stored. Its selector-like argument remains the original saved higher-primary value. It does not recheck completion or eligibility.

### Lower alternate argument reread

Suppose the lower primary comparison fails, the alternate comparison reads value B and matches, but the later L-alternate argument read yields value C.

The alternate lookup receives C, not the comparison’s B. Equality between these reads is not a contract requirement.

### Lower fallback preserves the saved primary argument

Suppose L-primary is initially saved as A. Lower alternate lookup returns zero after changing L-primary and both lower lookup input fields.

Contextual fallback uses the refreshed object-field inputs but still passes A as its selector-like argument. It does not reread L-primary or recheck lower completion.

### Zero allocation or construction result

If a successful lookup is followed by an allocation-call result of zero, construction is still invoked with zero as its first displayed input.

If construction returns zero, zero is written to the selected result field and its completion byte is set to one. Publication is not conditional on a nonzero constructed result.

### Completion and selector overlap

Any nonzero lower completion byte causes return after the L-primary snapshot, without lower selector comparisons.

With stable equal primary and alternate selector values, a matching selector takes the primary route because that comparison has precedence. This applies independently to both slots.

### Lower effects after higher publication

If higher publication occurs and a lower lookup later returns zero, the function performs no direct higher rollback. If that lower lookup also changes the higher result or completion byte, this contract does not require restoration of the earlier publication.

## Uncertainties

The following remain **documented_only**:

- Selector values, equality relationships, mutability, and changes between distinct reads.
- Exact semantic field types and object invariants.
- Reliable dependency prototypes and calling conventions.
- Meaning of the fourth argument and additional displayed lookup operands.
- Operand evaluation order within expressions where the representation does not establish it.
- Allocation guarantees, failure handling, ownership, and cleanup responsibilities.
- Dependency preservation of fields, globals, and other state.
- Embedded-context extents and validity requirements.
- Concurrency, reentrancy, atomicity, and memory visibility.
- Behavior for invalid addresses, misalignment, address overflow, or dependency-level errors.

Snapshot-versus-reread distinctions describe the supplied representation. They are not a promise that unsynchronized concurrent modification has defined runtime behavior.

## Evidence

This contract is based solely on the supplied static representation and opaque dependency mapping.

Established static evidence includes access widths and offsets; initial selector snapshots and subsequent global rereads; branch precedence; short-circuit eligibility; field rereads after alternate lookup failure; literal 68-byte requests; construction selection; and result-before-completion write order.

Inferred intent is limited to the role descriptions “lookup,” “allocation,” “construction,” and “completion.” Dependency internals, ownership, reliable ABI details, and runtime guarantees are not established.

No runtime tests have been performed. Acceptance examples express conditional consequences of the static evidence, not observed execution results.
