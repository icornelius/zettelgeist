# AGENTS.md

Working notes for coding agents in this repository. Humans are welcome to read
it; nothing here is agent-specific except the tone.

Read [ROADMAP.md](ROADMAP.md) before proposing work. It says what is broken, in
what order it is being fixed, and why — most "obvious" cleanups in this tree are
already scheduled into a phase, and doing them out of order creates conflicts.

## What this project is

ZettelGeist is a plaintext note-taking system. Notes ("zettels") are YAML or
Markdown-with-frontmatter files; `zimport` indexes them into a SQLite
full-text-search database, and `zfind` queries that index through a small query
language (ZQL) defined as a Tatsu grammar in `zettelgeist/zdb.py`.

Four console scripts are declared in `setup.py`:

| Command | Module | Purpose |
|---------|--------|---------|
| `zettel` | `zettelgeist/zettel.py` | Create and edit a zettel |
| `zimport` | `zettelgeist/zimport.py` | Index a directory of zettels |
| `zfind` | `zettelgeist/zfind.py` | Query the index |
| `zcreate` | `zettelgeist/zcreate.py` | Deprecated; prints a notice and is retired in Phase 4 |

## Layout

```
zettelgeist/      the package — all real source lives here
tests/            one test module (currently broken, see below)
docs/             NOT documentation — this is the GitHub Pages root
deprecated/       retired Jekyll site; also holds the mlb example zettels
sphinx-docs/      superseded by the wiki
project/          a Scala build, vestigial
bin/              four shell shims that duplicate the console_scripts
scripts/          deploy.sh, superseded by the release workflow
adhoc/ jupyter/   scratch material
```

Everything outside `zettelgeist/`, `tests/` and `docs/` is slated for deletion —
`deprecated/`, `sphinx-docs/`, `project/`, `adhoc/`, `jupyter/` and `src/` in
Phase 1, `bin/` and `scripts/` in Phase 2 once the entry points and the release
workflow are settled. Don't invest in any of it, and don't delete it ad hoc
either — those phases have an order to them.

## Setup

```sh
python -m venv venv && . venv/bin/activate
pip install -e .
```

`venv/` is already in `.gitignore`; `.venv/` is not.
Requires Python ≥ 3.10; developed against 3.12. Runtime dependencies are
`python-frontmatter` and `tatsu` — plus `PyYAML`, which every module imports but
which nothing declares (it arrives transitively through `frontmatter`). Phase 2
fixes that; until then, be aware the installed PyYAML version is unpinned.

## Tests

```sh
pip install pytest
python -m pytest
```

**The suite does not currently pass**, and this is known, not something to fix
opportunistically: `tests/test_zettelgeist.py` calls `yaml.load(doc)` with a
single argument, which PyYAML 6 removed. Repairing it is Phase 2 work, along with
adding pytest as a declared dev dependency and adding a CI job that runs it. If
you are working before Phase 2 lands, verify changes by exercising the commands
directly rather than by trusting a green run you did not get.

There is no CI test job. The single workflow, `.github/workflows/release.yml`,
builds and publishes to PyPI on any tag push.

## Working conventions

- **Branch off `master`.** Use a short topic branch per unit of work
  (`roadmap`, `issue-38`, …). `dev` is the branch that tracks what is about to
  land; don't treat it as a catch-all.
- **Pull requests go to `ZettelGeist/zettelgeist`**, not to a fork. So do issues:
  `gh issue list --repo ZettelGeist/zettelgeist`. A bare `gh` command may default
  to a fork that has issues disabled.
- **Never push a tag.** Tagging publishes to PyPI — the release workflow fires on
  `refs/tags/*`. Releases follow `docs/RELEASING.md` and are the maintainers'
  call.
- **One phase at a time.** Phases in ROADMAP.md are ordered by dependency, and
  each is meant to end at a state that could be tagged.

## Style

The existing code predates most of the Python it runs on: `%`-formatting
throughout, no type annotations, `os.path` rather than `pathlib`, bare `except:`
clauses, and diagnostics printed straight to stdout. Modernizing it is Phase 5,
deliberately last and deliberately incremental.

Until then: write new code in the modern idiom — f-strings, `pathlib`,
annotations on anything new — but **do not reformat or modernize code you are not
otherwise changing.** A drive-by reformat of a file makes the real diff
unreviewable, and Phase 1 lands a repo-wide formatting pass that will conflict
with it.

## Traps

- `docs/` is the GitHub Pages source (`master:/docs`, served at
  zettelgeist.org/zettelgeist). It contains a redirect to the wiki and a
  `.nojekyll`, not documentation. Adding a file there publishes it. Repository
  documentation goes at the root.
- The 29 baseball zettels under `deprecated/docs/example/mlb` are the corpus
  every issue-38 reproduction is stated against, and Phase 3 promotes them to a
  test fixture. Whatever else happens to `deprecated/`, those survive.
- `zettelgeist/zdb.py` holds the ZQL grammar as a plain (non-raw) string
  containing `\s`. It emits a `SyntaxWarning` on 3.12. Editing that string is
  delicate — whitespace inside it is significant to the parse, and a compiled-SQL
  regression test is the only thing that catches a mistake.
- `zutils.md5_for_file()` raises `NameError`; `hashlib` is never imported. It is
  a known latent bug, not a symptom of something you just broke.
- `bin/` duplicates the console scripts. If a command behaves oddly, check which
  one is on `PATH`.
