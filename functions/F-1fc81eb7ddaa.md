<!-- Trusted provenance: F-1fc81eb7ddaa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1fc81eb7ddaa

## Purpose

The established behavior is a sequence of seven dependency calls and width-specific memory accesses. Addresses are derived from an unsigned 32-bit input, bytes read from memory, and implicit execution-context values. The recovered path ends at an invalid-instruction boundary rather than a demonstrated normal return.

No application-level purpose is established. The evidence does not support classifying this operation as allocation, initialization, container manipulation, or ownership transfer.

This contract describes the supplied static representation, conditional on execution reaching each operation. It does not establish that the recovered function boundary is correct, that the entire path is reachable, or that the apparent operations are intentional.

## Interface

### Explicit inputs

| Contract name | Width and interpretation | Visible use |
|---|---|---|
| A | 32-bit value | Argument to the first dependency. |
| B | 32-bit value | No visible use. |
| X | Unsigned 32-bit integer | Address arithmetic and logical right shifts. |

B’s unused status applies only to this recovered path; it does not establish permission to change an external calling convention.

### Implicit context

The representation also depends on:

- An incoming return-context value used to form the first indirect call target.
- A recovered 32-bit signed context value, C.
- Four stack-associated locations, described below.
- An additional byte-sized register result associated with the final dependency call.

The stack offsets identify recovered context locations, not a verified public ABI. Their reference base and actual machine-level access mechanism remain unresolved.

| Location offset | Represented value | Observation point |
|---|---|---|
| `0x168` | H: 32-bit value | Argument preparation for F-7d50b94821c2. |
| `0x210` | J: signed 32-bit address contribution | The store at J + `0x20`. |
| `0x248` | Signed 32-bit address contribution | Referenced separately to capture an address value, calculate a byte-store address, and calculate a later pointer-load address. |
| `0x374` | Unsigned 32-bit value | Referenced before F-7d50b94821c2 and again after that call. |

These values must not be treated collectively as immutable entry snapshots. The representation does not establish that dependencies or intervening writes preserve their storage.

For the repeated location at `0x374`, this contract distinguishes:

- L-before: the value referenced for the store preceding F-7d50b94821c2.
- L-after: the value referenced for the byte read following that call.

For the location at `0x248`, it distinguishes:

- K-capture: the value used to compute a retained address D.
- K-byte: the subsequent value used to address a byte store.
- K-pointer: the later value used to address a pointer read.

These labels distinguish observations; they do not assert that the values differ. Whether each recovered reference corresponds to a separate physical stack load is documented_only.

### Derived values

| Name | Definition | Range |
|---|---|---|
| Q | X logically shifted right by 2 | `0…0x3fffffff` |
| R | Q logically shifted right by 2 | `0…0x0fffffff` |
| N | Zero-extended byte read at Q + `0x1f` | `0…255` |
| T | N logically shifted right by 2 | `0…63` |
| S | N multiplied by 64 | `0…16320` |
| V | N multiplied by 256 | `0…65280` |

Unsigned address expressions use 32-bit arithmetic. In particular, multiplying X by four can wrap. All products derived solely from N fit within 32 bits. The visible masks on N do not change its value, and S equals V logically shifted right by 2.

A 32-bit store occupies four bytes; a 16-bit store occupies two; a byte store occupies one. A 16-bit store retains only the low 16 bits. Byte order is not established.

The representation uses signed arithmetic for some context-derived addresses, including additions involving C, J, and the K observations. Overflow behavior, and the interpretation of doubling C through a signed left shift, are not sufficiently established to assign portable arithmetic semantics.

### Outputs and termination

No ordinary return value is established.

After the final byte store, the recovered path reaches an invalid-instruction marker. Whether actual execution traps, is intercepted, continues through unrecovered behavior, or terminates differently is unknown. Normal return is not an acceptance requirement.

## Behavior

### Preconditions and execution limits

The complete sequence is conditional on:

