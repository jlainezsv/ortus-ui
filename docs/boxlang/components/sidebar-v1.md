---
title: "Sidebar V1"
summary: "An in-page navigation component for long-form pages."
---

An in-page navigation component that stays visible while scrolling and highlights the current section using Scroll Spy.

## Understand

### Overview

The Sidebar provides persistent navigation for pages with multiple sections. It uses `IntersectionObserver` to detect which section is in view and applies an `.active` state to the corresponding navigation link.

The component is reusable across pages. It requires only a `<nav>` element with the `.sidebar` class, navigation links pointing to section IDs, and content sections with matching IDs.

### Anatomy

The Sidebar consists of two parts connected through standard HTML anchors:

::: columns
::: column
::: card
**Navigation**

A `<nav>` element with the `.sidebar` class containing anchor links. Each link uses an `href` fragment that targets a section `id`.

```html
<a href="#section-id">Section</a>
```

Links may also include the page path:

```html
<a href="/page#section-id">Section</a>
```
:::
:::
::: column
::: card
**Content Sections**

Content sections with IDs that match the navigation link fragments.

```html
<section id="section-id">
  ...
</section>
```
:::
:::

!!! tip "Anchor relationship"
    The fragment in each navigation link must match the `id` of its target content section.

### States

::: cards
- title: Default
  description: No active indicator.

- title: Hover
  description: Hover treatment without the active indicator.

- title: Active
  description: Active treatment with the existing left border. Applied automatically by Scroll Spy.
:::

Hover and active should remain visually distinct.

## Use

### When to Use

Use the Sidebar when a page contains multiple sections that users may need to navigate directly:

- Enterprise Playbooks
- Documentation
- Long-form guides
- Technical pages
- Content-heavy pages

### Examples

Minimal sidebar with three sections:

```html title="Reusable Sidebar"
<nav class="sidebar" aria-label="Documentation sections">
  <a href="#getting-started">Getting Started</a>
  <a href="#configuration">Configuration</a>
  <a href="#deployment">Deployment</a>
</nav>

<section id="getting-started">
  ...
</section>

<section id="configuration">
  ...
</section>

<section id="deployment">
  ...
</section>
```

To add a section, add an anchor link and a matching section. No JavaScript changes are required.

## Implement

### Requirements

::: columns
::: column
::: card
**Navigation**
  - A `<nav>` element with the `.sidebar` class.
  - Navigation links with anchor targets.
:::
:::
::: column
::: card
**Content**
  - Sections with matching IDs.
  - `initScrollSpy()` available from `app.js`.
:::
:::

### Code

**HTML structure**

```html title="Sidebar"
<nav class="ps-3 sticky-top top-header sidebar">
  <p class="text-secondary text-uppercase fs-8 mb-3">
    Enterprise Playbook
  </p>

  <ul class="list-unstyled d-grid gap-2 mb-0">
    <li>
      <a
        href="/testing-sidebar#introduction"
        class="d-block p-2 text-white fs-7"
      >
        Introduction
      </a>
    </li>
    <li>
      <a
        href="/testing-sidebar#features"
        class="d-block p-2 text-white fs-7 active"
      >
        Features
      </a>
    </li>
    <li>
      <a
        href="/testing-sidebar#pricing"
        class="d-block p-2 text-white fs-7"
      >
        Pricing
      </a>
    </li>
  </ul>
</nav>
```

**Initialization**

```js
initScrollSpy('.sidebar');
```

A new page does not need to modify `initScrollSpy()`. It only needs to provide a compatible Sidebar and matching section IDs.

**Suggested layout**

The Sidebar can be used inside a two-column page layout:

```html title="Two-column layout"
<div class="enterprise-playbook">
  <div class="container py-5">
    <div class="row g-5">
      <aside class="col-12 col-lg-3">
        <!-- Sidebar -->
      </aside>

      <main class="col-12 col-lg-9">
        <section id="introduction">
          ...
        </section>

        <hr class="my-5">

        <section id="features">
          ...
        </section>

        <hr class="my-5">

        <section id="pricing">
          ...
        </section>
      </main>
    </div>
  </div>
</div>
```

!!! note "Layout note"
    The layout is only a suggested implementation. The Sidebar can be used with different page structures.

**Responsive behavior**

The suggested layout uses Bootstrap's responsive grid (`col-12 col-lg-3` for the Sidebar, `col-12 col-lg-9` for the main content). The Sidebar uses the site's existing responsive layout system.

### Accessibility

- Use a semantic `<nav>` element.
- Provide an accessible label (e.g. `aria-label="Documentation sections"`).
- Navigation links must reference valid section IDs.

### Technical Behavior

The `initScrollSpy()` function:

1. Finds the Sidebar using the provided selector.
2. Reads the `href` fragment from each anchor link.
3. Resolves the corresponding section by `id`.
4. Observes all resolved sections with `IntersectionObserver`.
5. When a section enters the reading area (defined by `rootMargin: '-20% 0px -70% 0px'`), removes `.active` from all sidebar links and applies `.active` to the matching link.

No page-specific section configuration is required.

### Contract

::: columns
::: column
::: card
**Requires**
  - `.sidebar` navigation element
  - Navigation links with anchor targets
  - Sections with matching IDs
  - `initScrollSpy()` from `app.js`
:::
:::
::: column
::: card
**Provides**
  - Sticky navigation via existing layout classes
  - Anchor navigation
  - Active section detection
  - Automatic `.active` state
:::
:::

**Dependencies**

- Native `IntersectionObserver`
- Existing site CSS/Sass
- No additional JavaScript libraries
