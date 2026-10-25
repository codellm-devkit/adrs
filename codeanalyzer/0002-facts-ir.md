---
status: proposed            # draft; the record catalog is to be refined against real output in step 0 of ADR 0001
date: 2026-10-15
decision-makers: [Rahul Krishna]
consulted: [CLDK maintainers, via the RFC]
informed: [codellm-devkit contributors]
scope: the backend-to-core contract of ADR 0001 §5.2; every codeanalyzer-* language backend; the codeanalyzer core
derived-from: field inventories of codeanalyzer-python 1.6.1, codeanalyzer-typescript 1.6.4, codeanalyzer-java 3.3.4, taken 2026-10-14
---

# 0002: The facts IR between language backends and the core

## 1. Summary

A backend emits a stream of typed, flat records: files, declarations, heritage, imports, statements, control-flow edges, calls, points-to sets, engine-computed dataflow edges, artifacts, dependencies, config keys, signals, diagnostics. The core reads the stream and builds canonical schema v2. Nothing in the stream is nested, leveled, projected, or identified by a `can://` id. The core adds all of that.

Twelve decisions shape the IR. They were made with the maintainer on 2026-10-14 and 2026-10-15:

| # | Decision | Chosen |
| --- | --- | --- |
| D1 | Record granularity | one record type per kind, each with its own required fields |
| D2 | Language-specific fields | every field typed and enumerated in the IR schema. No opaque passthrough |
| D3 | Transport | MessagePack, one map per record, length-prefixed stream. §3.1 weighs the alternatives |
| D4 | Id minting | the core mints every `can://` id. Backends emit structured local references |
| D5 | Spans | byte offsets only, `[from, to)` into the file's UTF-8 text. The core derives line and column |
| D6 | Modifiers | one closed string list. The schema enumerates the vocabulary |
| D7 | Statement kinds at L3 | a closed required set. TypeScript upgrades from `statement`-only |
| D8 | Provenance | closed vocabulary per edge family. Engine names map to shared terms |
| D9 | References | structured `{file, path[], sig}` for locals, `{module, name}` or `{declaring_type, signature}` for externals |
| D10 | L4 facts | the core accepts both points-to sets (`pts`, over typed `AllocToken`, `ClassToken`, `SummaryToken` objects) and engine-computed `dataflow` edges. The token kind fixes the identity scope |
| D11 | Heritage | one `heritage` record with a relation enum. The core derives `base_types` and `interfaces` |
| D12 | Entrypoints | the core's rules engine only. Backends emit no entrypoint facts |

## 2. Context

The three v2 backends emit the same concepts under different names, shapes, and conventions. The inventories measured:

| | Python 1.6.1 | TypeScript 1.6.4 | Java 3.3.4 |
| --- | --- | --- | --- |
| record kinds on the wire | 21 | 22 | 24 |
| type kinds | class | class, interface, enum, type_alias, namespace | class, interface, enum, record, annotation |
| callable kinds | function | function, method, constructor, getter, setter, arrow, function_expression | method, constructor, initializer |
| L3 statement kinds | statement, branch, loop, return, raise, handler | statement | statement, return, branch, loop, switch |
| CFG edge kinds | 10 | 11 | 9 |
| column convention | 0-based char | UTF-16 derived | 1-based, inclusive end |
| leaf positions | flat ints, no bytes | flat ints, no bytes | `span` everywhere |
| modifiers | `modifiers: ["async"]` | `is_static`, `is_abstract`, `is_async`, … | `modifiers: [...]` |
| comment flag | `is_docstring` | `is_docstring` | `is_javadoc` |
| heritage | spellings | spellings plus resolved ids | resolved, split into base types and interfaces |
| decorator arguments | positional, keyword, expression | positional, keyword | `["k=expr"]` strings |
| `prov` on call edges | jedi, defuse | tsc, defuse, import | declared, rta |
| `prov` on ddg | ssa, reaching-defs, points-to | reaching-defs | ssa, points-to |

In each row, a consumer today must know which backend produced the file. The IR removes the row.

## 3. The stream

### 3.1 Transport (D3)

MessagePack. Each record is one map with a string `t` key that names the record kind.

Alternatives were weighed on 2026-10-15 and rejected for this version. Serialization is not a bottleneck anywhere in the pipeline. Every backend produces facts orders of magnitude slower than any codec parses them. So the choice is about typing, generated writers, and debuggability. Protobuf is the strongest alternative. The IR schema would be a `.proto`, and `protoc` would generate every backend's writer. This fits D2. It was set aside for now for two reasons. Proto3 has no required fields and its enums are integers, so the core would validate required-ness anyway. It also adds a toolchain to each backend. CBOR is the IETF counterpart of MessagePack. It has less TypeScript library coverage. Arrow IPC has the highest throughput. Its shape is wrong for nested records, and it brings a heavy Python dependency. JSON-RPC, as MCP and LSP use it, is a request-and-response protocol with a per-message envelope. It is not a bulk stream. Its place is the control plane in §3.6, not the data. The decision can be revisited at step 0 of ADR 0001 if the generated-writer work is larger than expected. A writer writes records as a length-prefixed sequence (`uint32` big-endian byte length, then the MessagePack bytes). This lets a reader skip a record it does not understand, and lets a writer stream without buffering. A backend writes to stdout when the core spawns it. It writes to a file when run with `--facts-out`. `canal facts dump` renders a stream as NDJSON for inspection. This keeps the debugging convenience of JSON without its size cost on every run.