- Each reached call target being executable under the actual ABI.
- Each dependency returning.
- Each memory access being permitted for its address, width, and alignment.
- The relevant arithmetic having a valid interpretation in the actual execution environment.

These are conditions for observing the sequence, not checks performed by the function. No validation, error-return path, or rollback is visible.

The order below is the recovered operation order. It does not establish atomicity, inter-thread synchronization, device-memory ordering, or the internal evaluation order of expressions where that order is unspecified.

### Initial calls and X-derived writes

| Order | Operation |
|---:|---|
| 1 | Invoke F-000000000001 with A. Capture its 32-bit result. |
| 2 | Store that result as 32 bits at `4X + 0x14`. |
| 3 | Read one byte at Q + `0x1f`; zero-extend and retain it as N. |
| 4 | Invoke F-000000000002 with N and Q. |
| 5 | Store R as 32 bits at Q + `0x34`. |
| 6 | Store R as 32 bits at Q + `0x18`. |
| 7 | Store R as 32 bits at R + `0x14`. |
| 8 | Store the low 16 bits of R at `4X`. |
| 9 | Invoke F-000000000003 with `4N` as both arguments. |
| 10 | Store R as 32 bits at Q + `0x18` again. |
| 11 | Store R as 32 bits at T + `0x24`. |
| 12 | Store R as 32 bits at R + `0x14` again. |
| 13 | Store S as 32 bits at T + `0x18`. |
| 14 | Store S as 16 bits at S + `4`. |
| 15 | Store V as 32 bits at N + `0x18`. |
| 16 | Store the low 16 bits of R at Q + `4`. |
| 17 | Store R as 32 bits at S + `0x24`. |
| 18 | Repeat that same 32-bit store, with no intervening visible operation. |

N is captured after the first dependency and its result store. Consequently, their effects can influence N. Subsequent uses of N employ the captured byte value, not another read at Q + `0x1f`.

The 16-bit store of S loses no value bits because S is at most 16320.

### Further calls and context-derived writes

| Order | Operation |
|---:|---|
| 19 | Invoke F-000000000004 with V and S. Capture its 32-bit result. |
| 20 | Store that result as 32 bits at `4V + 0x14`. |
| 21 | Store the low 16 bits of R at R + `2`. |
| 22 | Store R as 32 bits at S + `0x24`. |
| 23 | Store S as 16 bits at S + `4`. |
| 24 | Invoke F-000000000005 with V and S. |
| 25 | Store the low 16 bits of R at V + `6`. |
| 26 | Perform two distinct 32-bit reads at C + `0x3c`: one supplies a destination base, and the other supplies a stored value. |
| 27 | Store the value from the value-supplying read as 32 bits at the destination-supplying value plus `0x18`. |
| 28 | Store S as 32 bits at `4V + 0x3c`. |
| 29 | Reference J from its stack-associated location and store S as 32 bits at J + `0x20`. |
| 30 | Store S as 32 bits at S + `0x14`. |
| 31 | Reference L-before and store L-before logically shifted right by 2 as 32 bits at S + `0x24`. |
| 32 | Invoke F-7d50b94821c2 with V logically shifted right by 4 and H, with H referenced for this invocation. |

The two reads at C + `0x3c` are not a single captured pointer read. Their relative evaluation order is unresolved. The destination base and stored value are therefore not guaranteed to be identical observations. Both reads supply the subsequent store; the representation shows no intervening explicit store between them.

J, L-before, and H are observed at their respective uses after the preceding operations. Earlier calls or aliased writes cannot be assumed to leave their associated storage unchanged.

### Late observations, retained addresses, and termination

