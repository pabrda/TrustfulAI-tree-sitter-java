### **`GENERAL_RULES.md` (Version 3.0 - Goldene Quelle der Wahrheit)**

## UNTERSTÜTZTE SPRACHEN & ÖKOSYSTEME

1.	Python	    PyPI (pip)
2.	JavaScript	NPM (npm, yarn)
3.	TypeScript	NPM (npm, yarn)
4.	Java	    Maven Central (Maven, Gradle)
5.	Kotlin	    Maven Central (Maven, Gradle)
6.	Scala	    Maven Central (Maven, Gradle)
7.	Rust	    Crates.io (Cargo)
8.	Go      	Go Modules (Go Proxy)
9.	C#	        NuGet
10.	PHP	        Packagist (Composer)
11.	Ruby    	RubyGems (Gem)
12.	Swift	    Swift Package Index (SwiftPM)
13.	R	        CRAN
14.	Haskell	    Hackage (Cabal, Stack)
15.	Elixir	    Hex (Mix)
16.	Erlang	    Hex (Rebar3)
17.	Clojure	    Clojars / Maven (Leiningen)
18.	PowerShell	PowerShell Gallery
19.	Julia	    Julia Registry (Pkg)
20.	Nim	        Nimble
21.	F#	        NuGet
22.	OCaml	    OPAM
23.	PureScript	Pursuit (Spago)
24.	Zig	        Zig Package Manager (Emerging)
25. HTML        -
26. Bash        -
27. C           -
28. C++         -
29. Dart        Pub (pub)
30. Lua         LuaRocks
31. Perl        CPAN (cpanm)
32. Fortran     fpm (Fortran Package Manager)
33. D           DUB
34. Crystal     Shards
35. Elm         Elm Packages

---

## I. Architektonische Grundprinzipien

Diese drei Prinzipien bilden das unumstößliche Fundament der gesamten Graphen-Architektur. Sie haben Vorrang vor allen nachfolgenden Regeln und stellen die Konsistenz, Normalisierung und Reversibilität des `SemanticGraph` sicher.

### **1. Das Prinzip der Nicht-Redundanz (Abgeleitete vs. Gespeicherte Daten)**

**DIE REGEL:** Informationen, die aus der Struktur des Graphen abgeleitet werden können, **DÜRFEN NICHT** als persistente Eigenschaft im `metadata`-Feld eines `SemanticNode` gespeichert werden. Die Graphen-Struktur ist die alleinige Quelle der Wahrheit für die **kanonische Repräsentation**. Zur Laufzeit erstellte, temporäre Analyse-Strukturen (wie das ECS `World`) dürfen zur Performance-Optimierung abgeleitete oder denormalisierte Daten halten.

**BEGRÜNDUNG:** Das Speichern abgeleiteter Informationen (Aggregationen, Daten von Kindern) in einem Eltern-Knoten ist ein fundamentales Anti-Pattern. Es führt unweigerlich zu Daten-Synchronisationsproblemen bei Code-Änderungen, verletzt das Single-Responsibility-Prinzip (der Eltern-Knoten kennt plötzlich Implementierungsdetails seiner Kinder) und macht den Graphen fragil und inkonsistent. Alle derartigen Informationen müssen zur Abfragezeit ("on-demand") durch einen Resolver oder eine Analyse-Engine berechnet werden. Dies stellt sicher, dass jede Antwort auf eine Abfrage immer den aktuellen, wahren Zustand des Graphen widerspiegelt.

**KONKRETE VERSTÖSSE (STRENGSTENS VERBOTEN):**

| `NodeKind` des Eltern-Knotens | Verbotene Redundante Metadaten | Korrekte Ableitungsmethode (zur Abfragezeit) |
| :--- | :--- | :--- |
| `Module` | `dependencies: [...]` | Traversiere den Graphen und finde alle Kind-Knoten vom Typ `Import`. |
| `Block` | `statement_count: 5` | Zähle die direkten Kind-Knoten des `Block`-Knotens über `Contains`-Kanten. |
| `Call` | `argument_count: 3` | Zähle die Kind-Knoten vom Typ `Argument`. |
| `FunctionDef` | `parameters: [...]`, `returns: [...]` | **Primär:** Finde alle Kind-Knoten vom Typ `Parameter` (via `Accepts`-Kante) und `Type` (via `Returns`-Kante). **Gültiger Fallback:** Wenn die Sprachgrammatik die Extraktion von `Parameter`-Knoten unpraktikabel macht, **DARF** eine serialisierte Liste der Parameter im `parameters`-Metadatenfeld gespeichert werden. Nachgelagerte Systeme **MÜSSEN** beide Quellen (Kanten und Metadaten) berücksichtigen. |
| `InterfaceDef` | `required_methods: [...]` | Finde alle Kind-Knoten vom Typ `MethodDef` über `Contains`-Kanten. |

**Jede Transformationsregel, die die in dieser Tabelle aufgeführten verbotenen Metadaten erzeugt, ist per Definition fehlerhaft und muss korrigiert werden.**

### **2. Das Prinzip des Einheitlichen Metadaten-Schemas**

**DIE REGEL:** Es gibt nur **EIN** Schema für das `metadata`-Objekt eines `SemanticNode`. Dieses Schema wird durch die Kombination von zwei Tabellen in diesem Dokument definiert:
1.  **Die Haupt-Taxonomie-Tabelle (Abschnitt III.A):** Definiert die **strukturell kritischen und oft obligatorischen** Metadaten-Felder für jeden `NodeKind`.
2.  **Die Tabelle der Allgemeinen Verhaltens-Flags (Abschnitt IV):** Stellt ein **standardisiertes Vokabular** von allgemeinen, oft booleschen Flags bereit, die auf das `metadata`-Objekt **jedes relevanten `NodeKind`** angewendet werden können.

**BEGRÜNDUNG:** Ein einheitliches Schema ist entscheidend für die Konsistenz und die Fähigkeit, sprachunabhängige Abfragen zu schreiben. Ohne dieses Prinzip würde jede Transformationsregel ihre eigenen, inkonsistenten Metadaten-Schlüssel erfinden, was die nachgelagerte Analyse unmöglich machen würde.

**IMPLEMENTIERUNG:** Eine Transformationsregel **MUSS** sich an die folgenden Namenskonventionen für Metadaten-Schlüssel halten:
1.  **Globale Schlüssel:** Alle in diesem Dokument (Abschnitt III.A und IV) definierten Schlüssel sind globale, sprachunabhängige Standards und werden ohne Präfix verwendet.
2.  **Sprachspezifische Schlüssel:** Schlüssel, die nur für eine einzelne Sprache relevant sind und kein globales Äquivalent haben, **MÜSSEN** mit einem sprachspezifischen Präfix versehen werden (z.B. `_py_GIL_release: true`, `_cpp_is_constexpr: true`).
3.  **Experimentelle Schlüssel:** Schlüssel, die für die Entwicklung neuer, noch nicht standardisierter Features verwendet werden, **MÜSSEN** mit dem Präfix `_exp_` beginnen (z.B. `_exp_my_feature: ...`).

Die Erfindung neuer globaler Schlüssel ohne Präfix ist strengstens untersagt, es sei denn, sie werden durch eine formale Änderung dieses Dokuments genehmigt.

**SCHEMA-VALIDIERUNG (TOOLING):** Ein automatisiertes Test-Skript **MUSS** implementiert werden. Dieses Skript führt alle `transform`-Funktionen auf Beispiel-Code-Snippets aus und validiert die `metadata`-Objekte der erzeugten `SemanticNode`s gegen ein aus diesem Dokument generiertes JSON-Schema. Builds, die gegen das Schema verstoßen, **MÜSSEN** fehlschlagen.

### **3. Das Prinzip der Syntaktischen Treue für verworfene Knoten (Die "Klammer-Regel")**

**DIE REGEL:** Bestimmte AST-Knoten (wie `parenthesized_expression` oder `expression_statement`) sind rein syntaktischer Natur und repräsentieren keine eigenständige semantische Entität. Um den Graphen sauber zu halten, **DÜRFEN** diese transienten Knoten **NICHT** in einen eigenen `SemanticNode` umgewandelt werden. Die Transformationsregel muss sie "überspringen" (mittels `TraversalStrategy::Dissolve` oder `Forward`) und ihre Kinder direkt verarbeiten.

**JEDOCH:** Wenn das Verwerfen des Knotens zu einem Informationsverlust führen würde, der für eine bit-perfekte Code-Rekonstruktion erforderlich ist (z.B. Operator-Vorrang), **MUSS** ein Metadaten-Flag (wie `is_parenthesized: true`) an den `SemanticNode` des **Kindes** angehängt werden, um diese kritische syntaktische Information zu bewahren.

