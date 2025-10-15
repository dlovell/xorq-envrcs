# what

a repo to demonstrate user-specific `.envrc`/`.env` files for use with [`xorq`](https://github.com/xorq-labs/xorq)


# how

symlinking `.envrc.user` and `.envrc.secrets` into `xorq/.envrcs/` will result in this repo being used to configure a `xorq` development environment

NOTE: the user will want to provide their own ([sops](https://github.com/getsops/sops)-encrypted) secrets
