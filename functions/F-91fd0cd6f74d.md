<!-- Trusted provenance: F-91fd0cd6f74d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-91fd0cd6f74d

## Purpose

F-91fd0cd6f74d repeatedly obtains values from unresolved dependencies, propagates selected values into 32-bit destinations, performs byte-selected writes, captures a global word, and replaces its carried state before repeating.

The represented body has no normal return or loop-exit condition. This does not establish that execution continues indefinitely: a dependency may not return, a memory access may fault, or an external mechanism may interrupt execution.

No application-level purpose is established. Numerical offsets and scaling relationships do not establish objects, field meanings, ownership conventions, or a serialized format.

This contract describes static evidence. Every later event is conditional on earlier calls returning and earlier accesses succeeding.

## Interface

### Explicit inputs and carried state

| Input | Width | Established role |
|---|---:|---|
| First input | 32 bits | Not explicitly consumed by the represented body |
| Working word | 32 bits | Initial dependency arguments and initial propagated value |
| Companion word | 32 bits | Third argument of the iteration-opening call |
| Working byte | 8 bits | Value written by two ordered pairs of byte stores |

The working word, companion word, and working byte are carried across iterations. The first iteration uses their incoming values. At each completed iteration boundary, both carried words become the same dependency-derived value, and the working byte becomes zero.

The absence of an explicit use of the first input does not establish that its incoming machine representation is irrelevant to unresolved calling conventions.

There is no represented return value, completion status, or explicit error result.

### Implicit inputs and execution preconditions

Execution also depends on:

- Incoming continuation-like state used to calculate the first call target.
- Dependency results, including four represented 64-bit results.
- Memory at calculated addresses and one global word-sized location.
- An additional register-derived dependency argument whose definition is unresolved.
- Any changes to memory, control flow, registers, or resource lifetimes caused by dependencies.

To reach a particular event, all preceding dependency calls must return and all preceding memory accesses must complete. Calculated call targets and memory locations must support the relevant operations. The body does not check these conditions.

### Width and arithmetic model

A **word** is 32 bits, a **byte** is 8 bits, and a **wide result** is 64 bits. Wide results are consumed through their low and high 32-bit halves.

Numerical relationships below use a conditional 32-bit bitvector model:

- Word reduction retains the low 32 bits.
- Byte reduction retains the low 8 bits.
- Scaling and left shifts are followed by word reduction.
- Address arithmetic is reduced modulo \(2^{32}\).
- A loaded selector byte is an unsigned integer from 0 through 255.
- \(Q(x)\) means masking the word \(x\) with `0x3fffffff`, then shifting it right by 20 bits without sign extension.

Under this model, \(Q(x)\) lies between 0 and 1023.

The representation mixes signed-looking arithmetic with unsigned operations. The model supports the conditional numerical conclusions below; it does not establish executable overflow semantics, pointer-formation rules, or the platform’s complete address model.

### Numerical alignment

Twelve of the thirteen word-store destinations are numerically divisible by four. Their address expressions are multiples of 4, 16, or 4096 plus 20. Reduction modulo \(2^{32}\) preserves divisibility by four because \(2^{32}\) is itself divisible by four.

The exception is the word destination formed from a retained byte selector plus 20. That destination need not be divisible by four. The calculated word-read destination, \(Q(x)+20\), also need not be divisible by four.

Numerical four-byte alignment does not establish accessibility, a valid object, or satisfaction of any stronger platform alignment requirement. Alignment of the global word location is not established here.

## Behavior

### Repetition and opening boundary

Dependency IDs F-001 through F-146 identify call sites in execution order within one represented iteration; they do not identify proven distinct implementations.

Each iteration opens with F-001 receiving the current working word twice, followed by the companion word. F-002, F-003, F-004, and F-005 then execute with no explicit arguments before the first word store. The F-005 result selects that destination.

Calls remain observable even when their results are unused or reduce to zero. No dependency is established as pure, terminating, or memory-preserving.

### Ordered memory effects

The following table gives the order of memory effects explicitly represented in the body. Dependency calls between these groups remain part of the behavior.

