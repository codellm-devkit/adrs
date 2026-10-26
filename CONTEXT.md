# Code identity vocabulary

These terms distinguish the identity of analyzed code from service and deployment groupings.

## Language

**Namespace**:
A logical identity scope for a group of analyzed applications. A namespace can identify the software system that contains several service projects.
_Avoid_: Kubernetes namespace, build module

**Analyzed application**:
A project selected for analysis, with its own application name. It can represent a service, a monolith, or a project with multiple languages.
_Avoid_: Namespace, service group

**Repository**:
A source container with its own revision history. Repository boundaries do not determine service or analyzed-application boundaries.
_Avoid_: Analyzed application, service

**Producer contribution**:
One analyzer's facts and analysis state for an analyzed application and language view. Several contributions can describe one logical application.
_Avoid_: Analyzed application

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
