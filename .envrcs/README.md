# .envrcs/

direnv configuration split into composable fragments, sourced from the root `.envrc`.

## Tracked (templates & helpers)

| File | Purpose |
|---|---|
| `.envrc.sops` | Defines `use_sops` and `use_sops_if_exists` for decrypting secrets |
| `.envrc.nix-config` | Bootstraps nix-direnv; where to `watch_file` imported nix modules |
| `.envrc.secrets.template` | Decrypts `.env.secrets.demo.sops-encrypted`; shows how to add further bundles |
| `.envrc.user.template` | Default user env: sources `.envrc.user.uv`, loads `.env.local` |
| `.envrc.user.flake` | User env variant: nix flake (immutable install); no-ops without a `flake.nix`, and watches for one appearing |
| `.envrc.user.uv` | User env variant: uv sync + venv activation; no-ops without a `pyproject.toml`, and watches for one appearing |
| `.env.local.template` | Third layer: per-user, non-secret dotenv values |
| `.env.secrets.demo.sops-encrypted` | Encrypted demo bundle the secrets layer decrypts |

The root also tracks `.gitignore.template`, which generates the repo's
`.gitignore` (see below).

## Untracked (local / secret)

Never commit these; `.gitignore` covers them.

| File | Created from | Auto-created? |
|---|---|---|
| `.envrc.secrets` | `.envrc.secrets.template` | yes, mode `600` |
| `.envrc.user` | `.envrc.user.template` | yes, mode `600` |
| `.env.local` | `.env.local.template` | **no** — copy manually |
| `.env`, `*.env` | plaintext secret sources — never committed | no |

`.env.secrets.demo.sops-encrypted` is a dotenv-format, sops-encrypted demo
bundle. It is committed because it is encrypted — the plaintext it was built
from is not, and is covered by the ignore rules. It is a single-bundle
*demo*; the intended production shape is one concatenated bundle, described
below.

## Auto-create, and how to disable a fragment

The root `.envrc` creates `.gitignore`, `.envrc.secrets` and `.envrc.user`
from their templates if they are missing, so `direnv allow` is the only step
on a fresh clone. Existing files are never overwritten — a repo adopting this
layout keeps its own `.gitignore`, and a symlink to a template counts as
existing.

Because of the auto-create, **deleting one of these files does not disable
it** — it is recreated from the template on the next `direnv allow`/reload.
To durably disable a fragment, empty it instead:

```sh
> .envrcs/.envrc.user     # skip the user layer entirely
```

An existing-but-empty file satisfies the auto-create check and is a no-op
when sourced.

**If you symlinked the local file to its template** instead of copying it,
do not empty it — you would truncate the tracked template. Replace the
symlink with a real copy first (`cp --remove-destination
.envrcs/.envrc.user.template .envrcs/.envrc.user`), or comment out the lines
you want off. Editing through a symlink shows up as a dirty tracked file in
`git status`, which is the tell.

## The three layers

1. **`.envrc.secrets`** — sops-encrypted material. Shell fragment; sources
   `.envrc.sops` and calls `use_sops_if_exists` per bundle.
2. **`.envrc.user`** — per-user shell logic: which toolchain to bring up
   (`uv` or `flake`), anything needing conditionals or command substitution.
3. **`.env.local`** — per-user, non-secret values. Loaded with
   `dotenv_if_exists`, so it is parsed as `KEY=value` by `direnv dotenv`, not
   sourced as shell.

`.env.local` is deliberately *not* auto-created: its template presents
mutually exclusive strategies, and silently applying both would leave the
environment in a state nobody asked for. Copy it and uncomment exactly one.

`.gitignore.template` ignores `.envrcs/.env.*` wholesale, covering any future
per-user dotenv fragment, with a `!.envrcs/.env.*.template` negation so the
tracked templates stay visible in a fresh clone. It also ignores what the
user layer builds — `.direnv`, `.venv` (from `uv sync`) and `result*` (from a
nix build) — none of which belong in a commit.

## sops

`use_sops` watches the encrypted file *before* decrypting, so a file that
fails to decrypt is still watched and fixing it (or the key) retriggers the
reload. A failed `sops --decrypt` is reported via `log_error` and returns
non-zero rather than silently producing an environment with no secrets —
which is what happens if the decrypt runs inside a pipeline, where its exit
status is discarded.

