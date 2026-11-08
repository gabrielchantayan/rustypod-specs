<!-- Trusted provenance: F-5fb9be13a8f7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5fb9be13a8f7

## Purpose

F-5fb9be13a8f7 selects eligible records from two ordered candidate chains and invokes an operation associated with each selected record’s owner. When that operation returns zero, the function unlinks the record from an owner-local chain, updates an accounting total, makes two disposal-like calls, and decrements an owner counter.

Processing ends when:

- A nonnegative requested limit has been reached or exceeded.
- Neither candidate chain contains an eligible record.
- The owner-associated operation returns a nonzero result.

The return value is the accumulated accounting total, not the operation’s status.

Before selection begins, the function marks objects reachable through a separate global chain. Every explicit return path clears markers on the objects reachable through that chain at exit. Clearing does not restore previous marker values or necessarily visit the same objects marked on entry.

Resource reclamation is inferred intent. The established behavior is traversal, selection, temporary state modification, dependency invocation, unlinking, accounting, and counter modification. Actual memory release is not established.

## Interface

### Arguments and result

| Input | Width and interpretation | Visible use |
|---|---|---|
| Requested limit | Signed 32-bit integer | Checked before each selection. Negative values disable the limit-based stopping condition. |
| Second argument | 32-bit value | Unused by the visible body. |
| Third argument | 32-bit value | Unused by the visible body. |
| Initial operation context | 32-bit value | Copied into an addressable local word passed to F-26ff4a9b1180. |

The function returns a signed 32-bit accounting total, initially zero. Its arithmetic limitations are specified below; unrestricted inputs do not support a guarantee of a nonnegative or monotonically increasing result.

The local context word is initialized during every evaluation of the preliminary marking loop’s continuation condition, including the final evaluation. It is therefore initialized even when the marking chain is empty.

Once processing begins, this function does not visibly reinitialize the context word. Each invocation of F-26ff4a9b1180 receives the address of the same local storage. Its contents can change through dependency effects, including possible intervening use of a retained pointer. The storage has the lifetime of this function invocation; whether any dependency retains its address is unknown.

### Referenced layouts

All offsets are byte offsets. Pointer-bearing fields and ordinary word accesses in the supplied representation are 32 bits. The layouts describe accessed fields, not complete object definitions.

| Structure | Offset | Field |
|---|---:|---|
| Marking-chain root | +0x04 | First marking-chain object |
| Marking-chain object | +0xD4 | Next marking-chain object |
| Marking-chain object | +0xDC | 32-bit marker |
| Candidate container | +0x00 | Primary-chain head |
| Candidate container | +0x08 | Preferred-chain head |
| Candidate record | +0x00 | Owner pointer |
| Candidate record | +0x18 | Next record in owner-local chain |
| Candidate record | +0x1E | One-byte preferred-chain exclusion flag |
| Candidate record | +0x2C | Next record in candidate chain |
| Candidate record | +0x34 | 32-bit first disposal argument |
| Owner | +0x14 | Unsigned one-byte accounting factor |
| Owner | +0x3C | Signed 32-bit accounting component |
| Owner | +0x40 | Signed 32-bit accounting component |
| Owner | +0x44 | Signed 32-bit counter |
| Owner | +0x78 | 32-bit temporarily cleared field |
| Owner | +0x88 | Owner-local record-chain head |
| Owner | +0xE0 | 32-bit eligibility field; zero permits selection |

The owner-local chain and candidate chains use different links. Unlinking through +0x18 does not establish removal from a chain using +0x2C.

Marking-chain objects and owners must not be assumed to share a type or identity.

### Structural and lifetime preconditions

Normal completion requires accessible roots, containers, records, owners, and fields for each access reached. In particular:

- A selected record must belong to its owner’s +0x88 chain when unlinking occurs.
- Owner-local predecessor traversal must reach that record.
- Traversals intended to finish must reach a terminating null link or a qualifying record.
- Dependencies must leave storage needed by subsequent accesses usable.
- The owner must remain accessible through restoration, accounting, and the final counter update.

