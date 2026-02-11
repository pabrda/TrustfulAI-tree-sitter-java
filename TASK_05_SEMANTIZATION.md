# Task 5: AST Refinement and Semantization (PAGS Phase V + VII)

## Objective
Implement **Phase V: AST Refinement** and **Phase VII: Semantization** from the Protocol for Algorithmic Grammar Synthesis.

## Prerequisites
- Task 4 completed with zero conflicts
- Build upon grammar.js from Task 4

## Required Files (Must Be in Current Directory)

Before starting this task, run:

```bash
ls PAGS_PROTOCOL.md general_rules_de.md CONTEXT.md *_history.json
```

**All 4 files must exist. If not, complete Prerequisites phase first.**

### File Usage in This Task

- **PAGS_PROTOCOL.md:** Referenced for Phase V + Phase VII definitions
- **general_rules_de.md:** Referenced for USG Supertype Taxonomy (Section III.A) and Metadata Schema (Section IV)
- **CONTEXT.md:** Contains USG mapping requirements

## Context Documents
- Read: `PAGS_PROTOCOL.md` (Phase V + Phase VII sections)
- Use: `general_rules_de.md` (Section III.A for supertypes, Section IV for metadata)

## Implementation Requirements

### Part A: AST Refinement (Phase V)

#### 1. Verify Hidden Rules

**From PAGS Protocol Phase V:**
> "Ensure all wrapper rules (like `_statement`, `_definition`) start with `_`. They will be removed from the final tree."

**Audit grammar.js:**
- Identify all "choice-only" wrapper rules
- Ensure they have `_` prefix
- Document in `AST_REFINEMENT.md`

**Example:**
```javascript
// CORRECT: Hidden wrapper
_statement: $ => choice(
  $.return_statement,
  $.if_statement,
  $.expression_statement
),

// INCORRECT: Visible wrapper (creates noise in AST)
statement: $ => choice(  // ❌ Missing underscore
  $.return_statement,
  $.if_statement,
  $.expression_statement
),
```

**Create audit in AST_REFINEMENT.md:**
```markdown
| Rule Name | Has _ Prefix | Purpose | Correct? |
|-----------|--------------|---------|----------|
| _statement | ✓ | Wrapper for all statement types | ✓ |
| _expression | ✓ | Wrapper for all expression types | ✓ |
| _top_level_item | ✓ | Wrapper for module-level items | ✓ |
```

#### 2. Verify Field Tagging

**From PAGS Protocol Phase V:**
> "Every meaningful child (name, body, condition, left, right) must have a `field()`."

**Cross-reference `general_rules_de.md` Section III.A for required fields.**

**Audit all rules:**
```javascript
// Check every container rule has complete field coverage

function_definition: $ => seq(
  'func',
  field('name', $.identifier),          // ✓ Field present
  field('parameters', $.parameter_list), // ✓ Field present
  field('body', $.block)                 // ✓ Field present
),

// BAD EXAMPLE (missing fields):
if_statement: $ => seq(
  'if',
  $.expression,  // ❌ Missing field('condition', ...)
  $.block        // ❌ Missing field('consequence', ...)
),

// GOOD EXAMPLE:
if_statement: $ => seq(
  'if',
  field('condition', $.expression),
  field('consequence', $.block),
  optional(seq('else', field('alternative', $.block)))
),
```

**Create field audit checklist in `FIELD_COVERAGE.md`:**
```markdown
| Rule | Required Fields (from general_rules_de.md) | Implemented | Missing |
|------|---------------------------------------------|-------------|---------|
| function_definition | name, parameters, body | ✓ | None |
| class_definition | name, superclass (optional), body | ✓ | None |
| if_statement | condition, consequence, alternative | ✓ | None |
| binary_expression | left, operator, right | ✓ | None |
```

#### 3. Implement `inline` Array

**From PAGS Protocol Phase V:**
> "If a rule is simple and used everywhere (e.g., a keyword wrapper), add to `inline` array to reduce parser state size."

**Identify candidates:**
- Single-token wrappers
- Trivial pass-through rules

```javascript
module.exports = grammar({
  name: '[LANGUAGE_NAME]',
  
  // Inline simple wrapper rules
  inline: $ => [
    $._expression_wrapper,
    $._simple_type,
    // [ADD IDENTIFIED CANDIDATES]
  ],
  
  rules: {
    // ...
  }
});
```

### Part B: Semantization (Phase VII)

#### 4. Implement Layer I: Structural Semantics

