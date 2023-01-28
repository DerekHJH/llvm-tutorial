Now that we’re familiar with our language and the AST, let’s see how MLIR can help to compile Toy.

# Introduction: Multi-Level Intermediate Representation

Other compilers, like LLVM, offer a fixed set of predefined types and (usually low-level / RISC-like) instructions. It is up to the frontend for a given language to perform any language-specific type-checking, analysis, or transformation before emitting LLVM IR. For example, Clang will use its AST to perform not only static analysis but also transformations, such as C++ template instantiation through AST cloning and rewrite. Finally, other languages with construction at a higher-level than C/C++ may require non-trivial lowering from their AST to generate LLVM IR.

As a consequence, multiple frontends end up reimplementing significant pieces of infrastructure to support the need for these analyses and transformation. MLIR addresses this issue by being designed for extensibility. As such, there are few pre-defined instructions (operations in MLIR terminology) or types.

# Interfacing with MLIR

MLIR is designed to be a completely extensible infrastructure; there is no closed set of attributes (think: constant metadata), operations, or types. MLIR supports this extensibility with the concept of Dialects. Dialects provide a grouping mechanism for abstraction under a unique namespace.

In MLIR, Operations are the core unit of abstraction and computation, similar in many ways to LLVM instructions. Operations can have application-specific semantics and can be used to represent all of the core IR structures in LLVM: instructions, globals (like functions), modules, etc.

Here is the MLIR assembly for the Toy transpose operations:

```
%t_tensor = "toy.transpose"(%tensor) {inplace = true} : (tensor<2x3xf64>) -> tensor<3x2xf64> loc("example/file/path":12:1)
```

- %t_tensor: The name given to the result defined by this operation (which includes a prefixed sigil to avoid collisions). An operation may define zero or more results (in the context of Toy, we will limit ourselves to single-result operations), which are SSA values. The name is used during parsing but is not persistent (e.g., it is not tracked in the in-memory representation of the SSA value).

- "toy.transpose": The name of the operation. It is expected to be a unique string, with the namespace of the dialect prefixed before the “.”. This can be read as the transpose operation in the toy dialect.

- (%tensor): A list of zero or more input operands (or arguments), which are SSA values defined by other operations or referring to block arguments.

- { inplace = true }: A dictionary of zero or more attributes, which are special operands that are always constant. Here we define a boolean attribute named ‘inplace’ that has a constant value of true.

- (tensor<2x3xf64>) -> tensor<3x2xf64>: This refers to the type of the operation in a functional form, spelling the types of the arguments in parentheses and the type of the return values afterward.

- loc("example/file/path":12:1): This is the location in the source code from which this operation originated.

Shown here is the general form of an operation. As described above, the set of operations in MLIR is extensible. Operations are modeled using a small set of concepts, enabling operations to be reasoned about and manipulated generically. These concepts are:

- A name for the operation.
- A list of SSA operand values.
- A list of attributes.
- A list of types for result values.
- A source location for debugging purposes.
- A list of successors blocks (for branches, mostly).
- A list of regions (for structural operations like functions).

In MLIR, every operation has a mandatory source location associated with it. Contrary to LLVM, where debug info locations are metadata and can be dropped, in MLIR, the location is a core requirement, and APIs depend on and manipulate it. Dropping a location is thus an explicit choice which cannot happen by mistake.

To provide an illustration: If a transformation replaces an operation by another, that new operation must still have a location attached. This makes it possible to track where that operation came from.

It’s worth noting that the mlir-opt tool - a tool for testing compiler passes - does not include locations in the output by default. The -mlir-print-debuginfo flag specifies to include locations. (Run mlir-opt --help for more options.)

# Opaque API

MLIR is designed to allow all IR elements, such as attributes, operations, and types, to be customized. At the same time, IR elements can always be reduced to the above fundamental concepts. This allows MLIR to parse, represent, and round-trip IR for any operation. For example, we could place our Toy operation from above into an .mlir file and round-trip through mlir-opt without registering any toy related dialect:

```
func.func @toy_func(%tensor: tensor<2x3xf64>) -> tensor<3x2xf64> {
  %t_tensor = "toy.transpose"(%tensor) { inplace = true } : (tensor<2x3xf64>) -> tensor<3x2xf64>
  return %t_tensor : tensor<3x2xf64>
}
```

In the cases of unregistered attributes, operations, and types, MLIR will enforce some structural constraints (e.g. dominance, etc.), but otherwise they are completely opaque. For instance, MLIR has little information about whether an unregistered operation can operate on particular data types, how many operands it can take, or how many results it produces. This flexibility can be useful for bootstrapping purposes, but it is generally advised against in mature systems. Unregistered operations must be treated conservatively by transformations and analyses, and they are much harder to construct and manipulate.

This handling can be observed by crafting what should be an invalid IR for Toy and seeing it round-trip without tripping the verifier:

```
func.func @main() {
  %0 = "toy.print"() : () -> tensor<2x3xf64>
}
```

There are multiple problems here: the toy.print operation is not a terminator; it should take an operand; and it shouldn’t return any values. In the next section, we will register our dialect and operations with MLIR, plug into the verifier, and add nicer APIs to manipulate our operations.

# Defining a Toy Dialect

To effectively interface with MLIR, we will define a new Toy dialect. This dialect will model the structure of the Toy language, as well as provide an easy avenue for high-level analysis and transformation.

