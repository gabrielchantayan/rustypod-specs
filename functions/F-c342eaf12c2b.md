<!-- Trusted provenance: F-c342eaf12c2b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c342eaf12c2b

## Purpose

F-c342eaf12c2b performs an ordered sequence of indirect dependency invocations, three direct memory stores, and a final indirect transfer whose destination and continuation are unresolved.

The supplied static representation establishes arithmetic expressions, recovered value dependencies, store widths, and operation order. It does not establish an application-level purpose such as device initialization, serialization, or object management.

A central limitation concerns values associated with incoming registers. The representation explicitly captures one such value before the first call, while later expressions refer again to incoming-register-associated symbols. Those symbolic references do not independently prove that intervening dependencies preserve the corresponding machine registers.

This contract distinguishes:

- **Established static evidence:** operations and symbolic dataflow present in the supplied representation.
- **Conditional interpretation:** behavior requiring additional assumptions, particularly register preservation or signed-shift semantics.
- **`documented_only` unknowns:** unavailable guarantees that must not be inferred from normal dependency return.

No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Recovered width and interpretation | Use |
|---|---|---|
| First explicit input | Signed 32-bit integer | Increased by 50 for F-000000000001. |
| Second explicit input | Opaque 32-bit value | Forwarded unchanged to F-000000000001. |

The second input is not established to be an address, handle, owned object, or numeric quantity.

### Additional execution-state inputs

The apparent two-input interface is incomplete. The representation also depends on:

- An incoming-register-associated signed 32-bit value, called **A** here.
- An incoming-register-associated unsigned 32-bit value, called **B** here.
- Incoming link state used to derive the first indirect destination.
- An unresolved destination for the final transfer.

Two calls also expose auxiliary-register results distinct from their ordinary results.

### Entry capture versus later symbolic references

The following distinctions are required:

| Contract label | Meaning in the recovered representation |
|---|---|
| **A-entry** | A’s incoming value. |
| **A-saved** | An explicit capture of A-entry before F-000000000001. This captured value supplies the store at 0x00000041. |
| **A-byte** | The later reference to A used for the byte store, after F-000000000001 and F-000000000002. |
| **A-late** | The later reference to A used in the subtraction-and-shift argument to F-000000000006. |
| **B-entry** | B’s value used for the first call’s adjusted argument, before any dependency has been invoked. |
| **B-store** | The later reference to B used for the store at 0x00000045. |
| **B-arguments** | The later references to B used for F-000000000003’s two arguments. |

These labels distinguish use sites; they do not establish additional inputs, actual register changes, or separate physical register-read instructions.

The supplied representation uses the same incoming-associated symbol for A-entry, A-byte, and A-late, and likewise for the B references. That is recovered symbolic dataflow, not an independently verified machine-level preservation guarantee.

Consequently:

- A-saved is explicitly represented as the pre-call capture of A-entry.
- Equality of A-byte or A-late to A-entry requires the relevant preservation or equivalent dataflow guarantee.
- Equality of B-store or B-arguments to B-entry requires the relevant preservation or equivalent dataflow guarantee.
- Normal return from an intervening dependency is not sufficient evidence of preservation.

### Outputs and completion

No explicit return value is established. Ordinary call results are consumed internally where specified below.

The final transfer is represented as a call followed by return, but an accompanying diagnostic identifies unresolved dispatch. Normal return to the caller is therefore not guaranteed by the available evidence.

### Preconditions

Execution of the complete represented sequence requires:

- Valid indirect destinations and compatible calling conventions.
- A writable byte at displacement five from F-000000000002’s result.
- Writable 32-bit destinations beginning at 0x00000045 and 0x00000041.
- Sufficiently meaningful incoming execution state.
- Each preceding dependency to resume execution at the next represented operation.

There is no visible validation of these conditions. Register preservation is an additional prerequisite only for claims that equate later register-associated references with entry values.

## Behavior

### Dependency identifiers

