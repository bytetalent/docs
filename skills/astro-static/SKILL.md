---
name: Astro Static Patterns
description: Brochure-friendly Astro conventions — content collections, MDX, no JS.
category: architecture
applicable_phases: [arch_review, code_gen]
applicable_stacks: [astro-cloudflare]
version: 1
---

Astro static site patterns for brochure projects:
- Use content collections (src/content/) for marketing copy, services, team.
- MDX for any page that mixes prose with components.
- No client framework imports unless absolutely necessary; if needed, use
  client:visible directive sparingly.
- Layouts in src/layouts/; shared components in src/components/.
- Astro Image for all images; never raw <img> with external URLs.
- Tailwind via @astrojs/tailwind; brand kit values exposed as CSS variables.
