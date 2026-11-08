<!-- Trusted provenance: F-cec90b5dcee3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cec90b5dcee3

## Purpose

F-cec90b5dcee3 conditionally reserves an amount from a mutable storage structure and returns an unsigned 32-bit location value. It first considers a linked sequence of reusable regions when a captured header counter is below 60. If that search is bypassed or exhausted, it subtracts the request from a descending boundary, optionally calling F-48bafd709153 before committing the boundary change.

“Storage reservation,” “reusable region,” and “boundary” are interpretations of the observed metadata operations, not independently established public-interface terminology.

An admitted request causes an available-amount debit before any dependency call. This function contains no explicit rollback of that debit. Its value after F-48bafd709153 executes is nevertheless unknown: the dependency may restore, overwrite, relocate, or invalidate the field or its storage.

A zero return is therefore not a reliable indication that no work or mutation occurred. It can represent initial rejection, a nonzero dependency result after the debit, or a completed boundary reservation whose arithmetic result is zero.

## Interface

The interface has four argument positions and returns an unsigned 32-bit value.

| Position | Established behavior |
|---|---|
| First | A 32-bit address representation identifying the control structure. The supplied representation treats it as signed for its declared type but uses it to address memory. |
| Second | A 32-bit request pattern, interpreted as signed during normalization. An admitted normalized request is between 4 and 65,535 inclusive. |
| Third | Not examined by this function. |
| Fourth | Its incoming value does not affect the result. Where this argument position participates in admission, its value has first been replaced by a control byte read. |

The return is a numeric location, not an established native pointer. Successful reusable-region paths return offsets relative to the captured backing base. The boundary path returns a full 32-bit subtraction result. The units and caller interpretation are documented_only.

### Memory layout

The following offsets are relative to the control address.

| Offset | Width | Observed role |
|---|---:|---|
| `+0x02` | 8 bits | Admission gate. |
| `+0x08` | 8 bits | Header selector; independently reread to locate the initial reusable-region link. |
| `+0x0e` | 16 bits | Additive component of the boundary lower bound. |
| `+0x12` | 16 bits | Available amount, checked and then independently reread for debit. |
| `+0x44` | 32 bits | Backing-storage base address. |

The two 16-bit control fields are accessed as direct halfwords. Their native byte order and alignment requirements are not established by this material.

The selected header begins at the captured backing base plus the selector byte read after admission.

| Header offset | Width | Observed role |
|---|---:|---|
| `+3`, `+4` | Two bytes | Big-endian unsigned 16-bit lower-bound component, multiplied by two. |
| `+5`, `+6` | Two bytes | Big-endian unsigned 16-bit descending boundary. |
| `+7` | 8 bits | Counter captured before strategy selection. |

The initial reusable-region link occupies two bytes beginning at the backing base plus a fresh selector read plus one. Thus, with an unchanged selector, it lies at header offsets `+1` and `+2`. The function does not require the two selector reads to agree.

Each nonzero link is a 16-bit offset from the captured backing base to a node:

| Node offset | Width | Observed role |
|---|---:|---|
| `+0`, `+1` | Two bytes | Big-endian next-node offset; zero terminates traversal. |
| `+2`, `+3` | Two bytes | Big-endian unsigned region size. |

These backing-storage quantities are explicitly assembled from bytes. Their big-endian interpretation is independent of native halfword byte order.

### Preconditions and limits

Every address actually read or written must designate accessible memory. A zero selector, zero backing address, malformed link, or out-of-range offset receives no special validation.

There are no established checks for storage bounds, alignment, field overlap, link cycles, or concurrent mutation. A search through undersized nodes terminates only if it eventually encounters a zero link or a sufficiently large node. A cycle consisting entirely of undersized nodes can prevent termination.

Ordinary storage-reservation interpretations require stable, noninterfering metadata. Unexpected aliases can cause one described write to affect another field or a later read. The function supplies no synchronization.

If the dependency returns zero, subsequent accesses through the previously captured header address must still be valid for execution to proceed safely. The dependency’s result alone does not establish that precondition.

## Behavior

### Request normalization and admission

The first metadata read obtains the unsigned 16-bit available amount.

The request is then interpreted as signed 32-bit. A signed value below 4 is replaced by 4. Consequently:

- Patterns representing 0 through 3 become 4.
- Patterns `0x80000000` through `0xffffffff` become 4.
- Values 4 through `0x7fffffff` remain unchanged.

The normalized request is compared with the initial available amount using signed comparison. Because the amount is unsigned 16-bit, admission requires a normalized request no greater than 65,535.

When the request is no greater than the initial amount, the function reads the admission gate. Otherwise, it does not read that gate. The backing-base field is then read regardless of whether admission succeeds.

Admission succeeds exactly when the normalized request fits the initial amount and the gate read is zero. Initial rejection returns zero without an explicit memory write or dependency call. Equality between the request and available amount does not bypass the gate.

### Early accounting write

