---
title: "Sidebar"
summary: "An in-page navigation component for long-form pages."
---

The Sidebar keeps navigation visible while scrolling and uses Scroll Spy to indicate the section currently in view. The component is reusable across pages as long as the required HTML relationship between navigation links and content sections is maintained.

## Use Cases

Use the Sidebar when a page contains multiple sections that users may need to navigate directly.

::: cards
::: card title="Enterprise Playbooks" icon="phosphor-duotone:book-open"
Multi-section reference documents with deep hierarchies.
:::
::: card title="Documentation" icon="phosphor-duotone:books"
Technical docs with many subsections to navigate.
:::
::: card title="Long-form Guides" icon="phosphor-duotone:compass"
Step-by-step guides with distinct sections.
:::
::: card title="Technical Pages" icon="phosphor-duotone:code"
Pages with code-heavy content organized by topic.
:::
::: card title="Content-heavy Pages" icon="phosphor-duotone:file-text"
Any page where direct section access improves usability.
:::
:::

## Requirements

The Sidebar requires:
::: columns
::: column
::: card
Navigation
  - A `<nav>` element with the `.sidebar` class.
  - Navigation links that point to section IDs.
:::
:::
::: column
::: card
Content
  - Content sections with IDs matching the navigation links.
  - The reusable `initScrollSpy()` function available in `app.js`.
:::
:::
:::



The navigation and content are connected through standard HTML anchors.
::: columns
::: column 
### Navigation Link
  ```html
  <a href="#section-id">Section</a>
  
  Links may also include the page path:

  <a href="/page#section-id">Section</a>
  ```
:::
::: column
### Content Section
  ```html
  <section id="section-id">
    ...
  </section>
  ```

  The fragment must match the target section ID.
:::
:::

!!! tip "Anchor relationship"
    The fragment in each navigation link must match the `id` of its target content section.

## HTML

### Sidebar


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

### Sections

Each navigation link must point to a section with a matching ID:

```html title="Sections"
<section id="introduction">
  ...
</section>

<section id="features">
  ...
</section>

<section id="pricing">
  ...
</section>
```

## Suggested Layout

The Sidebar can be used inside a two-column page layout.

```html title="Two-column layout"
<div class="enterprise-playbook">
  <section id="enterprise-playbook-hero" class="text-light py-5">
    <div class="container py-5 text-center">
      <p class="badge badge-default text-uppercase rounded-pill px-3 py-2 mb-4">
        &#8226; BoxLang · Enterprise Playbook
      </p>

      <h1 class="display-3 text-light poppins-bold-text">
        Test
        <span class="text-gradient display-3 poppins-bold-text">
          Sidebar Component
        </span>
      </h1>
    </div>
  </section>

  <div class="container py-5">
    <div class="row g-5">
      <aside class="col-12 col-lg-3">
        <!-- Sidebar -->
      </aside>

      <main class="col-12 col-lg-9">
        <section id="introduction">
          <div style="height:600px">Introduction</div>
        </section>

        <hr class="my-5">

        <section id="features">
          <div style="height:600px">Features</div>
        </section>

        <hr class="my-5">

        <section id="pricing">
          <div style="height:600px">Pricing</div>
        </section>
      </main>
    </div>
  </div>
</div>
```

!!! note "Implementation note"
    The layout is only a suggested implementation. The Sidebar can be used with different page structures.

## Scroll Spy

The Sidebar uses the reusable `initScrollSpy()` function defined in:

```text
modules_app/contentbox-custom/_themes/boxlang/resources/assets/js/app.js
```

Initialize it with:

```js
initScrollSpy('.sidebar');
```

No page-specific section configuration is required.

### How It Works

::: stepper
1. **Find the Sidebar**

   Finds the Sidebar using its selector.

2. **Read navigation targets**

   Finds the anchor links inside the Sidebar.

3. **Resolve section IDs**

   Reads the fragment from each link and finds the corresponding section by ID.

4. **Observe sections**

   Observes those sections with `IntersectionObserver`.

5. **Detect the active section**

   Detects which section enters the reading area.

6. **Update navigation state**

   Removes `.active` from the navigation links and applies `.active` to the corresponding link.
:::

The relationship is:

```text
Sidebar link
    ↓
href="#features"
    ↓
<section id="features">
    ↓
IntersectionObserver
    ↓
Sidebar link receives .active
```

### Implementation

```js title="app.js"
function initScrollSpy(sidebarSelector) {
  const sidebar = document.querySelector(sidebarSelector);

  if (!sidebar) {
    return;
  }

  const navLinks = sidebar.querySelectorAll('a[href*="#"]');

  // Get the sections referenced by the sidebar links.
  const sections = Array.from(navLinks)
    .map((link) => {
      const targetId = link.hash;

      return targetId ? document.querySelector(targetId) : null;
    })
    .filter(Boolean);

  const observer = new IntersectionObserver(
    (entries) => {
      entries.forEach((entry) => {
        if (!entry.isIntersecting) {
          return;
        }

        const activeSectionId = entry.target.id;

        // Remove the active state from all navigation links.
        navLinks.forEach((link) => {
          link.classList.remove('active');
        });

        // Find the link associated with the visible section.
        const activeLink = Array.from(navLinks).find(
          (link) => link.hash === `#${activeSectionId}`
        );

        // Apply the existing active state.
        activeLink?.classList.add('active');
      });
    },
    {
      // Define the upper reading area where a section becomes active.
      rootMargin: '-20% 0px -70% 0px',
      threshold: 0
    }
  );

  // Start observing all sections referenced by the sidebar.
  sections.forEach((section) => {
    observer.observe(section);
  });
}
```

## Initialization

```js title="Initialization"
initScrollSpy('.sidebar');
```

A new page does not need to modify `initScrollSpy()`. It only needs to provide a compatible Sidebar and matching section IDs.

## Reuse Example

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

Use the same initialization:

```js
initScrollSpy('.sidebar');
```

## Styling

The Sidebar reuses existing site classes for layout, spacing, typography, responsive behavior, and visual states.

!!! info "Active state"
    Scroll Spy controls the `.active` class but does not define its visual appearance.

Interaction states:

::: cards
- title: Default
  description: No active indicator.

- title: Hover
  description: Hover treatment without the active indicator.

- title: Active
  description: Active treatment with the existing left border.
:::

Hover and active should remain visually distinct.

## Responsive Behavior

The suggested layout uses Bootstrap's responsive grid:

```html
<aside class="col-12 col-lg-3">
```

and:

```html
<main class="col-12 col-lg-9">
```

The Sidebar can therefore use the site's existing responsive layout system.

## Design Decisions

### Reusable behavior

The Scroll Spy implementation is independent from the Enterprise Playbook.

It does not contain page-specific selectors or section IDs.

The Sidebar selector is provided during initialization.

### Anchor-based relationship

The navigation and content are connected through standard HTML anchors.

This avoids maintaining a separate list of sections in JavaScript and keeps the relationship visible in the HTML.

### Native browser API

The component uses `IntersectionObserver` instead of a continuous `scroll` event.

No additional JavaScript dependency is required.

### Existing design system

The component reuses existing site classes wherever possible.

New styles should only be introduced when the existing design system does not provide the required behavior.

## Maintenance

When adding a section:

```html title="New section"
<a href="#new-section">New Section</a>

<section id="new-section">
  ...
</section>
```

No JavaScript changes are required.

### New page checklist

::: stepper
1. **Add the Sidebar class**

   Add the `.sidebar` class to the navigation.

2. **Add section anchors**

   Add links using section anchors.

3. **Match target IDs**

   Give each target section a matching `id`.

4. **Initialize Scroll Spy**

   Initialize Scroll Spy with:

   ```js
   initScrollSpy('.sidebar');
   ```
:::

## Component Contract

### Required

::: cards
- title: Sidebar navigation
  description: `.sidebar` navigation element.

- title: Navigation targets
  description: Navigation links with anchor targets.

- title: Matching sections
  description: Sections with matching IDs.

- title: Scroll Spy
  description: `initScrollSpy()` available from `app.js`.
:::

### Provides

- Sticky navigation via existing layout classes.
- Anchor navigation.
- Active section detection.
- Automatic `.active` state.

### Dependencies

- Native `IntersectionObserver`.
- Existing site CSS/Sass.
- No additional JavaScript libraries.
