<!-- Trusted provenance: F-eaf1de973a81; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eaf1de973a81

## Purpose

**Status: documented_only.** This contract describes the behavior established by the supplied static representation. Dependency semantics, global-data definitions, and parts of the calling convention remain unresolved. No runtime tests have been performed.

F-eaf1de973a81 conditionally replaces its first input word. A threshold comparison on the second input selects replacement directly; otherwise, a dependency decides whether replacement occurs. Once replacement is selected, the fifth input chooses between two dependency compositions.

The established behavior includes the gate, dependency-call ordering, intermediate word extraction, timing of global accesses, and visible 32-bit return. It does not establish a mathematical interpretation of the dependency operations.

**Inferred intent:** Repeated low/high word pairs and the sign-bit-clearing mask suggest processing of a multiword numeric representation. Neither a specific numeric format nor a particular mathematical function is established.

## Interface

### Inputs

| Position | Established representation | Role |
|---|---|---|
| First | 32-bit word | Original return candidate and an input to replacement operations. |
| Second | Unsigned 32-bit word | Controls the threshold gate and accompanies the first word in dependency calls. |
| Third | 32-bit word | Used only in the nonzero-selector replacement branch. |
| Fourth | 32-bit word | Accompanies the third word in that branch. |
| Fifth | Signed integer in the supplied representation | Zero selects one replacement branch; every nonzero value selects the other. Exact ABI width is not independently established. |

The first and second inputs form an ordered pair at dependency boundaries. The third and fourth inputs likewise form an ordered pair where used. This grouping does not prove a numeric encoding.

All original input words passed to dependencies retain their original bit patterns. In particular, masking the second input for the threshold comparison does not replace the second input supplied to subsequent calls.

### Output

The visible result is one **32-bit word**:

- If replacement is not selected, the result is exactly the original first input.
- If replacement is selected, the result is the 32-bit value assigned from the branch’s final dependency call.

Intermediate results are represented as 64-bit quantities, but the visible return is not a 64-bit result. Hidden result transport or a wider underlying ABI cannot be established from the supplied evidence.

### Intermediate layout

Whenever a dependency result is represented as 64 bits and subsequently supplied as two words:

- The first supplied word is bits 0–31.
- The second supplied word is bits 32–63.
- Extraction of the upper word uses an unsigned right shift by 32 bit positions.

The extracted words are passed as bit patterns. Signed-looking intermediate conversions do not establish a signed numeric interpretation of those patterns.

This layout specifies bit positions and argument order, not memory byte order or register assignment.

### Preconditions and ownership

No explicit input validation or range restriction is visible beyond the replacement gate.

No explicit allocation, release, pointer dereference, caller-memory write, or ownership transfer appears in the function itself. Dependency arguments may nevertheless have address-like meanings under contracts that are not supplied.

Normal completion requires the invoked dependencies and their required state to support the calls described here. Their validity requirements, memory ownership rules, and failure behavior are unknown.

## Behavior

### Replacement gate

The function clears bit 31 of the second input with **0x7FFFFFFF**, then compares the resulting unsigned value against **0x3E3FFFFF**.

Replacement is selected when either:

1. The masked value is strictly greater than **0x3E3FFFFF**; or
2. The masked value is at most **0x3E3FFFFF**, and **F-baa332e06e32** returns a nonzero result when called with the original first and second input words.

Evaluation is short-circuited:

- Above the threshold, **F-baa332e06e32** is not called.
- At or below the threshold, it is called once.
- If that call returns zero, the function returns the original first word without executing replacement operations or the explicit global accesses described below.

Equality with the threshold therefore requires the predicate call. Any nonzero predicate result selects replacement; it need not equal one.

The third, fourth, and fifth inputs do not participate in the gate. The fifth input matters only after replacement has been selected.

### Shared replacement operations and global-base timing

Every replacement path begins with these operations, in this order:

1. **F-c4b34006a13c** receives the original first and second words. Its represented 64-bit result supplies the **first shared pair**.
2. **F-c4b34006a13c** receives the original first and second words followed by the first shared pair. Its represented 64-bit result supplies the **second shared pair**.
3. A global base value is read. The first argument for **F-7fd6c3e1c305** is formed by adding the fixed constant **0x08033E54** to that value.
4. **F-7fd6c3e1c305** receives that global-derived argument, the integer **5**, and the first shared pair. Its represented 64-bit result supplies the **auxiliary pair**.

Each pair is supplied low word first, high word second.

The global-base read occurs **after both shared F-c4b34006a13c calls have returned**, not at function entry. If the gate invokes **F-baa332e06e32**, that predicate call also precedes the read. These dependencies have unknown side effects and may modify the base before it is read.

