<!-- Trusted provenance: F-20cd3fd7f55f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-20cd3fd7f55f

## Purpose

F-20cd3fd7f55f coordinates an ordered sequence of callbacks for a context and an event-like record. It optionally invokes a preliminary dependency, conditionally invokes an early callback, always invokes one of two alternative callbacks unless an earlier callback returns nonzero, and conditionally invokes a final callback.

A nonzero result from any result-bearing callback terminates processing and is returned unchanged. If no such callback returns nonzero, the function returns zero.

**Inferred intent:** the record represents an event or message, and nonzero callback results indicate that further processing should stop. The supplied material establishes the dispatch and return behavior, but not the semantic meaning of event categories, subtype values, or callback results.

## Interface

### Inputs

The function receives two arguments:

- **Context:** an address of an object-like structure containing a dispatch-table pointer and additional fields.
- **Record:** an address of a structure whose category determines how other fields are interpreted.

Neither argument is checked for nullness or validity.

### Width and layout model

The recovered interface uses integer-sized values for record addresses, fields, and results. The following contract describes the apparent 32-bit layout: four-byte integer and pointer words, with byte offsets as listed below. The supplied material does not independently establish the complete target ABI.

| Structure | Byte offset | Accessed value |
|---|---:|---|
| Context | `0x000` | Pointer to a dispatch table |
| Context | `0x050` | Integer used in the final equality comparison |
| Context | `0x108` | Integer narrowed for the preliminary dependency |
| Record | `0x004` | Integer category |
| Record | `0x008` | Either a nested-record pointer or an integer comparison value |
| Nested record | `0x004` | Integer subtype |

The record field at `0x008` has different interpretations on different paths:

- When the category is `0x16`, it is treated as a pointer, and the subtype is read from that pointed-to structure at offset `0x004`.
- When the category is `0x0f` during the final stage, its stored integer value is compared directly with the context field at `0x050`. The function does not dereference it on that path.
- Other categories do not, by themselves, cause this function to read that field.

The dispatch table contains callable entries at these byte offsets:

| Dispatch-table offset | Local opaque dependency ID | Role |
|---:|---|---|
| `0x0e0` | F-000000000001 | Conditional final callback |
| `0x10c` | F-000000000002 | Selected callback for category `0x16`, subtype `0x2a` |
| `0x110` | F-000000000003 | Selected callback for all other cases |
| `0x124` | F-000000000004 | Conditional early callback |

These four IDs are document-local labels for indirect dependency roles. They do not establish the identities of the actual functions stored in the table. Different entries may contain the same callable address, and entries may change during processing.

### Output

The result is an integer, apparently 32 bits wide:

- The exact nonzero value returned by the first result-bearing callback that returns nonzero; or
- Zero if all invoked result-bearing callbacks return zero.

Negative results are treated as nonzero. The function does not normalize them to a Boolean value.

### Preconditions

Every memory access reached during execution must be valid. In particular:

- The record category must be readable at each stage that executes.
- Whenever a stage observes category `0x16`, the current nested-record pointer and its subtype field must be readable.
- The context fields used on an executed path must be readable.
- Each selected dispatch entry must contain a valid callable address with a compatible calling convention.
- Dependencies must return normally for subsequent stages to execute.

These requirements apply to state as it exists at each stage, including changes made by earlier dependencies. The function supplies no fallback for invalid pointers, inaccessible fields, or invalid callback entries.

## Behavior

### 1. Preliminary dependency

The function first reads the record category.

If the category is `0x16`, it reads the pointer at record offset `0x008`, then reads the subtype at nested offset `0x004`. It subtracts `0x2500` from that subtype and compares the result with `2`.

Under ordinary 32-bit arithmetic, the relevant matching subtype is `0x2502`.

When the comparison matches, the function:

1. Reads the context integer at offset `0x108`.
2. Narrows that integer to a plain character value.
3. Calls F-8a6ca5c1de04 with the context and the narrowed value.

On an eight-bit-character target, this narrowing retains the low eight bits. Whether the dependency receives the resulting byte as a negative or nonnegative numeric value depends on plain-character signedness and argument-passing rules, which are not established here.

