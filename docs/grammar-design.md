# M30 Grammar Formalization

## Grammar Design Decisions

### 1. Parser Technology Choice

**Options:**
1. **Hand-written Recursive Descent** (like Go, m28)
   - Pros: Full control, good errors, handles indentation naturally
   - Cons: More code, harder to maintain

2. **PEG (Parsing Expression Grammar)**
   - Pros: No ambiguity, good for indentation, composable
   - Cons: Can be memory hungry, left recursion issues

3. **ANTLR with Indentation Tokens**
   - Pros: Mature tooling, good IDE support
   - Cons: Complex indentation handling

4. **Tree-sitter**
   - Pros: Incremental parsing, error recovery, IDE-ready
   - Cons: More complex to write

**Recommendation:** Start with hand-written recursive descent (like m28) for control, migrate to Tree-sitter later for IDE support.

### 2. Lexical Structure

```ebnf
# Tokens
IDENTIFIER = [a-zA-Z_][a-zA-Z0-9_]*
NUMBER = [0-9]+(\.[0-9]+)?([eE][+-]?[0-9]+)?
STRING = '"' (escape_seq | ~["])* '"' | "'" (escape_seq | ~['])* "'"
FSTRING = 'f"' (escape_seq | '{' expr '}' | ~["])* '"'

# Keywords (reserved)
KEYWORDS = 'let' | 'mut' | 'def' | 'class' | 'type' | 'match' | 'with' 
         | 'if' | 'then' | 'else' | 'for' | 'while' | 'in' | 'return'
         | 'spawn' | 'parallel' | 'race' | 'import' | 'from' | 'export'
         | 'try' | 'catch' | 'raise' | 'panic' | 'extends' | 'override'
         | 'module' | 'and' | 'or' | 'not' | 'None' | 'Ok' | 'Err'

# Operators (by precedence, lowest to highest)
OPERATORS = 
    '='                           # Assignment (right-assoc)
    'or'                          # Logical OR
    'and'                         # Logical AND  
    'not' | 'is' | 'in'          # Unary/membership
    '==' | '!=' | '<' | '>' | '<=' | '>=' # Comparison
    '|'                           # Union type / bitwise OR
    '^'                           # Bitwise XOR
    '&'                           # Intersection type / bitwise AND
    '<<' | '>>'                   # Bit shift
    '+' | '-'                     # Addition/subtraction
    '*' | '/' | '//' | '%'        # Multiplication/division
    '**'                          # Exponentiation (right-assoc)
    '-' | '+' | '~'              # Unary
    '.' | '[' | '('              # Call/index/member (left-assoc)

# Special tokens
INDENT = <increased indentation>
DEDENT = <decreased indentation>
NEWLINE = '\n' | '\r\n'
```

### 3. Indentation Handling

```python
# Lexer maintains indent stack
indent_stack = [0]

# At start of line:
if current_indent > indent_stack[-1]:
    indent_stack.append(current_indent)
    emit(INDENT)
elif current_indent < indent_stack[-1]:
    while indent_stack[-1] > current_indent:
        indent_stack.pop()
        emit(DEDENT)
    if indent_stack[-1] != current_indent:
        error("Inconsistent indentation")

# Significant newlines vs continuation
significant_newline = not inside_parens_or_brackets and 
                     not line_ends_with(':' or '=' or '\')
```

### 4. Core Grammar Rules