**BEGRÜNDUNG:** Dieses Flag wird **nicht als redundant betrachtet** (im Sinne von Prinzip 1), da der ursprüngliche Quellknoten für diese Information aus dem Graphen entfernt wurde und die Information andernfalls unwiederbringlich verloren ginge. Es ist die notwendige Bedingung für die Reversibilität der Transformation.

### **4. Das Prinzip der Architektonischen Schichten (Graph vs. ECS)**

**DIE REGEL:** Das System ist in zwei primäre Datenschichten unterteilt, mit einer klaren Trennung der Verantwortlichkeiten:
    1.  **Der `SemanticGraph` (mit seinen `SemanticNode`s):** Dies ist die **primäre, persistente und kanonische Quelle der Wahrheit**. Alle Informationen, die aus der Code-Analyse (sowohl "Broad Scan" als auch "Deep Dive") gewonnen werden, **MÜSSEN** in den `SemanticNode`-Strukturen gespeichert werden.
    2.  **Das ECS `World`:** Dies ist ein **temporärer, flüchtiger In-Memory-Index**, der zur Laufzeit für die Ausführung komplexer, performanter Abfragen (`universal_query`) aufgebaut wird. Es ist eine "denormalisierte Sicht" oder eine "Projektion" der Daten aus dem `SemanticGraph`.

**BEGRÜNDUNG:** Diese strikte Trennung verhindert Datenredundanz und Synchronisationsprobleme. Der `SemanticGraph` ist für die vollständige, strukturierte Repräsentation optimiert, während das ECS `World` für schnelle, analytische Abfragen optimiert ist. Transformationsregeln interagieren **niemals** direkt mit dem ECS `World`; sie produzieren ausschließlich `SemanticNode`s.

---

## II. Die Sechs Implementierungs-Verträge für Transformationsregeln

Diese sechs Regeln sind die konkreten, umsetzbaren Anweisungen für jeden Entwickler, der eine `transform`-Funktion schreibt. Sie leiten sich aus den übergeordneten Architekturprinzipien ab und stellen die Funktionsfähigkeit der nachgelagerten Analyse-Systeme sicher.

### **1. Der Vertrag der Eindeutigen Identifikation**

**PRINZIP:** Jede Entität, die referenziert werden kann, muss einen stabilen, global eindeutigen Bezeichner haben.
**ANWEISUNG:** Für jeden `SemanticNode`, der eine Definition darstellt (`definition: true`), **MUSS** das Feld `qualified_name` befüllt werden. Dieser Name **MUSS** durch die Kombination des `ScopeStack` mit dem Eigennamen des Knotens konstruiert werden.

### **2. Der Vertrag der Deklarations-vs-Verwendungs-Dichotomie**

**PRINZIP:** Das System muss klar zwischen Definition und Verwendung eines Symbols unterscheiden.
**ANWEISUNG:** Das `definition: bool`-Flag im `SemanticNode` **MUSS** korrekt gesetzt werden. `function_definition` führt zu `definition: true`. Ein `identifier` in einem Ausdruck führt zu einem `Reference`-Knoten mit `definition: false`.

### **3. Der Vertrag der Expliziten Konnektivität**

**PRINZIP:** Abhängigkeiten zu anderen Modulen müssen als explizite `Import`-Knoten materialisiert werden.
**ANWEISUNG:** Ein AST-Knoten wie `import_statement` **MUSS** zu einem `SemanticNode` vom `NodeKind::Import` transformiert werden. Dessen Metadaten **MÜSSEN** den `source_path` der Abhängigkeit enthalten.

### **4. Der Vertrag der Aufruf-Abstraktion**

**PRINZIP:** Eine Funktions- oder Methoden-Invocation muss als distinkter `Call`-Knoten repräsentiert werden, der sein Ziel strukturell beschreibt.
**ANWEISUNG:** Ein AST-Knoten wie `call_expression` **MUSS** zu einem `SemanticNode` vom `NodeKind::Call` transformiert werden. Die Transformation **MUSS** das Aufrufziel (`callee`) in seine strukturellen Bestandteile zerlegen und als `target_parts` (array<string>) in den Metadaten speichern (z.B. `a.b()` -> `["a", "b"]`). Der `target_name` enthält nur den letzten Teil des Pfades.

### **5. Der Vertrag der Quellentreue**

**PRINZIP:** Jeder `SemanticNode` muss eine präzise Verbindung zu seinem Ursprung im Quellcode haben.
**ANWEISUNG:** Für jeden erstellten `SemanticNode` **MUSS** das `range`-Feld mit den exakten Start/End-Koordinaten aus dem `tree-sitter`-Knoten befüllt werden. Zusätzlich **MUSS** der `content_hash` aus dem Quelltext innerhalb dieses `range` berechnet und gespeichert werden.

### **6. Der Vertrag der Gesteuerten Traversierung**

**PRINZIP:** Die Art und Weise, wie der `ParseEngine` den AST nach der Verarbeitung eines Knotens weiter durchläuft, muss explizit und deklarativ sein, um die Erzeugung von semantisch irrelevanten "Wrapper-Knoten" zu verhindern.
**ANWEISUNG:** Jede `transform!`-Registrierung **MUSS** eine `TraversalStrategy` (z.B. `CreateAndContain`, `Dissolve`, `Terminal`) angeben. Diese Strategie diktiert das Verhalten des `ParseEngine` und ist ein integraler Bestandteil der Regeldefinition.

---

## III. Die Finale, Detaillierte und Vollständige ECS Knoten- & Kanten-Taxonomie

### A. Knoten-Taxonomie (`NodeKind`)

#### **I. Struktur & Organisation (Das Grundgerüst)**

| NodeKind | Zweck | Beispielhafte AST-Knoten | Kritische Metadaten (Obligatorisch) |
| :--- | :--- | :--- | :--- |
| **`Module`** | Die oberste Organisationseinheit (Datei, Namespace, Paket). | `source_file`, `namespace_declaration`, `package_clause`. | `language` (string), `file_path` (string), `is_file_scoped` (bool). |
| **`ImplementationBlock`** | Ein struktureller Container für Methoden-Implementierungen. | `impl_item`, `class_body` (als Container). | `target_type` (string), `is_trait_impl` (bool), `is_extension` (bool). |
| **`ClassDef`** | Die Definition eines komplexen, benannten Datentyps. | `class_declaration`, `struct_item`, `record_declaration`. | `base_types` (array<string>), `is_struct` (bool), `is_generic` (bool). |
| **`InterfaceDef`** | Die Definition einer reinen Schnittstelle (Protokoll, Trait). | `protocol_declaration`, `trait_item`, `interface_declaration`. | `is_trait` (bool). |
| **`EnumDef`** | Die Definition eines Aufzählungstyps. | `enum_declaration`, `enum_item`. | `underlying_type` (string), `is_flags` (bool). |
| **`TypeAlias`** | Die Definition eines Alias für einen existierenden Typ. | `type_alias_statement`, `typedef`. | `aliased_type` (string), `is_generic` (bool), `is_newtype` (bool). |
| **`TypeParameters`** | Die Liste generischer Typparameter (`<T, U>`). | `type_parameter_list`, `generic_parameters`. | `is_variadic` (bool). |
| **`Block`** | Ein struktureller Container für Code (z.B., `{...}`). | `compound_statement`, `block`, `scope_block`. | `scope_level` (int), `condition_pragma` (string). |
| **`PackageDef`** | Die Definition eines Pakets oder Bibliotheksmoduls. | `package_declaration`, `module_def`. | `package_manager` (string), `version` (string), `repository` (string). |
| **`TraitDef`** | Die Definition eines Traits oder Protokolls mit assoziierten Typen. | `trait_definition`, `protocol_def`. | |
| **`Collection`** | Ein **rein syntaktischer Gruppierungs-Knoten** (z.B. Parameterliste). Wird typischerweise von der `ParseEngine` aufgelöst und erscheint nicht im finalen Graphen, es sei denn, die Gruppierung selbst hat eine Typ-Semantik (z.B. Tupel). | `parameters`, `argument_list`, `use_list` | `is_tuple` (bool) |

#### **II. Funktionen & Logik (Die Ausführung)**