### 3.2 Header and trailer

The first record is `header`. The last is `trailer`. The core refuses, by name, a stream whose `ir_version` it does not accept. It does so before it reads a second record.

| `t` | fields |
| --- | --- |
| `header` | `ir_version: u32`, `backend: {name, version}`, `languages: [str]` (from `python`, `typescript`, `javascript`, `java`, `go`, `c`, `cpp`. A backend may serve several. `codeanalyzer-typescript` declares both `typescript` and `javascript`), `max_level: u8` (the level the backend can serve on this machine), `module: {name, manifest}` (the declared build unit, ADR 0001 §6.2), `root: str` |
| `trailer` | `records: u64`, `files: u32`, `diagnostics: {error: u32, warning: u32}` |

```
header = {
  t: "header"
  ir_version: u32
  backend: { name: str, version: str }
  languages: ["python" | "typescript" | "javascript" | "java" | "go" | "c" | "cpp"]
  max_level: u8                   // 1..4, the level this backend can serve on this machine
  module: { name: str, manifest: str }
  root: str
}

trailer = {
  t: "trailer"
  records: u64
  files: u32
  diagnostics: { error: u32, warning: u32 }
}
```

### 3.3 Ordering

A record may reference only records that precede it. A `file` precedes every record about that file. A `decl` precedes its members. A `stmt` precedes the `cfg`, `call`, `pts`, and `dataflow` records that name it. The core validates this as it reads the stream. It rejects a forward reference as a protocol error. So a backend bug appears at the record, not as a dangling edge in the output.

### 3.4 References (D4, D9)

Backends never emit `can://`. A reference to a declaration is a structured local key:

```
ref      = { file: str, path: [str], sig: str }     # path = enclosing declaration names, outermost first
extref   = { module: str, name: str }                # a library or builtin target: Python, TypeScript, Go
         | { declaring_type: str, signature: str }   # a binary target: Java, C#
stmtref  = { callable: ref, loc: str }               # loc = the local statement key, § 4.5
```

The core mints the id from exactly these components. A reference the core cannot resolve is a diagnostic at the record that carried it. `path` and `sig` use the backend's own `signatureOf()`. The backend's schema decisions record the signature grammar for each language. This grammar is the one language-specific piece of the id.

### 3.5 Spans (D5)

```
span = [from: u32, to: u32]     # UTF-8 byte offsets into the file's text, [from, to)
```

The convention has no other rule. The core builds one line table per file from the `file` record's `text`. It adds `start: [line, col]` and `end: [line, col]` to every node. Lines are 1-based. Columns are 1-based and counted in characters. The end is exclusive. No backend computes a line or a column for the wire. A record with no source position omits `span`.

### 3.6 Process model

A backend is a one-shot process. The core spawns it, it streams facts, and it exits. Incremental analysis is the core's job, through its cache and sharding. A long-lived backend could answer requests over JSON-RPC, LSP-style. That would allow per-file re-analysis and avoid JVM startup on each run. This version deliberately leaves it out, for two reasons. Every backend would need a lifecycle, crash-handling, and protocol design before anything ships. Also, none of the records above would change if it were added later. This section records the option so that the reader sees the one-shot choice as a sequencing decision, not a limit.

## 4. Record catalog

Each record below appears in two forms. A table has one column per backend and shows who emits the field today. A **schema block** is the contract. Notation for the blocks: `name: T` is required. `name?: T` is optional. `[T]` is a list. `{...}` is an object. `"a" | "b"` is a closed enum. `u32` is an unsigned integer. `str` is a UTF-8 string. §3.4 and §3.5 define the shared shapes `ref`, `extref`, `stmtref`, and `span`. Every record carries `t`, the discriminator.

Every field is typed and enumerated (D2). `?` marks an optional field. Columns `py`, `ts`, `java` show whether that backend emits the field today. A reader can see which fields are shared and which a backend must add. A field no backend emits today is not in this catalog.

### 4.1 `file`

| field | type | py | ts | java | notes |
| --- | --- | --- | --- | --- | --- |
| `path` | str | ✓ | ✓ | ✓ | project-relative POSIX, the symbol-table key |
| `sha256` | str | ✓ | ✓ | ✓ | of the raw bytes |
| `text` | str | ✓ | ✓ | ✓ | **the whole file, once**. The core stores it as `module.source` |
| `role` | enum | ✓ | ✓ | ✓ | `source`, `artifact` |
| `language` | str | ✓ | ✓ | ✓ | the file's own language token, one of the header's `languages`. Under the TypeScript backend, `.js`, `.jsx`, `.mjs`, `.cjs` are `javascript`. The token goes into the id as today |
| `package` | str? | | | ✓ | the language's declared grouping: Java `package`, Go package path, TS namespace |
| `is_declaration` | bool? | | ✓ | | `.d.ts` |
| `is_tsx` | bool? | | ✓ | | |
| `format` | enum? | ✓ | ✓ | ✓ | artifacts only. Union: `toml yaml json jsonc ini properties requirements dockerfile yarnlock env xml gradle text binary jsp xhtml html ejs handlebars pug nunjucks liquid vue svelte astro` |
| `roles` | [enum]? | ✓ | ✓ | ✓ | artifacts only. Union: `dependency-manifest tool-config container-image service-topology iac ci env packaging legal docs script view-template unknown` |
| `extraction` | enum? | ✓ | ✓ | ✓ | artifacts only: `none partial full` |