No return value from F-8a6ca5c1de04 is inspected or propagated. Its mutations, if any, can affect every later stage.

### 2. Conditional early callback

The function reads the record category again after the preliminary stage.

If this fresh category is `0x16`, it reloads the nested-record pointer and reads the current subtype. If that subtype is zero, it invokes F-000000000004 through dispatch-table offset `0x124`, passing the original context and record addresses.

- If the callback returns nonzero, the function returns that value immediately.
- If it returns zero, processing continues.
- If the category or subtype condition does not match, the callback is not invoked.

This stage does not reuse the preliminary stage’s category or subtype. For example, F-8a6ca5c1de04 may change a subtype from `0x2502` to zero, causing the early callback to run.

The context’s dispatch-table pointer is obtained when this callback is selected, rather than being retained from function entry.

### 3. Required selected callback

Unless the early callback has already returned nonzero, the function reads the record category again.

If the fresh category is `0x16`, it reloads the nested-record pointer and reads the current subtype:

- Category `0x16` with subtype `0x2a` selects F-000000000002 at dispatch-table offset `0x10c`.
- Every other combination selects F-000000000003 at dispatch-table offset `0x110`.

Exactly one of these two callback roles is invoked when this stage is reached. Both receive the original context and record addresses.

A nonzero result is returned immediately and unchanged. A zero result permits the final stage.

The selected dispatch entry is loaded from the context’s current dispatch table. Consequently, mutations performed by the preliminary dependency or early callback can change both which role is selected and which callable address occupies that role.

### 4. Conditional final callback

After the selected callback returns zero, the function reads the record category again.

If the fresh category is `0x0f`, it:

1. Reads the integer at record offset `0x008`.
2. Reads the context integer at offset `0x050`.
3. Compares those values for equality.

If they are equal, it invokes F-000000000001 through dispatch-table offset `0x0e0`, passing the original context and record addresses.

A nonzero result is returned unchanged. If the callback returns zero, or if the final condition does not match, the function returns zero.

The equality comparison is between stored values. It does not establish that either value is an address, handle, identifier, or owned resource.

### Read freshness and ordering

Repeated reads are behaviorally significant. Category and subtype values are not captured once for the whole invocation.

The established dependency order is:

1. Optional F-8a6ca5c1de04.
2. Optional F-000000000004.
3. Exactly one of F-000000000002 and F-000000000003, if execution reaches this stage.
4. Optional F-000000000001.

Each early return prevents all later stages and their reads.

Within a category-`0x16` inspection, the category is checked before the nested-record pointer is dereferenced. Within the final category-`0x0f` inspection, the record comparison value is read before the context comparison value.

This ordering describes accesses established by the supplied representation. It does not establish volatile-access guarantees, atomicity, memory barriers, or behavior under unsynchronized concurrent mutation.

## Side effects

No direct memory writes, allocations, releases, or ownership transfers are shown in F-20cd3fd7f55f itself.

All invoked dependencies receive the context address. Every indirect callback also receives the record address. F-8a6ca5c1de04 receives a narrowed context-field value instead of the record address.

Dependency side effects are unknown. In particular, this contract does not assume that any dependency:

- Preserves the record, context, nested record, or dispatch table.
- Preserves unrelated global state.
- Retains or releases memory.
- Avoids reentrancy.
- Leaves earlier-read storage valid.

Continued execution requires later accesses to remain valid. A zero callback result is not evidence that the callback preserved memory or ownership.

The function neither copies nor restores the input structures. If dependencies mutate them and processing continues, later stages inspect the resulting state.

## Dependencies

