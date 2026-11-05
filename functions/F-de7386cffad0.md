<!-- Trusted provenance: F-de7386cffad0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-de7386cffad0

## Purpose

F-de7386cffad0 dispatches an event-like record through an object’s indirect-call table, with two conditional additions:

- Before dispatch, one event subtype triggers a call to F-dac1252a4c33 using a character-width value taken from the object.
- After a primary dispatch returns zero, a different event type can trigger a secondary dispatch when an event field matches an object field.

The function returns the first nonzero result produced by either dispatch that it actually invokes. Otherwise, it returns zero. The return value of F-dac1252a4c33 is not used.

“Object,” “event,” “subtype,” and “dispatch” are descriptive interpretations of the access pattern, not established application-level types. The numeric conditions and access ordering below define the observable contract.

## Interface

### Inputs

The function takes two arguments:

1. **Object reference.** A pointer to a word-addressed object containing an indirect-call-table address and additional fields.
2. **Event-record address.** An address supplied through an integer-typed argument and used to access record fields.

The supplied representation indicates four-byte integer fields and four-byte object-array strides. The layout below therefore describes the apparent 32-bit representation. A separate ABI definition is not supplied; pointer width, calling convention, and alignment requirements are not independently established.

### Object layout used by this function

All offsets are byte offsets from the object address.

| Offset | Access | Role established by use |
|---|---|---|
| `0x00` | Read as an address | Base of an indirect-call table |
| `0x50` | Read as an integer | Value compared with an event field during conditional secondary dispatch |
| `0xE4` | Read as an integer, then converted to character width | Second argument to F-dac1252a4c33 on its triggering path |

The indirect-call table is accessed at these byte offsets from its base:

| Table offset | Invocation condition |
|---|---|
| `0x10C` | Primary dispatch when the current event type is `0x16` and its current nested discriminator is `0x2A` |
| `0x110` | Primary dispatch in all other cases |
| `0xE0` | Secondary dispatch after a zero primary result, subject to an event-type and field-equality condition |

Each selected dispatch target receives the original object reference and original event-record address, in that order.

### Event layout used by this function

All offsets are byte offsets from the event-record address.

| Offset | Access | Role established by use |
|---|---|---|
| `0x04` | Read as an integer | Event-type discriminator |
| `0x08` | Read as an address when the type is `0x16` | Reference to a nested record |
| `0x08` | Read as an integer when the type is `0x0F` | Value compared with the object field at `0x50` |

For the nested record referenced by a type-`0x16` event:

| Offset | Access | Role established by use |
|---|---|---|
| `0x04` | Read as an integer | Nested discriminator used for helper invocation and primary dispatch selection |

The event field at `0x08` has different interpretations on different type-dependent paths. The function does not establish that it is always a valid address.

### Output

The result is an integer:

- A nonzero primary-dispatch result is returned unchanged.
- If primary dispatch returns zero and secondary dispatch is invoked, a nonzero secondary result is returned unchanged.
- All remaining completed paths return zero.

Both positive and negative nonzero dispatch results satisfy the early-return condition. No evidence establishes that nonzero means success, failure, or a particular application status.

## Behavior

### 1. Initial helper condition

The function first reads the event type at event offset `0x04`.

If that type is not `0x16`, it does not read the nested record for this initial condition and does not invoke F-dac1252a4c33.

If the type is `0x16`, it:

1. Reads the nested-record address from event offset `0x08`.
2. Reads the nested discriminator at nested offset `0x04`.
3. Subtracts `0x2500` from that discriminator in the represented integer computation.
4. Tests whether the result is `2`.

For ordinary representable arithmetic, the triggering discriminator is `0x2502`, decimal 9474. Under 32-bit modular subtraction, it is also the unique 32-bit triggering value. This normalization does not establish behavior for other values whose signed subtraction would overflow under a source-language interpretation; that issue remains documented under Uncertainties.

When the condition is satisfied, the function reads the integer at object offset `0xE4`, converts it to character width, and invokes F-dac1252a4c33 with:

- The original object reference.
- The converted character-width value.

The conversion is not a Boolean conversion or a test of whether the object field is nonzero. Values outside the character type’s range require the target’s character-conversion rules to determine the exact argument value.

The helper’s result is not inspected. Its side effects, if any, can affect all subsequent work.

### 2. Primary dispatch selection

After the initial helper stage, the function reads the event type at `0x04` again. This is a fresh read, not a reuse of the initial type value.

If this current type is `0x16`, the function freshly reads the nested-record address at event offset `0x08`, then the nested discriminator at nested offset `0x04`.

The primary target is selected as follows:

- Current type `0x16` and current nested discriminator `0x2A`: use table offset `0x10C`.
- Every other combination: use table offset `0x110`.

The function reads the current table address from object offset `0x00` and obtains the target from the selected table slot. It then calls that target with the original two arguments.

Consequently, F-dac1252a4c33 can change the primary-dispatch decision by modifying the event, its nested record, or the object’s table address before returning. The function does not preserve an earlier snapshot of those values.

If primary dispatch returns any nonzero integer, that value is returned immediately. No post-dispatch event-type read or secondary-dispatch condition is evaluated on that path.

### 3. Conditional secondary dispatch

Only after primary dispatch returns zero does the function read the event type at `0x04` again.

If the current type is not `0x0F`, the function returns zero without reading the type-`0x0F` comparison fields or invoking secondary dispatch.

If the current type is `0x0F`, it reads:

1. The event integer at offset `0x08`.
2. The object integer at offset `0x50`.

If those values differ, it returns zero.

If they are equal, it reads the current table address from object offset `0x00`, obtains the target at table offset `0xE0`, and calls it with the original object reference and event-record address.

A nonzero secondary result is returned unchanged. A zero secondary result leads to a zero function result.

The primary dispatch can therefore enable, disable, or redirect secondary dispatch by changing the event type, either comparison field, or the object’s table address before returning zero.

### 4. Call-count and ordering constraints

On every normally completed execution:

- F-dac1252a4c33 is invoked zero or one time.
- Exactly one primary target is invoked.
- The secondary target is invoked zero or one time.
- The helper, when invoked, precedes primary dispatch.
- Secondary dispatch, when invoked, follows a zero primary result.

There is no visible loop, retry, alternate primary target after a failure result, or repeated secondary invocation.

## Side effects

The function contains no explicit write to the object, event record, nested record, or indirect-call table. It does not visibly allocate, copy, free, retain, or release any storage.

Its observable effects include its ordered reads, the selected calls, and the effects of those calls. The supplied body does not constrain callees’ writes to shared state or their allocation and ownership behavior.

In particular:

- F-dac1252a4c33 must not be assumed to preserve the object or event state merely because the event address is not one of its explicit arguments.
- Either indirect target may modify the object, event, nested data, table, or global state.
- A zero result from primary dispatch does not imply that primary dispatch had no side effects.
- A nonzero result does not cause rollback of any earlier side effect.

No ownership transfer is explicitly expressed by this function. That observation does not establish the ownership contracts of its callees.

For normal completion, storage accessed after a callee returns must remain valid for those later accesses. For example, if primary dispatch returns zero, the event must still support the subsequent type read. If it returns nonzero, the function performs no further visible object or event reads before returning that result.

There are no visible null checks, bounds checks, target validation, or local fault-recovery paths. Invalid addresses or call targets are outside the established normal-completion contract; they are not converted into a specified error result.

## Dependencies

### F-dac1252a4c33

Directly invoked only when the initial type is `0x16` and the initial nested-discriminator arithmetic satisfies the helper condition.

Established interface at this call site:

- First argument: original object reference.
- Second argument: character-width conversion of the object integer at `0xE4`.
- Return value: unused.

Its implementation, ownership rules, global-state effects, and failure behavior are unknown.

### Indirect targets

The identities of the targets stored at table offsets `0x10C`, `0x110`, and `0xE0` are not supplied. No opaque function IDs can be assigned to them from the available evidence.

