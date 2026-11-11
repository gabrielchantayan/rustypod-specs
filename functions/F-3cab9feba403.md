<!-- Trusted provenance: F-3cab9feba403; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3cab9feba403

## Purpose

F-3cab9feba403 consumes a dependency-provided sequence of integer values and selectively forwards literal values or expands recognized percent-prefixed selectors into an output destination.

The established behavior resembles generation of formatting alternatives rather than ordinary date/time formatting: several selectors emit the fixed text `0123456789`, while others emit groups of resolved table entries. That interpretation is inferred intent, not an interface guarantee.

A percent value enters a persistent selector state. Unsupported values in that state are discarded without leaving it. A second percent does not produce a literal percent. Processing ends when the sequence reader returns zero.

## Interface

The function accepts five arguments, each represented by a 32-bit value:

| Argument | Established use |
|---|---|
| First | Not directly used by this function. |
| Second | Passed to the initialization dependency and every table-entry resolver call. Its semantic type is unknown. |
| Third | Used as the base address of a record containing 32-bit entries. Also copied into a separate 32-bit temporary whose address is passed to initialization. |
| Fourth | Passed as the output destination to both output dependencies. Its semantic type and ownership are unknown. |
| Fifth | Passed to initialization without direct interpretation. |

The function has no established return value. Its normal completion consists of returning when the sequence reader yields zero. It does not return an output length, success flag, or error code.

Two addressable 32-bit temporaries participate in the dependency interface:

- An initialization temporary initially contains the third argument. Initialization receives its address and may modify it. Subsequent record accesses still use the original third argument, not this temporary.
- A sequence-state temporary receives the initialization result. The sequence reader receives its address on every read and may update it.

The fourth argument is also initially assigned to the sequence-state temporary, but that value is overwritten by the initialization result before the first reader call. It is not supplied to initialization through that temporary.

### Record layout

All listed entries occupy four bytes. Offsets are byte offsets from the original third argument.

| Selector | Entry count | Entry offsets, in order |
|---|---:|---|
| `B` | 12 | `0x00`, `0x04`, …, `0x2c` |
| `b` | 12 | `0x30`, `0x34`, …, `0x5c` |
| `A` | 7 | `0x60`, `0x64`, …, `0x78` |
| `a` | 7 | `0x7c`, `0x80`, …, `0x94` |
| `p` | 2 | `0x98`, `0x9c` |

The highest accessed entry occupies offsets `0x9c` through `0x9f`. A record supporting every selector therefore needs accessible storage through byte offset `0x9f`. The function does not establish that the entries are addresses, indices, or any particular application-level type; it passes each as a 32-bit value to a resolver.

Only selectors actually processed cause these record reads. A sequence containing no table-expansion selector does not directly read the record through the third argument, although initialization still receives its copied value.

## Behavior

### Initialization and read order

Before reading the sequence, the function captures a 32-bit value from a global location for later resolver calls.

It then invokes F-838218ef65c4 with, in order:

1. The second argument.
2. A value read from that same global location for this initialization call.
3. The fifth argument.
4. The address of the initialization temporary containing the third argument.

The captured global value and the value supplied to initialization originate from the same location, but they are separate reads in the supplied representation. Their equality is not an unconditional guarantee if that location can change between reads.

The initialization result replaces the sequence-state temporary. The function calls F-d3922c7fe0ec with the address of that temporary to obtain the first sequence value.

The selector state initially is inactive. After each nonterminating value is processed, the function calls F-d3922c7fe0ec again with the same temporary’s address. Expansion output completes before the next sequence read.

### Termination

A reader result of zero causes immediate return, whether selector state is active or inactive.

Consequences:

- Zero is a terminator, not literal output.
- A trailing percent produces no output of its own.
- An unfinished selector state produces no reported error.
- There is no explicit flush, final separator, or cleanup call on normal return.

No special recovery is established for an initializer result that a caller might consider invalid. Whatever 32-bit value initialization returns is placed in the reader state and supplied to the reader.

### Literal mode

When selector state is inactive:

- A reader value of decimal 37, corresponding to `%`, activates selector state and produces no output.
- Any other nonzero reader value is forwarded to F-008d77a63c85 with the fourth argument as destination, the reader value as the output value, and an explicit third argument of `1`.

