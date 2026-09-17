# Posts

Longer notes and articles live here. Each post is an ordinary Markdown file.

The optional front matter supports only `title` and a one-line `tags` list:

```markdown
---
title: Why dlclose Is Not Process Restart
tags: [cpp, linux, qemu]
---

Post body...
```

If `title` is omitted, the first level-one heading is used. Creation and update
dates come from the file's Git history; they should not normally be added to
front matter.

When importing an article that was published before it entered this repository,
`published_at` may override the displayed creation date:

```markdown
published_at: 2026-05-05
```

This field must use `YYYY-MM-DD`. It records the article's original publication
date; it does not rewrite or replace Git history. Update dates always come from
Git.
