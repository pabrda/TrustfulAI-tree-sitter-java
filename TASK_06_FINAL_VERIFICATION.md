# Task 6: Final Verification (PAGS ABGS Step 9)

## Objective
Implement **ABGS Step 9: Test Corpus Creation (Verification)** and execute the **Final Verification Checklist** from the Protocol for Algorithmic Grammar Synthesis.

## Prerequisites
- Tasks 1-5 completed
- Grammar fully functional with zero conflicts

## Required Files (Must Be in Current Directory)

Before starting this task, run:

```bash
ls PAGS_PROTOCOL.md general_rules_de.md CONTEXT.md *_history.json
```

**All 4 files must exist. If not, complete Prerequisites phase first.**

### File Usage in This Task

- **PAGS_PROTOCOL.md:** Referenced for Final Verification Checklist + ABGS Step 9
- **<language>_history.json:** For version-specific test cases

## Context Documents
- Read: `PAGS_PROTOCOL.md` (Final Verification Checklist + ABGS Step 9)
- Use: `<language>_history.json` (Version-Specific Features section)

## Implementation Requirements

### 1. Create Comprehensive Test Corpus

**File:** `test/corpus/06_comprehensive.txt`

**Per PAGS ABGS Step 9:**
> "Version-specific examples; edge-case examples that trigger ambiguity"

**Required test categories (minimum 10 tests per category):**

#### Category A: Atoms (Complete Coverage)
- All identifier variants (simple, with underscore, with numbers, edge cases)
- All number formats (decimal, hex, binary, float, scientific)
- All string formats (simple, with escapes, multiline if applicable)
- Comments (single-line, multi-line, edge cases)

#### Category B: Structural Elements
- Functions (empty, simple, complex, nested)
- Classes (if applicable: empty, with methods, with inheritance)
- Control flow (if, if-else, if-elif-else, nested conditionals)
- Loops (while, for variants, nested loops)

#### Category C: Expression Complexity
- Deep nesting: `a + b * c / d - e`
- Parenthesization: `((a + b) * (c + d))`
- Mixed operators: `a && b || c == d`
- Unary chains: `!!x`, `--n`
- Call chains: `func1(func2(func3()))`

#### Category D: Edge Cases
- Minimal programs (single statement, empty function)
- Maximum nesting depth (deeply nested blocks)
- Operator combinations that previously caused conflicts
- Keyword as-close-as-possible-to-identifier edge cases

#### Category E: Version-Specific Tests

**Extract from `<language>_history.json`.**

For each version-specific feature:

```
================================================================================
Version [X.Y]: Feature [FEATURE_NAME]
================================================================================

[Example code using version-specific syntax]

--------------------------------------------------------------------------------

[Expected AST]

; VERSION: >= X.Y
```

**Example:**
```
================================================================================
Version 3.10: match/case pattern matching
================================================================================

match value:
    case 1:
        return "one"
    case _:
        return "other"

--------------------------------------------------------------------------------

(source_file
  (match_statement
    (identifier)
    (case_clause
      pattern: (literal_pattern (number))
      (block (return_statement (string))))
    (case_clause
      pattern: (wildcard_pattern)
      (block (return_statement (string))))))

; VERSION: >= 3.10
```

### 2. Create Real-World Sample Programs

**Directory:** `test/samples/`

**Create minimum 3 sample programs:**

#### `test/samples/01_minimal.ext`
**Content:** Smallest valid program

```
// Example (adjust for your language)
func main() {
}
```

#### `test/samples/02_typical.ext`
**Content:** Typical program using common features (50-100 lines)

```
// Example structure:
// - Import statements
// - Class or function definitions
// - Control flow
// - Expressions
// - Comments
```

#### `test/samples/03_complex.ext`
**Content:** Complex program with deep nesting, multiple functions/classes (200+ lines)

```
// Example structure:
// - Multiple classes with inheritance
// - Nested functions
// - Complex expressions
// - All statement types
// - Edge cases
```

**Verify all samples parse:**
```bash
tree-sitter parse test/samples/01_minimal.ext
tree-sitter parse test/samples/02_typical.ext
tree-sitter parse test/samples/03_complex.ext
```

### 3. Performance Test

**File:** `test/samples/04_performance.ext`

**Generate large file (10,000+ lines):**
- Repetitive but valid code
- Tests parser performance and incremental parsing

```bash
# Example generator script
for i in {1..1000}; do
  echo "func test_$i() { return $i; }" >> test/samples/04_performance.ext
done
```

**Verify:**
```bash
time tree-sitter parse test/samples/04_performance.ext
```

**Document parse time in `PERFORMANCE_METRICS.md`:**
- Initial parse time
- Memory usage (if measurable)
- Re-parse time (incremental parsing)

### 4. Execute Final Verification Checklist

