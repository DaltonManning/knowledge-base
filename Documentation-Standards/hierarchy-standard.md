# Technical Document Hierarchy Standard

[← Back to index](README.md)

The document hierarchy must reflect how an engineer thinks about the system. Typography should reveal that hierarchy, not invent it.

## Canonical hierarchy

```text
DOCUMENT
├── Front Matter
│   ├── Document Control
│   ├── Revision History
│   ├── Reader Guide
│   └── Table of Contents
│
├── Part I — System Definition
│   ├── Chapter 1 — Document Purpose and Applicability
│   └── Chapter 2 — System Overview and Design Basis
│
├── Part II — Technical Design and Implementation
│   ├── Chapter 3 — System Architecture
│   ├── Chapter 4 — Interfaces and Data Model
│   ├── Chapter 5 — Configuration and Implementation Artifacts
│   └── Chapter 6 — Executable Components
│
├── Part III — Operations and Lifecycle Support
│   ├── Chapter 7 — Operations and Scheduling
│   ├── Chapter 8 — Maintenance and Change Control
│   └── Chapter 9 — Troubleshooting, Backup, and Recovery
│
├── Part IV — Assurance, Security, and Acceptance
│   ├── Chapter 10 — Security and Access Control
│   └── Chapter 11 — Verification, Testing, and Acceptance
│
└── Appendices
    ├── Procedure Template
    ├── Component Data Sheet
    ├── Figure / Evidence Standard
    └── Engineering Authoring Rules
```

## Level meanings

| Level | Meaning | Rule |
|---|---|---|
| **Document** | One controlled technical deliverable | One document title only. |
| **Part** | A major reader concern | Usually 3–5 parts. A part should group several chapters. |
| **Chapter** | A stable technical subject | A chapter should still make sense after implementation details change. |
| **Section** | One technical concern inside a chapter | Prefer 2–6 sections per chapter. |
| **Subsection** | A real subdivision of one section | Use sparingly; avoid going deeper than this in normal prose. |
| **Engineering object** | Requirement, interface, decision, risk, test, procedure | Identified by stable IDs, not heading depth. |

## Rules that prevent bad hierarchy

1. **Never use heading depth to fake visual emphasis.** A warning, requirement, or decision is an engineering object, not a chapter.
2. **Never flatten lifecycle subjects into peer sections.** Architecture, interfaces, operations, troubleshooting, and security belong under different major concerns.
3. **Do not create one-section chapters unless the subject is independently important.** Merge trivial chapters into a stronger parent chapter.
4. **Do not let implementation artifacts drive the top-level structure.** Files, scripts, tables, and screenshots belong under the engineering concept they implement.
5. **Do not mix design and procedure at the same level.** Explain what the system is before explaining how to click through a maintenance task.
6. **Keep core narrative separate from reference material.** Reusable templates, screenshots, long lookup tables, and authoring rules belong in appendices when they interrupt the main technical story.
7. **A new chapter should answer a new reader question.** If it does not, it is probably a section.
8. **Use no more than four visible heading levels in Markdown.** In LaTeX, Part → Chapter → Section → Subsection is normally enough.

## Reader logic

The canonical reading path is:

```text
Why / boundary
    ↓
What / architecture
    ↓
How / interfaces + configuration + implementation
    ↓
Run / maintain / troubleshoot / recover
    ↓
Secure / verify / accept
```

A support engineer may jump directly to Part III. A reviewer may focus on Parts I, II, and IV. A new engineer should be able to read Parts I and II without touching procedures and still understand the system.