The function performs no visible restriction to an eight-bit character range. Nonzero negative values and positive values outside the ordinary character range are not independently rejected; outside selector state they follow the literal-output path. How the output dependency interprets such values is unknown.

### Selector mode

Every percent value activates or preserves selector state before any selector dispatch. Therefore a percent encountered while already in selector state does not escape a percent or complete the selector.

For other nonzero values while selector state is active:

| Selector | Action | State afterward |
|---|---|---|
| `_` | Invoke F-008d77a63c85 with destination and value `0x20`, representing a space. | Inactive |
| `A` | Resolve and emit seven entries starting at offset `0x60`. | Inactive |
| `a` | Resolve and emit seven entries starting at offset `0x7c`. | Inactive |
| `B` | Resolve and emit twelve entries starting at offset `0x00`. | Inactive |
| `b` | Resolve and emit twelve entries starting at offset `0x30`. | Inactive |
| `p` | Resolve and emit the entry at `0x98`, then resolve and emit the entry at `0x9c`. | Inactive |
| `H`, `I`, `J`, `M`, `S`, `U`, `W`, `Y`, `d`, `j`, `m`, `w`, `y` | Pass the fixed text `0123456789` to F-2484a607f41b. | Inactive |
| Any other nonzero value | No direct output or record access. | Active |

The dispatch contains no effective literal-percent expansion. Although the supplied representation contains a percent-output branch inside selector handling, the earlier unconditional percent handling prevents a reader value of 37 from reaching it.

Values such as `T`, `V`, `X`, `c`, and `x` do not complete a selector. Their behavior is the same as other unsupported nonzero values: discard the value and retain selector state. This is persistent scanning for a recognized selector, not merely consuming one character after each percent.

### Table expansion

For each entry in an array expansion, the observable operation order is:

1. Read that entry’s 32-bit value from the record.
2. Call F-e4e0c15c0e4c with the second argument, the previously captured global value, and that entry value.
3. Call F-2484a607f41b with the fourth argument and the resolver result.
4. Continue to the next entry, if any.

The function supplies no separators between entries and makes no direct empty-result or null-result checks.

Entries are read one at a time, not snapshotted as a group. If a dependency modifies a later entry before it is read, the later read uses the then-current contents. Similarly, the second `p` entry is read only after resolution and output of the first entry have completed.

Every resolver call uses the initially captured global value. The function does not refresh that value between entries or expansions.

### Arithmetic and bounds

Array entries are four bytes apart. Array indices run from zero through six or zero through eleven, respectively. These small index calculations introduce no intrinsic 32-bit overflow.

Address formation still requires a valid base and accessible memory. The function provides no explicit bounds checking, null check, alignment check, or protection against an invalid base-plus-offset address. Behavior for invalid addresses, machine-address overflow, or target-specific unaligned access is not established.

## Side effects

Direct output is delegated:

- F-008d77a63c85 handles ordinary literal values and the space expansion.
- F-2484a607f41b handles resolver results and the fixed digit text.

Literal output calls visibly include an explicit third argument of `1`. The space-output call exposes only the destination and value arguments. The available evidence does not establish a uniform full calling contract for F-008d77a63c85; the missing argument must not be assumed to equal `1`.

The function itself directly reads the global location and selected record entries. It does not directly modify either. It updates its own temporary state and selector flag.

Dependencies may have additional side effects. In particular, there is no evidence that they preserve globals, leave the record unchanged, avoid reentrant calls, or refrain from modifying the destination.

No direct allocation or release operation appears. This does not prove that dependencies allocate nothing. Ownership of the destination, record, initialization result, resolver results, and fixed-text argument is not transferred or retained by any explicit mechanism in this function. Dependency ownership rules remain unknown.

The addresses of the two temporaries are supplied to dependencies during calls. They are not caller-owned persistent state. No supported retention of those addresses beyond the invocation is established.

The function has no explicit output-failure handling. Output dependency return values are not inspected. An output operation’s failure does not create a visible local error branch; dependency-level failure, nonreturn, or exceptional behavior is outside the established contract.

## Dependencies

