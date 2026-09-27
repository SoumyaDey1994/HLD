# Web Vitals & React UI Performance

## 1. What are Web Vitals & why are they important?

**Web Vitals** are a set of metrics defined by Google to measure the **real-world user experience and performance of web applications**.

They primarily measure three dimensions:

- **Loading** — How quickly users see meaningful content
- **Interactivity** — How quickly the UI responds to user actions
- **Visual stability** — Whether the UI behaves consistently without unexpected movement

### Why they matter

Good Web Vitals generally mean:

- Faster page loading
- More responsive UI
- Better user experience
- Lower user frustration
- Better performance on slower networks/devices
- Objective metrics for identifying UI performance regressions

For an enterprise React application, Web Vitals should be treated as **engineering health metrics**, not just SEO metrics.

---

## 2. Top 5 Web Vitals & what they imply

| Metric | What it measures | Good target | What it tells us |
|---|---|---:|---|
| **FCP** | Time until first content appears | ≤ **1.8s** | How quickly the user sees something |
| **LCP** | Time until the main/largest content appears | ≤ **2.5s** | How quickly the page becomes useful |
| **INP** | Responsiveness to user interactions | ≤ **200ms** | Whether the UI feels responsive |
| **CLS** | Unexpected layout movement | ≤ **0.1** | Whether the UI remains visually stable |
| **TTFB** | Time until the server starts responding | ≤ **800ms** | Backend/network responsiveness |

> **Note:** FCP and TTFB are not currently Core Web Vitals, but they are highly useful supporting performance metrics.

### Simple mental model

```text
              USER EXPERIENCE
                    │
        ┌───────────┼───────────┐
        │           │           │
      Loading   Interaction  Stability
        │           │           │
    FCP → LCP      INP          CLS
        │
       TTFB
```

---

## 3. Special mention: TTI

### TTI — Time to Interactive

TTI measures approximately **when the page becomes reliably interactive**.

Think of the difference like this:

```text
Page Loading
    │
    ├── FCP ─────► Something is visible
    │
    ├── LCP ─────► Main content is visible
    │
    ├── TTI ─────► Application is ready to interact
    │
    └── INP ─────► How quickly it responds to interaction
```

### Why TTI is important for React applications

Large React SPAs can display UI before the application is actually ready to respond because they may still be:

- Downloading JavaScript
- Parsing/executing JavaScript
- Hydrating components
- Initializing state
- Processing API responses
- Rendering large component trees

Therefore, **TTI is particularly useful as an application-readiness indicator**, even though it is no longer a Core Web Vital.

For an enterprise SPA, you can also define a **custom "App Ready" metric**:

> Route loaded + critical APIs completed + primary components rendered + application ready for user interaction.

This can be more meaningful than generic TTI for your specific application.

---

## 4. How React can improve each Web Vital

### FCP — First Contentful Paint

**Goal:** Show something meaningful to the user quickly.

React strategies:

1. **Reduce initial JavaScript bundle**
   - Use tree shaking.
   - Remove unnecessary dependencies.
   - Avoid importing large libraries unnecessarily.

2. **Use route-level code splitting**
   ```js
   const Dashboard = lazy(() => import('./Dashboard'));
   ```

3. **Load critical resources first**
   - Fonts
   - CSS
   - Above-the-fold assets

4. **Avoid expensive work during initial render**
   - Don't perform heavy calculations during component initialization.

5. **Use SSR/streaming where appropriate**
   - Particularly useful when initial content can be rendered server-side.

---

### LCP — Largest Contentful Paint

**Goal:** Get the primary page content visible quickly.

React strategies:

1. **Prioritize the primary UI**
   - Render critical content first.
   - Lazy-load secondary components.

2. **Optimize API calls**
   - Parallelize independent requests.
   - Avoid unnecessary sequential API calls.

3. **Cache API data**
   - Use React Query/TanStack Query or equivalent caching mechanisms.

4. **Optimize large assets**
   - **Images:** WebP / AVIF; responsive sizing and compression
   - **Fonts:** WOFF2; subset fonts and load only required weights
   - **Icons:** SVG; use icon sprites or optimized SVGs where appropriate
   - **Charts:** Prefer lightweight SVG/canvas rendering and lazy-load heavy charting libraries

