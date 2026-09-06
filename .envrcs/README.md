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
| `.envrc.user.uv` | User env variant: uv sync + venv activation; guards on `pyproject.toml` existing, not on whether you use uv |
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

Note what "entirely" covers: `.env.local` is loaded *by* `.envrc.user`, not
by the root `.envrc`, so emptying the user layer silently takes the third
layer with it. The three layers are peers in what they hold, not in how they
are wired — see the tree at the end of this file. To keep `.env.local` while
dropping the toolchain, comment out the `source_env` line instead of emptying
the file.

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
reject anything else: any other type would have to be `eval`d as shell, and a
YAML or JSON payload run that way is a syntax error at best and arbitrary
command execution at worst.

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
stripped.

`#` fails the same way and is more likely still, since it is in every
generated-password character set:

```
plaintext:          DB_PASSWORD=P4ss#word
sops decrypts to:   DB_PASSWORD=P4ss#word
environment gets:   DB_PASSWORD=P4ss
```

Here double quotes *do* survive — which is exactly why you should not learn
the double-quote rule. **Single-quote every value.** It is the only form that
holds for both characters, and the only one you do not have to think about.

Nothing warns about this. The symptom is a service failing to authenticate
with a password that is right in the bundle and wrong in the environment, so
it surfaces a long way from its cause. Generated passwords are the usual way
to meet it: `$` is in most "special character" sets.

### Do not name a secret after a shell local

A bundle key that collides with a `local` declared anywhere up the call stack
is silently dropped. bash's `local` is dynamically scoped, so the `export`
that `direnv dotenv` emits writes that caller's slot, which is destroyed when
the frame returns. The value never reaches the environment, `use_sops`
returns 0, and direnv's success line lists the keys that *did* make it.

Six of the names belong to direnv itself and cannot be changed from here:

| frame | names |
|---|---|
| `use()` | `cmd` |
| `source_env()` | `rcpath`, `REPLY`, `rcpath_dir`, `rcpath_base`, `rcfile` |

`REPLY` and `cmd` are plausible names for a real secret. `.envrc.sops`'s own
locals are `_sops_`-prefixed to keep them out of the way, which is why that
prefix is not cosmetic.

`SCREAMING_SNAKE_CASE` keys avoid five of the six — but not `REPLY`. There is
no warning for this today; the fix that would remove the hazard entirely is
to emit `declare -gx` rather than `export`, which assigns at global scope.

### One concatenated bundle

Each `use_sops` spawns a sops process, and that spawn is almost the whole
cost. Measured with sops 3.13.3, median of 7 runs, keys and gpg-agent warm:

| | one bundle | 8 bundles | per extra bundle |
|---|---|---|---|
| age | 23 ms | 171 ms | 20 ms |
| PGP | 36 ms | 272 ms | 33 ms |

Decrypting one bundle costs the same whether it holds 1 key or 8 — the cost
is per *invocation*, not per secret. A bare sops process with no crypto at
all is 22 ms, so age decryption is about 1 ms of real work and PGP about 14
ms. The gpg-agent round trip is therefore a minor term, not the driver; its
expensive part is starting the agent (~28 ms), and that is once per session
rather than once per bundle.

