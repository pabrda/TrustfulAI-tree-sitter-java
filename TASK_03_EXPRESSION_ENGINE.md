# Task 3: Expression Engine (PAGS Phase III)

## Objective
Implement **Phase III: Expression Engine** and **ABGS Step 5** from the Protocol for Algorithmic Grammar Synthesis.

## Prerequisites
- Task 2 completed with all structure tests passing
- Build upon grammar.js from Task 2

## Required Files (Must Be in Current Directory)

Before starting this task, run:

```bash
ls PAGS_PROTOCOL.md general_rules_de.md CONTEXT.md *_history.json
```

**All 4 files must exist. If not, complete Prerequisites phase first.**

### File Usage in This Task

- **PAGS_PROTOCOL.md:** Referenced for Phase III definitions (marked as CRITICAL)
- **CONTEXT.md:** Contains operator precedence information
- **<language>_history.json:** Version-specific operator changes

## Context Documents
- Read: `PAGS_PROTOCOL.md` (Phase III section - marked as CRITICAL)
- Use: Language specification for operator precedence table

## Implementation Requirements

### 1. Define PREC Constant

**Source:** `specs/operators.yaml` (generated in Task 0)
**Implementation:** Generate PREC constant directly from YAML:

```javascript
// PRECEDENCE TABLE
// AUTO-GENERATED from specs/operators.yaml
// Precedence values validated against spec in Task 0
const PREC = {
  ASSIGN: 1,
  OR: 2,
  AND: 3,
  RELATIONAL: 4,
  ADD: 5,
  MULT: 6,
  UNARY: 7,
  CALL: 8,
  // Complete table from operators.yaml
};

module.exports = grammar({
  name: '[LANGUAGE_NAME]',
  // ...
});
```

**Document in EXTRACTION_LOG.md:**
- Source of precedence information (spec section)
- Complete precedence hierarchy
- Any language-specific precedence rules
**Verification:** Cross-check against `specs/operators.yaml` → all operators present.

### 2. Implement Monolithic Expression Rule

**Per PAGS Protocol:**
> "We do not use nested rules for precedence (e.g., `term`, `factor`, `unary`). This is outdated. Tree-sitter provides a **precedence table**."

Replace placeholder expression rule:

```javascript
expression: $ => choice(
  // Atoms
  $.identifier,
  $.number,
  $.string,

  // Complex expressions
  $.binary_expression,
  $.unary_expression,
  $.call_expression,
  $.parenthesized_expression,
  // [ADD OTHER EXPRESSION TYPES FROM YOUR LANGUAGE]
),
```

### 3. Implement Binary Expressions with Precedence

**Critical requirement from PAGS:**
> "Apply precedence to every binary and unary operator"

**Determine associativity from language spec:**
- Left-associative: `1 + 2 + 3` = `(1 + 2) + 3` → use `prec.left()`
- Right-associative: `a = b = c` = `a = (b = c)` → use `prec.right()`

For each operator in language specification:

```javascript
binary_expression: $ => choice(
  // Arithmetic operators (typically left-associative)
  prec.left(PREC.ADD, seq(
    field('left', $.expression),
    field('operator', '+'),
    field('right', $.expression)
  )),
  prec.left(PREC.ADD, seq(
    field('left', $.expression),
    field('operator', '-'),
    field('right', $.expression)
  )),
  prec.left(PREC.MULT, seq(
    field('left', $.expression),
    field('operator', '*'),
    field('right', $.expression)
  )),
  prec.left(PREC.MULT, seq(
    field('left', $.expression),
    field('operator', '/'),
    field('right', $.expression)
  )),
  prec.left(PREC.MULT, seq(
    field('left', $.expression),
    field('operator', '%'),
    field('right', $.expression)
  )),

  // Relational operators (typically left-associative)
  prec.left(PREC.RELATIONAL, seq(
    field('left', $.expression),
    field('operator', choice('<', '>', '<=', '>=', '==', '!=')),
    field('right', $.expression)
  )),

  // Logical operators
  prec.left(PREC.OR, seq(
    field('left', $.expression),
    field('operator', '||'),
    field('right', $.expression)
  )),
  prec.left(PREC.AND, seq(
    field('left', $.expression),
    field('operator', '&&'),
    field('right', $.expression)
  )),

  // Assignment (typically right-associative)
  // ALGORITHMIC NOTE: Right-associative per PAGS example "a = b = c"
  prec.right(PREC.ASSIGN, seq(
    field('left', $.expression),
    field('operator', '='),
    field('right', $.expression)
  )),

  // [ADD ALL OPERATORS FROM LANGUAGE SPEC WITH CORRECT ASSOCIATIVITY]
),
```

