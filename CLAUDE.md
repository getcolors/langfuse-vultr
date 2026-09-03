# CLAUDE.md

Guidance for agents working in this deployment. Read
`~/code/getcolors/CLAUDE.md` first.

## What this is

Desired state only. No source code. `colors.yml` is the single file to edit;
everything else is either generated (`.colors/`), secret (`.envrc.private`), or
an installed copy of the Package Skill launcher.

## Things specific to this deployment

- **Six machines, one VPC.** `langfuse-vultr-neon`, `-redis`, `-app` and
  `-clickhouse-{0,1,2}`, each with its own firewall group. `ssh
  langfuse-vultr` reaches the app host; every machine has its own alias.
- **`colors.yml` carries `neon-*` keys deliberately.** The package renders the
  `getcolors/neon` templates from a SHA pin rather than copying them, so it
  must speak that package's vocabulary.
- **`.envrc` maps the storage credential** onto `COLORS_PAR_NEON_R2_*`, which
  is what the imported storage-tier play reads via `lookup('env')`. Neon data
  and Langfuse events share the storage bucket here; a third bucket and token
  is the hardening path `colors.yml` describes.
- **Three secrets are operator-held**: `COLORS_PAR_LANGFUSE_ENCRYPTION_KEY`,
  `COLORS_PAR_LANGFUSE_SALT`, `COLORS_PAR_LANGFUSE_INIT_USER_PASSWORD`. A
  Postgres backup is readable only with the first two. They live in
  `.envrc.private` and must also live somewhere that is not this machine.
- **`vultr-http-sources: cloudflare`** is a symbolic source the package
  resolves at converge time and requires `cloudflare-proxied: true`.
- **Media renders in the UI only after the operator adds a CORS rule** to the
  storage bucket; the smoke gate prints a `WARN` line until then.

## Before any converge

```sh
./green build                # renders offline
./green create --dry-run     # walks the DAG, skips every side effect
```

## Afterwards

```sh
./green describe             # every host's last monitor result
./green rehearse             # restore-and-boot both stores, then the drills
ssh langfuse-vultr sudo langfuse-credential     # the project API keys
```

## Never

- Edit `.colors/` — it is generated.
- Edit `compute-prevent-destroy` in committed state.
- Export `COLORS_PAR_PROFILE`.
- Run `create`, `rehearse` or `delete` against this deployment without
  explicit authorization.
