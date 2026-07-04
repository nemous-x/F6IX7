# Stack skill: Frontend design

Injected for `frontend-ui` / `frontend-design` areas, alongside the framework skill.

- Check `memory/global/design-system.md` first: tokens, spacing scale, typography, component inventory. Never invent values the system already defines.
- Visual hierarchy: one primary action per view; size/weight/color express importance, not decoration.
- Consistency beats novelty — reuse existing screen patterns (list/detail, form layout, empty states) from the domain's ui.md.
- Every interactive state designed: default, hover, focus-visible, active, disabled, loading, error, empty.
- Responsive behavior specified before implementation: what stacks, what hides, what scrolls, at which breakpoints.
- Accessibility baseline: 4.5:1 text contrast, visible focus, touch targets ≥44px, semantic landmarks, labels on every input.
- Motion is functional (orientation, feedback), short (≤200ms for micro-interactions), and respects reduced-motion.
- Copy is part of design: sentence case, verbs on buttons, human error messages with a next step.