| Order | Operation |
|---:|---|
| 33 | Reference L-after, then read one byte at `(L-after logically shifted right by 2) + 0x18`. Zero-extend and retain that byte as M. |
| 34 | Reference K-capture and retain D, the address value K-capture + `0x31`. |
| 35 | Compute and retain E from C shifted left by one. The signed-shift interpretation remains subject to the arithmetic uncertainty above. |
| 36 | Reference K-byte and store byte value 100 at K-byte + `0x45`. |
| 37 | Reference K-pointer and read a 32-bit address at K-pointer + `0xa5`; retain that address as W. |
| 38 | Store D as 32 bits at M + `0x38`. |
| 39 | Store 100 as 32 bits at W. |
| 40 | Store D as 32 bits at W + `4`. |
| 41 | Store D as 32 bits at W + `8`. |
| 42 | Store W itself as 32 bits at W + `0x0c`. |
| 43 | Invoke F-000000000006 with 100, D, and S + `0x25`. |
| 44 | Store the additional register-supplied result byte associated with that call at E + `0x1e`. |
| 45 | Reach the invalid-instruction boundary. |

L-after is referenced after F-7d50b94821c2 returns. Nothing establishes equality with L-before.

D is retained before the byte store and pointer read. Later references to the K-associated location do not redefine D. Conversely, capturing D does not establish that K-byte or K-pointer equals K-capture. In particular, the byte store may affect storage used by the subsequent K reference if aliasing permits it.

M, D, E, and W are retained for their subsequent listed uses. The final destination uses E computed before the byte store and final dependency, not a newly evaluated doubling of C after that dependency.

### Aliasing and access preservation

No independent-object or non-aliasing assumption is established.

An earlier store may affect a later read, including a stack-associated reference. A 16-bit write may partially overwrite a previous 32-bit write. The initial result store may overlap the byte later captured as N. Writes through W may overlap earlier destinations or stack-associated storage.

The repeated stores remain separate accesses. Replacing the sequence with an unordered set of final values would not preserve fault-sensitive, instrumented, volatile, or device-backed behavior.

## Side effects

### Memory

The body performs byte, 16-bit, and 32-bit stores at computed addresses. Several destinations can be numerically small. In particular, M + `0x38` ranges from `0x38` through `0x137`.

M is a zero-extended byte, not an established pointer-sized value. Its use in address arithmetic does not justify replacing the byte read with a wider load.

The writes through W occupy four successive four-byte positions and supply, in order:

1. 100.
2. D.
3. D.
4. W.

This is a structural observation, not an established record type or ownership protocol.

### Dependency effects and failure

Dependency effects are unavailable. No dependency is assumed to preserve globals or stack-associated storage, leave pointed-to memory unchanged, allocate or release memory, return normally, or avoid callbacks, I/O, exceptions, and synchronization.

There is no visible rollback. A fault, nonreturning call, or other interruption may leave earlier effects in place. No error code or exception-handling contract is established.

### Ownership

No allocation, deallocation, reference-count update, ownership transfer, or lifetime guarantee is explicitly established. Storing W through W does not establish self-ownership.

## Dependencies

F-7d50b94821c2 is the sole supplied identity for a directly identified dependency. Its visible arguments are V logically shifted right by 4 and the H observation used for that call. No result is visibly consumed, and its semantic contract is unavailable.

The remaining IDs are specification-local opaque call-site labels. Distinct labels do not prove distinct ultimate implementations.

| ID | Target evidence | Visible contract |
|---|---|---|
| F-000000000001 | Incoming return context plus `0x1b0`. | A; a 32-bit result is stored. |
| F-000000000002 | Fixed code-relative target construction. | N and Q; no result visibly consumed. |
| F-000000000003 | Fixed code-relative target construction. | `4N` twice; no result visibly consumed. |
| F-000000000004 | Fixed code-relative target construction. | V and S; a 32-bit result is stored. |
| F-000000000005 | Fixed code-relative target construction. | V and S; no result visibly consumed. |
| F-000000000006 | Fixed code-relative target construction. | 100, D, and S + `0x25`; an additional register-associated byte is subsequently stored. |

The fixed target constructions do not show dispatch-table reads. Complete calling conventions and register-preservation obligations are not established.

## Acceptance examples

These are conditional static criteria, not executed tests. Each assumes that execution reaches the relevant operations and that arithmetic and memory accesses are valid.

