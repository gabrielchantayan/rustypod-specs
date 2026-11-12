<!-- Trusted provenance: F-fccd908ac67e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fccd908ac67e

## Purpose

F-fccd908ac67e is a globally directed state machine operating on a 32-bit base address and several persistent working values. Its observable behavior can include obtaining a replacement base from a dependency, passing stored words to other dependencies, clearing those words after normal dependency return, transforming a control word, and writing individual zero bytes at computed addresses.

The supplied static evidence establishes these operations and their ordering constraints. It does not establish a fixed sequence through them. Global values governing dispatch are unavailable, so acquisition before use, successful completion, termination, and a particular number of memory operations are not guaranteed.

Allocation, resource destruction, and buffer initialization are plausible interpretations, not established contracts. This specification assigns no such semantics to the dependencies.

## Interface

### Input and result

The sole explicit argument is a 32-bit integer. Selector 34 uses its bit pattern in a multiplication. No visible argument validation precedes that use.

Selector 4 returns a represented 64-bit aggregate:

- The lower 32 bits are a result accumulator, initially zero and replaceable by selector 25.
- The upper 32 bits come from a stack-derived word with no visible initialization or assignment.

The upper word’s value and ABI meaning are **documented_only**. It must not be specified as zero, an error indicator, or preserved caller data.

There is no other explicit return operation. Whether selector 4 is reached is unresolved.

### Arithmetic and access widths

The working addresses, field values, state encodings, and arithmetic operands are 32-bit quantities. Arithmetic results and address calculations retain their low 32 bits. Bitwise operations act on 32-bit patterns; arithmetic right shifts propagate bit 31. A negative decision is a result whose bit 31 is set.

These are machine-width requirements of the supplied representation, not a claim about portable signed arithmetic. Byte order, alignment requirements, and atomicity are not supplied.

Field operations use 32-bit loads or stores. The computed-address clearing operation writes exactly eight bits.

### Required environmental conditions

Every reached load or store requires an address valid for that access width. The base starts at zero, and no unconditional acquisition-before-access guarantee is visible. A zero dependency result does not establish an immediate return or prevent subsequent memory accesses.

Arguments passed to dependencies must satisfy their contracts, which are unavailable. In particular, zero arguments are not known to be acceptable.

## Behavior

### Dispatch model and persistent values

Selector numbers in this specification are relative dispatch selectors, not absolute state values. Each iteration obtains its selector by subtracting a freshly read dispatch-base global from the current state word, using 32-bit arithmetic.

Before that dispatch-base read, every iteration reads a fallback-transition global. This read occurs even for an operation that replaces the fallback or returns immediately. Selector 36 uses the fallback value captured for its own iteration.

The routine captures one coefficient at entry, called \(K\), and does not reread it. The initial-state global is read after initialization of the base, arithmetic temporary, two byte-address values, and return accumulator, but before initialization of the factor and index.

The following persistent roles are distinct:

| Role | Initial value | Selectors capable of replacing it |
|---|---:|---|
| Object base | 0 | 29 |
| Arithmetic temporary | 0 | 8, 10, 21, 24, 25, 29, 34 |
| Byte index | 0 | 7, 24 |
| Auxiliary address | 0 | 36 |
| Byte-write base | 0 | 21 |
| Factor | 0 | 12, 16, 21, 24 |
| Lower return accumulator | 0 | 25 |

These roles must not be collapsed. For example, selector 24 replaces the factor with a byte offset, and selector 29 consumes the latest arithmetic temporary, regardless of which earlier operation produced it.

A second coefficient global, \(C\), is shared by selectors 34, 8, the transforming branch of 10, and 24. Unlike \(K\), it is read when used. Separate occurrences of a global read remain separate observations; equality of their values is not guaranteed.

### Object access layout

Offsets are byte displacements from the current object base.

| Offset | Established access |
|---|---|
| `0x08` | Dependency argument and subsequent 32-bit clearing |
| `0x0c` | Control-word loads and possible 32-bit replacement |
| `0x28` | 32-bit zero store |
| `0x2c` | Address formation only |
| `0x60`, `0x64`, `0x68`, `0x6c`, `0x70` | Separate word tests, dependency arguments, and clearing |
| `0xa0`, `0xa4`, `0xa8`, `0xac`, `0xb0`, `0xb4` | Separate word tests, dependency arguments, and clearing |

