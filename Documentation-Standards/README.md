# Documentation Standards

[← Back to home](../README.md)

How formal technical documents should be structured and written. The goal is to make a system understandable, supportable, changeable, and verifiable, not to make the document look impressive.

---

| Note | Description |
|---|---|
| [Authoring Standard](authoring-standard.md) | The writing rules: the seven questions a document must answer, engineering-object IDs (`REQ-`, `DEC-`, `IF-` …), procedures, diagrams, tables, naming, and verification. |
| [Hierarchy Standard](hierarchy-standard.md) | The canonical Part → Chapter → Section structure and the rules that prevent bad hierarchy. |
| [Technical Document Template](technical-document-template.md) | A full Markdown template that implements both standards. Copy it to start a new controlled document. |

---

## Canonical structure

1. **System Definition**: purpose, scope, context, boundary, terminology, design basis
2. **Technical Design and Implementation**: architecture, interfaces, data model, configuration, artifacts, executable components
3. **Operations and Lifecycle Support**: runtime, schedules, maintenance, troubleshooting, backup, recovery
4. **Assurance, Security, and Acceptance**: identities and access, verification, testing, acceptance
5. **Appendices**: reusable procedures, component sheets, figure standards, authoring rules

Readers can stop at whatever depth their job needs without losing the system story.

> **Note:** The original bundle also had a LaTeX version (`latex/main.tex`, `technicaldoc.sty`) and a compiled PDF reference. Those files aren't in this repo; only the Markdown counterparts are.
