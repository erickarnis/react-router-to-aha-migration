---
name: react-router-to-aha-migration
description: >
  Guide for migrating a React Router 7 app to the AHA stack (Astro + HTMX + Alpine.js).
  Trigger when: porting React components to Astro, replacing React Router loaders with Astro pages,
  converting useFetcher/optimistic UI to HTMX partials, replacing React state with Alpine.js,
  migrating from client-side routing to server-rendered HTML, or evaluating AHA vs React tradeoffs.
---

# React Router to AHA Stack Migration

Patterns, tradeoffs, and measured results from porting a production React Router 7 app (50K LoC) to Astro + HTMX + Alpine.js.

## Before You Start

**"Complex interactive pages are impossible without React."** Wrong. A study session with card flip animations, keyboard shortcuts, and spaced-repetition scheduling ported successfully using Alpine.js as a state machine with HTMX POSTs.

**"You'll lose optimistic UI."** True, but it rarely matters. Server round-trips take 100-150ms. Use `hx-disabled-elt="this"` to prevent double-submit during the wait.

## Measured Results

### Code

| Metric | React Router 7 | AHA Stack | Delta |
|--------|----------------|-----------|-------|
| Files | 327 | 377 | +15% |
| Code Lines | 50,622 | 36,782 | **-27%** |
| Complexity | 4,252 | 2,857 | **-33%** |
| Routes/Pages LoC | ~8,500 | ~3,300 | **-61%** |
| Components LoC | ~6,000 | ~10,800 | +80% |

More files because every toggle button needs a partial page. Less code because no hydration, no client state management, no React hooks.

### Bundle Size

| Metric | React Router 7 | AHA Stack | Delta |
|--------|----------------|-----------|-------|
| JS gzipped | 673 KB | 62 KB | **-91%** |

### Lighthouse (Simulated Mobile 4G)

| Metric | React Router 7 | AHA Stack | Delta |
|--------|----------------|-----------|-------|
| Performance Score | 60 | 87 | **+27 pts** |
| LCP | 7,412 ms | 3,211 ms | -57% |
| Transfer Size | 1,837 KB | 808 KB | -56% |

## Route Migration Patterns

### Pattern 1: Loader → Astro Frontmatter

```tsx
// BEFORE: React Router
export async function loader({ request }: Route.LoaderArgs) {
  const species = await getSpecies()
  return { species }
}

export default function SpeciesPage({ loaderData }: Route.ComponentProps) {
  const { species } = loaderData
  return <SpeciesGrid species={species} />
}
```

```astro
---
// AFTER: Astro page
const species = await getSpecies()
---
<SpeciesGrid species={species} />
```

### Pattern 2: Action → Astro POST Handler

```tsx
// BEFORE: React Router action
export async function action({ request }: Route.ActionArgs) {
  const formData = await request.formData()
  await createDeck(formData.get("name") as string)
  return redirect("/decks")
}
```

```astro
---
// AFTER: Astro page
if (Astro.request.method === "POST") {
  const formData = await Astro.request.formData()
  await createDeck(formData.get("name") as string)
  return Astro.redirect("/decks")
}
---
<form method="POST">
  <input name="name" required />
  <button type="submit">Create</button>
</form>
```

### Pattern 3: useFetcher → HTMX Partial

The biggest conceptual shift. React's `useFetcher` becomes a component that posts to its own partial page. The partial renders the same Astro component — never duplicate component HTML.

```tsx
// BEFORE: React Router useFetcher
function FavouriteButton({ taxonId, isFavourited }: Props) {
  const fetcher = useFetcher()
  return (
    <fetcher.Form method="post" action="/api/favourite">
      <input type="hidden" name="taxonId" value={taxonId} />
      <button><HeartIcon filled={isFavourited} /></button>
    </fetcher.Form>
  )
}
```

```astro
---
// AFTER: Astro component
const { taxonId, isFavourited } = Astro.props
---
<button
  hx-post="/partials/favourite-button"
  hx-vals={JSON.stringify({ taxonId })}
  hx-target="this"
  hx-swap="outerHTML"
  hx-disabled-elt="this"
>
  <HeartIcon filled={isFavourited} />
</button>
```

```astro
---
// AFTER: Partial page (src/pages/partials/favourite-button.astro)
export const partial = true
import FavouriteButton from "@/components/FavouriteButton.astro"
import { toggleFavourite } from "@/lib/favourites.server"

if (Astro.request.method !== "POST") return new Response(null, { status: 405 })
const user = Astro.locals.user
if (!user) return new Response(null, { status: 200, headers: { "HX-Redirect": "/signin" } })

const formData = await Astro.request.formData()
const taxonId = Number(formData.get("taxonId"))
const result = await toggleFavourite(user.id, taxonId)
---
<FavouriteButton taxonId={taxonId} isFavourited={result.favourited} />
```