```
file = {
  t: "file"
  path: str                       // project-relative POSIX; the symbol-table key
  sha256: str                     // of the raw bytes
  text: str                       // the whole file, once
  role: "source" | "artifact"
  language: str                   // one of the header's languages
  package?: str
  is_declaration?: bool
  is_tsx?: bool
  format?: "toml" | "yaml" | "json" | "jsonc" | "ini" | "properties" | "requirements"
         | "dockerfile" | "yarnlock" | "env" | "xml" | "gradle" | "text" | "binary"
         | "jsp" | "xhtml" | "html" | "ejs" | "handlebars" | "pug" | "nunjucks"
         | "liquid" | "vue" | "svelte" | "astro"                        // artifacts only
  roles?: ["dependency-manifest" | "tool-config" | "container-image" | "service-topology"
         | "iac" | "ci" | "env" | "packaging" | "legal" | "docs" | "script"
         | "view-template" | "unknown"]                                  // artifacts only
  extraction?: "none" | "partial" | "full"                               // artifacts only
}
```

### 4.2 `decl`

One record per declaration. `kind` selects which optional fields apply. The schema states the allowed set for each kind.

| field | type | py | ts | java | notes |
| --- | --- | --- | --- | --- | --- |
| `kind` | enum | ✓ | ✓ | ✓ | `class interface enum record annotation struct trait type_alias namespace function method constructor initializer getter setter lambda field enum_constant record_component type_parameter variable` |
| `file` | str | ✓ | ✓ | ✓ | |
| `path` | [str] | ✓ | ✓ | ✓ | enclosing declaration names, outermost first. Empty for file-level |
| `name` | str | ✓ | ✓ | ✓ | |
| `sig` | str | ✓ | ✓ | ✓ | the backend's `signatureOf()` for callables, `name` for everything else |
| `span` | span? | ✓ | ✓ | ✓ | absent on implicit members |
| `body_span` | span? | | | ✓ | the `{...}` block. TypeScript and Python can supply it, and the TypeScript backend has an open request for it |
| `modifiers` | [enum] | ✓ | ✓ | ✓ | D6. Union: `public private protected package_private static abstract final async generator readonly const let var using exported ambient default optional override volatile transient native synchronized strictfp sealed non_sealed implicit` |
| `decorators` | [decorator] | ✓ | ✓ | ✓ | § 4.3 |
| `comments` | [comment] | ✓ | ✓ | ✓ | § 4.3 |
| `type` | str? | ✓ | ✓ | ✓ | fields, variables, parameters, record components, enum members |
| `initializer` | str? | ✓ | ✓ | ✓ | expression text |
| `value` | str? | ✓ | ✓ | | constant value when known (enum member, const) |
| `params` | [param] | ✓ | ✓ | ✓ | callables. § 4.3 |
| `return_type` | str? | ✓ | ✓ | ✓ | |
| `error_channel` | [str]? | | | ✓ | Java `throws`; Go `(T, error)`; Rust `Result` |
| `type_params` | [type_param]? | | ✓ | ✓ | § 4.3 |
| `overloads` | [overload]? | | ✓ | | TypeScript overload signatures. § 4.3 |
| `declaration` | str? | | | ✓ | signature text with parameter names |
| `accessor_of` | str? | | ✓ | | getter and setter: the property name |
| `aliased_type` | str? | | ✓ | | `type_alias` |
| `call_signatures` | [str]? | | ✓ | | interfaces |
| `index_signatures` | [str]? | | ✓ | | interfaces |
| `arguments` | [str]? | | | ✓ | enum constant constructor arguments |
| `scope` | enum? | ✓ | ✓ | | variables: `module class function namespace` |
| `declaration_kind` | enum? | | ✓ | | `const let var using` |
| `is_implicit` | bool? | | ✓ | ✓ | synthesized member: default constructor, record accessor, enum `values()` |
| `is_variadic` | bool? | | ✓ | ✓ | on the callable when the last parameter is variadic, and also on `params[]` |
| `is_entrypoint_class` | | | | | **not in the IR**: D12, the core decides |
| `metrics` | {cyclomatic: u32}? | ✓ | ✓ | ✓ | |
| `refs` | {types: [str], fields: [str]}? | ✓ | | ✓ | cheap cross-refs the symbol pass already knows. Python's `accessed_symbols` folds into this |


