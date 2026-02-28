# Competitive Analysis: React-to-AHA Migration Skill

Compared against every published blog post, case study, and guide covering React-to-HTMX/AHA migrations. Goal: identify gaps, validate our approach, and confirm unique value.

## What Exists

### Closest competitor: Contexte case study (htmx.org)

The gold standard for React-to-HTMX migration content. A SaaS company ported 21.5K LoC from React to Django + HTMX in 2 months.

**Their metrics:** 67% code reduction, -96% JS deps (255 to 9), -88% build time, -46% memory, -50-60% time-to-interactive.

**What it lacks:** Zero code examples. Zero migration patterns. Zero gotchas. It's an inspirational case study, not a how-to guide. The author explicitly caveats that "we would not expect every web application to see these sorts of numbers."

**Key difference:** They migrated to Django templates + HTMX. Our skill covers Astro + HTMX + Alpine (the full AHA stack). Different backend, different patterns.

### Other notable articles

| Article | Stack | Metrics | Code Patterns | Gotchas |
|---------|-------|---------|---------------|---------|
| Contexte (htmx.org) | React to Django + HTMX | Excellent | None | None |
| RisingStack streaming | Jinja + HTMX + Alpine | Vague | None | None |
| SoftwareSeni strategy | General framework | None | None | Risk assessment |
| Lou Franco blog | React to Django + HTMX | None | Minimal | Some |
| Builder.io comparison | HTMX vs React | Referenced | Some | None |
| Astro CRA migration | CRA to Astro (no HTMX) | None | Some | Some |
| Flavio Copes AHA Stack | Intro to AHA | None | Tutorial-level | None |

**Pattern: everyone covers React-to-HTMX OR React-to-Astro. Nobody covers both together.**

## What Our Skill Uniquely Provides

1. **Full AHA stack migration** - the only guide covering Astro + HTMX + Alpine.js together as a migration target
2. **10 specific code patterns** with before/after - no other article has this (Contexte has zero code)
3. **Same-app comparison data** - both apps built from the same spec, measured on the same machine
4. **Migration order** - prioritized route categories from easiest to hardest
5. **File count tradeoff** - honest about AHA having 15% more files
6. **8 common gotchas** - `import.meta.env`, `hx-boost` intercepting forms, `HX-Target` vs `HX-Request`, Alpine reinit after HTMX swap
7. **Performance checklist** - the hidden-images-in-display-none trap, font-display, preloads

## Our Numbers vs Contexte

| Metric | Contexte | Our Port |
|--------|----------|----------|
| Code reduction | -67% | -27% |
| JS dependencies | -96% | -91% (bundle) |
| Build time | -88% | Not measured |
| Lighthouse score | Not measured | +23-27 pts |
| LCP improvement | "50-60% faster" | -46 to -57% |

Contexte's code reduction is larger because they went from React SPA to server-rendered Django templates (Python handles more). Our port keeps the same server modules (Kysely, Supabase), so server code barely changes (-4%). The route/page code drop (-61%) is comparable.

## Assessment

The migration skill fills a clear gap. There's no published resource that:
- Covers the complete AHA stack as a migration target
- Provides detailed code patterns (not just metrics)
- Shows measured Lighthouse results from the same app on both stacks
- Documents integration gotchas specific to Astro + HTMX + Alpine

**Conclusion:** Publish as-is. The skill is differentiated and actionable.
