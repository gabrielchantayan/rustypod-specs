<!-- Trusted provenance: F-3d5bea7f0b2c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3d5bea7f0b2c

## Purpose

F-3d5bea7f0b2c is a stateful operation with two visible return paths and an unbounded repetition path. It invokes unresolved dependencies, conditionally writes through derived addresses, updates state for another iteration, and returns a 64-bit bit pattern when an exit condition is met.

The supplied static representation establishes explicit accesses, call ordering, arithmetic expressions, and control flow. It does not establish an application-level purpose, a complete calling convention, or dependency behavior.

Several register-derived operands have no displayed definitions. Their producers and their identities across calls are unresolved. In particular, appearing after a call does not establish that the call produced the value.

## Interface

### Displayed arguments

The displayed interface has four 32-bit arguments.

| Position | Established direct use |
|---|---|
| First | No explicit use is shown. |
| Second | No explicit use is shown. |
| Third | Passed to F-000000000002 on the update path; replaced after a complete update iteration. |
| Fourth | Written by the first two update-path stores; replaced after a complete update iteration. |

The absence of explicit uses for the first two arguments does not establish that they are irrelevant to machine execution. Implicit dependency inputs and the complete calling convention are not supplied.

### Additional state

Execution also depends on:

- An incoming link-register-derived value selecting the first call target.
- An incoming writable-base value used by the first update-path store.
- An additional incoming register value used as an address-like addend near the end of each complete update iteration.
- Three unresolved register-derived operand families.
- A stack-frame base exposed to a dependency.
- Global storage read at several points.

For description, the unresolved operand families are:

| Family | Visible uses |
|---|---|
| **A** | Gate-zero result calculation; second operand of F-000000000002; working-index calculation; destination calculation for the second word store. |
| **B** | Fourth operand of F-000000000007. |
| **C** | Calculation of the next iteration’s writable base and fourth argument. |

These labels describe recurring operands in the representation, not established registers in a complete machine model. No definitions are displayed for them.

Family A uses share one represented symbol, with no displayed assignment separating those uses. Nevertheless, neither a stable concrete register value across intervening calls nor a new output from any particular call is established. The contract therefore describes values at their use sites without assigning producers or asserting preservation across calls. The same restriction applies to families B and C.

### Return value

The result consists of two 32-bit halves:

- **Upper half:** the terminally read result word minus `0x54`, retaining 32 bits.
- **Lower half:** a path-selected 32-bit value shifted left by one, retaining 32 bits.

The path-selected value is either an arithmetic-right-shifted family A value or the low word of F-000000000003’s explicit 64-bit result.

No success-code convention, signedness of the complete result, or application interpretation is established.

### Widths and execution preconditions

The visible representation uses 32-bit arguments and working values, 32-bit address arithmetic, 32-bit word accesses, two 16-bit stores, one 8-bit read, and a 64-bit dependency result.

The arithmetic descriptions and numerical examples below use a **conditional machine interpretation**:

- Addition, subtraction, multiplication, and left shifts retain the low 32 bits where operating on 32-bit quantities.
- Signed right shifts propagate the sign bit.
- The scaled-sign branch interprets its 32-bit product as signed.

These conventions make the bit-level behavior precise but are not established portable language semantics. Signed overflow and signed shifts require target-specific interpretation.

Normal progress requires valid call targets and memory accesses, suitable alignment where required by the target, and dependencies that return. No explicit validation or recovery mechanism is visible.

## Behavior

### Global storage roles

Four labels describe global reads:

- **Gate word:** tested after the entry call in every iteration.
- **Mixing word:** sampled during a complete update path.
- **Result word:** read during terminal result assembly.
- **Final-call word:** supplied to the final dependency of a complete update iteration.

These are descriptive roles, not guarantees of independent storage. The supplied representation warns of overlapping global symbols. Aliasing with other accesses is not excluded.

### Entry call and gate decision

Each iteration begins with F-000000000001. Its target is formed from the current link-register-derived state plus `0xEDE`.

Initially, this state is incoming machine state. After a complete update iteration, the representation assigns it `0x0857344D` before the final call and subsequent repetition.

F-000000000001 has no displayed explicit arguments. Implicit register or stack inputs remain unknown.

After this call returns, the gate word is read.

If the gate is zero, the operation uses family A at that exit site as a signed 32-bit value, shifts it arithmetically right by nine, and proceeds to result assembly. There are no explicit update-path stores on this iteration.

The representation does **not** establish that this family A value was returned by F-000000000001. The dependency may also have produced side effects unrelated to the visible result expression.

If the gate is nonzero, execution enters the update path.

### Initial stores, calls, and second exit

First, the current fourth argument is stored as a 32-bit word at the current writable base plus `0x34`.

F-000000000002 is then invoked with three explicit operands:

1. The current third argument.
2. Family A at this operand-use site.
3. The current stack-frame base address.

The exposed stack area’s size, contents, permitted mutations, and lifetime are not specified.