The function provides no visible validation or recovery for violations of these conditions.

## Behavior

### Entry marking and retained state

The accounting total starts at zero. The function obtains the marking-chain head from the global root’s +0x04 field.

For each nonnull object reached, it writes the 32-bit value one to +0xDC and then reads +0xD4 to advance. Previous marker values are not saved.

During each marking-loop condition evaluation, the function also obtains the candidate-container pointer and copies the fourth argument into the local context word. The container pointer retained after the final evaluation is used throughout the processing phase.

This function does not visibly refresh that retained pointer from its global source after dependency calls. It does reread candidate-head fields from the retained container during later searches. Consequently, modifying the retained container’s fields can affect later selection; replacing only the global container pointer does not redirect this invocation.

### Limit check

Before every selection attempt, processing stops if the requested limit is nonnegative and the signed accounting total is at least that limit.

Thus:

- A zero limit prevents selection and dependency calls, but not entry marking or exit clearing.
- Every negative limit disables this check.
- The function does not estimate a candidate’s contribution before selecting it.
- A completed iteration can overshoot a positive limit.
- The limit is checked only after the preceding iteration has completed both disposal-like calls and the counter decrement.

These rules do not establish termination for negative limits or when arithmetic produces a negative total.

### Candidate selection

The preferred chain is searched first, starting from container +0x08 and following record +0x2C.

A preferred-chain record qualifies only when its +0x1E byte is zero and its owner’s +0xE0 field is zero. The exclusion byte is examined first. When it is nonzero, the owner eligibility field is not read for that record’s preferred-chain eligibility check.

The first qualifying preferred record is selected. If the preferred search reaches null, the primary chain is searched from container +0x00, also using +0x2C links.

Primary-chain qualification requires only that the owner’s +0xE0 field be zero. The +0x1E exclusion byte does not participate in that search.

If the primary search reaches null without a match, processing exits. Otherwise, the first qualifying primary record is selected.

Every new iteration restarts with the preferred-chain head. No traversal cursor is retained across completed iterations.

### Owner operation and result handling

For the selected record, the function obtains its owner, saves the owner’s 32-bit +0x78 field, and writes zero to that field.

It then calls F-26ff4a9b1180 with:

- The owner.
- The address of the local 32-bit context word.

If the dependency returns normally, the function restores the saved +0x78 value before examining the returned status. A change made to that field during the dependency call is overwritten by this restoration.

A zero status permits processing to continue.

A nonzero status causes a call to F-60ee867dc1bd with the owner and that status. If this call returns, the function takes the common exit path. The nonzero status itself is not returned to the caller.

On this failure path, the function performs no record unlink, accounting addition, disposal-like call, or counter decrement for the failed attempt. This restriction describes direct actions of F-5fb9be13a8f7, not effects performed inside dependencies.

### Owner-local unlink

After a zero operation status, the function examines the owner-local chain headed at owner +0x88. The visible sequence loads this head and separately reads the head field for comparison with the selected record; no atomic snapshot or synchronization is established.

If the selected record is the head, the function reads its +0x18 successor and writes that successor into owner +0x88.

Otherwise, it traverses the owner-local chain using +0x18 until it finds a predecessor whose link points to the selected record. It then replaces the predecessor’s link with the selected record’s +0x18 successor.

There is no visible null check or bounded failure path during predecessor search. Missing membership can therefore lead to invalid access or nontermination rather than a reported error.

No direct write removes the selected record from either candidate chain.

### Accounting and integer semantics

After unlinking, the accounting update uses:

- The signed 32-bit component at owner +0x40.
- The signed 32-bit component at owner +0x3C.
- Twenty times the unsigned byte at owner +0x14.
- The existing signed accounting total.
- A fixed contribution of sixty.

