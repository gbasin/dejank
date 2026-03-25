# Static Pattern Catalog

Scan for ALL of the following. Each pattern includes what to look for in the source and why it causes jank.

---

### 1. Effect-Driven State Initialization (Critical)

**What to find**: `useEffect` or `useEffect(..., [])` whose body calls a state setter with a value different from the initial `useState` default.

```tsx
// BAD: renders once with `false`, then immediately re-renders with `true`
const [isDark, setIsDark] = useState(false);
useEffect(() => {
  setIsDark(window.matchMedia('(prefers-color-scheme: dark)').matches);
}, []);
```

**What the user sees**: 1-2 frames of light theme before snapping to dark. A "flash."

**Fixes** (in order of preference):
1. Compute the correct initial value inline: `useState(() => window.matchMedia(...).matches)`
2. If it *must* be an effect (e.g., reading a ref), use `useLayoutEffect` -- it fires before paint
3. For external stores, use `useSyncExternalStore` which avoids the double-render entirely

**Special case**: When the initial state is read from `localStorage`, `sessionStorage`, `document.cookie`, or similar browser APIs -- same pattern, same fix (lazy initializer).

**Verify**: The `useState` initializer computes the correct value directly. The `useEffect` that called the setter is removed or converted to `useLayoutEffect` if a ref read is required.

---

### 2. Derived State in Effects (High)

**What to find**: `useEffect` that watches prop/state and sets another piece of state that could be computed during render.

```tsx
// BAD: every time `items` changes, renders twice -- once stale, once correct
const [filtered, setFiltered] = useState([]);
useEffect(() => {
  setFiltered(items.filter(i => i.active));
}, [items]);
```

**What the user sees**: A single frame of stale data before the filtered list updates.

**Fix**: Compute during render -- `const filtered = useMemo(() => items.filter(...), [items])` -- or just inline it if cheap.

**Verify**: The `useEffect` + `setState` pair is gone, replaced by inline computation or `useMemo`. The derived value updates in the same render as its source.

---

### 3. Chained Effect Cascade (High)

**What to find**: Two or more `useEffect` hooks where one sets state that triggers another. The chain `effect A -> setState -> re-render -> effect B -> setState -> re-render` produces multiple intermediate frames.

**What the user sees**: Cascading visual updates -- elements shifting, appearing, or changing in sequence over 2-4 frames instead of all at once.

**Fix**: Collapse into a single state update. Use `useReducer` if the logic is complex, or compute derived values during render. If effects are truly independent and order doesn't matter, they can stay separate -- but if B depends on A's state, they should be one unit.

**Verify**: The chain of effects is collapsed. A single state change produces one render cycle, not multiple sequential re-renders.

---

### 4. Conditional Mount/Unmount (High)

**What to find**: JSX that toggles component *existence* based on state/props.

```tsx
// BAD: component unmounts and remounts -- animation restarts, state resets
{isOpen && <Panel />}
{showSidebar ? <Sidebar /> : null}
```

**When this is jank**: When the condition toggles rapidly or during transitions. The unmount/remount cycle means:
- CSS enter animations replay
- Component internal state resets
- A frame of "nothing" may be painted between unmount and remount

**When this is fine**: Lazy rendering of expensive components that aren't needed. Modal open/close where you *want* state reset.

**Fixes** (in order of preference):
1. **React 19.2+**: Use `<Activity mode={isOpen ? "visible" : "hidden"}>` -- preserves state, hides DOM, destroys effects, deprioritizes hidden updates. The first-party solution for coarse-grained show/hide.
2. Use CSS to hide instead of unmounting:
```tsx
// STABLE: component stays mounted, no remount flash
<Panel className={isOpen ? 'visible' : 'hidden'} />
<Panel style={{ display: isOpen ? 'block' : 'none' }} />
<Panel hidden={!isOpen} />
```

**Judgment call**: Flag this as a finding but note that not every conditional render is a bug -- flag specifically when the condition is tied to state that changes during user interaction (hover, focus, animation, toggle).