**USG NodeKind:** Expression
**Metadata Requirements:** `operator` (string), `is_parenthesized` (bool)

### 4. Implement Unary Expressions

```javascript
unary_expression: $ => prec(PREC.UNARY, seq(
  field('operator', choice('-', '!', '~', '+')),  // From language spec
  field('operand', $.expression)
)),
```

**USG NodeKind:** Expression
**Metadata Requirements:** `operator` (string)

### 5. Implement Call Expression

```javascript
call_expression: $ => prec(PREC.CALL, seq(
  field('function', $.expression),
  field('arguments', $.argument_list)
)),

argument_list: $ => seq(
  '(',
  optional(seq(
    $.argument,
    repeat(seq(',', $.argument))
  )),
  ')'
),

argument: $ => $.expression,
```

**USG NodeKind:** Call
**Metadata Requirements:** `target_name` (string), `is_await` (bool)

### 6. Implement Parenthesized Expression

**Important:** This is a syntactic wrapper (Principle 3).

```javascript
parenthesized_expression: $ => seq(
  '(',
  $.expression,
  ')'
),
```

**Metadata Note:** The child expression should get `is_parenthesized: true` metadata.

### 7. Implement Additional Expression Types

Based on language features:

#### Member Access (if applicable)
```javascript
member_expression: $ => prec(PREC.CALL, seq(
  field('object', $.expression),
  choice('.', '->'),
  field('property', $.identifier)
)),
```

**USG NodeKind:** MemberAccess
**Metadata Requirements:** `operator` (string: ".", "->", "?.")

#### Index Access (if applicable)
```javascript
index_expression: $ => prec(PREC.CALL, seq(
  field('object', $.expression),
  '[',
  field('index', $.expression),
  ']'
)),
```

**USG NodeKind:** IndexAccess
**Metadata Requirements:** `is_slice` (bool)

#### Lambda/Anonymous Functions (if applicable)
```javascript
lambda_expression: $ => seq(
  field('parameters', $.lambda_parameters),
  '=>',
  field('body', choice($.expression, $.block))
),

lambda_parameters: $ => choice(
  $.identifier,
  seq('(', optional($.parameter_list), ')')
),
```

**USG NodeKind:** LambdaDef
**Metadata Requirements:** `parameters` (array<object>), `return_type` (string), `is_capturing` (bool)

### 8. Create Test Corpus

**File:** `test/corpus/03_expressions.txt`

**Critical test from PAGS Protocol:**
> "Precedence handles `1 + 2 * 3` correctly"

