<!-- Trusted provenance: F-c29ef7e5f4a8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c29ef7e5f4a8

## Purpose

F-c29ef7e5f4a8 repeatedly performs an ordered sequence of indirect calls, memory accesses, arithmetic transformations, and a software breakpoint. No normal return or termination condition is visible.

The supplied static representation establishes operational behavior, not a domain-level purpose. Descriptions such as initialization, scheduling, device control, or object management would be inferred intent rather than established behavior.

This contract distinguishes:

- **Established static evidence:** visible operations, operand widths, constants, ordering, and dataflow.
- **Inferred intent:** interpretations not required to explain those operations.
- **documented_only:** behavior that cannot be resolved from the supplied representation.

All examples are conditional static examples. No runtime tests have been performed.

## Interface

### Explicit interface and completion

The function has no explicit parameters and no explicit return value.

Execution enters an unconditional loop. After the last dependency call returns, another iteration begins. Dependencies, inaccessible memory, invalid call targets, breakpoint handling, or external intervention may prevent progress; the representation does not specify their runtime consequences.

### Ambient inputs

The effective inputs include:

- An incoming link-like value used to construct the first indirect call target.
- An incoming register-like value used in a late byte-store destination.
- A special address-base value used for several accesses and explicitly decreased during each iteration.
- Two word-sized values loaded from separate global storage locations.
- Memory reachable through those values, dependency results, fixed addresses, and computed addresses.
- Additional result channels associated with three dependency calls.

These are not established ordinary parameters. Their initialization, validity, and ABI interpretation remain documented_only.

### Widths and layouts

For this contract:

- A **byte** is 8 bits.
- A **halfword** is 16 bits.
- A **word** is 32 bits.
- A **paired result** has a low 32-bit portion and a high 32-bit portion.

The paired-result notation describes visible extraction of two portions; it does not establish a particular return convention.

Unsigned halfword loads produce values from 0 through 65535. Subsequent word stores of those values are zero-extended. Byte and halfword stores retain the low 8 and low 16 bits, respectively.

All offsets are byte offsets. Names such as **working base**, **record base**, and **selector** describe dataflow only. They do not establish structures, allocation boundaries, pointer validity, or semantic field types.

The representation uses 32-bit integer and address-like quantities. Unsigned 32-bit shifts have the represented logical or low-32-bit result behavior. Some other expressions use signed-looking additions, shifts, or multiplication; their exact overflow and machine-level address behavior remain documented_only. Examples below avoid relying on unresolved signed overflow unless explicitly qualified.

### Preconditions for continued progress

A complete iteration requires every reached call target and memory access to be valid in the execution environment. Dependencies must return, and execution must resume after the breakpoint.

The function visibly enforces none of these requirements. In particular, there are no bounds checks, null checks, alignment checks, or explicit checks of dependency success.

## Behavior

Dependency IDs F-000000000001 through F-000000000037 identify call sites in encounter order. They do not prove that the resolved implementations are distinct.

A call described without arguments has no explicit arguments in the supplied representation. It may still consume ambient state. Results described below are the results visibly consumed by this function, not necessarily a complete ABI description.

### 1. Opening calls and working-base stores

Each iteration begins with F-000000000001, supplied with two zero arguments. Its target is formed from the current link-like value plus 0xFFE.

F-000000000002 and F-000000000003 follow. A word is then loaded from the first global storage location and retained as the **working base**.

F-000000000004 is called, followed by F-000000000005, which supplies the **first paired result**. F-000000000006 receives:

- The first result’s low word.
- An expression adding its high word, its low word, and a byte whose value is zero.

The zero contribution does not change the arithmetic value where that addition is defined. Signed-looking overflow in the addition is not independently resolved.

A byte from an additional result channel associated with F-000000000006 is stored at working base + 0x11. F-000000000007 then executes, after which a zero byte is stored at working base + 0x15.

The working base is sampled before F-000000000004 and is not visibly reloaded for either store. A later modification of the originating global does not itself change the retained base in the represented dataflow.

### 2. Halfword writes and second paired result

F-000000000008 supplies a halfword result and an additional address-like result channel. The halfword is stored at that address + 0x0C.

F-000000000009 supplies the **second paired result**. The function then performs, in order:

- A zero halfword store at the second result’s high word + 0x2C.
- A word store at special base + 0x26C containing the second result’s low word logically shifted right by 26 bits.

The latter stored value is between 0 and 63.

F-000000000010 executes. A word is then loaded from special base + 0x320 and retained as the **secondary base**. F-000000000011 and F-000000000012 follow.

The second result’s low word, shifted left by two bits as an unsigned 32-bit quantity, is stored at secondary base + 0x54. Only the low 32 result bits are retained.

The secondary-base load occurs after F-000000000010 and before F-000000000011. There is no reload immediately before its associated store.

### 3. First selector and breakpoint

F-000000000013, F-000000000014, F-000000000015, F-000000000016, and F-000000000017 execute in that order.