```ebnf
# Program structure
program = (import_stmt | top_level_def)* EOF

import_stmt = 
    | 'import' module_path ('as' IDENTIFIER)?
    | 'from' module_path 'import' import_list

module_path = IDENTIFIER ('.' IDENTIFIER)*
import_list = IDENTIFIER (',' IDENTIFIER)* | '(' IDENTIFIER (',' IDENTIFIER)* ')'

# Top level definitions
top_level_def = 
    | module_def
    | type_def
    | class_def
    | function_def
    | let_binding

module_def = 'module' IDENTIFIER ':' INDENT module_body DEDENT
module_body = (export_stmt | top_level_def)*
export_stmt = 'export' (IDENTIFIER | function_def | type_def)

# Type definitions
type_def = 'type' IDENTIFIER type_params? '=' type_expr

type_expr =
    | type_union
    | type_record
    | type_enum

type_union = type_atom ('|' type_atom)*
type_record = '{' (IDENTIFIER ':' type_expr ',')* '}'
type_enum = ('|' IDENTIFIER type_constructor?)+

type_constructor = '(' type_expr (',' type_expr)* ')'

# Type annotations
type_annotation = ':' type_expr
type_params = '[' IDENTIFIER (',' IDENTIFIER)* ']'
type_constraints = 'where' constraint (',' constraint)*
constraint = IDENTIFIER ':' type_class ('+' type_class)*

# Class definitions  
class_def = 'class' IDENTIFIER type_params? extends_clause? ':' INDENT class_body DEDENT
extends_clause = 'extends' type_expr
class_body = (field_def | method_def)*

field_def = 'mut'? IDENTIFIER type_annotation
method_def = override? 'def' IDENTIFIER '(' params ')' return_type? '=' method_body
override = 'override'

# Function definitions
function_def = 'let' IDENTIFIER type_params? pattern_params return_type? '=' expr

pattern_params = pattern | '(' pattern (',' pattern)* ')'
pattern = 
    | IDENTIFIER type_annotation?
    | literal
    | '[' pattern (',' pattern)* (',' '...' IDENTIFIER)? ']'  # List pattern
    | '{' field_pattern (',' field_pattern)* '}'               # Record pattern
    | IDENTIFIER '(' pattern (',' pattern)* ')'                # Constructor

return_type = '->' type_expr

# Statements
stmt = 
    | let_binding
    | assignment
    | expr_stmt
    | return_stmt
    | raise_stmt
    | if_stmt
    | for_stmt
    | while_stmt
    | match_stmt
    | with_stmt
    | try_stmt
    | parallel_block

let_binding = 'let' pattern '=' expr
assignment = lvalue '=' expr
lvalue = IDENTIFIER | member_expr | index_expr

# Control flow
if_stmt = 'if' expr ':' INDENT stmt+ DEDENT elif_clause* else_clause?
elif_clause = 'elif' expr ':' INDENT stmt+ DEDENT  
else_clause = 'else' ':' INDENT stmt+ DEDENT

# One-line if expression
if_expr = 'if' expr 'then' expr 'else' expr

for_stmt = 'for' pattern 'in' expr ':' INDENT stmt+ DEDENT
while_stmt = 'while' expr ':' INDENT stmt+ DEDENT

match_stmt = 'match' expr 'with' INDENT match_arm+ DEDENT
match_arm = '|' pattern guard? '->' (expr | INDENT stmt+ DEDENT)
guard = 'if' expr

# Exception handling
try_stmt = 'try' ':' INDENT stmt+ DEDENT catch_clause* finally_clause?
catch_clause = 'catch' pattern ('as' IDENTIFIER)? ':' INDENT stmt+ DEDENT
finally_clause = 'finally' ':' INDENT stmt+ DEDENT

raise_stmt = 'raise' expr
panic_expr = 'panic' '(' expr ')'

# Resource management
with_stmt = 'with' with_item (',' with_item)* ':' INDENT stmt+ DEDENT
with_item = IDENTIFIER '=' expr

# Concurrency
parallel_block = parallel_type ':' INDENT spawn_stmt+ DEDENT
parallel_type = 'parallel' | 'race'
spawn_stmt = spawn_expr | let_binding_spawn | for_stmt_spawn

spawn_expr = 'spawn' ':' (expr | INDENT stmt+ DEDENT)
let_binding_spawn = 'let' IDENTIFIER '=' spawn_expr
for_stmt_spawn = 'for' pattern 'in' expr ':' spawn_expr

# Expressions (precedence handled in parser)
expr = 
    | lambda_expr
    | if_expr
    | match_expr
    | binary_expr
    | unary_expr
    | postfix_expr
    | primary_expr

lambda_expr = pattern_params '->' expr

match_expr = 'match' expr 'with' match_arm+

binary_expr = expr binary_op expr  # Precedence in parser

unary_expr = unary_op expr

postfix_expr = 
    | call_expr
    | member_expr  
    | index_expr
    | slice_expr
    | error_prop_expr

call_expr = expr '(' args ')'
args = (arg (',' arg)*)? ','?
arg = expr | IDENTIFIER '=' expr | '*' expr | '**' expr

member_expr = expr '.' IDENTIFIER
index_expr = expr '[' expr ']'
slice_expr = expr '[' expr? ':' expr? (':' expr?)? ']'
error_prop_expr = expr '?'

# Literals and primaries
primary_expr = 
    | literal
    | IDENTIFIER
    | list_expr
    | dict_expr
    | tuple_expr
    | set_expr
    | paren_expr

literal = NUMBER | STRING | FSTRING | 'true' | 'false' | 'None'

list_expr = '[' (expr (',' expr)*)? ']'
dict_expr = '{' (dict_entry (',' dict_entry)*)? '}'
dict_entry = expr ':' expr
set_expr = '{' expr (',' expr)+ '}'  # Note: {x} is ambiguous, need 2+ elements
tuple_expr = '(' ')' | '(' expr ',' ')' | '(' expr (',' expr)+ ')'
paren_expr = '(' expr ')'

# List comprehensions
list_comp = '[' expr 'for' pattern 'in' expr comp_clause* ']'
comp_clause = 'for' pattern 'in' expr | 'if' expr
```

