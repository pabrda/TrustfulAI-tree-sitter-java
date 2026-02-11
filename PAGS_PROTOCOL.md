---
title: Grammar Generation for ECS Engine
date: 2026-02-08
tags: [code analysis, ast, parser, language, grammar, tree-sitter]
---

Dies ist das **Protokoll für Algorithmische Grammatik-Synthese (PAGS)**.
Wir raten nicht; wir entwerfen. Wir „beheben“ keine Konflikte; wir verhindern sie durch strukturelle Voraussicht.

Dieser Leitfaden setzt voraus, dass Sie das `tree-sitter-cli` installiert und ein grundlegendes Projekt initialisiert haben (`tree-sitter init`).

---

# Die Architektur der Routine

Die Routine ist in **sechs Phasen** unterteilt. Sie dürfen nicht zur nächsten Phase übergehen, bevor die aktuelle Phase alle Verifikationsmetriken erfüllt.

1.  **Phase I: Dekonstruktion & Atomisierung** (Lexikalische Spezifikationen)
2.  **Phase II: Das Skelett** (Top-Level-Struktur)
3.  **Phase III: Die Ausdrucks-Engine** (Präzedenzlogik)
4.  **Phase IV: Der Konfliktauflösungs-Algorithmus** (LR(1)-Feinabstimmung)
5.  **Phase V: AST-Verfeinerung** (Felder & Verborgene Knoten)
6.  **Phase VI: Der Externe Scanner** (Kontextabhängigkeit)

---

## Phase I: Dekonstruktion & Atomisierung

Bevor wir eine einzige Regel definieren, müssen wir die „Atome“ des Universums definieren. In Tree-sitter sind dies Ihre **lexikalischen Knoten** (Terminale).

### Die Logik
Tree-sitter verwendet einen separaten Lexer. Wenn Ihr Regex zu locker ist, wird Ihr Parser an Mehrdeutigkeiten scheitern. Wir definieren zuerst Atome, um sicherzustellen, dass der Parser stets auf solidem Fundament steht.

### Die Routine
1.  **Sprachspezifikation lokalisieren:** Finden Sie die EBNF oder das Referenzhandbuch der Zielsprache.
2.  **Primitive implementieren:** Definieren Sie `identifier`, `number`, `string` und `comment`.
3.  **Strenges Regex erzwingen:** Verwenden Sie `token()`, um eine sofortige Auswertung zu erzwingen.

### Implementierungsbeispiel (`grammar.js`)

```javascript
module.exports = grammar({
  name: 'your_language',

  // ALGORITHMIC REASONING:
  // We define 'extras' early to tell the parser what to ignore globally.
  // Usually whitespace and comments.
  extras: $ => [
    /\s/,
    $.comment,
  ],

  rules: {
    // We need a temporary root to test atoms
    source_file: $ => repeat($._definition),

    _definition: $ => choice(
      $.identifier,
      $.number,
      $.string
    ),

    // ATOM 1: Identifiers
    // Logic: Must start with letter/underscore, followed by alphanumerics.
    identifier: $ => /[a-zA-Z_][a-zA-Z0-9_]*/,

    // ATOM 2: Numbers
    // Logic: Handle hex, decimals, and floats strictly.
    // Use token() to ensure this is treated as a single unit, not a tree.
    number: $ => token(choice(
      /\d+/,
      /0x[0-9a-fA-F]+/,
      /\d+\.\d+/
    )),

    // ATOM 3: Strings
    // Logic: Handle escaped quotes. This regex is a common point of failure.
    // " followed by (anything not " or \) OR (escaped anything) repeated, ending in "
    string: $ => /"[^"\\]*(\\.[^"\\]*)*"/,

    // ATOM 4: Comments
    // Logic: From // to end of line.
    comment: $ => token(/\/\/.*/),
  }
});
```

**Verifikation:** Erstellen Sie `test/corpus/atoms.txt`. Testen Sie jedes Atom isoliert. Wenn diese fehlschlagen, ist alles andere bedeutungslos.

---

