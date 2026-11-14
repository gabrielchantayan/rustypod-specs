<!-- Trusted provenance: F-4ce7ccdca398; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4ce7ccdca398

## Purpose

The supplied static representation describes an operation that interleaves unresolved indirect calls with ordered 8-bit and 32-bit memory accesses. Entry-state values and call results serve both as data and as address components. Some writes use low absolute addresses; others use destinations selected by bytes read from memory.

No application-level purpose is established. Descriptions such as initialization, allocation, serialization, or object management would add unsupported meaning.

The represented path ends at an invalid-instruction boundary, not a visible normal return. This contract specifies the visible execution prefix, conditional on reaching each event. It does not establish successful completion or instruction-level reachability.

## Interface

### Explicit inputs and output

There are two explicit 32-bit inputs: the **first argument** and **second argument**. Their signedness and application-level interpretation are unspecified.

No return value or normal-return behavior is established.

### Implicit inputs

Execution additionally depends on:

- An **entry base**, used for a word read at offset `0x74` and a byte write at offset `1`.
- An unsigned 32-bit **entry bitfield**.
- An **entry call anchor**, used to calculate the initial indirect-call destination.
- Secondary machine-register results exposed after calls.
- Accessible memory and executable indirect-call destinations.

This is not a complete calling convention. In particular, calls with no explicit arguments may still consume machine state.

### Widths and arithmetic

A **byte** is 8 bits; a **word** is 32 bits. Selected results are represented as 64-bit quantities with distinct low and high 32-bit halves.

Byte writes retain the low eight bits. Bytes used as selectors are zero-extended. Consequently, a destination formed by adding `0x11` to a byte lies between `0x11` and `0x110`, inclusive.

Unsigned 32-bit arithmetic discards bits above bit 31. Unsigned right shifts are logical. Masks and shifts apply in the stated order.

Some scaled addresses and shifted arguments instead involve signed-typed 32-bit intermediates. Their overflow, conversion, and negative-value shift behavior is not established by the supplied representation. A universal wrapping interpretation is therefore not part of this contract. Examples involving those operations require suitable values or independently established machine semantics.

Endianness, alignment constraints, pointer interpretation, address-space permissions, and hardware-register semantics remain unknown.

## Behavior

### Conditions and ordering

Every phase below is conditional on earlier calls returning and earlier accesses succeeding. No visible validation precedes the derived accesses or calls.

A group of four calls means four distinct, ordered calls with no explicit arguments—not one equivalent operation. Unless otherwise stated, “latest result” means the result of the last call in the immediately preceding group.

Unknown calls may change memory and other machine state. Repeated reads separated by calls cannot be replaced by one cached read. Arithmetic values explicitly retained by the represented path are distinguished from memory that may be reread.

### 1. Entry transfer and initial scaled access

Read the **entry word** from the entry base plus `0x74`. Write its low byte at the entry base plus `1`.

The initial indirect call receives, in order:

1. The entry word.
2. The second argument.
3. The second argument again.
4. The first argument.

A no-explicit-argument call supplies a word. The entry bitfield logically shifted right by 22 becomes the **entry fragment**, ranging from `0` through `1023`. A subsequent call receives the supplied word and the entry word logically shifted right by two.

After four no-explicit-argument calls, multiply the latest signed-typed result by four to obtain the **initial scaled base**. Read a word at that base plus `0x74`, then write its low byte at that base plus `1`. Call a dependency with the initial scaled base.

After four further no-explicit-argument calls, store the previously read word at four times a secondary register result plus `0x14`. Call a dependency with that secondary result. Its producing interface is unresolved.

### 2. Entry-fragment and scale-source transfers

A no-explicit-argument call supplies the first argument of a three-argument call. The other two arguments are identical: the entry fragment masked with `0x1ff`, then shifted left by 21.

After four no-explicit-argument calls:

- Store the entry fragment shifted left by 23 as a word at four times the latest result plus `0x14`.
- Call a dependency with four times that result.

After four more no-explicit-argument calls, retain the unsigned latest result as the **first scale source**. Then:

1. Retain four times that source as the **retained fourfold value**.
2. Store that value as a word at sixteen times the source plus `0x14`.
3. Mask the source with `0x3fffffff`, producing the **masked scale source**.
4. Retain the masked scale source logically shifted right by two as the **retained quarter**.
5. Call a dependency with the masked scale source.

After four no-explicit-argument calls, retain the latest result as the **index source**. Store the masked scale source at four times the index source plus `0x14`, then call a dependency with four times the index source.

### 3. Eightfold writes and the first shared-target call

After four no-explicit-argument calls, retain the unsigned latest result as the **record source**. Eight times the index source becomes the **eightfold base**.

Perform these writes in order:

1. Write the low byte of the eightfold base at the entry word plus `1`.
2. Write the record source shifted left by two as a word at the eightfold base plus `0x34`.

Call a dependency with the record source shifted left by two. A no-explicit-argument call supplies a word that is passed to another dependency. Retain the record source masked with `0x3fffffff` as the **masked record base**.

Four no-explicit-argument calls follow. The fourth is a call to **F-U01**, whose result is represented here as 64 bits. Then:

1. Write the low byte of the eightfold base at the masked record base plus `1`.
2. Write the high 32 bits of that combined result at the eightfold base plus `0x34`.
3. Call a dependency with the low 32 bits shifted left by two through a signed-typed intermediate.

The two word writes at the eightfold base plus `0x34` are distinct events separated by calls.

### 4. Sixteenfold and absolute-address writes

After four no-explicit-argument calls, sixteen times the index source becomes the **sixteenfold base**.

Write:

1. The low byte of the sixteenfold base at `0x11` plus the masked record base logically shifted right by 20.
2. The low byte of the retained fourfold value at the sixteenfold base plus `0x0d`.

Call a dependency with two identical arguments: the latest result shifted left by two.

A no-explicit-argument call supplies a word. Call another dependency with that word and its logical right shift by three. After four no-explicit-argument calls:

1. Write the low byte of the sixteenfold base at absolute address `0x11`.
2. Repeat the retained-fourfold byte write at the sixteenfold base plus `0x0d`.
3. Call a dependency with the latest result shifted left by two.

After four further no-explicit-argument calls, write the retained fourfold value as a word at the masked record base plus `0x14`. Then write the low byte of twice the masked record base at that base plus `0x0d`.

### 5. Thirty-twofold storage and selector reads

Retain the unsigned latest result as the **later scale source**. Derive:

- The **thirty-twofold base**: the source multiplied by 32.
- A retained word: the source shifted left by seven.
- The **extracted field**: the source masked with `0x07ffffff`, then logically shifted right by 17.

The extracted field ranges from `0` through `1023`.

After three no-explicit-argument calls, retain the latest result as the **shift source**. Write the retained seven-bit-left-shifted word at the thirty-twofold base plus `0x14`. Write the low byte of the shift source shifted left by two at that base plus `0x0d`. Call a dependency with the shift source shifted left by two.

After four no-explicit-argument calls:

1. Write the low byte of twice the retained quarter at the extracted field plus `7`.
2. Read a selector byte at the thirty-twofold base plus `1`.
3. Write the low byte of the extracted field at `0x11` plus that selector.

Call a dependency with three identical arguments: the shift source shifted left by 21.

A no-explicit-argument call supplies the unsigned **later word**, which is passed to another dependency. After four no-explicit-argument calls, read the byte at the thirty-twofold base plus `1` again. Retain this second byte as the **saved selector**.

Write zero at `0x11` plus the saved selector. Call a dependency with the latest word result shifted left by two. The saved selector need not equal the earlier selector.

### 6. Saved-selector processing and the second shared-target call

After four no-explicit-argument calls:

1. Write the low byte of the later word logically shifted right by three at the thirty-twofold base plus `0x0d`. Retain that byte as the **saved payload**.
2. Write the saved selector at absolute address `0x0d`.
3. Call a dependency with twice the saved selector, `0x98`, and `0x260`.

A no-explicit-argument call follows. Another dependency receives a secondary register result logically shifted right by three. The origin and producing interface of that secondary result are unresolved.

Four no-explicit-argument calls follow. Call **F-U01** with the latest result shifted left by two. At this site, the call has one explicit 32-bit argument and its returned value is retained as a signed-typed 32-bit word.

