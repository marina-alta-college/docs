# Design System — Maritime Clean

The site uses a custom DaisyUI theme called **Maritime Clean**. It's a light-only theme (no dark mode) with a cool blue-white palette.

## Colour Palette

| Name | Hex | CSS Variable | Usage |
|---|---|---|---|
| Cool Cloud | `#F5F8FA` | `--color-base-100` | Primary background |
| Cool White | `#EDF2F7` | `--color-base-200` | Alternating sections |
| Steel Mist | `#9FB4CC` | `--color-base-300` | Borders, dividers |
| Ink Navy | `#1A2E3F` | `--color-base-content` | Body text |
| Primary Navy | `#3B5171` | `--color-primary` | Primary buttons, links |
| Octopus Blue | `#6BA4D6` | `--color-secondary` | Secondary elements |
| Mediterranean Aqua | `#7BE8D4` | `--color-accent` | Accent highlights, dividers |
| Deep Navy | `#263748` | `--color-neutral` | Footer, CTA backgrounds |
| Deep Teal | `#4DC4B0` | `--color-success` | Success states |

## Typography

| Role | Font | Weight | CSS Variable |
|---|---|---|---|
| Headings | Cormorant Garamond | 600 | `--font-heading` |
| Body | DM Sans | 400 | `--font-body` |
| Logo | Montserrat | 700 | `--font-logo` |

Fonts are loaded from Google Fonts in `base.html`.

## CSS Architecture

The theme is defined in `static/css/input.css`:

1. **Tailwind import** + DaisyUI plugin
2. **Theme definition** — all DaisyUI theme variables
3. **Custom utility classes** — CTA gradients, prose styling, navbar glass effect
4. **Component styles** — calendar grid, initial avatars, scroll animations
5. **Accessibility** — skip link, reduced motion support

Build with:
```bash
npx @tailwindcss/cli -i static/css/input.css -o static/css/output.css --minify
```

## Key CSS Classes

| Class | Purpose |
|---|---|
| `.cta-gradient` | Navy gradient for CTA sections |
| `.text-cta-accent` | Mediterranean Aqua text on dark backgrounds |
| `.navbar-glass` | Frosted glass navbar effect |
| `.prose` | CMS rich text styling |
| `.prose-journal` | Extended prose for journal articles |
| `.reveal` | Scroll-reveal animation (CSS-only, no JS) |
| `.section-divider` | Aqua accent bar between sections |
| `.accent-border-left` | Pull-quote left border |
| `.initial-avatar` | Circular letter avatars for mentors |
| `.cal-*` | Calendar grid components |
| `.skip-link` | Accessibility skip-to-content link |

## Animations

All animations are CSS-only (no JavaScript):

- **Scroll reveal** — `animation-timeline: view()` with `@supports` fallback
- **Card hover** — `scale` transition on images
- **Contact pulse** — WhatsApp button attention pulse
- **Scroll progress** — Reading progress bar on journal posts
- **Reduced motion** — All animations disabled when `prefers-reduced-motion: reduce`

## Responsive Breakpoints

Uses Tailwind's default breakpoints:

| Breakpoint | Width | Prefix |
|---|---|---|
| Mobile | < 640px | (default) |
| Small | 640px | `sm:` |
| Medium | 768px | `md:` |
| Large | 1024px | `lg:` |
| Extra large | 1280px | `xl:` |

All pages are mobile-first — base styles are for mobile, larger breakpoints add complexity.

## Accessibility

- Semantic HTML throughout (`nav`, `main`, `article`, `section`, `aside`)
- Skip-to-content link (`.skip-link`)
- ARIA labels on interactive elements
- Colour contrast meets WCAG AA
- Keyboard navigable
- Reduced motion respected