| Opaque ID | Invocation and consumed result |
|---|---|
| F-8a6ca5c1de04 | Direct call when the preliminary category/subtype condition matches. Receives context and a plain-character conversion of the context field at `0x108`. No result is consumed. |
| F-000000000004 | Indirect call through offset `0x124` when a fresh inspection finds category `0x16` and subtype zero. Receives context and record. Nonzero result terminates processing. |
| F-000000000002 | Indirect call through offset `0x10c` when the selected-callback stage finds category `0x16` and subtype `0x2a`. Receives context and record. Nonzero result terminates processing. |
| F-000000000003 | Indirect call through offset `0x110` for every other selected-callback condition. Receives context and record. Nonzero result terminates processing. |
| F-000000000001 | Indirect call through offset `0x0e0` when the final stage finds category `0x0f` and equal comparison fields. Receives context and record. Its result determines whether the final return is nonzero or zero. |

The supplied material gives no dependency implementation contracts. Their names here describe neither ownership nor semantic event handling.

## Acceptance examples

These are conditional static acceptance examples, not observed runtime results. Unless stated otherwise, all accessed memory is valid, dependencies return normally, and dependencies leave relevant state unchanged.

1. **Default selection without nested-record access.**  
   The category is neither `0x16` nor `0x0f`. F-000000000003 runs. If it returns zero, the function returns zero. No nested-record dereference is required by this function.

2. **Special selected callback.**  
   The category is `0x16` and the subtype is `0x2a`. F-000000000002 runs; F-000000000003 does not. If it returns `7`, the function returns `7` without executing the final-stage reads.

3. **Early callback takes precedence.**  
   The category is `0x16` and the subtype is zero. F-000000000004 runs first. If it returns `-3`, the function returns `-3`; neither selected callback nor the final callback runs.

4. **Zero early result permits default selection.**  
   The category remains `0x16`, the subtype remains zero, and F-000000000004 returns zero. F-000000000003 then runs because zero is not `0x2a`.

5. **Preliminary call does not consume processing.**  
   The category is `0x16` and the subtype is `0x2502`. F-8a6ca5c1de04 runs. If it leaves the subtype unchanged, the early callback is skipped and F-000000000003 is selected.

6. **Mutation between stages is visible.**  
   The initial subtype is `0x2502`. F-8a6ca5c1de04 changes it to zero. F-000000000004 must then be considered using that new subtype. If this callback returns zero and changes the subtype to `0x2a`, F-000000000002 is selected next.

7. **Final equality match.**  
   The category is `0x0f`, F-000000000003 returns zero, and the record field at `0x008` equals the context field at `0x050`. F-000000000001 runs. If it returns `19`, the function returns `19`.

8. **Final equality mismatch.**  
   The final category is `0x0f`, but the two comparison values differ. The final callback is skipped and the function returns zero.

9. **Selected callback enables the final stage.**  
   A selected callback returns zero after changing the category to `0x0f` and establishing equal comparison fields. The final callback runs even if the record had a different category at entry.

10. **Selected callback suppresses the final stage by result.**  
    A selected callback establishes the same final-state conditions but returns nonzero. Its result is returned immediately; the final callback does not run.

## Uncertainties

The following are **documented_only** limitations:

- The apparent 32-bit integer/pointer layout and eight-bit character narrowing lack independent ABI confirmation.
- Plain-character signedness and the exact promoted argument value received by F-8a6ca5c1de04 are unknown.
- The supplied representation does not establish general overflow semantics for the preliminary subtraction. No validation or error branch surrounds that arithmetic.
- Category, subtype, context-field, and result meanings are unknown.
- Actual identities and implementations of indirect callbacks are unknown.
- Dependency mutation, allocation, release, retention, exception, and reentrancy behavior are unknown.
- Aliasing and concurrency guarantees are not supplied.
- Invalid-memory behavior is not specified as a recoverable error. No explicit checks or error results are shown.
- No runtime tests have been performed.

## Evidence

**Established static evidence:** the supplied function representation contains the listed field accesses, constants, conditional dependency calls, dispatch-table offsets, repeated reads, nonzero-result early returns, and final zero return. It shows no direct memory writes or ownership-management operations.

**Inferred intent:** event-like classification, object-like dispatch, and nonzero-as-stop processing are interpretations of that structure, not established domain semantics.

**Unknown:** target ABI details beyond the apparent layout, dependency implementations, external state invariants, and runtime outcomes. The acceptance examples specify consequences conditional on valid storage and stated dependency behavior; they are not test reports.