## Phase II: Das Skelett

Nun bauen wir die Wirbelsäule. Wir definieren die Hierarchie von der Dateiwurzel bis hin zur Funktions-/Klassenebene.

### Die Logik

Hier verwenden wir einen **Top-Down-Ansatz**. Wir definieren den Container, dann die Inhalte. Wir verwenden `_` (Unterstrich)-Präfixe für „versteckte Regeln“ (Knoten, die nicht im finalen Syntaxbaum erscheinen sollen, wie Wrapper-Knoten).

### Die Routine

1. `source_file` definieren.
2. Die Hauptblöcke definieren: `function_definition`, `class_definition`, `statement`.
3. `seq` (Sequenz), `choice` (Alternativen) und `repeat` (0 oder mehr) verwenden.

### Implementierungsbeispiel

```javascript
    // The Root
    source_file: $ => repeat($._top_level_item),

    _top_level_item: $ => choice(
      $.function_definition,
      $.class_definition,
      $._statement
    ),

    // Structural Node: Function
    // Logic: keyword -> name -> parameters -> body
    function_definition: $ => seq(
      'func',
      field('name', $.identifier),
      field('parameters', $.parameter_list),
      field('body', $.block)
    ),

    parameter_list: $ => seq(
      '(',
      // Comma separation logic: optional(seq(item, repeat(seq(',', item))))
      optional(seq(
        $.identifier,
        repeat(seq(',', $.identifier))
      )),
      ')'
    ),

    block: $ => seq(
      '{',
      repeat($._statement),
      '}'
    ),
```

**Algorithmischer Hinweis zu Feldern:** Beachten Sie `field('name', ...)`. Wenden Sie diese *jetzt* an. Warten Sie nicht. Sie sind später essenziell für das Abfragesystem.

---

## Phase III: Die Ausdrucks-Engine (Kritisch)

Ausdrücke beinhalten Rekursion, Präzedenz und Assoziativität.

### Die Logik

Wir verwenden keine verschachtelten Regeln für Präzedenz (z. B. `term`, `factor`, `unary`). Das ist veraltet. Tree-sitter stellt eine **Präzedenz-Tabelle** über `prec.left`, `prec.right` und `prec` bereit.

### Die Routine

1. Ein `PREC`-Objekt definieren, das Namen auf Ganzzahlen abbildet.
2. Eine monolithische `expression`-Regel mit `choice` konstruieren.
3. Auf jeden binären und unären Operator Präzedenz anwenden.

### Implementierungsbeispiel

```javascript
// PRECEDENCE TABLE
// Higher numbers = tighter binding
const PREC = {
  ASSIGN: 1,
  OR: 2,
  AND: 3,
  RELATIONAL: 4,
  ADD: 5,
  MULT: 6,
  UNARY: 7,
  CALL: 8,
};

// ... inside rules ...

    _statement: $ => choice(
      $.return_statement,
      $.expression_statement
    ),

    expression_statement: $ => seq($.expression, ';'),

    expression: $ => choice(
      $.identifier,
      $.number,
      $.string,
      $.binary_expression,
      $.unary_expression,
      $.call_expression
    ),

    // BINARY EXPRESSIONS
    // Logic: prec.left means 1 + 2 + 3 parses as (1 + 2) + 3
    binary_expression: $ => choice(
      prec.left(PREC.ADD, seq($.expression, '+', $.expression)),
      prec.left(PREC.ADD, seq($.expression, '-', $.expression)),
      prec.left(PREC.MULT, seq($.expression, '*', $.expression)),
      prec.right(PREC.ASSIGN, seq($.expression, '=', $.expression))
    ),

    // UNARY EXPRESSIONS
    unary_expression: $ => prec(PREC.UNARY, seq(
      choice('-', '!'),
      $.expression
    )),
```

---

## Phase IV: Der Konfliktauflösungs-Algorithmus

Wenn Sie `tree-sitter generate` ausführen, sehen Sie irgendwann: `Unresolved conflict for symbol sequence...`

**Keine Panik.** Folgen Sie diesem Algorithmus zur Auflösung.