| NodeKind | Zweck | Beispielhafte AST-Knoten | Kritische Metadaten (Obligatorisch) |
| :--- | :--- | :--- | :--- |
| **`FunctionDef`** | Die Definition einer benannten, aufrufbaren Routine. | `function_definition`, `fn_item`. | `throws` (array<string>). |
| **`MethodDef`** | Eine an einen Typ gebundene Funktion. | `method_declaration`, `impl_item` function. | `receiver_type` (string). |
| **`PropertyDef`** | Die Definition einer Eigenschaft mit Zugriffsmethoden. | `property_declaration`, `accessor_declaration`. | `property_type` (string), `is_auto_property` (bool), `is_computed` (bool). |
| **`Parameter`** | Die Definition eines formalen Arguments. | `parameter_declaration`, `typed_parameter`. | `param_type` (string), `default_value` (string). |
| **`Argument`** | Ein tatsächlicher Wert, der an einen Aufruf übergeben wird. | `argument`, `argument_list_item`. | `value_expression` (string), `is_named` (bool), `name` (string, falls benannt), `is_splat` (bool). |
| **`Call`** | Eine Invokation einer Funktion oder Methode. | `call_expression`, `invocation_expression`. | `target_name` (string), `is_await` (bool). |
| **`Instantiation`** | Die Erzeugung einer Instanz eines Typs. | `new_expression`, `object_creation_expression`. | `target_type` (string), `initializer_content` (string). |
| **`Expression`** | Eine Berechnung, die einen Wert ergibt. | `binary_expression`, `unary_expression`. | `operator` (string), `is_parenthesized` (bool). |
| **`Literal`** | Ein direkter Wert (Zahl, String, Boolean). | `string_literal`, `integer_literal`. | `value_type` (string), `raw_value` (string). |
| **`LambdaDef`** | Die Definition einer anonymen Funktion. | `lambda_expression`, `closure`. | `parameters` (array<object>), `return_type` (string), `is_capturing` (bool). |
| **`OperatorDef`** | Die Definition eines überladenen Operators. | `operator_declaration`. | `operator_symbol` (string), `operand_types` (array<string>), `return_type` (string). |

#### **III. Daten & Variablen (Der Zustand)**

| NodeKind | Zweck | Beispielhafte AST-Knoten | Kritische Metadaten (Obligatorisch) |
| :--- | :--- | :--- | :--- |
| **`VariableDef`** | Die Definition einer lokalen oder globalen Variable. | `let_declaration`, `variable_declarator`. | `variable_type` (string), `initializer` (string). |
| **`Field`** | Ein Mitglied eines komplexen Typs. | `field_declaration`. | `field_type` (string), `access_modifier` (string). |
| **`Assignment`** | Eine Operation, die einen Wert zuweist oder aktualisiert. | `assignment_expression`. | `operator` (string), `source_expression` (string), `target_access_type` (string: `"write"`, `"read_write"`). |
| **`Reference`** | Eine Lesezugriffs-Verwendung eines existierenden, unqualifizierten Namens. Ein `tree-sitter`-Knoten vom Typ `identifier` im Lese-Kontext wird zu diesem `NodeKind`. | `identifier` (im Lese-Kontext) | `access_type`: `"read"` (default). |
| **`MemberAccess`** | Der Zugriff auf ein Mitglied eines Objekts, Structs oder Namespace. | `attribute`, `field_expression`, `.` | `operator`: `"."`, `"->"`, `"?."` (optional chaining). |
| **`IndexAccess`** | Der Zugriff auf ein Element einer Sammlung über einen Index. | `subscript_expression`, `[]` | `is_slice` (bool). |
| **`Type`** | Ein Verweis auf einen Typ. | `type_specifier`, `type_identifier`. | `type_constructor`: `"base"`, `"generic"`, `"function"`, `"union"`, `"intersection"`, `"tuple"`. |
| **`TypeParameter`** | Ein Platzhalter für einen generischen Typ oder eine Lifetime. | `type_parameter`. | `variance` (string: `"invariant"`, `"covariant"`, `"contravariant"`), `constraints` (array<string>), `default_type` (string). |
| **`ArrayDef`** | Die Definition eines Array- oder Collection-Typs. | `array_declaration`, `list_literal`. | `element_type` (string), `is_fixed_size` (bool), `length` (int). |
| **`TupleDef`** | Die Definition eines Tupel-Typs. | `tuple_type`. | `is_named` (bool). |
| **`DeleteStatement`** | Beendet die Lebensdauer einer Variable oder gibt eine Ressource frei. | `delete_statement`, `unset`, `delete`. | `target_name` (string), `is_freeing_memory` (bool). |

#### **IV. Kontrollfluss & Fehlerbehandlung (Der Logikfluss)**

| NodeKind | Zweck | Beispielhafte AST-Knoten | Kritische Metadaten (Obligatorisch) |
| :--- | :--- | :--- | :--- |
| **`IfStatement`** | Bedingte Verzweigung. | `if_statement`. | `condition` (string), `is_ternary` (bool). |
| **`MatchExpression`** | Mehrweg-Verzweigung. | `switch_statement`, `match_expression`. | `switched_on_value` (string), `is_exhaustive` (bool). |
| **`Loop`** | Wiederholte Ausführung. | `for_statement`, `while_statement`. | `loop_type` (string), `condition` (string), `initializer` (string). |
| **`Case`** | Ein einzelner Fall in einem Switch/Match. | `case_clause`, `match_arm`. | `pattern` (string), `guard_condition` (string), `is_default` (bool). |
| **`ControlFlow`** | Unstrukturierter Kontrollfluss. | `break_statement`, `continue_statement`. | `flow_type` (string), `target_label` (string). |
| **`ReturnStatement`** | Gibt einen Wert zurück und transferiert die Kontrolle. | `return_statement`. | `returned_expression` (string). |
| **`ThrowStatement`** | Löst eine Ausnahme aus. | `throw_statement`, `raise_statement`. | `exception_expression` (string), `is_rethrow` (bool). |
| **`TryBlock`** | Ausnahmebehandlungs-Block. | `try_statement`. | `resources` (array<string>). |
| **`CatchClause`** | Fängt eine Ausnahme. | `catch_clause`. | `exception_type` (string), `exception_variable` (string). |
| **`ResourceBlock`** | Ressourcenmanagement-Block. | `using_statement`, `defer_statement`. | `resource_declaration` (string). |
| **`AssertStatement`** | Eine Laufzeit-Zusicherung. | `assert_statement`. | `condition` (string), `message` (string). |
| **`Label`** | Eine benannte Anweisung. | `labeled_statement`. | `label_name` (string). |

#### **V. Konnektivität & FFI (Die Ökosystem-Brücke)**

| NodeKind | Zweck | Beispielhafte AST-Knoten | Kritische Metadaten (Obligatorisch) |
| :--- | :--- | :--- | :--- |
| **`Import`** | Bringt externe Abhängigkeiten in den Geltungsbereich. | `import_statement`, `use_declaration`. | `source_path` (string), `alias` (string), `is_wildcard` (bool). |
| **`Export`** | Macht lokale Symbole sichtbar. | `export_statement`. | `is_default` (bool), `is_reexport` (bool). |
| **`Directive`** | Compiler/Präprozessor-Anweisungen. | `preproc_if`, `pragma`, `global_statement`. | `directive_type` (string), `condition` (string). |
| **`Annotation`** | Metadaten, die an Code angehängt sind. | `attribute_item`, `annotation`. | `annotation_name` (string), `arguments` (array<any>). |
| **`Linkage`** | Deklaration einer externen Verknüpfung. | `extern "C"`. | `linkage_type` (string), `scope` (string). |
| **`FFIBinding`** | Eine statische Struktur, die eine FFI-Verbindung definiert. | `JNIEXPORT`. | `host_language` (string), `host_name` (string), `abi` (string). |

#### **VI. UX & Domänenspezifisch (Die Schnittstelle)**

| NodeKind | Zweck | Beispielhafte AST-Knoten | Kritische Metadaten (Obligatorisch) |
| :--- | :--- | :--- | :--- |
| **`UIComponent`** | Ein visuelles Element (UI-Graph). | `jsx_element`, `element` (HTML). | `tag_name` (string), `id` (string), `attributes` (map<object>). |
| **`UIInteraction`** | Ein Event-Auslöser. | `event_binding`, `onClick`. | `event_type` (string), `handler_expression` (string), `is_bubbling` (bool). |
| **`StyleRule`** | Eine CSS-Regel oder ein Stil-Block. | `rule_set` (CSS), `style_element`. | `selector` (string), `properties` (map<string>), `media_query` (string). |
| **`Query`** | Eine Datenbank- oder Abfragesprachen-Anweisung. | `select_statement` (SQL). | `query_language` (string), `tables_accessed` (array<string>), `is_write_op` (bool), `parameters` (array<string>). |
| **`ExecutionBlock`** | Code, der in einer anderen Umgebung läuft. | `run_instruction` (Dockerfile). | `shell_type` (string), `script_content` (string), `is_background` (bool). |
| **`Comment`** | Dokumentation. | `comment`. | `comment_type` (string), `is_docstring` (bool), `attached_to` (string). |
| **`Template`** | Ein Template-String oder -Literal. | `template_string`. | `is_raw` (bool). |

#### **VII. Nebenläufigkeit & Asynchronität**

