# Task 2: Structural Skeleton (PAGS Phase II)

## Objective
Implement **Phase II: Skeleton** and **ABGS Steps 3-4** from the Protocol for Algorithmic Grammar Synthesis.

## Prerequisites
- Task 1 completed with all atom tests passing
- Build upon the grammar.js from Task 1

## Required Files (Must Be in Current Directory)

Before starting this task, run:

```bash
ls PAGS_PROTOCOL.md general_rules_de.md CONTEXT.md *_history.json
```

**All 4 files must exist. If not, complete Prerequisites phase first.**

### File Usage in This Task

- **PAGS_PROTOCOL.md:** Referenced for Phase II definitions
- **general_rules_de.md:** Referenced for USG NodeKind mappings (Section III.A)
- **CONTEXT.md:** Contains references
- **<language>_history.json:** Version-specific syntax changes

## Context Documents
- Read: `PAGS_PROTOCOL.md` (Phase II section)
- Use: `CONTEXT.md` (Structural Hierarchy section)
- Use: `general_rules_de.md` (Section III.A for NodeKind taxonomy)

## USG Compliance Requirements

**CRITICAL:** This task creates the first `SemanticNode`s. Every created node MUST be validated against `general_rules_de.md`.

### NodeKind Assignment Protocol

**For each tree-sitter node:**

1. **Identify Category** (from general_rules_de.md Section III.A):
   - Is it a definition? → `FunctionDef`, `ClassDef`, `VariableDef`
   - Is it a container? → `Block`, `Module`
   - Is it a statement? → `IfStatement`, `ReturnStatement`

2. **Determine NodeKind:**
   ```javascript
   // Tree-sitter Rule
   function_definition: $ => seq(...),

   // Implies USG NodeKind: FunctionDef
   // Metadata Requirements from III.A:
   // - throws: array<string> (mandatory)
   ```

3. **Validate Against Prohibitions:**
   ```javascript
   // ❌ FORBIDDEN (Principle 1):
   function_definition: $ => seq(
     'func',
     field('name', $.identifier),
     field('parameters', $.parameter_list),
     field('body', $.block),
     // ❌ NEVER:
     field('parameter_count', /* derived */)  // Derivable from graph!
   ),
   ```

### Metadata Schema Enforcement

**For FunctionDef (Example):**

Consult `general_rules_de.md` Section III.A Table "II. Funktionen & Logik":

| NodeKind | Kritische Metadaten (Obligatorisch) |
|----------|-------------------------------------|
| FunctionDef | `throws` (array<string>) |

**Additionally:** Section IV "Allgemeine Verhaltens-Flags":
- `visibility` (string): "public", "private", "protected"
- `is_async` (boolean)
- `is_static` (boolean)

**Implementation Note:**
Tree-sitter grammar defines ONLY structure. Metadata will be populated LATER in transform process. BUT: You must ENSURE all necessary information is EXTRACTABLE.

```javascript
// CORRECT: All info for metadata is available in fields
function_definition: $ => seq(
  optional($.visibility_modifier),  // For metadata.visibility
  optional('async'),                // For metadata.is_async
  'func',
  field('name', $.identifier),
  field('parameters', $.parameter_list),
  optional(seq('throws', field('exceptions', $.exception_list))),  // For metadata.throws
  field('body', $.block)
),
```

### Traversal Strategy Assignment

**For each node, document:**

| Tree-sitter Node | USG NodeKind | Valid Version Ranges | TraversalStrategy | Rationale |
|------------------|--------------|----------------------|-------------------|-----------|
| function_definition | FunctionDef | [["0.0", null]] | CreateAndContain | All versions |
| match_statement | MatchExpression | [["3.10", null]] | CreateAndContain | PEP 634 (3.10+) |
| print_statement | PrintStatement | [[null, "3.0"]] | CreateAndContain | Python 2.x only |
| _statement | (none) | N/A | Dissolve | Wrapper |

**Save in:** `USG_MAPPING.md`

## Implementation Requirements

### 1. Modify Root Structure

Replace temporary root with proper hierarchy:

```javascript
rules: {
  // NEW ROOT DEFINITION
  // Logic from PAGS: "Define the container first, then the contents"
  source_file: $ => repeat($._top_level_item),

  _top_level_item: $ => choice(
    // Use items from language specification
    $.function_definition,
    $.class_definition,
    // [ADD OTHER TOP-LEVEL ITEMS FROM SPEC]
  ),

  // Keep atoms from Task 1
  identifier: $ => /[a-zA-Z_][a-zA-Z0-9_]*/,
  number: $ => token(choice(...)),
  string: $ => /"([^"\\]|\\(.|
))*"/,
  comment: $ => token(/\/\/.*/),
  // etc.
}
```

### 2. Implement ABGS Step 3: Hierarchical Skeleton Definition