| Opaque ID | Established role and use |
|---|---|
| F-838218ef65c4 | Initialization. Receives the second argument, a global-derived value, the fifth argument, and an addressable copy of the third argument. Its 32-bit result becomes sequence state. |
| F-d3922c7fe0ec | Sequence reader. Receives the address of sequence state. Returns an integer interpreted as a value to process, with zero terminating the function. |
| F-008d77a63c85 | Literal/space output. Receives the fourth argument as destination. Ordinary literal calls also pass an explicit third argument of `1`; the space call’s complete argument contract is unresolved. |
| F-e4e0c15c0e4c | Entry resolver. Receives the second argument, the captured global value, and a 32-bit record entry. Its result is forwarded directly to F-2484a607f41b. |
| F-2484a607f41b | Output of resolver results or fixed digit text. Receives the fourth argument as destination. String interpretation, failure reporting, and ownership behavior are not established here. |

No dependency may be assumed to preserve global state or to own or release supplied memory merely from its role in this function.

## Acceptance examples

These examples are conditional on initialization completing, the reader returning the specified sequence, and dependencies returning normally. Quoted text denotes corresponding reader values followed by zero, not an established input encoding.

1. **Immediate termination.**  
   If the first reader result is zero, initialization and one reader call occur. No output or record-entry resolver call occurs.

2. **Ordinary literals.**  
   For `cat`, three literal-output calls occur in order, carrying `c`, `a`, and `t`, each with the explicit third argument `1`. The letters do not act as selectors without a preceding percent.

3. **Fixed digit expansion.**  
   For `%Y`, the percent produces no output. One F-2484a607f41b call receives `0123456789`. No record entry is read for that expansion.

4. **Repeated percent.**  
   For `%%Y`, the output-call behavior is the same as for `%Y`. Neither percent produces a literal percent.

5. **Unsupported selectors remain pending.**  
   For `%cXY`, `c` and `X` produce no output and do not reset selector state. `Y` emits the fixed digit text and resets it.

6. **Recognized selector restores literal mode.**  
   For `%YQ`, the digit-text output call occurs first. A literal-output call for `Q` follows.

7. **Space expansion.**  
   For `%_`, one space-output call occurs with value `0x20`. There is no fixed-text output call. The full ABI behavior of that call remains subject to the documented argument-count uncertainty.

8. **Seven-entry expansion.**  
   For `%A`, seven resolver/output pairs occur, reading entries at `0x60` through `0x78` in ascending four-byte steps. The function inserts no separator.

9. **Twelve-entry expansion.**  
   For `%b`, twelve resolver/output pairs occur, reading `0x30` through `0x5c` in order.

10. **Paired expansion and mutation visibility.**  
    For `%p`, the `0x98` entry is read, resolved, and output before the `0x9c` entry is read. If the first resolver or output operation changes the latter entry, the second resolver receives its changed value.

11. **Termination while pending.**  
    For `%T` followed by zero, neither `%` nor `T` produces output. Zero terminates processing without a fallback literal or reported incomplete-selector error.

12. **Initializer modification does not redirect record reads.**  
    If initialization changes the temporary containing the third argument, a later `%B` still reads from the original third-argument base.

## Uncertainties

The following are **documented_only**, not assumptions to complete the interface:

- The semantic types of the five arguments, the global value, sequence state, and record entries.
- The initialization and reader protocols, including whether the reader returns bytes, decoded characters, or other integer tokens.
- The exact target ABI and complete argument requirements of F-008d77a63c85.
- Whether F-2484a607f41b interprets every supplied value as a null-terminated string.
- Output buffering, capacity, termination, encoding, and failure behavior.
- Null resolver results and dependency-specific error representations.
- Dependency allocation, retention, release, reentrancy, and global mutation.
- Whether the record’s apparent month/day grouping reflects its actual application meaning.
- Whether the inaccessible percent-output branch reflects intended behavior. The established reachable behavior does not emit a percent for `%%`.

## Evidence

Established static evidence consists of the supplied function representation: argument uses, explicit 32-bit values and loads, dependency call order, record offsets, selector comparisons, state transitions, fixed digit text, and the zero-return termination path.

Formatting-related terminology is inferred intent only. No callee implementation or external interface contract was supplied.

No runtime tests have been performed. The acceptance examples specify conditional call-level behavior derived from static evidence; they are not observed executions.