```
decl = {
  t: "decl"
  kind: DeclKind
  file: str
  path: [str]                     // enclosing declaration names, outermost first
  name: str
  sig: str                        // signatureOf() for callables; name otherwise
  span?: span
  body_span?: span
  modifiers: [Modifier]
  decorators: [decorator]
  comments: [comment]
  type?: str
  initializer?: str
  value?: str
  params: [param]                 // callables; empty otherwise
  return_type?: str
  error_channel?: [str]
  type_params?: [type_param]
  overloads?: [overload]
  declaration?: str
  accessor_of?: str
  aliased_type?: str
  call_signatures?: [str]
  index_signatures?: [str]
  arguments?: [str]               // enum constant constructor arguments
  scope?: "module" | "class" | "function" | "namespace"
  declaration_kind?: "const" | "let" | "var" | "using"
  is_implicit?: bool
  is_variadic?: bool
  metrics?: { cyclomatic: u32 }
  refs?: { types: [str], fields: [str] }
}

DeclKind = "class" | "interface" | "enum" | "record" | "annotation" | "struct" | "trait"
         | "type_alias" | "namespace" | "function" | "method" | "constructor" | "initializer"
         | "getter" | "setter" | "lambda" | "field" | "enum_constant" | "record_component"
         | "type_parameter" | "variable"

Modifier = "public" | "private" | "protected" | "package_private" | "static" | "abstract"
         | "final" | "async" | "generator" | "readonly" | "const" | "let" | "var" | "using"
         | "exported" | "ambient" | "default" | "optional" | "override" | "volatile"
         | "transient" | "native" | "synchronized" | "strictfp" | "sealed" | "non_sealed"
         | "implicit"
```

For each `DeclKind`, the schema file states which optional fields are permitted. A field outside its kind's set is a validation error. It is not silently ignored.

### 4.3 Leaf shapes used inside `decl`

| shape | fields | notes |
| --- | --- | --- |
| `param` | `name`, `type?`, `default?`, `span?`, `modifiers`, `decorators`, `is_variadic?`, `is_optional?` | ordered |
| `decorator` | `name`, `qualified_name?`, `positional: [str]`, `keyword: {str: str}`, `expression?`, `span?` | Java's `"k=expr"` strings split into `keyword`, Python's `expression` kept |
| `comment` | `text`, `span`, `is_doc: bool` | `is_docstring` and `is_javadoc` both become `is_doc` |
| `type_param` | `name`, `bounds: [str]`, `default?`, `span?`, `decorators` | TypeScript `constraint` becomes one entry in `bounds` |
| `overload` | `params: [param]`, `return_type?`, `type_params`, `span` | TypeScript only today |


```
param = {
  name: str
  type?: str
  default?: str
  span?: span
  modifiers: [Modifier]
  decorators: [decorator]
  is_variadic?: bool
  is_optional?: bool
}

decorator = {
  name: str
  qualified_name?: str
  positional: [str]
  keyword: { [str]: str }
  expression?: str
  span?: span
}

comment = { text: str, span: span, is_doc: bool }

type_param = { name: str, bounds: [str], default?: str, span?: span, decorators: [decorator] }

overload = { params: [param], return_type?: str, type_params: [type_param], span: span }
```

### 4.4 `heritage` (D11)

| field | type | py | ts | java | notes |
| --- | --- | --- | --- | --- | --- |
| `from` | ref | ✓ | ✓ | ✓ | the declaring type |
| `relation` | enum | ✓ | ✓ | ✓ | `extends implements embeds satisfies mixin` |
| `target` | ref \| extref? | | ✓ | ✓ | when resolved |
| `spelling` | str | ✓ | ✓ | ✓ | as written, always |

From these records, the core emits `base_types` and `interfaces` on the type node. They hold resolved ids where `target` exists and spellings otherwise. The core emits the Neo4j `EXTENDS` and `IMPLEMENTS` edges only where resolved.


```
heritage = {
  t: "heritage"
  from: ref
  relation: "extends" | "implements" | "embeds" | "satisfies" | "mixin"
  target?: ref | extref
  spelling: str
}
```

### 4.5 `stmt` (D7)

One record per statement at L3, and one per call site at L1 and L2. A call site is a `stmt` of kind `call`. Canonical v2 places call nodes in `body` from L1 by the same rule.

| field | type | py | ts | java | notes |
| --- | --- | --- | --- | --- | --- |
| `callable` | ref | ✓ | ✓ | ✓ | |
| `loc` | str | ✓ | ✓ | ✓ | the local key: `line:col` of the statement start as the **backend** counts it. It serves only as a key. The core re-derives `line:col` from `span` for the id. Nested calls at one position carry `/2`, `/3` |
| `kind` | enum | ✓ | ✓ | ✓ | required set: `statement call return branch loop switch throw handler`. Additive: `config_access` (ts), `yield`, `await` |
| `span` | span | ✓ | ✓ | ✓ | |
| `defs` | [str] | ✓ | ✓ | ✓ | k-limited access paths defined |
| `uses` | [str] | ✓ | ✓ | ✓ | k-limited access paths used |
| `comment` | comment? | | | ✓ | |

The core synthesizes the entry, exit, and L4 port vertices. They are **not** in the IR.


```
stmt = {
  t: "stmt"
  callable: ref
  loc: str                        // local key: "line:col" as the backend counts it, "/2" "/3" for nested calls
  kind: StmtKind
  span: span
  defs: [str]                     // k-limited access paths
  uses: [str]
  comment?: comment
}

StmtKind = "statement" | "call" | "return" | "branch" | "loop" | "switch" | "throw" | "handler"   // required of every backend
         | "config_access" | "yield" | "await" | "select" | "defer"                              // additive
```

### 4.6 `cfg`

| field | type | notes |
| --- | --- | --- |
| `callable` | ref | |
| `src`, `dst` | str | `loc` keys, or `@entry`, `@exit` |
| `kind` | enum | `fallthrough true false switch_case loop_back exception return break continue yield_resume await_resume defer_resume` |