**Per PAGS Protocol requirement:**
> "Use `_` (underscore) prefixes for 'hidden rules' (nodes that should not appear in the final syntax tree)"

#### function_definition

Reference PAGS example structure:
```javascript
// ALGORITHMIC NOTE: Apply fields NOW per protocol
function_definition: $ => seq(
  'func',  // or keyword from your language
  field('name', $.identifier),
  field('parameters', $.parameter_list),
  field('body', $.block)
),
```

**Field requirements from general_rules_de.md Section III.A:**
- Implement exactly the fields specified for function_definition

**USG NodeKind:** FunctionDef
**Metadata Requirements:** `throws` (array<string>)

#### parameter_list

From PAGS Protocol:
> "Comma separation logic: optional(seq(item, repeat(seq(',', item))))"

```javascript
parameter_list: $ => seq(
  '(',
  optional(seq(
    $.parameter,
    repeat(seq(',', $.parameter))
  )),
  ')'
),

parameter: $ => seq(
  field('name', $.identifier),
  // Add type annotation if language supports it
  optional(seq(':', field('type', $.type_identifier)))
),
```

**USG NodeKind:** Parameter (for $.parameter)
**Metadata Requirements:** `param_type` (string), `default_value` (string)

#### block

```javascript
block: $ => seq(
  '{',
  repeat($._statement),
  '}'
),
```

**USG NodeKind:** Block
**Metadata Requirements:** `scope_level` (int), `condition_pragma` (string)

#### class_definition (if applicable)

Implement based on language specification.

```javascript
class_definition: $ => seq(
  'class',
  field('name', $.identifier),
  optional(seq('extends', field('superclass', $.identifier))),
  field('body', $.class_body)
),

class_body: $ => seq(
  '{',
  repeat(choice(
    $.method_definition,
    $.field_declaration
  )),
  '}'
),
```

**USG NodeKind:** ClassDef
**Metadata Requirements:** `base_types` (array<string>), `is_struct` (bool), `is_generic` (bool)

### 3. Implement ABGS Step 4: Statement & Control Flow Synthesis

Reference language specification for statement types.

**Per PAGS Protocol:**
> "Use `choice` to cover all syntactic variants"

```javascript
_statement: $ => choice(
  $.return_statement,
  $.if_statement,
  $.while_statement,
  $.for_statement,
  $.expression_statement,
  // [ADD ALL STATEMENT TYPES FROM SPEC]
),
```

#### return_statement
```javascript
return_statement: $ => seq(
  'return',
  optional(field('value', $.expression)),
  ';'  // or appropriate terminator
),
```

**USG NodeKind:** ReturnStatement
**Metadata Requirements:** `returned_expression` (string)

#### if_statement
```javascript
if_statement: $ => seq(
  'if',
  field('condition', $.condition),
  field('consequence', $.block),
  optional(seq(
    'else',
    field('alternative', choice($.block, $.if_statement))
  ))
),

// Condition might be parenthesized or not - check spec
condition: $ => choice(
  $.expression,
  seq('(', $.expression, ')')
),
```

**USG NodeKind:** IfStatement
**Metadata Requirements:** `condition` (string), `is_ternary` (bool)

#### while_statement

```javascript
while_statement: $ => seq(
  'while',
  field('condition', $.condition),
  field('body', $.block)
),
```

**USG NodeKind:** Loop
**Metadata Requirements:** `loop_type` (string: "while"), `condition` (string)

#### for_statement

```javascript
// Example for C-style for loop
for_statement: $ => seq(
  'for',
  '(',
  field('initializer', optional($._statement)),
  ';',
  field('condition', optional($.expression)),
  ';',
  field('increment', optional($.expression)),
  ')',
  field('body', $.block)
),
```

**USG NodeKind:** Loop
**Metadata Requirements:** `loop_type` (string: "for"), `condition` (string), `initializer` (string)

**Apply field() to ALL semantic child nodes** per PAGS requirement:
- condition, consequence, alternative (for if)
- condition, body (for while)
- initializer, condition, increment, body (for for-loop)

#### expression_statement
```javascript
expression_statement: $ => seq(
  $.expression,
  ';'  // or appropriate terminator
),
```

**NOTE:** This is a syntactic wrapper. It should use `Dissolve` TraversalStrategy.

### 4. Temporary Expression Placeholder

Since Phase III handles expressions, create minimal placeholder:

```javascript
expression: $ => choice(
  $.identifier,
  $.number,
  $.string,
  // More will be added in Task 3
),
```

### 5. Create Test Corpus

**File:** `test/corpus/02_structure.txt`

**Required test categories:**