### Die Konflikttypen

1. **Shift/Reduce:** Der Parser kann entweder das nächste Token konsumieren (Shift) oder die aktuelle Regel abschließen (Reduce).
2. **Reduce/Reduce:** Der Parser kann zwei unterschiedliche Regeln mit denselben Tokens abschließen. (Dies weist auf ein schlechtes Grammatikdesign hin.)

### Das Auflösungsprotokoll

**Schritt 1: Assoziativitätsprüfung**

* Handelt es sich um einen binären Operator? Verwenden Sie `prec.left` oder `prec.right`.

**Schritt 2: Bindungsstärke (Präzedenz)**

* „Haftet“ eine Regel eindeutig stärker als die andere? (z. B. Funktionsaufruf vs. Variablenzugriff).
* **Aktion:** Weisen Sie der stärkeren Regel ein höheres `prec(N, ...)` zu.

**Schritt 3: Das `conflicts`-Array (Der GLR-Hammer)**

* Verwenden Sie dies nur, wenn die Mehrdeutigkeit in der Sprache *echt* ist (z. B. C++ beim Parsen von `A * B` als Multiplikation vs. Zeigerdeklaration).
* **Aktion:** Weisen Sie Tree-sitter explizit an, den Parser zu verzweigen und beide Pfade zu versuchen.

```javascript
  // Example: C-style cast vs Parenthesized expression
  // (int) x  vs  (x) + y
  conflicts: $ => [
    [$.type_cast, $.parenthesized_expression]
  ],
```

**Schritt 4: Dynamische Präzedenz (`prec.dynamic`)**

* Verwenden Sie dies, wenn zwei Regeln dieselbe Zeichenkette matchen, aber eine nur in einem bestimmten Laufzeitkontext gültig ist. Weisen Sie Ganzzahlen zu. Der Parser wählt zur Laufzeit den Pfad mit der höchsten dynamischen Präzedenz.

---

## Phase V: AST-Verfeinerung

Ein roher Parsebaum ist verrauscht. Wir müssen ihn für den Endanwender (Syntax-Highlighter, semantische Analysatoren) aufbereiten.

### Die Routine

1. **Verborgene Regeln:** Stellen Sie sicher, dass alle Wrapper-Regeln (wie `_statement`, `_definition`) mit `_` beginnen. Dadurch werden sie aus dem finalen Baum entfernt und Kinder direkt mit Eltern verbunden.
2. **Feldkennzeichnung:** Stellen Sie sicher, dass jedes bedeutungsvolle Kind (Name, Body, Bedingung, links, rechts) ein `field()` besitzt.
3. **Inline:** Wenn eine Regel einfach ist und überall verwendet wird (z. B. ein spezifischer Keyword-Wrapper), fügen Sie sie dem `inline`-Array hinzu, um die Parser-Zustandsgröße zu reduzieren.

```javascript
  // Merges these rules into their parents during compilation
  inline: $ => [$._type_identifier, $._expression_wrapper],
```

---

## Phase VI: Der Externe Scanner (Die nukleare Option)

Wenn Ihre Sprache **signifikante Einrückung** (Python, Yaml) oder **benutzerdefinierte Begrenzer** (Ruby Heredocs) besitzt, reichen Regex und CFG nicht aus. Sie benötigen C/C++-Logik.

### Die Routine

1. `src/scanner.c` erstellen.
2. `tree_sitter_YOUR_LANG_external_scanner_create` implementieren.
3. Einen Stack von Einrückungsebenen oder Begrenzerzuständen verwalten.
4. Diesen Zustand in `serialize` und `deserialize` serialisieren.

*Hinweis: Dies ist fortgeschritten. Verwenden Sie es nur, wenn Phase I-V die Parsing-Logik nicht lösen können.*

---

## Die wasserdichte Verifikationsschleife (TDD)

Sie schreiben nicht erst die Grammatik und testen dann. Sie schreiben einen Test und dann die Grammatik.

