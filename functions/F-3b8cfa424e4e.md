<!-- Trusted provenance: F-3b8cfa424e4e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3b8cfa424e4e

## Purpose

F-3b8cfa424e4e produces a signed ordering result for two record references. Its decision has three levels:

1. A category derived independently from bit `0x40` of each record’s byte at offset `0x18c`.
2. A result obtained by preparing two temporary representations and passing them to an opaque comparison dependency.
3. An unsigned 32-bit field comparison at record offset `0x2c`, reached only when the dependency returns zero.

A set category bit sorts before a clear category bit. Within equal categories, the opaque comparison result takes precedence over the numeric field.

This ordering interpretation is inferred intent. The established behavior is the branching, dependency invocation, memory access, and return behavior described below. In particular, the available evidence does not establish that the dependencies produce a stable or mathematically consistent ordering.

## Interface

### Inputs

The function accepts two record references, designated **first record** and **second record** here.

The supplied interface represents these references as integer-valued address carriers rather than typed record pointers. The contract uses the 32-bit integer model represented by the supplied evidence. A port to another data model must establish address-carrier width and pointer conversion behavior separately.

The function directly accesses only the following portions of each referenced record:

| Offset from record base | Access width | Interpretation |
|---|---:|---|
| `0x18c` | 8 bits | Only bit `0x40` affects category selection. |
| `0x2c` | 32 bits | Unsigned numeric value, used only after equal categories and a zero dependency result. |

These offsets do not establish a complete record layout. Other bytes may be accessed by F-b3dff379655a, whose implementation is unavailable.

No null checks, record-length checks, address validation, or explicit alignment handling are visible. Callers must supply references valid for every access actually reached, including accesses performed by dependencies. Exact alignment requirements depend on the target execution environment and are not established here.

The two record references need not be distinct according to the visible control flow. Aliasing is neither rejected nor specially handled.

### Output

The function returns a signed 32-bit integer:

- `-1` when the first record’s derived category is smaller, or when the final unsigned comparison establishes a smaller first numeric value.
- `1` when the first record’s derived category is larger, or when the final numeric decision falls through to the greater-result branch.
- `0` when the final numeric decision reaches its equality-result branch.
- The exact nonzero result returned by F-2f4a2d2fe72a when that dependency determines the result.

The return range is therefore not restricted to `-1`, `0`, and `1`. There is no visible error-return channel distinct from the ordering result.

## Behavior

### 1. Determine both categories

The function first reads the first record’s byte at offset `0x18c` and derives a category:

- Bit `0x40` set: category `1`.
- Bit `0x40` clear: category `4`.

It then performs the corresponding byte read and category derivation for the second record.

Only the selected bit matters. Other bits in either byte do not affect these category values.

Both category-byte reads occur before either dependency is called. Both also occur before the function returns a category-only result; a decisive first category does not eliminate the second byte read.

The categories are retained as values for this decision. Later dependency activity does not cause the category bytes to be reread.

### 2. Resolve unequal categories directly

If the first category is smaller than the second category, the function returns `-1`.

If the first category is larger than the second category, the function returns `1`.

Consequently:

| First category bit | Second category bit | Result |
|---|---|---:|
| Set | Clear | `-1` |
| Clear | Set | `1` |

Neither dependency is called on these paths, and neither numeric field at offset `0x2c` is directly read by this function.

### 3. Prepare two temporary representations for equal categories

When the categories are equal, the function uses two distinct temporary storage regions, each 512 bytes in size.

The call sequence is:

1. F-b3dff379655a receives the first record reference and the first temporary region.
2. After that call returns, F-b3dff379655a receives the second record reference and the second temporary region.

The surrounding function does not initialize either region before passing it to the dependency. It does not visibly inspect either preparation call’s return value.

The existence and size of these regions do not establish the format, initialization extent, encoding, termination, or meaningful content of the prepared representations. Their use as prepared comparison inputs is inferred from the call pattern; their actual contents and validity are dependency obligations.

The two calls are sequential. Effects of the first call can therefore affect the second call. Nothing in the available evidence guarantees that the dependency preserves either record, global state, or storage reachable through either record reference.

### 4. Obtain the opaque comparison result

After both preparation calls return, the function calls F-2f4a2d2fe72a with:

1. The first temporary region.
2. The second temporary region.
3. The integer value `0x1b`, or decimal `27`.

The argument order preserves the first-record/second-record order.

The third argument is established as a numeric value, not as a proven byte count, character count, or other unit. The available evidence does not justify identifying this dependency with a particular standard comparison routine.

If the dependency returns a nonzero signed result, F-3b8cfa424e4e immediately returns that exact result. It does not normalize the result to a sign and does not directly read either numeric field on this path.

If the dependency returns zero, execution continues to the numeric tie-breaker.

### 5. Apply the unsigned numeric tie-breaker

The function compares the 32-bit unsigned fields at offset `0x2c` in the two records.

The first numeric comparison asks whether the first value is less than the second value. If so, the function returns `-1`.

If that comparison is false, the visible operation performs another comparison of those fields, this time testing whether the first value is less than or equal to the second value:

- If true, the function returns `0`.
- If false, it returns `1`.

For values that remain stable across these accesses, this is ordinary unsigned ascending order: smaller returns `-1`, equal returns `0`, and larger returns `1`.