This table is not a complete object declaration. It establishes neither a word load at `0x2c` nor a capacity for the region beginning there.

### Dependency processing and field clearing

The following operations share one observable contract:

| Selector | Field offset | Dependency |
|---:|---:|---|
| 0 | `0x70` | F-1c7acd1b0122 |
| 5 | `0xa0` | F-0cfbf9d2c168 |
| 11 | `0x60` | F-1c7acd1b0122 |
| 14 | `0x08` | F-5e3309c9d083 |
| 17 | `0xac` | F-0cfbf9d2c168 |
| 18 | `0xb0` | F-0cfbf9d2c168 |
| 20 | `0x68` | F-a8db34e89b44 |
| 26 | `0x64` | F-7edad8af3735 |
| 27 | `0xa8` | F-0cfbf9d2c168 |
| 30 | `0xa4` | F-0cfbf9d2c168 |
| 35 | `0x6c` | F-61ca0dee2b44 |
| 9 and every unmatched selector | `0xb4` | F-0cfbf9d2c168 |

For each row, the field’s loaded 32-bit value is the dependency argument. The required order is: argument load, dependency call, normal return, transition-global read, then 32-bit zero store to the field.

There is no internal nonzero prerequisite. A reached operation passes zero if its field contains zero. Dependency results are not visibly consumed.

If a dependency does not return normally, the later transition read and clearing store are not established. If it changes the field before returning, the subsequent zero store still targets that field. If it invalidates the underlying storage, safety of the later store is not established.

Unmatched selectors therefore trigger a dependency operation, not an explicit invalid-state error.

### Field-dependent transitions

The ordinary field tests have the following transition parameters:

| Selector | Tested offset | Displacement for zero |
|---:|---:|---:|
| 1 | `0x6c` | −33 |
| 6 | `0x60` | −10 |
| 13 | `0xac` | +6 |
| 15 | `0xa4` | +2 |
| 19 | `0xa8` | −12 |
| 23 | `0xb0` | +13 |
| 31 | `0xb4` | +27 |
| 32 | `0xa0` | +8 |
| 33 | `0x64` | −20 |

The field load precedes two reads of the operation’s transition global. If those reads yield \(t_1\) and \(t_2\), a zero field selects \(t_1\) plus the listed displacement; a nonzero field selects \(t_2\). Both reads occur for either outcome. These operations do not modify the tested field.

Selector 2 has a different contract. After testing `0x70`, it reads the dispatch-base global and then a separate transition global shared with selector 0. Nonzero selects the newly read dispatch base; zero selects the separate transition value. Both global reads occur regardless of the field value.

### Input scaling and base replacement

Selector 34 replaces the arithmetic temporary with the low 32 bits of the input multiplied by a current read of \(C\). Its transition-global read follows that computation.

Selector 29 multiplies the current arithmetic temporary by a separate global multiplier, retaining the product in the temporary. F-5e11a17b9827 receives that product and a second global argument. Its 32-bit result replaces the object base.

After the dependency returns, two reads of the acquisition-transition global occur. A zero result selects the **first** read plus 6; a nonzero result selects the **second** read without displacement.

Repeated acquisition operations repeatedly scale the current temporary. They do not necessarily rescale the original argument: control processing, byte processing, and result computation can all replace that temporary first.

The result’s interpretation as an allocation or acquisition is inferred intent. A zero result is only an established transition condition, not a defined error code.

### Control-word behavior

Selector 10 first captures the word at `0x0c` in the arithmetic temporary and then reads the control-processing transition global. That read precedes classification of the captured word.

- A captured zero leaves the temporary at zero, performs no control-word store, and uses the captured transition.
- A captured one leaves the temporary at one and instead tests `0x68`. Two reads of another transition global follow that field test. Zero selects the first read plus 13; nonzero selects the second. The earlier control-processing transition read still occurred.
- Any other captured value invokes the transformation described below.

Selector 8 invokes the same transformation directly. On selector 10’s transforming branch, `0x0c` is loaded again: the transformation uses this second load, not necessarily the value that selected the branch.

Let \(w\) be the transformation’s field load. Three occurrences of reads from \(C\) supply values identified by operand role as \(c_p\), \(c_x\), and \(c_s\). The transformed word is