F-000000000003 is invoked next, without displayed explicit arguments. Its explicit 64-bit result is split into low and high 32-bit words.

The working index is formed by adding ten to family A at the index-calculation site. The current fourth argument is then stored as a 32-bit word at family A’s store-address value plus `0x3E`.

The index calculation and this store address use the same unresolved operand symbol, with no intervening call or assignment displayed between them. Their occurrence after F-000000000003 does not establish that this dependency produced that operand.

The second destination is not guaranteed to be word-aligned.

The operation next tests whether the working index multiplied by `0x10000`, interpreted as a signed 32-bit result, is negative.

Under the stated machine interpretation, this tests bit 15 of the working index. It does not test the working index’s own sign, nor does it impose a monotonic bound on the full 32-bit index.

If the test is true:

- Both initial 32-bit stores have already occurred.
- The low word explicitly returned by F-000000000003 becomes the selected result value.
- Execution proceeds directly to result assembly.
- No subsequent update-path call or store occurs.

### Halfword stores

If the scaled-sign test is false, the low 16 bits of F-000000000003’s explicit result are stored at its high-word address plus `0x3C`.

F-000000000004 is then invoked without displayed explicit arguments.

F-000000000005 follows. Its explicit 32-bit result is used as an address, and the 16-bit value `0x0026` is stored at that address plus two.

There is no visible null check, range check, alignment check, or ownership check before either halfword store.

### Mixed-address calculation and update

A signed displacement is calculated from the working index: multiply by `0x10000000` at 32-bit width, then arithmetically shift the resulting 32-bit value right by nine.

The working index is added to that displacement, retaining 32 bits. The mixing word is then read and retained.

F-000000000006 is invoked with:

1. F-000000000005’s explicit 32-bit result shifted left by one.
2. The working index multiplied by `0x10000`, then reduced by `0x51`.

After the call, the previously formed sum is exclusive-ored with the previously sampled mixing word. This produces the mixed address.

Consequently, a modification to the mixing global during F-000000000006 does not replace the sample used by this explicit calculation.

The displacement is increased by `0x2F`, producing the update value. The mixed address is saved as a 32-bit value in the stack slot at frame offset `0x130`.

One byte is read at the mixed address plus one. F-000000000007 is then invoked with:

1. The literal `0x085734E8`.
2. The byte just read.
3. The update value.
4. Family B at this operand-use site.

Family B has no displayed definition. In particular, its use after F-000000000006 does not establish that F-000000000006 produced it.

After F-000000000007 returns, the update value is stored as a 32-bit word at the mixed address plus `0x24`. The call therefore precedes this explicit store.

No semantic type or ownership role is established for the literal, mixed address, or pointed-to storage.

### Loop-carried state

F-000000000008 is invoked without displayed explicit arguments.

At the subsequent calculation site, family C is increased by `0x30`. Twice this adjusted value becomes the next writable base.

The next iteration’s argument state is also replaced:

- The third argument becomes the mixed address shifted left by one.
- The fourth argument becomes the square of the newly derived writable base.

All these results retain 32 bits under the stated machine interpretation. Family C is not established to be an output of F-000000000008 merely because its use follows that call.

F-000000000009 is then invoked with the mixed address plus the additional incoming register addend.

The link-register-derived state is assigned `0x0857344D`. The final-call word is read and supplied as the sole displayed argument to F-00000000000a.

Execution repeats from F-000000000001. Completing the update path is not itself a return condition, and no iteration limit is visible.

### Terminal result assembly

Both exits share terminal result assembly.

The result word is read at this point. Its value minus `0x54`, reduced to 32 bits, supplies the upper half. This is not an entry-time snapshot; prior calls and aliasing writes may affect the read.

The lower half is formed by shifting the selected 32-bit value left by one:

- **Gate-zero exit:** family A at the exit-use site, first arithmetically shifted right by nine.
- **Scaled-sign exit:** the low word explicitly returned by F-000000000003.

No separate error indicator or exceptional result encoding is shown.

## Side effects

On a complete update iteration, the explicit memory accesses occur in this order, with the dependency calls described above interleaved:

1. Read the gate word after F-000000000001.
2. Store the current fourth argument at writable base plus `0x34`.
3. Store that argument at the family A-derived address plus `0x3E`.
4. Store a 16-bit truncation of the explicit 64-bit result at its high-word address plus `0x3C`.
5. Store the halfword `0x0026` at F-000000000005’s result address plus two.
6. Read the mixing word.
7. Store the mixed address in the stack slot at offset `0x130`.
8. Read a byte at mixed address plus one.
9. Store the update value at mixed address plus `0x24`.
10. Read the final-call word.

The result-word read occurs only on an exit. The gate-zero exit performs none of the listed update-path stores. The scaled-sign exit occurs after the first two word stores.

This ordering does not establish that destinations are distinct. Overlap can make earlier stores affect later reads. The exposed stack frame must not be assumed private or inaccessible to dependencies.

