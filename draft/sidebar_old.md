---
title: "Sidebar Old"
summary: "An in-page navigation component for long-form pages."
---

# Sidebar

An in-page navigation component for long-form pages.

!!! info
    The Sidebar keeps navigation visible while scrolling and uses Scroll Spy to indicate the section currently in view. The component is reusable across pages as long as the required HTML relationship between navigation links and content sections is maintained.

## Use Cases

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

## Requirements

The Sidebar requires the following:

::: columns
::: column width="50%"
- A `<nav>` element with the `.sidebar` class
- Navigation links that point to section IDs
- Content sections with IDs matching the navigation links
:::
::: column width="50%"
- The reusable `initScrollSpy()` function available in `app.js`
- Standard HTML anchors connecting nav and content
- No additional JavaScript dependencies
:::

The navigation and content are connected through standard HTML anchors:

::: columns
::: column width="50%"
**Navigation Link**

```html title="Link"
<a href="#section-id">Section</a>
```

With page path:

```html title="Absolute Link"
<a href="/page#section-id">Section</a>
```
:::
::: column width="50%"
**Content Section**

```html title="Section"
<section id="section-id">
  ...
</section>
```

The fragment must match the target section ID.
:::

!!! tip
    Links may include the page path, as long as the fragment matches the target section ID.

## HTML

=== "Sidebar"

    ```html linenums="1" title="sidebar-nav.html"
    <nav class="ps-3 sticky-top top-header sidebar">
      <p class="text-secondary text-uppercase fs-8 mb-3">
        Enterprise Playbook
      </p>

      <ul class="list-unstyled d-grid gap-2 mb-0">
        <li>
          <a href="/playbook#introduction"
             class="d-block p-2 text-white fs-7">
            Introduction
          </a>
        </li>
        <li>
          <a href="/playbook#features"
             class="d-block p-2 text-white fs-7 active">
            Features
          </a>
        </li>
        <li>
          <a href="/playbook#pricing"
             class="d-block p-2 text-white fs-7">
            Pricing
          </a>
        </li>
      </ul>
    </nav>
    ```

=== "Sections"

    ```html linenums="1" title="sections.html"
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

!!! note
    The section names and number of items can vary between pages. Each navigation link must point to a section with a matching ID.

## Suggested Layout

The Sidebar can be used inside a two-column page layout:

```html linenums="1" title="layout.html" hl_lines="11-12"
<div class="enterprise-playbook">
  <section class="text-light py-5">
    <div class="container py-5 text-center">
      <h1 class="display-3 text-light">Page Title</h1>
    </div>
  </section>

  <div class="container py-5">
    <div class="row g-5">
      <aside class="col-12 col-lg-3">
        <!-- Sidebar -->
      </aside>

      <main class="col-12 col-lg-9">
        <section id="introduction">...</section>
        <section id="features">...</section>
        <section id="pricing">...</section>
      </main>
    </div>
  </div>
</div>
```

!!! tip
    The layout is only a suggested implementation. The Sidebar can be used with different page structures.

## Scroll Spy

The Sidebar uses the reusable `initScrollSpy()` function defined in:

```text title="File path"
modules_app/contentbox-custom/_themes/boxlang/resources/assets/js/app.js
```

Initialize it with:

```js title="Initialization"
initScrollSpy('.sidebar');
```

### How It Works

The function performs the following steps:

1. **Find the Sidebar** using its CSS selector
2. **Collect anchor links** (`a[href*="#"]`) inside the Sidebar
3. **Read the fragment** from each link's `href`
4. **Resolve sections** by querying `document.querySelector(targetId)`
5. **Observe** all resolved sections with `IntersectionObserver`
6. **Detect** which section enters the reading area
7. **Remove `.active`** from all navigation links
8. **Apply `.active`** to the link matching the visible section

The relationship flows as:

```text title="Data flow"
Sidebar link → href="#features" → <section id="features"> → IntersectionObserver → .active applied
```

### Implementation

```js title="initScrollSpy.js" linenums="1"
/**
 * Synchronizes sidebar navigation with the section currently in view.
 *
 * The sidebar links must point to sections using their IDs:
 * <a href="#section-id">Section</a>
 * <section id="section-id">...</section>
 *
 * @param {string} sidebarSelector - Selector for the navigation sidebar.
 */
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

### Initialization

```js title="initScrollSpy.js"
initScrollSpy('.sidebar');
```

!!! tip
    A new page does not need to modify `initScrollSpy()`. It only needs to provide a compatible Sidebar and matching section IDs.

## Reuse Example

```html linenums="1" title="minimal-sidebar.html"
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

```js title="Initialization"
initScrollSpy('.sidebar');
```

## Styling

The Sidebar reuses existing site classes for layout, spacing, typography, responsive behavior, and visual states.

Scroll Spy controls the `.active` class but does not define its visual appearance.

| State | Description |
| --- | --- |
| **Default** | No active indicator |
| **Hover** | Hover treatment without the active indicator |
| **Active** | Active treatment with the existing left border |

!!! warning
    Hover and active states should remain visually distinct to avoid confusing users.

## Responsive Behavior

The suggested layout uses Bootstrap's responsive grid:

```html title="Responsive aside"
<aside class="col-12 col-lg-3">
```

```html title="Responsive main"
<main class="col-12 col-lg-9">
```

The Sidebar can therefore use the site's existing responsive layout system.

## Design Decisions

??? info "Reusable behavior"
    The Scroll Spy implementation is independent from the Enterprise Playbook. It does not contain page-specific selectors or section IDs. The Sidebar selector is provided during initialization.

??? info "Anchor-based relationship"
    The navigation and content are connected through standard HTML anchors. This avoids maintaining a separate list of sections in JavaScript and keeps the relationship visible in the HTML.

??? info "Native browser API"
    The component uses `IntersectionObserver` instead of a continuous `scroll` event. No additional JavaScript dependency is required.

??? info "Existing design system"
    The component reuses existing site classes wherever possible. New styles should only be introduced when the existing design system does not provide the required behavior.

## Maintenance

### Adding a section

```html linenums="1" title="new-section.html"
<a href="#new-section">New Section</a>

<section id="new-section">
  ...
</section>
```

No JavaScript changes are required.

### Creating a new page with a Sidebar

::: stepper
::: step "Add the navigation element"
Add the `.sidebar` class to the navigation element.
:::
::: step "Add anchor links"
Create links using section anchors (`href="#section-id"`).
:::
::: step "Add matching sections"
Give each target section a matching `id` attribute.
:::
::: step "Initialize Scroll Spy"
Call `initScrollSpy('.sidebar')` to activate the component.
:::

## Component Contract

::: columns
::: column width="33%"
**Required**

- `.sidebar` navigation element
- Navigation links with anchor targets
- Sections with matching IDs
- `initScrollSpy()` from `app.js`
:::
::: column width="33%"
**Provides**

- Sticky navigation via layout classes
- Anchor-based navigation
- Active section detection
- Automatic `.active` state
:::
::: column width="34%"
**Dependencies**

- Native `IntersectionObserver`
- Existing site CSS/Sass
- No additional JS libraries
:::
