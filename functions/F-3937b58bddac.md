<!-- Trusted provenance: F-3937b58bddac; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3937b58bddac

## Purpose

F-3937b58bddac coordinates seventy indirect call occurrences with interleaved memory reads and writes. It captures a word, copies that value to computed destinations, writes scaled references into other destinations, performs two lookup-selected byte writes, assigns a dependency result to a global, and clears a byte and a halfword before attempting a call to address zero.

No application-level purpose is established. Repeated scaling and field offsets suggest structured state manipulation, but do not establish allocation, serialization, object construction, device configuration, or ownership semantics.

This contract describes the supplied static representation. Execution of later operations is conditional on earlier calls returning and earlier accesses completing. A normal return is represented after the final call, but successful termination is not guaranteed.

## Interface

### Explicit inputs and output

| Item | Represented width | Established use |
|---|---:|---|
| First input | 32 bits | Forwarded to the first dependency. |
| Second input | 32 bits | No explicit use in the represented dataflow. |
| Third input | Unsigned 32 bits | Supplies a transformed call argument and an aligned-down destination base. |
| Return value | None | No explicit result is returned. |

The absence of explicit use of the second input does not establish that its incoming register or other ambient machine state is irrelevant to unknown dependencies.

### Implicit inputs and state

The procedure also depends on:

- An incoming link-register-derived value determining the first call target.
- Other indirect call targets represented by computed code addresses.
- A global value captured into a represented 32-bit address, designated **G-base**.
- A global destination designated **G-result**, whose declared type and assignment width are unavailable.
- A register-derived pointer designated **P-tail**, used for a late byte clear.
- Memory reachable through dependency results, the third input, the captured global address, and lookup-derived addresses.

These labels describe roles, not semantic types or recovered source identifiers.

### Dependency identifiers

**F-C01** through **F-C70** identify successive call occurrences. They do not establish seventy distinct functions. Target identity relationships and actual dependency signatures remain unresolved.

“No explicit arguments” means none are represented at that call site. It does not establish that the dependency ignores registers, globals, or other ambient state.

### Widths and arithmetic

The representation establishes:

- Byte accesses of 8 bits.
- One halfword store of 16 bits.
- Explicit word accesses of 32 bits.
- Two dependency results represented as 64-bit quantities, each split into 32-bit halves.
- Several other dependency results captured in 32-bit containers.
- A final dependency result assigned directly to G-result without an explicit width constraint.

In the descriptions below:

- **Q(x)** denotes four times the represented quantity, using the scaling operation at that occurrence.
- **R(x)** denotes sixteen times the represented quantity.
- **word**, **byte**, and **halfword** specify 32-, 8-, and 16-bit memory accesses.

Some scaling operations use represented signed multiplication or signed left shift; others use unsigned shifts. Q does not imply one universally established overflow rule. Examples using signed arithmetic stay within representable ranges unless expressly qualified.

The third-input transformations and shifts of unsigned 32-bit values have the represented 32-bit bit-selection behavior. In particular, an unsigned left shift by two retains the low 32 bits. Signed overflow, architectural address wrapping, and pointer validity are not resolved by that observation.

Byte truncation retains the low eight bits. Endianness is unspecified and matters when differently sized accesses overlap.

## Behavior

### Initial capture and first copy

The initial dependency sequence is:

| Calls | Explicit arguments and consumed results |
|---|---|
| F-C01 | Receives the first input; no result is consumed. |
| F-C02, F-C03 | F-C02 supplies a 32-bit value forwarded to F-C03. |
| F-C04, F-C05 | F-C04 supplies a 32-bit value forwarded to F-C05. |
| F-C06–F-C08 | No explicit arguments or consumed results. |
| F-C09 | No explicit arguments; supplies index **A**. |

F-C01’s target is formed by adding 0xFB0 to the incoming link-register-derived value. Its validity is not checked.

After F-C09, a 32-bit word **W** is read at **Q(A) + 0x74**. Its low byte is then written at **Q(A) + 1**. F-C10 receives Q(A) after that byte write.

