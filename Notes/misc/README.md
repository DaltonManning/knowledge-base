# Professional Technical Document System — Hierarchy Revision

This revision keeps the technical objects, tables, procedures, visual language, and engineering-writing approach from the previous system, but replaces the flat document structure with a deliberate four-part hierarchy.

## Recommended source of truth

- `latex/main.tex` — formal controlled document example
- `latex/technicaldoc.sty` — reusable presentation/style system
- `professional_technical_document_template_v2.pdf` — compiled visual reference
- `markdown/TECHNICAL_DOCUMENT_TEMPLATE.md` — living docs / Git-friendly counterpart
- `markdown/HIERARCHY_STANDARD.md` — canonical document information architecture
- `markdown/AUTHORING_STANDARD.md` — writing and engineering-content rules

## Canonical structure

1. **System Definition** — purpose, scope, context, boundary, terminology, design basis
2. **Technical Design and Implementation** — architecture, interfaces, data model, configuration, artifacts, executable components
3. **Operations and Lifecycle Support** — runtime, schedules, maintenance, troubleshooting, backup, recovery
4. **Assurance, Security, and Acceptance** — identities/access, verification, testing, acceptance
5. **Appendices** — reusable procedures, component sheets, figure standards, authoring rules

The design goal is that readers can stop at the depth appropriate to their job without losing the system story.