**Already completed in previous tasks via field() usage.**

**Verify:** Run `tree-sitter parse [test_file]` and confirm fields are visible.

#### 5. Implement Layer II: Categorial Semantics (Supertypes)

**CRITICAL:** Supertypes MUST EXACTLY match categories from `general_rules_de.md` Section III.A.

**Extract from general_rules_de.md:**

Section III.A is divided into categories:
- **I. Struktur & Organisation**
- **II. Funktionen & Logik**
- **III. Daten & Variablen**
- **IV. Kontrollfluss & Fehlerbehandlung**
- **V. Konnektivität & FFI**
- **VI. UX & Domänenspezifisch**
- **VII. Nebenläufigkeit & Asynchronität**
- **VIII. Low-Level & Unsichere Operationen**
- **IX. Metaprogrammierung & Generics**

**Map to Supertypes:**

```javascript
module.exports = grammar({
  name: '[LANGUAGE_NAME]',
  
  // Supertypes from general_rules_de.md categories
  supertypes: $ => [
    // From III.A - Expressions (Category II)
    $._expression,
    
    // From III.A - Statements (Category IV)
    $._statement,
    
    // From III.A - Declarations (Category I, II)
    $._declaration,
    
    // From III.A - Types (Category III)
    $._type,
    
    // [MORE based on language features]
  ],
  
  rules: {
    // Expressions (all NodeKinds from Category II that produce values)
    _expression: $ => choice(
      $.binary_expression,      // → Expression NodeKind
      $.unary_expression,       // → Expression NodeKind
      $.call_expression,        // → Call NodeKind
      $.lambda_expression,      // → LambdaDef NodeKind
      $.identifier,             // → Reference NodeKind (in read context)
      $.number,                 // → Literal NodeKind
      $.string,                 // → Literal NodeKind
      // [ALL expression types from general_rules_de.md Category II]
    ),
    
    // Statements (all NodeKinds from Category IV)
    _statement: $ => choice(
      $.if_statement,           // → IfStatement NodeKind
      $.while_statement,        // → Loop NodeKind
      $.for_statement,          // → Loop NodeKind
      $.return_statement,       // → ReturnStatement NodeKind
      $.expression_statement,   // Wrapper, will be dissolved
      // [ALL statement types from general_rules_de.md Category IV]
    ),
    
    // Declarations (all NodeKinds from Category I + II that are definitions)
    _declaration: $ => choice(
      $.function_definition,    // → FunctionDef NodeKind (definition: true)
      $.class_definition,       // → ClassDef NodeKind (definition: true)
      $.variable_declaration,   // → VariableDef NodeKind (definition: true)
      // [ALL declaration types]
    ),
    
    // Types (all NodeKinds from Category III describing types)
    _type: $ => choice(
      $.type_identifier,        // → Type NodeKind
      $.generic_type,           // → Type NodeKind (type_constructor: "generic")
      $.function_type,          // → Type NodeKind (type_constructor: "function")
      // [ALL type constructs]
    ),
  }
});
```

**Validation Checklist:**

Create `SUPERTYPE_VALIDATION.md`:

```markdown
| Tree-sitter Node | Supertype | USG NodeKind | Category (general_rules) | Valid? |
|------------------|-----------|--------------|---------------------------|--------|
| call_expression | _expression | Call | II. Funktionen & Logik | ✓ |
| if_statement | _statement | IfStatement | IV. Kontrollfluss | ✓ |
| function_definition | _declaration | FunctionDef | II. Funktionen & Logik | ✓ |
| class_definition | _declaration | ClassDef | I. Struktur & Organisation | ✓ |
| binary_expression | _expression | Expression | II. Funktionen & Logik | ✓ |
```

**Critical Rule:**

NEVER invent a supertype not documented in `general_rules_de.md`. If a language feature has no matching category, create a `_[feature]_specific` group and document in `USG_MAPPING.md` with rationale.

#### 6. Implement Layer III: Relational Semantics (Query Files)

**Create directory:** `queries/`

##### File: `queries/highlights.scm`

**From PAGS Protocol Phase VII Layer III:**
> "Semantics are defined outside the grammar."

**Implement syntax highlighting queries:**

