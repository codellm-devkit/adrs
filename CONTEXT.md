# Code identity vocabulary

These terms distinguish the identity of analyzed code from service and deployment groupings.

## Language

**Namespace**:
The identity of the overall application. For a microservice application, related services share this identity while each service keeps its own analyzed-application identity.
_Avoid_: Kubernetes namespace, build module

**Microservice application**:
A software application composed of related services. Its namespace identifies the whole application, and its services retain separate identities.
_Avoid_: Single service, repository

**Analyzed application**:
A project or a single service selected for analysis, with its own application name. It can also represent a monolith or contain multiple languages.
_Avoid_: Namespace, service group

**Application scope**:
The selected namespace and analyzed-application identity within which an operation applies. A namespace can contain several application scopes.
_Avoid_: Namespace alone, language view

**Canonical URI**:
The definitive URI for an entity. An application-owned entity's URI retains its namespace and analyzed-application identity across references.
_Avoid_: Display name, relative file key

**Repository**:
A source container with its own revision history. Repository boundaries do not determine service or analyzed-application boundaries.
_Avoid_: Analyzed application, service

**Producer contribution**:
One analyzer's facts and analysis state for an analyzed application and language view. Several contributions can describe one logical application.
_Avoid_: Analyzed application

**Producer**:
A component that emits analyzed-code records and their identities.
_Avoid_: Consumer, SDK caller

**Consumer**:
A component that reads, indexes, queries, or joins analyzed-code records while retaining their identities.
_Avoid_: Producer

**Service**:
A logical component of a software system. A service and an analyzed application do not require a one-to-one relationship.
_Avoid_: Build module

**Build module**:
A build unit declared by its ecosystem, such as a Maven module or an npm workspace package. It is distinct from a namespace and an analyzed application.
_Avoid_: Service, namespace

**Package family**:
A package name within its ecosystem, without a resolved version. Several package releases can belong to one family.
_Avoid_: Resolved release

**Package release**:
A known resolved package version, including the source or platform distinctions required by its package identity. Applications can share the same release.
_Avoid_: Package family, dependency occurrence

**Dependency occurrence**:
An analyzed application's dependency declaration or resolved use, with its own manifest and resolution evidence. Different occurrences can reference the same package release.
_Avoid_: Package release