The accounting fields are read after F-26ff4a9b1180 returns and after unlinking. Their earlier values are not preserved for this calculation. The relative read order among the accounting operands is not established.

The arithmetic has distinct stages and limitations:

1. **Initial signed addition.** The +0x40 and +0x3C components are added as signed 32-bit integers. This intermediate can overflow even if a later mathematical total would fit. The supplied evidence does not establish a wrapping or saturating result for that overflow.
2. **Byte-derived multiplication.** The byte is converted to unsigned 32-bit form and multiplied by twenty. Its product ranges from zero through 5,100, so this product cannot overflow 32 bits.
3. **Unsigned portion.** The signed component sum participates in unsigned 32-bit arithmetic with the product. The previous accounting total is also converted for addition in this unsigned portion, followed by the fixed sixty. This portion uses arithmetic modulo $2^{32}$.
4. **Result storage.** The resulting unsigned value is stored into the signed 32-bit accounting total. If it exceeds the signed positive range, its exact conversion requires target confirmation.

Unsigned arithmetic in the later stages does not establish wrapping behavior for the earlier signed component addition. Nor does it establish a portable signed result for the final conversion.

Where the signed intermediate is representable and the resulting total remains in signed range, the update corresponds to adding both owner components, twenty times the byte factor, and sixty to the previous total.

Subsequent limit comparisons use the stored total as signed.

### Disposal-like calls and counter update

After storing the updated total, the function performs these actions in order:

1. Reads record +0x34 and passes that 32-bit value to F-94cb53877c15.
2. Calls F-94cb53877c15 with the selected record’s address.
3. Reads owner +0x44 and writes its value minus one.
4. Returns to the limit check.

The counter is read after both calls return. If a dependency changes it, the decrement applies to the then-current value, not an earlier snapshot.

There is no zero guard. Zero becomes negative one under ordinary representable signed subtraction; decrementing the minimum signed 32-bit value raises a separate signed-overflow uncertainty.

Neither disposal call’s return value is used. Their names and argument order do not establish freeing, reference release, ownership transfer, or candidate-chain maintenance.

### Exit clearing

Every explicit exit begins a new traversal from the marking root’s then-current +0x04 head. Each reachable object receives a 32-bit zero at +0xDC before its +0xD4 link is read.

The function then returns the accounting total.

Exit traversal follows current membership:

- An object marked initially but detached before exit may remain marked.
- An object newly reachable at exit is cleared even if it was not marked on entry.
- A preexisting marker value other than zero is not restored.

Restoration of owner +0x78 and exit marker clearing are established only for the normal-return paths described above. No exceptional or nonlocal-exit cleanup is visible.

## Side effects

Direct externally visible writes comprise:

- Entry marker writes of one.
- Exit marker writes of zero.
- Temporary clearing and restoration of owner +0x78.
- Owner-local head or predecessor-link replacement.
- Owner counter decrement.

The function also initializes local context storage and exposes its address to F-26ff4a9b1180.

There is no visible locking, pointer validation, rollback, allocation, or explicit candidate-chain unlink. Successful earlier iterations remain completed when a later operation returns nonzero.

Dependency effects are unrestricted by the supplied evidence. Preservation of globals, eligibility, chain membership, object contents, or memory lifetime must not be assumed. Concurrent mutation and reentrancy are not assigned supported semantics.

## Dependencies

| Opaque ID | Established invocation contract |
|---|---|
| F-26ff4a9b1180 | Receives the selected owner and the address of the mutable local 32-bit context word while owner +0x78 is zero. Zero permits unlinking; nonzero initiates failure handling and exit. |
| F-60ee867dc1bd | Receives the owner and the nonzero operation status after +0x78 restoration. Its return value, if any, is unused. |
| F-94cb53877c15 | Invoked first with record +0x34, then with the record address. Neither return value is used. Ownership and disposal semantics are unknown. |