| Order | Memory effects |
|---:|---|
| 1 | Store the incoming working word at four times the F-005 result, plus 20 |
| 2 | Store four times the F-010 result at sixteen times that result, plus 20 |
| 3 | Store the same retained value at four times the F-015 result, plus 20 |
| 4 | Perform the first ordered pair of working-byte initialization stores |
| 5 | Perform the second ordered pair of working-byte initialization stores |
| 6 | Store the transformed F-015 value at the F-032-derived destination; immediately read the calculated word |
| 7 | Store zero at four times the F-039 result, plus 20 |
| 8 | Read an initialization-base selector and write the byte-reduced fourfold F-044 result |
| 9 | Read another selector there, retain it, and write the byte-reduced \(Q\) of the unscaled F-044 result |
| 10 | Use that retained selector for a word store; immediately capture the global word |
| 11 | Freshly read an initialization-base selector and write the byte derived from the global snapshot |
| 12 | Freshly read an initialization-base selector and write zero |
| 13 | Perform the first large-scale word-store pair |
| 14 | Read the F-061-derived selector location and write the byte derived from the first pair’s propagated value |
| 15 | Read the first F-088-derived selector location and write zero |
| 16 | Read the second F-088-derived selector location and write zero; later read it again and write zero |
| 17 | Perform the second large-scale word-store pair |
| 18 | Freshly read the second F-088-derived selector location and write the byte derived from the second pair’s propagated value |
| 19 | Freshly read that location again and write zero |
| 20 | Perform the third large-scale word-store pair |
| 21 | Update the working word, clear the working byte, perform the final word store, then update the companion word |

A selector read precedes its associated write. If that write overlaps the selector’s storage, its destination still uses the byte already read.

### Word propagation

The incoming working word is stored at four times the F-005 result, plus 20.

Four times the F-010 result is retained and stored twice:

1. At sixteen times the F-010 result, plus 20.
2. At four times the F-015 result, plus 20.

The first store precedes the second. This retained fourfold value also supplies all three explicit arguments to F-011.

The F-015 result shifted left by 31 bits is stored at sixteen times the F-032 result, plus 20. No dependency call separates this store from the calculated-address word read described below.

Zero is subsequently stored at four times the F-039 result, plus 20, before the first byte-selected write based on F-044.

Four times the F-034 result is stored at the retained selector plus 20. This is the only word-store destination selected directly by a byte; no word-sized pointer is fetched from that selector location.

Three further retained values are each stored at two destinations:

| Retained value | First destination | Subsequent destination |
|---|---|---|
| Four times the F-078 result | 4096 times the fourfold low half from F-061, plus 20 | Four times the F-083 result, plus 20 |
| F-100 result shifted left by 14 bits | 4096 times the fourfold F-100 result, plus 20 | Four times the F-119 result, plus 20 |
| Four times the F-136 result | 4096 times the fourfold F-124 result, plus 20 | Four times the F-141 result, plus 20 |

Each row uses the same saved value for both writes despite intervening calls. Those calls may alter either destination; the body does not establish that the first stored copy survives unchanged.

Finally, four times the F-146 result becomes the next working word and is stored at sixteen times that result, plus 20.

All destinations in this subsection except the retained-selector destination are numerically four-byte aligned under the stated model, including when arithmetic wraps. This says nothing about their semantic layout or accessibility.

### Repeated byte initialization

The **initialization base** is the F-010 result multiplied by \(2^{30}\), with word reduction.

The working byte is written at:

1. Initialization base plus 17.
2. Initialization base plus 1.

That ordered pair occurs twice. F-021 through F-027 execute between the pairs. Their possible effects prevent treating the second pair as redundant.

Under the conditional model, the base depends only on the low two bits of the F-010 result and has four possible values. No accessibility guarantee follows.

The first store in a pair can affect later observations through aliasing. Later selector reads from base plus 1 are actual reads, not automatic reuse of the working byte.

### Byte-selected writes

Every selector-directed byte destination is the numerical value of a loaded byte plus 17. Such destinations range from 17 through 272 inclusive. There is no intervening word-pointer dereference.

Ten selector reads occur in a completed iteration:

| Selector location | Reads | Associated written values, in order |
|---|---:|---|
| Initialization base plus 1 | 4 | Byte-reduced fourfold F-044 result; byte-reduced \(Q(\text{F-044 result})\); byte-reduced captured global word shifted right by 22; zero |
| 16384 times the fourfold low half from F-061, plus 1 | 1 | Byte-reduced \(Q\) of the fourfold F-078 result |
| \(2^{28}\) times the fourfold low half from F-088, plus 1 | 1 | Zero |
| Four times the fourfold low half from F-088, plus 1 | 4 | Zero; zero; byte-reduced \(Q\) of the F-100 result shifted left by 14; zero |