This is the same represented destination as the no-explicit-argument, combined-result call in phase 3. The fixed destination expressions at the two sites evaluate to the same address. Their differing argument and result representations do not establish separate targets or a consistent callable signature.

After the second F-U01 call returns:

1. Write the saved payload at twice the saved selector plus `0x0d`.
2. Write `0xc0` at absolute address `0x0d`.
3. Call another dependency with the retained F-U01 result shifted left by two.

### 7. Linked word updates

After four no-explicit-argument calls, write the latest result shifted left by two as a word at twice the saved selector plus `0x24`.

Read a word at twice the saved selector plus `0x74`, retaining it as the unsigned **linked word**. Retain the latest call result shifted left by five. Call a dependency with:

1. The linked word.
2. The linked word shifted left by one.
3. The retained five-bit-left-shifted result.
4. The linked word masked with `0x7fffffff`, then logically shifted right by two.

After four no-explicit-argument calls, write the retained five-bit-left-shifted result at the linked word plus `0x24`.

Read a word at the linked word plus `0x74` and pass that newly read word directly to a dependency. The linked word is therefore both data and an address base.

### 8. Combined-result transfers

Four no-explicit-argument calls follow; the fourth has a represented 64-bit result.

Retain its low half shifted left by two through a signed-typed intermediate as the **retained shifted half**. At four times its unsigned high half plus `0x14`, write the later word logically shifted right by three.

Call a dependency with the retained shifted half three times, followed by the high half logically shifted right by three.

A no-explicit-argument call supplies another represented 64-bit result. Call a dependency with:

1. Its low half.
2. Its high half.
3. Its low half again.
4. Its high half logically shifted right by three.

After four no-explicit-argument calls, write the retained shifted half at four times the latest result plus `0x14`. Call a dependency with four times that result.

### 9. Final byte-selected writes

After four no-explicit-argument calls, retain the unsigned latest result as the **final source**. Its left shift by two becomes the **final scaled value**, subsequently held in a signed-typed intermediate.

Read a byte at the linked word plus `1`. Write the low byte of the final scaled value at `0x11` plus that byte.

After three no-explicit-argument calls, reread the byte at the linked word plus `1`. At `0x11` plus this new selector, write the low byte of the final source masked with `0x3fffffff`, then logically shifted right by 20. Call a dependency with the latest result shifted left by two.

After four no-explicit-argument calls:

1. Read a byte at the final scaled value multiplied by `0x400`, plus `1`.
2. Write zero at `0x11` plus that byte.
3. Call a dependency with the final scaled value shifted left by three twice, the final scaled value, and the latest result shifted left by two.

After four further no-explicit-argument calls, repeat the selector read at the final scaled value multiplied by `0x400`, plus `1`. Write zero at `0x11` plus this newly read byte.

Call a dependency with the latest result shifted left by two. Four more no-explicit-argument calls precede the invalid-instruction boundary. No normal completion is established.

## Side effects

### Memory and call ordering

The visible prefix contains direct byte and word writes, including writes to absolute addresses `0x0d` and `0x11`.

No disjointness between address expressions is established. Computed locations may alias one another, selector locations, or low absolute addresses. Earlier writes may affect later reads.

The following distinctions are observable and must be retained:

- Both word writes at the eightfold base plus `0x34`.
- Both byte writes at the sixteenfold base plus `0x0d`.
- The saved-selector write and later `0xc0` write at absolute address `0x0d`.
- Every separately evaluated selector read.
- Calls between accesses, including calls whose results are unused.

The representation establishes an event order, not atomicity, synchronization, volatility, or a concurrency memory model.

### Ownership and lifetime

No allocation, release, ownership transfer, or lifetime extension is established. Call results used as addresses are not proven to identify valid or live storage.

No unresolved dependency, including F-U01, is assumed to preserve globals or memory, own returned storage, or release supplied storage.

### Errors and termination

There is no visible error result, argument rejection, recovery path, rollback, or cleanup path.

An inaccessible address, invalid call destination, nonreturning dependency, or processor exception may prevent later events. Earlier writes have no visible rollback mechanism.

The invalid-instruction boundary does not establish a specific hardware exception, exit status, intentional halt, or behavior beyond the represented prefix.

