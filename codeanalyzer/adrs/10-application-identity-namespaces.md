---
status: proposed
date: 2026-10-27
decision-makers: []
consulted: []
informed: []
---

# 10: Application identity namespaces

## Context

One microservice application contains several related services in the same graph. For example, `online-boutique` can contain `emailservice` and `recommendationservice`, each with its own `foo.py`.

Both files can have the local module key `foo.py`, but they belong to different services. Analysis and lookup must retain the overall application, the owning service, the language, and the relative file path.

This identity must remain intact through call-graph, data-flow, and taint analysis. Equal file names or signatures must not merge code from different services into one analysis node.

The namespace identifies the overall microservice application, such as `online-boutique`. Each service keeps its own `application_name`. A monolith uses the same two fields with the defaults described below.

The current analyzers identify each analyzed project or service with `--app-name`, or the input directory name by default. The SDK selects it with `application_name`. Neither interface has a separate field for the overall application's identity.

The namespace supplies that shared identity without replacing the individual service names. It also gives callers one explicit scope for discovery across related services.

Distinct service names already produce distinct module URIs today:

```text
can://emailservice/python/foo.py
can://recommendationservice/python/foo.py
```

Those IDs distinguish the services, but do not carry their shared `online-boutique` identity. Keeping only `foo.py` during lookup also discards the service identity they already contain.

For example, both SDK instances can hold the local key `foo.py`. The current shared path resolver produces these results:

| Selected SDK application | Input path | Known module keys | Current resolved key |
|---|---|---|---|
| `emailservice` | `emailservice/foo.py` | `["foo.py"]` | `foo.py` |
| `recommendationservice` | `emailservice/foo.py` | `["foo.py"]` | `foo.py` |

The resolver accepts the known key at the end of the path. It normalizes paths inside an already selected application. It does not select a service or prove that the selected service owns the supplied path.

**The following indexing and lookup antipatterns must be avoided:**

- Combining modules from several services under the bare key `foo.py`.
- Matching file suffixes across services without retaining their owners.
- Asking every service backend for a position and accepting the first callable returned.

**A namespace segment alone does not correct these antipatterns.** The index keys, SDK lookups, and graph joins must preserve the complete identity.

The terminology in this decision is:

| Term | Meaning | Example |
|---|---|---|
| Namespace | The identity of the overall application, such as a microservice application containing related services. | `online-boutique` |
| Analyzed application | A project or a single service selected for analysis, identified by `application_name` or `--app-name`. | `emailservice` |
| Service | A logical component of a software system. An analyzed application can represent a service or a monolith. | An email service |
| Build module | A build unit declared by its language ecosystem. | A Maven module |
| Producer | A code analyzer that emits records and their canonical IDs. | `codeanalyzer-python`, `codeanalyzer-go` |
| Consumer | Code that reads, indexes, queries, or joins those records. | `python-sdk`, a graph loader, a findings integration |

An analyzed application can contain multiple languages and build modules. Namespace, analyzed application, service, and build module are distinct concepts.

### Required repository and application topologies

The objective is general multi-service, multi-language analysis. Services can share one repository or use separate repositories. A service or monolith can contain one language or several languages.

This ADR defines URI identity across these arrangements. Source orchestration details belong in the RFC.

All six arrangements below are required. The examples use `foo` as the shared system namespace. The monolith examples also use `foo` as the default application name.

| Repository arrangement | Application structure | Languages | Logical application roots |
|---|---|---|---|
| One monorepo | Multiple services | One language | `can://foo/svc-1`, `can://foo/svc-2` |
| Multiple repositories | Multiple services | One language | `can://foo/svc-1`, `can://foo/svc-2` |
| One monorepo | Multiple services | Multiple languages | The same service roots, each with its required language views. |
| Multiple repositories | Multiple services | Multiple languages | The same service roots, each with its required language views. |
| One repository | One monolith | One language | `can://foo/foo` |
| One repository | One monolith | Multiple languages | `can://foo/foo`, shared by its language views. |

**Repository boundaries do not establish application boundaries. Several services can share a repository, and one analyzed application can contain several languages.**

