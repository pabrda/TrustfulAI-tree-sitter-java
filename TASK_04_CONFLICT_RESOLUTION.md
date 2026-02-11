# Task 4: Conflict Resolution (PAGS Phase IV)

## Objective
Implement **Phase IV: Conflict Resolution Algorithm** from the Protocol for Algorithmic Grammar Synthesis.

## Prerequisites
- Task 3 completed (expression engine implemented)
- Build upon grammar.js from Task 3
- May have unresolved conflicts from `tree-sitter generate`

## Required Files (Must Be in Current Directory)

Before starting this task, run:

```bash
ls PAGS_PROTOCOL.md general_rules_de.md CONTEXT.md *_history.json
```

**All 4 files must exist. If not, complete Prerequisites phase first.**

### File Usage in This Task

- **PAGS_PROTOCOL.md:** Referenced for Phase IV Conflict Resolution Algorithm
- **<language>_history.json:** For context-dependent ambiguities

## Context Documents
- Read: `PAGS_PROTOCOL.md` (Phase IV section + Conflict Types subsection)
- Use: `<language>_history.json` for version-specific ambiguities

## Implementation Requirements

### 1. Generate and Analyze Conflicts

**Execute:**
```bash
tree-sitter generate
```

**Capture output.** You will see messages like:
```
Unresolved conflict for symbol sequence:

  expression  '+'  •  expression  …

Possible interpretations:

  1:  (binary_expression  expression  '+'  •  expression)
  2:  expression  •  (binary_expression  '+'  expression)
```

**Create conflict log:** `CONFLICTS_LOG.md`

For each conflict, document:
- Symbol sequence causing conflict
- Conflict type (Shift/Reduce or Reduce/Reduce)
- Possible interpretations

### 2. Apply Conflict Resolution Protocol

**From PAGS Protocol Phase IV - Resolution Protocol:**

#### Step 1: Associativity Check

**Question:** Is this a binary operator conflict?

**If YES:**
- Verify operator has correct `prec.left()` or `prec.right()` wrapper
- Cross-reference language specification for associativity

**Example fix:**
```javascript
// BEFORE (conflict)
binary_expression: $ => seq($.expression, '+', $.expression),

// AFTER (resolved)
binary_expression: $ => prec.left(PREC.ADD, seq(
  $.expression, '+', $.expression
)),
```

#### Step 2: Binding Strength (Precedence)

**Question:** Does one rule clearly bind stronger than the other?

**Example:** Function call `func()` vs member access `obj.field`
- Call should bind tighter (or equal)
- Apply appropriate `prec(N, ...)` value

**If this resolves conflict:**
```javascript
// Adjust precedence for correct binding
call_expression: $ => prec(PREC.CALL, seq(...)),
member_expression: $ => prec(PREC.CALL, seq(...)),  // Same or higher
```

#### Step 3: The `conflicts` Array (GLR Hammer)

**From PAGS Protocol:**
> "Use only if ambiguity is *genuine* (e.g., C++ parsing `A * B` as multiplication vs pointer declaration)"

**Check `<language>_history.json`** for documented genuine conflicts.

**If genuine ambiguity exists:**
```javascript
module.exports = grammar({
  name: '[LANGUAGE_NAME]',

  // Explicit GLR branching
  conflicts: $ => [
    [$.type_cast, $.parenthesized_expression],
    // Example: Version-dependent keyword/identifier conflict
    [$.identifier, $.async_keyword],  // Pre-3.5 vs Post-3.5
    // [ADD CONFLICTS AS NEEDED]
  ],

  rules: {
    // ...
  }
});
```

**Document rationale in CONFLICTS_LOG.md for each GLR conflict.**

#### Step 4: Dynamic Precedence

**From PAGS Protocol:**
> "Use when two rules match the same string but one is only valid in a specific runtime context"

**Example scenario:** Context-dependent keywords (e.g., `await` valid only in async functions)

```javascript
// Lower dynamic precedence for context-limited variant
contextual_keyword: $ => prec.dynamic(-1, 'await'),

// Higher dynamic precedence for general identifier
identifier: $ => prec.dynamic(0, /[a-zA-Z_][a-zA-Z0-9_]*/),
```

### 3. Iterative Resolution

**Process:**
1. Apply Step 1 fixes → Run `tree-sitter generate`
2. If conflicts remain → Apply Step 2 fixes → Run `tree-sitter generate`
3. If conflicts remain → Apply Step 3 (conflicts array) → Run `tree-sitter generate`
4. If conflicts remain → Apply Step 4 (prec.dynamic) → Run `tree-sitter generate`

**Repeat until:**
```bash
tree-sitter generate
# Output: No unresolved conflicts
```

### 4. Document All Resolutions

**In `CONFLICTS_LOG.md`, for each resolved conflict:**

```markdown
## Conflict #1: Binary Expression Associativity

### Original Issue
```
Unresolved conflict for symbol sequence:
  expression  '+'  •  expression  …
