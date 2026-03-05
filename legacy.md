# Legacy & Reference

These directories contain historical work that informed the production site. They can be archived.

## mood-board/

**Repo**: https://github.com/marina-alta-college/mood-board

Interactive mood board app used during the design phase to choose the visual identity. Presented 6 design directions:

1. Neo-Brutalism
2. Coastal Modern
3. Editorial Luxe
4. Warm Minimal
5. Playful Geometric
6. Refined Slate

The founders chose elements that evolved into the **Maritime Clean** theme used on the production site.

**Tech**: Go + HTMX + Tailwind 4 + DaisyUI 5 (same stack as production).

**Status**: Complete. Can be archived. Not deployed anywhere.

## poc-site/

**Repo**: https://github.com/marina-alta-college/poc-site

Early proof-of-concept that demonstrated Sanity CMS integration with the Refined Slate design. Used a different Sanity project (`l0nqyg78`) and supported dark mode.

**Key differences from production**:
- Different Sanity project ID (`l0nqyg78` vs `e8pjc321`)
- Refined Slate theme (not Maritime Clean)
- Dark mode toggle (removed in production)
- JavaScript for theme toggle, mobile nav, etc. (production uses CSS-only patterns)
- Simpler page set (no journal, calendar, who-we-are)

**Status**: Superseded by the `site` repo. Can be archived.

## wp-content/

Not a git repo. Contains HTML exports of page content from a WordPress site that was used for early copywriting. These were used as reference when populating Sanity content.

Files:
- `front-page.html` — Homepage copy
- `what-we-offer.html` — What We Offer copy
- `who-we-are.html` — Who We Are copy
- `admissions-and-fees.html` — Admissions copy
- `posts/post-1.html` — First blog post

**Status**: Reference only. Content has been migrated to Sanity.

## notes/

Planning documents:
- `actors.md` — Target audience notes (parents, students, the "flip")
- `color-scheme.html` — Colour palette exploration

**Status**: Reference only.
