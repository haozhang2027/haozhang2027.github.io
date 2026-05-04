# Personal Static Blog Design

## Goal

Build a personal static website at `https://haozhang2027.github.io/` for Hao's Blogs. The site should be content-first, similar in spirit to Lilian Weng's blog: clean navigation, Markdown posts, archives, tags, search, projects, and an about page.

## Confirmed Decisions

- GitHub username: `haozhang2027`
- Repository name: `haozhang2027.github.io`
- Site URL: `https://haozhang2027.github.io/`
- Site title: `Hao's Blogs`
- Contact email: `haozhang2027@gmail.com`
- Language: mixed Chinese and English
- Navigation language: English
- Homepage description: `Thoughts on software, AI, learning, and life.`
- Technology: Hugo + PaperMod + GitHub Pages + GitHub Actions

## Architecture

The source repository stores Hugo configuration, Markdown content, theme wiring, and deployment automation. Hugo builds the source into static HTML/CSS/JS. GitHub Actions publishes the generated static site to GitHub Pages.

PaperMod provides the blog theme, post list, archive, tags, search page, RSS, and light/dark mode. The site should avoid custom complexity in the first version and rely on PaperMod defaults where possible.

## Site Structure

- `/`: homepage with profile text, social links, and recent posts
- `/posts/`: all blog posts
- `/archives/`: chronological archive
- `/tags/`: tag index
- `/projects/`: project list page
- `/about/`: about page
- `/search/`: search page

## Content Model

Posts are Markdown files under `content/posts/`. Each post uses front matter with title, date, tags, and draft status. Projects are represented on a single `content/projects.md` page for the first version.

## Deployment

The repository should include `.github/workflows/hugo.yml`. On push to `main`, GitHub Actions installs Hugo, builds the site, and deploys the generated files to GitHub Pages. GitHub repository settings should use Pages source `GitHub Actions`.

## Verification

Local verification should include:

- `hugo version` to confirm Hugo is installed
- `hugo --minify` to confirm the site builds
- `hugo server -D` for local preview when available

If Hugo or network access is unavailable locally, the implementation should still leave source files and GitHub Actions ready for use after dependencies are installed.
