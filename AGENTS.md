# Repository guide for coding agents

## Purpose

This is both Yushun Tang's GitHub Profile repository and a small Git-native
technical blog. GitHub renders the root `README.md` on the profile page. The
repository deliberately avoids a website framework, GitHub Pages, JavaScript,
CSS, a database, and a general-purpose static-site generator.

The design priority is:

1. Content must remain ordinary, portable Markdown.
2. Git history is the source of truth for article dates and revisions.
3. Generated navigation should stay simple and deterministic.
4. Content should outlive the generator.

## Source and generated files

- `posts/*.md`: hand-written, canonical long-form articles.
- `til/*.md`: hand-written, canonical short notes.
- `config.toml`: hand-written profile, project, and index configuration.
- `tools/build.py`: the standard-library-only index generator.
- `README.md`: generated profile and blog home page; do not edit directly.
- `toc/page*.md`: generated post pagination; do not edit directly.
- `toc/tag/`: generated per-tag hubs and show-all pages; do not edit directly.

Do not introduce a second rendered copy of an article. Files under `posts/` and
`til/` are both source of truth and the pages readers open. Generate indexes and
navigation, not article bodies. Do not hand-edit `README.md` or anything under
`toc/`.

## Content format

Articles are normal Markdown. They may use this intentionally limited front
matter:

```markdown
---
title: "Copy Fail on ARM64: Understanding and Porting CVE-2026-31431"
tags: [linux, security, arm64]
published_at: 2026-05-05
---
```

Only `title` and a one-line `tags` list are supported. A first-level Markdown
heading may provide the title instead. Do not normally add manually maintained
creation or update dates to article metadata.

An imported older article may additionally declare:

```markdown
published_at: 2026-05-05
```

`published_at` is an explicit override for the displayed creation date and
sorting. Use it only when an article was genuinely published before entering
this repository. It does not alter Git history. Update dates remain Git-derived.

The Copy Fail article was imported from `sudoytang/copyfail-arm64` with its
original publication date. This repository now holds its canonical article
content; the project repository retains only a moved notice.

## Tags

Each tag must match `^[A-Za-z0-9_-]+$` (non-empty). Prefer short slug-like
tags (`cpp`, not `c++`). The generator does not rewrite, alias, or fold tags
silently. Distinct tag strings that `casefold` to the same value (for example
`cpp` and `Cpp`) are an error. The build fails and names the file and tag, or
both colliding strings.

`posts_per_page` in `config.toml` is also the tag hub preview size and the
show-all page size. Do not add a separate tag page-size key without a strong
reason.

Generated layout, only for tags that appear on at least one post or TIL:

- Hub: `toc/tag/<tag>.md` — title is the tag string as-is; `## Posts` and
  `## TILs` list up to `posts_per_page` newest entries (omit a section if
  empty). If a kind has more than that many entries, the hub links to
  `[Show all posts →](<tag>/posts/1.md)` or
  `[Show all TILs →](<tag>/tils/1.md)`. Hubs have no Older/Newer links.
- Show-all posts, only when that tag has more posts than `posts_per_page`:
  `toc/tag/<tag>/posts/1.md`, `2.md`, … (page size `posts_per_page`, newest
  first). Nav is Older/Newer between pages plus a link back to the tag hub.
- Show-all TILs: `toc/tag/<tag>/tils/<n>.md`, same rules.

List current tags (sorted, with counts) without writing files:

```sh
uv run --project tools python tools/build.py tags
```

## Dates

For a committed article:

- `created_at` is the oldest Git author date from `git log --follow`.
- `updated_at` is the newest Git author date from `git log --follow`.

For new or modified working-tree content, the generator uses the current date
in the timezone from `config.toml` as a pending fallback. Generated pages show
calendar dates only. Keep full Git history available when building.

## Build and verification

Python is managed with uv and must remain dependency-free unless a dependency
has a compelling benefit.

Run from the repository root:

```sh
uv run --project tools python tools/build.py
```

CI runs the same generator with no arguments
(`uv run --project tools --python 3.12 python tools/build.py`).

After a change, run the build twice and confirm the second run produces no
diff. Also run `git diff --check`. When testing pagination, ensure obsolete
generated `toc/page*.md` files and obsolete files under `toc/tag/` are removed
without deleting unrelated files.

## Automation

`.github/workflows/build-index.yml` runs after source or generator changes. It
checks out full history, runs the generator, and commits changed `README.md` and
`toc/` output (including `toc/tag/`) with the GitHub Actions bot.
Generated-only commits do not trigger another build.

Humans may still run the build locally for preview, but normally only source
content needs to be committed. Keep the workflow small; this is index
generation, not a deployment pipeline.

## Scope discipline

Prefer a direct, boring implementation over abstractions or frameworks. Do not
add badges, statistics cards, analytics, comments, decorative images, dynamic
widgets, or a frontend toolchain. Archive, tag, or recently-updated pages may be
added later when actual content volume justifies them.