```
================================================================================
Precedence: Multiplication before addition
================================================================================

1 + 2 * 3

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (binary_expression
      left: (number)
      operator: "+"
      right: (binary_expression
        left: (number)
        operator: "*"
        right: (number)))))

================================================================================
Precedence: Parentheses override
================================================================================

(1 + 2) * 3

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (binary_expression
      left: (parenthesized_expression
        (binary_expression
          left: (number)
          operator: "+"
          right: (number)))
      operator: "*"
      right: (number))))

================================================================================
Precedence: Division and multiplication
================================================================================

10 / 2 * 3

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (binary_expression
      left: (binary_expression
        left: (number)
        operator: "/"
        right: (number))
      operator: "*"
      right: (number))))

================================================================================
Associativity: Left-associative addition
================================================================================

1 + 2 + 3

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (binary_expression
      left: (binary_expression
        left: (number)
        operator: "+"
        right: (number))
      operator: "+"
      right: (number))))

================================================================================
Associativity: Right-associative assignment
================================================================================

a = b = c

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (binary_expression
      left: (identifier)
      operator: "="
      right: (binary_expression
        left: (identifier)
        operator: "="
        right: (identifier)))))

================================================================================
Associativity: Subtraction (left)
================================================================================

10 - 5 - 2

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (binary_expression
      left: (binary_expression
        left: (number)
        operator: "-"
        right: (number))
      operator: "-"
      right: (number))))

================================================================================
Unary: Negation
================================================================================

-5

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (unary_expression
      operator: "-"
      operand: (number))))

================================================================================
Unary: Logical NOT
================================================================================

!true

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (unary_expression
      operator: "!"
      operand: (identifier))))

================================================================================
Unary: Precedence over binary
================================================================================

-a * b

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (binary_expression
      left: (unary_expression
        operator: "-"
        operand: (identifier))
      operator: "*"
      right: (identifier))))

================================================================================
Unary: Double negation
================================================================================

!!value

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (unary_expression
      operator: "!"
      operand: (unary_expression
        operator: "!"
        operand: (identifier)))))

================================================================================
Call: Simple
================================================================================

func()

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (call_expression
      function: (identifier)
      arguments: (argument_list))))

================================================================================
Call: With arguments
================================================================================

func(a, b, c)

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (call_expression
      function: (identifier)
      arguments: (argument_list
        (argument (identifier))
        (argument (identifier))
        (argument (identifier))))))

================================================================================
Call: Nested
================================================================================

outer(inner())

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (call_expression
      function: (identifier)
      arguments: (argument_list
        (argument
          (call_expression
            function: (identifier)
            arguments: (argument_list)))))))

================================================================================
Call: Method call (if applicable)
================================================================================

obj.method()

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (call_expression
      function: (member_expression
        object: (identifier)
        property: (identifier))
      arguments: (argument_list))))

================================================================================
Relational: Less than
================================================================================

a < b

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (binary_expression
      left: (identifier)
      operator: "<"
      right: (identifier))))

================================================================================
Relational: Chained (if language allows)
================================================================================

a < b < c

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (binary_expression
      left: (binary_expression
        left: (identifier)
        operator: "<"
        right: (identifier))
      operator: "<"
      right: (identifier))))

================================================================================
Logical: AND has precedence over OR
================================================================================

a || b && c

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (binary_expression
      left: (identifier)
      operator: "||"
      right: (binary_expression
        left: (identifier)
        operator: "&&"
        right: (identifier)))))

================================================================================
Complex: All operators combined
================================================================================

a + b * c == d && !e || f()

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (binary_expression
      left: (binary_expression
        left: (binary_expression
          left: (identifier)
          operator: "+"
          right: (binary_expression
            left: (identifier)
            operator: "*"
            right: (identifier)))
        operator: "=="
        right: (identifier))
      operator: "||"
      right: (call_expression
        function: (identifier)
        arguments: (argument_list)))))

================================================================================
Parentheses: Nested
================================================================================

((a))

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (parenthesized_expression
      (parenthesized_expression
        (identifier)))))

================================================================================
Mixed: Unary with parentheses
================================================================================

-(a + b)

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (unary_expression
      operator: "-"
      operand: (parenthesized_expression
        (binary_expression
          left: (identifier)
          operator: "+"
          right: (identifier))))))
```

**Create minimum 20 test cases** covering:
- Each precedence level (minimum 2 tests per level)
- Associativity (left vs right)
- Unary operators
- Function calls
- Complex nested expressions

## Verification Checklist

1. **Generation:**
   ```bash
   tree-sitter generate
   ```
   Expected: May have conflicts (will be addressed in Task 4)

2. **Tests Pass:**
   ```bash
   tree-sitter test
   ```
   Expected: All expression tests pass

3. **Manual Precedence Verification:**
   ```bash
   echo "1 + 2 * 3" | tree-sitter parse
   ```
   Expected: Confirm AST shows `1 + (2 * 3)`

4. **Associativity Verification:**
   ```bash
   echo "a = b = c" | tree-sitter parse
   ```
   Expected: Confirm AST shows `a = (b = c)`

5. **Field Check:**
   ```bash
   tree-sitter parse --debug test_file.ext
   ```
   Expected: Verify left, right, operator fields present

## Deliverables

1. **Updated `grammar.js`** with complete PREC table and expression rules

2. **`test/corpus/03_expressions.txt`** with minimum 20 test cases

3. **`EXPRESSION_PRECEDENCE.md`** documenting:
   - Precedence table rationale (source from spec)
   - Associativity decisions per operator
   - Any edge cases in expression parsing
   - Test results for critical precedence cases

## Success Criteria

From PAGS Protocol Phase III:
> "Construct a monolithic `expression` rule with `choice`. Apply precedence to every binary and unary operator."

**Verification:**
- PREC constant complete and correctly ordered
- All operators have explicit precedence
- Associativity matches language specification
- Test "1 + 2 * 3" parses as (1 + (2 * 3))
- Test "1 + 2 + 3" parses as ((1 + 2) + 3)

**Note:** Conflicts are expected at this stage. They will be resolved in Task 4.