F-000000000001 through F-00000000000d are opaque documentation IDs for call or transfer sites. Distinct IDs do not prove distinct implementations.

### Initial invocation and captured state

Before the first dependency invocation, the representation captures A-entry as A-saved.

F-000000000001 has a destination derived from incoming link state plus 3,360 bytes. It receives, in order:

1. The first explicit input plus 50.
2. The unchanged second explicit input.
3. B-entry plus 51.

The third expression is unsigned 32-bit addition, with a result modulo \(2^{32}\). The first expression is represented as signed 32-bit addition; behavior outside the signed representable range is unresolved.

No ordinary result from this call is subsequently consumed.

### Address result and direct stores

F-000000000002 is invoked next, with no explicit arguments shown. Its ordinary result is a 32-bit address-like value.

The following direct stores occur in order:

| Order | Destination | Width | Value |
|---:|---|---:|---|
| 1 | F-000000000002’s result plus five bytes | 8 bits | Byte-sized conversion of A-byte |
| 2 | Absolute address 0x00000045 | 32 bits | B-store shifted left by 22 bits |
| 3 | Absolute address 0x00000041 | 32 bits | A-saved |

For the byte store, the expected eight-bit-byte interpretation retains A-byte’s lowest eight bits. Signed versus unsigned character interpretation does not change those stored bits.

The unsigned left shift retains a 32-bit result. Only B-store’s lowest ten bits can contribute.

The last store uses the explicit pre-call capture, not the later A reference used by the byte store. Thus the representation does not justify describing both stores merely as writes of an unqualified, preserved incoming A.

The two fixed 32-bit ranges are adjacent and do not overlap each other under the eight-bit-byte interpretation:

- 0x00000041 through 0x00000044.
- 0x00000045 through 0x00000048.

The higher-addressed range is written first. The representation warns that these global regions overlap smaller named storage objects; it does not establish independent fields or ownership boundaries.

The byte destination is not constrained against aliasing either range. A later fixed-address store may overwrite the earlier byte.

### Unsigned argument preparation

After the stores, F-000000000003 receives:

1. B-arguments shifted left by 22 bits, retaining a 32-bit unsigned result.
2. B-arguments shifted right logically by four bits.

The representation recomputes the first argument from its B reference; it does not reload the stored value at 0x00000045. No dependency invocation is shown between that store and this call. The expressions share the same recovered B symbol, but their relationship to B-entry remains dependent on the earlier calls’ preservation or equivalent dataflow.

An unsigned 32-bit auxiliary-register result associated with F-000000000003 supplies the next argument. Its ordinary result is not used for that purpose.

F-000000000004 receives that auxiliary result shifted right logically by 17 bits. Its argument is therefore between zero and 32,767 inclusive. No ordinary result from F-000000000004 is subsequently consumed.

### Ordinary-result forwarding and signed arithmetic

F-000000000005 is invoked with no explicit arguments shown. Its opaque 32-bit ordinary result is forwarded to F-000000000006.

F-000000000006 receives:

1. The ordinary result from F-000000000005.
2. A-late minus nine, followed by a signed right shift of nine bits.

The subtraction precedes the shift. It is not subtraction of an already shifted value.

A-late is the later reference to the incoming-associated A symbol, not the explicit A-saved capture used for the fixed-address store. The representation alone does not prove A-late equals A-entry after the intervening calls.

Signed subtraction overflow and right-shift behavior for negative operands remain unresolved.

### Auxiliary signed result and further forwarding

F-000000000007 is invoked with no explicit arguments shown.

A signed 32-bit auxiliary-register result associated with this call is shifted right by 18 bits and supplied to F-000000000008. This auxiliary result must not be substituted with F-000000000007’s ordinary result.

F-000000000008 produces a signed 32-bit ordinary result. F-000000000009 receives:

1. That unchanged ordinary result.
2. The same result shifted right by 14 bits as a signed value.

These arguments originate from the ordinary dependency result, not a fixed-address memory reload.

