# ADRs

Architecture Decision Records for the codellm-devkit project.

## Structure

- [`codeanalyzer/`](codeanalyzer/) — ADRs for the codeanalyzer component
- [`sdk/`](sdk/) — ADRs for the sdk component

## Conventions

- ADRs are maintained per component, in their respective folder.
- One file per decision, numbered sequentially within a component (e.g. `0001-xxx.md`).
- Each ADR follows the [template](template.md): Title, Status, Context, Decision, Consequences.
- Statuses: `proposed`, `accepted`, `deprecated`, `superseded by ADR-XXXX`.
- New decisions are added to the index in the component's `README.md`.
