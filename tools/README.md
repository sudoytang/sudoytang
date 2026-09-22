# Blog tools

The index generator uses Python 3.11 or newer and only the standard library.
From the repository root, run:

```sh
uv run --project tools python tools/build.py
```

Running `python tools/build.py` directly also works when a suitable Python is
already available.

`README.md`, numbered `toc/page*.md` files, and `toc/tag/` indexes are
generated. Do not edit them directly. Obsolete generated `toc/page*.md` and
`toc/tag/` files are removed; other files under `toc/` are left alone.

List tags without writing files:

```sh
uv run --project tools python tools/build.py tags
```

The GitHub Actions workflow normally runs the generator and commits changed
indexes after source content is pushed. Running it locally remains useful for
previewing a change.

For committed content, creation is the oldest Git author date returned by
`git log --follow`, and update is the newest. For a new or modified working-tree
file, the build date in the timezone configured by `config.toml` is used as its
pending creation or update date. Generated pages display calendar dates only.
