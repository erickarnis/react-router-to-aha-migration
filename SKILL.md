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

Common objections and what we found after completing the port:

**"Complex interactive pages are impossible without React."** Wrong. A study session with card flip animations, keyboard shortcuts, and spaced-repetition scheduling ported successfully using Alpine.js as a state machine with HTMX fire-and-forget POSTs.

**"You'll lose optimistic UI."** True, but it rarely matters. Server round-trips take 100-150ms. Use `hx-disabled-elt="this"` to prevent double-submit during the wait.

**"Server modules will need a full rewrite."** Most server files copy over with just import path changes (`~/lib/` → `@/lib/`, `process.env` → `import.meta.env`).

**What may surprise you:**
- The partial page pattern creates more files than expected (components directory grew 80%)
- Total code reduction is larger than expected (27% fewer lines, 33% less complexity)
- AHA pages can be slower than React if images aren't optimized (see Performance section)

## Measured Results

### Code

| Metric | React Router 7 | AHA Stack | Delta |
|--------|----------------|-----------|-------|
| Files | 327 | 377 | +15% |
| Code Lines | 50,622 | 36,782 | **-27%** |
| Complexity | 4,252 | 2,857 | **-33%** |
| Routes/Pages LoC | ~8,500 | ~3,300 | **-61%** |
| Components LoC | ~6,000 | ~10,800 | +80% |
| Server modules LoC | ~13,000 | ~12,500 | -4% |

More files because every toggle button needs a partial page. Less code because no hydration, no client state management, no React hooks.

### Bundle Size

| Metric | React Router 7 | AHA Stack | Delta |
|--------|----------------|-----------|-------|
| JS files | 250 | 2 | -99% |
| JS gzipped | 673 KB | 62 KB | **-91%** |
| Client directory | 7.0 MB | 1.6 MB | -77% |

### Lighthouse (Simulated Mobile 4G)

**Landing page:**

| Metric | React Router 7 | AHA Stack | Delta |
|--------|----------------|-----------|-------|
| Performance Score | 66 | 89 | **+23 pts** |
| FCP | 5,411 ms | 2,931 ms | -46% |
| LCP | 5,411 ms | 2,931 ms | -46% |
| CLS | 0.000 | 0.000 | same |
| Transfer Size | 1,432 KB | 594 KB | -59% |

**Explore page (species grid with filters):**

| Metric | React Router 7 | AHA Stack | Delta |
|--------|----------------|-----------|-------|
| Performance Score | 60 | 87 | **+27 pts** |
| FCP | 5,853 ms | 3,211 ms | -45% |
| LCP | 7,412 ms | 3,211 ms | -57% |
| Transfer Size | 1,837 KB | 808 KB | -56% |

## Route Migration Patterns

### Pattern 1: Loader → Astro Frontmatter

Data fetching moves from a loader function into the Astro `---` fence.

```tsx
// BEFORE: React Router
export async function loader({ request }: Route.LoaderArgs) {
  const url = new URL(request.url)
  const page = parseInt(url.searchParams.get("page") ?? "1")
  const species = await getSpecies({ page })
  return { species, page }
}

export default function SpeciesPage({ loaderData }: Route.ComponentProps) {
  const { species, page } = loaderData
  return <SpeciesGrid species={species} page={page} />
}
```

```astro
---
// AFTER: Astro page
const url = new URL(Astro.request.url)
const page = parseInt(url.searchParams.get("page") ?? "1")
const species = await getSpecies({ page })
---
<SpeciesGrid species={species} page={page} />
```

No separate loader function, no `loaderData` typing. Fetching and rendering share one file.

### Pattern 2: Action → Astro POST Handler

React Router actions become POST handlers in the same Astro page frontmatter.

```tsx
// BEFORE: React Router action
export async function action({ request }: Route.ActionArgs) {
  const formData = await request.formData()
  const name = formData.get("name") as string
  await createDeck(name)
  return redirect("/decks")
}
```