## Decision

We will add namespace as an explicit identity input to the language analyzers and the Python SDK. The URI contract applies to Python, Java, TypeScript/JavaScript, Go, and future language analyzers.

This decision requires two protections:

1. Analyzers emit the overall application and the service identity in each application-owned URI, such as `can://online-boutique/emailservice/python/foo.py`.
2. A Python SDK instance selected for `online-boutique/emailservice` resolves `foo.py` within `emailservice`. It must not return `recommendationservice`'s file merely because its local key also matches.

The code analyzers produce these IDs. The Python SDK, graph loaders, and integrations that attach findings to code consume them. Analyzers and consumers must retain the IDs through call, data-flow, and taint analysis, as well as lookup, indexing, deduplication, and graph joins.

This change has immediate priority.

### 1. Resolve namespace and application separately

The analyzer CLI will accept an optional `--namespace` alongside `--app-name`.

```text
application_name = supplied --app-name, otherwise the project-root directory name
namespace        = supplied --namespace, otherwise the resolved application_name
```

The SDK will accept `namespace` separately from `application_name`. When a caller selects one application, the SDK will use the same resolution rule.

| Inputs | Resolved namespace | Resolved application |
|---|---|---|
| Project root `inventory`, no identity arguments | `inventory` | `inventory` |
| Project root `inventory`, `--app-name catalog` | `catalog` | `catalog` |
| `--namespace foo --app-name svc-1` | `foo` | `svc-1` |
| Project root `svc-1`, `--namespace foo` | `foo` | `svc-1` |

Monolithic applications require no additional identity arguments. Integrations can supply one shared namespace for several service projects.

For example, a Python SDK caller can select two related services with these proposed configurations:

```python
email = CLDK.python(
    backend=Neo4jConnectionConfig(
        uri="bolt://localhost:7688",
        namespace="online-boutique",
        application_name="emailservice",
    )
)
recommendations = CLDK.python(
    backend=Neo4jConnectionConfig(
        uri="bolt://localhost:7688",
        namespace="online-boutique",
        application_name="recommendationservice",
    )
)
```

Both instances use the same Neo4j database and namespace. Their different application names select different services and therefore different `foo.py` identities:

```text
can://online-boutique/emailservice/python/foo.py
can://online-boutique/recommendationservice/python/foo.py
```

The `namespace` configuration field is proposed by this ADR; the current SDK does not support it yet.

If this caller omits `namespace`, the proposed default for the email service is `can://emailservice/emailservice/python/foo.py`. It does not automatically join the `online-boutique` namespace. The caller must supply the common namespace to group these related services.

For a monolith with project root `inventory`, omitted identity inputs produce `can://inventory/inventory/python/foo.py`. A graph attachment without a project root must supply `application_name` because no directory name is available as a default.

### 1.1 Identify the source boundary of each application

A multi-service analysis must associate each analyzed application with its intended source boundary. An application name alone does not assign files to that application.

In a monorepo, service analyses can use distinct selected roots while sharing one namespace. Across repositories, analyses can use the same namespace with separately selected application names.

The input directory remains a fallback name source. It does not prove that an entire repository is one service, or that each language is a separate application.

File keys must have a declared relative-path base within the selected application. Absolute checkout locations must not become application identity implicitly.

For selected project roots named `checkout-core` and `checkout-payment`, the default application names remain distinct. With namespace `foo`, their files have distinct identities:

```text
can://foo/checkout-core/python/src/utils.py
can://foo/checkout-payment/python/src/utils.py
```

Assigning both distinct applications the same namespace/application pair is a caller configuration error. The URI represents the supplied application identity; it cannot infer that the caller intended two applications.

Repeated analyses and different language views of the same application can intentionally share its namespace/application pair.

The six required arrangements do not require combining multiple repositories under one application name. They do not require an additional repository segment in the URI.

### 2. Always retain both URI segments

The application root will have this form:

```text
can://<namespace>/<application_name>
```

The two root segments are `namespace` and `application_name`. Both remain present, including when their values match:

```text
can://inventory/inventory/python/server.py
can://foo/svc-1/python/server.py
can://foo/svc-2/python/server.py
```

We will not collapse equal segments. The identity grammar must not change according to whether the namespace equals the application name.

Each identity input occupies one URI segment. The RFC must define validation and encoding that prevent distinct accepted input pairs from producing the same root.

The language remains part of the code identity below the application root. An application can therefore contain several language views:

```text
can://foo/foo/python/server.py
can://foo/foo/javascript/server.js
can://foo/foo/java/src/Server.java
```

The RFC must make the remaining file, symbol, and shared-entity suffix rules consistent across producers. Ambiguous existing suffixes must not survive solely for compatibility.

The same source revision retains the same code identity across deployment environments. Deployment environment does not select an identity namespace automatically.

### 3. Apply the root consistently

The qualified root will apply to every existing application-owned `can://` identity. This includes code, body nodes, artifacts, configuration keys, and external symbols.

```text
can://foo/svc-1/python/server.py
can://foo/svc-1/artifact/requirements.txt
```

#### Preserve ownership during analysis

Call graphs, data-flow graphs, and taint paths must retain each node's namespace, application, language, and file ownership. The graph must use canonical node IDs even when display labels show only local names.

For example, two services can each contain `foo.py` and a function named `foo.handle`. Those functions remain separate nodes during analysis. Their matching names must not combine their callers, data-flow facts, or taint paths.

Cross-service analysis can connect these distinct nodes when evidence supports that relationship. Each edge must retain its qualified endpoints. Shared namespace membership or matching file and symbol names do not establish an analysis edge.

#### Polyglot applications and shared entities

One logical namespace/application pair can have several producer and language contributions. Selecting a language view must preserve the owning application identity.

Equal IDs must identify the same semantic entity even when a consumer discards Neo4j labels. Different runtime or language meanings must remain distinct.

External-symbol identities must retain the language/runtime and dependency context needed to distinguish their meanings. Matching module and symbol spellings alone do not establish cross-language identity.

Artifacts can share identity across producers only for the same source file within the declared application scope. Configuration keys also require a common extraction-kind and key grammar.

Re-analysis, retries, replacement, and cleanup must preserve other producer contributions. Removing one language's contribution must not delete shared artifacts or detach another language's relationships.

Application enumeration must identify the logical application and distinguish its language views.

Cache reuse must preserve the selected namespace, application, producer, language view, and source context. Reused facts must not retain IDs from another selected scope.

For example, Python and JavaScript analyzers can both read the same service's `settings.yaml`. They must refer to the same artifact URI:

```text
can://foo/svc-1/artifact/settings.yaml
```

The RFC will define the URI endings for configuration keys and external symbols so every language analyzer applies the same identity rules.

The RFC will also define the JSON and Neo4j records for results from several analyzers under one application root. For example, a Python result and a JavaScript result each retain their own analyzer name and version. That specification will define supported version combinations and how callers distinguish available, failed, and incomplete analysis results.

#### Version-aware dependency identity

Package URLs use a separate identity scheme. Current versionless `pkg:` nodes identify package families. Declaration constraints and resolved lock versions belong to application-specific dependency evidence.

Package-family identity alone cannot distinguish two applications that resolve different versions.

We will distinguish package families, resolved package releases, and application-owned dependency occurrences.

| Identity | Meaning | Scope |
|---|---|---|
| Package family | A package name within its ecosystem, without a resolved version. | Can be shared across applications. |
| Package release | A known resolved release, identified by a version-qualified PURL and any required source or platform qualifiers. | Shared only for the same complete package identity. |
| Dependency occurrence | One application's dependency declaration or resolved use, with its manifest and resolution evidence. | Owned by the namespace and analyzed application. |

Known releases will use version-qualified package identities. For example, these are distinct releases:

```text
pkg:npm/require-in-the-middle@8.0.0
pkg:npm/require-in-the-middle@7.2.0
```

Two applications that use the same complete release identity can reference one shared release record. Different resolved versions must remain distinct.

Each application retains its own dependency occurrences, even when applications reference the same release. Multiple versions within one analyzed application must not collapse solely because their package names match.

