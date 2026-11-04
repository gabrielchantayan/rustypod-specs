<!-- Trusted provenance: F-e5d97eec74ec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e5d97eec74ec

## Purpose

F-e5d97eec74ec accepts a receiver and an index request. Its ordinary contract is to determine an **effective index**, obtain any required admission, perform index-dependent interactions, and return that effective index. Rejection returns −1.

A receiver-controlled diversion takes precedence over this ordinary contract. It retrieves a four-byte value through an indirect dependency and passes that value to a dependency reported as non-returning.

The application-level operation is not established. In particular, the evidence does not prove element removal, movement, notification, destruction, or a change in collection size. Indexed-content processing is a plausible interpretation, not a required behavior.

This specification distinguishes established static behavior from inferred intent and unresolved properties. All acceptance examples are conditional static expectations; none have been executed.

## Interface

### Inputs and result

| Item | Contract |
|---|---|
| Receiver | Address used for field access and indirect dependency resolution. No local validity check is established. |
| Requested index | Signed 32-bit integer. |
| Ordinary result | Signed 32-bit integer: −1 on rejection, otherwise the retained effective index. |
| Diversion outcome | No normal result under the reported non-returning dependency contract. |

The requested index 2,147,483,647 has a special selection meaning, but only when an associated bound observation is positive. Otherwise, it remains an ordinary admission request with that exact value.

An admitted effective index of −1 suppresses the later operation interactions and returns −1. Consequently, the result alone cannot distinguish rejection from admitted suppression.

### Receiver representation

Only the following receiver-relative fields are established:

| Byte displacement | Represented width and interpretation | Use |
|---|---|---|
| 0 | 32-bit word used as a dispatch-storage address | Indirect dependency resolution. |
| 4 | Signed 32-bit integer | Selection, optional-interaction eligibility, and endpoint construction. |
| 16 | 32-bit integer converted to a character-sized value | Diversion decision. |

This specification calls the field at displacement 4 the **bound**. That name does not establish that it is a count, capacity, or last valid index. No general nonnegativity invariant is established.

Indirect dependency entries occur at byte displacements 64, 184, and 188 within dispatch storage. These displacements are not method ordinals. The complete receiver layout, pointer ABI, alignment requirements, and calling convention remain **documented_only**.

### Preconditions

Every reached read requires accessible storage of the represented width. Every reached indirect call requires accessible dispatch storage and a callable target. The diversion additionally requires its first dependency’s result to designate storage supporting a four-byte read.

Dependencies may affect storage used afterward. Validity is therefore required at each subsequent access, not merely at function entry. No dependency is assumed to preserve receiver lifetime, dispatch storage, globals, or other memory.

## Behavior

### Diversion takes precedence

The diversion decision depends on the character-sized conversion of the receiver’s mode field. It is not a full-width comparison of that field against zero.

When the converted value is nonzero:

- F-indirect-A receives the receiver and the unchanged original request.
- Its result designates the location of a four-byte value.
- The retrieved value is passed to F-feee9156a257.

F-feee9156a257 is reported as non-returning. Under that contract, there is no ordinary result and no ordinary admission or operation interaction.

The diversion preserves the original request even when it equals the maximum signed 32-bit value. The selection rule described below does not apply before this interaction.

The evidence does not resolve character width or integer-to-character conversion semantics. A nonzero full-width mode field is therefore insufficient, by itself, to establish diversion. For example, treating 256 as a zero converted value would require an additional eight-bit low-order conversion assumption.

### Admission and effective-index selection

Ordinary processing has two admission alternatives:

| Conditions | Admission requirement | Effective index |
|---|---|---|
| Request is 2,147,483,647 and the selection-eligibility bound observation is positive | F-402eef64af83 is bypassed | One less than a separately observed bound |
| Any other request, including the maximum request when its eligibility bound is nonpositive | F-402eef64af83 receives the receiver and original request | Original request, if admission succeeds |

For dependency admission, zero rejects; every nonzero result admits. No further meaning is established for the magnitude or sign of a nonzero result.

Rejection returns −1 and prevents all later operation interactions by this function. It does not reverse effects already caused by F-402eef64af83.

No additional local range check follows successful admission. A negative request, a request beyond the observed bound, or the maximum signed request can therefore become the effective index if the dependency admits it. Admission alone does not establish that subsequent arithmetic is representable.

The two bound observations involved in special selection are distinct. One authorizes selection; the other supplies the value whose predecessor becomes the effective index. No dependency call intervenes, but a coherent snapshot is not established. With a stable positive bound, selection yields an effective index between 0 and 2,147,483,646 inclusive.

### Operation obligations

An admitted effective index of −1 has no later operation obligations: F-indirect-B, F-indirect-C, and F-d6032bc0598f are all omitted.