| NodeKind | Zweck | Beispielhafte AST-Knoten | Kritische Metadaten (Obligatorisch) |
| :--- | :--- | :--- | :--- |
| **`Await`** | Wartet auf eine asynchrone Operation. | `await`, `co_await`. | `awaited_expression` (string), `is_timeout` (bool). |
| **`Yield`** | Gibt einen Wert zurück, behält aber den Kontext. | `yield`, `co_yield`. | `yielded_expression` (string), `is_yield_from` (bool), `is_delegated` (bool). |
| **`Spawn`** | Startet einen unabhängigen Ausführungspfad. | `go`, `tokio::spawn`. | `spawned_function` (string), `is_detached` (bool), `thread_name` (string). |
| **`Channel`** | Ein Kommunikationskanal. | `chan`. | `channel_type` (string), `buffer_size` (int), `is_bidirectional` (bool). |
| **`Lock`** | Ein Synchronisations-Primitiv. | `mutex_lock`. | `lock_type` (string), `is_shared` (bool). |

#### **VIII. Low-Level & Unsichere Operationen**

| NodeKind | Zweck | Beispielhafte AST-Knoten | Kritische Metadaten (Obligatorisch) |
| :--- | :--- | :--- | :--- |
| **`UnsafeBlock`** | Ein Block, der unsichere Operationen erlaubt. | `unsafe { ... }`. | `language` (string), `reason` (string), `pinned` (bool). |
| **`PointerOp`** | Eine Operation, die Speicheradressen involviert. | `*ptr`, `&var`. | `operator` (string), `operand` (string), `is_raw` (bool). |
| **`SizeOf`** | Ermittelt die Speichergröße. | `sizeof(T)`. | `target_type` (string), `is_alignof` (bool). |
| **`Cast`** | Eine Typ-Konvertierung. | `static_cast`, `as`. | `source_type` (string), `target_type` (string), `is_unsafe` (bool). |

#### **IX. Metaprogrammierung & Generics**

| NodeKind | Zweck | Beispielhafte AST-Knoten | Kritische Metadaten (Obligatorisch) |
| :--- | :--- | :--- | :--- |
| **`MacroDef`** | Die Definition einer Code-generierenden Anweisung. | `macro_rules!`, `#define`. | `parameters` (array<string>), `body_text` (string), `is_hygienic` (bool). |
| **`MacroCall`** | Die Invokation einer Code-generierenden Anweisung. | `println!`. | `macro_name` (string), `arguments` (array<any>), `expansion_level` (int). |
| **`Constraint`** | Eine Einschränkung für einen generischen Typ. | `where T: Trait`. | `constrained_type` (string), `constraint_type` (string), `is_bounds` (bool). |

#### **X. Datenintegrität & Sicherheit**

**Hinweis:** Konzepte wie `TaintSource` und `TaintSink` sind keine primären `NodeKind`s, die während des Parsens erzeugt werden. Sie sind **Analyse-Ergebnisse**, die in einer nachgelagerten Analyse-Phase als ECS-Komponenten (z.B. `TaintMarker`) an bestehende Knoten (`VariableDef`, `Call`, etc.) angehängt werden.

### B. Kanten-Taxonomie (`EdgeType`)

| EdgeType | Richtung | Zweck |
| :--- | :--- | :--- |
| **`Contains`** | Parent -> Child | Strukturelle Verschachtelung. Die primäre Kante zum Aufbau der Hierarchie. |
| **`Defines`** | Declaration -> Identifier | Verbindet eine Definition mit ihrem Namen. |
| **`Accepts`** | Function -> Parameter | Verbindet eine aufrufbare Entität mit ihren formalen Parametern. |
| **`Returns`** | Function -> Type | Verbindet eine aufrufbare Entität mit ihrem Rückgabetyp. |
| **`AnnotatedBy`** | Code -> Annotation | Verbindet einen Knoten mit seinen Annotationen/Dekoratoren. |
| **`Calls`** | Caller -> Callee | Eine Verhaltens-Kante, die eine Aufrufstelle mit ihrer Zield-Definition verbindet. |
| **`Imports`** | Importer -> Importee | Eine Abhängigkeits-Kante, die eine Import-Anweisung mit dem importierten Modul verbindet. |
| **`References`** | Usage -> Definition | Die aufgelöste Kante, die eine Symbol-Verwendung mit ihrer Definition verbindet. |
| **`Implements`** | Implementer -> Interface | Verbindet einen Typ mit der von ihm implementierten Schnittstelle. |
| **`Embeds`** | Host Node -> Guest Module | **Polyglotte Injektion.** Verbindet einen Knoten in einer Host-Sprache mit dem `Module`-Knoten des eingebetteten Codes. |
| **`NextSibling`** | Sibling -> Sibling | **Sequenzielle Ordnung.** Verbindet einen Knoten mit seinem direkten Nachfolger im selben Block. |
| **`Instantiates`** | Generic Type -> Type Argument | **Generische Instanziierung.** Verbindet einen generischen Typ (z.B. `Vec`) mit seinen konkreten Typ-Argumenten (z.B. `String`). |
| **`ExpandsTo`** | MacroCall -> Virtual Root | **Makro-Expansion.** Verbindet einen `MacroCall`-Knoten (im Quelltext) mit der Wurzel eines **virtuellen Subgraphen**, der den expandierten Code repräsentiert. |


---

### **IV. Allgemeine Metadaten & Verhaltens-Flags**

    Diese Tabelle definiert einen standardisierten Satz von Metadaten-Schlüsseln. Diese werden in zwei Phasen befüllt:
    1.  **Phase 1 (Transformation):** Die `transform`-Funktionen befüllen diese Felder basierend auf der direkten AST-Analyse.
    2.  **Phase 2 (Anreicherung):** Nachgelagerte Analyse-Systeme (z.B. der Ownership Analyzer) können zusätzliche Metadaten hinzufügen oder bestehende verfeinern. Felder, die ausschließlich in Phase 2 befüllt werden, sind explizit gekennzeichnet.

### **1. Geltungsbereich, Sichtbarkeit & Export**

| Metadaten-Schlüssel | Typ | Anwendbar auf | Bedeutung & Klärung von Mehrdeutigkeiten |
| :--- | :--- | :--- | :--- |
| **`visibility`** | `string` | `FunctionDef`, `MethodDef`, `ClassDef`, `Field`, `EnumDef` | Definiert explizite Sichtbarkeit. Werte: `"public"`, `"private"`, `"protected"`, `"internal"`, `"fileprivate"`. **Mehrdeutigkeit:** Python fehlt dieses Schlüsselwort; hier wird die Namenskonvention (`_`, `__`) abgebildet. `public` ist der Standard, falls nicht anders angegeben. |
| **`is_exported`** | `boolean` | `FunctionDef`, `VariableDef`, `ClassDef` | `true`, wenn das Symbol explizit aus seinem Modul exportiert wird (z.B. `export` in JS/TS, `pub` in Rust auf Modulebene). Dies ist spezifischer als `visibility: "public"`. |
| **`is_static`** | `boolean` | `MethodDef`, `Field`, `VariableDef` | `true`, wenn das Mitglied zum Typ selbst gehört, nicht zu einer Instanz. **Mehrdeutigkeit:** In C kann `static` auch "interne Verknüpfung" bedeuten. In diesem Fall muss zusätzlich `visibility: "internal"` gesetzt werden. |
| **`is_implicit`** | `boolean` | Any | `true`, wenn der Knoten nicht direktem Quellcode entspricht, sondern vom Compiler generiert wird (z.B. ein Standard-Konstruktor). Solche Knoten haben kein `range`. |

### **2. Vererbung & Polymorphismus**

