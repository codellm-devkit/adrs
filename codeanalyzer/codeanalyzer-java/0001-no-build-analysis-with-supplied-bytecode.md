---
status: proposed
date: 2026-10-20
decision-makers: [Rahul Krishna]
consulted: []
informed: []
scope: Java analysis without a target build, supplied bytecode, and missing library models
related: [../0001-shared-core-with-thin-language-backends.md, ../0002-facts-ir.md]
---

# 0001: Analyze Java without a target build and accept supplied bytecode

## 1. Context

An enterprise customer can supply source code and compiled artifacts without permission for the analyzer to run their build. Internal dependencies can remain unavailable. The analyzer needs to report useful facts and describe the gaps.

Source, application bytecode, and dependency metadata provide different information. Application bytecode does not contain every fact about its dependencies. A missing library can affect calls between application methods through inheritance, callbacks, and values that cross the library boundary.

This document is an RFC. The CLI names, default policy, and library model remain proposals. It does not authorize implementation or settle runtime packaging.

### 1.1 Current behavior

The reviewed Java checkout is commit `56b044fc5c86159226d7c07e755ef32356d82041`. Its [build configuration](https://github.com/codellm-devkit/codeanalyzer-java/blob/56b044fc5c86159226d7c07e755ef32356d82041/build.gradle) pins WALA 1.6.7.

| Existing input or flag | Current behavior |
| --- | --- |
| `--input` | Takes a project directory. The source parser discovers source roots under that directory. |
| `--analysis-level` | Defaults to 1. For full-project analysis, a value greater than 1 enters the WALA path. |
| `--no-build` | Skips compilation in the WALA path. It does not skip the earlier dependency collection. |
| `--build-cmd` | Selects a custom build command. Without this flag or `--no-build`, the WALA path selects an automatic build. |
| `--no-clean-dependencies` | Retains the dependency collection directory. It does not disable dependency collection. |

For project analysis, `CodeAnalyzer.run` calls `downloadLibraryDependencies` before source extraction. Maven runs `dependency:copy-dependencies`. Gradle runs an injected dependency task. This path remains active with `--no-build`, including at level 1. The current flag therefore does not promise an offline run or freedom from target build code. See [CodeAnalyzer](https://github.com/codellm-devkit/codeanalyzer-java/blob/56b044fc5c86159226d7c07e755ef32356d82041/src/main/java/com/ibm/cldk/CodeAnalyzer.java) and [BuildProject](https://github.com/codellm-devkit/codeanalyzer-java/blob/56b044fc5c86159226d7c07e755ef32356d82041/src/main/java/com/ibm/cldk/utils/BuildProject.java).

The scope builder scans application `.class` files under the project directory. It adds collected dependency JARs and reads platform classes from `$JAVA_HOME/jmods`. It rejects an empty set of application classes. It has no CLI input for separate application artifacts or a supplied classpath. See [ScopeUtils](https://github.com/codellm-devkit/codeanalyzer-java/blob/56b044fc5c86159226d7c07e755ef32356d82041/src/main/java/com/ibm/cldk/utils/ScopeUtils.java).

The Java backend currently constructs graphs itself. Under the proposed shared architecture, it remains a Java fact generator. The Rust core builds and emits the application graphs. This draft preserves that boundary from [the shared-core ADR](../0001-shared-core-with-thin-language-backends.md).

### 1.2 WALA already has fallback hierarchy modes

The current backend calls `ClassHierarchyFactory.make`. In WALA 1.6.7, this selects `MissingSuperClassHandling.NONE`. WALA also provides `makeWithRoot` and `makeWithPhantom`. The former substitutes the hierarchy root for a missing superclass. The latter creates phantom superclasses. Its API documentation marks phantom support as incomplete, identifies known defects, and recommends the root mode. These are existing WALA features. See [ClassHierarchyFactory at v1.6.7](https://github.com/wala/WALA/blob/v1.6.7/core/src/main/java/com/ibm/wala/ipa/cha/ClassHierarchyFactory.java).

With the default mode, an unresolved superclass can cause class exclusion. Missing interfaces follow a different path: the bytecode loader records a warning and omits an unresolved interface. A missing interface therefore does not automatically remove the application class. See [ClassHierarchy](https://github.com/wala/WALA/blob/v1.6.7/core/src/main/java/com/ibm/wala/ipa/cha/ClassHierarchy.java) and [BytecodeClass](https://github.com/wala/WALA/blob/v1.6.7/core/src/main/java/com/ibm/wala/classLoader/BytecodeClass.java).

The pinned `PhantomClass` throws `UnsupportedOperationException` for several member queries, including `getDeclaredMethods`. A fallback mode needs evaluation with the actual loader and analysis passes. The presence of its API does not establish compatibility. See [PhantomClass at v1.6.7](https://github.com/wala/WALA/blob/v1.6.7/core/src/main/java/com/ibm/wala/classLoader/PhantomClass.java).

### 1.3 Averroes and hierarchy completion solve different problems

Averroes generates replacement library bytecode that models possible library behavior under the separate compilation assumption. Its model includes allocations, callbacks, exceptions, and field or array accesses.

The original algorithm reads application references **and metadata from referenced library classes and their ancestors**. It can omit dependencies used only inside library implementations. Its claim does not establish analysis with every library definition absent. Reflection also requires explicit treatment. See [Ali and Lhoták, ECOOP 2013, sections 2 and 3](https://plg.uwaterloo.ca/~olhotak/pubs/ecoop13.pdf).

The published tool accepts application JARs, a Java runtime directory, and optional library JARs. Its interface does not establish that arbitrary missing libraries are safe. Compatibility with this backend and current target bytecode requires a separate evaluation. See [the Averroes repository](https://github.com/themaplelab/averroes).

JPhantom addresses class hierarchy completion more directly. It derives constraints from available bytecode and generates missing class definitions that satisfy verifier requirements. That guarantee concerns type consistency. It does not reconstruct the behavior of a missing library. See [Balatsouras and Smaragdakis, OOPSLA 2013, sections 1 and 2](https://yanniss.github.io/jphantom-oopsla13.pdf).

This evidence separates three goals: retain application classes, construct a consistent hierarchy, and model library behavior. Empty stubs or an `Object` superclass do not establish all three. A constant-pool scan alone does not justify a complete hierarchy or a sound application call graph.

## 2. Proposed decision

If this RFC is accepted, we will separate source analysis, supplied bytecode, dependency inputs, and target builds. We will evaluate existing WALA modes before adopting a generated library. We will expose missing inputs and model assumptions in diagnostics.

### 2.1 Default source analysis without a target build

The proposed default reads source and explicit local inputs. It does not run Maven, Gradle, project wrappers, annotation processors, dependency installation, or target code. It does not fetch dependencies. The analyzer still runs its own Java implementation.

This mode preserves declarations, source expressions, and unresolved references where parsing succeeds. Missing dependencies can prevent name and type resolution. Parser or resolver failures require diagnostics. This draft does not promise that every source file or call site succeeds.

Source alone does not provide application bytecode for the current WALA path. An explicit request for bytecode analysis without suitable classes must report that missing input. It must not silently claim the requested analysis completed.

### 2.2 Explicit artifact inputs

These CLI names are candidates. None exists in the reviewed Java CLI.

| Candidate | Proposed meaning | Limits still under review |
| --- | --- | --- |
| `--classes <path>` | Repeatable input for application class directories or ordinary JARs. Supplying these inputs selects analysis without a target build. | Duplicate classes, module ownership, and archive layouts need defined rules. |
| `--classpath <path>` | Repeatable input for dependency JARs or explicitly selected class directories. Both resolvers receive the applicable inputs. | Ordering, directory expansion, and platform selection need defined rules. |
| `--phantom-library` | Earlier candidate name for a missing-library policy. | A boolean hides the difference between hierarchy repair and behavioral models. The name and default remain open. |

The proposal keeps source under `--input` for source locations and declarations. Classes can come from a separate CI artifact. A match between source and bytecode requires evidence beyond a shared class name. The analyzer must report absent or uncertain matches and must not invent source spans.

An entire Maven cache is not an exact classpath. It can contain several versions of the same class. A dependency manifest or explicit artifact list can select the intended versions without Maven execution.

WARs, EARs, and executable JARs need layout-aware handling. Nested libraries and classloader boundaries need explicit rules. Their file extensions alone do not establish support or complete dependencies. The initial implementation scope can remain class directories and ordinary JARs.

Platform classes remain necessary for the WALA analysis. The proposal requires an explicit, compatible platform input and a diagnostic for missing or incompatible classes. Whether the interface selects a JDK directory or another platform artifact remains open.

### 2.3 Separate library models from artifact collection

The preferred baseline uses supplied library definitions. For incomplete inputs, the evaluation compares these alternatives:

| Alternative | Purpose | Main limitation |
| --- | --- | --- |
| WALA default mode | Establish current behavior and exclusions. | Missing superclasses can remove application classes. |
| WALA root mode | Retain classes through a hierarchy approximation. | It discards missing superclass relationships. |
| WALA phantom mode | Preserve a placeholder for a missing superclass. | The pinned implementation has known limitations. |
| Hierarchy completion, such as JPhantom | Generate definitions consistent with available bytecode constraints. | Consistent types do not supply library behavior. |
| Averroes-style behavioral model | Approximate library effects under stated assumptions. | It needs suitable metadata, analysis support, and explicit assumptions. |

Any generated definition must remain distinguishable from a supplied definition. Supplied definitions take precedence. Conflicts and unsupported constraints produce diagnostics.

Simple placeholder bodies do not model callbacks, returned objects, heap effects, or reflective behavior. A framework entrypoint rule can expose a handler as an entrypoint. It does not recover the library call that invokes that handler.

The draft proposes an explicit experimental policy for incomplete libraries. It does not enable placeholders automatically whenever the classpath is incomplete. The evaluation must establish the supported cases before a default changes.

### 2.4 Facts and diagnostics

Java-specific parsing, resolution, bytecode interpretation, and library models remain in the Java backend. The backend emits facts about declarations, calls, types, and engine-derived relations. The Rust core combines these facts into the application graphs.

An unresolved target remains unresolved unless evidence supports an external reference. A guessed dependency identity must not become a resolved `extref`. Generated library classes must not appear as application source declarations.

Diagnostics identify missing types, affected application classes, exclusions, approximation policies, and source-to-bytecode mismatches. A run report includes supplied, retained, and excluded application class counts. Machine-readable model details beyond existing diagnostics need an agreed schema.

The [facts IR draft](../0002-facts-ir.md) defines a closed `call.prov` vocabulary. It has no `phantom`, `assumed`, or `averroes` value. This ADR does not introduce those values implicitly. If diagnostics cannot provide sufficient attribution, per-fact model provenance requires an explicit IR decision.

### 2.5 Explicit build permission

A target build remains a separate opt-in convenience. Its CLI spelling and interaction with existing flags remain open.

The no-build contract must cover dependency collection and build-tool discovery as well as compilation. It must also prevent changes to supplied dependency directories. Reusing the current `--no-build` name requires a compatibility decision because its behavior changes.

The shared-core ADR still describes build-tool materialization and module ownership from build systems. Acceptance of this proposal requires reconciliation of those sections. Offline module ownership needs supplied metadata or an explicit unknown state. Packaging and runtime distribution remain outside this ADR.

## 3. Consequences

Customers can supply artifacts from their own CI without a build inside the analyzer. Source-only analysis remains useful without private dependencies. Supplied dependency metadata can improve resolution without dependency installation.

Incomplete libraries can still remove targets, introduce speculative targets, or prevent an analysis pass. Even complete dependency inputs do not establish whole-program soundness. Entrypoints, reflection, native methods, dynamic loading, and the selected analysis remain relevant.

Generated models introduce maintenance work and additional assumptions. Broader behavioral models can increase false targets and analysis cost. This draft makes no claim about implementation size, recovered edge percentage, or parity with complete dependencies.

## 4. Confirmation

This ADR contains no implementation. Acceptance requires a documented evaluation before selection of a missing-library policy:

1. **Current behavior.** A minimal fixture distinguishes a missing superclass, a missing interface, and a missing method. The report records exclusions and warnings.
2. **No target execution.** A future integration check rejects any Maven, Gradle, wrapper, annotation-processor, or dependency-install process in the proposed default mode. A read-only input tree and disabled network expose unintended writes or downloads.
3. **Known targets.** Small fixtures assert exact retained classes and call targets. Cases include inherited methods, overrides, constructors, callbacks, returned objects, exceptions, reflection, and unsupported bytecode.
4. **Policy comparison.** The same compiled input runs with complete dependencies, partial dependencies, and each candidate policy. The report counts exclusions, unresolved sites, target differences, failures, time, and peak memory.
5. **Independent review.** Complete-dependency output is a comparison baseline, not ground truth. Manually specified targets establish correctness for fixtures. Reviewed disagreements characterize the larger corpus.
6. **Artifact integrity.** Checks cover duplicate classes, stale source, absent debug metadata, nested archives, platform mismatches, and precedence between supplied and generated definitions.
7. **Facts contract.** The generated facts pass the agreed IR schema and semantic checks. Model assumptions remain visible without unsupported provenance values or fabricated source locations.

## 5. Open questions

1. Which artifact formats and source-to-bytecode matches belong in the first implementation?
2. Which WALA fallback works with the pinned loader, entrypoint selection, and analysis passes?
3. Does hierarchy completion provide enough value before a behavioral model is necessary?
4. What metadata and assumptions must a supported Averroes-style model require?
5. Does model attribution need an IR extension or only structured diagnostics?
6. How does the CLI introduce source-only defaults and explicit builds without ambiguous legacy flag behavior?
7. How will customers supply module ownership and the target platform without build-tool execution?