**From PAGS Protocol Final Verification Checklist:**

#### Checklist Item 1: Generation Clean
```bash
tree-sitter generate
```
**Expected:** No warnings, no conflicts

**Document result:** ✓ or ✗

#### Checklist Item 2: Atoms Robust
```bash
tree-sitter test -f 'Identifier'
tree-sitter test -f 'Number'
tree-sitter test -f 'String'
tree-sitter test -f 'Comment'
```
**Expected:** All tests pass

**Document result:** ✓ or ✗

#### Checklist Item 3: Precedence Correct
```bash
echo "1 + 2 * 3" | tree-sitter parse
```
**Expected:** AST shows `(1 + (2 * 3))` structure

**Verify manually:** ✓ or ✗

#### Checklist Item 4: AST Clean
```bash
tree-sitter parse test/samples/02_typical.ext
```
**Expected:** No `_statement`, `_expression` wrapper nodes visible in output

**Document result:** ✓ or ✗

#### Checklist Item 5: Corpus Complete
```bash
tree-sitter test
```
**Expected:** All tests pass (100% green)

**Document result:** X/Y tests passed

#### Checklist Item 6: Lexer Determinism
**Manual review of grammar.js:**
- [ ] Keywords use `token()` or proper precedence
- [ ] Ambiguous tokens have explicit priority
- [ ] Immediate tokens prevent whitespace issues where needed

#### Checklist Item 7: Error Recovery
**Test with intentionally broken code:**
```bash
echo "func test( { }" | tree-sitter parse
```
**Expected:** Parser recovers gracefully, produces partial tree with ERROR nodes

**Document result:** ✓ or ✗

#### Checklist Item 8: Incremental Parsing Stability
**Test with edits:**
```bash
tree-sitter parse test/samples/02_typical.ext
# Make small edit to file
tree-sitter parse test/samples/02_typical.ext
```
**Expected:** Most of tree remains unchanged (minimal node churn)

**Document result:** ✓ or ✗

#### Checklist Item 9: AST API Stability
**Check `src/node-types.json`:**
- [ ] Node names are consistent
- [ ] Field names match documentation
- [ ] No breaking changes from previous iterations

#### Checklist Item 10: Query Layer
**Test all query files:**
```bash
tree-sitter query queries/highlights.scm test/samples/02_typical.ext
tree-sitter query queries/locals.scm test/samples/02_typical.ext
tree-sitter query queries/tags.scm test/samples/02_typical.ext
```
**Expected:** No errors, meaningful captures

**Document result:** ✓ or ✗

### 5. Execute Supplementary Protocol Checks

**From PAGS Supplementary Protocols:**

#### Recursion Normalization
**Manual review:** Check grammar.js for left recursion.
- [ ] No left-recursive rules exist (all converted to iteration)

#### Performance Awareness
**Manual review:**
- [ ] No massive `choice` chains (> 50 alternatives)
- [ ] Deep nesting minimized where possible

#### Grammar Evolvability
**Documentation check:**
- [ ] Grammar supports version-specific features via optional rules
- [ ] Deprecated syntax handling documented

## Deliverables

1. **`test/corpus/06_comprehensive.txt`** with minimum 50 test cases across all categories

2. **`test/samples/` directory** with:
   - `01_minimal.ext`
   - `02_typical.ext`
   - `03_complex.ext`
   - `04_performance.ext`

3. **`FINAL_VERIFICATION_REPORT.md`** with:
   - Complete checklist results (✓ or ✗ for each item)
   - Performance metrics
   - Any known limitations or edge cases
   - Version compatibility matrix

4. **`GRAMMAR_DOCUMENTATION.md`** with:
   - Grammar overview
   - Supported features
   - Version compatibility
   - Known limitations
   - Usage examples

5. **Updated `README.md`** for the grammar repository

## Success Criteria

**From PAGS Protocol Final Verification Checklist:**

All 10 checklist items MUST be ✓ (passing):

1. ✓ `tree-sitter generate` produces no warnings
2. ✓ All atoms are robust against edge cases
3. ✓ Precedence handles `1 + 2 * 3` correctly
4. ✓ AST is clean with no unnecessary wrapper nodes
5. ✓ Every major syntax feature has a corresponding corpus test
6. ✓ Lexer determinism protocol applied
7. ✓ Error recovery functional
8. ✓ Incremental stability verified
9. ✓ AST API stable
10. ✓ Query layer optimized and conflict-free

**CRITICAL:** `tree-sitter test` must show 100% pass rate.

## Final Compilation

**Execute ABGS Step 10:**
```bash
tree-sitter generate
```

**Verify artifacts generated:**
- `src/parser.c`
- `src/node-types.json`
- `src/grammar.json`

**Grammar is now complete and ready for use.**