In the first row, \(Q\) applies to the **unscaled** F-044 result, not its fourfold value.

Repeated reads of a common address are separate events. Dependencies, earlier stores, or external activity may change the byte between reads. The repeated zero stores do not justify reusing an earlier selector.

### Retained selector

The second selector read at initialization base plus 1 has an extended lifetime. It supplies both:

- The destination for the byte-reduced \(Q\) of the F-044 result, at selector plus 17.
- The later destination for the fourfold F-034 word, at selector plus 20.

F-052 through F-056 execute between these stores. Changes they make to the selector’s original storage do not redirect the later word store: its address uses the already-read byte.

The word destination ranges from 20 through 275. It is four-byte aligned exactly when the retained selector is divisible by four.

### Calculated word read and global snapshot

The calculated-address read obtains a word from \(Q(\text{F-020 result})+20\). There is no additional multiplication by four. Its numerical address ranges from 20 through 1043, and it need not be four-byte aligned.

The transformed word store selected by F-032 immediately precedes this read. No dependency call intervenes. If their byte ranges overlap, the read can reflect the preceding store, subject to the unresolved memory representation.

The loaded word is shifted right by 22 bits without sign extension and then left by 28 bits, with word reduction, to form an argument to F-035.

The global word is read immediately after the retained-selector word store. No dependency call intervenes. Overlap between that store and global storage is not excluded.

This global value is retained while F-057 through F-061 execute. Its unsigned right shift by 22, followed by byte reduction, supplies a later byte store. That store uses a freshly read selector.

Thus the written value is a snapshot, but the destination is selected later. Intervening global changes do not alter the captured value; global preservation by dependencies is not required for this relationship.

### Iteration transition

After F-146 returns:

- Its fourfold word-reduced result replaces the working word.
- The working byte becomes zero.
- The final word store occurs.
- Only after that store does the companion word receive the new working word.

If execution reaches the next iteration, F-001 receives three identical words and both initialization pairs write zero.

The first F-001 target depends on incoming continuation-like state. Later iterations use internally established continuation-like state rather than reusing the original incoming basis unchanged.

## Side effects

A complete represented iteration contains:

- 146 unresolved call events.
- 13 word stores.
- 14 byte stores.
- Ten byte-selector reads.
- One calculated-address word read.
- One global word read.

These counts exclude accesses inside dependencies.

Twelve word stores have numerically four-byte-aligned destinations under the conditional bitvector model. Address wraparound does not remove that property. The retained-selector word store and calculated word read may lack four-byte alignment. Platform-specific alignment requirements and actual accessibility remain unknown for every access.

There are no represented checks for bounds, valid targets, accessible memory, alignment suitability, dependency-result validity, ownership, or lifetime. Accesses may overlap one another or global storage. Dependencies may invalidate storage before a later access.

No allocation, release, ownership transfer, synchronization, or cleanup operation is explicitly established by the body. Such operations inside dependencies remain unknown.

A fault or nonreturning dependency prevents subsequent events. Earlier effects have no represented rollback. Endianness, atomicity, volatile behavior, barriers, and concurrent-observation guarantees are not established.

## Dependencies

F-001 through F-146 are opaque call-site references, not proven distinct callable identities.

Targets are calculated from continuation-like values and byte adjustments. The common adjustment is 4094. Represented exceptions are:

| Call sites | Adjustment |
|---|---:|
| F-021, F-033 | 4086 |
| F-045 | 3976 |
| F-062, F-089, F-125 | 4036 |
| F-101 | 4056 |
| F-113 | 3986 |

These calculations establish neither valid executable targets nor a calling convention.

Explicit argument lists contain zero through four words. A site without explicit arguments may still depend on hidden machine state.

Width-sensitive relationships include:

- F-061 and F-088 return wide results. Their low halves supply later fourfold values; F-062 and F-089 respectively consume their high halves.
- F-092 and F-128 return wide results. F-093 and F-129 respectively receive the low half followed by the high half.
- F-047 receives a forwarded word result and the F-044 result shifted left by 30.
- F-064 receives a forwarded word result and the fourfold low half from F-061 shifted left by 28.
- F-103 receives a forwarded word result and the unscaled F-034 result shifted left by 28.
- F-127 receives a forwarded word result and the fourfold F-124 result shifted left by 28.
- F-137 receives four words: two identical values equal to 1024 times the fourfold F-124 result, the F-124 result shifted left by 14, and the third large-scale pair’s propagated value.

