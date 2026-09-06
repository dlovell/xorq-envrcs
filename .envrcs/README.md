# .envrcs/

direnv configuration split into composable fragments, sourced from the root `.envrc`.

## Tracked (templates & helpers)

| File | Purpose |
|---|---|
| `.envrc.sops` | Defines `use_sops` and `use_sops_if_exists` for decrypting secrets |
| `.envrc.nix-config` | Bootstraps nix-direnv; where to `watch_file` imported nix modules |
| `.envrc.secrets.template` | Decrypts `.env.secrets.demo.sops-encrypted`; shows how to add further bundles |
| `.envrc.user.template` | Default user env: loads `.env.local`; both toolchain fragments commented, pick one |
| `.envrc.user.flake` | User env variant: nix flake (immutable install); no-ops without a `flake.nix`, and watches for one appearing |
| `.envrc.user.uv` | User env variant: uv sync + venv activation; no-ops without a `pyproject.toml`, and watches for one appearing |
| `.env.local.template` | Third layer: per-user, non-secret dotenv values |
| `.env.secrets.demo.sops-encrypted` | Encrypted demo bundle the secrets layer decrypts |
| `demo-age-key.txt` | Throwaway private key for the demo bundle — committed on purpose, see below |

The root also tracks `.gitignore.template`, which generates the repo's
`.gitignore` (see below).

## Untracked (local / secret)

Never commit these. The generated `.gitignore` covers them — but only the
generated one: the root `.envrc` skips the auto-create when the repo already
has a `.gitignore`, which is the normal case for a repo adopting this layout.
**If you brought your own `.gitignore`, add these rules to it yourself:**

```gitignore
# plaintext secret sources — the ones that actually matter
.env
*.env
.envrcs/.env.*

# auto-created local fragments
.envrcs/.envrc.secrets
.envrcs/.envrc.user

# build products of the user layer
.direnv
.venv
/result
/result-*

# tracked exceptions, and they must come last
!.envrcs/.env.*.template
!*.sops-encrypted
```

This is the whole of `.gitignore.template` minus its self-ignore line; if the
two ever drift, that file is the source of truth.

Until you do, `direnv allow` creates `.envrcs/.envrc.secrets` as an ordinary
untracked file that `git add -A` will happily stage. By default it holds no
secret material — `source_env .envrc.sops`, and a `use sops_if_exists` call
prefixed with `SOPS_AGE_KEY_FILE=...` for the demo bundle — but it is a
shell fragment, so a stray `export SECRET=...` typed into it would be staged
too, and mode `600` protects it from other local users, not from git.

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

### The committed demo key

`demo-age-key.txt` is an age **private** key, committed deliberately. A demo
bundle is only a demo if every clone can decrypt it; encrypting it to one
person's key would mean a wall of sops errors on `direnv allow` for everyone
else, and "a fresh clone needs only `direnv allow`" would be false. The key
guards nothing: the whole plaintext is `secret=value`.

`.envrc.secrets.template` names it as a var-assignment prefix on the one call
that needs it, so `SOPS_AGE_KEY_FILE` is discarded when the function returns
and cannot become the default for a real bundle. A real bundle is encrypted
to the recipients who should read it, and their private keys never enter the
repo.

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
   The template ships with both toolchain lines commented out — which one a
   checkout wants is a per-user choice, and neither fragment is a sensible
   default to impose. Uncomment one.
3. **`.env.local`** — per-user, non-secret values. Loaded with
   `dotenv_if_exists`, so it is parsed as `KEY=value` by `direnv dotenv`, not
   sourced as shell.

`.env.local` is deliberately *not* auto-created: its template presents
mutually exclusive strategies, and silently applying both would leave the
environment in a state nobody asked for. Copy it and uncomment exactly one.