A backend emits the control-flow edges for its own constructs. The core validates one entry, one exit, and reachability. The core computes dominance, CDG, and reaching definitions itself.


```
cfg = {
  t: "cfg"
  callable: ref
  src: str                        // a stmt.loc, or "@entry"
  dst: str                        // a stmt.loc, or "@exit"
  kind: "fallthrough" | "true" | "false" | "switch_case" | "loop_back" | "exception"
      | "return" | "break" | "continue" | "yield_resume" | "await_resume" | "defer_resume"
}
```

### 4.7 `call`

A `call` record attaches to a `stmt` of kind `call`. It is separate from `stmt`, so a call's resolution can arrive after its statement.

| field | type | py | ts | java | notes |
| --- | --- | --- | --- | --- | --- |
| `site` | stmtref | ✓ | ✓ | ✓ | |
| `callee` | ref \| extref? | ✓ | ✓ | ✓ | absent when unresolved. The core writes `callee: null` |
| `prov` | [enum] | ✓ | ✓ | ✓ | D8: `resolver defuse import rta cha declared`. `jedi`, `tsc`, `go/types`, `clang` all map to `resolver` |
| `method_name` | str | ✓ | ✓ | ✓ | |
| `receiver_expr` | str? | ✓ | ✓ | ✓ | |
| `receiver_type` | str? | ✓ | ✓ | ✓ | |
| `args` | [arg] | ✓ | ✓ | ✓ | `arg = {expr?, type?, value?, name?, loc?}`. The IR drops Python's `ast_kind` and keeps Java's argument `loc` |
| `type_args` | [str]? | | ✓ | | |
| `return_type` | str? | ✓ | ✓ | ✓ | |
| `is_constructor` | bool? | ✓ | ✓ | ✓ | |
| `is_static` | bool? | | | ✓ | |
| `is_optional_chain` | bool? | | ✓ | | |
| `accessibility` | enum? | | | ✓ | of the resolved callee |

The core merges multiple `call` records for one site, from different resolvers, with a `prov` union. `merge_edges` does the same inside each backend today.


```
call = {
  t: "call"
  site: stmtref
  callee?: ref | extref           // absent when unresolved
  prov: ["resolver" | "defuse" | "import" | "rta" | "cha" | "declared"]
  method_name: str
  receiver_expr?: str
  receiver_type?: str
  args: [arg]
  type_args?: [str]
  return_type?: str
  is_constructor?: bool
  is_static?: bool
  is_optional_chain?: bool
  accessibility?: "public" | "protected" | "private" | "package_private"
}

arg = { expr?: str, type?: str, value?: str, name?: str, loc?: str }
```

### 4.8 `pts` and `dataflow` (D10)

| record | fields | emitted by |
| --- | --- | --- |
| `pts` | `var: str` (a k-limited access path), `objects: [token]` (non-empty), `callable: ref?` (absent for a global or module-level name), `prov: [enum]` | engines that hold points-to sets or name partitions: Scalpel, SVF, a future core solver |
| `dataflow` | `src: stmtref`, `dst: stmtref`, `var: str`, `prov: [enum]` | engines that compute heap def-use edges: WALA via ModRef |

`prov` for both records: `points-to`.


```
pts = {
  t: "pts"
  var: str                        // k-limited access path
  objects: [token]                // non-empty
  callable?: ref                  // absent for a global or module-level name
  prov: ["points-to"]
}

dataflow = {
  t: "dataflow"
  src: stmtref
  dst: stmtref                    // may be in a different callable from src
  var: str
  prov: ["points-to"]
}

token = AllocToken | ClassToken | SummaryToken

AllocToken   = { kind: "alloc",   file: str, loc: str, type?: str, context?: [str] }   // scope: program-wide
ClassToken   = { kind: "class",   callable: ref, id: u32 }                             // scope: that callable
SummaryToken = { kind: "summary", origin: extref, type?: str, site?: stmtref }         // scope: program-wide, or that site
```

#### The token

A token names one abstract object. It is a typed, discriminated object, not a string, because D2 forbids packing facts into text. Its `kind` fixes both its fields and its **identity scope**. The identity scope is how far the core may match two equal tokens.

```
token = AllocToken | ClassToken | SummaryToken
```

**`AllocToken`**, an object created by a statement in the analyzed program:

| field | type | required | meaning |
| --- | --- | --- | --- |
| `kind` | `"alloc"` | yes | |
| `file` | str | yes | project-relative path of the allocating statement |
| `loc` | str | yes | the statement's local key, in the same form as `stmt.loc`. It joins to the `stmt` record |
| `type` | str | no | the allocated type as the engine names it: `Session`, `java.util.ArrayList` |
| `context` | [str] | no | for a context-sensitive engine, the call-site or object context that tells this abstract object from another made at the same site. Absent means context-insensitive |

Identity scope: **program-wide**. Matched across callables and files.

**`ClassToken`**, an equivalence class from a unification engine, which has no notion of objects:

| field | type | required | meaning |
| --- | --- | --- | --- |
| `kind` | `"class"` | yes | |
| `callable` | ref | yes | the callable the class was built in |
| `id` | u32 | yes | the engine's class number, unique within that callable in this stream |

Identity scope: **the named callable only**. Never matched across callables, whatever the number.