**Verify**: The component stays mounted across the toggle. Internal state persists and CSS enter animations do not replay.

---

### 5. Unstable Key Props (High)

**What to find**: `key` prop set to a value that changes when it shouldn't -- `Math.random()`, `Date.now()`, `uuid()` called inline, or array index on a list that reorders.

```tsx
// BAD: new key every render = full unmount/remount every frame
<Panel key={Math.random()} />
<Item key={`${item.id}-${Date.now()}`} />
```

**What the user sees**: Component flashes/blinks on every re-render. Animations restart. Input fields lose focus.

**Fix**: Use a stable identifier. If using index, ensure the list never reorders or filters.

**Verify**: The `key` prop uses a stable identifier. Grep for the old unstable expression to confirm it is removed.

---

### 6. Unsized Async Content -- Layout Shift (Critical)

**What to find**: `<img>`, `<video>`, `<iframe>`, or any container that loads async content without explicit dimensions.

```tsx
// BAD: content below the image jumps when it loads
<img src={url} />
<img src={url} className="w-full" /> // width set, but no height/aspect-ratio

// STABLE
<img src={url} width={640} height={480} />
<img src={url} className="w-full aspect-video" />
<div className="aspect-[4/3]"><img src={url} className="object-cover" /></div>
```

**What the user sees**: Content below the element jumps/shifts when the async content arrives.

**Fix**: Set explicit `width`+`height` attributes, or use CSS `aspect-ratio`. For dynamic content, use a skeleton placeholder with the correct dimensions.

**Verify**: The element has explicit `width`+`height` attributes or CSS `aspect-ratio`. Content below does not shift when the async content loads.

---

### 7. Skeleton/Placeholder Absence (Medium)

**What to find**: Components that render loading states as *structurally different* from their loaded state -- different height, different layout, or just a spinner.

```tsx
// BAD: loading state is 40px tall, loaded state is 400px tall
if (loading) return <Spinner />;
return <DataTable rows={data} />;
```

**What the user sees**: Layout jumps when data arrives because the loading placeholder was a different size.

**Fix**: Use a skeleton that matches the loaded component's dimensions. Or use `min-height` on the container.

**Verify**: Loading and loaded states have matching outer dimensions. The layout does not jump when data arrives.

---

### 8. Suspense Boundary Flash (Medium)

**What to find**: `<Suspense>` with a `fallback` that can resolve within a few frames (~100ms), causing the fallback to flash briefly before content appears.

**Also find**: `React.lazy()` imports without any Suspense boundary (these will throw, but before that they cause a blank flash).

**What the user sees**: A loading spinner/skeleton blinks for a split second.

**Fix**: Wrap the state update that triggers the Suspense in `startTransition` -- React will keep showing the old UI while the new content loads, avoiding the flash. For short fetches, set a minimum delay on the loading state display.

**Verify**: Navigate to the Suspense-wrapped content. The previous UI stays visible during short loads instead of flashing a fallback.

---

### 9. Hydration Mismatch Sources (Critical for SSR/SSG)

**What to find**: Client-only values used in JSX during initial render:

- `window.*`, `document.*`, `navigator.*`
- `localStorage.*`, `sessionStorage.*`
- `Date.now()`, `new Date()` (if value differs server/client)
- `Math.random()`
- `typeof window !== 'undefined'` used to conditionally render different JSX

```tsx
// BAD: server renders null, client renders the component -> flash
{typeof window !== 'undefined' && <ClientOnlyWidget />}
```

**What the user sees**: Content briefly appears as the server-rendered version, then flickers as React hydration replaces it with the client version.

**Fix**: Use a hydration-safe pattern -- `useEffect` to set a `mounted` flag, then conditionally render. Or use the framework's client-only wrapper (`next/dynamic` with `ssr: false`, Remix `ClientOnly`).

**SSR detection**: Check for `next.config.*`, `remix.config.*`, `astro.config.*`, or exports like `getServerSideProps`, `generateStaticParams`, `loader`, `getStaticProps` in route files. Also check for `hydrateRoot` or `renderToPipeableStream` in entry files. If none found, skip this pattern -- the project is a pure client-side SPA.