The offset **0x08033E54** is established. The base’s declaration, the addition’s expression type, the result width, overflow behavior, and any address interpretation are not established. This contract does not assume 32-bit modular addition or any particular pointer arithmetic.

The selector-specific operations begin after **F-7fd6c3e1c305** returns.

### Zero-selector replacement

When the fifth input equals zero, the branch performs the following dependency composition, in order:

1. **F-c4b34006a13c** receives the first shared pair followed by the auxiliary pair.
2. Two separate global words are read for the next call. **F-1b22d7e17791** receives the pair produced by the preceding call, followed by those two global words in their designated argument positions.
3. **F-c4b34006a13c** receives the pair produced by that call, followed by the second shared pair.
4. **F-1b22d7e17791** receives the latest pair followed by the original first and second input words.

The value retained from the final call becomes the visible 32-bit return.

The two branch-specific global reads occur **after the branch’s initial F-c4b34006a13c call has returned**, during preparation of the following **F-1b22d7e17791** call. They also follow all shared replacement calls.

The supplied representation does not establish which of these two global arguments is read first. Their argument positions are established; their relative evaluation order is not. They must not be treated as an atomic snapshot or as values necessarily captured together.

The third and fourth inputs are unused in this branch.

### Nonzero-selector replacement

When the fifth input is nonzero, the branch performs these operations, in order:

1. Two separate global words are read for the branch’s first **F-c4b34006a13c** call. That call receives the second shared pair followed by the two global words. Its result is the **global-related pair**.
2. **F-c4b34006a13c** receives the second shared pair followed by the auxiliary pair. Its result is the **auxiliary-related pair**.
3. **F-9f634613286e** receives the original third and fourth words followed by the 32-bit bit pattern **0xFFFFFFFF**. Its result supplies another pair.
4. **F-8a693c103d1c** receives that last pair followed by the auxiliary-related pair.
5. **F-c4b34006a13c** receives the resulting pair followed by the first shared pair.
6. **F-8a693c103d1c** receives the resulting pair followed by the original third and fourth words.
7. **F-8a693c103d1c** receives the resulting pair followed by the previously retained global-related pair.
8. **F-387e686c467e** receives the resulting pair followed by the original first and second input words.

The value retained from the final call becomes the visible 32-bit return.

Here, the two branch-specific global reads occur **after F-7fd6c3e1c305 returns**, for the first call in this branch. No selector-specific dependency call precedes them. As in the zero-selector branch, their relative evaluation order is not established.

Later calls use the retained global-related result; the visible function does not reread those two global words for the seventh operation.

Every nonzero selector value chooses this composition. The selector itself is not forwarded to a dependency.

### Required ordering and errors

Global-read timing is part of the contract. An entry-time snapshot, or hoisting a global read across an earlier dependency call, is not justified: that dependency may change the value subsequently read.

No visible error handling surrounds dependency calls. There is no wrapper-level retry, fallback, error-result interpretation, or restoration of external state. If a dependency fails to return normally, the evidence establishes no alternative return.

Overflow, rounding, exceptional numeric values, and domain errors inside dependencies remain unknown.

## Side effects

### Established static behavior

- The function performs no explicit caller-memory or persistent-state write.
- Replacement paths read a global base and two branch-specific global words.
- The base is read once at the stated shared-operation boundary.
- Each branch visibly accesses the two other global words once each for its designated call.
- Dependency calls occur in the order described above.
- A bypass performs no explicit global access in the visible function, although the predicate may access external state.
- The fifth input controls only branch selection and is not passed onward.

The visible global accesses do not establish the globals’ storage qualifiers, synchronization properties, or atomicity.

### Unknown effects

Any invoked dependency may read or modify memory, including the globals later accessed by this function. It may also affect numeric status, error state, devices, or other shared state.

No dependency is assumed to preserve globals, preserve external state generally, or acquire or release ownership. Consequently, purity, thread safety, reentrancy, and freedom from externally visible side effects are not established.

## Dependencies

All dependency contracts are unresolved. The following table records only their visible use.

| Opaque ID | Established use |
|---|---|
| **F-baa332e06e32** | Receives the original first and second words at or below the threshold. Its result is tested only for zero versus nonzero. |
| **F-c4b34006a13c** | Produces results represented as 64 bits. Its initial call visibly supplies two words; later calls supply four words grouped as two ordered pairs. |
| **F-7fd6c3e1c305** | Receives the global-derived argument using offset **0x08033E54**, the integer **5**, and the first shared pair. Produces a represented 64-bit result. |
| **F-1b22d7e17791** | Called twice in the zero-selector branch with two pairs. The first result is used as a represented 64-bit quantity; the final call supplies the retained 32-bit return value. |
| **F-9f634613286e** | Receives the third and fourth input words and **0xFFFFFFFF**. Produces a represented 64-bit result. |
| **F-8a693c103d1c** | Called three times in the nonzero-selector branch. Each call receives two pairs and produces a represented 64-bit result. |
| **F-387e686c467e** | Final call in the nonzero-selector branch. Receives an intermediate pair and the original first and second words; supplies the retained 32-bit return value. |