**`SummaryToken`**, an object the engine knows only through a model of code outside the project:

| field | type | required | meaning |
| --- | --- | --- | --- |
| `kind` | `"summary"` | yes | |
| `origin` | extref | yes | the library callable or symbol the model is attached to: `{module: "requests", name: "get"}` |
| `type` | str | no | the modeled result type: `Response` |
| `site` | stmtref | no | the project call site that produced this instance, when the engine distinguishes instances per site. Absent means one abstract object for every call to `origin` |

Identity scope: **program-wide when `site` is absent, that call site only when present**.

#### What the core does with it

1. Two tokens are equal when their `kind` and every present field are equal.
2. `may_alias(a, b)` holds when some token in `objects(a)` equals some token in `objects(b)`, and that token's identity scope covers both. For a `ClassToken`, this means both `pts` records name the same `callable`.
3. Rule 2, applied to a def and a later use in one callable, yields a `points-to` `ddg` edge inside that callable. This comes in addition to the core's own reaching-definitions result.
4. Rule 2, applied to a def in one callable and a use in another, yields a `heap_flow` edge between callables. Only an `AllocToken` or a site-less `SummaryToken` can satisfy this.
5. The core merges a `dataflow` record into `ddg` unchanged, or into `heap_flow` when `src` and `dst` are in different callables.

A backend never emits syntactic `ddg` (`ssa`).

#### Worked example

```python
def handle(req):          # 1
    s = Session()         # 2   allocation site
    a = s                 # 3
    b = a                 # 4
    c = s                 # 5
    b.user = req.user     # 6   write through b
    log(c.user)           # 7   read through c

def report():             # 20
    for s in sessions:    # 21
        print(s.user)     # 22   read in another callable
```

An allocation-site engine emits four `pts` records for `handle`. Each has `objects: [{kind: "alloc", file: "api.py", loc: "2:9", type: "Session"}]`. If the engine tracks the object into `sessions`, it also emits one for `s` in `report`, with the same token. Scalpel emits the same four for `handle`, with `objects: [{kind: "class", callable: handle, id: 3}]`. It emits nothing for `report`. In both cases, the core finds `objects(b) ∩ objects(c)` non-empty inside `handle`. It then emits `ddg 6 → 7, var: c.user, prov: points-to`. Only in the first case does the core also find the token of `b` in `handle` equal to the token of `s` in `report`. That token has program-wide scope. The core then emits `heap_flow 6 → 22, var: user`. The IR needs four records instead of six pairs. Twelve names would need twelve records instead of sixty-six.

#### Which form an engine emits

An engine emits what it computed, and nothing it would have to reconstruct. WALA holds sets and derives flow-sensitive du-pairs from them. It emits the pairs as `dataflow`, not the sets, because its pairs respect statement order. A flow-insensitive set that the core derives again would be less precise. Scalpel holds a partition and nothing else. It emits `pts` with `ClassToken`s. A future Andersen-style engine, in a backend or in the core, emits `pts` with `AllocToken`s and `SummaryToken`s.

**Where the oracle lives (decided 2026-10-15).** The points-to oracle is an engine, and it stays in the backend. Scalpel runs over Python SSA, WALA's ModRef over bytecode, and SVF over LLVM IR. None of them can run without the language's IR, which the core never has. The oracle's answer crosses the boundary, in one of the two shapes above. The core turns relations between names into dependences between statements, because the core holds the statements. The core merges computed edges as they are. A backend with a working oracle is never asked to give it up.

The core may later add a language-neutral pointer analysis of its own: an Andersen-style solver over `alloc`, `copy`, `load`, and `store` facts. The solver is small. The expensive part is the per-language modeling of libraries and runtime behavior that mature engines carry. That cost moves into facts. It does not disappear. Its realistic use is for the backends that have no oracle at all: TypeScript, Go, and C today. For them, the choice is a core solver or nothing. Open question 5 in §9 records it.

### 4.9 `import` and `export`

| field | type | py | ts | java | notes |
| --- | --- | --- | --- | --- | --- |
| `file` | str | ✓ | ✓ | ✓ | |
| `module` | str | ✓ | ✓ | ✓ | as written |
| `name` | str | ✓ | ✓ | ✓ | `*` for namespace or wildcard, `""` for side-effect |
| `alias` | str? | ✓ | ✓ | | |
| `target` | str? | ✓ | ✓ | ✓ | resolved file when in the project |
| `kind` | enum | | ✓ | ✓ | `named default namespace side_effect static wildcard` |
| `is_type_only` | bool? | | ✓ | | |
| `span` | span | ✓ | ✓ | ✓ | |

`export` has the same shape, with `kind` in `named default namespace re_export`. The core keeps unresolved imports in `unresolved_imports`, with the `prov` the backend gave.


```
import = {
  t: "import"
  file: str
  module: str                     // as written
  name: str                       // "*" for namespace or wildcard; "" for side-effect
  alias?: str
  target?: str                    // resolved project file
  kind: "named" | "default" | "namespace" | "side_effect" | "static" | "wildcard"
  is_type_only?: bool
  span: span
}

export = {
  t: "export"
  file: str
  module?: str
  name: str
  alias?: str
  target?: str
  kind: "named" | "default" | "namespace" | "re_export"
  is_type_only?: bool
  span: span
}
```

