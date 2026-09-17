---
title: "GitHub Profile as a Blog"
tags: [git, github, blogging, python]
---

# GitHub Profile as a Blog

I wanted a place to publish technical notes, but I did not want another system
to maintain.

Most blogging setups begin with a reasonable choice—Hugo, Jekyll, Astro,
Next.js, or another static-site generator—and gradually acquire themes,
deployment pipelines, domains, analytics, comments, and configuration. These
tools are useful, but they also create a second project alongside the writing
itself.

My actual requirement was much smaller:

> Write Markdown, commit it, and move on.

That led me to a deliberately minimal design: use a GitHub Profile repository
itself as both my homepage and my technical blog.

There is no GitHub Pages site, web framework, JavaScript bundle, CSS theme,
database, or server. GitHub already provides most of the infrastructure I need:

- a Git repository for storage;
- Markdown rendering;
- file and repository search;
- revision history;
- links to individual files;
- Issues and Pull Requests for feedback;
- and a profile page that can serve as the entry point.

The repository is the blog.

## The Profile README as a Homepage

GitHub has a special convention: if a public repository has the same name as
its owner, its root `README.md` is displayed on that user's profile.

For me, that repository is:

```text
github.com/sudoytang/sudoytang
```

Instead of filling the profile README with language badges, contribution
graphs, dynamic statistics, and decorative images, I use it as a simple content
index:

```text
Yushun Tang

Short introduction

Featured
Recent Posts
Recent TILs
Projects
Contact
```

The homepage is still a normal Markdown file rendered by GitHub. It links
directly to articles stored in the same repository.

## Posts and TILs

The repository has two kinds of content:

```text
posts/
til/
```

`posts/` contains relatively complete articles. They do not need to be long,
but they should explain a subject rather than merely record a fact.

`til/` contains short Today I Learned notes. A TIL might be a command I tend to
forget, an explanation that finally clicked, or the conclusion of a debugging
session. It may be only a few paragraphs—or even a few lines.

This distinction is more psychological than technical.

If every useful observation has to become a polished long-form article,
publishing develops a high activation cost. A TIL gives small pieces of
knowledge somewhere legitimate to live.

## Articles Are Both Source and Output

I briefly considered separating authoring files from rendered files:

```text
content/posts/example.md   # source
posts/example.md           # generated output
```

The generated version could include creation dates, update dates, revision
counts, and navigation.

For this repository, that would create more problems than it solves.

GitHub already renders Markdown, so generating another copy of the same article
would lead to duplicate search results, duplicated content, confusing history,
and rewritten relative links. It would also make it unclear which file was
canonical.

Instead, the boundary is:

```text
posts/*.md       hand-written and directly readable
til/*.md         hand-written and directly readable
README.md        generated
toc/page*.md     generated
```

Only aggregate views are generated. Article bodies are not.

This is an important property of the design:

> Content should outlive the generator.

If I delete every tool in the repository several years from now, the articles
will still be ordinary Markdown files that can be read directly or migrated
into another system.

## Minimal Metadata

An article may contain a small amount of front matter:

```markdown
---
title: "Copy Fail on ARM64: Understanding and Porting CVE-2026-31431"
tags: [linux, security, arm64]
---
```

Only a title and a one-line tag list are normally needed. The first level-one
Markdown heading can also provide the title.

Dates are not manually maintained for newly written articles. Git is already
the source of truth:

```text
created_at = oldest commit that contains the file
updated_at = newest commit that modifies the file
```

The generator obtains these values from `git log --follow`, so a rename does not
immediately destroy the article's apparent history.

There is one explicit exception for imported articles. If an article was
originally published elsewhere before entering this repository, it may declare:

```markdown
published_at: 2026-05-05
```

This does not rewrite or pretend to change Git history. It distinguishes two
different facts:

- when the file entered this repository;
- when the article was originally published.

The displayed creation date and chronological sorting use `published_at` when
it exists. The update date remains derived from Git.

## Git History Is the Revision System

Traditional blogs often display two pieces of version information:

```text
Published
Last updated
```

Here, each article is a file in a Git repository. Reimplementing revision
history would be redundant.

GitHub already exposes:

- every commit that modified the file;
- the commit message;
- the author and timestamp;
- the exact diff;
- and the earlier contents of the file.

A maintained technical article can therefore have a useful history:

```text
Initial version
Correct explanation of page ownership
Add ARM64 example
Clarify MMIO behavior
```

This is more informative than displaying an artificial value such as
`Revision 7`. A typo fix, rename, or generated metadata update could all
increment such a number without representing a meaningful editorial revision.

Git commits already describe what changed.

## A Small, Boring Generator