The different visible argument counts for **F-c4b34006a13c** are unresolved. They do not establish a variable-argument interface or authorize inventing missing arguments.

The argument **0xFFFFFFFF** is established as a 32-bit bit pattern. Its interpretation as signed negative one, a scaling parameter, or a mode is not established.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

### Threshold equality and bypass

Given a first word of **0x12345678**, a second word of **0x3E3FFFFF**, and a zero result from **F-baa332e06e32**:

- The predicate is called once with those original words.
- The return is **0x12345678**.
- No replacement dependency or explicit global read occurs.
- The remaining inputs do not change that bypass behavior.

### Masked sign bit

For a second word of **0xBE3FFFFF**, the masked comparison value is **0x3E3FFFFF**.

The predicate must be called with **0xBE3FFFFF**, not with the masked value. A zero predicate result returns the first word unchanged. This does not imply that the predicate behaves identically for **0xBE3FFFFF** and **0x3E3FFFFF**.

### First value above the threshold

For a second word of **0x3E400000**:

- Replacement is selected without calling the predicate.
- A zero selector chooses the zero-selector branch.
- A nonzero selector chooses the other branch.

A second word of **0xBE400000** makes the same threshold decision, without establishing equal dependency outputs.

### Predicate-forced replacement

For a second word of **0x00000000**, any representable nonzero predicate result selects replacement. No replacement output can be prescribed without the subsequent dependency results.

### Global-base mutation before its read

Suppose the second shared **F-c4b34006a13c** call changes the global base and returns normally.

The following **F-7fd6c3e1c305** argument must be derived from the base read after that call, with offset **0x08033E54**. A value captured at entry or before the shared calls is not an acceptable substitute. The concrete addition result remains conditional on unresolved expression semantics.

### Branch-specific global mutation

In the zero-selector branch, suppose its initial **F-c4b34006a13c** call changes one of the two global words. The following **F-1b22d7e17791** call uses the subsequent read, not a value saved before that call.

In the nonzero-selector branch, changes made by **F-7fd6c3e1c305** precede the global-word reads for the first branch call.

Neither example imposes an order between the two global-word reads.

### Selector comparison across executions

For fixed first through fourth inputs, selector values **1** and **−1** choose the same branch and argument-construction rules.

They do **not**, by themselves, guarantee equal concrete argument sequences: intermediate arguments depend on dependency results and on globals read during execution. Equal argument sequences and equal returns additionally require equivalent relevant external state and dependency behavior.

### Word extraction

A represented intermediate result of **0x1122334455667788** is subsequently supplied as **0x55667788** followed by **0x11223344**. This requirement says nothing about memory byte order.

## Uncertainties

The following remain **documented_only**:

1. **Numeric meaning:** the encoding of paired words and the mathematical purpose of each operation.
2. **Dependency contracts:** valid inputs, output semantics, side effects, failures, and ownership behavior.
3. **Initial argument count:** the two-word initial use of **F-c4b34006a13c** versus its later four-word uses.
4. **Global definitions:** initial values, storage declarations, access qualifiers, and synchronization.
5. **Base-expression semantics:** the offset is known, but expression typing, arithmetic width, overflow behavior, and address interpretation are not.
6. **Global-argument evaluation order:** neither branch establishes which of its two global-word arguments is evaluated first.
7. **ABI details:** exact selector width, hidden arguments, multiword result transport, and any wider return convention.
8. **Runtime behavior:** no execution or interoperability evidence is supplied.

These gaps do not erase the established call boundaries or permit moving global reads across them.

## Evidence

The supplied static body of **F-eaf1de973a81** establishes:

- The unsigned masked threshold comparison and short-circuit predicate.
- Preservation of original input words at dependency boundaries.
- Shared and selector-specific dependency ordering.
- Low/high extraction from represented 64-bit results.
- The fixed offset **0x08033E54** and the base read after both shared **F-c4b34006a13c** calls.
- Branch-local global-word reads at their respective call boundaries.
- The absence of an established relative evaluation order for those two global arguments.
- The visible 32-bit return and absence of explicit wrapper-level writes or error handling.

The opaque dependency IDs identify call targets, not independently established behavior. Numeric intent remains inference; unresolved contracts remain explicit unknowns. No runtime tests have been performed.