1. **Corpus-Datei erstellen:** `test/corpus/functions.txt`
2. **Eingabe schreiben:**

   ```
   ==================
   Basic Function
   ==================
   func add(a, b) {
     return a + b;
   }
   ---
   (source_file
     (function_definition
       name: (identifier)
       parameters: (parameter_list (identifier) (identifier))
       body: (block
         (return_statement (binary_expression ...))
       )
     )
   )
   ```
3. **Ausführen:** `tree-sitter generate && tree-sitter test`
4. **Fehlschlag:** Debug-Ausgabe analysieren (`tree-sitter parse test.file -d`).
5. **Beheben:** Grammatik anpassen.
6. **Bestanden:** Zur nächsten Funktion übergehen.

### Abschließende Checkliste für „wasserdichten“ Status

* [ ] `tree-sitter generate` erzeugt keine Warnungen.
* [ ] Alle Atome (Bezeichner, Strings) sind robust gegenüber Randfällen.
* [ ] Die Präzedenz behandelt `1 + 2 * 3` korrekt (Multiplikation bindet stärker).
* [ ] Der AST ist sauber (keine unnötigen Wrapper-Knoten sichtbar).
* [ ] Jede größere Syntaxfunktion besitzt einen entsprechenden Corpus-Test.

Folgen Sie dieser Routine mit mathematischer Präzision, und Sie werden einen Parser in Industriequalität erstellen.


# Phase VII: Semantisierung — Der Intelligenz-Layer

Dies ist ein kritischer Wendepunkt im Grammatik-Engineering.

Die meisten Entwickler stoppen, sobald der Parser Eingaben akzeptiert. Das ist ein Fehler. Ein Parser, der nur einen rohen Concrete Syntax Tree (CST) erzeugt, ist für Menschen oder IDEs nutzlos. Wir müssen den **CST** (wie der Text aussieht) in einen **AST** (was der Code bedeutet) transformieren.

Diesen Prozess nennen wir **Semantisierung**.

Im Tree-sitter-Ökosystem erfolgt Semantisierung über eine strikte Drei-Schichten-Architektur:

1. **Strukturelle Semantik** (Felder & Verborgene Knoten)
2. **Kategoriale Semantik** (Supertypen)
3. **Relationale Semantik** (Query-Dateien)

---

## Layer I: Strukturelle Semantik (`grammar.js`)

Die Baumstruktur selbst trägt Bedeutung.

### Feld-Erzwingungsprotokoll

Indizes ändern sich — Rollen nicht. Jeder bedeutungstragende Knoten erhält ein `field`.

```javascript
function_def: $ => seq(
  'func',
  field('name', $.identifier),
  field('body', $.block)
)
```

### Rauschreduktions-Algorithmus

Wrapper-Regeln mit `_` verbergen interne Grammatikdetails.

```javascript
rules: {
  _statement: $ => choice($.return, $.loop, $.expression_statement),
  return: $ => seq('return', $.expression)
}
```

---

## Layer II: Kategoriale Semantik (`grammar.js`)

Unterschiedliche Knoten werden als gleiche abstrakte Kategorie behandelt.

```javascript
module.exports = grammar({
  name: 'lang',

  supertypes: $ => [
    $._expression,
    $._statement,
    $._declaration
  ],

  rules: {
    _expression: $ => choice(
      $.binary_expression,
      $.call_expression,
      $.identifier
    ),
  }
});
```

---

## Layer III: Relationale Semantik (Query-Engine)

Semantik wird außerhalb der Grammatik definiert.

### Syntax-Highlighting — `highlights.scm`

```scheme
(function_definition
  name: (identifier) @function)

["func" "return" "if"] @keyword
```

### Scope & Definition — `locals.scm`

```scheme
(block) @local.scope

(function_definition
  name: (identifier) @local.definition)

(identifier) @local.reference
```

### Code-Navigation — `tags.scm`

```scheme
(function_definition
  name: (identifier) @name) @definition.function
```

---

## Verifikations-Checkliste für Semantisierung

