# Blog tools

The index generator uses Python 3.11 or newer and only the standard library.
From the repository root, run:

```sh
uv run --project tools python tools/build.py
```

Running `python tools/build.py` directly also works when a suitable Python is
already available.

`README.md` and numbered `toc/page*.md` files are generated. Do not edit them
directly. The generator never deletes other files under `toc/`.

For committed content, creation is the oldest Git author date returned by
`git log --follow`, and update is the newest. For a new or modified working-tree
file, the build date in the timezone configured by `config.toml` is used as its
pending creation or update date. Generated pages display calendar dates only.
