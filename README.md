# ADRs

Architecture Decision Records for the codellm-devkit project.

## Structure

- [`codeanalyzer/`](codeanalyzer/) — ADRs for the codeanalyzer component
- [`sdk/`](sdk/) — ADRs for the sdk component

## Conventions

- ADRs are maintained per component, in their respective folder.
- One file per decision, numbered sequentially within a component (e.g. `0001-xxx.md`).
- Each ADR follows the [template](template.md): YAML front matter (`status`, `date`, `decision-makers`, `consulted`, `informed`, per [MADR 4.0](https://adr.github.io/madr/)), then Title, Context, Decision, Consequences, Confirmation.
- Statuses: `proposed`, `accepted`, `rejected`, `deprecated`, `superseded by ADR-NNNN`. An accepted or rejected ADR is immutable; a change is a new ADR that supersedes it.
- New decisions are added to the index in the component's `README.md`.
