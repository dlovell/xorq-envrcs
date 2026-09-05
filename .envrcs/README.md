# .envrcs/

direnv configuration split into composable fragments, sourced from the root `.envrc`.

## Tracked (templates & helpers)

| File | Purpose |
|---|---|
| `.envrc.sops` | Defines `use_sops` and `use_sops_if_exists` for decrypting secrets |
| `.envrc.nix-config` | Bootstraps nix-direnv; where to `watch_file` imported nix modules |
| `.envrc.secrets.template` | Shows which sops-encrypted files the secrets layer expects |
| `.envrc.user.template` | Default user env: sources `.envrc.user.uv`, loads `.env.local` |
| `.envrc.user.flake` | User env variant: nix flake (immutable install) |
| `.envrc.user.uv` | User env variant: uv sync + venv activation |
| `.env.local.template` | Third layer: per-user, non-secret dotenv values |

The root also tracks `.gitignore.template`, which generates the repo's
`.gitignore` (see below).

## Untracked (local / secret)

Never commit these; `.gitignore` covers them.

| File | Created from | Auto-created? |
|---|---|---|
| `.envrc.secrets` | `.envrc.secrets.template` | yes, mode `600` |
| `.envrc.user` | `.envrc.user.template` | yes, mode `600` |
| `.env.local` | `.env.local.template` | **no** — copy manually |
| `.env.secrets.*` | sops-encrypted secret bundles | no |

The repo root also carries demo sops inputs: `secrets.yaml` (dotenv-format,
encrypted — the default path `use_sops` looks for) and `plain-text.yaml`.

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
tracked templates stay visible in a fresh clone.

## sops

`use_sops` watches the encrypted file *before* decrypting, so a file that
fails to decrypt is still watched and fixing it (or the key) retriggers the
reload. A failed `sops --decrypt` is reported via `log_error` and returns
non-zero rather than silently producing an environment with no secrets —
which is what happens if the decrypt runs inside a pipeline, where its exit
status is discarded.

Do not add `set -x` while debugging this fragment: bash traces `eval` *after*
expansion, so the decrypted `export` lines land in stderr on every reload.

## nix

`.envrc.nix-config` pins nix-direnv and is sourced by `.envrc.user.flake`.
nix-direnv watches `flake.nix`/`flake.lock` but not the modules they import,
so edits to those do not invalidate the cached devShell. The fix is an
explicit `watch_file`, which the fragment carries commented out — this repo
ships no nix files, so the line is documentation until it does. Its paths are
relative to `.envrcs/`, e.g. `watch_file ../nix/{commands,packages}.nix`.

## Path conventions

- **`$direnv_root`** — exported by the root `.envrc`; points to the repo root. Use it for paths that must survive worktree copies (e.g. `$direnv_root/.venv`).
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
