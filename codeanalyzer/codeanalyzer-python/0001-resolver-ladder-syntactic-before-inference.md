---
status: proposed            # draft, to be refined after the measurements in section 7
date: 2026-10-15
decision-makers: [Rahul Krishna]
consulted: [CLDK maintainers]
informed: [codeanalyzer-python contributors]
scope: codeanalyzer-python L1 and L2; the inference-oracle seam ADR codeanalyzer/0001 assumes every backend with an engine will have
related: [../0001-shared-core-with-thin-language-backends.md]
---

# 0001: Resolve syntactically first, infer with Jedi only for the leftovers

## 1. Summary

Today Jedi runs first, on everything. While the backend builds the symbol table, `SymbolTableBuilder` asks Jedi to infer the receiver, the callee, and the argument types of every call site. That is up to three inferences per site, plus one per argument. The syntactic tiers of the defuse linker run afterward, on what Jedi left unresolved.

This ADR inverts the order. A ladder of syntactic steps over `ast` resolves every call site that it can answer unambiguously, with no inference. The backend consults Jedi only for the sites that remain, and memoizes the answers by scope. Provenance records which step answered. The same ladder applies to types. Annotations come first. Inference runs only where something downstream needs a type that the source does not state.

The change is entirely inside this backend. It ships as a minor release and does not depend on the core ADR landing. The core ADR expects this same shape of every backend that carries an inference engine. So Python becomes the reference for that rule, as it is for the schema.

## 2. Context

### 2.1 What Jedi is used for

`syntactic_analysis/symbol_table_builder.py` uses Jedi for parameter, return, and receiver type inference, for call-site callee resolution through `goto`, and for decorator names. The backend derives the L1 call graph from those stored call sites. At L2, `semantic_analysis/defuse_linker.py` adds edges through its tiers T1 to T5, and `merge_edges` unions them.

### 2.2 What it costs

- Jedi runs up to three inferences per call site, plus one per argument (`symbol_table_builder.py:784-803`).
- Jedi caches every module it has inferred into, so the resident set of each Ray worker grows with the project.
- Output depends on the interpreter that Jedi runs under. Resolution spellings differ across Python versions. In the 1.6.0 verification, two independent runs on two machines differed in about 0.4% of call edges. The difference is attributed to Jedi inference under Ray's file ordering.
- The linker runs second. It links an unresolved method call to every same-named method in the application. A syntactic tier that guesses fails in this way.

### 2.3 Where the memory goes after 1.6.0, and what is not known

Release 1.6.0 cut level-4 peak RSS on full Odoo from 42.6 GB to 5.7 GB. The PR records the remaining phase peaks. The symbol-table phase is the highest, at 5.9 GB. The Neo4j emit peaks at 5.06 GB. The L3 and L4 phases are below the L1 peak, after 1.6.0 frees their intermediates early.

The PR does not record what makes up each peak. The 5.9 GB at L1 is some mix of four things. They are Jedi's per-worker state, the driver that accumulates every `PyModule` with its source, the Ray object store (capped at 1 GB), and the worker processes themselves. The L4 phase holds the facts tables, the port vertices, and the SDG IR. Its peak is below L1 only because 1.6.0 frees them aggressively. No measurement in the repository shows which is the next ceiling once inputs grow past Odoo: Jedi, the driver's tree, or the L4 structures. Section 7 makes that the first thing this ADR measures. This draft does not claim where the peak comes from.

## 3. Decision

### 3.1 The ladder

The ladder offers a call site to each step in order. A step answers only when it has exactly one candidate. After a step answers, the ladder does not consult higher steps.