```

### Analysis
- Conflict Type: Shift/Reduce
- Cause: Missing associativity specification for + operator

### Resolution Applied
- Step 1: Associativity Check
- Action: Wrapped in `prec.left(PREC.ADD, ...)`
- Rationale: Addition is left-associative (1 + 2 + 3 = (1 + 2) + 3)

### Code Change
```javascript
// grammar.js line 45
prec.left(PREC.ADD, seq($.expression, '+', $.expression))
```

### Verification
- `tree-sitter generate` → Conflict resolved
- Test: "1 + 2 + 3" parses correctly as left-associative
```

**Create similar documentation for EVERY conflict.**

### 5. Apply Supplementary Protocol: Lexer Determinism

**From PAGS Supplementary Protocols:**

If conflicts involve token boundaries:

```javascript
// Use token() with precedence
keyword: $ => token(prec(1, 'keyword')),

// Use token.immediate() to prevent whitespace
member_access: $ => seq(
  $.expression,
  token.immediate('.'),
  $.identifier
),

// Use alias() for token renaming to avoid conflicts
true_literal: $ => alias('true', $.boolean),
```

### 6. Expand Test Corpus

**File:** `test/corpus/04_conflict_edge_cases.txt`

**Create tests for EVERY resolved conflict:**

```
================================================================================
Conflict Resolution: Left-associative addition
================================================================================

1 + 2 + 3 + 4

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (binary_expression
      left: (binary_expression
        left: (binary_expression
          left: (number)
          operator: "+"
          right: (number))
        operator: "+"
        right: (number))
      operator: "+"
      right: (number))))

================================================================================
Conflict Resolution: [DESCRIBE RESOLVED CONFLICT]
================================================================================

[Test case that triggered original conflict]

--------------------------------------------------------------------------------

[Expected AST showing correct resolution]
```

**Create minimum 1 test per resolved conflict.**

**Additional edge case tests:**

```
================================================================================
Edge Case: Function call vs multiplication
================================================================================

a(b) * c

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (binary_expression
      left: (call_expression
        function: (identifier)
        arguments: (argument_list
          (argument (identifier))))
      operator: "*"
      right: (identifier))))

================================================================================
Edge Case: Member access chain
================================================================================

a.b.c.d()

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (call_expression
      function: (member_expression
        object: (member_expression
          object: (member_expression
            object: (identifier)
            property: (identifier))
          property: (identifier))
        property: (identifier))
      arguments: (argument_list))))

================================================================================
Edge Case: Nested parentheses with operators
================================================================================

((a + b) * (c - d))

--------------------------------------------------------------------------------

(source_file
  (expression_statement
    (parenthesized_expression
      (binary_expression
        left: (parenthesized_expression
          (binary_expression
            left: (identifier)
            operator: "+"
            right: (identifier)))
        operator: "*"
        right: (parenthesized_expression
          (binary_expression
            left: (identifier)
            operator: "-"
            right: (identifier)))))))
```

## Verification Checklist

1. **Zero Conflicts:**
   ```bash
   tree-sitter generate
   ```
   **Expected:** Output must show "No unresolved conflicts"

2. **Zero Warnings:**
   ```bash
   tree-sitter generate 2>&1 | grep -i warning
   ```
   **Expected:** No output

3. **All Tests Pass:**
   ```bash
   tree-sitter test
   ```
   **Expected:** All tests pass (Tasks 1-3 + new conflict tests)

4. **Manual Parse Test:**
   ```bash
   # For each conflict documented in CONFLICTS_LOG.md, test the edge case
   echo "[edge case code]" | tree-sitter parse
   ```
   **Expected:** Correct parse tree

## Deliverables

1. **Updated `grammar.js`** with all conflict resolutions

2. **`CONFLICTS_LOG.md`** with complete documentation:
   - Every conflict encountered
   - Analysis of conflict type
   - Resolution strategy applied (Step 1/2/3/4)
   - Code changes made
   - Verification results

3. **Updated `test/corpus/04_conflict_edge_cases.txt`**

4. **`RESOLUTION_SUMMARY.md`** with:
   - Total conflicts found
   - Resolution strategy distribution (how many Step 1, Step 2, Step 3, Step 4)
   - Any genuine ambiguities requiring GLR branching
   - Performance impact (if any)

## Success Criteria

From PAGS Protocol Phase IV:
> "When you run `tree-sitter generate`, you will eventually see: `Unresolved conflict for symbol sequence...` **Do not panic.** Follow this algorithm for resolution."

**CRITICAL REQUIREMENT:**
```bash
tree-sitter generate
# Must output: No unresolved conflicts
# Must output: 0 warnings
```

**You may not proceed to Task 5 until this is achieved.**

**All conflict tests must pass:**
```bash
tree-sitter test -f "Conflict Resolution"
# Expected: All green
```
