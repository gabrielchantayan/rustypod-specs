<!-- Trusted provenance: F-d46e11b72ebb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d46e11b72ebb

## Purpose

F-d46e11b72ebb performs an ordered sequence of indirect calls, memory reads, memory writes, and global-state accesses. It retains selected values across calls, scales dependency results into addresses, and uses two unsigned-byte lookups to select later byte-write destinations.

No application-level purpose is established. The supplied representation does not justify classifying the function as an allocator, serializer, dispatcher, or object initializer.

The terminal sequence assigns a dependency result to a global slot, clears a byte through an unresolved register-output pointer, clears a halfword at an address derived from **Q**, and attempts a call to address zero. **Q** is the retained low 32 bits of F-50’s result. No dependency contract establishes that **Q** depends on any displayed input parameter.

Normal return is not guaranteed.

## Interface

### Parameters and execution context

| Input | Represented width | Established use |
|---|---:|---|
| First parameter | 32 bits | No explicit use in the displayed body. |
| Second parameter | 32 bits | No explicit use in the displayed body. |
| Third parameter, **P** | Unsigned 32 bits | Forms an argument to F-13 and the destination of a word store. |

The first two parameters cannot be declared irrelevant to machine execution: dependency calling conventions are unresolved, including calls displaying no explicit arguments.

Other execution inputs include:

- An incoming link-register value used to construct the first call target.
- Fixed address anchors and offsets used to construct subsequent indirect targets.
- A global slot supplying a retained base address.
- Memory at the computed read and write addresses.
- An unresolved additional register-output pointer used near the end.

### Output and prerequisites

The displayed interface returns no explicit value. Observable outputs are calls, memory accesses, and a global assignment.

Every described operation is conditional on preceding operations completing and dependencies returning control. Successful traversal requires callable targets and memory supporting the accesses actually attempted. The function does not visibly establish those conditions.

There are no visible bounds, null, alignment, or result-validity checks.

### Width and arithmetic conventions

This contract describes the recovered 32-bit address/value model:

- A word is 32 bits, a halfword 16 bits, and a byte 8 bits.
- Unsigned right shifts are logical.
- Byte stores retain the low eight bits of their supplied value.
- Two dependency results are represented as 64-bit quantities and split into low and high 32-bit portions.
- Unsigned 32-bit arithmetic discards bits above bit 31.

Several dependency results are represented as signed 32-bit integers before multiplication, left shifting, or address addition. Signed overflow, shifting negative values, and overflowing signed left shifts are not assigned portable semantics here. Modular interpretations of those operations require target-level confirmation. The low half of the first paired result is also converted to a signed representation before scaling.

Address formulas below identify operations, not validated objects or arrays. Endianness and alignment rules remain unresolved.

## Behavior

### Call and value notation

F-01 through F-68 are opaque, document-local dependency IDs assigned in call-occurrence order. They do not assert distinct callable identities.

“No explicit arguments” describes the supplied representation, not a proven zero-argument ABI. Unless an argument is stated, each occurrence below displays no explicit arguments.

Letters such as **W**, **G**, and **Q** are editorial names for retained values, not source identifiers.

### Initial retained word and parameter transformation

The opening sequence is F-01; F-02 followed by F-03 receiving F-02’s 32-bit result; then F-04, F-05, F-06, and F-07. F-01’s target is derived from the incoming link register.

Let **A** be F-07’s signed-represented result and **X = 4 × A**. The function reads a 32-bit word **W** from **X + 116**, then writes its low byte to **X + 1**. F-08 receives **X**.

F-09, F-10, and F-11 follow. F-12 supplies a signed-represented result **B**, after which the function stores **W** as a word at **16 × B + 20**.

F-13 then receives **(P logical-right-shifted by 2) left-shifted by 16**, evaluated as an unsigned 32-bit value. This retains original bits 2–17 of **P**, placing them in argument bits 16–31. All other bits are discarded.

**W** is retained across intervening calls. Its later uses do not reread the original word address.

### Forwarding pairs and second retained-word store

F-14’s 32-bit result is passed to F-15. F-16’s 32-bit result is passed to F-17. F-18, F-19, and F-20 then execute.

F-21 supplies a signed-represented result **C**. The function stores **W** at **4 × C + 20**, then invokes F-22 with **4 × C**.

No equality among **A**, **B**, and **C** is established.

### Store through the third parameter

After F-23, F-24, and F-25, F-26 supplies a signed-represented result **D**. Let **Y** be **D left-shifted by 2**.

The function stores **Y** as a word at **(P with its low two bits cleared) + 20**, then invokes F-27 with two explicit arguments, **Y, Y**.

The address calculation uses unsigned 32-bit arithmetic. Clearing **P**’s low bits changes the destination, not the stored value, and does not establish allocation validity.

F-28’s 32-bit result is passed to F-29, and F-30’s 32-bit result is passed to F-31. F-32, F-33, and F-34 follow.

F-35 supplies a signed-represented result **E**. The function stores a zero word at **4 × E + 20**, then invokes F-36 with **4 × E**.

### First paired result and captured global base