`use_sops_if_exists` watches the path *before* testing whether it exists,
mirroring direnv's own `source_env_if_exists` and `dotenv_if_exists`. A
missing file is recorded as a watch with `exists:false`, so building the
bundle later retriggers the reload rather than leaving the secrets layer
silently empty until someone runs `direnv reload` by hand. The same applies
to the `flake.nix`/`pyproject.toml` guards in the user fragments.

`use_sops_if_exists` shares `use_sops`' default path,
`$direnv_root/.envrcs/.env.secrets.concatenated.sops-encrypted` — the
production bundle below — so a bare `use sops_if_exists` decrypts it instead
of silently doing nothing. This repo ships no such bundle, so the bare form
no-ops here and the demo names its file explicitly.

### One concatenated bundle

Each `use_sops` costs a sops process and a key-agent round trip, so N bundles
cost N of them and make `direnv reload` noticeably slower. The intended
production shape is therefore a single bundle built from many plaintext
sources, which `.envrc.secrets.template` carries as a commented example.

Encrypted sops files cannot be concatenated — each carries its own metadata
and MAC, and `cat a.sops b.sops | sops --decrypt` fails with `Error while
unflattening "lastmodified": Duplicate value`. The merge has to happen on the
plaintext, encrypted once. Piping keeps the merged plaintext off disk:

```sh
cat a.env b.env |
  sops --encrypt --input-type dotenv --output-type dotenv /dev/stdin \
  > .envrcs/.env.secrets.concatenated.sops-encrypted
```

sops has no `--stdin` flag, and bare `sops --encrypt` fails with `no file
specified` despite its help text, so `/dev/stdin` must be named explicitly.
`--input-type`/`--output-type` are required because there is no filename
extension to sniff. With no `.sops.yaml` in this repo there are also no
creation rules, so the recipient must be given as `--pgp <fingerprint>`.

Plaintext sources are ignored (`.env`, `*.env`, `.envrcs/.env.*`); the
encrypted bundle is not (`!*.sops-encrypted`). Committing the bundle is what
lets a fresh clone reach a working environment without first obtaining the
plaintext by some other channel.

Do not add `set -x` while debugging this fragment: bash traces `eval` *after*
expansion, so the decrypted `export` lines land in stderr on every reload.

## nix

`.envrc.nix-config` pins nix-direnv and is sourced by `.envrc.user.flake`.
nix-direnv watches `flake.nix`/`flake.lock` but not the modules they import,
so edits to those do not invalidate the cached devShell. The fix is an
explicit `watch_file`, which the fragment carries commented out — this repo
ships no nix files, so the line is documentation until it does. Its paths are
relative to `.envrcs/`, e.g. `watch_file ../nix/{commands,packages}.nix`.

For the same reason, `.envrc.user.flake` calls `use flake "$direnv_root"`
and not a bare `use flake`. The fragment runs with PWD at `.envrcs/`, and
nix-direnv's `use_flake` defaults its flake expression to `.` — so the bare
form evaluates `.envrcs`, records `MISSING .envrcs/flake.nix` in
`DIRENV_WATCHES`, and never notices a change to the real flake.

## Path conventions

- **`$direnv_root`** — exported by the root `.envrc`; points to the repo root. Use it for paths that must survive worktree copies (e.g. `$direnv_root/.venv`), and for anything a helper would otherwise resolve against PWD (e.g. `use flake "$direnv_root"`).
- Fragment-relative names (`source_env .envrc.sops`, `dotenv_if_exists .env.local`) resolve against `.envrcs/` itself, so bare names work between fragments.

Prefer these over bare relative paths like `../.venv`, which break when the
evaluation directory isn't what you expect.

## How it fits together

```
.envrc (repo root)
├── export direnv_root
├── auto-create .gitignore, .envrcs/.envrc.{secrets,user} if missing
├── source_env_if_exists .envrcs/.envrc.secrets
│   └── .envrc.sops → use_sops on encrypted .env files
└── source_env_if_exists .envrcs/.envrc.user
    ├── one of: .envrc.user.{uv,flake}
    └── dotenv_if_exists .env.local → per-user non-secret values, if configured
```

A fresh clone only needs:

```sh
direnv allow
```

To also set per-user non-secret values (optional, not auto-created):

```sh
cp .envrcs/.env.local.template .envrcs/.env.local
# edit .envrcs/.env.local, uncomment ONE strategy
```