| Step | Resolves | Mechanism | `prov` | Cost |
| --- | --- | --- | --- | --- |
| 1 | annotated parameter and return types | `ast` annotations | `["annotation"]` | free |
| 2 | names bound in the same module: local and module-level functions, classes, `self.method()` where the enclosing class defines it | `ast` plus a per-module scope table | `["syntactic"]` | free |
| 3 | names bound through imports: `from a import f`, `import a; a.f()`, relative imports, `__init__` re-exports, following the import to the defining module | the module graph the backend already builds for `resolve_imports` | `["syntactic"]` | one pass over imports |
| 4 | inherited methods: `self.method()` where a project base class defines it, through the hierarchy known from steps 2 and 3 | the linker's existing tiers, moved here | `["defuse"]` | cheap |
| 5 | receivers that are the result of a call, attribute chains through instances, unannotated parameters used as receivers, dynamic dispatch | Jedi, or a swapped oracle (section 3.4) | `["jedi"]` | the expensive step, now limited to the leftovers |
| none | still unresolved | | `callee: null`. External homing as today. | |

### 3.2 Rules that make it safe, not only cheaper

- **No step guesses.** A step resolves a site when it has one candidate. The site escalates when the step has two candidates, when the module rebinds the name, or when a `from x import *` is in scope. A guessing step produces the over-linking defect of section 2.2.
- **First answer wins.** The ladder does not consult higher steps once a step has answered. So the output is deterministic across hosts for every site resolved below step 5. This shrinks the cross-version drift to the sites that need inference.
- **Jedi is memoized by scope, not by spelling.** The Java backend's performance notes record the reason. One spelling resolves differently in two classes of the same file. The memo key is the enclosing scope plus the expression.
- **Provenance is honest.** `prov` names the step. A consumer that wants only inference-grade edges can filter. A precision audit can count.

### 3.3 Types follow the same shape

Annotations are free and come first. For an unannotated parameter or return, inference runs only if a downstream consumer needs the type. At L1 the only such consumers are `parameters[].type` and `return_type` on the callable. So a flag controls inference for those fields. The flag defaults to on, for compatibility, until the measurement in section 7 shows what inference costs. The flag also gives an explicit option to turn inference off for large inputs.

### 3.4 The oracle is an interface

Step 5 is a declared interface inside the backend. Given a scope and an expression, it returns zero or one targets with a provenance token. Jedi is the first implementation. `scip-python` over Pyright is the first candidate for a second, faster implementation, because it produces whole-project resolution in one batch. The Rust checkers, `ty` and Pyrefly, are preview or beta as of this date. The project watches them and does not adopt them. The core sees which oracle answered only through `prov`.

### 3.5 Modules: one per distribution, files from the wheel's RECORD

The core ADR defines a module as the build unit that an ecosystem declares. Python declares distributions. So the backend reports one module per distribution and must say which files each module owns. The backend does not walk `__init__.py` trees. It does not infer a depth. It does not special-case PEP 420 namespace packages.

- **The file list comes from the build backend, not from this analyzer.** The backend builds the wheel into a scratch directory (`pip wheel` or a direct PEP 517 call). It does this for each `pyproject.toml` in the input, including workspace members. The backend then reads the wheel's `RECORD`, which lists every file the distribution owns, with its sha256. This uses the package discovery of the build backend itself (setuptools, hatch, pdm, or flit). It does not reimplement that discovery.
- **The join to the source tree is by content hash.** The backend already hashes every source file for its cache. The match `RECORD.sha256 == source.sha256` maps the installed path to the source path. For identical files, the path suffix breaks the tie. The backend does not strip a `site-packages` prefix, because `src/` layouts and `package_dir` remaps break it.
- **The editable install stays.** The analysis venv keeps `pip install -e`, so Jedi resolves imports to the source tree. The backend builds the wheel but never installs it. So resolution and spans share one copy of the code.
- **What does not match is reported, not guessed.** A `RECORD` entry with no source match (a generated `_version.py`, a compiled extension) is a diagnostic. A source file that no distribution lists (tests, scripts) is unclaimed. The backend still analyzes it.
- **Cost.** The backend does one wheel build per distribution per run. It caches the build on the `pyproject.toml` hash, so warm runs skip it. On Odoo that is a copy of a few thousand files. It costs seconds, against minutes.
- **No framework units.** An Odoo addon or a Django app is a path prefix that a consumer can filter on. It is not a module. If a consumer ever needs it as a unit, that takes an additive rule in the rules pack. It also takes a module-segment spec conversation about the id segment. Nothing here depends on it.