F-37, F-38, and F-39 precede F-40, whose result is represented as 64 bits.

Let **L** and **H** be its low and high 32-bit portions. Both are used through signed 32-bit representations in this part of the sequence. Let **Z** be the signed-represented low portion left-shifted by two.

The next operations are ordered as follows:

1. Store **Z** as a word at **4 × H + 20**.
2. Read the global base-address slot and retain its value as **G**.
3. Invoke F-41 with **Z, H, Z, Z**, in that order.
4. Invoke F-42, F-43, and F-44.
5. Invoke F-45 and retain its signed-represented result as **J**.
6. Store **Z** as a word at **4 × J + 20**.
7. Read the 32-bit word **V** from **G + 20**.
8. Invoke F-46 with **4 × J**.

The capture of **G** and the read of **V** are separate observable events. A dependency changing the global slot after capture does not redirect the later read. A dependency changing memory at the retained address before that read can change **V**.

If **4 × J + 20** aliases **G + 20**, the direct store precedes the read and supplies its value under ordinary sequential memory semantics. No concurrency or volatile-memory guarantees are established.

### Second paired result and first byte lookup

F-47, F-48, and F-49 precede F-50, which supplies another represented 64-bit result.

Let **Q** be its unsigned low 32 bits and **R** its high 32-bit portion. The function then:

- Reads an unsigned byte **T₁** at **(W logical-right-shifted by 23) + 1**.
- Writes the low byte of retained **V** to **T₁ + 17**.
- Invokes F-51 with **R, R, R, Q left-shifted by 2**, in that order.
- Invokes F-52 and passes its 32-bit result to F-53.

The left shift of **Q** is unsigned 32-bit arithmetic.

The lookup address ranges from 1 through 512 inclusive. The loaded byte is a numeric destination selector, not a word-sized pointer. Consequently, its selected byte-write address ranges from 17 through 272 inclusive.

### Second byte lookup and remaining dependencies

F-54, F-55, and F-56 precede F-57, whose signed-represented result is retained as **K**.

The function reads an unsigned byte **T₂** from **((Q masked with 0x3fffffff) logical-right-shifted by 21) + 1**, then writes the low byte of **V logical-right-shifted by 22** to **T₂ + 17**.

This second lookup uses original bits 21–29 of **Q**. Its lookup address again ranges from 1 through 512 inclusive, and its selected destination ranges from 17 through 272 inclusive. The stored value is bits 22–29 of **V**.

The remaining call sequence before terminal stores is:

| Occurrence | Explicit behavior |
|---|---|
| F-58 | Receives **K left-shifted by 2**. |
| F-59, F-60, F-61 | Execute in that order. |
| F-62 | Supplies signed-represented result **M**. |
| F-63 | Receives **M left-shifted by 2** twice. |
| F-64 | Executes with no explicit arguments. |
| F-65 → F-66 | F-66 receives F-65’s 32-bit result. |
| F-67 | Supplies the result assigned to the terminal global slot. |

Both byte-write values use the previously retained **V**. Neither operation reloads **G + 20**.

### Terminal sequence

After F-67 returns, the function performs these operations in order:

1. Assign F-67’s result to a global slot associated by the representation with supervisor-call state.
2. Store one zero byte through the unresolved additional register-output pointer.
3. Store a zero halfword at **(Q masked with 0x3fffffff) logical-right-shifted by 7**.
4. Invoke F-68 at address zero with three explicit zero arguments.
5. Return only if control reaches the displayed return.

The halfword destination ranges from zero through 8,388,607 inclusive and may be odd. No alignment correction occurs.

That address is derived from F-50’s retained result **Q**, not directly from **P** or either other displayed parameter. Any relationship between **Q** and those parameters is unresolved.

The additional register-output pointer cannot be securely attributed to a particular dependency from the supplied representation.

## Side effects

### Direct memory-access order

Dependency-internal effects are additional to the following direct accesses:

| Order | Direct access |
|---:|---|
| 1 | Read word **W** from **4 × A + 116**. |
| 2 | Write low byte of **W** to **4 × A + 1**. |
| 3 | Write word **W** to **16 × B + 20**. |
| 4 | Write word **W** to **4 × C + 20**. |
| 5 | Write word **Y** to **(P with low two bits cleared) + 20**. |
| 6 | Write a zero word to **4 × E + 20**. |
| 7 | Write word **Z** to **4 × H + 20**. |
| 8 | Read the global base slot into **G**. |
| 9 | Write word **Z** to **4 × J + 20**. |
| 10 | Read word **V** from **G + 20**. |
| 11 | Read lookup byte **T₁**. |
| 12 | Write low byte of **V** to **T₁ + 17**. |
| 13 | Read lookup byte **T₂**. |
| 14 | Write bits 22–29 of **V** to **T₂ + 17**. |
| 15 | Assign F-67’s result to the terminal global slot. |
| 16 | Clear one byte through the unresolved pointer. |
| 17 | Clear one halfword at the masked-and-shifted **Q** address. |

The calls interleave these accesses exactly as described in Behavior.

