# Jules Workflow Orchestration for Tree-sitter Grammar Generation

## Overview
This document orchestrates the execution of 6 sequential tasks for generating a complete Tree-sitter grammar according to the Protocol for Algorithmic Grammar Synthesis (PAGS).

## Prerequisites - File Checklist

### Step 1: Create Project Directory

```bash
mkdir tree-sitter-<language>
cd tree-sitter-<language>
```

### Step 2: Place Required Documents

**You must have these 4 files in the project directory:**

#### File 1: `PAGS_PROTOCOL.md`
**Source:** Your 550-line protocol document
**Verification:**
```bash
ls PAGS_PROTOCOL.md
```

#### File 2: `general_rules_de.md`
**Source:** USG specification (you provided as attachment)
**Verification:**
```bash
ls general_rules_de.md
```

#### File 3: `<language>_history.json`
**Source:** You must create this file
**Template:**
```json
{
  "language": "LANGUAGE_NAME",
  "modern_version": "X.Y",
  "reverse_timeline": [
    {
      "version_boundary": "X.Y",
      "change_type": "Introduction",
      "feature": "Feature Name",
      "tree_sitter_nodes": {
        "added_nodes": [],
        "removed_nodes": [],
        "semantic_shift_nodes": []
      },
      "defolding_impact": {
        "description": "",
        "legacy_behavior": "",
        "edge_implications": ""
      }
    }
  ]
}
```

**Create file:**
```bash
nano python_history.json  # Replace 'python' with your language
# Paste template and fill in
```

**Validate:**
```bash
python3 -m json.tool python_history.json  # Must not error
```

#### File 4: `CONTEXT.md`
**Source:** CONTEXT.md template (provided separately)
**Create file:**
```bash
nano CONTEXT.md
# Paste template
# Fill in: Language Name, Grammar Spec URL/Path
```

#### File 5 (Optional): Official Grammar File
**If grammar spec is not available via URL:**
```bash
# Download grammar file to project directory
wget <URL> -O grammar_spec.txt
# Update CONTEXT.md with: Official Grammar Specification → Location: ./grammar_spec.txt
```

### Step 3: Install tree-sitter CLI

```bash
npm install -g tree-sitter-cli
tree-sitter --version  # Verify
```

### Step 4: Initialize Grammar Project

```bash
tree-sitter init
```

**This creates:**
```
tree-sitter-<language>/
├── grammar.js        (empty template)
├── package.json
├── src/
├── bindings/
└── [your 4 required files from Step 2]
```

### Step 5: Verification

**Run this checklist:**

```bash
#!/bin/bash
# verify.sh

echo "Checking files..."

# Required documents
[ -f "PAGS_PROTOCOL.md" ] && echo "✅ PAGS_PROTOCOL.md" || { echo "❌ PAGS_PROTOCOL.md MISSING"; exit 1; }
[ -f "general_rules_de.md" ] && echo "✅ general_rules_de.md" || { echo "❌ general_rules_de.md MISSING"; exit 1; }
[ -f "CONTEXT.md" ] && echo "✅ CONTEXT.md" || { echo "❌ CONTEXT.md MISSING"; exit 1; }

# Language-specific
LANG_HISTORY=$(ls *_history.json 2>/dev/null | head -n1)
if [ -n "$LANG_HISTORY" ]; then
    echo "✅ $LANG_HISTORY"
    python3 -m json.tool "$LANG_HISTORY" > /dev/null 2>&1 && echo "  └─ Valid JSON ✅" || { echo "  └─ Invalid JSON ❌"; exit 1; }
else
    echo "❌ *_history.json MISSING"
    exit 1
fi

# Tree-sitter
command -v tree-sitter > /dev/null 2>&1 && echo "✅ tree-sitter CLI" || { echo "❌ tree-sitter CLI NOT INSTALLED"; exit 1; }

# Grammar initialized
[ -f "grammar.js" ] && echo "✅ grammar.js initialized" || { echo "❌ Run 'tree-sitter init' first"; exit 1; }

echo ""
echo "✅ All prerequisites met. Ready for Task 1."
```