**Verify**: Server and client render identical initial HTML. No React hydration warnings appear in the browser console. The client-only content uses a hydration-safe pattern (`useEffect` + mounted flag, or framework client-only wrapper).

---

### 10. Layout Thrashing in Event Handlers (Medium)

**What to find**: Reading layout properties (`offsetHeight`, `offsetWidth`, `getBoundingClientRect()`, `scrollTop`, `clientHeight`, `getComputedStyle()`) then immediately writing to the DOM or setting state in the same synchronous block.

```tsx
// BAD: read -> write -> read -> write forces multiple reflows
function handleResize() {
  const h = element.offsetHeight;    // read (forces layout)
  element.style.height = h * 2;     // write (invalidates layout)
  const w = element.offsetWidth;     // read (forces layout AGAIN)
  element.style.width = w * 2;      // write
}
```

**What the user sees**: Sluggish animations, dropped frames during scroll/resize.

**Fix**: Batch all reads, then batch all writes. Or use `requestAnimationFrame`. Or better -- use CSS for the layout logic so the browser handles it.

**Verify**: DOM reads and writes are batched separately. No forced reflow warnings appear in the Chrome Performance panel during the interaction.

---

### 11. Animating Layout Properties (Medium)

**What to find**: CSS transitions or animations on `width`, `height`, `top`, `left`, `right`, `bottom`, `margin`, `padding`, `border-width`, `font-size`. In Tailwind: `transition-all` is a common culprit.

```css
/* BAD: triggers layout recalculation every frame */
.panel { transition: height 0.3s, width 0.3s; }

/* FAST: only composite properties -- GPU-accelerated */
.panel { transition: transform 0.3s, opacity 0.3s; }
```

**What the user sees**: Janky/stuttery animations, especially on lower-powered devices.

**Fix**: Animate only `transform` and `opacity`. Use `scale()` instead of width/height changes, `translate()` instead of top/left. If you must animate layout properties, add `will-change` and accept the trade-off.

**Tailwind-specific**: Flag `transition-all` -- it transitions every property including layout ones. Use `transition-transform`, `transition-opacity`, or `transition-colors` instead.

**Verify**: Animations target only `transform` and `opacity`. No layout recalculations per frame in the Performance panel. In Tailwind, `transition-all` is replaced with a specific transition utility.

---

### 12. Missing `startTransition` for Expensive Updates (Low)

**What to find**: State updates that trigger expensive re-renders (large lists, heavy computation) without being wrapped in `startTransition`.

**What the user sees**: UI freezes momentarily -- input feels laggy, buttons feel unresponsive.

**Fix**: Wrap non-urgent updates in `startTransition` to let React yield to the browser for paint between renders.

**Verify**: The UI remains responsive during the update. Typing or clicking does not feel laggy while the expensive re-render is in progress.

---

### 13. Ref Callback Remount (Medium -- reduced with React Compiler)

**What to find**: Inline ref callbacks that create a new function identity every render.

```tsx
// BAD: new function every render -> ref detaches and reattaches
<div ref={(node) => { /* measure or focus */ }} />
```

**What the user sees**: Focus loss, measurement jitter, or elements re-initializing.

**Fix**: Memoize the ref callback with `useCallback`, or use a ref object (`useRef`).

**React 19 notes**:
- `forwardRef` is deprecated -- refs are now passed as regular props.
- React Compiler auto-memoizes inline ref callbacks, reducing relevance in compiled projects. Check for the DevTools sparkle badge -- if the component is compiled, this pattern is likely handled.
- Ref callbacks can now return a cleanup function (like `useEffect`). React calls the cleanup on unmount instead of calling the ref with `null`.

**Verify**: The ref callback has stable identity (memoized with `useCallback` or using `useRef`). Focus and measurements are not disrupted on re-render.

---

### 14. Z-Index / Stacking Context Flash (Low)