**The break-even is three to four sources.** Below that, concatenating saves
20–35 ms per reload, which nobody perceives, and costs you a build step and
everything under [Not done yet](#not-done-yet). At eight sources it saves
150–240 ms on every reload — and a reload fires on every `cd` into the tree,
so that is the difference between instant and laggy. Concatenate when you
have many sources; keep separate bundles when you have two.

`.envrc.secrets.template` carries the commented `use sops_if_exists` line for
the concatenated bundle and points here; the recipe below is the only copy.

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
whatever recipients that file names.

A creation rule with no `path_regex` matches *every* input path, `/dev/stdin`
included — which is exactly the shape a personal `~/.sops.yaml` tends to
have, and why this recipe appears to work without a recipient flag until
someone runs it on a machine without that file. Naming the recipient is what
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

## Adopting this layout in another repo

Everything above describes this repo. Adopting the layout elsewhere is a
different job, and these are its edges.

**Prerequisites.** `direnv` always. `sops`, plus whatever key backend your
bundles use, only if you keep the secrets layer — and note that a missing
`sops` fails *quietly*: the fragment logs `sops: command not found` and
`sops --decrypt failed for ...`, the load continues, and nothing else
complains. `uv` or `nix` only if you enable that toolchain fragment.

**What to copy.** The root `.envrc`, `.gitignore.template`, and all of
`.envrcs/` except `demo-age-key.txt` and `.env.secrets.demo.sops-encrypted`
— those two exist so *this* repo demonstrates itself. Leaving them out is
safe: the demo line in `.envrc.secrets.template` watches a bundle that isn't
there and no-ops.

**Fix your `.gitignore` before the first `direnv allow`, not after.** The
auto-create skips an existing `.gitignore`, so `.envrcs/.envrc.secrets` and
`.envrcs/.envrc.user` are stageable from the moment they are created. Three
things to do, in this order:

1. Add the block from [Untracked](#untracked-local--secret) **at the end of
   your file**. "The negations come last" means last overall, not last within
   the block — a later `.env*` of your own will re-ignore
   `.envrcs/.env.*.template` and your committed bundle.
2. Remove any existing rule that ignores `.envrc`. It is a common one, and
   this layout needs the root `.envrc` tracked.
3. Check, rather than assume:
   ```sh
   git check-ignore -v .envrc .envrcs/.env.local.template \
       .envrcs/.env.secrets.demo.sops-encrypted .envrcs/.envrc.secrets
   ```
   The first three should print nothing. The last should be ignored.

**If you already have an `.envrc`**, its contents move into the root `.envrc`
*after* the `export direnv_root` line and *before* the `source_env` calls, or
into a fragment of your own sourced alongside them. `$direnv_root` and the
layout-dir default have to be set before anything uses them.

**The tracked templates are placeholders, not content.** `.env.local.template`
ships `EXAMPLE_*` names; `.envrc.secrets.template` ships the demo bundle line
with its `SOPS_AGE_KEY_FILE=` prefix. Rewrite both for your project before
your team clones — the generated copies are never overwritten afterwards,
so a template fix does not reach a checkout that already has one. Whoever
needs it has to delete their generated file and reload.

## Not done yet

Recorded here rather than in a tracker so it survives a clone.

**A generator for the concatenated bundle** — worth building only past the
break-even above. The recipe is written out but nothing runs it, so the
concatenated shape is a thing you assemble by hand every time. Whatever builds it also needs the thing the recipe does not have:
*a staleness story*. When one plaintext source changes, nothing tells you the
bundle is out of date. The bundle itself is watched, so direnv reloads when
the bundle changes — but not when its inputs do, and the inputs are ignored
files that may not even be on the same machine.

**An example plaintext source** for that generator. It needs a name the ignore
rules tolerate: everything matching `.env`, `*.env` or `.envrcs/.env.*` is
ignored, and only `*.sops-encrypted` is re-included, so an example input has
to live outside those patterns or be explicitly negated.

**`declare -gx` in place of `export`**, to close the collision hazard in
[Do not name a secret after a shell local](#do-not-name-a-secret-after-a-shell-local)
rather than document it. `declare -gx NAME=VAL` assigns at global scope and
cannot be captured by an enclosing frame; `declare -g NAME` followed by a
plain `export NAME=VAL` does **not** work, so it has to be the single command.
What makes it more than a one-line change is that it means rewriting
`direnv dotenv`'s output, and that output is not uniformly shaped — a single
dump can carry `export $'cmd'=$'hello'`, `export QUOTED=$'a b'` and
`export EMPTY=''`. A rewrite has to handle every name spelling without
touching values.

Deliberately not done, so they are not mistaken for oversights:

- The nix `watch_file` in `.envrc.nix-config` stays commented out until this
  repo has nix files worth watching.
- **No `.sops.yaml`.** Creation rules would not shorten the recipe anyway:
  sops matches them against the *input* path, and the recipe's input is
  `/dev/stdin`, so a rule keyed on the bundle's name never fires. Naming the
  recipient on the command line is one flag, and it makes the result depend
  on the command rather than on where it was run.

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
