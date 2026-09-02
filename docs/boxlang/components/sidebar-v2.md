---
title: "Sidebar V2"
summary: "In-page navigation for long-form pages with automatic scroll tracking."
---

=== "Understand"

    ### Overview

    The Sidebar is an in-page navigation component for long-form pages. It stays visible while scrolling and automatically highlights the current section using Scroll Spy.

    It is reusable across pages as long as the navigation links and content sections share matching anchor IDs.

    ### Anatomy

    The Sidebar has two parts connected through standard HTML anchors:

    **Navigation** — A `<nav>` element with the `.sidebar` class containing anchor links. Each link targets a section by `id`.

    **Content sections** — Sections with IDs that match the navigation link fragments.

    ```html
    <a href="#section-id">Section</a>

    <section id="section-id">
      ...
    </section>
    ```

    Links may also include the page path. The fragment must match the target section ID.

    !!! tip "Anchor relationship"
        The `href` fragment in each navigation link must match the `id` of its target content section.

    ### States

    - **Default** — No active indicator.
    - **Hover** — Hover treatment without the active indicator.
    - **Active** — Active treatment with the existing left border. Applied automatically by Scroll Spy.

    Hover and active should remain visually distinct.

=== "Use"

    ### When to use

    Use the Sidebar when a page contains multiple sections that users may need to navigate directly:

    - Enterprise Playbooks
    - Documentation
    - Long-form guides
    - Technical pages
    - Content-heavy pages

    ### Examples

    Minimal sidebar with three sections:

    ```html
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

=== "Implement"

    ### Requirements

    - A `<nav>` element with the `.sidebar` class
    - Navigation links with anchor targets
    - Content sections with matching IDs
    - `initScrollSpy()` available from `app.js`

    ### Code

    **Full sidebar markup**

    ```html
    <nav class="ps-3 sticky-top top-header sidebar">
      <p class="text-secondary text-uppercase fs-8 mb-3">Enterprise Playbook</p>

      <ul class="list-unstyled d-grid gap-2 mb-0">
        <li>
          <a href="/page#introduction" class="d-block p-2 text-white fs-7">
            Introduction
          </a>
        </li>
        <li>
          <a href="/page#features" class="d-block p-2 text-white fs-7 active">
            Features
          </a>
        </li>
        <li>
          <a href="/page#pricing" class="d-block p-2 text-white fs-7">
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

    No page-specific configuration is required. A new page only needs a compatible Sidebar markup and matching section IDs.

    **Suggested layout**

    The Sidebar can be used inside a two-column page layout:

    ```html
    <div class="container py-5">
      <div class="row g-5">
        <aside class="col-12 col-lg-3">
          <!-- Sidebar -->
        </aside>

        <main class="col-12 col-lg-9">
          <section id="introduction">...</section>
          <hr class="my-5">
          <section id="features">...</section>
          <hr class="my-5">
          <section id="pricing">...</section>
        </main>
      </div>
    </div>
    ```

    !!! note
        The layout is only a suggested implementation. The Sidebar can be used with different page structures.

    ### Accessibility

    - Use a semantic `<nav>` element.
    - Provide an accessible label when needed (e.g. `aria-label="Documentation sections"`).
    - Navigation links must reference valid section IDs.

    ### Technical behavior

    The `initScrollSpy()` function uses `IntersectionObserver` to track which section is in view:

    1. Finds the Sidebar using the provided selector.
    2. Reads the `href` fragment from each anchor link.
    3. Resolves the corresponding section by `id`.
    4. Observes all resolved sections.
    5. When a section enters the reading area (`rootMargin: '-20% 0px -70% 0px'`), removes `.active` from all sidebar links and applies `.active` to the matching link.

    **Dependencies:** Native `IntersectionObserver`. Existing site CSS/Sass. No additional JavaScript libraries.
