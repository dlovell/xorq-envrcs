# what

a repo to demonstrate `direnv` `.envrc` file organization

The root `.envrc` exports `$direnv_root`, auto-creates the untracked local
files from their tracked templates, and sources two fragments from
`.envrcs/`. A fresh clone needs only `direnv allow`.

## three layers

| layer | file | holds |
|---|---|---|
| secrets | `.envrcs/.envrc.secrets` | sops-encrypted material |
| user | `.envrcs/.envrc.user` | per-user shell logic: which toolchain to bring up |
| local | `.envrcs/.env.local` | per-user, non-secret `KEY=value` |

Each is untracked and generated from a tracked `*.template`. The first two
are created automatically on `direnv allow`; the third is copied by hand,
because its template offers mutually exclusive strategies to choose between.

Plaintext secret sources are never committed; sops-encrypted bundles are.

See [`.envrcs/README.md`](.envrcs/README.md) for the fragment-by-fragment
reference — auto-create rules and how to disable a fragment, the sops
helpers, path conventions, and the concatenated-bundle design.