W remains the captured word throughout its subsequent uses; the procedure does not reload it from its original address.

F-C11 through F-C13 have no explicit arguments. F-C14 then supplies index **B**, also without explicit arguments. W is stored as a 32-bit word at **R(B) + 0x14**.

This destination uses sixteen-byte scaling, unlike the four-byte scaling at later word destinations.

### Third-input transformation and reference write

F-C15 receives the 32-bit result of logically shifting the third input right by two and then left by sixteen. Its lower sixteen bits are zero; its upper sixteen bits contain original input bits 2 through 17. Higher original bits are discarded.

The next calls are:

| Calls | Explicit arguments and consumed results |
|---|---|
| F-C16, F-C17 | F-C16’s 32-bit result is forwarded to F-C17. |
| F-C18, F-C19 | F-C18’s 32-bit result is forwarded to F-C19. |
| F-C20–F-C22 | No explicit arguments or consumed results. |
| F-C23 | No explicit arguments; supplies index **C**. |

W is stored as a 32-bit word at **Q(C) + 0x14**. F-C24 then receives Q(C).

F-C25 through F-C27 have no explicit arguments. F-C28 supplies index **D**. Its represented signed left shift by two produces **L**.

L is stored as a 32-bit word at:

**(third input AND 0xFFFFFFFC) + 0x14**

The mask clears the input’s low two bits. The following addition is represented in unsigned 32-bit arithmetic, so it can wrap. The alignment operation is not a validity or bounds check.

F-C29 receives L twice, as two equal explicit arguments.

### Zero field and first paired result

The following sequence precedes a zero word store:

| Calls | Explicit arguments and consumed results |
|---|---|
| F-C30, F-C31 | F-C30’s 32-bit result is forwarded to F-C31. |
| F-C32, F-C33 | F-C32’s 32-bit result is forwarded to F-C33. |
| F-C34–F-C36 | No explicit arguments or consumed results. |
| F-C37 | No explicit arguments; supplies index **E**. |

A 32-bit zero is stored at **Q(E) + 0x14**. F-C38 then receives Q(E).

F-C39 through F-C41 have no explicit arguments. F-C42 supplies a represented 64-bit result:

- **J** is its lower 32-bit half.
- **K** is its upper 32-bit half.
- **M** is the result of the represented signed left shift of J by two.

M is stored as a 32-bit word at **Q(K) + 0x14**.

Only after that store does the procedure read G-base and capture its value into the represented 32-bit address **P**. F-C43 then receives four explicit arguments, in order:

**M, K, M, M**

### Captured address and delayed contents

F-C44 through F-C46 have no explicit arguments. F-C47 supplies index **N**.

A 32-bit word containing M is stored at **Q(N) + 0x14**. The procedure then reads the 32-bit word **V** at **P + 0x14**, before F-C48 receives Q(N).

P is the previously captured address, not a fresh read of G-base. Dependencies between capture and use may change G-base without changing this represented address choice. However, V is read only at this later point: intervening writes, including the immediately preceding word store if it aliases, can affect V.

F-C49 through F-C51 have no explicit arguments. F-C52 then supplies another represented 64-bit result:

- **T** is its lower 32 bits, subsequently used as unsigned.
- **H** is its upper 32 bits.

### First lookup-selected byte write

After F-C52, an unsigned byte **U₁** is read at:

**(W logically shifted right by 23) + 1**

The low byte of V is then written at **U₁ + 0x11**.

The lookup address ranges from 1 through 512. Because U₁ is an unsigned byte, the store destination ranges from 0x11 through 0x110. The loaded byte determines the destination; the lookup address itself is not the store destination.

F-C53 follows the byte write and receives:

**H, H, H, T shifted left by two**

The final argument is an unsigned 32-bit shift result, retaining the low 32 bits.