**Usage:**
```bash
chmod +x verify.sh
./verify.sh
```

**If output shows any ❌, fix before proceeding.**

### Directory Structure After Setup

```
tree-sitter-<language>/
├── PAGS_PROTOCOL.md          # Your 550-line protocol
├── general_rules_de.md        # USG specification
├── CONTEXT.md                 # Filled template
├── <language>_history.json    # Version history
├── grammar.js                 # From tree-sitter init
├── package.json               # From tree-sitter init
├── src/                       # From tree-sitter init
├── bindings/                  # From tree-sitter init
└── [optional] grammar_spec.txt  # If downloaded locally
```


## Task Execution Sequence

### Task 0: Planning Phase 

**Input:** `TASK_00_PLANNING_PHASE.md`
**Prerequisites:** All setup files in place
**Jules command:**
```
Execute Task 0 Planning Phase according to TASK_00_PLANNING_PHASE.md.

Extract and normalize all language specifications.
If existing grammar exists at <URL>, audit for reuse potential.
Generate complete specs/ directory with YAML specifications.
Generate SYNTHESIS_PLAN.md.

This phase must complete before any grammar.js code is written.
```

**Human verification after Task 0:**
```bash
yamllint specs/*.yaml
cat SYNTHESIS_PLAN.md
# Review synthesis plan for completeness
```
**Requirement:** SYNTHESIS_PLAN.md must be approved before Task 1.



### Task 1: Lexical Foundation
**Input:** `TASK_01_LEXICAL_FOUNDATION.md`
**Jules command:**
```
Implement Task 1 according to TASK_01_LEXICAL_FOUNDATION.md.

Input documents:
1. CONTEXT.md (contains references only)
2. Official grammar specification at [URL from CONTEXT.md]
3. <language>_history.json for version-specific keywords

Extract all lexical elements from the official grammar specification.
Cross-reference keywords with history.json for version-specific additions/removals.
Document ALL extractions in GRAMMAR_EXTRACTION_LOG.md.

Strict adherence to PAGS Protocol Phase I required.
```

**Human verification after Task 1:**
```bash
tree-sitter generate
tree-sitter test
```
**Requirement:** ALL atom tests must pass before proceeding.

---

### Task 2: Structural Skeleton
**Input:** `TASK_02_STRUCTURAL_SKELETON.md`
**Prerequisites:** Task 1 merged and verified
**Jules command:**
```
Implement Task 2 according to TASK_02_STRUCTURAL_SKELETON.md.

Build upon grammar.js from Task 1.
Use specifications from CONTEXT.md.
Map all tree-sitter nodes to USG NodeKinds from general_rules_de.md Section III.A.
Document all mappings in USG_MAPPING.md.

Strict adherence to PAGS Protocol Phase II required.
```

**Human verification after Task 2:**
```bash
tree-sitter generate
tree-sitter test
tree-sitter parse test_file.ext
```
**Requirement:** All structure tests must pass, fields must be visible.

---

### Task 3: Expression Engine
**Input:** `TASK_03_EXPRESSION_ENGINE.md`
**Prerequisites:** Task 2 merged and verified
**Jules command:**
```
Implement Task 3 according to TASK_03_EXPRESSION_ENGINE.md.

Build upon grammar.js from Task 2.
Extract operator precedence table from language specification.
Implement all binary and unary operators with correct precedence and associativity.

Strict adherence to PAGS Protocol Phase III (CRITICAL) required.
```

**Human verification after Task 3:**
```bash
tree-sitter generate  # May have conflicts - expected
tree-sitter test
echo "1 + 2 * 3" | tree-sitter parse
```
**Requirement:** Expression tests pass, precedence verified.
**Note:** Conflicts expected, will be resolved in Task 4.

---