The rules that cover all of this, and the reason for each, are in
`.gitignore.template` itself; the block under
[Untracked](#untracked-local--secret) above is a copy of it for adopters.
Both are wholesale rather than per-file, so a future per-user dotenv
fragment is covered without an edit.

## sops

`use_sops` watches the encrypted file *before* decrypting, so a file that
fails to decrypt is still watched and fixing it (or the key) retriggers the
reload. A failed `sops --decrypt` is reported via `log_error` and returns
non-zero for callers that check it. direnv evaluates `.envrc` without
`errexit`, so the load continues either way — what the check buys is the
error message, instead of the silent empty environment a pipeline gives you
by discarding the decrypt's exit status.

The `direnv dotenv` parse is captured and checked for the same reason. The
two parsers do not agree: sops will happily encrypt `NOTE=he said "hi"`, and
`direnv dotenv` rejects the decrypted result as an invalid line. Piped
straight into `eval`, that is an empty environment reported as success.

Only `dotenv` bundles are supported. The `output_type` parameter exists to
reject anything else: the YAML/JSON path it replaces ran `eval` on the
decrypted payload, which is a syntax error at best and arbitrary command
execution at worst.

Both helpers need `$direnv_root` to build their default path. If it is unset
— this fragment reused somewhere that does not export it — they `log_error`
and return non-zero instead of defaulting to `/.envrcs/...` and quietly
doing nothing.

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

### Quote values containing `$`

`direnv dotenv` expands values the way a shell would, so an unquoted `$` and
the word after it are treated as a variable reference. The variable is
almost never set, so the reference expands to nothing and the value is
**silently truncated** — sops decrypted it correctly, `use_sops` returns 0,
and direnv reports success:

```
plaintext:          DB_PASSWORD=P4ss$word
sops decrypts to:   DB_PASSWORD=P4ss$word
environment gets:   DB_PASSWORD=P4ss
```

Single-quote the value in the plaintext, before encrypting. Verified through
a full sops round trip:

```dotenv
DB_PASSWORD='P4ss$word'
```

Double quotes do **not** work *for this one* — `"P4ss$word"` truncates exactly
like the bare form, because the expansion happens before the quotes are
stripped. Single-quoting is the habit to keep, since it is the only form that
holds for every character.

Nothing warns about this. The symptom is a service failing to authenticate
with a password that is right in the bundle and wrong in the environment, so
it surfaces a long way from its cause. Generated passwords are the usual way
to meet it: `$` is in most "special character" sets.

### One concatenated bundle

Each `use_sops` costs a sops process — and, for PGP recipients, a gpg-agent
round trip — so N bundles cost N of them and make `direnv reload` noticeably
slower. The intended
production shape is therefore a single bundle built from many plaintext
sources. `.envrc.secrets.template` carries the commented `use sops_if_exists`
line for it and points here; the recipe below is the only copy.

Encrypted sops files cannot be concatenated — each carries its own metadata
and MAC, so feeding two bundles to one `sops --decrypt` fails with a
`Duplicate value` unflattening error. The merge has to happen on the
plaintext, encrypted once. Piping keeps the merged plaintext off disk:

```sh
bundle=.envrcs/.env.secrets.concatenated.sops-encrypted
cat a.env b.env |
  sops --encrypt --pgp "$FPR" --input-type dotenv --output-type dotenv /dev/stdin \
  > "$bundle.tmp" &&
  mv "$bundle.tmp" "$bundle"
```

sops has no `--stdin` flag, and bare `sops --encrypt` fails with `no file
specified` despite its help text, so `/dev/stdin` must be named explicitly.
`--input-type`/`--output-type` matter because there is no filename extension
to sniff: without them sops succeeds but writes a JSON/binary bundle that
`use_sops` cannot parse.

The recipient must be named — `--pgp <fingerprint>`, or `--age <public key>`
as the demo bundle uses. There is no `.sops.yaml` here, but sops searches
upward from the **current directory**, not the repo root, so a `~/.sops.yaml`
will supply creation rules and the encrypt will quietly succeed against
whatever recipients that file names. Naming the recipient explicitly is what
makes the result depend on the command rather than on where you ran it.

Write through `.tmp` and `mv`: `>` creates the output file before sops runs,
so a failed encrypt would otherwise leave a truncated bundle that
`use_sops_if_exists` treats as existing and fails to decrypt on every reload.
The `.tmp` name stays ignored — `.envrcs/.env.*` matches it and
`!*.sops-encrypted` does not re-include it.

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

PWD is the recurring hazard here. `.envrc.user.flake` runs with PWD at
`.envrcs/`, and nix-direnv derives three separate things from it — the flake
expression, the layout dir, and the body of the generated
`nix-direnv-reload` helper. None of the three fails loudly: nix itself
searches upward and builds the right flake, so a bare `use flake` *works*
while watching the wrong lock file. The fragment passes `$direnv_root`
explicitly *and* runs from it; the comment on those lines says what each one
costs.

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