An original request of −1 is not an unconditional shortcut. With diversion inactive, it still requires F-402eef64af83. Only successful admission reaches the suppression outcome.

For every other admitted effective index, the following interaction contract applies, subject to valid accesses, representable arithmetic, and normal completion of preceding dependencies:

| Dependency | When required | Arguments |
|---|---|---|
| F-indirect-B | The effective index’s successor is strictly less than the bound observed for this comparison | Receiver; successor of effective index; predecessor of a separately observed bound; 32-bit all-ones argument |
| F-indirect-C | Effective index is not −1 | Receiver; 32-bit all-ones argument |
| F-d6032bc0598f | Effective index is not −1, after F-indirect-C completes normally | Receiver; effective index |

The required interaction order is F-indirect-B, when eligible, before F-indirect-C, followed by F-d6032bc0598f.

Equality at the optional-interaction boundary is insufficient. If the successor equals the comparison bound, F-indirect-B is omitted.

The bound used for F-indirect-B’s endpoint is a separate observation from the bound used for eligibility. No guarantee requires those values to agree. Nor is endpoint ordering validated: the call can receive endpoints whose relationship differs from what the earlier comparison might suggest.

After the required interactions complete normally, the retained effective index is returned. No result from these three dependencies is used to select the ordinary result.

### Arithmetic boundaries

Index and bound arithmetic is represented as signed 32-bit arithmetic. The evidence does not establish execution semantics when a mathematical result falls outside −2,147,483,648 through 2,147,483,647.

Relevant boundaries are:

- The maximum signed effective index has no representable successor.
- The minimum signed bound has no representable predecessor.
- Selection from a stable positive bound is representable.
- A stable minimum bound cannot satisfy optional-interaction eligibility when the index successor is representable.
- A later bound observation can encounter the minimum value even after an earlier observation authorized selection or endpoint construction.

No wrapping, saturation, trapping, or other overflow rule is adopted here. Outcomes requiring unrepresentable arithmetic remain **documented_only**.

In particular, successful admission of the maximum signed request does not establish a defined completion. Its successor is needed for the optional-interaction comparison.

## Side effects

### Observable access and call ordering

The following relative ordering is established:

- The mode decision precedes ordinary selection or admission.
- On diversion, F-indirect-A completes before the four-byte read through its result; that read precedes F-feee9156a257.
- When admission is required, F-402eef64af83 completes before operation eligibility is evaluated.
- An effective index of −1 suppresses operation eligibility evaluation and all later operation calls.
- When F-indirect-B is invoked, it completes before dispatch information is read to resolve F-indirect-C.
- F-indirect-C completes before F-d6032bc0598f.

These distinctions are observable when dependencies change state. For example, a bound change during admission can affect the subsequent optional-interaction decision. A dispatch change during F-indirect-B can affect which target is subsequently used for F-indirect-C.

The effective index remains the value already selected or admitted. A dependency’s mutation of receiver fields does not itself cause reselection.

No complete ordering is established among independent reads used to prepare a single call. In particular, the specification does not require an order between dispatch resolution and endpoint-field reads within the same call preparation.

Separate bound observations do not establish volatile access, atomicity, synchronization, or permission for unsynchronized concurrent mutation.

### Mutation and ownership

No explicit write to receiver storage or other pointed-to storage appears in the supplied function body. This is not an observational read-only guarantee: every dependency may have effects not described here.

No local allocation, deallocation, retention, release, or ownership transfer is established.

Passing the receiver does not prove borrowing or transfer of ownership. Likewise, the address returned by F-indirect-A has no established ownership or lifetime guarantee. The body reads four bytes through that address and performs no explicit release of it.

Continued receiver access after a dependency is not evidence that the dependency preserves receiver lifetime. If a dependency invalidates storage needed later, no local protection or recovery is established.

### Rejection and interrupted execution

A zero result from F-402eef64af83 is the sole explicit local rejection condition.

There is no distinct local error result for invalid storage, unusable dispatch information, dependency failure, or unrepresentable arithmetic. No local exception handling or rollback is established. Earlier dependency effects are not explicitly undone if execution is interrupted.

The diversion is not rejection: under the reported non-returning contract, it has no normal integer result.

## Dependencies

The indirect IDs below identify call roles, not proven concrete target identities.