1. Feld-Test → Rollen sichtbar
2. Flattening-Test → keine Wrapper sichtbar
3. Highlight-Test → Tokens markiert
4. Scope-Test → korrekte Umbenennung möglich


Dies ist die **Algorithmic Breakdown of Grammar Synthesis (ABGS)**.

Wir zerlegen den Prozess der Grammatikbildung (Phase I) in eine deterministische Abfolge von 10 Schritten. Jeder Schritt ist funktional isoliert und erfordert spezifische Eingabedaten.

---

# ALGORITHMISCHE ZERLEGUNG DER GRAMMATIKBILDUNG (ABGS)

## Phase I: Die Superset-Grammatik-Synthese

### Schritt 1: Lexikalische Atomisierung (Terminale)

| Logik | Beschreibung | Benötigte Daten (Input) | Output |
| :--- | :--- | :--- | :--- |
| **Ziel** | Definition aller primitiven Token (Atome) der Sprache. | 1. **Sprachspezifikation:** Kapitel "Lexical Structure" (Regex für Identifier, Literale). 2. **Versions-Union:** Die Vereinigungsmenge aller gültigen Zeichen (z.B. `$` in JS, `_` in Python). | `identifier`, `number`, `string`, `comment` (als Regex-Regeln in `token()`). |
| **Aktion** | Implementierung der `rules` für die Atome und der `extras` (Whitespace, Kommentare). | | |

### Schritt 2: Schlüsselwort-Klassifizierung und Konflikt-Vorbereitung

| Logik | Beschreibung | Benötigte Daten (Input) | Output |
| :--- | :--- | :--- | :--- |
| **Ziel** | Unterscheidung zwischen Schlüsselwörtern (Terminale) und Bezeichnern (Non-Terminale). | 1. **Vollständige Schlüsselwortliste** aller Versionen (z.B. `async`, `await`, `yield`, `def`). 2. **Versions-Divergenz:** Welche Schlüsselwörter sind in welcher Version erlaubt/verboten? | `keyword_rules` (z.B. `'if'`, `'for'`). |
| **Aktion** | Schlüsselwörter als String-Literale definieren. Schlüsselwörter, die in älteren Versionen als Bezeichner erlaubt waren, in die `conflicts` Liste aufnehmen (z.B. `async` in Python < 3.7). | | |

### Schritt 3: Hierarchische Skelett-Definition

| Logik | Beschreibung | Benötigte Daten (Input) | Output |
| :--- | :--- | :--- | :--- |
| **Ziel** | Definition der Top-Level-Struktur (Root-Node) und der Hauptcontainer. | 1. **EBNF-Definition** der Sprache (Top-Level-Regeln). 2. **Strukturelle Hierarchie:** Wie sind Dateien, Module, Klassen und Funktionen verschachtelt? | `source_file`, `_top_level_item`, `class_definition`, `function_definition` (als `seq` und `repeat` Regeln). |
| **Aktion** | Verwendung von `_` (Unterstrich) für Wrapper-Regeln, die nicht im finalen AST erscheinen sollen (z.B. `_statement`). | | |

### Schritt 4: Statement- und Kontrollfluss-Synthese

| Logik | Beschreibung | Benötigte Daten (Input) | Output |
| :--- | :--- | :--- | :--- |
| **Ziel** | Definition aller Kontrollfluss- und Anweisungsregeln. | 1. **Kontrollfluss-Spezifikation:** Regeln für `if`, `while`, `for`, `switch`/`match`. 2. **Versions-Union:** Superset-Regeln für divergierende Syntax (z.B. `for` mit oder ohne Klammern). | `if_statement`, `loop_statement`, `return_statement` (als `choice` und `seq` Regeln). |
| **Aktion** | Nutzung von `choice` zur Abdeckung aller syntaktischen Varianten (z.B. `for` Schleife mit Initializer vs. `for-each` Schleife). | | |

### Schritt 5: Präzedenz- und Assoziativitäts-Tabelle (Ausdrücke)

