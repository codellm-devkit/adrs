---
status: proposed
date: 2026-10-25
decision-makers: []
consulted: []
informed: []
---

# 10: Application identity namespaces

## Context

The current analyzers identify an analyzed application with `--app-name`, or the input directory name by default. The SDK selects that application with `application_name`. Neither interface has a separate identity namespace.

For example, two unrelated applications can each contain an `emailservice` project with an `email_server.py` file. Both projects can produce this ID:

```text
can://emailservice/python/email_server.py
```

The service name separates projects within one system. It does not distinguish the same project name across systems in a shared graph. Additional graph properties do not change that URI identity.

The current SDK can accept `foo/emailservice` as one `application_name` string. That compatibility does not give either producer or consumer separate namespace and application concepts.

The terminology in this decision is:

| Term | Meaning | Example |
|---|---|---|
| Namespace | A logical identity scope for a group of analyzed applications. | `foo` |
| Analyzed application | The project selected for analysis, identified by `application_name` or `--app-name`. | `svc-1` |
| Service | A logical component of a software system. An analyzed application can represent a service or a monolith. | An email service |
| Build module | An ecosystem-declared build unit, as defined in ADR 20. | A Maven module |

An analyzed application can contain multiple languages and build modules. Namespace, analyzed application, service, and build module are distinct concepts.

### Required repository and application topologies

The objective is general multi-service, multi-language analysis. Repository count, service count, and language count are independent dimensions.

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

Repository boundaries do not establish application boundaries. Several services can share a repository, and one analyzed application can contain several languages.

## Decision

We will add namespace as an explicit identity input to the Python, Java, and TypeScript/JavaScript analyzers and the Python SDK.

This change has immediate priority and will ship through the current analyzers and SDK. It does not depend on the shared core or facts-IR migration in ADRs 20 and 21.

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

The caller must choose distinct namespaces for unrelated application groups that require isolation within the same identity store. Default resolution alone cannot guarantee uniqueness between unrelated projects with identical directory names.

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

Both segments remain present, including when their values match:

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

The build-module concept described in ADR 20, section 6.2, remains distinct from application identity. Its future core implementation is not a prerequisite for this change.

The same source revision retains the same code identity across deployment environments. Deployment environment does not select an identity namespace automatically.

### 3. Apply the root consistently

The qualified root will apply to every existing application-owned `can://` identity. This includes code, body nodes, artifacts, configuration keys, and external symbols.

```text
can://foo/svc-1/python/server.py
can://foo/svc-1/artifact/requirements.txt
```

#### Polyglot applications and shared entities

One logical namespace/application pair can have several producer and language contributions. Selecting a language view must preserve the owning application identity.

Equal IDs must identify the same semantic entity even when a consumer discards Neo4j labels. Different runtime or language meanings must remain distinct.

External-symbol identities must retain the language/runtime and dependency context needed to distinguish their meanings. Matching module and symbol spellings alone do not establish cross-language identity.

Artifacts can share identity across producers only for the same source file within the declared application scope. Configuration keys also require a common extraction-kind and key grammar.

Re-analysis, retries, replacement, and cleanup must preserve other producer contributions. Removing one language's contribution must not delete shared artifacts or detach another language's relationships.

Application enumeration must identify the logical application and distinguish its language views.

Cache reuse must preserve the selected namespace, application, producer, language view, and source context. Reused facts must not retain IDs from another selected scope.

The RFC will specify canonical shared-entity suffixes and the physical root/contribution representation. Producer metadata, compatibility checks, availability, and completeness behavior belong in that specification.

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

This is proposed API, not an example supported by the current SDK:

```python
analysis = CLDK.python(
    backend=Neo4jConnectionConfig(
        uri="bolt://localhost:7688",
        namespace="foo",
        application_name="svc-1",
    )
)
```

The selected application root is `can://foo/svc-1`. Its descendant scope is `can://foo/svc-1/`.

The language-specific factory selects the language view. Ordinary requests resolve both namespace and application name to select one analyzed application. File paths, signatures, and class names retain their existing meaning within that view.

The implementation must use a common identity rule for anchor lookup, prefix scope, module lookup, artifacts, configuration, and external symbols. Supplying a slash-qualified string as one application name will not substitute for the separate fields.

