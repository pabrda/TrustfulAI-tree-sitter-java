# Task 1: Lexical Foundation (PAGS Phase I)

## Objective
Implement **Phase I: Deconstruction & Atomization** and **ABGS Steps 1-2** from the Protocol for Algorithmic Grammar Synthesis.

## Required Files (Must Be in Current Directory)

Before starting this task, run:

```bash
ls PAGS_PROTOCOL.md general_rules_de.md CONTEXT.md *_history.json
```

**All 4 files must exist. If not, complete Prerequisites phase first.**

### File Usage in This Task

- **PAGS_PROTOCOL.md:** Referenced for Phase I definitions
- **CONTEXT.md:** Contains grammar specification location
- **<language>_history.json:** Version-specific feature data

## Context Documents
- Read and strictly follow: `PAGS_PROTOCOL.md` (Phase I section)
- Use: `CONTEXT.md` for grammar specification location
- Use: `<language>_history.json` for version-specific keywords

## Implementation Requirements

### 1. Create `grammar.js` with Basic Structure

```javascript
module.exports = grammar({
  name: '[LANGUAGE_NAME]',

  // STEP 1: Define extras array
  // Logic: Global ignore tokens (whitespace, comments)
  extras: $ => [
    /\s/,
    $.comment,
  ],

  rules: {
    // Temporary root for atom testing
    source_file: $ => repeat($._definition),

    _definition: $ => choice(
      $.identifier,
      $.number,
      $.string
    ),

    // ATOM DEFINITIONS GO HERE (next step)
  }
});
```

### 2. Implement Step 1: Lexical Atomization

**Input Sources:**
1. **`specs/lexical.yaml`** (generated in Task 0)
2. **`SYNTHESIS_PLAN.md`** → Lexical Atoms section
 
**Action:** Implement atoms **directly from normalized specs**, not from manual parsing.

According to PAGS Protocol Phase I Logic:
> "Tree-sitter uses a separate lexer. If your regex is too loose, your parser will fail on ambiguities."

#### identifier

**Source:** `specs/lexical.yaml` → `identifiers.pattern`
**Implementation:** Use pattern from YAML directly:

```javascript
// From specs/lexical.yaml
identifier: $ => /[a-zA-Z_][a-zA-Z0-9_]*/,  // Pattern from lexical.yaml
```

**Document in EXTRACTION_LOG.md:**
- Source: [Grammar spec section/line number]
- Original EBNF: [paste original rule]
- Converted regex: [your implementation]
**Traceability:** Pattern already documented in lexical.yaml with spec_reference.

#### number

**Extraction Task:** Locate "Numeric Literals" section in grammar spec.
**Pattern Recognition:** Find ALL number format rules (decimal, hex, binary, float, scientific)

**MUST use `token(choice(...))` per PAGS protocol requirement**

```javascript
// Example structure (adjust based on extracted patterns)
number: $ => token(choice(
  /\d+/,                    // Decimal (from spec)
  /0x[0-9a-fA-F]+/,         // Hexadecimal (from spec)
  /0b[01]+/,                // Binary (from spec, if present)
  /\d+\.\d+/,               // Float (from spec)
  /\d+(\.\d+)?[eE][+-]?\d+/ // Scientific (from spec)
)),
```

**Document in EXTRACTION_LOG.md:**
- Each number format with source reference
- Any version-specific formats (cross-reference with history.json)

#### string

**Extraction Task:** Locate "String Literals" section in grammar spec.
**Pattern Recognition:** Identify:
- Quote styles (single, double, triple)
- Escape sequences (`\n`, `\t`, `\"`, `\\`, etc.)
- Multiline support

```javascript
// Standard pattern for double-quoted strings with escapes
string: $ => /"([^"\\]|\\(.|
))*"/,

// Adjust based on spec:
// - Single quotes: /'([^'\\]|\\(.|
))*'/
// - Triple quotes: /"""[\s\S]*?"""/
// - Raw strings: Language-specific
```

**Document in EXTRACTION_LOG.md:**
- All string literal variants from spec
- Escape sequence handling
- Version-specific string types (f-strings, raw strings, etc.)

#### comment

**Extraction Task:** Locate "Comments" section in grammar spec.
**Pattern Recognition:** Identify:
- Single-line delimiter (e.g., `//`, `#`)
- Multi-line delimiters (e.g., `/* */`, `"""`)

```javascript
// Single-line comment example
comment: $ => token(/\/\/.*/),

// Multi-line comment example (if applicable)
// comment: $ => token(/\/\*[^*]*\*+([^/*][^*]*\*+)*\//),
```

**Document in EXTRACTION_LOG.md:**
- Comment syntax with spec reference
- Any version-specific comment styles

### 3. Implement Step 2: Keyword Classification

**Critical:** Cross-reference with `<language>_history.json` for version-specific keywords.

**Extraction Process:**

1. **Locate keyword list** in grammar specification
2. **Parse history.json** for version boundaries:

```javascript
// Example: Reading from history.json
// {
//   "version_boundary": "3.10",
//   "change_type": "Introduction",
//   "feature": "match/case keywords",
//   "tree_sitter_nodes": {
//     "added_nodes": ["match_statement", "case_clause"]
//   }
// }
```

3. **Categorize keywords by version:**

