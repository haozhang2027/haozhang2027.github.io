# Personal Static Blog Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first version of Hao's Blogs as a Hugo + PaperMod static site ready for GitHub Pages.

**Architecture:** Hugo will compile Markdown content and PaperMod theme templates into static files. GitHub Actions will publish the built site to GitHub Pages from the `main` branch.

**Tech Stack:** Hugo, PaperMod, Markdown, GitHub Pages, GitHub Actions

---

## File Structure

- Create/modify `hugo.yml`: Hugo site configuration, PaperMod options, menus, outputs, profile mode, and social links.
- Create `.gitignore`: ignore Hugo build and local cache output.
- Create `.github/workflows/hugo.yml`: GitHub Pages deployment workflow.
- Create `content/about.md`: personal about page.
- Create `content/projects.md`: initial project listing page.
- Create `content/archives.md`: PaperMod archive page.
- Create `content/search.md`: PaperMod search page.
- Create `content/posts/hello-world.md`: initial example post.
- Add `themes/PaperMod`: PaperMod theme as a git submodule when network access is available.
- Keep `docs/superpowers/specs/2026-05-05-personal-static-blog-design.md`: design record.

### Task 1: Initialize Project and Theme

**Files:**
- Create: `.gitignore`
- Create: `hugo.yml`
- Create directory: `content/posts`
- Create/modify: `themes/PaperMod`

- [ ] **Step 1: Initialize git if needed**

Run: `git status --short`
Expected: either repository status output or "not a git repository".

If it is not a git repository, run: `git init -b main`
Expected: git creates a repository on branch `main`.

- [ ] **Step 2: Verify Hugo availability**

Run: `hugo version`
Expected: Hugo version output. If unavailable, install Hugo before local build verification.

- [ ] **Step 3: Add PaperMod theme**

Run: `git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod`
Expected: `themes/PaperMod` exists and `.gitmodules` is created.

- [ ] **Step 4: Create baseline config files**

Create `.gitignore` and `hugo.yml` with the site configuration for `Hao's Blogs`, `https://haozhang2027.github.io/`, PaperMod theme, English navigation, mixed-language content, search output, archive menu, tags menu, projects menu, about menu, and contact email.

### Task 2: Add First Content Pages

**Files:**
- Create: `content/about.md`
- Create: `content/projects.md`
- Create: `content/archives.md`
- Create: `content/search.md`
- Create: `content/posts/hello-world.md`

- [ ] **Step 1: Add static pages**

Create About, Projects, Archives, and Search pages with PaperMod-compatible front matter. About should introduce Hao's Blogs as a place for notes on software, AI, learning, and life. Projects should include one starter section for future software projects and one contact line with `haozhang2027@gmail.com`.

- [ ] **Step 2: Add first post**

Create `content/posts/hello-world.md` with title, date, tags, draft false, and a short bilingual starter post.

### Task 3: Add GitHub Pages Deployment

**Files:**
- Create: `.github/workflows/hugo.yml`

- [ ] **Step 1: Add workflow**

Create a GitHub Actions workflow that triggers on pushes to `main`, installs Hugo extended, builds with `hugo --minify --baseURL "https://haozhang2027.github.io/"`, uploads the Pages artifact, and deploys to GitHub Pages.

### Task 4: Verify Locally

**Files:**
- Read: all generated source files

- [ ] **Step 1: Build site**

Run: `hugo --minify`
Expected: exit code 0 and generated `public/` output.

- [ ] **Step 2: Optional preview**

Run: `hugo server -D --bind 127.0.0.1 --port 1313`
Expected: local preview available at `http://127.0.0.1:1313/`.

- [ ] **Step 3: Review git status**

Run: `git status --short`
Expected: only intended site files, theme submodule metadata, docs, and generated local build files if not ignored.