## 4. Consequences

- Jedi's work falls by the share of call sites that the syntactic steps resolve. That share is unknown until section 7 item 1 measures it.
- Per-worker resident set falls with it, because Jedi caches fewer modules when it gets fewer questions. Section 7 item 2 measures how much.
- Determinism improves for every site resolved below step 5.
- `symbol_table_builder.py` loses its Jedi calls. This is also the largest step toward the file-level split that the refactor proposal asks for. The builder becomes the `ast` walk. Resolution becomes its own stage.
- This change does not promise byte-identical output, and it should not. Edges that Jedi resolved before, and that steps 2 to 4 now resolve, will carry a different `prov`. Each site where Jedi and the syntactic steps disagree will surface in a diff to adjudicate one by one. That diff is the precision audit.

## 5. Confirmation

- Before any code moves, the four measurements in section 7 are taken on full Odoo 20.0 and recorded in this ADR.
- The ladder ships behind a flag. The precision audit (section 7, item 3) decides which steps default on. The repository keeps the audit's disagreement set as a test fixture.
- Each step has named tests. Each test asserts a specific resolved target, not "non-empty."
- The output diff against 1.6.1 is reviewed edge by edge. A `prov` change is expected. A target change must be adjudicated and recorded.

## 6. Risks

| Risk | Mitigation |
| --- | --- |
| a syntactic step resolves to the wrong target where Jedi was right, for example a name shadowed at runtime | the no-guess rule, plus the audit in section 7 item 3 before the flag defaults on |
| the memory ceiling is not where this ADR assumes, so the gain is smaller than hoped | the measurement in section 7 item 2 comes before any code moves. This draft makes no claim about the ceiling |
| step ordering interacts with the linker's tiers in ways the current tests do not cover | the linker's tiers become steps. Each step has named tests that assert a specific resolved target |

## 7. Measurements this draft depends on

Each measurement uses the workload that the 1.6.0 memory work used, full Odoo 20.0, so the numbers compare.

1. **Resolution share per step.** Run the ladder in audit mode. For every call site, record which step would have answered and what Jedi answers. Report counts per step and the agreement rate between steps 2 to 4 and Jedi.
2. **Per-phase memory attribution.** At `-vv`, `_log_mem` already logs driver RSS at phase boundaries. Add per-worker RSS for the Ray workers. Add a breakdown of the driver's L1 peak into the accumulated `PyModule` tree versus other state. Repeat with Jedi disabled to isolate its share. Report L1, L2, L3, L4, and emit peaks separately.
3. **Precision audit.** Collect the call sites where a syntactic step and Jedi disagree. Classify each by hand as "step right," "Jedi right," or "both defensible." The result decides which steps default on.
4. **Wall clock per phase.** Measure it with and without the ladder. This shows the size of the gain against the 1.6.0 figures of 19 min 10 s before and 15 min 19 s after.

## 8. Open questions

1. Does step 4 belong before or after step 3 for method calls on `self` when the base class is imported? The draft order follows the cheapest-first rule. The audit may say otherwise.
2. Should type inference for unannotated callables default on or off once the cost is known?
3. Is the TypeScript backend's planned precision audit the same instrument, so the two backends share the audit format?

## 9. References

- `../0001-shared-core-with-thin-language-backends.md`, sections 5.3 and 7
- Python backend releases 1.6.0 (level-4 memory cut) and 1.6.1 (streamed artifacts and nodes)
- Python backend defects: output depends on the interpreter; the defuse linker over-links unresolved method calls
- The Python backend's refactor proposal
- The Java backend's performance notes on memoizing resolution by scope rather than spelling
- The TypeScript backend's plan to strengthen the defuse tiers for JavaScript with a precision audit
- `cldk-devtools/skills/codeanalyzer-backend/references/tooling-menu.md`, "resolver-based vs framework-based" axis
