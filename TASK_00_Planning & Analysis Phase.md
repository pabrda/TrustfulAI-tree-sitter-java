# Task 0: Planning & Analysis Phase (Pre-Grammar Synthesis)

## Objective
Conduct comprehensive language analysis and generate normalized specifications before any grammar code is written. This phase implements the **Phase 0: Normalize Specification** from the unified PAGS+TSGS approach.

## Prerequisites
- Project directory created
- tree-sitter CLI installed
- All context documents present

## Required Inputs

### 1. Core Documents (Already in Workflow)
- `PAGS_PROTOCOL.md`
- `general_rules_de.md`
- `<language>_history.json`
- `CONTEXT.md`

### 2. NEW REQUIRED: Language Specification Bundle
Create directory: `specs/`

#### `specs/lexical.yaml`
**Purpose:** Normalized lexical specification extracted from official grammar.

**Structure:**
```yaml
language: <LANGUAGE_NAME>
spec_source: "<URL or file path to official grammar>"
spec_version: "<version number>"

character_classes:
  letter: "[a-zA-Z]"
  digit: "[0-9]"
  whitespace: "[ \\t\\n\\r]"
  # Add language-specific classes

identifiers:
  pattern: "<regex pattern from spec>"
  restrictions:
    - "Cannot start with digit"
    - "Cannot be keyword"
  spec_reference: "<section/line number in official grammar>"

numbers:
  formats:
    - name: "decimal"
      pattern: "\\d+"
      spec_reference: "<section>"
    - name: "hexadecimal"
      pattern: "0x[0-9a-fA-F]+"
      spec_reference: "<section>"
    - name: "float"
      pattern: "\\d+\\.\\d+"
      spec_reference: "<section>"
  # Add all number formats from spec

strings:
  formats:
    - name: "double_quoted"
      pattern: '\"([^\"\\\\]|\\\\(.|\n))*\"'
      escape_sequences: ["\\n", "\\t", "\\\"", "\\\\"]
      spec_reference: "<section>"
  # Add all string variants

comments:
  single_line:
    delimiter: "//"
    pattern: "//.*"
  multi_line:
    start: "/*"
    end: "*/"
    pattern: "/\\*[^*]*\\*+([^/*][^*]*\\*+)*/"
  # Adjust based on language

keywords:
  core: ["if", "else", "for", "while", "return"]
  # Extract complete list from spec
  contextual: ["async", "await"]  # Keywords that can also be identifiers
  version_specific:
    - keyword: "match"
      min_version: "3.10"
      source: "<language>_history.json"
```

**Creation Process:**
1. Download/access official language grammar
2. Extract lexical chapter
3. Parse all token definitions
4. Cross-reference with `<language>_history.json` for version-specific additions
5. Document every extraction with spec reference

#### `specs/operators.yaml`
**Purpose:** Complete operator precedence and associativity table.

```yaml
operators:
  - symbol: "="
    arity: "binary"
    precedence: 1
    associativity: "right"
    category: "assignment"
    spec_reference: "<section>"
    
  - symbol: "||"
    arity: "binary"
    precedence: 2
    associativity: "left"
    category: "logical_or"
    spec_reference: "<section>"
    
  - symbol: "&&"
    arity: "binary"
    precedence: 3
    associativity: "left"
    category: "logical_and"
    spec_reference: "<section>"
  
  # Complete table extracted from spec
  # Precedence values: higher number = tighter binding
  
  - symbol: "-"
    arity: "unary"
    precedence: 7
    position: "prefix"
    spec_reference: "<section>"
```

#### `specs/syntax.yaml`
**Purpose:** Structural hierarchy and EBNF rules.

```yaml
top_level:
  - "function_definition"
  - "class_definition"
  - "import_statement"
  # List all top-level constructs

ebnf_rules:
  function_definition:
    pattern: |
      'func' identifier '(' parameter_list ')' block
    fields:
      - name: "name"
        type: "identifier"
        usg_role: "definition"
      - name: "parameters"
        type: "parameter_list"
        usg_role: "signature"
      - name: "body"
        type: "block"
        usg_role: "implementation"
    spec_reference: "<section>"
  
  # Extract all major constructs from spec
```

#### `specs/versions.yaml`
**Purpose:** Enhanced version matrix (replaces/augments `<language>_history.json`).

