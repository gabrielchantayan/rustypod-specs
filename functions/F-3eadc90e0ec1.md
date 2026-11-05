<!-- Trusted provenance: F-3eadc90e0ec1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3eadc90e0ec1

## Purpose

F-3eadc90e0ec1 performs a conditional sequence of calls for a nonzero object address. It checks two object fields in order, conditionally invokes F-2642535311f0 for each nonzero field, and then performs up to three indirect invocations: an optional initial invocation, a mandatory invocation, and an optional final invocation.

Two ordered reads of the global dispatch-base location have distinct roles. The first supplies a retained base for the mandatory and final target lookups. The second supplies the base for the initial optional target lookup. These reads must not be collapsed into a single snapshot in the behavioral contract.

The sequence is consistent with cleanup or release bracketed by notifications. That is **inferred intent**, not an established resource-management protocol. The supplied representation does not establish that any call frees memory, that the optional calls form a pair, or that their numeric arguments identify phases.

A zero object address returns immediately.

## Interface

### Inputs

The supplied representation exposes four explicit argument positions:

| Position | Width and established use |
|---|---|
| First | A 32-bit value compared with zero, otherwise used as the object base address. Passed to the initial optional target and the mandatory target. |
| Second | A 32-bit value with no explicit use in the visible body. |
| Third | A 32-bit value with no explicit use in the visible body. |
| Fourth | A 32-bit value passed in the fourth represented argument position of the initial optional invocation. |

The initial optional invocation has a fifth represented argument derived from incoming machine state outside these four explicit positions. Its origin, width beyond the represented 32-bit value, and semantic meaning are **documented_only** unknowns. It must not be treated as an initialized constant or as an object-derived value.

The absence of explicit uses for the second and third arguments does not establish whether machine state associated with them is observable by F-2642535311f0. That dependency’s actual calling interface is not supplied.

### Result and completion

No explicit result value or status code is represented.

Normal completion depends on every reached memory access being valid and every invoked target returning normally. There is no visible error-return path, exception handler, rollback, or recovery from an invalid target.

### Object layout

The body explicitly reads the following object-relative locations:

| Byte offset | Width | Role |
|---|---|---|
| +4 | 32 bits | A nonzero value triggers the first invocation of F-2642535311f0. |
| +8 | 32 bits | A nonzero value triggers the second invocation of F-2642535311f0. |

These are zero-versus-nonzero tests, not positivity tests. Every nonzero 32-bit pattern satisfies either condition. Neither field value is visibly dereferenced or explicitly passed as an argument.

No other object field is explicitly accessed by this body.

### Dispatch layout and base selection

A global location contains a dispatch-base address. The visible sequence reads that location twice:

| Global read | Use |
|---|---|
| First | Retained for the mandatory target lookup and the final optional target lookup. |
| Second | Used to locate the initial optional target. |

The relevant byte offsets within the selected structures are:

| Offset | Role |
|---|---|
| +0x18, decimal 24 | Mandatory indirect target, obtained through the retained first base. |
| +0x30, decimal 48 | Optional indirect target: initially obtained through the second global-read base, and finally obtained through the retained first base. |

The supplied representation indicates 32-bit integer and address values. Exact ABI rules, callable-pointer encoding, alignment requirements, and portable address-arithmetic semantics remain unresolved.

## Behavior

### 1. Zero-address branch

The first action is a comparison of the first argument with zero.

If it is zero, the function returns without explicitly reading either object field, reading the dispatch-base global, or invoking any target. The remaining inputs do not affect this visible branch.

### 2. First field-dependent invocation

For a nonzero object address, the function reads the 32-bit field at object offset +4.

If the value is zero, it proceeds without calling F-2642535311f0. Otherwise, it invokes F-2642535311f0 and proceeds only if that invocation returns.

No explicit arguments are shown for this dependency call. The contract therefore does not identify the object, the field address, or the field contents as its arguments.

### 3. Second field-dependent invocation

Next, the function reads the 32-bit field at object offset +8.

This read occurs after the first dependency invocation, if that invocation occurred. It is not part of an earlier two-field snapshot.

If the value read at +8 is nonzero, the function invokes F-2642535311f0 again. A zero value skips this invocation.

Consequently, a nonzero-object execution can explicitly invoke F-2642535311f0 zero, one, or two times, provided execution reaches the corresponding stages. Changes made by the first invocation can affect the second decision. In particular, the first invocation may change the field’s value or make its later access invalid; neither possibility is guarded against.