```C++
/// This is the definition of the Toy dialect. A dialect inherits from
/// mlir::Dialect and registers custom attributes, operations, and types. It can
/// also override virtual methods to change some general behavior, which will be
/// demonstrated in later chapters of the tutorial.
class ToyDialect : public mlir::Dialect {
public:
  explicit ToyDialect(mlir::MLIRContext *ctx);

  /// Provide a utility accessor to the dialect namespace.
  static llvm::StringRef getDialectNamespace() { return "toy"; }

  /// An initializer called from the constructor of ToyDialect that is used to
  /// register attributes, operations, types, and more within the Toy dialect.
  void initialize();
};
```

This is the C++ definition of a dialect, but MLIR also supports defining dialects declaratively via tablegen. Using the declarative specification is much cleaner as it removes the need for a large portion of the boilerplate when defining a new dialect. It also enables easy generation of dialect documentation, which can be described directly alongside the dialect. In this declarative format, the toy dialect would be specified as:

```C++
// Provide a definition of the 'toy' dialect in the ODS framework so that we
// can define our operations.
def Toy_Dialect : Dialect {
  // The namespace of our dialect, this corresponds 1-1 with the string we
  // provided in `ToyDialect::getDialectNamespace`.
  let name = "toy";

  // A short one-line summary of our dialect.
  let summary = "A high-level dialect for analyzing and optimizing the "
                "Toy language";

  // A much longer description of our dialect.
  let description = [{
    The Toy language is a tensor-based language that allows you to define
    functions, perform some math computation, and print results. This dialect
    provides a representation of the language that is amenable to analysis and
    optimization.
  }];

  // The C++ namespace that the dialect class definition resides in.
  let cppNamespace = "toy";
}
```

To see what this generates, we can run the mlir-tblgen command with the gen-dialect-decls action like so:

```bash
${build_root}/bin/mlir-tblgen -gen-dialect-decls 
${mlir_src_root}/examples/toy/Ch2/include/toy/Ops.td -I 
${mlir_src_root}/include/
```

After the dialect has been defined, it can now be loaded into an MLIRContext:

```C++
context.loadDialect<ToyDialect>();
```

By default, an MLIRContext only loads the Builtin Dialect, which provides a few core IR components, meaning that other dialects, such as our Toy dialect, must be explicitly loaded.

# Defining Toy Operations 

Now that we have a Toy dialect, we can start defining the operations. This will allow for providing semantic information that the rest of the system can hook into. As an example, let’s walk through the creation of a toy.constant operation. This operation will represent a constant value in the Toy language.

Now that we have a Toy dialect, we can start defining the operations. This will allow for providing semantic information that the rest of the system can hook into. As an example, let’s walk through the creation of a toy.constant operation. This operation will represent a constant value in the Toy language.

```
%4 = "toy.constant"() {value = dense<1.0> : tensor<2x3xf64>} : () -> tensor<2x3xf64>
```

This operation takes zero operands, a dense elements attribute named value to represent the constant value, and returns a single result of RankedTensorType. An operation class inherits from the CRTP mlir::Op class which also takes some optional traits to customize its behavior. Traits are a mechanism with which we can inject additional behavior into an Operation, such as additional accessors, verification, and more. Let’s look below at a possible definition for the constant operation that we have described above:

```C++
class ConstantOp : public mlir::Op<
                     /// `mlir::Op` is a CRTP class, meaning that we provide the
                     /// derived class as a template parameter.
                     ConstantOp,
                     /// The ConstantOp takes zero input operands.
                     mlir::OpTrait::ZeroOperands,
                     /// The ConstantOp returns a single result.
                     mlir::OpTrait::OneResult,
                     /// We also provide a utility `getType` accessor that
                     /// returns the TensorType of the single result.
                     mlir::OpTraits::OneTypedResult<TensorType>::Impl> {

 public:
  /// Inherit the constructors from the base Op class.
  using Op::Op;

  /// Provide the unique name for this operation. MLIR will use this to register
  /// the operation and uniquely identify it throughout the system. The name
  /// provided here must be prefixed by the parent dialect namespace followed
  /// by a `.`.
  static llvm::StringRef getOperationName() { return "toy.constant"; }

  /// Return the value of the constant by fetching it from the attribute.
  mlir::DenseElementsAttr getValue();

  /// Operations may provide additional verification beyond what the attached
  /// traits provide.  Here we will ensure that the specific invariants of the
  /// constant operation are upheld, for example the result type must be
  /// of TensorType and matches the type of the constant `value`.
  LogicalResult verifyInvariants();

  /// Provide an interface to build this operation from a set of input values.
  /// This interface is used by the `builder` classes to allow for easily
  /// generating instances of this operation:
  ///   mlir::OpBuilder::create<ConstantOp>(...)
  /// This method populates the given `state` that MLIR uses to create
  /// operations. This state is a collection of all of the discrete elements
  /// that an operation may contain.
  /// Build a constant with the given return type and `value` attribute.
  static void build(mlir::OpBuilder &builder, mlir::OperationState &state,
                    mlir::Type result, mlir::DenseElementsAttr value);
  /// Build a constant and reuse the type from the given 'value'.
  static void build(mlir::OpBuilder &builder, mlir::OperationState &state,
                    mlir::DenseElementsAttr value);
  /// Build a constant by broadcasting the given 'value'.
  static void build(mlir::OpBuilder &builder, mlir::OperationState &state,
                    double value);
};
```

and we can register this operation in the ToyDialect initializer:

```
void ToyDialect::initialize() {
  addOperations<ConstantOp>();
}
```