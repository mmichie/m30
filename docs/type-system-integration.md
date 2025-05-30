# Type System Integration with Parser

## Overview: Two-Phase Approach

The type system doesn't run during parsing - it operates on the AST afterward:

```
Source Code → Lexer → Parser → AST → Type Checker → Typed AST → Code Generation
                                 ↑
                          Type annotations
                          captured but not
                          validated yet
```

## 1. Parser's Role in Type System

The parser only:
1. **Recognizes type syntax** and stores it in the AST
2. **Doesn't validate** types exist or make sense
3. **Preserves all information** for the type checker

```go
// AST nodes carry optional type annotations
type FunctionDef struct {
    Name       string
    TypeParams []string          // Generic parameters [T, U]
    Params     []Param
    ReturnType *TypeExpr         // Optional return type
    Body       Expr
}

type Param struct {
    Pattern Pattern
    Type    *TypeExpr  // Optional type annotation
}

type LetBinding struct {
    Pattern Pattern
    Type    *TypeExpr  // Optional: let x: int = 5
    Value   Expr
}
```

## 2. Type Expression AST

```go
// Type expressions are parsed into their own AST
type TypeExpr interface {
    typeExpr()
}

type NamedType struct {
    Name string
    Args []TypeExpr  // For generics: List[T]
}

type UnionType struct {
    Types []TypeExpr  // int | string
}

type FunctionType struct {
    Params []TypeExpr
    Return TypeExpr   // (int, string) -> bool
}

type RecordType struct {
    Fields map[string]TypeExpr  // {x: int, y: int}
}
```

## 3. Parser Examples

```m30
# Parser sees this:
let add (x: int) (y: int) -> int = x + y

# Creates AST:
FunctionDef {
    Name: "add",
    Params: [
        Param{Pattern: IdentPattern("x"), Type: NamedType("int")},
        Param{Pattern: IdentPattern("y"), Type: NamedType("int")}
    ],
    ReturnType: NamedType("int"),
    Body: BinaryOp("+", Ident("x"), Ident("y"))
}
```

## 4. Type Checking Phase

After parsing, a separate type checker:

```go
type TypeChecker struct {
    env         *TypeEnvironment
    constraints []Constraint
}

func (tc *TypeChecker) CheckProgram(ast *Program) (*TypedProgram, error) {
    // 1. Build initial type environment from declarations
    tc.collectDeclarations(ast)
    
    // 2. Infer types for unannotated expressions
    for _, def := range ast.Definitions {
        tc.inferDefinition(def)
    }
    
    // 3. Solve constraints from inference
    solution := tc.solveConstraints()
    
    // 4. Apply solution and check consistency
    return tc.applyAndCheck(ast, solution)
}
```

## 5. Type Inference Integration

```m30
# Parser sees:
let map f list =
    match list with
    | [] -> []
    | x :: xs -> f(x) :: map(f, xs)

# Type checker infers:
# map : [T, U] (T -> U, List[T]) -> List[U]
```

The inference process:
1. Assign fresh type variables where types are missing
2. Generate constraints from usage
3. Solve constraints (unification)
4. Check solution is valid

## 6. Islands of Inference Implementation

```go
// Module boundaries require type annotations
type ModuleExport struct {
    Name     string
    Type     TypeExpr  // REQUIRED - parser enforces this
    Internal bool
}

// Inside modules, inference is full
func (tc *TypeChecker) checkModuleBody(module *Module) error {
    // Create isolated type environment for this module
    moduleEnv := NewTypeEnvironment(tc.globalEnv)
    
    // Inside module: full HM inference
    for _, def := range module.PrivateDefinitions {
        inferredType := tc.inferWithHM(def, moduleEnv)
        moduleEnv.bind(def.Name, inferredType)
    }
    
    // Exported functions must match declared types
    for _, export := range module.Exports {
        inferredType := moduleEnv.lookup(export.Name)
        declaredType := tc.evaluateTypeExpr(export.Type)
        
        if !tc.unify(inferredType, declaredType) {
            return fmt.Errorf("export %s: inferred type %v doesn't match declared %v",
                export.Name, inferredType, declaredType)
        }
    }
}
```

## 7. Error Messages with Type Info

Since parsing and type checking are separate, we can give better errors:

```m30
# Parse error (syntax):
let x: = 5
       ^ Error: Expected type after ':'

# Type error (semantic):
let x: string = 5
                ^ Error: Cannot assign int to string

# Inference error:
let f x = x + "hello"
let y = f(5)
        ^ Error: Cannot unify int with string
          Inferring f : T -> U where T supports + with string
          But f(5) requires T = int
          And int doesn't support + with string
```

## 8. Practical Parser Implementation

```go
// In parser, type annotations are just syntax
func (p *Parser) parseTypeAnnotation() (*TypeExpr, error) {
    if p.current.Type != COLON {
        return nil, nil  // No annotation
    }
    p.advance() // consume ':'
    return p.parseTypeExpr()
}

func (p *Parser) parseTypeExpr() (TypeExpr, error) {
    // Parse base type
    base := p.parseTypeAtom()
    
    // Check for union
    if p.current.Type == PIPE {
        types := []TypeExpr{base}
        for p.current.Type == PIPE {
            p.advance()
            types = append(types, p.parseTypeAtom())
        }
        return &UnionType{types}, nil
    }
    
    return base, nil
}
```

## 9. Why This Separation Matters

1. **Simpler parser** - just recognizes syntax
2. **Better errors** - type errors have full program context
3. **Faster parsing** - no type checking overhead
4. **IDE friendly** - can parse incomplete code
5. **Modular** - can swap type systems without changing parser

## 10. Alternative: Gradual Typing

Some languages (TypeScript, Python) parse with no type checking, then optionally type check:

```python
# Python parses this fine:
def add(x: CompletelyMadeUpType) -> NoSuchType:
    return x + 1

# mypy (type checker) complains later
```

M30 could do similar - parse all type annotations, but type checking could be:
- Mandatory at compile time
- Optional in REPL/development
- Configurable strictness levels