### 5. Allow namespace-wide search with a warning and pagination

Namespace-wide search supplies a namespace without an application selector. It uses the namespace prefix, such as `can://foo/`, across matching applications. A language-specific SDK view retains its language selection.

The SDK must emit an explicit warning that the search spans applications within the selected namespace. The warning must identify the namespace and the broader scope.

Namespace-wide results must be paginated. Each request returns a bounded page and continuation information. Pagination must bound database retrieval and SDK reconstruction, rather than slice an unbounded result after loading it.

Every result must retain its namespace and owning application. Identical relative files, class names, or signatures in different applications must remain distinguishable.

| Request scope | Selector | Behavior |
|---|---|---|
| Ordinary analysis | Resolved namespace and application name | Queries one application through its existing language view. |
| Namespace-wide search | Namespace without an application selector | Warns explicitly and returns paginated results across matching applications. |

The RFC will define the search interface, warning delivery, page model, size limits, and continuation rules. This decision does not introduce an unbounded namespace-wide result mode.

### 6. Require regeneration across the affected repositories

This is a coordinated, multi-repository breaking change. The new qualified format replaces the existing unqualified identity contract. Defaulted monolithic invocations also produce new IDs because both root segments remain present.

All affected producer and consumer repositories must adopt the new contract. Existing analysis data must be regenerated from source with the updated analyzers.

Regeneration includes analysis files, emitted graph snapshots, Neo4j projections, and identity-bearing caches. Downstream indexes and persisted references must be rebuilt from the new outputs.

In-place ID migration and compatibility aliases are not part of the upgrade path. Existing raw IDs become invalid. Consumers must regenerate their derived state instead of translating old handles.

New IDs alone do not remove old graph records. The regeneration workflow must replace obsolete application-owned analysis data and preserve unrelated datasets and shared entities.

The updated SDK must reject an unsupported identity contract clearly. It must not silently attach to legacy records or infer the contract from slash count. SDK attachment remains read-only.

The RFC will define contract versions, coordinated release gates, regeneration procedures, and cache invalidation rules. Regeneration is an explicit upgrade step, separate from SDK attachment.

### 7. Ship independently of the shared core migration

The current analyzer emitters and SDK readers will implement the namespace contract before the shared core migration. They must use the same identity grammar and pass common conformance checks.

Namespace delivery does not require a new shared engine, a facts stream, or prior extraction of the identity code into a shared library.

ADR 20 proposes canonical schema ownership in `codeanalyzer/schema`. ADR 21 assigns canonical URI generation to the future shared core. That core must adopt the namespace contract after this change lands.

Backend facts remain structured local references under ADR 21. Later core adoption must preserve the namespace-qualified identities without introducing another URI grammar.

ADR 20, section 8, proposes unchanged output during core migration. Its comparison baseline must include the namespace contract delivered by ADR 10. The namespace change is a separate breaking release, rather than part of that byte-identical refactor.

Go-specific analyzer and SDK work is outside this ADR iteration. Namespace support applies to the existing Python, Java, and TypeScript/JavaScript paths.

### Alternatives considered

| Alternative | Reason not selected |
|---|---|
| Require namespace on every invocation. | Monolithic projects must retain a useful default without another required argument. |
| Collapse equal namespace and application segments. | The selected format keeps both segments and one consistent root structure. |
| Encode both values inside the existing application-name argument. | Producers and consumers still lack separate identity inputs and metadata. |
| Keep namespace only as graph metadata. | The same URI can still identify unrelated projects in a shared store. |

## Consequences

Distinct namespace/application pairs separate same-named services, files, and symbols. All supported language views use the same application root.

Monolithic commands remain convenient through default resolution. Their URIs contain repeated root segments, and their existing analysis data requires replacement.

The change requires coordinated work across producer configuration, output schemas, SDK configuration, query scope, local invocation, and regeneration. All existing analysis outputs and identity-bearing derived state require regeneration.

Current producers and SDK readers must ship coordinated namespace support without waiting for ADRs 20 and 21. The future core will inherit this identity contract.

Namespace provides an identity scope. It does not establish service-call edges, analysis completeness, deployment ownership, or successful analyzer execution.

The identity contract must support references between services and language views without merging their endpoints. Namespace membership alone does not establish a call or data-flow relationship.