| Logik | Beschreibung | Benötigte Daten (Input) | Output |
| :--- | :--- | :--- | :--- |
| **Ziel** | Definition der Operator-Hierarchie und der Assoziativität. | 1. **Operator-Tabelle:** Vollständige Liste aller Operatoren (`+`, `*`, `:=`, `<<`, `->`). 2. **Präzedenz-Level:** Numerische Rangfolge der Operatoren (Multiplikation > Addition). 3. **Assoziativität:** Links- (`+`, `*`) oder Rechts-assoziativ (`=`, `**`). | `PREC` Konstante (numerische Level). `binary_expression` (mit `prec.left`, `prec.right`). |
| **Aktion** | Erstellung einer monolithischen `expression` Regel, die alle Operatoren über `choice` und `prec` abdeckt. | | |

### Schritt 6: Feld-Injektion und AST-Verfeinerung

| Logik | Beschreibung | Benötigte Daten (Input) | Output |
| :--- | :--- | :--- | :--- |
| **Ziel** | Semantische Kennzeichnung aller wichtigen Kind-Knoten. | 1. **USG-Schema (Vorab):** Welche Felder benötigt der USG (z.B. `name`, `body`, `condition`)? 2. **Rollen-Definition:** Welche Kind-Knoten spielen welche Rolle? | `field('name', ...)` und `field('body', ...)` in allen Container-Regeln. |
| **Aktion** | Anwendung der `field()` Funktion auf alle Kind-Knoten, die im finalen USG benötigt werden. | | |

### Schritt 7: Konflikt-Management und GLR-Tuning

| Logik | Beschreibung | Benötigte Daten (Input) | Output |
| :--- | :--- | :--- | :--- |
| **Ziel** | Auflösung aller verbleibenden Shift/Reduce-Konflikte. | 1. **`tree-sitter generate` Output:** Liste der ungelösten Konflikte. 2. **Sprachkontext:** Wissen über die tatsächliche Auflösung (z.B. "In C++ gewinnt der Typ-Cast"). | `conflicts: $ => [...]` Array. Gezielte Anwendung von `prec.dynamic`. |
| **Aktion** | Iterative Anwendung von `prec.dynamic` oder expliziter `conflicts` Deklaration, bis `tree-sitter generate` keine Warnungen mehr ausgibt. | | |

### Schritt 8: Supertype-Definition (Vorbereitung für USG)

| Logik | Beschreibung | Benötigte Daten (Input) | Output |
| :--- | :--- | :--- | :--- |
| **Ziel** | Gruppierung von CST-Knoten in abstrakte Kategorien. | 1. **USG-Taxonomie:** Definition der abstrakten Kategorien (`_expression`, `_declaration`, `_type`). | `supertypes: $ => [...]` Array. |
| **Aktion** | Registrierung der Wrapper-Regeln (z.B. `_expression`) im `supertypes` Array. | | |

### Schritt 9: Test-Korpus-Erstellung (Verifikation)

| Logik | Beschreibung | Benötigte Daten (Input) | Output |
| :--- | :--- | :--- | :--- |
| **Ziel** | Erstellung eines umfassenden Test-Sets zur Validierung der Grammatik. | 1. **Versions-spezifische Beispiele:** Code, der nur in v1 oder v2 gültig ist (z.B. Python 2 `print` vs. Python 3 `f-string`). 2. **Edge-Case-Beispiele:** Ambiguitäten, die Konflikte auslösen (z.B. verschachtelte Generics). | `test/corpus/*.txt` Dateien mit erwartetem AST-Output. |
| **Aktion** | Ausführung von `tree-sitter test` und Behebung aller Diskrepanzen. | | |

### Schritt 10: Finalisierung und Kompilierung

| Logik | Beschreibung | Benötigte Daten (Input) | Output |
| :--- | :--- | :--- | :--- |
| **Ziel** | Generierung der finalen Parser-Artefakte. | 1. **`grammar.js` (Final).** 2. **`node-types.json` (Generiert).** | `parser.c`, `parser.h`, `node-types.json` (für den Mapper). |
| **Aktion** | Ausführung von `tree-sitter generate`. | | |