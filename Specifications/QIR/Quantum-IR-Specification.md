# Quantum Intermediate Representation Specification

Version 0.1

Copyright (c) Microsoft Corporation. All rights reserved.

## Introduction

This specification defines an intermediate representation for compiled
quantum computations.
The intent is that a quantum computation written in any language can be
compiled into this representation as a common intermediate _lingua franca_.
The intermediate representation would then be an input to a code generation
step that is specific to the target execution platform, whether simulator
or quantum hardware.

We see compilation as having three high-level phases:

1. A language-specific phase that takes code written in some quantum language,
   performs language-specific transformations and optimizations, and compiles
   the result into this intermediate format.
2. A generic phase that performs transformations and analysis of the code
   in the intermediate format.
3. A target-specific phase that performs additional transformations and
   ultimately generates the instructions required by the execution platform
   in a target-specific format.

By defining our representation within the popular open-source LLVM framework,
we enable users to easily write code analyzers and code transformers that
operate at this level, before the final target-specific code generation.

## Role of This Specification

The representation defined in this specification is intended to be the target
representation for language-specific compilers.
In particular, a language-specific compiler may rely on the existence of the
various utility and quantum functions defined in this specification.

It is neither required nor expected that any particular execution target actually
implement every runtime function specified here.
Rather, it is expected that the target-specific compiler will translate the
functions defined here into the appropriate representation for the target, whether
that be code, calls into target-specific libraries, metadata, or something else.

This applies to quantum functions as well as classical functions.
We do not intend to specify a gate set that all targets must support, nor even
that all targets use the gate model of computation.
Rather, the quantum functions in this document specify the interface that
language-specific compilers should meet.
It is the role of the target-specific compiler to translate the quantum functions
into an appropriate computation that meets the computing model and capabilities
of the target platform.

## Executable Code Generation Considerations

There are several areas where a code generator may want to significantly deviate
from a simple rewrite of basic intrinsics to target machine code:

- The intermediate representation assumes that the runtime does not perform
  garbage collection, and thus carefully tracks stack versus heap allocation
  and reference counting for heap-allocated structures. A runtime that provides
  full garbage collection may wish to remove the reference count field from several
  intermediate representation structures and elide calls to `quantum.rt.free`
  and the various `unreference` functions.
- Depending on the characteristics of the target architecture, the code generator
  may prefer to use different representations for the various types defined here.
  For instance, on some architectures it will make more sense to represent small
  types as bytes rather than as single or double bits.
- The primitive quantum operations provided by a particular target architecture
  may differ significantly from the intrinsics defined in this specification.
  It is expected that code generators may significantly rewrite sequences of
  quantum intrinsics into sequences that are optimal for the specific target.

## Identifiers

Identifiers in LLVM begin with a prefix, '@' for global symbols and '%' for
local symbols, followed by the identifier name.
Names must be in 8-bit ASCII, and must start with a letter or one of the special
characters '\$', '_', '-', and '.'; the rest of the characters in the name must
be either one of those characters or a digit.
It is possible to include other ASCII characters in names by surrounding the name
in quotes and using '\\xx' to represent the hex encoding of the character.
LLVM has no analog to namespaces or similar named scopes that are present in
many modern languages.

To the extent possible, symbols in the QIR should have identifiers that match
the identifier used in the source language.
The identifiers of local symbols should be converted to LLVM by merely adding the
'%' prefix.
Anonymous local variables generated by the compiler can be represented
as %0, %1, etc., as is usual in LLVM.

Similarly, global symbols should have their identifiers converted by adding the
'@' prefix.
If the source language provides a named scoping mechanism, such as Python modules
or Q# namespaces, then the fully-qualified name of the global symbol should be used.

:[DataTypes](./Data-Types.md)

## Metadata

### Representing Source-Language Attributes

Many languages allow attributes to be placed on callable and type definitions.
For instance, in Q# attributes are compile-time constant values of specific
user-defined types that themselves have the `Microsoft.Quantum.Code.Attribute`
attribute.

The language compiler should represent these attributes as LLVM metadata associated
with the callable or type.
For callables, the metadata representing the attribute should be attached
to the LLVM global symbol that defines the implementation table for the callable.
The identifier of the metadata node should be "!quantum.", where "!" is the LLVM
standard prefix for a metadata value, followed by the namespace-qualified name of
the attribute.
For example, a callable `Your.Op`with two attributes, `My.Attribute(6, "hello")`
and `Their.Attribute(2.1)`, applied to it would be represented in LLVM as follows:

```LLVM
@Your.Op = constant [4 x %CallableImpl*]
  [
    %CallableImpl* @Your.Op-body,
    %CallableImpl* @Your.Op-adj,
    %CallableImpl* @Your.Op-ctl,
    %CallableImpl* @Your.Op-ctladj
  ], !quantum.My.Attribute {i64 6, !"hello\00"},
     !quantum.Their.Attribute {double 2.1}
```

LLVM does not allow metadata to be associated with structure definitions,
so there is no direct way to represent attributes attached to user-defined types.
Thus, attributes on types are represented as named (module-level)
metadata, where the metadata node's name is "quantum." followed by the
namespace-qualified name of the type.
The metadata itself is the same as for callables, but wrapped in one more
level of LLVM structure in order to handle multiple attributes on the
same structure.
For example, a type `Your.Type`with two attributes, `My.Attribute(6, "hello")`
and `Their.Attribute(2.1)`, applied to it would be represented in LLVM as follows:

```LLVM
!quantum.Your.Type = !{ !{!"quantum.My.Attribute\00", i64 6, !"hello\00"},
                        !{ !"quantum.Their.Attribute\00", double 2.1} }
```

### Standard LLVM Metadata

#### Debugging Information

Compilers are strongly urged to follow the recommendations in
[Source Level Debugging with LLVM](http://llvm.org/docs/SourceLevelDebugging.html).

#### Branch Prediction

Compilers are strongly urged to follow the recommendations in
[LLVM Branch Weight Metadata](http://llvm.org/docs/BranchWeightMetadata.html).

### Other Compiler-Generated Metadata

> **TODO**: what quantum-specific "well-known" metadata do we want to specify?

## Quantum Instruction Set and Runtime

### Standard Operations

As recommended by the [LLVM documentation](https://llvm.org/docs/ExtendingLLVM.html),
we do not define new LLVM instructions for standard quantum operations.
Instead, we define a set of quantum functions that may be used by language-specific
compilers and that should be recognized and appropriately dealt with by target-specific
compilers:

| Function      | Signature                       | Description |
|---------------|---------------------------------|-------------|
| quantum.x     | `void(%Qubit)`                  | Pauli X gate. |
| quantum.y     | `void(%Qubit)`                  | Pauli Y gate. |
| quantum.z     | `void(%Qubit)`                  | Pauli Z gate. |
| quantum.h     | `void(%Qubit)`                  | Hadamard gate. |
| quantum.s     | `void(%Qubit)`                  | S (phase) gate. |
| quantum.t     | `void(%Qubit)`                  | T gate. |
| quantum.cnot  | `void(%Qubit, %Qubit)`          | CNOT gate. |
| quantum.rz    | `void(%Double, %Qubit)`         | Z rotation; the `double` is the rotation angle. |
| quantum.r1    | `void(%Double, %Qubit)`         | Rotation around the |1> state; the `double` is the rotation angle. |
| quantum.mz    | `%Result(%Qubit)`               | Measure in the Z basis. |
| quantum.ccnot | `void(%Qubit, %Qubit, %Qubit)`  | Toffoli gate. |
| quantum.swap  | `void(%Qubit, %Qubit)`          | SWAP gate. |
| quantum.r     | `void(%Pauli, %Double, %Qubit)` | General single-qubit rotation. |
| quantum.m     | `%Result(%Pauli, %Qubit)`       | General single-qubit measurement. |
| quantum.m2    | `%Result(%Pauli, %Qubit, %Pauli, %Qubit)` | General two-qubit joint measurement. |
| quantum.measure | `%Result(i64, [0 x %Pauli], [0 x %Qubit])` | General multi-qubit joint measurement. The `i64` is the count of entries in both the `%Pauli` and `%Qubit` arrays. |
| quantum.exp   | `void(i64, [0 x %Pauli], double, [0 x %Qubit])` | Applies the exponential of a multi-qubit Pauli operator. The `i64` is the count of entries in both the `%Pauli` and `%Qubit` arrays. The `double` multiplies the Pauli operator. |
| quantum.expfrac | `void(i64, [0 x %Pauli], i64, i64, [0 x %Qubit])` | Applies the exponential of a multi-qubit Pauli operator. The `i64` is the count of entries in both the `%Pauli` and `%Qubit` arrays. The Pauli operator is multiplied by the second `i64` divided by q raised to the power given by the third `i64`. |
| quantum.i     | `void(%Qubit)`                 | Applies the identity gate to a qubit. |

### Qubit Management Functions

We define the following functions for managing qubits:

| Function        | Signature                                     | Description |
|-----------------|-----------------------------------------------|-------------|
| quantum.alloc   | `void(i64, [0 x %Qubit]*)`                    | Fill in a pre-allocated array with newly allocated qubits. The `i64` is the number of qubits to allocate. |
| quantum.borrow  | `void(i64, [0 x %Qubit]*, i32, [0 x %Qubit])` | Borrow an array of qubits. The first two arguments are as for `quantum.alloc`. The remaining count and array passed in are the qubits currently in scope that are not safe to be borrowed. |
| quantum.release | `void(i64, [0 x %Qubit]*)`                    | Release the allocated qubits in the array. |
| quantum.return  | `void(i64, [0 x %Qubit]*)`                    | Return the borrowed qubits in the array. |

Borrowing qubits means supplying qubits that are guaranteed not to be otherwise
accessed while they are borrowed.
The code that borrows the qubits guarantees that the state of the qubits when
returned is identical, including entanglement, to their state when borrowed.
It is always acceptable to satisfy `borrow` by allocating new qubits.

The array of in-scope qubits passed to `borrow` is used by the runtime to safely
select qubits that will not be accessible during the scope of the borrowing.
The compiler is responsible for inserting the code required to compute this array.
We may change the signature of `borrow` to take an array of arrays instead,
as this may allow less runtime copying of qubits.
For in-scope callable values, it is necessary to determine the qubits, if any,
that appear in the callable's capture tuple.
See [Capture Tuples](#capture-tuples), below, for a description of how this is done.

Allocated qubits are not guaranteed to be in any particular state.
If a language guarantees that allocated qubits will be in a specific state, the compiler
should insert the code required to set the state of the qubits returned from `alloc`.

It will likely be useful to provide usage hints to `alloc` and `borrow`.
Since we don't know yet what form these hints may take, we leave them out for now.

## Classical Runtime

### Memory Management

The quantum runtime is not expected to provide garbage collection.
Rather, the compiler should generate code that generates proper allocation
for values on the stack or heap, and ensure that values are properly unreferenced
when they go out of scope.

#### Stack versus Heap Allocation

We assume that the source language does not provide any mechanism for mutable values
that persist across call values.
That is, this discussion assumes that the source language provides no feature
analogous to C `static` variables or to class static members as in C++, Java, or C#.

Any value that the compiler can prove will not be part of the return value can thus
always be allocated on the stack using the LLVM `alloca` intrinsic.
Values that might be part of the return value must be allocated on the heap; if such
a value is determined at run time to no longer be a possible part of the return value,
it may be explicitly released before the return.
Similarly, values that require too much memory space to put on the stack can be
allocated on the heap and explicitly released after their last use.

Values passed as arguments to a callable should not be released by the callable.

Values that are returned from a callable must not be allocated on the callee's stack.
The calling code can rely on this, and can apply the same logic as above to either pass
the value to the next caller or release the value after its last use.

We define the following functions for allocating and releasing heap memory,
They should provide the same behavior as the standard C library functions malloc and free.

| Function              | Signature   | Description |
|-----------------------|-------------|-------------|
| quantum.rt.heap_alloc | `i8*(i32)`  | Allocate a block of memory on the heap. |
| quantum.rt.heap_free  | `void(i8*)` | Release a block of allocated heap memory. |

#### Reference Counting

The reference count of a heap-allocated structure should be incremented whenever a new
long-lived reference to the structure is created and decremented whenever such a
reference is overwritten or goes out of scope.
A reference is long-lived if it is potentially part or all of the return value of the
current callable.

In some cases, it may be possible to avoid incrementing the reference count of a
structure when a new alias of the structure is created, as long as that alias is local
and not part of the return value.
For instance, in the following somewhat artificial Q# snippet:

```qsharp
function First<'T>(tuple: ('T1, 'T2)) : 'T1
{
  let x = tuple;
  let (first, second) = x;
  return first;
}
```

There is no need to increment the reference count for `tuple` when `x` is initialized,
nor to decrement it at the function exit when `x` goes out of scope.

## Example: Teleport

The following Q# code runs and tests a teleport operation.
The implementation is somewhat artificial in order to provide a richer example.

```qsharp
:[Teleport.qs](./Teleport.qs)
```

Here is the translation to LLVM using the representation from this document.
We assume that quantum instrinsics such as H and CNOT are inlined.
We are not using singleton-tuple equivalence, so that tuples of a single element
are explicitly allocated as a one-element LLVM structure.
We assume for now that arrays are represented as an LLVM structure containing a reference count,
an element count, and a zero-length (variable-length) array, and are passed by reference but are immutable.

Comments are added for explanation and would not be generated by the compiler.

**Note that this is hand-coded LLVM and is probably not valid.**

```LLVM
:[Teleport.ll](./Teleport.ll)
```