F-115 has an additional unresolved register-derived argument. Its first argument is the F-100 result shifted left by 14 and subsequently left by 28. Under the conditional 32-bit model, that argument is zero. F-113 and F-114 still precede F-115, and none of these calls can be discarded solely because this argument reduces to zero.

No dependency is established as pure, terminating, register-preserving, global-preserving, or responsible for retaining or releasing memory.

## Acceptance examples

These are conditional static criteria, not runtime observations. Each assumes that the preceding calls and accesses complete.

1. **Opening boundary.** With working word 7 and companion word 9, F-001 receives \(7,7,9\). Four no-explicit-argument calls precede the first store. If F-004 does not return, F-005 and the store are not reached.

2. **Initialization order.** If F-010 returns 1, the initialization base is `0x40000000`. A working byte of `0xAB` is written at base plus 17 before base plus 1. The pair repeats only after the intervening dependency activity.

3. **Byte narrowing.** If F-044 returns 64, its fourfold value is 256 and the first associated byte store writes zero. If F-044 returns `0xffffffff`, \(Q\) of its unscaled result is 1023, so the later associated byte store writes 255.

4. **Retained destination.** A retained selector of 5 makes the word destination 25, which is not four-byte aligned. Changing the selector’s original storage to 9 does not redirect that store to 29.

5. **Fresh destination.** If a repeatedly read selector location changes from 5 to 9 before its next read, the next associated byte store targets 26 rather than 22. An earlier zero store does not eliminate this read.

6. **Global snapshot.** A captured global word of `0x00400000` yields byte value 1. Later global changes do not alter that value, while changes observed by the fresh selector read can alter its destination.

7. **Calculated read.** If F-020 returns `0xffffffff`, the calculated word-read address is 1043, not 4112. Address 1043 is not divisible by four; no alignment check is represented.

8. **Wrapping preserves store alignment.** If F-005 returns `0xffffffff`, fourfold scaling plus 20 reduces to address 16. The first word store remains numerically four-byte aligned despite wraparound. This does not establish that address 16 is accessible or satisfies other platform requirements.

9. **Zero-valued argument.** Under the bitvector model, F-115 receives zero as its first explicit argument and still receives its unresolved second argument. Its effects remain unknown.

10. **Next iteration.** If F-146 returns 3, the final store writes word 12 at address 68. If the store succeeds and execution continues, both carried words are 12, the working byte is zero, and the next F-001 receives \(12,12,12\).

## Uncertainties

The following remain **documented_only**:

- Application purpose and semantic interpretation of accessed memory.
- Dependency identities, hidden inputs, side effects, ownership behavior, and errors.
- Validity and reachability of calculated call targets.
- Exact ABI, wide-return conventions, and origin of the extra register-derived argument.
- Executable semantics of signed-looking overflow, shifts, and pointer formation.
- Actual address-space properties, mappings, platform alignment requirements, endianness, and fault handling.
- Alignment and layout of the global word location.
- Aliasing, memory lifetimes, concurrent modification, and external termination.
- Behavior of material absent from the supplied representation.

These uncertainties do not negate the conditional arithmetic result: twelve word-store addresses are divisible by four under the stated model. They limit what that numerical property establishes about executable memory accesses.

Repeated propagation patterns may suggest encoded state processing. That is inferred intent, not an established requirement.

## Evidence

Established static evidence consists of the supplied representation’s explicit inputs, unconditional repetition, calculated calls, word and byte accesses, split wide results, retained selector, global snapshot, and carried-state transition.

The alignment conclusion follows from the twelve non-selector word-store address expressions and reduction modulo \(2^{32}\). The retained-selector store and calculated word read have no corresponding universal four-byte divisibility property.

The supplied representation reports removed unreachable material and overlapping global symbols. Those reports limit interpretation; they do not establish additional executed behavior or independent global storage.

No runtime execution or runtime tests have been performed. No dependency implementation has been inspected. Numerical conclusions involving wraparound remain conditional on the stated 32-bit bitvector model.