## Dependencies

All calls are indirect and unresolved.

**F-U01** identifies one statically equal represented destination used at two highlighted sites:

| Site | Explicit arguments | Represented result use |
|---|---|---|
| Phase 3, final call in the four-call group | None | 64-bit combined result; high half stored, shifted low half passed onward |
| Phase 6, after the four-call group | One 32-bit shifted result | Signed-typed 32-bit result retained and later shifted for another call |

This equality follows from evaluating the fixed destination expressions, not from assuming similar behavior. It establishes target-address equality only. Actual identity, executable validity, signature, argument consumption, and return convention remain unresolved.

**F-USET** is an opaque reference to the remaining unresolved dependency set, not a claim that those calls all reach one function. Other destination relationships are not comprehensively established here.

“No explicit arguments” does not establish a zero-argument interface. A represented combined result does not establish a conventional scalar 64-bit return ABI. The inconsistent representations at F-U01 must remain visible rather than being reconciled by an invented interface.

## Acceptance examples

These are conditional static examples, not executed tests.

### Entry truncation

If the entry read yields `0x123456ab`, the first byte write stores `0xab` at the entry base plus `1`. The initial call still receives the full word `0x123456ab`.

### Entry-fragment boundary

If the entry bitfield is `0xffc00000`, the entry fragment is `0x3ff`.

Each duplicated masked-and-shifted argument is `0x3fe00000`. The unsigned 32-bit fragment shift used for the later word write yields `0xff800000`. The nine-bit mask applies to the duplicated arguments, not to that later write.

### Shared destination, differing representations

If execution reaches both F-U01 sites, their represented destination addresses must compare equal.

The phase-3 site has no explicit arguments and consumes a combined result. The phase-6 site explicitly supplies one shifted 32-bit value and retains a 32-bit result. Acceptance does not require these representations to describe a coherent real-world ABI.

### Combined-result halves

If the phase-3 combined result has low half `3` and high half `7`, the second word write at the eightfold base plus `0x34` stores `7`. The following call receives `12`.

This does not establish the target’s actual return convention.

### Independent selectors

If one selector read yields `0xff`, its byte-selected destination is `0x110`, not an address obtained by sign-extending negative one.

If a later read yields `0x02`, that later destination is `0x13`. Reusing the earlier selector would violate the represented behavior.

### Absolute-address overwrite and interruption

If execution reaches both relevant writes, absolute address `0x0d` first receives the saved selector and later receives `0xc0`, with intervening calls and the computed payload write preserved.

If an intervening dependency does not return, subsequent writes are not required. No visible mechanism reverses earlier writes.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and data-structure identities.
- Complete calling convention and origin of implicit inputs.
- Dependency identities, signatures, side effects, and executable validity.
- Interpretation of F-U01’s differing argument and result representations.
- Whether combined results represent scalar returns, paired registers, or representation artifacts.
- Which interfaces produce the secondary register results.
- Signed overflow, signed shifts, unsigned-to-signed conversions, and machine address wrapping.
- Endianness, alignment, permissions, hardware semantics, and concurrency behavior.
- Actual executable reachability of the represented linear path.
- Behavior beyond the invalid-instruction boundary.
- Normal return, error reporting, ownership, and lifetime conventions.

F-U01’s represented destination equality is established and is not included among these unknowns. Equality alone resolves none of its interface or execution uncertainties.

## Evidence

**Established static evidence:** two explicit 32-bit inputs; implicit machine-state inputs; ordered byte and word accesses; unresolved indirect calls; secondary register results; selected combined results; and a terminal invalid-instruction boundary.

Static evaluation of the two highlighted fixed destination expressions establishes the shared F-U01 destination. One site represents a no-explicit-argument call with a 64-bit result; the other represents a one-explicit-argument call with a retained 32-bit result.

**Inferred intent:** no application-level intent is adopted. “Base,” “record,” “selector,” and “payload” describe arithmetic or dataflow roles only.

**Limitations:** the supplied representation warns of invalid instruction data. This contract preserves its visible prefix without claiming validated instruction semantics, executable reachability, or successful completion.

No runtime tests, dependency observations, memory traces, or instruction-level validation have been performed.