The second comparison is significant because the fields are referenced again rather than explicitly cached once. If the underlying memory changes between comparisons, the result need not describe one stable pair of values. Exact physical load counts and ordering require lower-level evidence; the source-facing contract establishes the two conditional comparisons and does not replace them with a guaranteed single-snapshot comparison.

No subtraction is used to derive the result. Large unsigned values are not treated as negative signed values, and the shown decision does not depend on subtraction overflow.

## Side effects

### Directly visible effects

The function:

- Reads each category byte once in the initial category-selection stage.
- Uses two separate 512-byte temporary regions on the equal-category path.
- Passes those regions to dependencies in the order described above.
- Directly reads the numeric fields only after the opaque comparison returns zero.
- Performs no visible direct write to either input record.
- Performs no visible explicit heap allocation or deallocation.
- Exposes no explicit output other than its integer return value.

The two temporary regions account for 1,024 bytes of declared local storage. This does not establish the total stack-frame size.

### Dependency-mediated effects

No purity or preservation guarantee is established for either dependency. They may have effects not visible in this function’s body.

In particular, the numeric comparisons occur after all three dependency calls. If those calls modify the records’ numeric fields, the later comparisons use the resulting accessible contents, not values captured before the calls.

The function does not visibly transfer ownership of either record or invoke a record-release operation. However, ownership behavior inside the dependencies is unknown; their calls must not be assumed to preserve or release memory.

The temporary regions have local invocation lifetime. The function provides no visible mechanism to keep them alive after returning. Whether dependencies retain their addresses, and whether such retention is permitted, is unknown.

If a dependency does not return normally, the evidence establishes no recovery, cleanup protocol, or substitute ordering result.

## Dependencies

| Opaque function ID | Established invocation | Return handling | Unknown contract |
|---|---|---|---|
| F-b3dff379655a | Called twice for equal categories: first record and first 512-byte region, then second record and second region. | Return values are not visibly inspected. | Record accesses, write bounds, representation format, initialization extent, global effects, ownership effects, and failure behavior. |
| F-2f4a2d2fe72a | Called after both preparation calls with the two regions in record order and integer argument `27`. | Nonzero is returned unchanged; zero enables the numeric tie-breaker. | Meaning of `27`, accessed extent, comparison semantics, mutation behavior, global effects, and failure behavior. |

The enclosing function shows neither a preparation-success check nor an explicit fallback for unusable prepared data. Safe execution therefore depends on dependency contracts that are not supplied.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not reported runtime results.

### Category precedence

Given a first category byte of `0x40` and a second category byte of `0x00`, the result must be `-1`. Neither dependency is called, and the numeric fields are not directly accessed by this function.

Reversing those category bits must produce `1` with the same dependency suppression.

A byte of `0xc0` has the same category effect as `0x40`; a byte of `0x80` has the same category effect as `0x00`.

### Equal categories preserve preparation order

Given two set category bits, or two clear category bits, the first preparation call receives the first record and its temporary region. The second receives the second record and a distinct temporary region. The comparison dependency is called only after both preparation calls return.

### Nonzero comparison results are preserved

Given equal categories and normally returning preparation calls:

- If F-2f4a2d2fe72a returns `-7`, the function returns `-7`.
- If it returns `12`, the function returns `12`.

In both cases, the function does not proceed to its direct numeric-field comparisons.

### Unsigned boundary behavior

Given equal categories, a zero opaque comparison result, and stable numeric fields:

| First numeric value | Second numeric value | Required result |
|---:|---:|---:|
| `0x00000000` | `0xffffffff` | `-1` |
| `0x80000000` | `0x7fffffff` | `1` |
| `0xffffffff` | `0xffffffff` | `0` |

These cases distinguish unsigned ordering from signed ordering.

### Dependency mutations precede the tie-breaker

If dependencies return normally, the comparison result is zero, and dependency activity changes the numeric fields to stable values of `9` and `3`, the numeric result is `1`, regardless of their pre-call contents.

### Identical references do not guarantee zero

If both inputs reference the same record, the categories match. The function still performs both preparation calls and the opaque comparison.

A zero result is required only under additional conditions: the opaque comparison returns zero, and the numeric accesses observe equal values. Pointer identity alone does not establish these conditions.

## Uncertainties

The following are **documented_only**:

- Complete record layout, allocation size, and semantic meaning of the two accessed fields.
- Target byte order and precise alignment requirements for the 32-bit field accesses.
- Dependency implementations, error conventions, and memory-safety requirements.
- Whether the preparation dependency fully initializes all storage later consumed.
- The interpretation of the comparison dependency’s third argument.
- Whether either dependency modifies records, temporary regions, globals, or ownership state.
- Behavior under concurrent mutation, asynchronous memory changes, invalid addresses, or dependency failure.
- Exact machine-level read scheduling and total stack usage.
- Whether the overall function satisfies comparator properties such as reflexivity, antisymmetry, or transitivity.

The apparent intent is category-first ordering, followed by an opaque prepared-representation comparison and an unsigned numeric tie-breaker. No more specific application meaning is established.

## Evidence

This contract is based solely on the supplied static function body and opaque dependency mapping.

Established observations include the two byte-mask category decisions, category constants `1` and `4`, two distinct 512-byte temporary regions, dependency call order, comparison argument `0x1b`, exact propagation of nonzero comparison results, and unsigned numeric comparisons at offset `0x2c`.

The evidence does not include dependency bodies, a complete record definition, an execution trace, or independent platform documentation. No runtime tests have been performed.
