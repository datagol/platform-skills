---
name: datagol-frontend-design
description: Create distinctive, production-grade frontend interfaces with high design quality. Use this skill when the user asks to build web components, pages, artifacts, posters, or applications (examples include websites, landing pages, dashboards, React components, HTML/CSS layouts, or when styling/beautifying any web UI). Generates creative, polished code and UI design that avoids generic AI aesthetics.
---

This skill guides creation of distinctive, production-grade frontend interfaces that avoid generic "AI slop" aesthetics. Implement real working code with exceptional attention to aesthetic details and creative choices.

The user provides frontend requirements: a component, page, application, or interface to build. They may include context about the purpose, audience, or technical constraints.

## Design Thinking

Before coding, understand the context and commit to a BOLD aesthetic direction:
- **Purpose**: What problem does this interface solve? Who uses it?
- **Tone**: Pick an extreme: brutally minimal, maximalist chaos, retro-futuristic, organic/natural, luxury/refined, playful/toy-like, editorial/magazine, brutalist/raw, art deco/geometric, soft/pastel, industrial/utilitarian, etc. There are so many flavors to choose from. Use these for inspiration but design one that is true to the aesthetic direction.
- **Constraints**: Technical requirements (framework, performance, accessibility).
- **Differentiation**: What makes this UNFORGETTABLE? What's the one thing someone will remember?

**CRITICAL**: Choose a clear conceptual direction and execute it with precision. Bold maximalism and refined minimalism both work - the key is intentionality, not intensity.

Then implement working code (HTML/CSS/JS, React, Vue, etc.) that is:
- Production-grade and functional
- Visually striking and memorable
- Cohesive with a clear aesthetic point-of-view
- Meticulously refined in every detail

## Frontend Aesthetics Guidelines

Focus on:
- **Typography**: Choose fonts that are beautiful, unique, and interesting. Avoid generic fonts like Arial and Inter; opt instead for distinctive choices that elevate the frontend's aesthetics; unexpected, characterful font choices. Pair a distinctive display font with a refined body font.
- **Color & Theme**: Commit to a cohesive aesthetic. Use CSS variables for consistency. Dominant colors with sharp accents outperform timid, evenly-distributed palettes.
- **Motion**: Use animations for effects and micro-interactions. Prioritize CSS-only solutions for HTML. Use Motion library for React when available. Focus on high-impact moments: one well-orchestrated page load with staggered reveals (animation-delay) creates more delight than scattered micro-interactions. Use scroll-triggering and hover states that surprise.
- **Spatial Composition**: Unexpected layouts. Asymmetry. Overlap. Diagonal flow. Grid-breaking elements. Generous negative space OR controlled density.
- **Responsive composition**: Design the desktop, tablet, and phone layouts as first-class experiences. Every generated interface must include explicit responsive behavior: collapse sidebars and multi-column grids, stack headers/actions, resize type with `clamp()`, keep touch targets at least ~44px, prevent horizontal overflow, and make modals/forms usable within `100svh` on phones.
- **Backgrounds & Visual Details**: Create atmosphere and depth rather than defaulting to solid colors. Add contextual effects and textures that match the overall aesthetic. Apply creative forms like gradient meshes, noise textures, geometric patterns, layered transparencies, dramatic shadows, decorative borders, custom cursors, and grain overlays.

NEVER use generic AI-generated aesthetics like overused font families (Inter, Roboto, Arial, system fonts), cliched color schemes (particularly purple gradients on white backgrounds), predictable layouts and component patterns, and cookie-cutter design that lacks context-specific character.

**NEVER use emoji as UI icons.** Emoji render inconsistently across platforms and operating systems (different shapes, sizes, colours on macOS vs Windows vs Android), look unprofessional in product UIs, and break at small sizes. Always use inline SVG icons instead:

- Create a `components/Icons.tsx` file that exports all icons as tiny React components (`(props: { size?: number; color?: string; style?: CSSProperties }) => JSX.Element`).
- Use a consistent 24×24 `viewBox`, `fill="none"`, `stroke="currentColor"`, `strokeWidth={1.5}`, `strokeLinecap="round"`, `strokeLinejoin="round"` — matching the Heroicons / Lucide style.
- Size icons via the `size` prop (default 18). Colour via `color` prop or `currentColor` inheritance.
- Never use `fontSize` + an emoji character (`>🎫</span>`) for a UI element. Replace every such instance with an SVG icon component.
- Emoji are allowed **only** in user-generated content display (e.g. rendering a message body that a user typed). Never in nav, buttons, badges, empty states, headers, or status indicators.

Interpret creatively and make unexpected choices that feel genuinely designed for the context. No design should be the same. Vary between light and dark themes, different fonts, different aesthetics. NEVER converge on common choices (Space Grotesk, for example) across generations.

**IMPORTANT**: Match implementation complexity to the aesthetic vision. Maximalist designs need elaborate code with extensive animations and effects. Minimalist or refined designs need restraint, precision, and careful attention to spacing, typography, and subtle details. Elegance comes from executing the vision well.

Remember: Claude is capable of extraordinary creative work. Don't hold back, show what can truly be created when thinking outside the box and committing fully to a distinctive vision.

## Implementation in the DataGOL Codex sandbox

Generated DataGOL apps use a Next.js + React + TypeScript stack by default.
Weave these constraints into the aesthetic decisions, not against them:

- **Global CSS + component styles are the canonical pattern.** No Tailwind config
  unless the existing repo already uses it. Use `app/globals.css` for resets,
  responsive rules, and `:root` design tokens (colors, radii, shadows, font
  stacks), plus component-level styles or inline `style={{ ... }}` where they
  keep the code simpler.
- **Custom fonts via `@import` or Next font utilities.** If using CSS imports,
  put the import at the top of `app/globals.css` (Google Fonts, Fontshare,
  fonts.bunny.net) and reference the family in `:root` font variables. Pick the
  family intentionally per the aesthetic direction — never import "Inter" by
  reflex.
- **Motion:** don't assume a motion library exists. For most generated apps, use
  CSS keyframes/transitions and React state. Only add a package such as `motion`
  after checking `package.json` and explaining why the dependency is warranted.
- **Decorative SVGs / noise overlays** belong inline as React components,
  regular CSS backgrounds, or `data:` URIs in CSS — keeping the app lightweight.

When the user accepts a `datagol-detailed-plan` whose §2 (UX) describes restrained
minimalism (typical for a CRUD/admin tool), execute that with precision —
exquisite type, generous space, hairline dividers — rather than reaching
for the safest grey-and-blue defaults. When the plan describes a
landing/marketing/portfolio surface, lean into the maximalist end.

## Cross-references

- **`datagol-app-development`** invokes this skill on every design pass. The CRUD
  scaffolds it produces still need a real aesthetic; "list/detail/form"
  is the *layout*, not the *design*.
- **`datagol-detailed-plan`** §2 (UX) is where the aesthetic direction is named
  before any code. Read it back to confirm intentionality before writing
  files.
- **`datagol-agent-chat-ui`** generated chat apps benefit from this skill too —
  conversational UI is wide open aesthetically.