```yaml
language: <LANGUAGE_NAME>
version_strategy: "superset"

versions:
  - version: "3.10"
    release_date: "2021-10-04"
    features:
      added:
        - feature: "match_statement"
          syntax: "match VALUE: case PATTERN: BODY"
          nodes: ["match_statement", "case_clause"]
      removed: []

  - version: "3.0"
    release_date: "2008-12-03"
    features:
      removed:
        - feature: "print_statement"
          replacement: "print() function"
          nodes: ["print_statement"]

# CRITICAL: Version constraints as TUPLES [min, max)
version_constraints:
  # Syntax-level constraints
  match_statement:
    usg_nodekind: "MatchExpression"
    valid_ranges:
      - ["3.10", null]  # Introduced 3.10, never removed
    spec_reference: "PEP 634"
    
  print_statement:
    usg_nodekind: "PrintStatement"
    valid_ranges:
      - [null, "3.0"]  # Removed in 3.0
    spec_reference: "Python 2.x"
    
  walrus_operator:
    usg_nodekind: "Assignment"
    valid_ranges:
      - ["3.8", null]
    spec_reference: "PEP 572"
  
  # Keyword/Identifier conflicts
  async_as_keyword:
    usg_nodekind: "Keyword"
    valid_ranges:
      - ["3.5", null]
    conflicts_with: ["async_as_identifier"]
    
  async_as_identifier:
    usg_nodekind: "Reference"
    valid_ranges:
      - [null, "3.5"]
    conflicts_with: ["async_as_keyword"]

conflict_matrix:
  - construct_a: "async_keyword"
    construct_b: "identifier_async"
    resolution: "GLR with version filtering"
    context: "Pre-3.5: async is identifier; Post-3.5: async is keyword"
```

**Extraction Process:**
1. Parse `<language>_history.json` for all `added_nodes` and `removed_nodes`
2. For each node, determine `min_version` (when added) and `max_version` (when removed)
3. Cross-reference with official language specification for confirmation
4. Document all version-dependent syntax in `version_constraints` section


#### `specs/usg_constraints.yaml`
**Purpose:** Map language constructs to USG requirements.

```yaml
# Generated semi-automatically from general_rules_de.md Section III.A

node_mappings:
  function_definition:
    usg_nodekind: "FunctionDef"
    category: "II. Funktionen & Logik"
    required_metadata:
      - name: "throws"
        type: "array<string>"
        derivable: false
      - name: "visibility"
        type: "string"
        derivable: false
    required_fields:
      - "name"
      - "body"
    optional_fields:
      - "parameters"
      - "return_type"
    traversal_strategy: "CreateAndContain"
    
  if_statement:
    usg_nodekind: "IfStatement"
    category: "IV. Kontrollfluss"
    required_metadata:
      - name: "condition"
        type: "string"
        derivable: false
      - name: "is_ternary"
        type: "bool"
        derivable: false
    required_fields:
      - "condition"
      - "consequence"
    optional_fields:
      - "alternative"
    traversal_strategy: "CreateAndContain"
    
  # Complete mapping for all language constructs
```

### 3. NEW REQUIRED: Existing Grammar Analysis (If Extending)

#### `analysis/existing_grammar_audit.md`
**Purpose:** If tree-sitter grammar already exists, audit it before modification.

```markdown
# Existing Grammar Audit: tree-sitter-<language>

## Source
- Repository: <URL>
- Commit: <hash>
- Date: <date>

## Structure Analysis

### Node Inventory
| Node Name | Category | USG Compatible? | Issues |
|-----------|----------|-----------------|--------|
| function_definition | Definition | ✓ | Missing 'throws' field |
| if_statement | Statement | ✓ | Missing 'alternative' field name |
| expression | Expression | ✗ | Uses nested precedence rules (outdated) |

### Field Coverage
| Construct | Required Fields (USG) | Implemented | Missing |
|-----------|----------------------|-------------|---------|
| function_definition | name, parameters, body | name, body | parameters |

### Conflict Analysis
- Unresolved conflicts: 3
- GLR conflicts: 1 (documented)
- Unnecessary conflicts: 2 (fixable)

### Version Support
- Targets version: <X.Y>
- Multi-version: No
- Version-specific constructs: None

## Reuse Decision Matrix
| Component | Reuse? | Rationale |
|-----------|--------|-----------|
| Lexical atoms | Yes | Well-tested, matches spec |
| Expression engine | No | Uses outdated nested precedence |
| Statement rules | Partial | Good structure, missing fields |
| Conflict resolution | No | Ad-hoc, not algorithmic |

## Action Plan
1. Import lexical atoms from existing grammar
2. Rebuild expression engine per PAGS Phase III
3. Extend statement rules with missing fields
4. Re-resolve conflicts per PAGS Phase IV algorithm
```

### 4. NEW REQUIRED: Edge Case Corpus

