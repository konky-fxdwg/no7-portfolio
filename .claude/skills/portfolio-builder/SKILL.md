---
name: portfolio-builder
description: >
  Build and deploy a polished AI tools portfolio site from screenshots and a design YAML,
  then publish it to GitHub Pages. Use this skill whenever the user wants to create a portfolio,
  deploy a portfolio site, build a showcase from screenshots, or says something like
  "ポートフォリオを作って" / "サイトをデプロイして" / "スクリーンショットからサイトを作りたい".
  Also trigger when the user references a design YAML + image directory together.
---

# portfolio-builder

Turn a folder of product screenshots and a design-spec YAML into a production-quality
portfolio site, then deploy it to GitHub Pages — all with subagents working in parallel.

## Inputs

| Variable | Description | Example |
|---|---|---|
| `IMAGE_DIR` | Path to folder containing product screenshots | `/path/to/images/` |
| `DESIGN_YAML` | Path to design-spec YAML (colors, typography, mood) | `quiet_minimal_editorial.yaml` |
| `REPO_NAME` | GitHub repository name for Pages deployment | `my-portfolio` |

Ask the user for any missing inputs before starting.

## Workflow

### Phase 1 — Research + Skill scaffold (parallel)

Spawn two subagents **in the same turn**:

**Research Agent** (`Explore` type):
- Read every image in `IMAGE_DIR` and describe each product in 1–2 sentences
- Read the existing site if one exists; identify 3 improvement points
- Recommend section structure: Nav / Hero / Works / About / Contact
- Return findings as bullet points; no code

**Design-spec Agent** (inline, not a subagent):
- Read `DESIGN_YAML` and extract: color palette, typography rules, spacing philosophy, card style, button style, animation constraints
- Summarise as a token-efficient reference block to pass to the Build Agent

### Phase 2 — Build

Once Phase 1 completes, spawn one **Build Agent** (`claude` type) with:

```
Build a single-file HTML/CSS portfolio site with these requirements:

DESIGN SPEC: <paste design-spec summary>
PRODUCTS: <paste research findings>

Structure:
  Nav      — logo left, [Works · About · Contact] right, sticky + blur backdrop
  Hero     — centered, eyebrow label, large heading, subtitle, pill CTA button
  Works    — 2-column card grid; each card has browser chrome mockup, title, desc, tech tags, CTA button
  About    — placeholder frame (dashed border) with label "About" — content TBD
  Contact  — placeholder frame with label "Contact" — content TBD
  Footer   — © year + name, single line

Quality bars:
  - Semantic HTML (header/main/section/article/footer, aria-labels)
  - CSS custom properties for all design tokens
  - Scroll-reveal animation via IntersectionObserver
  - Responsive: 2-col → 1-col at 780px
  - <link rel="icon"> if favicon.png exists in the project root
  - .nojekyll file alongside index.html
  - NO external dependencies — pure HTML/CSS/JS

Save output to: <output_dir>/index.html and <output_dir>/.nojekyll
```

### Phase 3 — Deploy

After Build Agent completes:

1. `git init` in the output directory
2. `git add .` and commit
3. `gh repo create <REPO_NAME> --public --source=. --remote=origin --push`
4. `gh api repos/<owner>/<REPO_NAME>/pages --method POST --field 'source[branch]=main' --field 'source[path]=/'`
5. Poll `gh api repos/<owner>/<REPO_NAME>/pages --jq '.status'` until `"built"`
6. Open in browser: `open -a "Google Chrome" "https://<owner>.github.io/<REPO_NAME>/"`

Return the live URL to the user.

## Output

```
✅ Deployed: https://<owner>.github.io/<REPO_NAME>/
```

## Notes

- About and Contact sections are intentionally left as placeholder frames. Ask the user for content before filling them in.
- If the user already has a GitHub repo, skip `gh repo create` and just push.
- If `gh auth status` fails, ask the user to run `! gh auth login` first.