The repository contains one primary build tool:

```text
tools/build.py
```

It is intentionally an ordinary Python script using the standard library. Its
responsibilities are limited:

1. Read `config.toml`.
2. Scan `posts/*.md` and `til/*.md`.
3. Parse the supported metadata.
4. Obtain creation and update dates from Git.
5. Sort entries by publication date.
6. Generate the profile README.
7. Generate numbered Markdown pages when the post list grows.
8. Remove obsolete generated pagination files.

It is not a reusable blogging framework and is not intended to become one.

There are no template engines or plugin architectures. The code only needs to
be readable, deterministic, and easy to modify.

Running the generator twice with unchanged inputs must produce no diff.

## Why uv Is Still Useful

The generator has no third-party Python dependencies, but its execution
environment should still be explicit.

The tools directory uses uv:

```bash
uv run --project tools python tools/build.py
```

This records the supported Python version and keeps the environment
reproducible without turning the script into an unnecessarily elaborate Python
package.

If tests or a genuinely useful dependency are added later, there is already a
clear place to manage them.

## Resolving the Build-Before-Commit Problem

Using Git for dates introduces a small ordering problem.

A local workflow might look like this:

```bash
vim posts/new-post.md
uv run --project tools python tools/build.py
git add .
git commit
git push
```

When the generator runs, the new article does not have a commit yet. Its real
Git creation date cannot be queried because it does not exist.

The script has a working-tree fallback, but GitHub Actions provides a cleaner
normal workflow.

After an article commit is pushed, the Action:

1. checks out the complete Git history;
2. installs uv;
3. runs the generator;
4. checks whether `README.md` or `toc/` changed;
5. commits the generated files if necessary.

The article commit already exists when the build runs, so its real Git date is
available.

A publication normally produces two commits:

```text
Add post about embedded QEMU
Update generated blog index
```

The first commit represents content. The second represents derived navigation.

The workflow only listens for changes under `posts/`, `til/`, the
configuration, and the generator itself. Its own generated commit changes only
`README.md` and `toc/`, so it does not create an infinite build loop.

My normal publishing workflow is therefore reduced to:

```bash
vim posts/new-post.md
git add posts/new-post.md
git commit -m "Add post about something"
git push
```

The index updates itself.

## Leaving Context for Future Agents

The repository is structurally simple, but its omissions are intentional.

An agent encountering it without context might reasonably propose:

- introducing Jekyll or another static-site generator;
- generating separate rendered copies of every article;
- adding a frontend framework;
- adding badges, analytics, or dynamic statistics;
- or refactoring a small script into a general framework.

To preserve the reasoning behind the design, the repository includes an
`AGENTS.md` file.

It explains:

- what the repository is;
- which files are canonical;
- which files are generated;
- how dates are defined;
- why article bodies are not generated;
- how to build and verify the index;
- and which forms of complexity are deliberately excluded.

`AGENTS.md` does not affect the GitHub Profile display, but it gives future
coding agents enough context to work with the existing design rather than
accidentally replacing it.

## Migrating an Existing Article

My first article already lived in a project repository. Copying it into the
profile repository without a plan would have created two sources of truth.

The migration therefore follows a simple rule:

1. Move the canonical article body into `posts/`.
2. Preserve its original publication date with `published_at`.
3. Replace the old article with a short moved notice.
4. Update the project README to point to the new canonical location.
5. Leave the old Git history intact.

The article now belongs to the blog, while the project repository remains
focused on source code and project-specific documentation. Existing links do
not immediately become 404s.

## What This Design Gives Up

This is not a complete replacement for a conventional blog.

It does not currently provide:

- a custom visual design;
- clean standalone URLs;
- RSS or Atom feeds;
- a comment system;
- client-side search;
- detailed analytics;
- sophisticated SEO;
- or arbitrary page templates.

The reading experience is whatever GitHub provides.

For me, those limitations are acceptable. The main problem is not the absence
of a theme or an RSS feed. It is whether writing remains easy enough to
continue, and whether the content remains understandable and portable years
later.

Plain Markdown, Git, and a small index generator satisfy those requirements
surprisingly well.

## The Result

The final model is simple:

```text
GitHub Profile   = homepage
Git repository   = content store
Markdown         = article format
Git history      = dates and revisions
GitHub search    = search
GitHub Actions   = index maintenance
Issues and PRs   = feedback
```

There is no separate deployment target. Publishing is just committing a file.

If the collection eventually grows large enough to need RSS, a custom domain,
or more sophisticated navigation, the content can be migrated to Hugo, Zola,
Astro, or something else.

Until that need actually appears, I do not need to maintain a blogging
platform.

I only need to write.