```
================================================================================
Function: Empty body
================================================================================

func myFunction() {
}

--------------------------------------------------------------------------------

(source_file
  (function_definition
    name: (identifier)
    parameters: (parameter_list)
    body: (block)))

================================================================================
Function: With parameters
================================================================================

func add(a, b) {
}

--------------------------------------------------------------------------------

(source_file
  (function_definition
    name: (identifier)
    parameters: (parameter_list
      (parameter name: (identifier))
      (parameter name: (identifier)))
    body: (block)))

================================================================================
Function: With return statement
================================================================================

func getValue() {
  return 42;
}

--------------------------------------------------------------------------------

(source_file
  (function_definition
    name: (identifier)
    parameters: (parameter_list)
    body: (block
      (return_statement (number)))))

================================================================================
If statement: Simple
================================================================================

func test() {
  if condition {
    return;
  }
}

--------------------------------------------------------------------------------

(source_file
  (function_definition
    name: (identifier)
    parameters: (parameter_list)
    body: (block
      (if_statement
        condition: (identifier)
        consequence: (block
          (return_statement))))))

================================================================================
If statement: With else
================================================================================

func test() {
  if condition {
    return 1;
  } else {
    return 2;
  }
}

--------------------------------------------------------------------------------

(source_file
  (function_definition
    name: (identifier)
    parameters: (parameter_list)
    body: (block
      (if_statement
        condition: (identifier)
        consequence: (block
          (return_statement (number)))
        alternative: (block
          (return_statement (number)))))))

================================================================================
While loop: Simple
================================================================================

func loop() {
  while condition {
    return;
  }
}

--------------------------------------------------------------------------------

(source_file
  (function_definition
    name: (identifier)
    parameters: (parameter_list)
    body: (block
      (while_statement
        condition: (identifier)
        body: (block
          (return_statement))))))

================================================================================
For loop: C-style
================================================================================

func iterate() {
  for (i = 0; i < 10; i = i + 1) {
    return;
  }
}

--------------------------------------------------------------------------------

(source_file
  (function_definition
    name: (identifier)
    parameters: (parameter_list)
    body: (block
      (for_statement
        initializer: (expression_statement
          (identifier))
        condition: (identifier)
        increment: (identifier)
        body: (block
          (return_statement))))))

================================================================================
Nested control flow
================================================================================

func nested() {
  if outer {
    while inner {
      return;
    }
  }
}

--------------------------------------------------------------------------------

(source_file
  (function_definition
    name: (identifier)
    parameters: (parameter_list)
    body: (block
      (if_statement
        condition: (identifier)
        consequence: (block
          (while_statement
            condition: (identifier)
            body: (block
              (return_statement))))))))

================================================================================
Class: Empty (if applicable)
================================================================================

class MyClass {
}

--------------------------------------------------------------------------------

(source_file
  (class_definition
    name: (identifier)
    body: (class_body)))

================================================================================
Class: With methods (if applicable)
================================================================================

class MyClass {
  func method() {
  }
}

--------------------------------------------------------------------------------

(source_file
  (class_definition
    name: (identifier)
    body: (class_body
      (method_definition
        name: (identifier)
        parameters: (parameter_list)
        body: (block)))))
```

**Create minimum 15 test cases** covering:
- Functions (empty, with params, with body)
- Classes (if applicable)
- Each statement type
- Nested structures

## Verification Checklist

1. **Generation Clean:**
   ```bash
   tree-sitter generate
   ```
   Expected: No errors

2. **Tests Pass:**
   ```bash
   tree-sitter test
   ```
   Expected: All tests pass

3. **Field Visibility:**
   ```bash
   tree-sitter parse test_file.ext
   ```
   Expected: Check field names appear correctly (name:, body:, condition:, etc.)

4. **Manual Inspection:**
   ```bash
   tree-sitter parse --debug test_file.ext
   ```
   Expected: Verify no unnecessary wrapper nodes

5. **USG Compliance:**
   - [ ] All nodes mapped to correct NodeKind (documented in USG_MAPPING.md)
   - [ ] All required fields present (documented in FIELD_COVERAGE.md)
   - [ ] No forbidden redundant metadata

## Deliverables

1. **Updated `grammar.js`** with complete structural skeleton

2. **`test/corpus/02_structure.txt`** with minimum 15 test cases

3. **`USG_MAPPING.md`** documenting:
   - Complete table: Tree-sitter Node → USG NodeKind → Metadata
   - TraversalStrategy per Node
   - Rationale for Dissolve decisions

4. **`FIELD_COVERAGE.md`** documenting:
   - Audit of all field() assignments
   - Cross-reference to general_rules_de.md field requirements
   - Missing fields (with rationale if any)

5. **`STRUCTURE_VERIFICATION.md`** documenting:
   - Hierarchy design decisions
   - Any wrapper rules created and why

## Success Criteria

From PAGS Protocol:
> "Define the container first, then the contents. We use `_` for hidden rules."

**Verification:**
- All _top_level_item variants parse correctly
- Field names visible in parse output
- No unnecessary nesting visible in AST

**All structure tests MUST pass before proceeding to Task 3.**