If the resolved version is unknown, the occurrence must remain explicitly unresolved. A declared version range is not a resolved release. A versionless family identity must not masquerade as a concrete release.

Application namespace qualifies the dependency occurrence. It does not replace the ecosystem-specific namespace inside a PURL, such as an npm scope or Maven group.

JSON output, Neo4j projections, and SDK reconstruction must preserve the same dependency and version identities.

The RFC will specify occurrence records, evidence fields, reconstruction behavior, and canonical PURL construction for each supported ecosystem.

The canonical output must carry the resolved namespace and application name as separate identity fields. Consumers must not infer either field from a directory name after the analysis completes.

The RFC will specify field placement in JSON and Neo4j. Identity namespace must remain distinct from existing configuration-format or Kubernetes namespace properties.

### 4. Add SDK selectors without combining their meanings

The SDK will expose both selectors in its configuration and pass them through its local analyzer invocation and Neo4j readers.

#### 4.1 Make the caller's application binding explicit

The proposed SDK configurations in section 1 select `online-boutique/emailservice` and `online-boutique/recommendationservice` separately.

The `email` instance selects the application root `can://online-boutique/emailservice`. Its descendant scope is `can://online-boutique/emailservice/`.

Relative paths remain convenient after that selection:

```python
email.locate("foo.py", 28)
recommendations.locate("foo.py", 28)
```

The first request targets the email service's `foo.py`; the second targets the recommendation service's `foo.py`. The same path and line do not make them the same source position.

The SDK caller chooses the application scope through configuration inputs or the defaults in section 1. The language-specific factory selects its language view.

The SDK resolves, validates, and enforces this binding for ordinary requests. A namespace alone does not select an application. An explicit namespace-wide search follows section 5.

If the ordinary application scope cannot resolve from its inputs, the SDK must report the missing selector. It must not silently broaden the request.

| Responsible party | Required behavior |
|---|---|
| SDK caller | Supplies the identity inputs, or uses the declared defaults, and chooses the language view. |
| Multi-application integration | Maps each source position to its owning application and selects the corresponding SDK binding. |
| SDK | Resolves the selected scope and enforces it throughout lookup, graph queries, and returned references. |

An integration must use declared application identity or a source-root mapping to select a binding. A namespace or an arbitrary path prefix does not establish service ownership.

An integration must not ask every application backend for a position and select the first callable returned. If ownership remains ambiguous, resolution must report ambiguity without a guessed owner.

#### 4.2 Preserve canonical identity through analysis, lookup, and graph joins

Identity indexes, deduplication, and graph joins for application-owned entities must use the full canonical URI.

For example, both services can contain `foo.py` with a function whose signature is `foo.handle`. A combined index keyed only by `foo.py` or `foo.handle` gives the two owners the same slot.

The required module index preserves both records:

| Owning service | Key that loses ownership | Required canonical module key |
|---|---|---|
| `emailservice` | `foo.py` | `can://online-boutique/emailservice/python/foo.py` |
| `recommendationservice` | `foo.py` | `can://online-boutique/recommendationservice/python/foo.py` |

The signature `foo.handle` must also resolve within its selected service. Its canonical callable ID must retain that service's module identity.

Secondary lookup keys include a module's `name` or `norm_path`, a relative file key such as `foo.py`, and a callable signature such as `foo.handle`. These keys help find candidates. They do not uniquely identify code across services.

A secondary index can return several canonical IDs for the same search key. It must retain every owner instead of collapsing them into one `foo.py` record.

Callers can supply relative file paths within an established application binding. The SDK must resolve each path within that application and preserve the resulting canonical module URI.

If an input supplies a full URI outside the selected application or language view, the SDK must reject the scope conflict. It must not remove the conflicting prefix and retry against a local suffix.

Path normalization can use declared source-root mappings and aliases. It must preserve application ownership.

For example, an integration can declare `/repo/src/emailservice/` as the email service's source root:

| Scanner path | Selected service | Required result |
|---|---|---|
| `/repo/src/emailservice/foo.py` | `emailservice` | Resolve to that service's `foo.py`. |
| `/repo/src/emailservice/foo.py` | `recommendationservice` | Report a scope conflict. |

The integration can remove the declared email source root to obtain `foo.py`. It must not remove arbitrary leading directories until the path matches a different service's local file key.

Bindings from findings to modules or callables must join on their resolved canonical identities. Filename tails, unqualified signatures, and backend iteration order must not select another application.

An unowned source position that cannot resolve to one application must produce an ambiguity or missing-ownership diagnostic. Explicit namespace-wide search can return several matches with their complete identities.

The implementation must use a common identity rule for anchor lookup, prefix scope, module lookup, artifacts, configuration, and external symbols. Supplying a slash-qualified string as one application name will not substitute for the separate fields.

These rules govern emitted identities and consumer operations. An analyzer can use local references internally before it resolves them to canonical output identities.

### 5. Allow namespace-wide search with a warning and pagination

An explicit namespace-wide search supplies a namespace without an application selector. It uses the namespace prefix, such as `can://foo/`, across matching applications. A language-specific SDK view retains its language selection.

An omitted application selector in an ordinary request must not silently select this broader mode.

The SDK must emit an explicit warning that the search spans applications within the selected namespace. The warning must identify the namespace and the broader scope.

Namespace-wide results must be paginated. Each request returns a bounded page and continuation information. Pagination must bound database retrieval and SDK reconstruction, rather than slice an unbounded result after loading it.

Every result must retain its namespace and owning application. Identical relative files, class names, or signatures in different applications must remain distinguishable.

| Request scope | Selector | Behavior |
|---|---|---|
| Ordinary analysis | Resolved namespace and application name | Queries one application through its existing language view. |
| Namespace-wide search | Explicit search across a namespace without an application selector | Warns explicitly and returns paginated results across matching applications. |

The RFC will define the search interface, warning delivery, page model, size limits, and continuation rules. This decision does not introduce an unbounded namespace-wide result mode.

### 6. Require regeneration across the affected repositories

This is a coordinated, multi-repository breaking change. The new qualified format replaces the existing unqualified identity contract. Defaulted monolithic invocations also produce new IDs because both root segments remain present.

All affected producer and consumer repositories must adopt the new contract. Existing analysis data must be regenerated from source with the updated analyzers.

Regeneration includes analysis files, emitted graph snapshots, Neo4j projections, and identity-bearing caches. Downstream indexes and persisted references must be rebuilt from the new outputs.

In-place ID migration and compatibility aliases are not part of the upgrade path. Existing raw IDs become invalid. Consumers must regenerate their derived state instead of translating old handles.

New IDs alone do not remove old graph records. The regeneration workflow must replace obsolete application-owned analysis data and preserve unrelated datasets and shared entities.

The updated SDK must reject an unsupported identity contract clearly. It must not silently attach to legacy records or infer the contract from slash count. SDK attachment remains read-only.

The RFC will define contract versions, coordinated release gates, regeneration procedures, and cache invalidation rules. Regeneration is an explicit upgrade step, separate from SDK attachment.

### Alternatives considered

| Alternative | Reason not selected |
|---|---|
| Require namespace on every invocation. | Monolithic projects must retain a useful default without another required argument. |
| Collapse equal namespace and application segments. | The selected format keeps both segments and one consistent root structure. |
| Encode both values inside the existing application-name argument. | Producers and consumers still lack separate identity inputs and metadata. |
| Keep namespace only as graph metadata. | A module URI still omits the shared microservice application identity. Callers cannot use that identity as a URI scope. |

## Consequences

Distinct namespace/application pairs separate same-named services, files, and symbols. All supported language views use the same application root.

Monolithic commands remain convenient through default resolution. Their URIs contain repeated root segments, and their existing analysis data requires replacement.

The change requires coordinated work across producer configuration, output schemas, SDK configuration, query scope, local invocation, and regeneration. All existing analysis outputs and identity-bearing derived state require regeneration.

Namespace provides an identity scope. It does not establish service-call edges, analysis completeness, deployment ownership, or successful analyzer execution.