### 4. Two global reads and the initial optional invocation

After both field-dependent stages, the function performs these ordered reads:

1. It reads the global dispatch-base location and retains that first base.
2. It reads the same global location again.
3. Through the base obtained by the second read, it reads the target at byte offset +0x30.

There is no explicit callee invocation between the two global reads. Nevertheless, the representation contains two reads, not one reused value. If the global changes between them and the second read observes that change, the initial optional target comes from a different structure than the later targets.

If the selected initial target is zero, the initial invocation is skipped. Otherwise, the target is invoked with these represented argument positions:

| Position | Value |
|---|---|
| First | Original object address |
| Second | 32-bit zero |
| Third | Selected initial target itself |
| Fourth | Fourth explicit input |
| Fifth | Value derived from incoming machine state described in the interface |

The third position carries the callable value selected for this invocation. No meaning beyond that forwarding is established.

This invocation does not replace the retained first base with the second base. Once the first base has been retained, subsequent changes to the global alone do not redirect the later lookups.

### 5. Mandatory invocation through the retained base

After the initial optional invocation returns, or immediately if it was skipped, the function reads the target at byte offset +0x18 through the retained first base.

The target is then invoked with the original object address as its sole explicitly represented argument.

There is no null check for this target and no validity check for the retained base. A zero mandatory target is therefore an unchecked invocation attempt, not an instruction to skip the operation.

The target lookup occurs after the initial optional invocation. If that invocation changes the +0x18 entry in the retained structure, the later read can select the changed target. This remains true even when the initial optional target was obtained from a different structure through the second global read.

Conversely, changing only the global dispatch-base value during the initial invocation does not redirect the mandatory lookup. It still uses the first retained base.

### 6. Final optional invocation through the retained base

After the mandatory invocation returns, the function reads the target at byte offset +0x30 through the retained first base.

If this target is zero, the function returns. Otherwise, it invokes the selected target with two explicitly represented argument positions: zero, followed by one.

This is a new target read, not reuse of the initial target. Therefore:

- The final invocation may occur when the initial invocation was skipped.
- The final invocation may be skipped when the initial invocation occurred.
- The two optional invocations may select different targets.
- Different targets can arise from different initial bases, intervening entry changes, or both.

No pairing guarantee is established.

The supplied representation expresses the final transfer as an indirect invocation followed by return, but includes a recovery limitation concerning its mechanics. The contract establishes the selected target and represented arguments, not an exact call-versus-tail-transfer implementation.

## Side effects

### Direct effects

The visible body performs memory reads and transfers control to dependencies. It contains no explicit stores to the object fields, the global dispatch-base location, or the dispatch entries.

It does not explicitly allocate, copy, clear, or release storage.

The two global reads and the separately timed target reads are part of the established access sequence. They must not be replaced in this specification by an assumption that all dispatch state is captured at once.

### Callee-mediated state changes

The effects of F-2642535311f0 and the indirect targets are unknown. No contract here assumes that they preserve globals, object contents, dispatch contents, or memory validity. Nor does it assume that they own or release any memory.

Later accesses impose conditional validity requirements:

- The object’s +8 field must remain readable after the possible first dependency invocation.
- The global location must be readable for both dispatch-base reads.
- The structure selected by the second global read must permit the initial +0x30 target read.
- The structure at the retained first base must permit the later +0x18 read.
- That retained structure must also permit the final +0x30 read after the mandatory invocation.

These requirements describe what must remain valid to reach normal completion. They do not establish ownership or a general prohibition against deallocation. The body contains no object-field read after the mandatory invocation.

### Errors and arithmetic

No pointer validation, bounds checks, explicit overflow handling, or synchronization mechanism is visible.

Offsets are byte displacements, not element indices. Successful behavior is not established for address calculations that wrap the represented address width, cross inaccessible storage, or violate platform alignment requirements. The representation does not justify assigning portable signed-overflow or pointer-wrap semantics to those cases.

Invalid addresses, invalid indirect targets, nonreturning callees, and nonlocal control transfers have no specified recovery behavior.

## Dependencies