The selected record must remain readable through the +0x34 argument load. The evidence does not establish how either disposal-like call manages its lifetime. The owner is accessed after both calls and must remain usable for the counter update.

Continued candidate traversal must also remain compatible with dependency effects. Because this function does not directly remove candidate-chain links, it cannot independently guarantee that a later search avoids an invalidated record.

## Acceptance examples

These are conditional static criteria, not runtime observations. Unless stated otherwise, referenced storage is valid, arithmetic stays within the relevant signed ranges, and dependencies return normally.

1. **Zero limit.** A requested limit of zero produces no candidate selection or dependency call. Entry marking and exit clearing still occur. The returned total is zero.

2. **No match.** If no preferred record passes both checks and no primary record has an owner eligibility field equal to zero, the function clears the exit marking chain and returns zero.

3. **Preferred priority.** If both chains contain qualifying records, the first qualifying preferred-chain record is selected before any primary-chain record.

4. **Flag asymmetry.** A record with a nonzero +0x1E byte is skipped during preferred search without reading its owner eligibility field for that check. If later encountered in primary search, it qualifies when its owner’s +0xE0 field is zero.

5. **Limit overshoot.** With limit one, owner components ten and twenty, and byte factor two, one successful iteration adds 130. Both disposal-like calls and the counter decrement occur before the next limit check returns 130.

6. **Operation failure after prior work.** With total 130, limit 200, and another eligible candidate, an operation status of seven causes +0x78 restoration followed by F-60ee867dc1bd with the owner and seven. If it returns, exit clearing occurs and the result is 130. No direct unlink or disposal is performed for that failed attempt.

7. **Context identity and conditional persistence.** If a first operation changes the context word and a second operation is reached, both receive the same local address. If no intervening dependency modifies that storage, the second sees the changed value. Without that assumption, only storage identity and absence of visible reinitialization are guaranteed; the two intervening disposal-like calls cannot be assumed harmless.

8. **Head removal.** If the selected record is the current owner-local head, owner +0x88 is replaced by its +0x18 successor. No predecessor search is required.

9. **Post-call counter value.** If the second disposal-like call leaves the owner counter at five, the subsequent direct update stores four, regardless of its value before the calls.

10. **Changed marking membership.** If an initially marked object is detached from the marking chain before exit, clearing need not reach it. Its marker is not guaranteed to become zero.

11. **Signed intermediate overflow.** If +0x40 contains 2,147,483,647 and +0x3C contains one, the initial signed addition overflows. Later unsigned arithmetic does not justify an acceptance criterion requiring a wrapped total.

## Uncertainties

The following remain **documented_only**:

- The accounting units and semantic meaning of the fields and return value.
- Behavior when the initial signed component addition overflows.
- Exact target conversion of an out-of-range unsigned result to signed 32-bit form.
- Behavior when the signed counter decrement overflows.
- Whether F-94cb53877c15 releases storage, references, or other resources.
- Whether dependencies maintain candidate-chain links or eligibility.
- Whether dependencies retain or subsequently modify the context address.
- Preservation of globals, object identity, and memory lifetime across calls.
- Supported concurrency, reentrancy, and nonlocal-exit behavior.
- Outcomes for malformed pointers, inaccessible storage, cycles, or missing owner-local membership.

Cycles can prevent termination. A negative limit alone does not ensure eventual exhaustion. No runtime tests have been performed.

## Evidence

Established static evidence is the supplied function body and opaque dependency mapping. It supports the described access widths and offsets, selection precedence, short-circuit eligibility check, explicit writes, call order, return paths, and distinction between the initial signed addition and subsequent unsigned arithmetic.

Reclamation, byte accounting, and processing-mode markers are inferred interpretations, not established API semantics. Dependency implementations and target-specific exceptional arithmetic behavior were not supplied.

This contract uses static analysis only. No implementation access, execution, or runtime verification was performed.