```astro
---
// AFTER: Astro page
if (Astro.request.method === "POST") {
  const formData = await Astro.request.formData()
  const name = formData.get("name") as string
  await createDeck(name)
  return Astro.redirect("/decks")
}
---
<form method="POST">
  <input name="name" required />
  <button type="submit">Create</button>
</form>
```

### Pattern 3: useFetcher → HTMX Partial

React's `useFetcher` with optimistic UI becomes an HTMX partial that returns updated component HTML.

```tsx
// BEFORE: React Router useFetcher with optimistic UI
function FavouriteButton({ taxonId, isFavourited }: Props) {
  const fetcher = useFetcher()
  const optimistic = fetcher.formData
    ? fetcher.formData.get("intent") === "favourite"
    : isFavourited

  return (
    <fetcher.Form method="post" action="/api/favourite">
      <input type="hidden" name="taxonId" value={taxonId} />
      <input type="hidden" name="intent" value={optimistic ? "unfavourite" : "favourite"} />
      <button type="submit">
        <HeartIcon filled={optimistic} />
      </button>
    </fetcher.Form>
  )
}
```

```astro
---
// AFTER: Astro component (no client JS)
interface Props { taxonId: number; isFavourited: boolean; size?: "sm" | "md" | "lg" }
const { taxonId, isFavourited, size = "md" } = Astro.props
---
<button
  hx-post="/partials/favourite-button"
  hx-vals={JSON.stringify({ taxonId, size })}
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
const taxonId = parseInt(formData.get("taxonId") as string)
const size = (formData.get("size") as string) || "md"
const result = await toggleFavourite(user.id, taxonId)
---
<FavouriteButton taxonId={taxonId} isFavourited={result.favourited} size={size} />
```

The partial renders the same Astro component — never duplicate component HTML in a partial page.

### Pattern 4: useState → Alpine.js x-data

Client-side UI state (open/close, tabs, selections) moves from React hooks to Alpine directives.

```tsx
// BEFORE: React useState
function Dropdown({ items }: Props) {
  const [open, setOpen] = useState(false)
  return (
    <div>
      <button onClick={() => setOpen(!open)}>Menu</button>
      {open && (
        <ul>{items.map(item => <li key={item.id}>{item.name}</li>)}</ul>
      )}
    </div>
  )
}
```

```html
<!-- AFTER: Alpine.js -->
<div x-data="{ open: false }">
  <button @click="open = !open">Menu</button>
  <ul x-show="open" @click.outside="open = false" x-cloak>
    <!-- server-rendered items -->
  </ul>
</div>
```

### Pattern 5: useSearchParams → HTMX hx-get + hx-include

URL-driven filters move from React Router's `useSearchParams` to HTMX attributes.

```tsx
// BEFORE: React Router
function Filters() {
  const [params, setParams] = useSearchParams()
  return (
    <select
      value={params.get("sort") ?? "name"}
      onChange={(e) => setParams(prev => { prev.set("sort", e.target.value); return prev })}
    >
      <option value="name">Name</option>
      <option value="recent">Recent</option>
    </select>
  )
}
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
  <option value="name">Name</option>
  <option value="recent">Recent</option>
</select>
```

### Pattern 6: React State Machine → Alpine State Machine

```tsx
// BEFORE: React
function StudySession({ cards }: Props) {
  const [index, setIndex] = useState(0)
  const [flipped, setFlipped] = useState(false)
  const card = cards[index]

  const handleRating = (rating: number) => {
    fetch("/api/study", { method: "POST", body: JSON.stringify({ cardId: card.id, rating }) })
    setFlipped(false)
    setIndex(i => i + 1)
  }

  return (
    <div>
      {!flipped ? (
        <div onClick={() => setFlipped(true)}>
          <img src={card.photo} />
          <p>What species?</p>
        </div>
      ) : (
        <div>
          <h2>{card.name}</h2>
          <button onClick={() => handleRating(1)}>Again</button>
          <button onClick={() => handleRating(3)}>Good</button>
        </div>
      )}
    </div>
  )
}
```

