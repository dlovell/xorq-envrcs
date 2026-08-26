# .envrcs/

direnv configuration split into composable fragments, sourced from the root `.envrc`.

## Tracked (templates & helpers)

| File | Purpose |
|---|---|
| `.envrc.sops` | Defines `use_sops` and `use_sops_if_exists` for decrypting secrets |
| `.envrc.nix-config` | Bootstraps nix-direnv |
| `.envrc.secrets.template` | Shows which sops-encrypted files the secrets layer expects |
| `.envrc.user.template` | Default user env: sources `.envrc.user.uv` |
| `.envrc.user.flake` | User env variant: nix flake (immutable install) |
| `.envrc.user.uv` | User env variant: uv sync + venv activation |

## Untracked (local / secret)

These are created locally by copying (or symlinking) the templates. Never commit them.

| File | Created from |
|---|---|
| `.envrc.secrets` | `.envrc.secrets.template` |
| `.envrc.user` | Copy one of `.envrc.user.{template,flake}` |
| `.env.secrets.*` | sops-encrypted secret bundles |

The repo root also carries demo sops inputs: `secrets.yaml` (dotenv-format,
encrypted — the default path `use_sops` looks for) and `plain-text.yaml`.

## Path conventions

- **`$direnv_root`** — exported by the root `.envrc`; points to the repo root. Use it for paths that must survive worktree copies (e.g. `$direnv_root/.venv`).
- Fragment-relative names (`source_env .envrc.sops`) resolve against `.envrcs/` itself, so bare names work between fragments.

Prefer these over bare relative paths like `../.venv`, which break when the
evaluation directory isn't what you expect.

## How it fits together

```
.envrc (repo root)
├── export direnv_root
├── source_env_if_exists .envrcs/.envrc.secrets
│   └── .envrc.sops → use_sops on encrypted .env files
└── source_env_if_exists .envrcs/.envrc.user
    └── one of: .envrc.user.{template,flake}
```

To get started, copy the templates:

```sh
cp .envrcs/.envrc.secrets.template .envrcs/.envrc.secrets
cp .envrcs/.envrc.user.template .envrcs/.envrc.user
direnv allow
```
