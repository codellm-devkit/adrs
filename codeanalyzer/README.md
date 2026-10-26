# ADRs — codeanalyzer

Architecture Decision Records for the **codeanalyzer** component.

Use the shared [filename and ordering rules](../README.md#adr-filenames-and-ordering).
Copy the [template](../template.md) as a starting point.

Decisions that concern one backend live in a subfolder named after that backend's
repository, numbered within the subfolder, e.g. `codeanalyzer-python/0001-….md`.
Diagrams and other assets go under `assets/`, named after the ADR they belong to.

## Index

| # | Title | Status |
|---|-------|--------|
| [10](10-application-identity-namespaces.md) | Application identity namespaces | proposed (draft) |
| [20](20-shared-core-with-thin-language-backends.md) | A shared Rust core with thin language backends | proposed |
| [21](21-facts-ir.md) | The facts IR between language backends and the core | proposed (draft) |
| [30](30-external-library-model.md) | External Library Model (ELM) | proposed (draft) |
| [codeanalyzer-python/0001](codeanalyzer-python/0001-resolver-ladder-syntactic-before-inference.md) | Resolve syntactically first, infer with Jedi only for the leftovers | proposed (draft) |
| [codeanalyzer-java/0001](codeanalyzer-java/0001-no-build-analysis-with-supplied-bytecode.md) | Analyze Java without a target build and accept supplied bytecode | proposed (draft) |
