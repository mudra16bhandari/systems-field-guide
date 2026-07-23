---
name: engineering-concept-writer
description: Generates a clear, high-signal engineering concept article directly in <category>/<concept-slug>.md when invoked with a topic (e.g. /engineering-concept-writer <topic>). Automatically updates README.md and TOPICS.md.
---

# Engineering Concept Writer

Use this skill whenever the user invokes `/engineering-concept-writer <topic>` or asks to write an engineering concept article for a specific topic.

---

## ⚡ Command Execution Steps

When invoked with `/engineering-concept-writer <topic>`:

1. **Identify Category & Slug**:
   - Map `<topic>` to its appropriate category folder (e.g., `networking`, `backend`, `distributed-systems`, `databases`, `caching`, `messaging`, `devops`, `performance`, `security`, `mobile`, `frontend`, `ai-systems`).
   - Format the filename as `<category>/<concept-slug>.md` (e.g., `distributed-systems/idempotency.md`, `databases/acid.md`).

2. **Generate Concept Article**:
   - Follow [`templates/concept-template.md`](file:///Users/mudra_bhandari/Desktop/systems-field-guide/templates/concept-template.md).
   - **Tone**: Simple, clear, easy to understand. No dense textbook jargon.
   - **Visual Diagrams**: Use standard, universally compatible `graph TD` or `graph LR` Mermaid flowcharts with icons.
   - **Modular Sections**: Include *only* the sections that add real value for this specific concept. Include code snippets *only* if they add practical value.

3. **Update Navigation**:
   - **Update `README.md`**: Add `- [Concept Name](<category>/<concept-slug>.md)` under its category section.
   - **Update `TOPICS.md`**: Update entry to `- [x] **[Concept Name](<category>/<concept-slug>.md)** *(Completed)*`. If the topic is new, add it under its category section in `TOPICS.md`.

---

## 📐 Suggested Sections (Include Only What Makes Sense)

- `# Title`
- `## Definition` (1-2 simple sentences)
- `## Why it Exists` (What problem it solves & what happens without it)
- `## Intuition` (Simple daily life analogy)
- `## Engineering Story` (Real-world scenario e.g. Stripe, Netflix, Uber)
- `## How it Works` (Step-by-step breakdown)
- `## Diagram` (Standard `graph TD` Mermaid flowchart with component icons)
- `## Code Example` *(Include ONLY if code adds value)*
- `## Advantages & Limitations`
- `## Tradeoffs` *(Include ONLY if relevant)*
- `## Common Mistakes` *(Include ONLY if relevant)*
- `## Related Concepts`
- `## Interview Questions` *(Include ONLY if relevant)*
- `## TLDR` (3-5 simple takeaway points)
