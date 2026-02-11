# Task 7: TSGS-Driven Refinement (Optional - Advanced)

## Objective
Apply TSGS (Tree-sitter Grammar Synthesis) techniques to refine grammar based on real-world corpus analysis and oracle feedback.

## Prerequisites
- Tasks 1-6 completed
- Grammar functional and passing all tests
- Access to large corpus of real-world code
- Reference compiler/interpreter available as oracle

## When to Use This Task
- Grammar works but has edge cases in real code
- You have access to 10,000+ lines of target language code
- You want to optimize precedence/conflict resolution based on empirical data

## Implementation

### 1. Corpus Collection
**Gather diverse code samples:**
- Popular open-source projects
- Language standard library
- Test suites from reference implementation

**Minimum corpus size:** 50,000 lines of code

### 2. Decomposition Forest Analysis
**Identify problematic constructs:**
```bash
# Parse entire corpus
for file in corpus/*.ext; do
  tree-sitter parse "$file" 2>&1 | tee -a parse_errors.log
done

# Extract ERROR nodes
grep "ERROR" parse_errors.log > problematic_constructs.txt
```

**For each ERROR:**
1. Extract minimal failing snippet
2. Apply decomposition operators (from TSGS Phase 1)
3. Find smallest substring that parses incorrectly

### 3. Oracle Consultation
**For ambiguous parses:**
```bash
# Example: Compare tree-sitter parse vs reference compiler
echo "$snippet" > test.ext
tree-sitter parse test.ext > ts_parse.txt
<reference_compiler> --dump-ast test.ext > ref_parse.txt
diff ts_parse.txt ref_parse.txt
```

**If trees differ:**
- Document discrepancy in `TSGS_REFINEMENTS.md`
- Identify grammar rule causing difference
- Adjust precedence/conflict resolution per TSGS Phase 4

### 4. Distributional Matrix (Advanced)
**If you have many similar ambiguities:**
1. Build substring/context distribution matrix (TSGS Phase 3)
2. Identify swappable bubbles
3. Merge productions to reduce ambiguity

**This is highly advanced - use only if standard PAGS methods insufficient.**

## Deliverables
- `TSGS_REFINEMENTS.md` documenting corpus-driven improvements
- Updated grammar.js
- New test cases from corpus failures

## Success Criteria
- Parse success rate on corpus: >99%
- All corpus-derived test cases pass