| Dependency or selection site | Established interaction | Unresolved behavior |
|---|---|---|
| F-2642535311f0 | Called conditionally after each nonzero field read, in +4 then +8 order. | Actual arguments, effects, ownership, global preservation, and failure behavior. |
| Initial optional indirect target | Selected from +0x30 through the second global-read base; called only if nonzero, with five represented arguments. | Identity, semantic purpose, complete ABI, and effects. |
| Mandatory indirect target | Selected from +0x18 through the retained first base after the initial optional invocation; called without a null check. | Identity, semantic purpose, ownership effects, and return behavior. |
| Final optional indirect target | Selected from +0x30 through the retained first base after the mandatory invocation; called only if nonzero, with two represented arguments. | Identity, phase meaning, complete ABI, and transfer mechanics. |

No opaque function IDs are supplied for the indirect targets. Their selection sites do not establish their identities or imply that any two sites resolve to the same function.

## Acceptance examples

These examples are conditional specifications, not executed tests. Except where an invalid-access case is stated, they assume readable required storage, callable selected targets, and normally returning callees.

1. **Zero object address.**  
   Regardless of other arguments or dispatch contents, the function returns without the described field reads, global reads, or calls.

2. **Zero fields and no optional targets.**  
   Both field reads produce zero. Both global reads select the same stable structure, whose +0x30 entry remains zero. No dependency or optional invocation occurs. The mandatory target is read and invoked once.

3. **Nonzero fields and stable optional target.**  
   Each field is nonzero when read. F-2642535311f0 is invoked twice, with the second field read between the invocations. Both global reads select the same structure. The initial optional, mandatory, and final optional invocations follow in that order, using their respective argument lists.

4. **First dependency changes the second condition.**  
   If +4 is nonzero and the first dependency invocation changes +8 from nonzero to zero, the second invocation is skipped. If it instead changes +8 from zero to nonzero, the second invocation occurs.

5. **The global changes between its two reads.**  
   The first read obtains one structure’s base. Before the second read, the global changes, and the second read obtains another structure’s base. The initial optional target is read at +0x30 in the second structure. The mandatory and final optional targets are read in the first structure, at +0x18 and +0x30 respectively. This example is conditional on the changed value being observed; it does not establish synchronization guarantees.

6. **Different structures independently enable optional invocations.**  
   If the second-read structure’s +0x30 entry is zero while the retained structure’s final +0x30 entry is nonzero, only the final optional invocation occurs. Reversing those conditions permits only the initial optional invocation.

7. **Initial invocation changes the retained mandatory entry.**  
   Even if the initial target came from another structure, a change it makes to the retained structure’s +0x18 entry affects the subsequent mandatory lookup.

8. **A callee replaces only the global.**  
   Once both global reads have occurred, replacing the global alone does not redirect either remaining lookup. Both use the retained first base.

9. **Mandatory invocation changes the final target.**  
   If the mandatory invocation clears the retained structure’s +0x30 entry, the final invocation is skipped. If it installs a different nonzero callable there, that callable is selected.

10. **Zero mandatory target.**  
    A readable +0x18 entry containing zero leads to an unchecked invocation attempt. No successful skip or recoverable status result is established.

## Uncertainties

The following remain **documented_only**:

- Object type, complete layout, and ownership.
- Meaning of the two conditional fields.
- F-2642535311f0’s actual interface and effects.
- Indirect target identities and semantics.
- Origin and meaning of the fifth initial-call argument.
- Whether the represented indirect argument lists capture the complete machine-level interface.
- Exact ABI, alignment, pointer representation, and final-transfer mechanics.
- Concurrent or asynchronous mutation rules and synchronization requirements.
- Whether any callee frees storage, fails, terminates execution, or transfers control nonlocally.

The concurrency uncertainty does not erase the established two-read sequence. What remains unknown is how external mutation is synchronized and observed, not whether the representation separately reads the global before the initial target lookup.

Cleanup, release, and phase-notification interpretations remain inferred intent only.

## Evidence

This contract rests solely on static inspection of the supplied representation and the dependency mapping for F-2642535311f0.

Established evidence includes the zero-address guard; ordered 32-bit reads at object offsets +4 and +8; conditional dependency invocations; two ordered reads of the dispatch-base global; use of the second-read base for the initial +0x30 target lookup; retention of the first-read base for the later +0x18 and +0x30 lookups; represented call arguments; and the absence of explicit stores or recovery logic.

The supplied recovery diagnostic limits confidence in the final indirect transfer’s mechanics. It does not establish additional behavior.

No dependency implementations were supplied. No runtime tests have been performed.
