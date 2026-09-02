---
title: "Buttons"
summary: "An in-page navigation component for long-form pages."
---

!!! note
    This is a note.

!!! tip
    This is a tip.

!!! warning
    This is a warning.

!!! danger
    This is dangerous.

!!! info
    Additional information.

??? note "Show more"
    Additional content goes here.

```html title="Example" linenums="1"
<div class="example">
    ...
</div>


**Tabs** y **content blocks** tienen sintaxis propia de BxSites, pero aquí quiero ser cuidadoso: **no quiero darte otra sintaxis que no esté confirmada para la versión que tienes instalada**.

Y aquí está el punto importante:

### Yo haría una prueba MUY pequeña

Crea temporalmente `docs/test.md` con solamente:

```markdown
# BxSites Component Test

## Admonition

!!! note "Note"
    This is a test note.

## Warning

!!! warning "Warning"
    This is a test warning.

## Code

```html title="Example"
<div class="example">
  Helloooo
</div>


Luego:

```bash
boxlang module:bxSites build --projectRoot=.