An unsigned halfword is loaded from fixed address 0x21C48994 and retained as the **first selector**. F-000000000018 follows.

A word is loaded from special base + 0x280 and passed to F-000000000019. That dependency supplies a word-sized result.

A software breakpoint with immediate value 0x92 occurs next. It is unconditional in the visible sequence and precedes the record-base load and every operation described in the remaining phases.

If execution resumes:

- The result of F-000000000019 is shifted left by two bits to produce the **table displacement**.
- A constant-derived **index base** is formed as 0x21C4.
- A word is loaded from first selector + 0x44 and retained as the **record base**.

The index-base calculation derives from shifting 0x21C48960 right by 18 bits and multiplying by four.

The first selector is a zero-extended 16-bit value used directly in address arithmetic. No additional pointer-table lookup or validity check precedes the record-base load.

### 4. Table and returned-address writes

F-000000000020 executes, followed by F-000000000021, which supplies a word result.

A word is loaded from first selector + 0x04. F-000000000022 receives the result of F-000000000021 as its first argument and this loaded word as its second argument.

A word is then loaded from the second global storage location and retained as the **table base**. F-000000000023 follows.

A halfword containing the low 16 bits of the table base is stored at table base + table displacement. This is a store of bits from the base itself, not of the selector or the result from F-000000000021.

F-000000000024 executes. A word is loaded from index base + 0x54, which is fixed address 0x2218, and retained as the **selected base**.

F-000000000025 supplies an address-like word result. A halfword containing 0x21C4 is stored at that returned address + 0x04.

The table base is sampled between F-000000000022 and F-000000000023. The selected base is sampled after F-000000000024 and before F-000000000025.

### 5. Second selector and special-base decrease

F-000000000026 and F-000000000027 execute.

An unsigned halfword is loaded from selected base + 0x34 and retained as the **second selector**. F-000000000028 then supplies the **third paired result**.

Its low word is stored at the then-current special base + 0x258. This store precedes the explicit special-base decrease.

An unsigned halfword is loaded from the third result’s high word + 0x14 and retained as the **field value**.

The following derived values and state change occur:

- The third result’s low word is shifted left by six bits to form the **large displacement**.
- Its high word is multiplied by four to form the **scaled high word**.
- The special base is decreased by 0xF4.
- The second selector is multiplied by four to form the **selector displacement**.

The third result’s low word is then stored as a word at selector displacement + 0x74.

The selector displacement is at most 262140. The selector-based destination calculation therefore does not require overflow for a 16-bit selector. Arithmetic involving arbitrary low and high words has the signed-operation uncertainties described under Interface.

### 6. Record writes and byte transfer

F-000000000029 executes. The field value is stored as a zero-extended word at record base + 0x54.

F-000000000030 follows. The same retained field value is stored as a zero-extended word at record base + 0x64. The field is not visibly reloaded between these stores.

F-000000000031 executes. A halfword containing the low 16 bits of the record base is stored at large displacement plus an additional address-like result channel associated with that call.

F-000000000032 and F-000000000033 follow.

A byte is loaded from second selector + selector displacement. This address is mathematically five times the second selector, ranging from zero through 327675.

F-000000000034 executes before that saved byte is stored at:

**field value + incoming register-like value + 0x19**

The byte is not visibly reread after F-000000000034. Consequently, the call separates the source read from the destination write.

F-000000000035 executes. The scaled high word is stored as a word at record base + 0x44.

F-000000000036 follows. The link-like value is assigned the fixed value 0x08712031. A word is loaded from scaled high word + 0x44 and passed to F-000000000037.

### 7. Repetition

After F-000000000037 returns, execution repeats without a visible exit test.

The first iteration’s initial call target depends on the incoming link-like value. In the represented dataflow, later iterations use the fixed value assigned near the end of the preceding iteration when constructing that first target.

The explicit special-base decrease is not visibly undone. Any additional changes made by dependencies remain unknown; no preservation or restoration guarantee is established.

## Side effects

### Direct memory effects

Each complete iteration contains **15 explicit stores**:

| Width | Count | Destinations, in encounter order within that width |
|---|---:|---|
| Byte | 3 | working base + 0x11; working base + 0x15; field value + incoming register-like value + 0x19 |
| Halfword | 5 | additional result address + 0x0C; second paired high word + 0x2C; table base + table displacement; F-000000000025 result + 0x04; large displacement + F-000000000031 additional result |
| Word | **7** | special base + 0x26C; secondary base + 0x54; pre-decrease special base + 0x258; selector displacement + 0x74; record base + 0x54; record base + 0x64; record base + 0x44 |

These counts cover only explicit stores in the function. Dependency-internal writes are unknown and do not add any established stores to this count.

Other direct effects include the ordered memory reads described above, the special-base decrease, the link-like value assignment, and the software breakpoint.

### Aliasing and ordering

No disjointness guarantee exists among globals, computed addresses, or dependency-accessible memory. Later reads may therefore observe earlier writes when addresses overlap.

Dependencies occur between several related accesses. In particular:

- F-000000000030 separates the two field-value word stores.
- F-000000000034 separates the final byte read from its write.
- Several sampled bases remain in use after intervening calls.

The representation establishes sequence, not platform-wide memory ordering, atomicity, cache behavior, or memory-mapped-device semantics.

### Ownership and errors

No allocation, release, ownership transfer, or lifetime-management operation is explicitly established. Returned address-like values do not prove ownership or any release obligation.

No explicit error return, status validation, rollback, cleanup path, or retry branch is visible. A fault, non-returning dependency, or non-resuming breakpoint handler may stop progress after earlier effects have occurred.

## Dependencies

All dependency IDs are opaque call-site identifiers scoped to this contract.

| Dependency | Explicit interaction |
|---|---|
| F-000000000001 | Two zero arguments; target derived from link-like value + 0xFFE |
| F-000000000005 | First paired result |
| F-000000000006 | Two word-sized arguments; additional byte result channel |
| F-000000000008 | Halfword result and additional address-like result channel |
| F-000000000009 | Second paired result |
| F-000000000019 | Loaded word argument and word-sized result |
| F-000000000021 | Word result |
| F-000000000022 | Two word arguments |
| F-000000000025 | Address-like word result |
| F-000000000028 | Third paired result |
| F-000000000031 | Additional address-like result channel |
| F-000000000037 | Loaded word argument |

The remaining sites have no explicit arguments or consumed ordinary results in the representation.

Later call targets are constructed from fixed values and offsets. Call-site numbering does not establish unique target identities or dependency semantics.

No dependency is established to preserve globals, ambient registers, the special base, or memory. None is established to allocate or release memory or report failure through a particular channel.

## Acceptance examples

These examples are not executed tests. Unless stated otherwise, they assume valid accesses, returning dependencies, and breakpoint continuation.

### Unsigned second-result shifts

If F-000000000009 supplies low word 0xFC000003 and high word H:

- A zero halfword is stored at H + 0x2C.
- The word written at special base + 0x26C is 63.
- The later word written at secondary base + 0x54 is 0xF000000C.

The last result follows from the explicitly unsigned 32-bit left shift.

### Table-base truncation

If the retained table base is 0x12345678 and the table displacement is 12, the halfword destination is 0x12345684 and the stored value is 0x5678.

This store follows F-000000000023 and precedes F-000000000024. It does not store the first selector.

### Field zero-extension

If the halfword loaded from the third paired result’s high word + 0x14 is 0xFFFF, both record-field word stores contain 0x0000FFFF.

F-000000000030 separates those stores. Neither store sign-extends the value to 0xFFFFFFFF.

### Selector-addressed transfer

If the second selector is 7, its selector displacement is 28:

- The third result’s low word is stored at address 0x90.
- The later source byte is read from address 35.
- That saved byte is written only after F-000000000034.

If the selector is zero, the byte source address is zero. No protective branch is visible.

### Store count and aliasing

If a complete iteration runs, seven explicit word-store operations occur at the seven destinations listed under Side effects.

Even if two destinations resolve to the same address, they remain separate store operations in the visible sequence. Unknown dependency effects neither establish nor justify an eighth explicit word store.

### Breakpoint interruption

If execution does not resume after the breakpoint, the earlier calls and stores remain preceding effects. The record-base load and all later operations in that iteration are not reached. No rollback is visible.

### Conditional special-base progression

If dependencies leave the special base unchanged and an iteration begins with value B, its third-result store uses B + 0x258. The explicit decrease subsequently leaves B − 0xF4 for the next iteration, provided that subtraction has the expected machine interpretation.

This is conditional on dependency behavior, not a preservation guarantee.

## Uncertainties

- **documented_only — environment:** Instruction set, privilege level, alignment rules, byte order, and address-space permissions are not established.
- **documented_only — calling convention:** Ordinary results, paired results, additional channels, and ambient inputs lack a complete ABI description.
- **documented_only — arithmetic boundaries:** Signed-looking additions, left shifts, multiplication, and address overflow lack independently verified machine semantics.
- **documented_only — dependency behavior:** Target identity, ambient-state preservation, memory effects, ownership effects, and failure behavior are unknown.
- **documented_only — breakpoint:** The environment may resume, trap, terminate, or transfer control elsewhere.
- **documented_only — memory interpretation:** Low computed addresses and the changing special base may represent actual machine behavior or limitations of the static representation.
- **Inferred intent only:** The repeated writes may resemble state preparation or transfer. No specific application purpose is established.

## Evidence

The sole evidence is the supplied static representation of F-c29ef7e5f4a8.

It establishes an unconditional loop containing 37 indirect call sites, the ordered accesses and dataflow described above, an unconditional breakpoint with immediate 0x92, and an explicit special-base decrease of 0xF4.

The explicit store inventory is three byte stores, five halfword stores, and **seven word stores**. The seven word destinations are individually enumerated under Side effects; dependency-internal effects remain unknown.

No runtime execution, external documentation checks, or implementation-level validation has been performed.