| Opaque ID | Established call-site contract | Unresolved properties |
|---|---|---|
| F-indirect-A | Resolved at dispatch displacement 64; receives receiver and original request; returns an address used for a four-byte read | Identity, result validity, lifetime, ownership, state changes, and failure behavior |
| F-feee9156a257 | Receives the retrieved four-byte value; reported as non-returning | Value interpretation, termination mechanism, ownership effects, and state changes |
| F-402eef64af83 | Receives receiver and original request when admission is required; zero rejects and nonzero admits | Admission policy, additional result meaning, ownership effects, state changes, and failures |
| F-indirect-B | Resolved at dispatch displacement 184; receives receiver, two index-related endpoints, and a 32-bit all-ones argument | Operation meaning, endpoint inclusivity, result contract, ownership effects, state changes, and failures |
| F-indirect-C | Resolved at dispatch displacement 188; receives receiver and a 32-bit all-ones argument | Operation meaning, result contract, ownership effects, state changes, and failures |
| F-d6032bc0598f | Receives receiver and effective index after F-indirect-C | Operation meaning, result contract, ownership effects, state changes, and failures |

The all-ones argument is established as a 32-bit pattern. Its application-level interpretation as a sentinel, mask, count, or other quantity is unknown.

No dependency is assumed to preserve globals, release memory, or take ownership.

## Acceptance examples

These examples are conditional requirements derived from static evidence, not runtime tests. Unless otherwise stated, the converted mode is zero, fields remain stable, storage and targets are usable, arithmetic is representable, and dependencies return normally.

| Conditions | Required outcome |
|---|---|
| Maximum signed request; bound 5 | Admission dependency is bypassed. Effective index is 4. F-indirect-B is omitted. F-indirect-C precedes F-d6032bc0598f with index 4. Result: 4. |
| Request 1; bound 5; admission nonzero | F-indirect-B receives endpoints 2 and 4 plus the all-ones argument. F-indirect-C and F-d6032bc0598f follow. Result: 1. |
| Request 4; bound 5; admission nonzero | The successor equals the bound, so F-indirect-B is omitted. The remaining operation dependencies execute in order. Result: 4. |
| Admission zero | Result: −1. No operation dependency executes. Admission effects are not reversed. |
| Request −1; admission nonzero | Admission occurs, but all operation dependencies are omitted. Result: −1. |
| Request −2; bound 3; admission nonzero | F-indirect-B receives endpoints −1 and 2. The remaining operation dependencies follow. Result: −2. |
| Maximum signed request; bound zero; admission zero | Admission receives the unchanged maximum request. Result: −1. |
| Maximum signed request; bound zero; admission nonzero | The effective index remains the maximum signed value. Successor evaluation is outside the representable range; subsequent behavior is documented_only. |
| Request 1; admission changes bound from 5 to 2 and admits | With bound 2 afterward, F-indirect-B is omitted. The remaining operation dependencies execute. Result: 1. |
| Converted mode nonzero | F-indirect-A receives the original request. The four-byte value retrieved through its result reaches F-feee9156a257. Under the reported contract, no normal result occurs. |

Additional state-sensitive expectations:

- If effective index 1 qualifies against bound 5, but endpoint construction observes bound 3, F-indirect-B receives endpoints 2 and 2.
- If selection eligibility observes a positive bound but selection itself observes zero, admission remains bypassed. The effective index becomes −1, all operation dependencies are omitted, and the result is −1.
- If selection observes the minimum signed bound after positive eligibility, predecessor arithmetic and its consequences remain **documented_only**.
- If F-indirect-B changes usable dispatch information, F-indirect-C is resolved afterward; an earlier cached target is not required by this contract.
- If F-indirect-A returns an unreadable address, no local validation or recovery precedes the attempted four-byte read.

These changing-observation examples describe conditional access behavior. They do not establish that arbitrary concurrent modification is valid.

## Uncertainties

The following remain **documented_only**:

- Complete receiver layout, alignment, pointer ABI, and calling convention.
- Character width and integer-to-character conversion rules.
- Execution semantics of unrepresentable signed arithmetic.
- Concrete indirect target identities and complete dependency contracts.
- Application meaning of the bound, endpoint pair, and all-ones argument.
- Dependency effects on globals, storage, ownership, allocation, and release.
- Storage lifetime and failure propagation across calls.
- Basis and mechanism of the reported non-returning property.
- Behavior if F-feee9156a257 returns contrary to that reported property.
- Concurrency requirements and consistency between separate observations.

The endpoint pair is compatible with processing a trailing indexed range. That is inferred intent only; it does not establish range inclusivity or the operation performed.

## Evidence

The supplied static representation establishes the relevant field widths and displacements, indirect dispatch displacements, diversion precedence, admission alternatives, argument relationships, separate bound observations, interaction order, retained-index result, and absence of explicit pointed-storage writes.

Prior prose and reviewer feedback are not independent behavioral evidence. The contract is expressed as observable obligations rather than a translation of implementation statements.

The non-returning property is reported by the supplied representation, not demonstrated by execution. Application semantics are inferred only where explicitly labeled. Conversion rules, overflow behavior, dependency internals, and ownership guarantees remain unresolved.

No runtime execution, smoke checks, or tests have been performed.
