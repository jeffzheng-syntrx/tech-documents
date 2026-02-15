# AGENTS.md

## Purpose
This repository stores guidance and reusable templates for generating engineering technical design documents.

## When responding to a request for a tech design document
- Use the template in `tech-design-doc-template.md`.
- Output **Markdown only**.
- Keep the section order and headings exactly as defined in the template.
- Target audience: Senior Engineers and QA Manager.
- Be concise, professional, and implementation-ready.
- Avoid vague claims unless attached to measurable targets.
- Always include failure modes, edge cases, and test mapping.
- Assume backend stack is **Java 18 + Spring Boot** unless explicitly stated otherwise.

## Missing information handling
If required inputs are missing:
1. Add `# Open Inputs` as the first section with up to 5 specific questions.
2. Add `# Assumptions` immediately after `Open Inputs`.
3. Continue and complete the rest of the document.

## Quality checklist before finalizing
- Measurable NFR targets exist (TPS, latency, availability, retention, security, ops limits).
- Functional scenarios include normal/alternate/failure flows and edge-case notes.
- Test strategy contains a scenario-to-test coverage matrix.
- Migration/rollout and rollback are explicit when applicable.
- Risks are captured with impact, mitigation, and clear owner.