On admission, the function rereads the available-amount halfword, subtracts the normalized request, and writes the result back as a 16-bit value.

This calculation uses the second read, not exclusively the amount used for admission. If the field is unchanged, the subtraction is nonnegative and exact. If its value has changed, no new admission check occurs; the stored result is reduced modulo 65,536.

This debit precedes reusable-region traversal, boundary testing, and any dependency call. The function itself performs no explicit rollback afterward. That absence of rollback does not establish the eventual field value: subsequent aliased writes or unspecified dependency effects may change it.

After the debit, the function reads the selector, derives the header address from the already captured backing base, and captures the header counter.

### Reusable-region traversal

A captured counter below 60 enables traversal. The function rereads the selector to locate the initial link slot.

For each link:

- Zero ends traversal and selects boundary reservation.
- A nonzero value identifies a node relative to the captured backing base.
- The node’s size low byte is read before its high byte, and the bytes form an unsigned 16-bit size.
- A size below the request causes traversal to continue using the node’s own next-link field.

Skipped nodes are not explicitly modified. The first node large enough is selected; no best-fit search is established.

The selected node is handled according to whether at least four units would remain.

#### Remainder from zero through three

If the size is at least the request but strictly below request plus four, the entire node is removed from the linked sequence.

The observable mutation order is:

1. Read the selected node’s next-link high byte and write it to the preceding link slot’s high-byte position.
2. Read the selected node’s next-link low byte and write it to the preceding link slot’s low-byte position.
3. Write the counter at the previously selected header.

The counter write uses the captured counter and the captured size low byte. Its result is equivalent, modulo 256, to captured counter plus size minus request.

For stable, nonaliasing metadata, the remainder is 0, 1, 2, or 3. Because traversal required a captured counter below 60, the resulting counter is at most 62.

The return is the selected node’s nonzero offset. There is no explicit clearing of the node, rewriting of its size, or initialization of its payload.

The two link-byte transfers are separate operations, not an established overlap-safe multi-byte copy. In particular, the first destination write can affect the source of the later read if the locations alias.

#### Remainder of at least four

If the size is at least request plus four, the node stays linked. Its reduced size is written in high-byte-then-low-byte order.

The return is the node offset plus the original size minus the request. This identifies a tail portion under the storage-reservation interpretation.

The return is not truncated to 16 bits. With independently valid fields, it can exceed 65,535 even though the input offset and size are each 16-bit quantities. For example, an offset of 65,535 and size 65,535 with request 4 produce return 131,066.

This branch does not explicitly write the counter or link fields. The threshold addition is safe in 32-bit arithmetic because admitted requests are at most 65,535.

### Boundary selection and dependency condition

Boundary reservation occurs when the captured counter is at least 60 or traversal reaches a zero link.

The function reads the current 16-bit boundary before deciding whether to call the dependency.

A captured counter of at least 60 forces the call. In this case, the lower-bound expression is not evaluated.

Otherwise, the function forms a candidate by subtracting the normalized request from the captured boundary using unsigned 32-bit arithmetic. It compares that candidate, interpreted as signed 32-bit, against this lower bound:

- The current control halfword at `+0x0e`;
- Plus twice the current big-endian header quantity at `+3` and `+4`.

The lower bound ranges from 0 through 196,605. A candidate equal to the lower bound is accepted without a dependency call. A candidate strictly below it triggers the call.

If subtraction underflows, its signed interpretation is negative, between −65,535 and −1, and therefore below every possible lower bound.

The supplied expression establishes the values used and short-circuit omission of the lower-bound reads when the counter forces a call. It does not independently establish a total read order among all byte operands within compound expressions.

### Dependency outcome and boundary commit

F-48bafd709153 has no explicit arguments in the supplied representation.

If it returns nonzero, F-cec90b5dcee3 immediately returns zero. The available-amount debit has already occurred, but its post-call value is unspecified. This function performs no explicit rollback or further boundary write on that outcome.

If the dependency returns zero, the function rereads the boundary at the previously captured header address. It does not recapture the backing base, selector, or header address, and does not repeat the lower-bound check.

If no call was necessary, it uses the boundary value read before the comparison.

The selected boundary value is then reduced by the request using unsigned 32-bit arithmetic. The function writes the low 16 bits to the boundary, high byte first and low byte second, and returns the full 32-bit result.

A boundary equal to the request produces a zero return and a stored zero boundary. If a zero-returning dependency leaves the reread boundary below the request, subtraction wraps in 32 bits; the stored boundary contains only the low 16 bits.

A dependency that invalidates the captured header storage can instead make these subsequent accesses invalid. No preservation or repair guarantee is established.

## Side effects

The function’s explicit writes are path-dependent:

| Path | Explicit writes |
|---|---|
| Initial rejection | None. |
| Any admitted path | Available-amount halfword debit. |
| Nearly exact reusable-region fit | Preceding link’s two bytes, then captured header’s counter. |
| Larger reusable region | Selected node’s size high byte, then low byte. |
| Boundary commit | Captured header’s boundary high byte, then low byte. |
| Nonzero dependency result | No additional explicit writes after the call returns. |

