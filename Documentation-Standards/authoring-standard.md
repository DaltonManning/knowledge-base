# Engineering Technical Documentation Standard

[← Back to index](README.md)

This is the writing standard that sits behind the templates in this bundle. The goal is not to make documents look impressive; the goal is to make systems understandable, supportable, changeable, and verifiable.


## 1. Document hierarchy comes first

Use this structural order for formal system documents:

```text
System Definition
  → Technical Design and Implementation
    → Operations and Lifecycle Support
      → Assurance, Security, and Acceptance
```

Within that structure, use **Parts** for major engineering concerns, **Chapters** for stable technical subjects, **Sections** for one concern inside a chapter, and **Subsections** only when the section genuinely contains multiple independent concepts. Requirements, decisions, interfaces, risks, tests, and procedures are engineering objects with stable IDs; they are not additional heading levels.

The complete rules are in the [Hierarchy Standard](hierarchy-standard.md).

## 2. Core principle

A professional technical implementation document answers seven questions in this order:

1. **Why does the system exist?** — purpose, scope, audience, requirements.
2. **What is the system?** — boundary, architecture, components, ownership.
3. **How does information move?** — interfaces, protocols, data contracts, timing.
4. **What makes this installation unique?** — configuration, tag mappings, files, schedules, identities, paths.
5. **How is it operated and changed?** — procedures, maintenance, deployment, rollback.
6. **How does it fail and how is it recovered?** — diagnostics, failure modes, backup/recovery.
7. **How do we prove it is correct?** — tests, acceptance criteria, evidence, revision history.

If a document contains procedures but not the first three items, it is an instruction sheet, not a complete technical document.

## 3. Information types must stay distinct

Use explicit labels/IDs for information with different engineering meaning:

| Type | Prefix | Purpose |
|---|---|---|
| Requirement | `REQ-` | Defines behavior that must be true |
| Design decision | `DEC-` | Records a chosen design and why |
| Interface | `IF-` | Defines a boundary/contract between components |
| Risk | `RSK-` | Records a failure condition, consequence, mitigation |
| Configuration item | `CI-` | Identifies a controlled artifact/value |
| Procedure | `PRC-` | Defines a repeatable controlled action |
| Test | `TST-` | Defines how correctness is verified |
| Change | `CR-` | Connects a modification to reason, implementation, and evidence |

Do not hide requirements inside explanatory prose. Do not write opinions as if they are requirements. Do not use a screenshot as the only record of a configuration value.

## 4. Section pattern

Every major section should begin with a one- or two-sentence **section intent**. This tells the reader what question the section answers.

For dense topics, use a left-label/right-content pattern:

- **Purpose** — why it exists.
- **Source** — where the information originates.
- **Trigger** — what causes execution.
- **Inputs** — data and parameters consumed.
- **Outputs** — artifacts or state produced.
- **Dependencies** — what must exist for success.
- **Failure behavior** — what happens when it goes wrong.
- **Recovery** — how to return to a known state.
- **Verification** — how to prove success.

This pattern is especially effective for scripts, services, controllers, interfaces, scheduled jobs, calculations, and report generators.

## 5. Diagrams

A technical diagram should answer one engineering question.

Use these diagram types deliberately:

- **Context diagram** — what is inside/outside the system boundary.
- **Architecture diagram** — major components and connections.
- **Data-flow diagram** — where information originates and where it goes.
- **Sequence diagram** — time/order of calls or events.
- **Network diagram** — nodes, interfaces, VLAN/subnet/security boundaries.
- **State diagram** — modes and transitions.
- **Folder/configuration tree** — artifact organization and lifecycle.

Prefer vector diagrams. Keep labels short. Put details in the caption or text, not inside tiny boxes.

## 6. Tables

Use a table when the reader must compare values across rows/columns. Do not put paragraphs into huge spreadsheet-style tables just because the source document did.

Strong table patterns:

- component responsibility matrix
- interface matrix
- data dictionary
- tag/alias map
- configuration inventory
- schedule/job matrix
- access-control matrix
- test/verification matrix
- revision history

Every table should be understandable without reading several pages of surrounding prose.

## 7. Procedures

A professional procedure has metadata before the first step:

- purpose
- owner / authorized role
- prerequisites
- impact / expected outage if applicable
- backup or rollback point
- verification method

Each step should contain one controlled action and, when useful, an expected result.

Bad:

> Change the configuration and make sure it works.

Better:

> Copy the current production configuration to the rollback location, increment the WIP revision, modify the configured alias, validate the configuration, deploy it, and verify the runtime exposes the new revision.

Best: split that sentence into separate numbered steps, each with an observable result.

## 8. Configuration and lifecycle

For systems maintained as files/configuration, make lifecycle state visible from the path or repository:

```text
PREV/   previous known-good production baseline
WIP/    work in progress / test candidate
PROD/   exact version currently used by production
```

Never allow runtime jobs to reference `WIP`. Always define what `PREV` means: immediately previous production version, selected rollback baseline, or full archive.

## 9. Naming

Names are part of the interface.

Prefer:

- `Unit_DataRptConfig.csv`
- `Unit_StatsList.csv`
- `fig-architecture-overview.pdf`
- `PRC-014-Replace-Historian-Alias.md`

Avoid:

- `newconfig2-final-final.xlsx`
- `Screenshot 10-2-26.png`
- `test.sql`

When several names refer to the same logical value, explicitly map them:

| Layer | Name |
|---|---|
| Physical historian tag | `R310_TEMP_PV` |
| Alias | `REACTOR_TEMP` |
| Batch characteristic | `MaxReactorTemp` |
| SQL column | `MaxTemp_C` |
| Report heading | `Max Reactor Temperature` |

## 10. Screenshots

Screenshots are useful for:

- proving a runtime version/status
- showing a non-obvious UI path
- identifying a specific field or control
- documenting a product that cannot export configuration as text

Screenshots are poor for:

- architecture
- long tables
- file/folder inventories
- values that can be captured as text/config
- code

Crop tightly, add a caption, and reference the exact feature that matters.

## 11. Verification

Every meaningful change should trace to a test.

A good verification states:

- controlled input or precondition
- independent source of truth
- action or stimulus
- expected result
- tolerance/precision where applicable
- evidence retained

Example:

> Query the historian independently for the batch start/end interval and calculate the maximum reactor temperature. Compare the result to characteristic `MAX_REACTOR_TEMP` in the batch record. Acceptance: absolute difference ≤ 0.1 °C and timestamps correspond to the same batch designators.

## 12. Troubleshooting

Troubleshoot in the order of the data path:

```text
source → transport/historian → trigger/context → calculation → extraction → publication → consumer
```

Determine the last stage known correct, then test the next boundary. Preserve failed-state logs and files before restarting or clearing anything.

## 13. Recommended tool split

### Markdown

Use for living documentation, notes, repositories, knowledge bases, issue-driven work, and anything frequently edited with Git. Mermaid/D2 can handle most system diagrams. Markdown diff/review is excellent.

### LaTeX

Use for controlled engineering deliverables, long formal implementation documents, stable pagination, revision-controlled issued PDFs, complex cross-references, long tables, and highly consistent typography.

### Practical workflow

Author day-to-day details in Markdown if that is faster. Keep formal issued documents in the LaTeX template. Do not duplicate uncontrolled facts in both places: decide which artifact is authoritative for each subject.