Namespace-wide search increases query breadth and requires application-qualified results. Explicit warnings expose the broader scope, and pagination bounds each response and its reconstruction cost.

Version-qualified releases and application-owned occurrences add records and relationships. They preserve version differences and dependency provenance across producer outputs and SDK backends.

## Confirmation

This draft contains no implementation. The later RFC must turn the identity, scope, warning, and pagination requirements into conformance checks.

| Area | Required evidence |
|---|---|
| Independent delivery | Namespace support works with the current analyzers and SDK without the shared core or a facts-IR migration. |
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
| Parsing | Reserved characters cannot create equivalent roots from distinct accepted pairs. Readers do not depend on fixed slash positions in diagnostic paths. |
| Breaking change | Producers and consumers adopt the new contract together. Readers reject old data, and regenerated outputs and derived references use only the new identities. |
| Regeneration | Source-based regeneration replaces affected analysis files, projections, caches, and indexes. It removes obsolete analysis records without modifying unrelated datasets. |
| Ordinary scope | Ordinary requests resolve both identity selectors and return results from only the selected application. |
| Namespace access | Namespace-wide search emits an explicit warning and preserves namespace and application identity in every result. |
| Namespace pagination | Search retrieves and reconstructs bounded pages, provides continuation information, and never requires an unbounded namespace result in memory. |

## RFC follow-up

After this ADR iteration, the RFC will specify:

1. CLI, in-process, and SDK configuration details, including local analyzer invocation.
2. Canonical schema fields, graph properties, ID-contract versioning, and generated identity helpers.
3. Segment validation, encoding, and normalization.
4. Immediate release sequencing across current producers and SDK readers, followed by later adoption in the shared core.
5. Source-based regeneration, obsolete-record removal, cache and index invalidation, and regeneration of downstream references.
6. Namespace-wide search interfaces, warning delivery, page sizes and limits, result models, and continuation rules.
7. Conformance fixtures, diagnostics, and acceptance gates.
8. Package-release and dependency-occurrence schemas, identity keys, and PURL qualifiers for each supported ecosystem. Evidence includes manifests, constraints, dependency kinds, directness, and originating resolution sources where known. Reader reconstruction must preserve known versions and occurrence evidence.
9. Source-boundary mappings, repository/revision/root provenance, producer contributions, shared-entity lifecycle, and fixtures for all six required topologies. Specify per-producer metadata, version compatibility, availability, and aggregate completeness behavior. A failed required contribution must remain distinguishable from an empty language view. Replacement and cleanup must preserve sibling contributions and shared relationships.

This draft stops at the ADR. It does not start an RFC or implementation plan.

## References

- [ADR 20: shared core and thin language backends](20-shared-core-with-thin-language-backends.md).
- [ADR 21: facts IR and canonical URI ownership](21-facts-ir.md).
- Current CLI inputs: [Python](https://github.com/codellm-devkit/codeanalyzer-python/blob/54acde45556869271c16d5fead1134a821012811/codeanalyzer/__main__.py#L95), [Java](https://github.com/codellm-devkit/codeanalyzer-java/blob/581edfaf2bab6711927887d129767a301660e3a1/src/main/java/com/ibm/cldk/CodeAnalyzer.java#L127), and [TypeScript](https://github.com/codellm-devkit/codeanalyzer-typescript/blob/beb34433b97dd7634941bbb6592f65665eece6e9/src/cli.ts#L19).
- Current SDK [connection configuration](https://github.com/codellm-devkit/python-sdk/blob/c125d486a0cf5f5d80a0e4b7fdeb2913c0a24d45/cldk/analysis/commons/backend_config.py#L94) and [Python identity scope](https://github.com/codellm-devkit/python-sdk/blob/c125d486a0cf5f5d80a0e4b7fdeb2913c0a24d45/cldk/analysis/python/neo4j/neo4j_backend.py#L479).
- Current SDK [dependency reconstruction and locked-version loss](https://github.com/codellm-devkit/python-sdk/blob/c125d486a0cf5f5d80a0e4b7fdeb2913c0a24d45/cldk/analysis/commons/artifacts.py#L82).
- [ECMA-427: Package-URL specification, first edition](https://ecma-international.org/publications-and-standards/standards/ecma-427/).