F-48bafd709153 may have additional, unspecified effects. In particular, the accounting field’s state after the call cannot be inferred merely from the earlier debit or the absence of caller-side rollback.

There is no explicit payload initialization, payload copying, allocation from an external service, deallocation, or clearing of consumed storage in this function. The dependency may perform such operations; its implementation is unavailable.

Ownership transfer, storage lifetime, and a release protocol are not established. Returning an offset does not by itself establish any ownership guarantee.

The operation is not established as transactional: admission causes a write before later decisions, and the caller supplies no explicit undo operation. This describes operation order, not a guarantee that the dependency leaves the debit intact.

## Dependencies

### F-48bafd709153

Established caller-facing behavior:

- Invoked only from boundary reservation.
- Invoked when the captured counter is at least 60, or when the signed candidate is below the computed lower bound.
- Receives no explicit arguments in the supplied representation.
- A nonzero result causes an immediate zero return.
- A zero result causes a boundary reread through the retained header address, followed by subtraction and boundary writes.

The following are documented_only:

- Return-value meaning beyond the zero/nonzero distinction.
- Memory or global reads and writes.
- Whether it restores or otherwise changes the available amount.
- Whether it preserves, moves, replaces, or releases backing storage.
- Whether it maintains validity of the retained header address.
- Whether it guarantees a sufficient boundary for the subsequent subtraction.
- Ownership and lifetime effects.

Neither outcome permits assuming that globals or storage were preserved.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Except where explicitly varied, memory is valid, metadata is stable, and fields do not unexpectedly alias.

1. **Minimum request.** Request 1, available amount 20, and gate zero admit a normalized request of 4. The debit writes 16 before either reservation strategy proceeds.

2. **Signed-negative pattern.** Request `0xffffffff` normalizes to 4, not 4,294,967,295. With sufficient amount and gate zero, it follows the same request-size behavior as request 4.

3. **Insufficient amount.** Request 12 and available amount 11 return zero. The gate is not read; the backing-base field is read. There is no explicit write or dependency call.

4. **Gate at exact capacity.** Request 8, available amount 8, and gate 1 return zero without a debit. Exact capacity does not override the gate.

5. **Independent debit read.** If admission reads amount 20 for request 8 but the debit rereads 5, the debit stores 65,533. There is no second capacity check.

6. **Skip and split.** With request 8 and a counter below 60, a size-6 node is skipped. A following node at offset `0x0200` with size 20 is reduced to 12, and the return is `0x020c`. Neither link is explicitly rewritten.

7. **Nearly exact fit.** For request 8, a selected size-10 node is unlinked. A captured counter of 5 becomes 7. The return is the node offset, and its size is not explicitly rewritten.

8. **Exact split threshold.** For request 8, a size-12 node remains linked with size 4. A size-11 node instead takes the unlink branch and increases the captured counter by 3.

9. **Lower-bound equality.** With an exhausted list, counter 0, boundary 100, request 8, control component 20, and header component 36, the lower bound and candidate are both 92. No dependency call occurs; the boundary and return become 92.

10. **Forced call and nonzero result.** Counter 60 forces a dependency call without evaluating the lower bound. A nonzero result produces return zero. The debit occurred before the call, but the final available amount is unspecified. If the dependency leaves that field intact and valid, it retains the debited value.

11. **Zero return after commit.** With an exhausted list, counter below 60, boundary equal to the request, and both lower-bound components zero, the boundary is written as zero and the return is zero without a dependency call.

12. **No post-call lower-bound check.** If the dependency returns zero, preserves valid access to the captured header, and leaves its boundary at 3 for request 4, the function stores `0xffff` and returns `0xffffffff`. No additional capacity check intervenes.

## Uncertainties

- **Inferred intent:** Variable-size reservation using reusable regions and a descending boundary.
- **documented_only:** Public field names, units, storage bounds, and interpretation of returned locations.
- **documented_only:** Native access byte order, alignment requirements, and ABI details.
- **documented_only:** Permitted aliases, link integrity, cycle freedom, and concurrency guarantees.
- **documented_only:** Ownership, lifetime, release protocol, and caller interpretation of zero.
- **documented_only:** Dependency effects, including the post-call available amount and continued validity of retained addresses.
- **documented_only:** Ordering between operand reads where the supplied compound expressions do not establish a separate sequence.

## Evidence

The supplied static function body establishes request normalization, admission reads, the independent debit read and write, captured base and header selection, reusable-region traversal, split and unlink mutations, boundary arithmetic, dependency conditions, and return handling.

It establishes that the debit precedes the dependency and that this function contains no explicit rollback. It does not establish the accounting field’s value or storage validity after the dependency executes.

The storage-reservation interpretation is inferred from those operations. No runtime tests have been performed. No dependency implementation, caller contract, or external interface documentation was supplied to resolve the documented_only items.