The identity contract must support references between services and language views without merging their endpoints. Namespace membership alone does not establish a call or data-flow relationship.

Namespace-wide search increases query breadth and requires application-qualified results. Explicit warnings expose the broader scope, and pagination bounds each response and its reconstruction cost.

The caller selects the application binding, and the SDK enforces it. Multi-application consumers must preserve ownership through source mapping, lookup, indexes, deduplication, and graph joins.

Namespace-qualified emission alone cannot prevent a wrong-service match after a consumer discards application identity. Consumer adoption must include scope enforcement and ambiguity diagnostics.

Version-qualified releases and application-owned occurrences add records and relationships. They preserve version differences and dependency provenance across producer outputs and SDK backends.

## Confirmation

This draft contains no implementation. The later RFC must turn the identity, scope, warning, and pagination requirements into conformance checks.

| Area | Required evidence |
|---|---|
| Language coverage | Python, Java, TypeScript/JavaScript, Go, and future language analyzers use the same namespace and application identity rules. |
| Topology coverage | All six required repository/service/language arrangements retain correct application and language ownership. |
| Source boundaries | File identities use the declared relative-path base of their owning application across the six arrangements. |
| Defaults | An omitted namespace resolves from the application name. An omitted application name resolves from the project root. |
| Fixed root | A monolith retains both identical segments. Explicit and default inputs that resolve to the same pair produce identical IDs. |
| Isolation | Two namespaces can each contain `svc-1`, `server.py`, and identical symbols without merging their identities or query results. |
| Project separation | Two applications in one namespace can contain identical relative files and symbols without collisions. |
| Language views | Existing language facades select the same qualified application root and retain their language-specific views. |
| Producer contributions | Each language view retains its owning namespace/application identity. |
| Polyglot updates | Emission order, reruns, failures, and removal of one contribution preserve sibling-language code and shared-entity relationships. |
| Semantic identity | Equal IDs denote equal entities across languages. Different external meanings and configuration extraction kinds cannot merge by spelling alone. |
| Identity families | Code, artifacts, configuration keys, externals, and embedded references use the same qualified root. Package-family identity does not erase application-specific dependency-version evidence. |
| Package releases | Different known versions produce distinct version-qualified PURLs. Equal complete release identities can be shared across applications. |
| Dependency occurrences | Applications retain separate occurrences. Multiple versions within one application retain distinct occurrence and release identities. |
| Unknown versions | Unresolved dependencies remain explicitly unresolved. Declared ranges and versionless families do not become fabricated release identities. |
| Dependency parity | JSON, Neo4j, and SDK reads preserve release and occurrence identities without name-only collapse. |
| SDK parity | Local and Neo4j analysis use the same resolved identity inputs. File and signature APIs retain application-scoped behavior. |
| Explicit binding | Caller configuration or declared defaults resolve one namespace/application pair and its language view. Namespace alone cannot select a service. |
| Missing binding | An ordinary request with no resolvable application reports the missing selector without namespace-wide fallback. |
| Canonical identity keys | Indexes, deduplication, and graph joins retain distinct canonical URIs for same-named files and symbols in different applications. |
| Analysis ownership | Call, data-flow, and taint analyses keep same-named files and functions in different services as distinct nodes. Cross-service edges retain qualified endpoints and supporting evidence. |
| Relative paths | Relative paths and declared source-root aliases resolve within the bound application and retain its canonical module URI. |
| Scope conflict | A full URI from another application or language view is rejected without a suffix fallback into the selected application. |
| Service routing | Two applications share `server.js` and overlapping line numbers. Resolution preserves the declared owner regardless of backend order. |
| Ambiguous ownership | A source position without one established owner reports ambiguity or missing ownership instead of accepting the first callable. |
| Finding joins | A finding's canonical module or callable reference cannot join to another application's same-named file or symbol. |
| Parsing | Reserved characters cannot create equivalent roots from distinct accepted pairs. Readers do not depend on fixed slash positions in diagnostic paths. |
| Breaking change | Producers and consumers adopt the new contract together. Readers reject old data, and regenerated outputs and derived references use only the new identities. |
| Regeneration | Source-based regeneration replaces affected analysis files, projections, caches, and indexes. It removes obsolete analysis records without modifying unrelated datasets. |
| Ordinary scope | Ordinary requests resolve both identity selectors and return results from only the selected application. |
| Namespace access | Namespace-wide search emits an explicit warning and preserves namespace and application identity in every result. |
| Namespace pagination | Search retrieves and reconstructs bounded pages, provides continuation information, and never requires an unbounded namespace result in memory. |