### Task 4: Conflict Resolution
**Input:** `TASK_04_CONFLICT_RESOLUTION.md`
**Prerequisites:** Task 3 merged
**Jules command:**
```
Implement Task 4 according to TASK_04_CONFLICT_RESOLUTION.md.

Build upon grammar.js from Task 3.
Apply PAGS Protocol Phase IV Conflict Resolution Algorithm:
- Step 1: Associativity Check
- Step 2: Binding Strength
- Step 3: GLR Conflicts Array
- Step 4: Dynamic Precedence

Reference <language>_history.json for context-dependent ambiguities.

CRITICAL REQUIREMENT: tree-sitter generate must produce ZERO unresolved conflicts.
Document all resolutions in CONFLICTS_LOG.md.
```

**Human verification after Task 4:**
```bash
tree-sitter generate  # MUST show: No unresolved conflicts
tree-sitter test
```
**Requirement:** ZERO conflicts, ZERO warnings.
**This is a blocking requirement - do not proceed until satisfied.**

---

### Task 5: Semantization
**Input:** `TASK_05_SEMANTIZATION.md`
**Prerequisites:** Task 4 merged with zero conflicts
**Jules command:**
```
Implement Task 5 according to TASK_05_SEMANTIZATION.md.

Build upon grammar.js from Task 4.
Implement PAGS Protocol Phase V (AST Refinement) and Phase VII (Semantization).

Use general_rules_de.md Section III.A for supertype taxonomy.
Create complete queries/ directory with highlights.scm, locals.scm, tags.scm.

All supertypes MUST match categories from general_rules_de.md exactly.
```

**Human verification after Task 5:**
```bash
tree-sitter generate
tree-sitter test
tree-sitter highlight test_file.ext
tree-sitter query queries/highlights.scm test_file.ext
tree-sitter parse test_file.ext | grep "_statement"  # Should be empty
```
**Requirement:** All semantization tests pass, queries functional.

---

### Task 6: Final Verification
**Input:** `TASK_06_FINAL_VERIFICATION.md`
**Prerequisites:** Task 5 merged
**Jules command:**
```
Implement Task 6 according to TASK_06_FINAL_VERIFICATION.md.

Create comprehensive test corpus per PAGS ABGS Step 9.
Execute complete Final Verification Checklist.
Create real-world sample programs.
Document all verification results in FINAL_VERIFICATION_REPORT.md.

Use <language>_history.json for version-specific test cases.
```

**Human verification after Task 6:**
```bash
tree-sitter test  # Must be 100% pass rate
tree-sitter parse test/samples/01_minimal.ext
tree-sitter parse test/samples/02_typical.ext
tree-sitter parse test/samples/03_complex.ext
time tree-sitter parse test/samples/04_performance.ext
```
**Requirement:** All verification checklist items ✓, 100% test pass rate.

---

## Finalization

**After all 6 tasks merged:**

1. **Generate final artifacts:**
   ```bash
   tree-sitter generate
   ```

2. **Verify artifacts exist:**
   - `src/parser.c`
   - `src/node-types.json`
   - `src/grammar.json`

3. **Create release documentation:**
   - `README.md` (usage instructions)
   - `GRAMMAR_DOCUMENTATION.md` (technical reference)
   - `CHANGELOG.md` (version history)

4. **Tag release:**
   ```bash
   git tag -a v1.0.0 -m "Initial grammar release"
   git push origin v1.0.0
   ```

## Workflow Diagram

```
[Prerequisites Complete] 
         ↓
[Task 1: Atoms] → Verify → Pass
         ↓
[Task 2: Structure] → Verify → Pass
         ↓
[Task 3: Expressions] → Verify → Pass (conflicts OK)
         ↓
[Task 4: Conflicts] → Verify → Pass (ZERO conflicts required)
         ↓
[Task 5: Semantics] → Verify → Pass
         ↓
[Task 6: Final Verification] → 100% Pass
         ↓
   [Release v1.0.0]
```

## Success Metrics

**Grammar is complete when:**
- [ ] `tree-sitter generate` → 0 warnings, 0 conflicts
- [ ] `tree-sitter test` → 100% pass rate
- [ ] All PAGS Final Verification Checklist items ✓
- [ ] Real-world sample programs parse correctly
- [ ] Query files functional (highlights, locals, tags)
- [ ] Documentation complete