### Remaining invocations and final transfer

F-00000000000a, F-00000000000b, and F-00000000000c are invoked in that order. No explicit arguments are shown, and their ordinary results are not subsequently consumed by the recovered sequence.

Control then transfers through F-00000000000d. Its destination, inputs, dispatch semantics, and eventual continuation are unresolved.

## Side effects

### Direct effects and ordering

The represented sequence contains:

- Twelve indirect dependency calls before the final transfer.
- One unresolved final indirect call or transfer site.
- One byte store through a dependency result.
- Two fixed-address 32-bit stores.

The direct store order is established. A complete memory-access trace is not: dependencies may perform additional reads, writes, allocation, release, synchronization, or external operations.

No direct allocation, release, reference-count operation, lock, status check, or error-reporting operation is visible in F-c342eaf12c2b itself.

### Register and global state

Unknown dependency register preservation must not be conflated with unknown global preservation.

For registers, the specific limitation affects interpreting repeated incoming-associated symbols as unchanged entry values. A-saved remains a distinct explicit capture in the recovered dataflow.

For memory, no dependency is established to preserve either fixed-address region or unrelated globals. The final contents of those regions therefore cannot be inferred solely from the direct stores if later dependencies return.

The absence of explicit arguments at a call site does not imply absence of register, stack, flag, or global inputs.

### Ownership and lifetime

F-000000000002’s result is not established to identify newly allocated memory or an owned object. The byte destination is not established to be a field.

No explicit release of this result is shown. This establishes neither a leak nor a caller-release obligation.

Other ordinary and auxiliary results likewise carry no established ownership or lifetime guarantee.

### Failure and interruption

There are no visible null checks, bounds checks, dependency-status checks, retries, or rollback operations.

If a dependency does not resume the represented sequence, subsequent operations need not occur. If a store fails, earlier effects are not visibly undone.

The response to inaccessible or misaligned memory is platform-dependent and unresolved here.

## Dependencies

| Opaque ID | Explicit inputs shown | Consumed output |
|---|---|---|
| F-000000000001 | Adjusted first input; second input; adjusted B-entry | None |
| F-000000000002 | None shown | Address-like 32-bit ordinary result |
| F-000000000003 | Left-shifted and logically right-shifted B references | Unsigned 32-bit auxiliary result |
| F-000000000004 | Auxiliary result logically shifted right 17 | None |
| F-000000000005 | None shown | Opaque 32-bit ordinary result |
| F-000000000006 | Previous ordinary result; signed expression using A-late | None |
| F-000000000007 | None shown | Signed 32-bit auxiliary result |
| F-000000000008 | Auxiliary result signed-shifted right 18 | Signed 32-bit ordinary result |
| F-000000000009 | Previous ordinary result; that result signed-shifted right 14 | None |
| F-00000000000a | None shown | None |
| F-00000000000b | None shown | None |
| F-00000000000c | None shown | None |
| F-00000000000d | None established | Continuation unresolved |

Apart from the first destination’s dependence on incoming link state, ordinary call destinations are represented by fixed address-derived expressions. Dependency implementations, signatures, hidden inputs, preservation rules, and error behavior are unavailable.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. They constrain the stated call arguments or direct stores, not independent dependency effects.

### Entry values with required preservation

Assume:

- The first explicit input is 100.
- A-entry is 521 and B-entry is 291.
- All reached dependencies resume the represented sequence.
- Required preservation or equivalent dataflow makes A-byte and A-late equal 521, and B-store and B-arguments equal 291.
- The byte destination is writable and does not alias either fixed-address range.

Expected observations:

- F-000000000001 receives 150 as its first argument and 342 as its third.
- The byte store writes 9.
- The direct store at 0x00000045 writes 0x48C00000.
- The following direct store at 0x00000041 writes 521 from A-saved.
- F-000000000003 receives 0x48C00000 and 18.
- F-000000000006’s second argument is 1, because subtracting nine from 521 gives 512, whose right shift by nine is 1.

