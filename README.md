# ADRs

Architecture Decision Records for the codellm-devkit project.

## Structure

- [`codeanalyzer/`](codeanalyzer/) — ADRs for the codeanalyzer component
- [`sdk/`](sdk/) — ADRs for the sdk component

## Conventions

- ADRs are maintained per component, in their respective folder.
- One file per decision, with a prefix unique within its component directory. The [filename and ordering rules](#adr-filenames-and-ordering) apply.
- Each ADR follows the [template](template.md): YAML front matter (`status`, `date`, `decision-makers`, `consulted`, `informed`, per [MADR 4.0](https://adr.github.io/madr/)), then Title, Context, Decision, Consequences, Confirmation.
- Statuses: `proposed`, `accepted`, `rejected`, `deprecated`, `superseded by ADR-<prefix>`. An accepted or rejected ADR is immutable; a change is a new ADR that supersedes it.
- New decisions are added to the index in the component's `README.md`.

## ADR filenames and ordering

- Use `<prefix>-<short-title>.md`. The prefix contains decimal digits. Its width can vary, and leading zeroes remain part of its spelling.
- Order ADRs by their complete filenames in lexicographic order, using C/POSIX character ordering. Numeric or natural sorting does not apply.
- List each component directory's ADRs in that same order in its index. References to other directories can form separate groups.
- Match the prefix in the filename, document title, index label, and associated asset names.
- Preserve existing prefixes unless renumbering is explicitly requested. Update references, indexes, titles, and asset names together after a rename.
- Express parent, dependency, and related-decision relationships explicitly in the ADR text or metadata. The prefix establishes order without implying a hierarchy.

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