## RFC follow-up

After this ADR iteration, the RFC will specify:

1. CLI, in-process, and SDK configuration details, including local analyzer invocation, caller binding, and SDK scope enforcement.
2. Canonical schema fields, graph properties, ID-contract versioning, and generated identity helpers.
3. Segment validation, encoding, and normalization.
4. Coordinated releases across the supported language analyzers, SDK readers, and integrations that index or join their output.
5. Source-based regeneration, obsolete-record removal, cache and index invalidation, and regeneration of downstream references.
6. Namespace-wide search interfaces, warning delivery, page sizes and limits, result models, and continuation rules.
7. Conformance fixtures, diagnostics, and acceptance gates.
8. Package-release and dependency-occurrence schemas, identity keys, and PURL qualifiers for each supported ecosystem. Evidence includes manifests, constraints, dependency kinds, directness, and originating resolution sources where known. Reader reconstruction must preserve known versions and occurrence evidence.
9. Source-boundary mappings, repository/revision/root provenance, producer contributions, shared-entity lifecycle, and fixtures for all six required topologies. Specify per-producer metadata, version compatibility, availability, and aggregate completeness behavior. A failed required contribution must remain distinguishable from an empty language view. Replacement and cleanup must preserve sibling contributions and shared relationships.
10. Canonical URI use in call, data-flow, and taint analysis, lookup, identity indexes, deduplication, and graph joins. Specify source-position ownership, integration routing, declared path aliases, and scope-conflict and ambiguity diagnostics. Include same-named modules and functions, overlapping source positions, and reversed backend order in conformance fixtures.

This draft stops at the ADR. It does not start an RFC or implementation plan.

## References

- Current CLI inputs: [Python](https://github.com/codellm-devkit/codeanalyzer-python/blob/54acde45556869271c16d5fead1134a821012811/codeanalyzer/__main__.py#L95), [Java](https://github.com/codellm-devkit/codeanalyzer-java/blob/581edfaf2bab6711927887d129767a301660e3a1/src/main/java/com/ibm/cldk/CodeAnalyzer.java#L127), and [TypeScript](https://github.com/codellm-devkit/codeanalyzer-typescript/blob/beb34433b97dd7634941bbb6592f65665eece6e9/src/cli.ts#L19).
- Current SDK [connection configuration](https://github.com/codellm-devkit/python-sdk/blob/c125d486a0cf5f5d80a0e4b7fdeb2913c0a24d45/cldk/analysis/commons/backend_config.py#L94) and [Python identity scope](https://github.com/codellm-devkit/python-sdk/blob/c125d486a0cf5f5d80a0e4b7fdeb2913c0a24d45/cldk/analysis/python/neo4j/neo4j_backend.py#L479).
- Current SDK [application-local path normalization](https://github.com/codellm-devkit/python-sdk/blob/c125d486a0cf5f5d80a0e4b7fdeb2913c0a24d45/cldk/analysis/commons/keys.py#L34) and [TypeScript module lookup](https://github.com/codellm-devkit/python-sdk/blob/c125d486a0cf5f5d80a0e4b7fdeb2913c0a24d45/cldk/analysis/typescript/neo4j/neo4j_backend.py#L1389).
- Current SDK [dependency reconstruction and locked-version loss](https://github.com/codellm-devkit/python-sdk/blob/c125d486a0cf5f5d80a0e4b7fdeb2913c0a24d45/cldk/analysis/commons/artifacts.py#L82).
- [ECMA-427: Package-URL specification, first edition](https://ecma-international.org/publications-and-standards/standards/ecma-427/).