Aliasing is not excluded. Earlier writes can alter later reads, including either lookup byte. In particular, the first selected destination can overlap the second lookup location. If both lookup reads select the same destination, the second byte write is later; its effect may subsequently be altered by dependencies or terminal stores.

### Ownership and state preservation

No direct allocation, release, reference-count adjustment, or explicit ownership transfer is established.

Dependencies may nevertheless allocate, release, retain, consume, or reinterpret supplied values. Their preservation of globals, memory, registers, and ownership is unknown. A result retained by the displayed body must not be confused with a guarantee that its referenced memory remains valid.

The representation also warns that some global symbols overlap smaller symbols. The exact overlap and its effects on the two explicit global accesses are unresolved.

### Errors and partial progress

There is no visible error result, recovery branch, rollback, or validation of dependency results before address use.

An invalid target, inaccessible location, unsupported alignment, or nonreturning dependency can prevent further progress. Fault delivery is environment-dependent. Earlier effects are not visibly undone.

## Dependencies

The complete occurrence inventory is F-01 through F-68, ordered in Behavior. These IDs identify call sites only; distinct occurrences may resolve to the same target.

Established dependency relationships include:

- Seven single-result forwarding pairs: F-02 → F-03, F-14 → F-15, F-16 → F-17, F-28 → F-29, F-30 → F-31, F-52 → F-53, and F-65 → F-66.
- Address-forming results from F-07, F-12, F-21, F-26, F-35, F-40, F-45, and F-50.
- Additional shifted arguments derived from F-57 and F-62.
- Four explicit arguments to each of F-41 and F-51.
- A global-assignment result from F-67.
- Three explicit zero arguments to the address-zero occurrence F-68.

F-01 uses an incoming-link-register-derived target. Other nonterminal targets are constructed from fixed anchors and offsets. No resolved external identities or trustworthy dependency contracts are supplied.

The representation of F-40 and F-50 as 64-bit returns does not establish their actual ABI. Likewise, absence of explicit result consumption does not imply absence of dependency side effects.

## Acceptance examples

These are conditional static criteria, not executed tests. Each assumes execution reaches the relevant operations and that memory and arithmetic semantics permit them.

1. **Retained word.** If F-07 returns 64, **X = 256**. If the word read at address 372 is 0x12345678, the following byte write stores 0x78 at address 257. Later direct stores of **W** still use 0x12345678 even if a dependency changes address 372.

2. **Parameter transformation.** For **P = 0x0000000b**, F-13 receives 0x00020000. The later **Y** word store targets address 0x1c.

3. **Unsigned truncation.** For **P = 0x00040000**, F-13 receives zero: the right shift yields 0x00010000, and shifting that value left by 16 discards its set bit at 32-bit width.

4. **Captured base, later content.** If **G** is captured as 0x1000 and a later dependency changes the global slot to 0x2000, the read supplying **V** still targets 0x1014. Its value is determined at that later read, not at base capture.

5. **Byte extraction.** If retained **V = 0xabcdef12**, the first selected byte write supplies 0x12 and the second supplies 0xaf. If both lookup reads return 5, both writes target address 22; the second occurs later.

6. **Lookup aliasing.** If the first byte write targets the second lookup’s address, and no intervening effect changes that byte, the second lookup reads the first write’s value. It must not be treated as a preloaded table entry.

7. **Result-derived unaligned destination.** If F-50’s low portion is **Q = 128**, the terminal halfword destination is address 1. The function does not align it. No displayed parameter value alone establishes this outcome.

8. **Terminal failure.** If execution reaches F-68 and address zero faults, normal return is not required. The global assignment and both clearing stores precede that call, assuming those earlier operations completed.

## Uncertainties

The following remain **documented_only**:

- Dependency identities, signatures, hidden inputs, side effects, and preservation contracts.
- The actual ABI interpretation of paired results and calls without explicit arguments.
- Any relationship between **Q** and the displayed parameters.
- Signed arithmetic behavior outside representable ranges, including negative left shifts and signed address additions.
- Endianness, address mapping, alignment support, and exception handling.
- The origin, lifetime, and validity of the additional register-output pointer.
- The exact storage width and semantic role of the terminal global slot.
- The concrete extent and consequences of overlapping global symbols.
- Ownership, object boundaries, concurrency, volatility, and synchronization.
- Whether address zero is callable, traps, or is unmapped.
- Application-level meanings of offsets, bit fields, and repeated scaling.

Repeated scale-by-four and offset-20 accesses suggest a shared addressing convention. That is inferred intent, not proof of an object layout.

## Evidence

Established static evidence consists of the supplied function representation: displayed parameter widths, ordered indirect calls, explicit argument forwarding, retained results, masks and shifts, access widths, global accesses, unresolved register-output store, and terminal address-zero call.

The supplied provenance envelope includes a snapshot identifier. That identifier is provenance metadata, not evidence contained in the function representation; it is not reproduced here. Publication provenance is supplied by the publishing system.

Dependency IDs and descriptive value labels in this document are editorial. Application-level interpretations are not established by them.

No runtime tests have been performed. Acceptance examples express arithmetic and ordering consequences of the static representation under their stated conditions.