5. **Avoid blocking the main thread**
   - Defer non-critical JavaScript and expensive computations.

---

### INP — Interaction to Next Paint

**Goal:** Make clicks, typing, filtering, navigation, etc. feel instantaneous.

React strategies:

1. **Minimize unnecessary re-renders**
   - Keep state close to where it is used.
   - Avoid unnecessarily updating large component trees.

2. **Use memoization intelligently**
   ```js
   React.memo()
   useMemo()
   useCallback()
   ```

3. **Defer non-urgent updates**
   ```js
   startTransition(() => {
     setFilteredData(data);
   });
   ```

4. **Virtualize large lists/tables**
   - Don't render thousands of DOM nodes simultaneously.

5. **Move expensive computation away from the main thread**
   - Web Workers
   - Background processing
   - Debouncing/throttling where appropriate

---

### CLS — Cumulative Layout Shift

**Goal:** Keep the UI visually stable.

React strategies:

1. **Reserve space for dynamic content**
   - Images
   - Tables
   - Charts
   - Async components

2. **Use stable loading states**
   - Skeletons should have approximately the same dimensions as the final content.

3. **Don't inject content above existing content**
   - Avoid suddenly inserting banners/alerts that push the page down.

4. **Define image/media dimensions**
   ```html
   <img width="800" height="400" />
   ```

5. **Avoid late-loading UI changes**
   - Especially fonts, notifications, and dynamically loaded components.

---

### TTFB — Time to First Byte

**Goal:** Reduce the time before the application starts receiving a response.

This is less about React itself and more about the **architecture surrounding React**.

React/application strategies:

1. **Reduce unnecessary API calls**
   - Don't fetch data that isn't needed for initial rendering.

2. **Parallelize API requests**
   ```text
   API A ───────┐
   API B ───────┼──► Render
   API C ───────┘
   ```

3. **Use caching**
   - Browser
   - CDN
   - API/data layer

4. **Optimize backend APIs**
   - Database queries
   - Payload size
   - Serialization
   - Service-to-service calls

5. **Use CDN/edge infrastructure where applicable**
   - Reduce network distance between users and application resources.

---

## 5. How to monitor Web UI performance

A good performance strategy should combine **development-time diagnostics, automated audits, and production monitoring**.

### A. Lighthouse

Use **Lighthouse** for automated performance audits and benchmarking.

It provides useful measurements and diagnostics around:

- FCP
- LCP
- CLS
- TBT
- Performance score
- Resource loading
- JavaScript/CSS optimization opportunities

It can be used during:

- Local development
- PR validation
- CI/CD
- Release validation

---

### B. Chrome DevTools

Use **Chrome DevTools** for deeper investigation when a metric is poor.

#### Performance

Helps identify:

- Long-running JavaScript
- Excessive rendering
- Layout/reflow
- Paint operations
- Main-thread blocking
- Rendering bottlenecks

#### Network

Helps analyze:

- API latency
- TTFB
- Request waterfalls
- Payload sizes
- Resource loading
- Caching behavior

#### React DevTools Profiler

Useful for identifying:

- Expensive React components
- Unnecessary re-renders
- Slow component updates
- Rendering bottlenecks

---

### Notable mentions

**Real User Monitoring (RUM)**

Useful for measuring Web Vitals from **actual production users**, across different browsers, devices, networks and geographic locations.

**Performance Budgets**

Define acceptable thresholds for metrics such as **LCP, INP, bundle size, API latency and App Ready/TTI**, and use them as regression checks in CI/CD.

### Recommended approach

```text
Development
    │
    └── Chrome DevTools + React Profiler
                    │
                    ▼
              Optimization
                    │
                    ▼
CI/CD ───────► Lighthouse + Performance Budgets
                    │
                    ▼
Production ─────► RUM
```

This gives you a good combination of **diagnosis → validation → production monitoring** without making the performance-monitoring process unnecessarily complex.

---

## Key takeaway

Don't treat Web Vitals as isolated numbers.

For a React application, the optimization chain is:

**Reduce initial JS → optimize rendering → minimize re-renders → optimize API calls → reduce main-thread work → continuously measure real users.**

That combination helps keep **FCP, LCP, INP, CLS, TTFB and TTI/App Ready** healthy over time.