### 4.10 `external`

| field | type | notes |
| --- | --- | --- |
| `ref` | extref | the identity the core homes under `@external` |
| `kind` | enum | `function method constructor class module` |

Optional. A backend may pre-declare externals. The core also homes any `extref` it meets in a `call` or `heritage` without a pre-declaration.


```
external = {
  t: "external"
  ref: extref
  kind: "function" | "method" | "constructor" | "class" | "module"
}
```

### 4.11 `dependency` and `config_key`

| record | fields | notes |
| --- | --- | --- |
| `dependency` | `name`, `group?`, `ecosystem` (`pypi npm maven go cargo`), `spec`, `kind` (`runtime dev optional peer build`), `extras: [str]`, `declared_in: str` (artifact path), `direct: bool`, `locked_version?`, `provides_imports: [str]`, `prov: [enum]` (`declared lockfile installed-metadata`) | Java's `group` is the one language-only field |
| `config_key` | `artifact: str`, `key`, `namespace` (`env yaml json toml ini properties dockerfile xml`), `value?`, `span?`, `references: [str]`, `variant?` (`arg`, `env` for the dual-mint cases) | the core derives the id |
| `config_read` | `site: stmtref`, `callee: extref`, `key?` | a config-reading call site. The core does the literal and dataflow joins to `config_key` |
| `view_dispatch` | `site: stmtref`, `target?: str`, `via` (`forward include redirect view-name`) | Java today. The core does the join to an artifact |


```
dependency = {
  t: "dependency"
  name: str
  group?: str
  ecosystem: "pypi" | "npm" | "maven" | "go" | "cargo"
  spec: str
  kind: "runtime" | "dev" | "optional" | "peer" | "build"
  extras: [str]
  declared_in: str                // artifact path
  direct: bool
  locked_version?: str
  provides_imports: [str]
  prov: ["declared" | "lockfile" | "installed-metadata"]
}

config_key = {
  t: "config_key"
  artifact: str                   // artifact path
  key: str
  namespace: "env" | "yaml" | "json" | "toml" | "ini" | "properties" | "dockerfile" | "xml"
  value?: str
  span?: span
  references: [str]
  variant?: "arg" | "env"
}

config_read = { t: "config_read", site: stmtref, callee: extref, key?: str }

view_dispatch = {
  t: "view_dispatch"
  site: stmtref
  target?: str
  via: "forward" | "include" | "redirect" | "view-name"
}
```

### 4.12 `signal`

The core's rules engine reads this record (D12). A backend emits only the evidence that is not already a `decl` field, and no entrypoint decision:

| field | type | notes |
| --- | --- | --- |
| `subject` | ref | |
| `kind` | enum | `route http_method manifest_main framework_marker` |
| `value` | str | |
| `span` | span? | |

`decl` and `heritage` already carry decorators, annotations, and base classes, so most rules need no `signal` at all. Java's name-matching finders become rules. What they match on is already in `decl.decorators`.


```
signal = {
  t: "signal"
  subject: ref
  kind: "route" | "http_method" | "manifest_main" | "framework_marker"
  value: str
  span?: span
}
```

### 4.13 `diagnostic`

| field | type |
| --- | --- |
| `code` | str |
| `severity` | enum `error warning info` |
| `file?`, `span?` | |
| `message` | str |

Diagnostics travel in the stream, not on stderr, so they arrive in the output document where ADR 0001 §7 puts them.


```
diagnostic = {
  t: "diagnostic"
  code: str
  severity: "error" | "warning" | "info"
  file?: str
  span?: span
  message: str
}
```

### 4.14 Enforcement

Both ends of the stream enforce the schema, from one source. Neither end can diverge from the other.

**One source.** The schema blocks in this section become one file, `codeanalyzer/schema/facts/`. Everything that reads or writes the stream is generated from it. No hand-written copy of a record shape exists anywhere. The IR version integer is that file's version.

**Producers: the backends.** The schema generates each backend's facts writer: Python dataclasses, TypeScript types, Java records, Go structs. A backend constructs a typed `Stmt` or `Decl`, never a map. So these errors fail before a byte is written: a misspelled field, a wrong type, a kind outside its enum, a missing required field, or a `decl` field not permitted for its `kind`. They fail in the compiler for Java, Go, and TypeScript. They fail at construction time for Python. The writer is the only component that serializes. So a backend never touches MessagePack directly.

**Receiver: the core.** The same schema generates the Rust types. The core deserializes every incoming record into its typed struct by `t`. It rejects a record that does not fit, at that record, with a diagnostic that names the field. The record does not surface later as a dangling edge. In addition to the per-record shape, the core checks what a type system cannot state. It checks the ordering rule of §3.3. It checks that every `ref` and `stmtref` resolves to a record already seen. It checks that `loc` keys are unique within a callable. It checks that `pts.objects` is non-empty. It checks that a `ClassToken` matches only inside its own callable. It checks the `ir_version` in the header before it reads anything else.

| Check | Backend writer | Core reader |
| --- | --- | --- |
| field names, types, enums, required-ness, per-kind field sets | compile or construction time | again, on every record |
| record order and forward references | no | yes |
| references resolve | no | yes |
| semantic invariants: one entry and one exit per CFG, non-empty sets, token identity scope | no | yes |
| IR version compatibility | declares it in the header | enforces it before reading further |