F-C54 supplies a 32-bit value forwarded to F-C55. F-C56 through F-C58 have no explicit arguments. F-C59 then supplies index **O**, without explicit arguments.

### Second lookup-selected byte write

After F-C59, an unsigned byte **U₂** is read at:

**((T AND 0x3FFFFFFF) logically shifted right by 21) + 1**

A byte is then written at **U₂ + 0x11**, containing the low eight bits of **V logically shifted right by 22**. These are V’s original bits 22 through 29; original bits 30 and 31 are excluded.

This lookup also addresses the range 1 through 512, and its destination also ranges from 0x11 through 0x110.

The lookup is performed at this later point, not alongside the first lookup. Intervening calls or the first byte write can alter its source byte. The two expressions identify overlapping address ranges, not proven independent tables.

F-C60 then receives the represented signed left shift of O by two.

### Terminal calls and writes

F-C61 through F-C63 have no explicit arguments. F-C64 supplies index **S**. F-C65 receives two equal arguments, each formed by the represented signed left shift of S by two.

The remaining observable order is:

| Stage | Effect |
|---|---|
| F-C66 | Called with no explicit arguments; no result is consumed. |
| F-C67, F-C68 | F-C67’s captured 32-bit result is forwarded to F-C68. |
| F-C69 and global assignment | F-C69 is called without explicit arguments; its result is assigned to G-result. Neither the result width nor assignment width is established. |
| Byte clear | An 8-bit zero is written through P-tail. |
| Halfword clear | A 16-bit zero is written at **(T AND 0x3FFFFFFF) logically shifted right by 7**. |
| F-C70 | A call to address zero is attempted with three explicit zero arguments. |
| Nominal return | A void return is represented after F-C70. |

The final halfword address ranges from zero through 0x7FFFFF. Odd addresses are possible; no alignment correction is represented.

P-tail is exposed through an additional register-derived value. Its originating dependency, validity, and relationship to the G-result assignment are not established.

## Side effects

### Direct access counts

If the complete represented sequence is reached, it contains **twelve explicit stores**, **five explicit reads**, and **seventy call occurrences**.

The stores are:

| Kind | Count | Roles |
|---|---:|---|
| Explicitly width-qualified 32-bit stores | 6 | Two copies of W; the third-input-derived reference; the zero field; two stores of M. |
| Byte stores | 4 | Initial truncation of W; two lookup-selected writes; P-tail clear. |
| Halfword stores | 1 | Final masked-and-shifted address clear. |
| Global assignment of unresolved width | 1 | Assignment of F-C69’s result to G-result. |

The reads are:

- The 32-bit capture of W.
- The global read supplying P.
- The 32-bit capture of V.
- The two unsigned byte lookups.

G-result is not counted as a known 32-bit destination. Likewise, the represented 32-bit capture from G-base does not supply its missing global declaration.

These counts exclude dependency-internal accesses and implicit machine effects.

### Ordering and aliasing

All explicit accesses retain the call-relative order described above. No evidence permits reordering them across dependencies.

Accessed locations are not proven disjoint. A word store may change a later lookup byte; the M store preceding V’s load may alter V; the two lookup-selected destinations may coincide; P-tail may alias any earlier destination; and the final halfword clear may overwrite earlier data.

Correctness therefore requires preserving intermediate accesses and widths, not merely equivalent final contents under an assumption of disjoint ordinary memory.

### Ownership, validity, and errors

No allocation, release, ownership transfer, reference counting, locking, or lifetime extension is established. Unknown dependencies may affect memory or ownership; they cannot be assumed to preserve either.

Continued execution requires valid callable targets, returning dependencies, and valid memory accesses at the time each is performed. Applicable alignment and lifetime requirements belong to the execution environment.

No explicit bounds checks, null checks, status checks, retries, rollback, or error return are represented. Failure can leave earlier writes committed.

The address-zero call may fault, transfer to platform-specific code, or behave otherwise under the environment. The static representation does not resolve that behavior.

## Dependencies

The established dependency contract consists of call order, represented arguments, and consumed results—not inferred application services.

