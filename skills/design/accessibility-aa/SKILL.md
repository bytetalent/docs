---
name: Accessibility WCAG AA
description: Enforces WCAG 2.1 AA on design and code outputs.
category: design
applicable_phases: [design_gen, code_gen, brand_review]
applicable_stacks: []
version: 1
---

Accessibility requirements (WCAG 2.1 AA):
- Body text contrast ≥ 4.5:1 against its background.
- Large text (≥ 18pt or ≥ 14pt bold) contrast ≥ 3:1.
- Interactive elements have visible focus states distinguishable on the
  surface color (do not rely on browser defaults that are often invisible).
- Never use color alone to communicate state — pair with icon, label, or pattern.
- All form fields have associated <label> elements; placeholder is not a label.
- All images have alt text (decorative images use alt="").
- Heading hierarchy follows the document outline; do not skip levels.