```html
<!-- AFTER: Alpine + HTMX -->
<div
  x-data="{ flipped: false }"
  @keydown.space.window.prevent="!flipped && (flipped = true)"
>
  <div x-show="!flipped" @click="flipped = true">
    <img src={card.photo} />
    <p>What species?</p>
  </div>
  <div x-show="flipped" x-cloak>
    <h2>{card.name}</h2>
    <button
      hx-post="/partials/study-review"
      hx-vals='{"cardId": "...", "rating": 1}'
      hx-target="#study-area"
      @click="flipped = false"
    >Again</button>
    <button
      hx-post="/partials/study-review"
      hx-vals='{"cardId": "...", "rating": 3}'
      hx-target="#study-area"
      @click="flipped = false"
    >Good</button>
  </div>
</div>
```

The HTMX POST returns the next card's HTML. Alpine handles the flip animation.

### Pattern 7: Infinite Scroll → HTMX revealed

```tsx
// BEFORE: React — IntersectionObserver + state + cleanup
const ref = useRef(null)
useEffect(() => {
  const observer = new IntersectionObserver(([e]) => { if (e.isIntersecting) loadMore() })
  observer.observe(ref.current)
  return () => observer.disconnect()
}, [])
return <div>{items.map(i => <Card {...i} />)}<div ref={ref} /></div>
```

```html
<!-- AFTER: HTMX revealed trigger on last item -->
<!-- Server renders the last card with a sentinel: -->
<div
  hx-get="/partials/feed?page=3&cursor=CURSOR"
  hx-trigger="revealed"
  hx-swap="afterend"
>
  <!-- card content -->
</div>
```

Server returns next batch of cards. Last card in each batch carries the sentinel for the next page. No client-side state, no cleanup.

### Pattern 8: React Context/Providers → Middleware + Astro.locals

```tsx
// BEFORE: React context for auth
function App() {
  return (
    <AuthProvider>
      <ThemeProvider>
        <Layout><Outlet /></Layout>
      </ThemeProvider>
    </AuthProvider>
  )
}
```

```typescript
// AFTER: Astro middleware
export const onRequest = defineMiddleware(async (context, next) => {
  context.locals.user = await resolveUser(context.request)
  return next()
})
```

Access in any page: `Astro.locals.user`. No provider trees, no hydration cost.

### Pattern 9: Multi-Region Updates → OOB Swaps

React mutations that update multiple UI regions (main content + sidebar count, list + header badge) use `hx-swap-oob="true"` to update extra elements in a single response.

```tsx
// BEFORE: React — action revalidates, multiple components re-render
export async function action({ request }: Route.ActionArgs) {
  const formData = await request.formData()
  const project = await createProject(formData.get("name") as string)
  throw redirect(`/project/${project.id}`)
}
// React Router revalidates all loaders; sidebar and main content both re-render
```

```astro
---
// AFTER: Partial returns main content + OOB sidebar update in one response
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

Use `HX-Push-Url` with OOB for common operations (one roundtrip). Use `HX-Redirect` only when simplicity matters (two roundtrips — client makes a new request to the redirect URL).

### Pattern 10: Loading States → .htmx-request Class

React's `fetcher.state` and `useNavigation().state` become CSS-driven via HTMX's `.htmx-request` class, added to the triggering element during requests.

```tsx
// BEFORE: React — check fetcher.state for pending UI
<button disabled={fetcher.state !== "idle"}>
  {fetcher.state !== "idle" ? <Spinner /> : "Save"}
</button>
```

```html
<!-- AFTER: HTMX — .htmx-request class + Tailwind group -->
<button class="group" hx-post="/partials/save" hx-disabled-elt="this">
  <span class="group-[.htmx-request]:hidden">Save</span>
  <img class="hidden group-[.htmx-request]:inline size-4" src="/spinner.svg" alt="" />
