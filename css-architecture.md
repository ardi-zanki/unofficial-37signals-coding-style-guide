# CSS Architecture

> Native CSS patterns from 37signals - no preprocessors needed.

---

## No Preprocessors

Fizzy uses **native CSS only** - no Sass, PostCSS, or Tailwind. Modern CSS has everything needed:

- Native nesting
- CSS variables
- Cascade layers
- Container queries

## Cascade Layers

Use `@layer` for specificity management:

```css
@layer reset, base, components, utilities;

@layer reset {
  * { box-sizing: border-box; }
}

@layer base {
  body { font-family: system-ui; }
}

@layer components {
  .card { /* component styles */ }
}

@layer utilities {
  .hidden { display: none; }
}
```

**Why**: Layers give explicit control over cascade order without specificity wars.

## OKLCH Color Space

Use OKLCH for perceptually uniform colors:

```css
:root {
  --color-primary: oklch(55% 0.15 250);
  --color-primary-light: oklch(70% 0.12 250);
  --color-primary-dark: oklch(40% 0.18 250);
}
```

**Why**: Unlike HSL, lightness changes in OKLCH look consistent across hues.

## CSS Variables for Theming

```css
:root {
  --color-ink: oklch(20% 0 0);
  --color-paper: oklch(98% 0 0);
  --color-link: oklch(50% 0.15 250);
}

@media (prefers-color-scheme: dark) {
  :root {
    --color-ink: oklch(90% 0 0);
    --color-paper: oklch(15% 0 0);
  }
}
```

## Native Nesting

```css
.card {
  padding: 1rem;

  &__title {
    font-size: 1.25rem;
  }

  &:hover {
    background: var(--color-hover);
  }

  @media (width < 768px) {
    padding: 0.5rem;
  }
}
```

## Modern Features Used

### `color-mix()` for Opacity

```css
.overlay {
  background: color-mix(in oklch, var(--color-ink), transparent 50%);
}
```

### `@starting-style` for Entry Animations

```css
dialog[open] {
  opacity: 1;

  @starting-style {
    opacity: 0;
  }
}
```

### `:has()` for Parent Selection

```css
.card:has(.card__badge) {
  padding-right: 3rem;
}

.form:has(:invalid) .submit-btn {
  opacity: 0.5;
}
```

### Container Queries

```css
.card {
  container-type: inline-size;
}

.card__content {
  @container (width < 300px) {
    flex-direction: column;
  }
}
```

## File Organization

```
app/assets/stylesheets/
├── application.css      # Imports and layer definitions
├── reset.css            # Minimal reset
├── base.css             # Typography, colors
├── components/          # Component styles
│   ├── card.css
│   ├── button.css
│   └── dialog.css
└── utilities.css        # Helper classes
```

## Key Principles

1. **No build step complexity** - Native CSS means no compilation
2. **Variables over mixins** - CSS custom properties replace Sass variables
3. **Layers over specificity hacks** - Explicit cascade control
4. **Modern syntax** - Nesting, `:has()`, container queries are well-supported
5. **Perceptual color** - OKLCH for consistent, accessible color scales