```javascript
// Keywords present in ALL versions
const CORE_KEYWORDS = ["if", "else", "for", "while", "return"];

// Keywords added in specific versions (from history.json)
const KEYWORDS_V3_10 = ["match", "case"];  // Added in 3.10

// Implement in grammar:
rules: {
  // Core keywords (all versions)
  'if': $ => 'if',
  'else': $ => 'else',
  'for': $ => 'for',
  'while': $ => 'while',
  'return': $ => 'return',

  // Version-specific keywords
  'match': $ => 'match',  // Note: Only valid >= 3.10
  'case': $ => 'case',    // Note: Only valid >= 3.10

  // [ADD ALL KEYWORDS FROM GRAMMAR SPEC]
}
```

4. **Apply Lexer Determinism Protocol** from PAGS Supplementary Protocols:
   - Keywords must override identifiers
   - If version conflicts exist (keyword becomes identifier in older versions), document in code comment

**Document in EXTRACTION_LOG.md:**
- Complete keyword list with spec source
- Version matrix showing which keywords valid in which versions
- Any keyword/identifier conflicts

### 4. Create Test Corpus

**File:** `test/corpus/01_atoms.txt`

**Structure per PAGS verification requirement:**

```
================================================================================
Identifier: Simple
================================================================================

myVariable

--------------------------------------------------------------------------------

(source_file
  (identifier))

================================================================================
Identifier: With underscore
================================================================================

_private_var

--------------------------------------------------------------------------------

(source_file
  (identifier))

================================================================================
Identifier: With numbers
================================================================================

var123

--------------------------------------------------------------------------------

(source_file
  (identifier))

================================================================================
Identifier: Edge case - single underscore
================================================================================

_

--------------------------------------------------------------------------------

(source_file
  (identifier))

================================================================================
Number: Decimal
================================================================================

42

--------------------------------------------------------------------------------

(source_file
  (number))

================================================================================
Number: Hexadecimal
================================================================================

0xFF

--------------------------------------------------------------------------------

(source_file
  (number))

================================================================================
Number: Binary (if supported)
================================================================================

0b1010

--------------------------------------------------------------------------------

(source_file
  (number))

================================================================================
Number: Float
================================================================================

3.14

--------------------------------------------------------------------------------

(source_file
  (number))

================================================================================
Number: Scientific notation
================================================================================

1.5e10

--------------------------------------------------------------------------------

(source_file
  (number))

================================================================================
String: Simple
================================================================================

"hello world"

--------------------------------------------------------------------------------

(source_file
  (string))

================================================================================
String: With escapes
================================================================================

"hello\nworld\t\"quoted\""

--------------------------------------------------------------------------------

(source_file
  (string))

================================================================================
String: Empty
================================================================================

""

--------------------------------------------------------------------------------

(source_file
  (string))

================================================================================
Comment: Single line
================================================================================

// this is a comment

--------------------------------------------------------------------------------

(source_file)

================================================================================
Comment: At end of line
================================================================================

myVar // inline comment

--------------------------------------------------------------------------------

(source_file
  (identifier))

================================================================================
Keyword: if
================================================================================

if

--------------------------------------------------------------------------------

(source_file)

================================================================================
Keyword: return
================================================================================

return

--------------------------------------------------------------------------------

(source_file)
```

**Create minimum 20 test cases total:**
- 5 identifier variants
- 5 number formats
- 5 string variants
- 3 comment scenarios
- 2 keyword examples

**Version-Specific Tests:**

If `<language>_history.json` documents version-specific features, add:

```
================================================================================
Version 3.10: match keyword
================================================================================

match

--------------------------------------------------------------------------------

(source_file)

; VERSION: >= 3.10
```

## Verification Checklist

Execute the following commands and ensure all pass:

1. **Generation Clean:**
   ```bash
   tree-sitter generate
   ```
   Expected: Must complete without errors

2. **Tests Pass:**
   ```bash
   tree-sitter test
   ```
   Expected: All tests in `01_atoms.txt` must pass (green)

3. **Manual Parse Test:**
   ```bash
   echo "testIdentifier" | tree-sitter parse
   ```
   Expected: Should produce valid parse tree

4. **Extraction Verification:**
   - [ ] All keywords from grammar spec present
   - [ ] All number formats from grammar spec implemented
   - [ ] String escape sequences match spec

5. **Version Matrix Check:**
   - [ ] `<language>_history.json` keywords cross-referenced
   - [ ] Version-specific atoms documented in tests

## Deliverables

1. **`grammar.js`** with complete atom definitions

2. **`test/corpus/01_atoms.txt`** with minimum 20 test cases (5 per atom type)

3. **`ATOMS_VERIFICATION.md`** documenting:
   - Which regex patterns were chosen and why
   - Any edge cases handled
   - Test results output

4. **`GRAMMAR_EXTRACTION_LOG.md`** documenting:
   - URL/path of source grammar specification
   - Each extracted element with source location reference (section/line number)
   - EBNF/original pattern → tree-sitter conversion
   - Any ambiguities encountered
   - Version matrix for keywords (which versions have which keywords)

## Success Criteria

From PAGS Protocol:
> "Create `test/corpus/atoms.txt`. Test each atom individually. If these fail, everything else is meaningless."

**ALL atom tests MUST be green before proceeding to Task 2.**

**Verification command:**
```bash
tree-sitter test
# Expected output: All tests passed
```
