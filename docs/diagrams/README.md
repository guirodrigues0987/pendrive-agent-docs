# Diagrams

Diagrams derived from `docs/description.md` and `docs/decisions-and-adjustments.md`
(C4 view - level 2, containers), written in Mermaid.

## Final version

Corrected after being checked against the real code (see
`docs/v1-comparison.md` and `docs/decisions-and-adjustments.md`).

- `structural.mmd` - container view.
- `sequence.mmd` - main flow, execution of a full scan.
- `sequence-failure.mmd` - failure scenario: inference server down or invalid
  response. No equivalent in v1.

## Initial version (v1)

Produced only from the prose description, without access to the code. Kept to
allow a before/after comparison.

- `structural-v1.mmd` - container view.
- `sequence-v1.mmd` - main flow.