Normal dependency return alone does not satisfy the preservation assumption.

### Entry capture without a preservation guarantee

If A-entry is 521, the explicit capture is represented as A-saved equal to 521. If execution reaches the fixed-address store, that store writes A-saved.

Without further preservation evidence, the entry value alone does not determine the byte store or F-000000000006’s second argument. In particular, neither 9 nor 1 can be required for those later uses merely because all preceding dependencies returned normally.

This criterion does not assert that dependencies actually change A; it preserves the unresolved relationship.

### Unsigned truncation and wraparound

If B-entry is 1,024, the third argument to F-000000000001 is 1,075.

Separately, if B-store and B-arguments are 1,024 at their represented uses:

- The 32-bit left-shifted value is zero.
- The direct store at 0x00000045 writes zero.
- F-000000000003 receives zero and 64.

Equality of those later values to B-entry requires the relevant preservation guarantee.

If B-entry is 0xFFFFFFFF, the first call’s third argument is 50. If the later B references also equal 0xFFFFFFFF, the fixed-address shifted store is 0xFFC00000 and F-000000000003’s second argument is 0x0FFFFFFF.

### Auxiliary outputs and signed shifts

If F-000000000003 supplies 0xFFFFFFFF as its identified unsigned auxiliary result, F-000000000004 receives 32,767 regardless of that call’s ordinary result.

Under arithmetic signed-right-shift semantics:

- A-late equal to zero makes F-000000000006’s second argument negative one.
- A signed auxiliary result of negative one from F-000000000007 makes F-000000000008’s argument negative one.
- A result of negative one from F-000000000008 makes both arguments to F-000000000009 negative one.

The first criterion concerns A-late, not necessarily A-entry.

### Aliasing and interruption

If the byte destination equals 0x00000045, the byte store precedes the 32-bit store there and may be overwritten.

If F-000000000004 never resumes the represented sequence, F-000000000005 and later sites are not reached through that sequence. All three direct stores precede F-000000000004; no rollback is shown.

## Uncertainties

- **`documented_only` — register preservation:** Equality between repeated incoming-associated references and entry values is not machine-level established. The explicit A-saved capture must remain distinguished from later A references.
- **`documented_only` — calling convention:** Hidden inputs, auxiliary-result conventions, register preservation, and result validity are unresolved.
- **`documented_only` — signed overflow:** The first input plus 50 and A-late minus nine can exceed signed 32-bit range. Wrapping or trapping is not established.
- **`documented_only` — negative signed shifts:** Arithmetic sign extension is a conditional interpretation, not independently verified behavior.
- **`documented_only` — address arithmetic:** Address bounds, validity, and overflow treatment are unknown.
- **`documented_only` — memory attributes:** Endianness, alignment tolerance, volatility, atomicity, and device-memory semantics are unknown.
- **`documented_only` — dependency effects and ownership:** No memory-preservation, allocation, retention, or release guarantees are established.
- **`documented_only` — final control flow:** The final destination, dispatch mechanism, and return behavior are unresolved.
- **`documented_only` — application purpose:** No domain-level role is established.

Argument preparation is a plausible interpretation of the arithmetic, but adds no behavioral guarantee.

## Evidence

The supplied static representation supports the ordered dependency sites, three direct stores, arithmetic constants, operand widths and signedness, ordinary-result forwarding, and two auxiliary-register dependencies.

It separately shows an A-entry capture before the first call, use of that capture for the store at 0x00000041, and later references to the incoming-associated A and B symbols. This supports the distinction between recovered symbolic dataflow and unproven machine-level preservation.

Annotations identify overlapping global storage and unresolved final dispatch. They limit field-layout and continuation claims without establishing additional functionality.

No dependency implementations, ABI specification, caller evidence, memory map, execution traces, or runtime results were supplied. The contract therefore leaves those missing guarantees explicit rather than treating normal return as evidence of preserved registers, globals, or ownership.