**What to find**: Elements that create a new stacking context conditionally (`position`, `z-index`, `transform`, `opacity < 1`, `will-change` toggled via state), causing content to visually "pop" in front of or behind other content for a frame.

**What the user sees**: An element briefly appears above/below where it should be.

**Fix**: Keep the stacking context stable -- apply `position: relative` and `z-index` unconditionally if the element participates in stacking.

**Verify**: Stacking context properties (`position`, `z-index`) are applied unconditionally. The element does not visually pop between layers during state changes.

---

### 15. Font Flash -- FOUT/FOIT (High)

**What to find**: Custom font loading without preloading or `font-display` control. In Next.js: using `<link href="fonts.googleapis.com/...">` instead of `next/font`.

**What the user sees**: Text renders in a fallback font then swaps to the custom font (FOUT -- Flash of Unstyled Text), or text is invisible until the font loads (FOIT -- Flash of Invisible Text). Both cause layout shift when glyph metrics differ.

**Fixes**:
1. Use `font-display: optional` -- if the font isn't cached, skip it entirely (zero flash)
2. Preload critical fonts: `<link rel="preload" href="font.woff2" as="font" crossorigin>`
3. Use font metric overrides (`size-adjust`, `ascent-override`, `descent-override`) to match fallback glyph metrics
4. In Next.js: use `next/font` which handles preloading and metric overrides automatically

**Verify**: Hard-refresh the page. Text renders in the final font immediately (or the fallback is visually identical via metric overrides). No visible font swap or layout shift from glyph metric differences.

---

### 16. Unstable Context Value (High)

**What to find**: A `Context.Provider` whose `value` prop is a new object/array literal on every render.

```tsx
// BAD: new object every render -> all consumers re-render every time parent renders
<ThemeContext.Provider value={{ theme, toggleTheme }}>
```

**What the user sees**: Unrelated parts of the UI re-render and potentially flicker when a parent component updates, even if the context value hasn't meaningfully changed.

**Fix**: Memoize the value:
```tsx
const value = useMemo(() => ({ theme, toggleTheme }), [theme, toggleTheme]);
<ThemeContext.Provider value={value}>
```

**Note**: React Compiler auto-memoization mitigates this but does not eliminate all cases. `@eslint-react/no-unstable-context-value` catches it statically.

**Verify**: The `value` prop is memoized. Context consumer components do not re-render when the provider's parent updates but the context value has not changed.

---

### 17. Component Defined Inside Component (Critical)

**What to find**: A function component defined inside another component's render body.

```tsx
function Parent() {
  // BAD: ChildPanel is a new component type every render -> full remount every time
  function ChildPanel() {
    return <div>{/* ... */}</div>;
  }
  return <ChildPanel />;
}
```

**What the user sees**: The inner component fully remounts on every parent render -- all state is destroyed, DOM is rebuilt, animations restart. Visually identical to an unstable key.

**Fix**: Move the component definition outside the parent function. If it needs parent scope, pass values as props.

**Verify**: The inner component is defined outside the parent function. It retains state across parent re-renders and does not remount.

---

### 18. Async Waterfall in Component Tree (High)

**What to find**: Sequential data fetching in nested components where a child cannot start fetching until its parent renders with data.

```tsx
// BAD: waterfall -- parent fetches, renders child, child fetches, renders grandchild
function Dashboard() {
  const { data: user } = useQuery('user');
  if (!user) return <Spinner />;
  return <UserProjects userId={user.id} />;  // fetches after parent resolves
}
```

**What the user sees**: Cascading loading states -- each level adds a full round-trip of latency before the next level can even start. Content pops in section by section.

**Fixes**:
1. Hoist fetches to the top level and fetch in parallel
2. Use React Server Components with parallel data loading
3. Use framework-level data loading (`loader` in Remix/React Router, `generateMetadata`/`fetch` in Next.js Server Components)
4. Use `<Suspense>` with parallel `use()` or `useSuspenseQuery` calls

**Verify**: Open the browser Network tab and trigger the data loading. Fetches that were previously sequential now start in parallel (overlapping in the waterfall view).