### Pattern 4: useSearchParams → HTMX hx-get + hx-include

```tsx
// BEFORE: React Router
const [params, setParams] = useSearchParams()
<select
  value={params.get("sort") ?? "name"}
  onChange={(e) => setParams(prev => { prev.set("sort", e.target.value); return prev })}
>
```

```html
<!-- AFTER: HTMX -->
<select
  name="sort"
  hx-get="/explore/species"
  hx-trigger="change"
  hx-target="#species-results"
  hx-push-url="true"
  hx-include="[name='region'], [name='taxon'], [name='q']"
>
```

`hx-include` sends sibling filter values with each request. `hx-push-url="true"` keeps the URL in sync.

### Pattern 5: React State Machine → Alpine State Machine

```tsx
// BEFORE: React — multiple useState hooks + event handlers
const [flipped, setFlipped] = useState(false)
const handleRating = (rating: number) => {
  fetch("/api/study", { method: "POST", body: JSON.stringify({ cardId, rating }) })
  setFlipped(false)
}
```

```html
<!-- AFTER: Alpine + HTMX -->
<div x-data="{ flipped: false }" @keydown.space.window.prevent="!flipped && (flipped = true)">
  <div x-show="!flipped" @click="flipped = true">
    <!-- front of card -->
  </div>
  <div x-show="flipped" x-cloak>
    <button
      hx-post="/partials/study-review"
      hx-vals='{"cardId": "...", "rating": 1}'
      hx-target="#study-area"
      @click="flipped = false"
    >Again</button>
    <!-- more rating buttons -->
  </div>
</div>
```

Alpine owns the flip state. HTMX POST returns the next card's HTML.

### Pattern 6: Infinite Scroll → HTMX revealed

Replace IntersectionObserver + useEffect + cleanup with a single HTMX attribute:

```html
<!-- Server renders the last card with a sentinel -->
<div
  hx-get="/partials/feed?page=3&cursor=CURSOR"
  hx-trigger="revealed"
  hx-swap="afterend"
>
  <!-- card content -->
</div>
```

Server returns next batch of cards. Last card in each batch carries the sentinel for the next page.

### Pattern 7: Multi-Region Updates → OOB Swaps

When a mutation needs to update multiple UI regions (main content + sidebar count, list + header badge):

```astro
---
// Partial returns primary content + OOB updates in one response
export const partial = true
const project = await createProject(formData.get("name") as string)
Astro.response.headers.set("HX-Push-Url", `/project/${project.id}`)
---
<div id="main-content" hx-swap-oob="true">
  <ProjectDetail project={project} />
</div>
<div id="sidebar-projects" hx-swap-oob="true">
  <ProjectsList activeId={project.id} />
</div>
```

### Pattern 8: Loading States → .htmx-request Class

```html
<!-- HTMX adds .htmx-request class during requests -->
<button class="group" hx-post="/partials/save" hx-disabled-elt="this">
  <span class="group-[.htmx-request]:hidden">Save</span>
  <span class="hidden group-[.htmx-request]:inline">Saving...</span>
</button>
```

## Migration Order

Port routes in this order (easiest to hardest):

1. **Static pages** — Landing, about. Copy HTML, add Astro frontmatter.
2. **Auth pages** — Signin, signup, reset password. POST handlers.
3. **List pages** — Server-rendered lists with URL pagination.
4. **Filter pages** — HTMX filters with `hx-include`.
5. **Detail pages** — Multiple sections, toggles, dialogs.
6. **Interactive pages** — Alpine state machines.
7. **Feed pages** — HTMX infinite scroll.

## File Count Tradeoff

AHA apps have **more files** because every toggle button needs a partial page (`src/pages/partials/`). Components can't share action handlers — each needs its own endpoint. Keep partials minimal: authenticate, call a server function, render the component.

## What NOT to Port

- **Error boundaries** — Replace with HTMX error events (`htmx:responseError`, `htmx:sendError`) in a global handler.
- **Suspense boundaries** — The server renders complete HTML. Use `loading="lazy"` for images.

## Common Gotchas

1. **`import.meta.env` not `process.env`** — Astro/Vite doesn't populate `process.env` from `.env` files.

2. **`hx-boost` is global** — Set on `<body>`, intercepts all links AND form submissions. Use `hx-boost="false"` on external links, file downloads, and forms that need full-page POST (e.g., file uploads).

3. **Fragment vs full page** — Check `HX-Target`, not just `HX-Request`. Boosted links omit `HX-Target`; filter swaps include it. A fragment returned to a boosted link replaces the entire page body.

4. **Alpine dies after HTMX swap** — `Alpine.start()` runs once. Add an `htmx:afterSettle` listener in the Layout that calls `Alpine.initTree(document.body)` to re-initialize Alpine directives in swapped DOM.