</button>
```

`hx-disabled-elt="this"` prevents double-submit. `group-[.htmx-request]:hidden` and `group-[.htmx-request]:inline` toggle visibility during the request.

## Migration Order

Port routes in this order (easiest to hardest):

1. **Static pages** — Landing, pricing, about. Copy HTML, add Astro frontmatter. No interactivity needed.
2. **Auth pages** — Signin, signup, reset password. Standard forms with POST handlers.
3. **List pages** — Profiles, decks, places. Server-rendered lists with URL pagination.
4. **Filter pages** — Explore species/decks. HTMX filters with `hx-include`.
5. **Detail pages** — Species detail, deck detail. Multiple sections, toggles, dialogs.
6. **Interactive pages** — Study session, deck builder. Alpine state machines.
7. **Feed pages** — For You, activity feed. HTMX infinite scroll.

## File Count Tradeoff

AHA apps have **more files** than React Router apps because:
- Every toggle button needs a partial page (`src/pages/partials/`)
- Components can't share action handlers — each needs its own endpoint
- No equivalent of `useFetcher` that handles multiple intents in one route action

The partials pattern is more explicit (easier to debug) but creates more files. Keep partials minimal: authenticate, call a server function, render the component.

## Server Module Reuse

Server-side modules (`*.server.ts`) port with minimal changes:
- Same database queries (Kysely, Supabase, etc.)
- Same validation logic (Valibot, Zod)
- Change `~/lib/` imports to `@/lib/` (Astro alias)
- Change `process.env` to `import.meta.env` (Astro/Vite requirement)
- Change `context.locals` to `Astro.locals` in page files

Most server files copy over with just import path changes.

## What NOT to Port

- **Optimistic UI** — Drop it. HTMX waits for the server response (100-150ms). Use `hx-disabled-elt="this"` to prevent double-submit.
- **Client-side caching** — HTMX re-fetches from the server on every interaction. The browser's HTTP cache handles the rest.
- **Error boundaries** — Replace with HTMX error events (`htmx:responseError`, `htmx:sendError`) in a global handler.
- **Suspense boundaries** — The server renders complete HTML. Use `loading="lazy"` for images.

## Common Gotchas

1. **`import.meta.env` not `process.env`** — Astro/Vite doesn't populate `process.env` from `.env` files.

2. **`hx-boost` is global** — Set on `<body>`, intercepts all links AND form submissions. Use `hx-boost="false"` on external links, file downloads, and forms that need full-page POST (e.g., file uploads).

3. **Fragment vs full page** — Check `HX-Target`, not just `HX-Request`. Boosted links omit `HX-Target`; filter swaps include it. A fragment returned to a boosted link replaces the entire page body.

4. **Alpine dies after HTMX swap** — `Alpine.start()` runs once. Add an `htmx:afterSettle` listener in the Layout that calls `Alpine.initTree(document.body)` to re-initialize Alpine directives in swapped DOM.

5. **`export const partial = true`** — Every file in `src/pages/partials/` needs this. Without it, Astro wraps the output in a full HTML document.

6. **Hidden images still load** — Chrome fetches `loading="lazy"` images inside `display: none` containers. Use `<picture>` with `<source media>` for responsive hiding, or Alpine `:src` for overlay images.

7. **`font-display: swap` not `block`** — `block` hides text until fonts load. `swap` shows fallback text immediately. On slow connections, this alone cuts LCP by seconds.

8. **Pass variant props through `hx-vals`** — If a component has size/style variants, include them in `hx-vals` so the partial returns the correct variant after a toggle.

## Performance Optimization Checklist

After porting, run through these optimizations:

1. Wrap images hidden on mobile (`hidden md:block`) in `<picture>` with `<source media>` — the browser only fetches sources whose media query matches
2. Use Alpine `:src` binding for lightbox/modal overlay images (`:src="open && '/path/to/image.webp'"`) — the image only loads when opened
3. Add `font-display: swap` to all `@font-face` declarations
4. Preload fonts with `<link rel="preload" as="font" crossorigin>` — starts download before CSS is parsed
5. Preconnect to external CDNs with `<link rel="preconnect">`
6. Add `width` and `height` to all `<img>` tags (prevents CLS)
7. Add `fetchpriority="high"` to the LCP image (hero/above-fold)
8. Use `hx-disabled-elt="this"` on all toggle buttons