Established interface at each call site:

- Arguments: original object reference, then original event-record address.
- Result: integer tested for zero and otherwise returned unchanged.

The slots are distinct selection locations, but the evidence does not require their stored targets to be distinct functions.

## Acceptance examples

These examples are conditional expectations from static evidence, not reports of runtime tests. Unless stated otherwise, they assume valid storage, callable targets, normal returns, and no callee mutation of fields relevant to later decisions.

1. **Helper condition followed by ordinary primary dispatch.**  
   Initial type is `0x16`, nested discriminator is `0x2502`, and the object field at `0xE4` contains `7`. F-dac1252a4c33 is called with the object and character value `7`. Because the unchanged discriminator is not `0x2A`, primary dispatch uses slot `0x110`. If that call returns `11`, the function returns `11` without secondary evaluation.

2. **Special primary selection without helper invocation.**  
   Type is `0x16` and nested discriminator is `0x2A`. The helper is not invoked. Primary dispatch uses slot `0x10C`. If it returns zero and the type remains `0x16`, the function returns zero without secondary dispatch.

3. **Nested storage is unnecessary on a non-nested path.**  
   Type is neither `0x16` nor `0x0F` and remains unchanged. The function does not interpret event offset `0x08` as a nested-record address. Primary dispatch uses slot `0x110`; a zero result is followed by a zero function result.

4. **Matching secondary condition.**  
   Type is `0x0F`, event offset `0x08` and object offset `0x50` both contain the same integer, and primary dispatch returns zero. Primary dispatch uses slot `0x110`, followed by secondary dispatch through slot `0xE0`. If secondary dispatch returns `-3`, the function returns `-3`.

5. **Mismatching secondary condition.**  
   Type is `0x0F`, primary dispatch returns zero, and the two comparison fields differ. Secondary dispatch is not invoked, and the function returns zero.

6. **Nonzero primary result suppresses secondary work.**  
   Even if the event would satisfy the secondary condition, a primary result of `-1` is returned immediately. The secondary condition’s type and comparison fields are not read afterward.

7. **Helper mutation affects primary selection.**  
   The initial helper condition is satisfied. If F-dac1252a4c33 changes the nested discriminator to `0x2A` while leaving a valid type-`0x16` record, the fresh reads select slot `0x10C`, not slot `0x110`.

8. **Primary mutation affects secondary eligibility.**  
   Primary dispatch changes the event to type `0x0F`, establishes matching comparison fields, and returns zero. Secondary dispatch occurs even if the event had a different type when primary dispatch began.

9. **Secondary zero is final.**  
   If secondary dispatch is invoked and returns zero, the function returns zero. It does not attempt another target.

## Uncertainties

The following are **documented_only**, not inferred guarantees:

- Exact ABI, pointer representation, alignment rules, and integer signedness beyond the apparent four-byte layout.
- Plain-character width, signedness, and out-of-range conversion behavior. An eight-bit low-byte interpretation is plausible for the apparent environment but is not independently established.
- Overflow semantics for the helper-condition subtraction outside ordinary representable arithmetic.
- Application meanings of the event types, nested discriminators, object fields, and dispatch results.
- Identities and contracts of the three indirect targets.
- Callee preservation of globals, memory validity, ownership, or table contents.
- Concurrency and synchronization behavior. Fresh reads are established in the supplied representation, but no atomicity or coherent cross-field snapshot is established.
- Behavior following invalid memory accesses, invalid targets, or callees that fail to return normally.

## Evidence

Established static evidence consists of the supplied function body: conditional field reads, a character conversion, one named dependency call, three possible indirect-call slots, repeated reads across call boundaries, and zero-versus-nonzero return handling.

The four-byte object-field offsets follow the supplied integer-array indexing and its apparent 32-bit layout. Interpreting the structures as an object, event record, nested discriminator, and dispatch table is inferred intent; those labels add no validation or ownership guarantees.

No runtime tests have been performed. Acceptance examples state conditional consequences of the static control flow, not observed execution results.