\[
K\left(c_p w+
\left(c_x\mathbin{\mathrm{xor}}\operatorname{ashr}_{32}(c_s,30)\right)\right)
\pmod {2^{32}}.
\]

The operand labels do not impose a relative evaluation order. They also do not require the three reads to agree.

The transformed word becomes both the arithmetic temporary and the replacement word at `0x0c`. The store precedes a new read of the control-processing transition global. Selectors 8 and 10 share that global.

### Auxiliary region and byte decision

Selector 36 forms the auxiliary address as object base plus `0x2c`, then stores a 32-bit zero at `0x28`. Its next state is the fallback transition already read at the beginning of that iteration.

Selector 7 resets the byte index to zero before reading its transition. Selector 24 shares that transition global.

Selector 21 computes a 32-bit decision from the current index and five global operands. To specify the arithmetic without assigning semantic meanings to those globals, let their read values be \(a,d,e,f,g\), let \(n\) be the index, and abbreviate \(x=na\). The decision is

\[
\begin{aligned}
\Delta(n)={}&-21-\bigl((x-1)\mathbin{\mathrm{or}}\mathtt{0xffffffeb}\bigr)\\
&+\Bigl(
\bigl[x+18-
  \bigl((x+d+((ne+1)\mathbin{\mathrm{or}}f))
       \mathbin{\mathrm{or}}(x+19)\bigr)\bigr]
\mathbin{\mathrm{xor}}
\operatorname{not}\bigl((x+19)\mathbin{\mathrm{or}}g\bigr)
\Bigr).
\end{aligned}
\]

Every intermediate is a 32-bit pattern. The multiplication represented by \(x\) precedes the remaining compound expression; additional operand-read ordering within that expression is unresolved.

The decision replaces the arithmetic temporary. Two reads of the decision-transition global follow. A nonnegative decision selects the first read; a negative decision selects the second read plus 4. Both reads occur in either case.

After these reads, the factor receives the decision and the byte-write base receives the auxiliary address. Thus both the temporary and factor retain the decision, independently of the selected sign branch.

No conventional length comparison or terminating-loop condition is established by this expression.

### Byte clearing and index replacement

For selector 24, let \(n\) be the index on entry and \(h\) a read from a separate subtraction-operand global. Its resulting factor and arithmetic temporary satisfy

\[
\text{factor}=Kn,\qquad
\text{temporary}=CK(n-h)\pmod {2^{32}}.
\]

The factor is replaced before the temporary. The coefficient and subtraction-operand reads required for the temporary occur before the memory store; their relative order is unresolved.

Exactly one zero byte is written at byte-write base plus the new factor. The transition-global read follows the store, and index replacement with the temporary follows that read.

Consequently, this operation may change three persistent arithmetic roles while performing one byte store. It does not establish contiguous clearing, a bounded address range, or a particular iteration count.

### Result production and transition-only operations

Selector 12 sets the factor to zero before reading its transition.

Selector 16 reads that same transition global first, then replaces the factor from a separate global. Reversing those reads would change the established ordering.

Selector 25 places \(K\) times the current factor in the arithmetic temporary, reads its transition global, and only then updates the lower return accumulator from the temporary. A factor previously replaced by selector 24 is therefore used as a byte offset value, not as an earlier decision or status.

Selectors 3, 22, and 28 each obtain their next state from their respective transition globals, without another state-specific effect.

Selector 4 returns the represented aggregate. Unless selector 25 has executed, its lower word remains zero. Its upper word remains unspecified by the supplied evidence.

## Side effects

Established side effects comprise dependency calls, 32-bit field clearing, control-word replacement, and individual zero-byte stores. No explicit global store appears in the supplied representation.

Dependency calls may themselves change globals, object memory, ownership, or process state. None of those effects may be ruled out. In particular, a transition read after normal dependency return observes the environment after that call, not an assumed preserved environment.

A field clearing operation does not prove resource release. Passing a stored word to a dependency does not establish ownership transfer or permission to destroy the referenced object.

Effects are not transactional. Earlier calls and stores can remain observable when a later access fails or a dependency does not return. No explicit rollback, bounds enforcement, exception handling, or general failure-reporting protocol is established.

## Dependencies