#### `specs/edge_cases.yaml`
**Purpose:** Document known problematic constructs before grammar design.

```yaml
edge_cases:
  - id: "EC001"
    name: "Dangling else ambiguity"
    description: "if (a) if (b) x; else y; - which if owns the else?"
    language_resolution: "Else binds to nearest if"
    test_input: |
      if (outer)
        if (inner)
          action1;
        else
          action2;
    expected_parse: "else binds to inner if"
    pags_phase: "IV (Conflict Resolution)"
    resolution_strategy: "prec.right on if_statement"
    
  - id: "EC002"
    name: "Operator precedence edge case"
    description: "a + b * c ** d"
    test_input: "a + b * c ** d"
    expected_parse: "a + (b * (c ** d))"
    pags_phase: "III (Expression Engine)"
    
  # Extract from:
  # - Language spec "ambiguities" section
  # - Known parser bugs in other implementations
  # - Community-reported edge cases
```

## Planning Phase Execution

### Step 1: Specification Extraction
**Jules Command:**
```
Extract and normalize language specification for <LANGUAGE>.

Inputs:
- Official grammar specification: <URL from CONTEXT.md>
- <language>_history.json

Tasks:
1. Parse official grammar lexical chapter → generate specs/lexical.yaml
2. Extract operator precedence table → generate specs/operators.yaml
3. Extract EBNF rules for major constructs → generate specs/syntax.yaml
4. Parse <language>_history.json → generate enhanced specs/versions.yaml
5. Cross-reference general_rules_de.md Section III.A → generate specs/usg_constraints.yaml

Document all extractions with source references.
Validate YAML syntax.
```

### Step 2: Existing Grammar Analysis (If Applicable)
**Jules Command:**
```
Audit existing tree-sitter-<language> grammar for reuse potential.

Repository: <URL>

Tasks:
1. Clone and analyze structure
2. Inventory all nodes and map to USG NodeKinds
3. Check field coverage against general_rules_de.md
4. Analyze conflict resolution strategies
5. Document version support
6. Generate reuse decision matrix in analysis/existing_grammar_audit.md

Recommendation: Which components to reuse, which to rebuild.
```

### Step 3: Edge Case Collection
**Jules Command:**
```
Compile edge case corpus for <LANGUAGE>.

Sources:
- Official grammar spec "ambiguities" section
- Language reference "corner cases"
- Existing parser bug reports
- Community forums (StackOverflow, GitHub issues)

Generate specs/edge_cases.yaml with:
- Minimal test inputs
- Expected parse structures
- Resolution strategies

Prioritize cases that affect conflict resolution or precedence.
```

### Step 4: Generate Synthesis Plan
**Jules Command:**
```
Generate grammar synthesis execution plan.

Inputs:
- All specs/*.yaml files
- analysis/existing_grammar_audit.md (if exists)

Output: `SYNTHESIS_PLAN.md` containing:
1. Lexical atoms to implement (from lexical.yaml)
2. Operator precedence table (from operators.yaml)
3. Structural skeleton outline (from syntax.yaml)
4. Version handling strategy (from versions.yaml)
5. USG compliance checklist (from usg_constraints.yaml)
6. Predicted conflicts and resolution strategies (from edge_cases.yaml)
7. Reuse decisions (from audit)

This plan becomes input to Tasks 1-6.
```

## Deliverables

1. **`specs/lexical.yaml`** - Normalized lexical specification
2. **`specs/operators.yaml`** - Complete precedence table
3. **`specs/syntax.yaml`** - Structural EBNF rules
4. **`specs/versions.yaml`** - Enhanced version matrix
5. **`specs/usg_constraints.yaml`** - USG mapping requirements
6. **`specs/edge_cases.yaml`** - Edge case corpus
7. **`analysis/existing_grammar_audit.md`** (if applicable)
8. **`SYNTHESIS_PLAN.md`** - Execution roadmap for Tasks 1-6

## Verification Checklist

- [ ] All YAML files validate (`yamllint specs/*.yaml`)
- [ ] Every extraction has spec source reference
- [ ] Version matrix cross-referenced with history.json
- [ ] USG constraints match general_rules_de.md Section III.A exactly
- [ ] Edge cases cover all known ambiguities from spec
- [ ] Synthesis plan addresses all predicted conflicts

## Success Criteria

**Task 0 complete when:**
- All specs/*.yaml files generated and validated
- Synthesis plan generated
- Human review confirms: "This plan, if executed, will produce a complete, USG-compliant grammar"

**Proceed to Task 1 only after human approval of SYNTHESIS_PLAN.md.**