---
name: czechitas-slidev
description: >
  Creating and converting presentations for Czechitas Digital Academy: Cybersecurity
  courses using the Slidev template. Use this skill whenever the user wants to create
  a new Czechitas presentation, convert a PPTX to Slidev format, add slides or sections,
  update course content, or sync a course repo with the template. Also trigger when the
  user mentions Czechitas, Slidev, course slides, Digital Academy, or GitHub Pages
  deployment of a presentation. Always use this skill — even for small edits — to
  ensure brand and structure consistency.
---

# Czechitas Slidev Skill

## Template Repository
`https://github.com/srameko/czechitas-cybersecurity-slidev-template`

Always work directly with the repo — clone or fetch from there. The CLAUDE.md in the
repo root is the authoritative reference; read it if anything is unclear.

---

## Key Rules

- **NEVER modify `theme/`** — shared Czechitas brand, always copy verbatim from template
- **`public/ondrej.png`** must always be present — never rename or delete
- Slide content is always in **English**
- Replace `REPO_NAME` in `package.json` and `deploy-slides.yml` before first deploy
- Repo must be **public** and GitHub Pages enabled (Settings → Pages → GitHub Actions)

---

## Brand Reference

### Colors
```css
--czechitas-pink:    #E6007E   /* primary, headings on gradient */
--czechitas-blue:    #2D2E83   /* headings on white background  */
--czechitas-cyan:    #00BFE7
--czechitas-yellow:  #FFCB04
--czechitas-orange:  #F36F21
--czechitas-green:   #8CC63E
--czechitas-purple:  #91268F
--czechitas-gradient: linear-gradient(135deg, #E6007E 0%, #6B1FA0 50%, #00BFE7 100%)
```

### Fonts
- Sans: **Open Sans** (300, 400, 700, 800)
- Mono: **Source Code Pro**

---

## Layouts

| Layout | Use for |
|---|---|
| `cover` | Title slide — gradient bg, woman-with-laptop illustration |
| `bio` | Lecturer intro — image, name, subtitle, bullets, optional QR |
| `section` | Chapter divider — gradient bg, large white heading |
| `default` | ~90% of content — white bg, blue heading |
| `image-right` | Slide with image on right (3:2 text:image ratio) |
| `center` | Closing slides (Feedback, Thank You) |

---

## HTML Utility Classes

```html
<!-- Icon grid (agenda, topic overview) — 3 cols default, cols-2 for 2 -->
<div class="icon-grid">
  <div class="icon-card">
    <div class="icon">🔍</div>
    <div class="label">Topic</div>
  </div>
</div>

<!-- Callout boxes -->
<div class="callout">Info (cyan)</div>
<div class="callout warning">Warning (orange)</div>

<!-- Chat/AI conversation pattern — use with v-click -->
<div class="chat-prompt">User message</div>
<div class="chat-response">AI response</div>
```

---

## Standard Slide Structure

```
slides.md              ← frontmatter + slide order via src: + lecturer bio
slides/                ← NN-name.md (00-agenda.md, 01-topic.md, …)
public/                ← images (ondrej.png always present)
theme/                 ← DO NOT MODIFY
.github/workflows/     ← deploy-slides.yml
```

Every file in `slides/` must start with its own `---` frontmatter defining the layout.

### slides.md skeleton
```md
---
theme: ./theme
title: Course Name
author: Lecturer Name
mdc: true
shiki:
  theme: github-light
favicon: /favicon.png
fonts:
  sans: Open Sans
  mono: Source Code Pro
---
---
layout: cover
subtitle: Digital Academy — Cybersecurity
author: Lecturer Name
date: Month YYYY
---
# Course Title
---
layout: bio
image: /ondrej.png
name: "Ondřej Šrámek"
subtitle: "GMON, GNFA, GCTI"
---
- bullet
::qr::
<QRCode url="https://linktr.ee/ondrejsramek" :size="120">Linktree</QRCode>
---
src: ./slides/00-agenda.md
---
[... src: includes for each section ...]
---
layout: center
---
# Feedback
<QRCode url="https://FEEDBACK_FORM_URL" :size="200">Feedback form</QRCode>
---
layout: center
---
# Thank You for Your Attention!
Ondřej Šrámek

**Czechitas · DA Cybersecurity · Month YYYY**
```

---

## Workflow: Create New Presentation

1. Clone template repo or create new repo from it
2. Replace `REPO_NAME` in:
   - `package.json` — `name` field and `build` script base path
   - `.github/workflows/deploy-slides.yml` — PDF export step
3. Update `slides.md` — title, author, date, bio details
4. Create section files in `slides/` — follow `NN-name.md` naming
5. Add images to `public/`
6. Enable GitHub Pages: Settings → Pages → Source → GitHub Actions
7. Push to `main` — CI builds, exports PDF, deploys to Pages

---

## Workflow: Convert PPTX to Slidev

The source PPTX will be in the repository. Steps:

1. Read the PPTX structure — extract slides, titles, content, speaker notes
2. Map PPTX slide types to Slidev layouts:
   - Title slide → `cover`
   - Section divider / chapter title → `section`
   - Content with image → `image-right`
   - Text-only content → `default`
   - Final slide → `center`
3. Convert bullet points, tables, and text verbatim into Markdown
4. Replace any visual elements (icons, diagrams) with appropriate HTML utility
   classes (`icon-grid`, `callout`) or describe what image is needed in `public/`
5. Produce `slides.md` + individual files in `slides/`
6. List any images that need to be added to `public/` manually

---

## GitHub Actions — Deploy Pipeline

Pipeline: push to `main` → build Slidev → export PDF → deploy to GitHub Pages

Actions used (note: pin to SHA when hardening):
- `actions/checkout@v4`
- `actions/setup-node@v4`
- `actions/upload-pages-artifact@v3`
- `actions/deploy-pages@v4`

PDF accessible after deploy at:
`https://srameko.github.io/<repo-name>/<repo-name>.pdf`

---

## Common Issues

- **Image not showing** — paths start with `/image.png`, no `public` prefix
- **bio without QR** — omit `::qr::` slot, layout handles it gracefully
- **REPO_NAME not replaced** — breaks build and PDF export
- **Deploy 404** — repo is private or Pages not enabled
- **PDF export failing** — do not add `playwright install-deps`; `playwright-chromium` in devDependencies is sufficient
- **Wrong layout** — chapter dividers use `section`, not `subtitle`