The shape check runs twice, on purpose. A backend built against an older generated writer, or one that bypasses the writer, cannot get a malformed record past the core. The core never trusts a stream because the writer was supposed to be correct. One schema, two enforcers, no prose contract between them.

## 5. What is not in the IR

The core computes all of the following, so that each is computed once:

- `can://` ids, local ordinal ids, global ordinal ids (D4)
- line and column (D5)
- `entry`, `exit`, `formal_in`, `formal_out`, `actual_in`, `actual_out`
- `cdg`, syntactic `ddg`, `summary`, `param_in`, `param_out`, `heap_flow`
- the merged `call_graph` list, `callee` backfill, `weight`
- `base_types` and `interfaces` on type nodes (from `heritage`)
- external homing and `external_symbols`
- `config_uses` and the resolved `view_dispatches` (the joins)
- entrypoints and the entrypoint report (D12)
- `content_hash`, the envelope, `max_level`, both projections

The IR also deliberately drops three things that the inventories found on the wire today. The first is Python's `ast_kind` on call arguments. The second is Python's absolute `path` and mtime `last_modified` on callables (machine-dependent, ADR 0001 §7). The third is TypeScript's `call_graph` edges whose source is a module id. The core gives module-scope calls a synthesized `<module>` callable instead.

## 6. Vocabularies

The schema closes these vocabularies. A backend that needs a new value opens a pull request against `codeanalyzer/schema/facts/`, and the IR version changes.

| vocabulary | values |
| --- | --- |
| `decl.kind` | § 4.2 |
| `stmt.kind` required | `statement call return branch loop switch throw handler` |
| `stmt.kind` additive | `config_access yield await select defer` |
| `cfg.kind` | § 4.6 |
| `modifiers` | § 4.2 |
| `heritage.relation` | `extends implements embeds satisfies mixin` |
| `call.prov` | `resolver defuse import rta cha declared` |
| `pts.prov`, `dataflow.prov` | `points-to` |
| `dependency.prov` | `declared lockfile installed-metadata` |
| `artifact.format`, `artifact.roles`, `config_key.namespace` | § 4.1, § 4.11 |

The `header` names the backend and its version, so the engine name is not lost. ADR 0001's envelope carries `core: {version, ir}` beside `analyzer`.

## 7. Consequences

- The schema in `codeanalyzer/schema/facts/` generates each backend's facts writer and the core's reader (§4.14). So the catalog above is the only hand-written artifact.
- TypeScript emits seven statement kinds instead of one, a `body_span`, and `heritage` records with resolved targets. It already has the data.
- Java splits decorator arguments into `keyword`, drops the `"k=expr"` strings, and turns five finders into rules plus `decl.decorators`.
- Python folds `accessed_symbols` into `refs`, stops emitting `ast_kind`, `path`, and `last_modified`, and sends byte spans only. This also removes `byte_offsets()` and its column bug.
- No backend computes a line, a column, an id, a dominator, a reaching definition, a summary, or an entrypoint.

## 8. Confirmation

- The schema file in `codeanalyzer/schema/facts/` is the only hand-written artifact. It generates every writer and the core's reader. A generated writer that no longer matches the file fails its build (§4.14).
- The core rejects any record that does not match its generated type, any forward reference, any unresolved reference, and any violated invariant. Each rejection carries a diagnostic that names the record (§4.14).
- Step 0 of ADR 0001 validates the catalog in §4 against real backend output. It must be possible to rebuild each of the three backends' current `analysis.json` from a facts stream that validates against this schema.
- The vocabularies in §6 are closed. A value outside them is a validation error. So divergence appears as a failed pull request, not as a stale document.

## 9. Open questions for step 0

1. TypeScript declaration merging: today a merged name yields ids `Foo`, `Foo#type`, `Foo#interface`. A getter, a setter, and a method of one name share an id. In the IR these are distinct `decl` records with the same `path` and `name`. The core's id grammar must decide the suffix rule once, for every language with merging or overloading.
2. `stmt.loc` as a key: the backend's `line:col` serves only as a reference inside the stream. The open question is whether the core should key statements by `span.from` instead and drop `loc` entirely. The decision comes when the first backend is ported.
3. The k-limit for `defs` and `uses` access paths is a core parameter today (`--graph-field-depth`). Two options are undecided. In the first, a backend emits full paths and the core truncates them. In the second, the core passes the limit to the backend. The first costs stream size. The second couples the backend to a core flag.
4. Whether `config_access` stays a statement kind (TypeScript) or becomes a `config_read` record. Java and Python use a `config_read` record for the same fact.
5. A core-side pointer analysis over `alloc`, `copy`, `load`, `store` facts for backends without an oracle (§4.8). It is not in this version. The record kinds above do not change if it is added.

## 10. References

- `0001-shared-core-with-thin-language-backends.md`, §5.2, §6.2, §7
- the three inventories, taken 2026-10-14 against `codeanalyzer-python` 1.6.1, `codeanalyzer-typescript` 1.6.4, `codeanalyzer-java` 3.3.4
- `cldk-devtools/skills/designing-cldk-changes/references/canonical-schema.md`, the keystone this IR feeds
- the TypeScript backend's UTF-16 offset defect and body-span request; the module-segment and `heap_flow` specs under `.github/docs/design/specs/`