| Metadaten-Schlüssel | Typ | Anwendbar auf | Bedeutung & Klärung von Mehrdeutigkeiten |
| :--- | :--- | :--- | :--- |
| **`is_abstract`** | `boolean` | `ClassDef`, `MethodDef` | `true`, wenn eine Klasse nicht instanziiert oder eine Methode nicht implementiert werden kann und von einer Unterklasse implementiert werden muss. |
| **`is_final`** | `boolean` | `ClassDef`, `MethodDef` | `true`, wenn von einer Klasse nicht geerbt oder eine Methode nicht überschrieben werden kann (z.B. `final` in Java/C++, `sealed` in C# für Klassen). |
| **`is_virtual`** | `boolean` | `MethodDef` | `true`, wenn eine Methode explizit zum Überschreiben vorgesehen ist. In Sprachen, in denen Methoden standardmäßig virtuell sind (Java, Python), kann dies weggelassen oder auf `true` gesetzt werden. |
| **`is_override`** | `boolean` | `MethodDef` | `true`, wenn eine Methode explizit eine Methode aus einer Basisklasse oder Schnittstelle überschreibt (z.B. `@Override` in Java, `override` in C#). |

### **3. Ausführungs- & Verhaltens-Modifikatoren**

| Metadaten-Schlüssel | Typ | Anwendbar auf | Bedeutung & Klärung von Mehrdeutigkeiten |
| :--- | :--- | :--- | :--- |
| **`is_async`** | `boolean` | `FunctionDef`, `MethodDef`, `LambdaDef`, `Block` | `true`, wenn die Funktion/der Block die Ausführung unterbrechen kann, um auf I/O zu warten. |
| **`is_generator`** | `boolean` | `FunctionDef`, `MethodDef`, `LambdaDef` | `true`, wenn die Funktion Werte über `yield` zurückgibt und ihren Zustand zwischen den Aufrufen beibehält. |
| **`is_unsafe`** | `boolean` | `Block`, `FunctionDef`, `MethodDef` | `true`, wenn der Code-Block oder die Funktion Operationen durchführt, die die Typ- oder Speichersicherheit des Systems umgehen. |
+| **`is_mutable`** | `boolean` | `VariableDef`, `Parameter` | `true`, wenn eine Variable explizit als veränderbar deklariert ist (z.B. `mut` in Rust). Der Standardwert ist `false` (unveränderlich).

### **4. Speicherverwaltung, Ownership & Referenzen**

| Metadaten-Schlüssel | Typ | Anwendbar auf | Bedeutung & Klärung von Mehrdeutigkeiten |
| :--- | :--- | :--- | :--- |
| **`ownership_action`** | `string` | `Assignment`, `Argument`, `ReturnStatement` | Beschreibt die Ownership-Semantik einer Operation. Werte: `"move"`, `"copy"`, `"borrow_immutable"`, `"borrow_mutable"`. **Hinweis: Dieses Feld wird von der nachgelagerten Ownership-Analyse (FRD-002) befüllt und wird nicht von der initialen `transform`-Funktion erwartet.** |
| **`lifetime`** | `string` | `VariableDef`, `TypeParameter` | Speichert den Namen einer expliziten Lebensdauer (z.B. `'a` in Rust). |
| **`is_raw_pointer`** | `boolean` | `Type`, `VariableDef` | `true`, wenn der Typ ein roher Zeiger ist (`*const T`, `*mut T` in Rust; `int*` in C). Dies unterscheidet ihn von Smart Pointern oder sicheren Referenzen. |
| **`is_reference`** | `boolean` | `Type`, `VariableDef`, `Parameter` | `true`, wenn der Typ eine sichere Referenz ist (`&T` in Rust, `ref` in C#). |
| **`memory_management`** | `string` | `ClassDef`, `Module` | Gibt das primäre Speicherverwaltungsmodell an. Werte: `"gc"` (Garbage Collection), `"arc"` (Automatic Reference Counting), `"manual"`, `"ownership"`. |

### **5. Konstruktoren, Destruktoren & Konvertierungen**

| Metadaten-Schlüssel | Typ | Anwendbar auf | Bedeutung & Klärung von Mehrdeutigkeiten |
| :--- | :--- | :--- | :--- |
| **`is_constructor`** | `boolean` | `MethodDef` | `true`, wenn die Methode eine Instanz ihres Typs initialisiert. |
| **`is_destructor`** | `boolean` | `MethodDef` | `true`, wenn die Methode beim Zerstören der Instanz aufgerufen wird, um Ressourcen freizugeben. |
| **`is_type_conversion`** | `boolean` | `MethodDef` | `true`, wenn eine Methode eine implizite oder explizite Typkonvertierung definiert (z.B. `operator bool()` in C++). |
| **`is_receiver`** | `boolean` | `Parameter` | `true`, wenn dieser Parameter der Empfänger eines Methodenaufrufs ist (z.B. `self`, `this`). |

### **6. Polyglotte Injektion & Metaprogrammierung**

| Metadaten-Schlüssel | Typ | Anwendbar auf | Bedeutung & Klärung von Mehrdeutigkeiten |
| :--- | :--- | :--- | :--- |
| **`injection_host`** | `boolean` | `Literal`, `UIComponent` | `true`, wenn dieser Knoten Code in einer anderen Sprache enthält (z.B. ein String-Literal mit SQL). |
| **`injection_heuristic`** | `string` | `Literal`, `UIComponent` | Ein Hinweis für die Engine, wie die eingebettete Sprache zu identifizieren ist (z.B. `tag_name:script`, `variable_name:sql_query`). |
| **`injected_language`** | `string` | `Literal`, `UIComponent` | Die aufgelöste Sprache des eingebetteten Inhalts (z.B. `sql`, `javascript`). Wird von der Analyse-Engine nach Anwendung der Heuristik gesetzt. |
| **`is_compile_time`** | `boolean` | `MacroCall`, `FunctionDef` | `true`, wenn der Code zur Kompilierzeit ausgeführt wird (z.B. `comptime` in Zig, `const fn` in Rust). |


### **7. Der Vertrag der Werkzeug-Nutzung (Toolkit Mandate)**

    **PRINZIP:** Kritische, wiederkehrende Logik, die für die Einhaltung der Verträge erforderlich ist, muss zentralisiert und standardisiert werden, um Fehler und Inkonsistenzen zu vermeiden.
    **ANWEISUNG:** Eine `transform`-Funktion **MUSS** die bereitgestellten Hilfsfunktionen aus dem "Standard Transform Toolkit" (`src/transform/base.rs`) verwenden, wo immer dies anwendbar ist. Insbesondere:
    1.  Die Erstellung eines `SemanticNode` **MUSS** über den `new_semantic_node()`-Helfer erfolgen.
    2.  Die Erstellung von qualifizierten Namen **MUSS** über den `construct_qualified_name()`-Helfer erfolgen.
    3.  Die Extraktion von Aufrufzielen (`callee`) **MUSS** über den `extract_structured_callee()`-Helfer erfolgen.
    Die direkte, manuelle Neuimplementierung dieser Logik innerhalb einer `transform`-Funktion ist strengstens untersagt.

### **8. Virtuelle Knoten & Provenienz (Hybrid-Parsing)**

Diese Felder sind **zwingend erforderlich**, wenn Knoten durch externe Engines (Clang, Rust Analyzer) erzeugt werden, die nicht direktem Text im Quellfile entsprechen (z.B. Makro-Expansionen, Template-Instanziierungen).

| Metadaten-Schlüssel | Typ | Anwendbar auf | Bedeutung & Klärung von Mehrdeutigkeiten |
| :--- | :--- | :--- | :--- |
| **`is_virtual`** | `boolean` | Alle `NodeKind`s | `true`, wenn dieser Knoten **nicht** physisch im Quelltext existiert, sondern durch eine semantische Engine generiert wurde (z.B. das Ergebnis einer Makro-Expansion). Solche Knoten haben oft ein `range`, das auf den Aufruf zeigt, aber einen `content_hash` des generierten Codes. |
| **`provenance`** | `string` | Alle `NodeKind`s | Die ID oder der Name der Definition, die diesen Knoten erzeugt hat (z.B. der Name des Makros oder Templates). |
| **`expansion_depth`** | `int` | Alle `NodeKind`s | Die Rekursionstiefe der Expansion. Dient als Schutzmechanismus gegen unendliche Rekursion (siehe Regel VII.3). |
---

## V. Obligatorische Regeln für Syntaktische Präzision (Reversibilität)

Um eine bit-perfekte Rekonstruktion zu gewährleisten, gelten die folgenden Regeln für **ALLE** Transformationen.

1.  **Trivia-Erfassung:** Jeder `SemanticNode` **MUSS** nicht-semantische Token (Leerraum, Kommentare) im `trivia`-Metadatenobjekt erfassen.
2.  **Syntaktischer Zucker:** Wenn ein Konstrukt mehrere gültige syntaktische Formen hat (z.B. `i++` vs. `i+=1`), **MUSS** die spezifische Form in `metadata.syntactic_form` aufgezeichnet werden.
3.  **Explizite Gruppierung:** `parenthesized_expression`-Knoten sind Komposita. Sie erzeugen keinen separaten semantischen Knoten. Stattdessen erhält der Kind-Knoten `metadata.is_parenthesized = true`.
4.  **Operator-Vorrang:** Für `Expression`-Knoten müssen `precedence` (int) und `associativity` (string) in den Metadaten gespeichert werden, um eine korrekte Rekonstruktion ohne redundante Klammern zu gewährleisten.
5.  **Rein Syntaktische Knoten:** Knoten, die rein syntaktisch sind (z.B. `;`, `{`, `}`, `if`, `else`-Schlüsselwörter), werden **NICHT** auf semantische Knoten abgebildet. Ihre Existenz wird durch den `NodeKind` impliziert.
6.  **Rohtext-Erfassung (Prinzip der minimalen Erfassung):** Das `raw_text`-Feld dient als "Escape Hatch", um eine perfekte Quelltext-Rekonstruktion in Fällen zu gewährleisten, in denen die exakte syntaktische Form nicht allein aus den strukturierten, semantischen Feldern des `SemanticNode` (wie `name`, `raw_value`, `operator`, etc.) eindeutig ableitbar ist.
Eine `transform`-Funktion **SOLLTE** es vermeiden, `raw_text` zu befüllen. Sie **MUSS** es jedoch befüllen, wenn eine der folgenden Bedingungen zutrifft:
    1.  **Syntaktischer Zucker mit semantischer Äquivalenz:** Der Quelltext verwendet eine spezifische Schreibweise, die bei einer generischen Rekonstruktion verloren gehen würde (z.B. `0xFF` vs. `255`, `1_000_000` vs. `1000000`, `f"string"` vs. `"string"`).
    2.  **Komplexe Literale:** Der Inhalt eines Literals ist selbst strukturiert oder enthält spezielle Escape-Sequenzen, die bei einer einfachen Rekonstruktion aus `raw_value` falsch interpretiert werden könnten (z.B. C++ `R"(...)"`, Python `"""..."""` mit beibehaltener Einrückung).
    3.  **Unmodellierte Ausdrücke:** Ein Ausdruck ist zu komplex oder sprachspezifisch, um ihn vollständig in untergeordnete `SemanticNode`s zu zerlegen, und wird daher als `Terminal` behandelt.
 Für einfache Identifier (`Reference`) und die meisten einfachen Literale (`Literal`) **DARF** `raw_text` **NICHT** gesetzt werden, da ihre textuelle Form direkt und eindeutig aus den Feldern `name` bzw. `raw_value` rekonstruierbar ist.

---

Das ist eine **exzellente Beobachtung**.

Die Antwort ist: **Jein.**

Wenn du `tree-sitter-graph` (TSG) verwendest, ändert sich die **Rolle** dieser Helper fundamental. Du schreibst keine manuellen Rust-Traversierungen mehr (`for child in node.children...`), da TSG das deklarativ erledigt.

Aber: **TSG ist keine Turing-vollständige Programmiersprache.** Es ist eine Mapping-DSL. Es kann keine SHA256-Hashes berechnen, keine komplexen Pfad-Normalisierungen durchführen und keine externen Datenbanken abfragen.

Daher müssen die "Helper" nun als **Global Functions (Native Bindings)** definiert werden, die du in die TSG-Laufzeitumgebung injizierst, damit deine `.tsg`-Dateien sie aufrufen können (z.B. `let hash = call_native("calculate_hash", node.text)`).

Hier ist die **radikal bereinigte und korrigierte Version von Abschnitt VI**, die genau diese Architektur widerspiegelt: Trennung zwischen TSG-Globals und externen Engine-Treibern.

---

## VI. Anhang: Globale Funktionen & Native Bindings (Die Brücke zur Engine)

Da wir für Tree-sitter-basierte Sprachen **`tree-sitter-graph` (TSG)** und für komplexe Sprachen **spezialisierte Engines** verwenden, entfällt die Notwendigkeit für manuelle Traversierungs-Helper.

Stattdessen definiert dieser Abschnitt die **Native Bindings**, die der `ParseEngine` bereitstellen muss.

### **A. Globale Funktionen für `tree-sitter-graph` (TSG)**

Diese Funktionen **MÜSSEN** in den Ausführungskontext jeder `.tsg`-Datei injiziert werden. Sie ermöglichen es der deklarativen Logik, komplexe Berechnungen durchzuführen.

| Funktions-Signatur (in TSG) | Rust-Implementierung (Backend) | Zweck & Vertrag |
| :--- | :--- | :--- |
| `calculate_hash(text: string) -> string` | `sha256(text)` | **MUSS** für das `content_hash`-Feld jedes Knotens aufgerufen werden. |
| `extract_trivia(node: syntax_node) -> json` | `extract_trivia_json(node)` | Extrahiert Whitespace/Kommentare vor dem Knoten. Gibt ein JSON-Objekt zurück, das direkt in `metadata.trivia` gespeichert wird. |
| `normalize_path(path: string) -> string` | `std::fs::canonicalize` | Wandelt relative Import-Pfade in absolute, kanonische Pfade um. |
| `scope_stack_push(current: string, name: string) -> string` | `format!("{}::{}", current, name)` | Konstruiert den qualifizierten Namen für verschachtelte Definitionen. |
| `generate_uuid() -> string` | `Uuid::new_v4()` | Erzeugt eine eindeutige ID für virtuelle Knoten, die keinen stabilen Pfad haben. |

**Beispiel-Nutzung in TSG:**
```scheme
(function_definition
  name: (identifier) @name
  body: (block) @body
) @func {
  node @func.kind = "FunctionDef"
  node @func.content_hash = calculate_hash(node.text)
  node @func.trivia = extract_trivia(@func)
}
```

### **B. Treiber für Externe Semantische Engines (Non-TSG)**

Für Sprachen, die **nicht** über Tree-sitter geparst werden, sind diese Funktionen die **alleinigen Einstiegspunkte**. Die `ParseEngine` delegiert die gesamte Datei an diese Funktionen.

| Funktion | Engine | Zuständigkeit |
| :--- | :--- | :--- |
| **`resolve_clang_translation_unit(path, args)`** | `libclang` | Parst C/C++/ObjC/CUDA. Muss den gesamten AST traversieren, Makros expandieren und einen vollständigen `SemanticGraph` zurückgeben. |
| **`resolve_rust_crate(path)`** | `rust-analyzer` | Parst Rust-Code. Muss `ra_ap_syntax` nutzen, Makros expandieren, Traits auflösen und den Graphen zurückgeben. |
| **`resolve_wasm_binary(bytes)`** | `walrus` | Parst WebAssembly (`.wasm`/`.wat`). Muss den Bytecode in einen CFG (Control Flow Graph) umwandeln und als `SemanticGraph` zurückgeben. |

---

## **VII. Integration Externer Semantischer Engines (Hybrid-Parsing)**

Dieser Abschnitt definiert die Regeln für Sprachen (C, C++, Objective-C, CUDA, Rust), bei denen eine rein syntaktische Analyse (Tree-sitter) unzureichend ist und stattdessen eine semantische Engine (`libclang`, `ra_ap_syntax`) verwendet wird.

### **1. Das Prinzip des Semantischen Primats**

**DIE REGEL:** Wenn eine externe Engine (z.B. Clang) verwendet wird, ist deren **semantische Sicht** (Cursor/AST) die Quelle der Wahrheit für die Erstellung von `SemanticNode`s, nicht der rohe Text-Token-Strom.
**ANWEISUNG:**
*   **Clang:** `CXCursor` ist die primäre Iterationseinheit. `CXCursor_UnexposedExpr` **MUSS** aufgelöst werden, indem die Kinder traversiert werden. Wenn keine Kinder existieren, wird ein Fallback auf `Expression` mit Rohdaten durchgeführt.
*   **Rust:** `ra_ap_syntax::ast` (Typed AST) ist die primäre Iterationseinheit.

### **2. Der Vertrag der Makro-Dualität (Struktur vs. Semantik)**

**PRINZIP:** Um sowohl statische Analyse (Linting) als auch tiefe semantische Analyse (Taint Tracking) zu ermöglichen, müssen Makros sowohl als Aufruf als auch als expandierter Code repräsentiert werden.
**ANWEISUNG:**
1.  Der Makro-Aufruf im Quelltext **MUSS** als `MacroCall` (physischer Knoten, `is_virtual: false`) abgebildet werden.
2.  Die Engine **MUSS** die Expansion anstoßen (Clang: `getCursorReferenced`/Traversal; Rust: `ctx.sema.expand`).
3.  Das Ergebnis der Expansion **MUSS** als separater Subgraph aus **virtuellen Knoten** (`is_virtual: true`) abgebildet werden.
4.  Der `MacroCall` **MUSS** über eine `ExpandsTo`-Kante mit der Wurzel des virtuellen Subgraphen verbunden werden.

### **3. Der Vertrag der Expansions-Begrenzung**

**PRINZIP:** Semantische Engines können unendliche Graphen erzeugen (rekursive Makros/Templates).
**ANWEISUNG:** Der Generator **MUSS** ein hartes Limit für die Expansions-Tiefe implementieren (Empfehlung: 128 Ebenen). Wird dieses Limit erreicht, **MUSS** die Expansion abgebrochen und der `MacroCall` mit `metadata.expansion_error = "depth_limit_exceeded"` markiert werden.

### **4. Sprachspezifische Mapping-Mandate**

#### **A. Clang (C / C++ / ObjC / CUDA)**
*   **`CXCursor_TranslationUnit`** -> **`Module`**.
*   **`CXCursor_InclusionDirective`** -> **`Import`** (Der `source_path` ist der aufgelöste Pfad der Header-Datei).
*   **Implizite Knoten:** Clang generiert Cursors für implizite Konstruktoren/Casts. Diese **MÜSSEN** als `SemanticNode` mit `is_implicit: true` und `range: null` (oder Range des Aufrufs) übernommen werden.
*   **Objective-C:** `OBJC_MESSAGE_EXPR` **MUSS** zu `Call` normalisiert werden (`target_name` ist der Selektor).
*   **CUDA:** `CUDA_KERNEL_CALL_EXPR` (`<<<...>>>`) **MUSS** zu `Call` transformiert werden, wobei die Grid-Konfiguration als spezielles Argument oder Metadatum erfasst wird.

#### **B. Rust Analyzer (Rust)**
*   **`ast::MacroRules` vs. `ast::MacroDef`:** Beide **MÜSSEN** zu `MacroDef` gemappt werden, unterschieden durch `syntactic_form` ("declarative" vs. "procedural").
*   **`ast::Impl`:** **MUSS** zu `ClassDef` (mit `is_extension: true`) gemappt werden. Methoden innerhalb des `impl`-Blocks erhalten eine `Contains`-Kante vom `ClassDef`.
*   **Patterns:** Komplexe Patterns (`ast::TuplePat`, `ast::StructPat`) in `let` oder `match` **MÜSSEN** rekursiv in `LogicContainer` (für Destructuring) und `VariableDef` (für Bindungen) aufgelöst werden.

#### **C. Walrus (WebAssembly)**

**Engine:** `walrus` (Rust Crate)
**Kontext:** Analyse von `.wasm` Binärdateien (und `.wat` via Konvertierung).

*   **`walrus::Module`** -> **`Module`**
    *   `is_file_scoped: true`
    *   `language: "wasm"`
*   **`walrus::Function`** -> **`FunctionDef`**
    *   `name`: Wenn im "Name Section" vorhanden, sonst `func_$index`.
    *   `return_type`: Signatur-Ergebnis (z.B. `i32`).
    *   `parameters`: Signatur-Parameter.
*   **`walrus::Export`** -> **`Export`**
    *   Verbindet den exportierten Namen mit der internen `FunctionDef` oder `Global`.
*   **`walrus::Import`** -> **`Import`**
    *   `source_path`: `module`-Feld des Imports (z.B. `"env"`).
    *   `alias`: `name`-Feld des Imports.
*   **`walrus::Instr` (Instruktionen)** -> **`Call` / `Expression`**
    *   **`Call` / `CallIndirect`**: -> **`Call`**.
    *   **`Const`**: -> **`Literal`**.
    *   **`Binop` (Add, Sub...)**: -> **`Expression`**.
    *   **`LocalGet` / `GlobalGet`**: -> **`Reference`**.
    *   **`LocalSet` / `GlobalSet`**: -> **`Assignment`**.
*   **Speicher & Tabellen:**
    *   **`walrus::Memory`** -> **`VariableDef`** (Typ: `WasmMemory`, `is_static: true`).
    *   **`walrus::Table`** -> **`VariableDef`** (Typ: `WasmTable`).



### **5. Umgang mit "Unexposed" und "Unknown" Knoten**

**PRINZIP:** Semantische Engines verbergen oft komplexe C++-Konstrukte oder Compiler-Intrinsics hinter generischen Knotentypen (z.B. `UnexposedExpr` in Clang).
**ANWEISUNG:**
1.  **Traversierung:** Versuche immer, die Kinder eines solchen Knotens zu besuchen. Oft verbirgt sich die wahre Logik dort.
2.  **Fallback:** Wenn keine Kinder existieren und der Typ nicht auflösbar ist, **MUSS** ein generischer `Expression`-Knoten erstellt werden. Das Feld `raw_text` **MUSS** zwingend befüllt werden, und `metadata.engine_kind` (z.B. "CXCursor_UnexposedExpr") **SOLLTE** zur Diagnose gespeichert werden.


---

## VIII. Versions-Bereichs-Inferenz & Semantische Constraint-Propagierung

### Prinzip: Syntax als Versions-Orakel

**DIE REGEL:** Wenn ein `SemanticNode` aus einem AST-Konstrukt erstellt wird, das nur in bestimmten Sprachversionen gültig ist, **MUSS** der Knoten mit seinem **gültigen Versionsbereich** annotiert werden. Dieser Bereich fungiert als **Constraint**, der propagiert werden kann, um Mehrdeutigkeiten aufzulösen und semantische Bedeutung zu bestimmen.

**BEGRÜNDUNG:** Viele Sprachkonstrukte haben versionsabhängige Semantik. Durch das Tracking, in welchen Versionen ein geparster Konstrukt gültig ist, können wir:
1. **Überladene Syntax disambiguieren** (z.B. `async` als Schlüsselwort vs. Identifier)
2. **Sprachversion inferieren** von Quellcode ohne explizite Deklaration
3. **Cross-Version-Kompatibilität validieren** in Analyse-Tools
4. **Constraint-Solving** durchführen, um zu bestimmen, welche Interpretation mehrdeutiger Syntax korrekt ist

---

### A. Versions-Bereichs-Metadaten (Obligatorisch)

**Für ALLE `SemanticNode`s:**

| Metadaten-Feld | Typ | Beschreibung | Beispiel |
|----------------|-----|--------------|----------|
| `valid_versions` | `VersionRange` | Semantische Versionen, in denen dieses Konstrukt gültig ist | `{"min": "3.10", "max": null}` (≥3.10) |
| `version_origin` | `string` | Quelle des Versions-Constraints | `"syntax"`, `"inference"`, `"file_pragma"` |
| `version_conflicts` | `array<string>` | Andere NodeKinds, mit denen dies in anderen Versionen kollidiert | `["identifier_async"]` (wenn `async_keyword`) |

**VersionRange-Struktur:**
```json
{
  "min": "3.10",           // Minimale Version (inklusiv), null = keine untere Grenze
  "max": "3.12",           // Maximale Version (exklusiv), null = keine obere Grenze
  "deprecated_in": "4.0",  // Version, in der Konstrukt deprecated wurde (optional)
  "removed_in": "5.0"      // Version, in der Konstrukt entfernt wurde (optional)
}
```

---

### B. Regeln zur Versions-Bereichs-Zuweisung

#### Regel 1: Direkte Syntax-Zuordnung

**Wenn ein tree-sitter-Knoten auf ein versionsspezifisches Konstrukt abgebildet wird:**

**Beispiel (Python match-Statement - hinzugefügt in 3.10):**
```javascript
// In Transformationslogik
match_statement -> {
  NodeKind: "MatchExpression",
  metadata: {
    valid_versions: {
      min: "3.10",
      max: null
    },
    version_origin: "syntax"
  }
}
```

**Implementierung in Task 0:**
In `specs/versions.yaml`:
```yaml
version_constraints:
  match_statement:
    min_version: "3.10"
    rationale: "Structural pattern matching eingeführt in PEP 634"
    
  walrus_operator:
    min_version: "3.8"
    rationale: "Assignment expressions (`:=`) eingeführt in PEP 572"
```

**In Task 2 (USG-Mapping):**
```javascript
// grammar.js Kommentar oder Mapping-Dokumentation
match_statement: $ => seq(
  'match',
  field('value', $.expression),
  ':',
  field('cases', $.case_block)
),
// USG: MatchExpression, valid_versions: {"min": "3.10"}
```

---

#### Regel 2: Schlüsselwort vs. Identifier Disambiguierung

**Wenn ein Token je nach Version Schlüsselwort ODER Identifier sein kann:**

**Beispiel (Python `async` - Schlüsselwort ab 3.5+, Identifier vorher):**

```javascript
// Im Lexer/Parser
identifier: $ => {
  const base_pattern = /[a-zA-Z_][a-zA-Z0-9_]*/;
  // In Superset-Grammatik parsen beide
  return token(base_pattern);
},

async_keyword: $ => 'async',

// Versions-Constraint wird während Transformation hinzugefügt
async_keyword -> {
  NodeKind: "Keyword",
  metadata: {
    valid_versions: {
      min: "3.5",
      max: null
    },
    version_origin: "syntax",
    version_conflicts: ["identifier_named_async"]
  }
}

identifier (wenn text == "async") -> {
  NodeKind: "Reference", 
  metadata: {
    valid_versions: {
      min: null,
      max: "3.5"  // Nur als Identifier gültig < 3.5
    },
    version_origin: "syntax"
  }
}
```

**GLR-Behandlung:**
```javascript
// In grammar.js
conflicts: $ => [
  [$.async_keyword, $.identifier]  // Parser versucht beide
],
```

**Post-Parse-Auflösung:**
Nach dem Parsen mit GLR, verwende Versions-Inferenz, um ungültige Zweige zu beschneiden:
1. Parse erstellt ZWEI Knoten für `async`: Schlüsselwort-Interpretation + Identifier-Interpretation
2. Prüfe `valid_versions` von jedem
3. Wenn Datei andere 3.10+-Features hat → beschneide Identifier-Interpretation
4. Wenn Datei nur <3.5-kompatible Syntax hat → beschneide Schlüsselwort-Interpretation

---

#### Regel 3: Constraint-Propagierung (Inferenz)

**Wenn der Versionsbereich eines Knotens aus dem Kontext inferiert werden kann:**

**Szenario:** Funktion verwendet `match`-Statement → gesamte Funktion muss ≥3.10 sein

```javascript
// Propagierungsregel
FunctionDef node {
  // Enthält MatchExpression-Kind mit valid_versions.min = "3.10"
  
  // Propagiere Constraint aufwärts:
  metadata.valid_versions = {
    min: max(eigene_syntax_min, max(alle_kinder.valid_versions.min)),
    max: min(eigene_syntax_max, min(alle_kinder.valid_versions.max))
  }
  
  // Aktualisiere version_origin
  metadata.version_origin = "inference"
}
```

**Implementierung:**
```python
# In semantischer Analysephase (nach dem Parsen)
def propagate_version_constraints(node):
    """Propagiere Versions-Constraints von Kindern zu Eltern."""
    child_constraints = [child.metadata.valid_versions 
                         for child in node.children 
                         if child.metadata.valid_versions]
    
    if child_constraints:
        # Berechne Schnittmenge aller Constraints
        min_version = max(c.min for c in child_constraints if c.min)
        max_version = min(c.max for c in child_constraints if c.max) or None
        
        # Aktualisiere Knoten-Constraint (Schnittmenge)
        node.metadata.valid_versions = {
            "min": min_version,
            "max": max_version
        }
        node.metadata.version_origin = "inference"
```

---

### C. Anwendungsfälle

#### Anwendungsfall 1: Automatische Versionserkennung

**Input:** Python-Datei ohne `# -*- coding: utf-8 -*-`-Pragma

```python
# mystery.py
def process(data):
    match data:
        case {"type": "A", "value": v}:
            return v * 2
        case _:
            return 0
```

**Analyse:**
1. Parse Datei → erstellt `MatchExpression`-Knoten
2. Prüfe `valid_versions.min` → "3.10"
3. **Inferiere:** Diese Datei benötigt Python ≥3.10
4. **Berichte:** "Python 3.10+ erkannt (Pattern Matching verwendet)"

#### Anwendungsfall 2: Mehrdeutigkeits-Auflösung

**Input:** Mehrdeutige Syntax mit versionsabhängiger Bedeutung

```python
# C-Beispiel
A * B;
```

**Zwei Interpretationen:**
1. **Multiplikations-Ausdruck** (in allen Versionen gültig)
2. **Pointer-Deklaration** (nur in C gültig, nicht in C++ mit `auto`)

**Auflösung:**
1. Parser erstellt beide Knoten (GLR)
2. Knoten 1: `BinaryExpression` mit `valid_versions: {min: null, max: null}`
3. Knoten 2: `VariableDef` mit `valid_versions: {min: null, max: "C++03"}`
4. Prüfe Dateikontext: enthält `std::vector` (C++-Feature)
5. **Beschneide Knoten 2** (inkompatible Versionsrange)
6. **Wähle Knoten 1** als korrekte Interpretation

#### Anwendungsfall 3: Cross-Version-Kompatibilitätsprüfung

**Input:** Bibliothek, die angibt, Python 3.7-3.12 zu unterstützen

**Analyse:**
```python
# Scanne alle SemanticNodes
for node in semantic_graph.all_nodes():
    if node.metadata.valid_versions.min > "3.7":
        report_error(f"Feature {node} benötigt {node.metadata.valid_versions.min}, "
                     f"inkompatibel mit angegeben Support für 3.7")
```

**Beispiel-Fehler:**
```
FEHLER: library.py:42: match-Statement benötigt Python ≥3.10
        Angegebene Mindestversion: 3.7
        Aktion: Aktualisiere pyproject.toml `requires-python = ">=3.10"` oder entferne Feature
```

---

### D. Integration in bestehendes Protokoll

#### In Task 0 (Planungsphase)

**Hinzufügen zu `specs/versions.yaml`:**
```yaml
version_constraints:
  # Syntax-Level-Constraints
  match_statement:
    usg_nodekind: "MatchExpression"
    min_version: "3.10"
    max_version: null
    spec_reference: "PEP 634"
    
  walrus_operator:
    usg_nodekind: "Assignment"
    min_version: "3.8"
    spec_reference: "PEP 572"
    
  # Schlüsselwort-Konflikte
  async_as_keyword:
    usg_nodekind: "Keyword"
    min_version: "3.5"
    conflicts_with: "async_as_identifier"
    
  async_as_identifier:
    usg_nodekind: "Reference"
    max_version: "3.5"
```

#### In Task 2 (Strukturelles Skelett)

**Hinzufügen von Versions-Constraints zu USG_MAPPING.md:**
```markdown
| Tree-sitter Node | USG NodeKind | Gültige Versionen | Konflikte |
|------------------|--------------|-------------------|-----------|
| match_statement | MatchExpression | ≥3.10 | - |
| assignment_expression | Assignment | ≥3.8 (Walrus) | - |
| identifier "async" | Reference | <3.5 | async_keyword |
```

#### In Task 4 (Konfliktauflösung)

**Hinzufügen von versionsbasierter Konfliktauflösung:**
```javascript
// In CONFLICTS_LOG.md
Konflikt: async-Token-Mehrdeutigkeit
├─ Interpretation A: async_keyword (≥3.5)
├─ Interpretation B: identifier (alle Versionen)
└─ Auflösung: GLR mit Post-Parse-Versionsfilterung
   - Wenn Datei andere ≥3.5-Features enthält → A
   - Andernfalls → beide gültig (benötigt Laufzeit-Entscheidung)
```

#### In Task 6 (Verifikation)

**Hinzufügen von Versions-Validierungstest:**
```bash
# Test Versionsinferenz
echo "match x: case 1: pass" | tree-sitter parse --infer-version
# Erwartet: "Python 3.10+"

# Test Versionskonflikt
echo "async = 5" | tree-sitter parse --target-version 3.4
# Erwartet: Gültig

echo "async = 5" | tree-sitter parse --target-version 3.7
# Erwartet: Warnung - "async ist Schlüsselwort in 3.7+"
```

---

### E. Obligatorische Deliverables

**Task 0 Ergänzungen:**
1. `specs/versions.yaml` → Abschnitt `version_constraints`

**Task 2 Ergänzungen:**
1. `USG_MAPPING.md` → Spalte "Gültige Versionen" hinzufügen
2. `VERSION_CONSTRAINT_PROPAGATION.md` → Inferenzregeln dokumentieren

**Neues Verifikationsskript:**
```bash
# scripts/validate_version_constraints.py
#!/usr/bin/env python3
"""Validiere Versions-Constraint-Metadaten auf allen Knoten."""

def validate_constraints(semantic_graph):
    for node in semantic_graph.nodes:
        # Prüfe Metadaten vorhanden
        assert "valid_versions" in node.metadata
        
        # Prüfe Propagierungskorrektheit
        if node.children:
            child_min = max(c.metadata.valid_versions.min 
                           for c in node.children)
            assert node.metadata.valid_versions.min >= child_min
```

---

### F. GLR-Zweig-Behandlung

#### CST-zu-SemanticGraph-Transformation

**Transformationslogik für mehrdeutige Knoten:**

```python
def transform_ambiguous_node(cst_node):
    """
    Transformiere CST-Knoten mit mehreren GLR-Interpretationen.
    Filtere ungültige Zweige basierend auf Versions-Constraints.
    """
    if not cst_node.has_glr_branches():
        # Normaler Fall - keine Mehrdeutigkeit
        return create_semantic_node(cst_node)
    
    # Sammle alle gültigen Interpretationen
    valid_interpretations = []
    
    for branch in cst_node.glr_branches:
        # Prüfe auf ERROR-Knoten (ungültige Parse-Zweige)
        if branch.contains_error():
            continue  # Überspringe ungültige Zweige
        
        # Hole Versions-Constraint für diese Interpretation
        constraint = get_version_constraint(branch.node_type)
        
        # Prüfe Kompatibilität mit Dateikontext
        if is_compatible_with_file_context(constraint):
            valid_interpretations.append({
                'branch': branch,
                'constraint': constraint
            })
    
    # Entscheide basierend auf Anzahl gültiger Interpretationen
    if len(valid_interpretations) == 0:
        raise ParseError(f"Keine gültige Interpretation für {cst_node}")
    
    elif len(valid_interpretations) == 1:
        # Eindeutig aufgelöst
        return create_semantic_node(
            valid_interpretations['branch'],
            metadata={'valid_versions': valid_interpretations['constraint']}
        )
    
    else:
        # Echte Mehrdeutigkeit - benötigt Spezialbehandlung
        return create_ambiguous_semantic_node(valid_interpretations)
```

**Ergebnis:** SemanticGraph enthält normalerweise **EINEN** Knoten pro Konstrukt, annotiert mit Versions-Metadaten. Nur in seltenen Fällen bleibt ein `AmbiguousNode` bestehen.