Dependencies can have additional effects not described by the explicit accesses. There are no established guarantees about global preservation, hidden register preservation, allocation, release, synchronization, or retained pointers.

Passing a stack address does not establish ownership transfer. No explicit allocation or release is identifiable. Invalid addresses, unavailable targets, alignment faults, and nonreturning dependencies have no visible recovery path.

## Dependencies

These opaque IDs identify call sites for this contract. They do not establish distinct underlying callees.

| ID | Established explicit interaction |
|---|---|
| F-000000000001 | Iteration-entry call with a link-register-derived target; no displayed explicit arguments. |
| F-000000000002 | Receives the current third argument, family A at its operand site, and the stack-frame base. |
| F-000000000003 | Returns an explicit 64-bit value used for the early-exit result and a later halfword store. |
| F-000000000004 | Called after the first halfword store; no displayed explicit arguments. |
| F-000000000005 | Returns an explicit 32-bit value used as a store address and a later call operand. |
| F-000000000006 | Receives two scaled arithmetic values after the mixing-word sample. |
| F-000000000007 | Receives a literal, a loaded byte, the update value, and unresolved family B. |
| F-000000000008 | Called before calculations using unresolved family C; no production of C is established. |
| F-000000000009 | Receives the mixed address plus an incoming register-derived addend. |
| F-00000000000a | Receives the final-call word before repetition. |

No dependency is established as the producer of families A, B, or C.

F-882dff43a182 identifies the shared terminal destination in the supplied mapping. The representation shows a transfer to that destination within this operation, not a separate dependency invocation with an established independent interface.

## Acceptance examples

These are conditional static criteria, not executed tests. They assume returning dependencies, valid accesses, and the stated 32-bit machine interpretation.

### Gate-zero result

Suppose:

- The gate read yields zero.
- Family A equals `0x00000400` at the gate-zero calculation.
- The terminal result-word read yields `0x00000100`.

The arithmetic right shift produces two. The lower result half is four, and the upper half is `0x000000AC`. The returned bit pattern is `0x000000AC00000004`.

No explicit update-path store occurs. This example places no requirement on which operation produced family A.

### Scaled-sign exit after two stores

Suppose the gate is nonzero, family A at the adjacent index and second-store calculations is `0x00007FF6`, and F-000000000003 explicitly returns low word `0x80000001`.

The working index is `0x00008000`. Its scaled 32-bit product is negative, so execution exits after both initial word stores. The lower return half is `0x00000002`.

No halfword store or later update-path call occurs. The example does not attribute the family A value to F-000000000003 or require it to equal family A’s earlier operand value.

### Bit boundary rather than range bound

A working index of `0x00007FFF` continues past the scaled-sign test. An index of `0x00008000` exits.

An index of `0x00010000` also continues: multiplying it by `0x10000` yields zero at 32-bit width. Rejecting every index at or above `0x8000` would not match this conditional behavior.

### Sampling across calls

If F-000000000006 changes the mixing global after its explicit sample, the mixed-address calculation still uses the earlier sample.

By contrast, the upper return half uses the result word read at terminal assembly, not a pre-call snapshot.

### Repetition state

Suppose family C equals one at its calculation site. The adjusted value is `0x31`, the next writable base is `0x62`, and the next fourth argument is `0x00002584`.

A complete update iteration installs these values, replaces the third argument with twice the mixed address, performs the remaining calls, and repeats. This criterion does not require F-000000000008 to have produced family C.

## Uncertainties

The following remain **documented_only**:

- Producers and cross-call identities of all unresolved register-derived families.
- Whether their repeated symbolic uses accurately describe concrete register continuity.
- Architecture, calling convention, instruction encoding, and byte order.
- Completeness of the represented control flow and implicit inputs.
- Target semantics for signed overflow, shifts, and unaligned accesses.
- Semantic types, accessible extents, and validity of derived addresses.
- Dependency side effects, preservation guarantees, failure behavior, and ownership rules.
- Stack-area size, initial contents, permitted retention, and lifetime.
- Aliasing among globals, stack storage, and derived destinations.
- Termination conditions beyond the two visible exit tests.
- Interpretation of the packed return value.
- Any separately meaningful entry interface for F-882dff43a182.

Inferred intent is limited to the observation that the complete update path constructs state for another iteration. No domain-level interpretation is justified.

## Evidence

This contract is based solely on static inspection of the supplied representation and opaque dependency mapping.

Established evidence includes displayed argument widths, explicit dependency results, access widths and ordering, branch expressions, stack-address exposure, loop-carried assignments, and two-half result construction.

The unresolved register-derived operands are declared and used without displayed definitions. Their placement near calls does not establish call-specific provenance. Their repeated symbolic appearance does not resolve preservation across intervening calls.

Numerical examples depend on the explicitly stated 32-bit machine interpretation. No runtime tests, dependency inspection, or implementation-access verification have been performed.