### 5. Ambiguity Resolution

**Problem areas:**

1. **Set vs Dict literals**
```m30
{}          # Empty dict
{x}         # Error: ambiguous
{x,}        # Set with one element  
{x: y}      # Dict
{x, y}      # Set
```

2. **Type vs Value syntax**
```m30
List[int]   # Type
list[int]() # Constructor call
```

3. **Indentation after operators**
```m30
# Should allow
let x = very_long_expression +
        another_expression

# But not
let x = 
    expression  # Error: unexpected indent
```

### 6. Parser Implementation Strategy

```go
// parser/parser.go
type Parser struct {
    lexer      *Lexer
    current    Token
    previous   Token
    errors     []ParseError
}

// Precedence climbing for expressions
func (p *Parser) expression(minPrec int) Expr {
    left := p.unary()
    
    for p.current.Precedence() >= minPrec {
        op := p.current
        p.advance()
        right := p.expression(op.Precedence() + 1) // +1 for left assoc
        left = BinaryExpr{left, op, right}
    }
    
    return left
}

// Indentation-aware parsing
func (p *Parser) block() []Stmt {
    p.expect(INDENT)
    stmts := []Stmt{}
    
    for p.current.Type != DEDENT {
        stmts = append(stmts, p.statement())
    }
    
    p.expect(DEDENT)
    return stmts
}
```

### 7. Error Recovery

```m30
# Parser should recover at:
- Statement boundaries (NEWLINE at indent level 0)
- Block boundaries (DEDENT)
- Comma-separated lists
- Closing delimiters

# Good error messages:
let x = 
    5 +    # Error: Expected expression after '+', found newline
    
class Point
    x: int  # Error: Expected ':' after class name
```

## Next Steps

1. **Implement lexer** with indentation handling
2. **Build recursive descent parser** following grammar
3. **Add precedence climbing** for expressions  
4. **Test with example programs** from design docs
5. **Iterate on syntax** based on parsing challenges

The grammar is designed to be:
- **LL(1)** where possible (single token lookahead)
- **Unambiguous** through careful precedence/associativity
- **Error-recoverable** at natural boundaries
- **IDE-friendly** for future tooling