### Unsigned wraparound and truncation

For X equal to `0xffffffff`:

- Q is `0x3fffffff`.
- R is `0x0fffffff`.
- The first result-store address is `0x10` after 32-bit wraparound.
- The 16-bit store at `4X` targets `0xfffffffc` and supplies `0xffff`.

Using an unwrapped wider address for the first store would not match the established unsigned arithmetic.

### Maximum captured byte

If N is 255:

- T is 63, S is 16320, and V is 65280.
- F-000000000003 receives 1020 twice.
- F-000000000004 receives 65280 and 16320.
- F-7d50b94821c2 receives 4080 as its first argument.
- The store at N + `0x18` targets `0x117` and supplies 65280.

Signed eight-bit interpretation of N would not match the evidence.

### Zero captured byte

If N is zero, T, S, and V are zero. Later operations access low addresses, including `0x14`, `0x18`, and `0x24`.

No zero-value special case is visible. Inaccessibility of these addresses may prevent later execution; silently skipping them is not established behavior.

### Separate context reads

Suppose the destination-supplying read at C + `0x3c` observes `0x1000`, while the value-supplying read observes `0x2000`. The resulting store supplies the 32-bit value `0x2000` at `0x1018`, assuming valid arithmetic and access.

The contract does not specify which observation occurs first. A mandatory single-read self-address store would incorrectly strengthen the evidence.

### Stack-associated value changed across a call

Suppose L-before is `0x100`, and the location associated with L yields `0x200` after F-7d50b94821c2 returns.

The earlier store at S + `0x24` supplies `0x40`. The subsequent byte read targets `0x98`, derived from L-after, rather than `0x58`, derived from L-before.

If that byte is 7, the later D store targets `0x3f`. Neither equality of the L observations nor preservation of the byte is assumed.

### Captured address versus later K observations

Suppose K-capture is `0x1000`; D is then `0x1031`. If K-byte is `0x2000`, the byte store targets `0x2045`. If K-pointer subsequently is `0x3000`, the pointer read targets `0x30a5`.

The later stores and final call still use retained D, `0x1031`. These differing observations are conditional possibilities, not a claim that a particular mechanism causes them.

### Repetition and final state

If execution reaches both consecutive stores at S + `0x24`, both accesses occur even though their supplied values are equal.

After the final dependency returns, its additional result byte is stored at retained E + `0x1e`. The next recovered event is the invalid-instruction boundary, not a demonstrated return.

## Uncertainties

The following remain documented_only:

- Application purpose, data-structure meanings, and valid object boundaries.
- Correctness of the recovered function boundary, reachability, and complete control flow.
- Architecture, endianness, alignment requirements, and fault delivery.
- Complete ABI and actual implementation of stack-associated references.
- Preservation or mutation of stack-associated locations across calls and aliased writes.
- Relative order of the two reads at C + `0x3c`.
- Signed address overflow and signed left-shift behavior.
- Indirect dependency identities and all unavailable dependency semantics.
- Origin, meaning, and availability of the final additional register result.
- Concurrency, atomicity, synchronization, and device-memory requirements.
- Validity of numerically small addresses.
- Runtime meaning of the terminal invalid-instruction marker.

These uncertainties do not justify replacing the visible accesses with a more conventional operation.

## Evidence

**Established static evidence:** the supplied representation contains three explicit 32-bit inputs, six indirect calls, one call to F-7d50b94821c2, width-specific memory operations, byte zero-extension, two separate reads at C + `0x3c`, repeated references to stack-associated values, and a terminal invalid-instruction marker.

**Derived arithmetic facts:** N lies in `0…255`; its visible masks preserve its value; S equals V logically shifted right by 2; byte-derived products fit within 32 bits; R can lose bits in 16-bit stores.

**Inferred intent:** no application-level interpretation is adopted. Record-like storage and self-address writes describe structure only.

**Verification status:** static analysis of the supplied representation only. No runtime tests, independent implementation inspection, or behavioral validation have been performed.