| Opaque function ID | Visible use | Unresolved properties |
|---|---|---|
| F-5e11a17b9827 | Two 32-bit arguments; 32-bit result replaces the base | Meaning of arguments, result validity, ownership, failure behavior |
| F-5e3309c9d083 | One 32-bit argument from `0x08` | Operation, argument requirements, ownership effects |
| F-1c7acd1b0122 | One 32-bit argument from `0x60` or `0x70` | Operation, argument requirements, ownership effects |
| F-7edad8af3735 | One 32-bit argument from `0x64` | Operation, argument requirements, ownership effects |
| F-a8db34e89b44 | One 32-bit argument from `0x68` | Operation, argument requirements, ownership effects |
| F-61ca0dee2b44 | One 32-bit argument from `0x6c` | Operation, argument requirements, ownership effects |
| F-0cfbf9d2c168 | One 32-bit argument from a reached field between `0xa0` and `0xb4` | Operation, argument requirements, ownership effects |

No dependency is assumed to preserve globals, tolerate zero, release storage, or return normally.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Each assumes the named selectors are reached and all relevant accesses are valid.

1. **Call before clearing.** With value \(v\) at `0x68`, selector 20 passes \(v\) to F-a8db34e89b44. After normal return, the transition read precedes the zero store. Clearing before the call or before that read violates the required order.

2. **No implicit nonzero guard.** Selector 17 with zero at `0xac` still calls F-0cfbf9d2c168 with zero. A test elsewhere in the state machine does not create a guard inside this operation.

3. **Distinct transition observations.** If selector 1’s transition reads yield 100 and 200, a zero field selects 67 and a nonzero field selects 200. Both reads occur in either case.

4. **Selector 2 observes both alternatives.** Even when `0x70` is zero, the dispatch-base read precedes the separate transition read. Only the latter determines the selected next state.

5. **Control-word reread matters.** If selector 10 first observes 2 at `0x0c`, but its transformation load observes 5, the transformation uses 5. The preliminary transition read has already occurred, and another transition read follows the replacement store.

6. **Byte arithmetic propagates.** With \(K=3\), index 2, the coefficient read equal to 2, and subtraction operand 1, selector 24 produces factor 6 and temporary 6. It writes one zero byte at byte-write base plus 6, reads its transition, then replaces the index with 6. An ensuing selector 25 produces lower result 18.

7. **Acquisition consumes the current temporary.** Immediately after the preceding selector 24, acquisition multiplier 5 produces first dependency argument 30. If selector 25 has instead replaced the temporary with 18, the argument becomes 90.

8. **Zero-result branch uses the first read.** If F-5e11a17b9827 returns zero and subsequent transition reads yield 100 and 200, selector 29 selects 106, not 206. A nonzero result selects 200. Neither case alone establishes eventual return.

9. **Width truncation is observable.** With \(K=3\) and index `0x80000000`, selector 24’s factor is `0x80000000`, not an unbounded product. The byte address uses 32-bit addition with that factor.

10. **Return and unexpected dispatch.** Reaching selector 4 before selector 25 yields lower word zero without defining the upper word. Reaching an unmatched selector instead performs the `0xb4` call-and-clear operation; it does not report an invalid selector.

## Uncertainties

The following remain **documented_only**:

- Global contents, absolute state encodings, reachable paths, lifecycle order, and termination.
- Stability of globals or object fields across separate reads.
- Object size, alignment, byte-region capacity, and address validity.
- Dependency semantics, ownership changes, global preservation, and abnormal outcomes.
- Intended meanings of the control word, arithmetic factor, input scaling, and acquisition arguments.
- Evaluation order inside compound expressions beyond the established ordering constraints.
- Actual ABI interpretation of the represented return aggregate and its uninitialized upper source word.

The evidence does not justify replacing these unknowns with a conventional allocator, destructor, bounded initialization loop, or status-return convention.

## Evidence

**Established static evidence:** the supplied representation and opaque dependency mapping support the access widths and offsets, selector actions, persistent-value relationships, repeated global observations, ordered calls and stores, dispatch fallback behavior, and represented return construction described above.

**Inferred intent:** resource acquisition, resource cleanup, and initialization are possible interpretations only. They are not acceptance requirements.

**Unavailable evidence:** dependency bodies, global definitions, caller contracts, complete object layouts, and independent ABI confirmation were not supplied.

No runtime tests have been performed.
