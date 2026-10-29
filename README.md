# Architecture

Architecture decisions and technical proposals for the codellm-devkit project.

## Structure

- [`adrs/`](adrs/) — Architecture Decision Records, organized by component
- [`rfcs/`](rfcs/) — Requests for Comments, organized by component
- [`templates/adr.md`](templates/adr.md) — ADR template
- [`templates/rfc.md`](templates/rfc.md) — RFC template
- [`CONTEXT.md`](CONTEXT.md) — shared terminology

## Conventions

- ADRs record decisions and their rationale. RFCs describe technical designs for review, including interfaces, compatibility, rollout, and validation.
- Records live in component directories under `adrs/` or `rfcs/`. Each record has its own file and prefix.
- ADR and RFC IDs are independent. A reference names its kind and component, such as `ADR-codeanalyzer-10` or `RFC-codeanalyzer-10`.
- Matching ADR and RFC numbers do not establish a relationship. An RFC can link to several ADRs, and one ADR can lead to several RFCs.
- Relationships use explicit relative links. RFC metadata paths resolve relative to the RFC file after it is copied from the template.
- A shared record has one canonical location. Other component indexes can link to it.
- Each ADR follows the [ADR template](templates/adr.md): YAML front matter, then Title, Context, Decision, Consequences, Confirmation, per [MADR 4.0](https://adr.github.io/madr/).
- Statuses: `proposed`, `accepted`, `rejected`, `deprecated`, `superseded by ADR-<prefix>`. An accepted or rejected ADR is immutable; a change is a new ADR that supersedes it.
- Each RFC follows the [RFC template](templates/rfc.md). Its statuses are `draft`, `proposed`, `accepted`, `rejected`, `withdrawn`, or `superseded`.
- A draft RFC is still in development. A proposed RFC is ready for discussion. Acceptance records agreement on the design. Rollout tracks implementation.
- Acceptance of an RFC does not silently change an accepted ADR. Any changed architectural decision needs its own ADR.
- New records are added to their component's index. The [filename and ordering rules](#filenames-and-ordering) apply to both document kinds.

<a id="adr-filenames-and-ordering"></a>

## Filenames and ordering

- Use `<prefix>-<short-title>.md`. The prefix contains decimal digits. Its width can vary, and leading zeroes remain part of its spelling.
- Order records by their complete filenames in lexicographic order, using C/POSIX character ordering. Numeric or natural sorting does not apply.
- List each component directory's records in that same order in its index. References to other directories can form separate groups.
- Match the prefix in the filename, document title, index label, and associated asset names.
- Preserve existing prefixes unless renumbering is explicitly requested. Update references, indexes, titles, and asset names together after a rename.
- Prefixes are unique within each document kind and component directory. ADRs and RFCs can use the same prefix independently.
- Express parent, dependency, and related-record relationships explicitly in the text or metadata. The prefix establishes order without implying a hierarchy.

For example, this is the required order:

```text
10-example.md
11-example.md
110-example.md
111-example.md
120-example.md
20-example.md
```

`111` follows `110`, precedes `120`, and precedes `20`. Filenames compare character by character, so `1` sorts before `2`.