```scheme
; Keywords
[
  "if"
  "else"
  "for"
  "while"
  "return"
  "func"
  "class"
  ; [ADD ALL KEYWORDS]
] @keyword

; Functions
(function_definition
  name: (identifier) @function)

(call_expression
  function: (identifier) @function.call)

; Types (if applicable)
(type_identifier) @type

; Variables
(identifier) @variable

; Literals
(number) @number
(string) @string
(comment) @comment

; Operators
[
  "+"
  "-"
  "*"
  "/"
  "="
  "=="
  "!="
  "<"
  ">"
  "<="
  ">="
  "&&"
  "||"
  "!"
] @operator

; Punctuation
[
  "("
  ")"
  "{"
  "}"
  "["
  "]"
  ","
  ";"
] @punctuation.bracket

; Fields (for method calls, member access)
(member_expression
  property: (identifier) @property)

; Parameters
(parameter
  name: (identifier) @parameter)
```

##### File: `queries/locals.scm`

**Implement scope and definition tracking:**

```scheme
; Scopes
(block) @local.scope
(function_definition) @local.scope

; Definitions
(function_definition
  name: (identifier) @local.definition)

(parameter
  name: (identifier) @local.definition)

; References
(identifier) @local.reference
```

##### File: `queries/tags.scm`

**Implement code navigation:**

```scheme
; Function definitions
(function_definition
  name: (identifier) @name) @definition.function

; Class definitions
(class_definition
  name: (identifier) @name) @definition.class

; Method definitions (if applicable)
(method_definition
  name: (identifier) @name) @definition.method
```

### 7. Create Verification Tests

**File:** `test/corpus/05_semantization.txt`

```
================================================================================
Semantization: Field visibility
================================================================================

func test(a, b) {
  return a + b;
}

--------------------------------------------------------------------------------

(source_file
  (function_definition
    name: (identifier)
    parameters: (parameter_list
      (parameter name: (identifier))
      (parameter name: (identifier)))
    body: (block
      (return_statement
        (binary_expression
          left: (identifier)
          operator: "+"
          right: (identifier))))))

================================================================================
Semantization: No wrapper nodes visible
================================================================================

return 42;

--------------------------------------------------------------------------------

(source_file
  (return_statement
    (number)))

; NOTE: _statement wrapper should NOT appear in output

================================================================================
Semantization: Supertype grouping
================================================================================

if (x) { y; }

--------------------------------------------------------------------------------

(source_file
  (if_statement
    condition: (identifier)
    consequence: (block
      (expression_statement (identifier)))))

; NOTE: if_statement is member of _statement supertype
```

## Verification Checklist

### AST Refinement Verification

1. **Field Test:**
   ```bash
   tree-sitter parse test_file.ext | grep "name:"
   tree-sitter parse test_file.ext | grep "body:"
   ```
   → All expected fields visible

2. **Flattening Test:**
   ```bash
   tree-sitter parse test_file.ext | grep "_statement"
   ```
   → Should return NO results (hidden rules invisible)

3. **Node Count:**
   - Parse a test file
   - Count nodes manually
   - Should be minimal (no unnecessary wrappers)

### Semantization Verification

4. **Highlight Test:**
   ```bash
   tree-sitter highlight test_file.ext
   ```
   → Tokens correctly colored

5. **Query Test:**
   ```bash
   tree-sitter query queries/highlights.scm test_file.ext
   tree-sitter query queries/locals.scm test_file.ext
   tree-sitter query queries/tags.scm test_file.ext
   ```
   → All queries return expected matches

6. **Supertype Test:**
   - Verify `src/node-types.json` contains supertype information
   - Check that all expression nodes are marked as subtypes of `_expression`

## Deliverables

1. **Updated `grammar.js`** with:
   - Complete field coverage
   - Verified hidden rules
   - Supertypes array
   - Inline array (if applicable)

2. **Complete `queries/` directory:**
   - `highlights.scm`
   - `locals.scm`
   - `tags.scm`

3. **`test/corpus/05_semantization.txt`**

4. **`AST_REFINEMENT.md`** documenting:
   - Field audit results
   - Hidden rules verification
   - Inline decisions

5. **`SUPERTYPE_VALIDATION.md`** documenting:
   - Complete mapping table
   - Rationale for each supertype member
   - Cross-reference to general_rules_de.md section + table

6. **`SEMANTIZATION_VERIFICATION.md`** with output from all verification commands

## Success Criteria

From PAGS Protocol Phase VII Semantization Verification Checklist:

1. ✓ Field test → roles visible
2. ✓ Flattening test → no wrappers visible
3. ✓ Highlight test → tokens marked
4. ✓ Scope test → correct renaming possible

**All four criteria must pass.**