Important result roles are:

- **F-C09, F-C14, F-C23, F-C28, F-C37, F-C47, F-C59, F-C64:** supply values used in scaled addresses or arguments.
- **F-C42 and F-C52:** supply represented 64-bit results whose halves have distinct uses.
- **F-C69:** supplies the directly assigned G-result value, with unresolved width.
- **F-C70:** targets address zero.

No dependency is known to preserve globals, pointed-to memory, ownership, or ambient registers. Captured values and later memory reads must remain distinct: retaining P does not freeze memory at P, and retaining W or V does not imply that their original locations remain unchanged.

## Acceptance examples

These are conditional static acceptance criteria, not observed executions. All preceding calls and accesses must complete, and any relevant arithmetic and memory accesses must be valid.

1. **Initial truncation and call ordering.**  
   If A is 0x100 and the word at 0x474 supplies 0x123456AB, the initial byte store writes 0xAB at 0x401. F-C10 subsequently receives 0x400.

2. **Different destination scales.**  
   If B and C both equal 3, W is stored first at 0x44 and later at 0x20. Substituting four-byte scaling for the first destination violates the contract.

3. **Third-input bit selection.**  
   With third input 0x00040007, F-C15 receives 0x00010000. The subsequent third-input-derived word destination is 0x00040018.

4. **Captured address, delayed data.**  
   If G-base supplies 0x1000 when captured, a later change of G-base to 0x2000 does not redirect V’s read: it remains at 0x1014. A change to memory at 0x1014 before that read does affect V.

5. **First lookup destination.**  
   If W is 0x01800000, the first lookup reads byte address 4. If that byte supplies 0x20 and V is 0x89ABCDEF, the following store writes 0xEF at 0x31.

6. **Second lookup and halfword clear.**  
   If T is 0xC0600100, its masked value is 0x00600100. The second lookup reads byte address 4, and the final halfword clear targets 0xC002. With V equal to 0x89ABCDEF, the second byte store writes 0x26 at the destination selected by the byte read at that later lookup.

7. **Mutable lookup source.**  
   If the two lookup addresses coincide but an intervening dependency changes the byte there, the second store uses the newly read byte to select its destination. Reusing the first lookup result is not equivalent.

8. **Unresolved global width.**  
   Acceptance cannot require a four-byte G-result assignment or preservation of adjacent bytes without additional declarations or platform evidence. The established requirement is the assignment after F-C69 and before the two clearing writes.

9. **No guaranteed return.**  
   If address zero is not callable, reaching F-C70 need not return normally. No represented mechanism rolls back earlier stores.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and semantic object or device layouts.
- Actual dependency identities, signatures, preservation rules, and side effects.
- Whether the represented 64-bit results correspond to scalar returns or reconstructed register pairs.
- F-C69’s result width and G-result’s type, assignment width, and layout.
- G-base’s declaration and the layout of potentially overlapping globals.
- P-tail’s origin, lifetime, ownership, and validity.
- Signed overflow, signed-shift edge cases, architectural address wrapping, and pointer interpretation.
- Endianness, access alignment requirements, and overlapping-access consequences.
- The mapping and meaning of low memory addresses.
- Hidden dependency failure reporting and runtime reachability.
- The effect of calling address zero.

The supplied warning about overlapping globals does not establish a complete global layout or resolve the final assignment’s width.

## Evidence

**Established static evidence:** the supplied representation supports the seventy call occurrences, ordered argument forwarding, integer transformations, six explicitly width-qualified word stores, four byte stores, one halfword store, one width-unresolved global assignment, five explicit reads, address-zero call, and nominal void return described here.

**Inferred intent:** repeated four-byte scaling may represent word-addressed references, and repeated offset 0x14 may designate a common field. Neither inference establishes a type, ownership model, or application purpose.

**Unknown:** dependency behavior, missing global declarations, platform execution semantics, runtime reachability, and successful completion.

No runtime tests have been performed.
