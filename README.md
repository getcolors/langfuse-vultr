# langfuse-vultr

Desired state for a self-hosted [Langfuse](https://langfuse.com) v4 on six
Vultr machines in Amsterdam, using the
[`langfuse`](https://github.com/getcolors/langfuse) Package Skill: a Neon
storage tier for Postgres, a Redis host, three ClickHouse replicas with
Keeper, and the application host behind Caddy and Cloudflare at
`langfuse.bigconfig.online`. Cloudflare R2 holds Neon's layers and WAL,
Langfuse's raw events and media, and the backups.

This repository holds `colors.yml`, the installed launcher, `.envrc`, and
`devenv.nix`. Everything else is generated (`.colors/`) or secret
(`.envrc.private`).

## Use

```sh
direnv allow                 # once, after the toolchain is installed
./green build                # render .colors/langfuse-vultr/ — no credentials needed
./green create --dry-run     # walk the workflow, skip every side effect
./green create               # converge for real
./green describe             # every host's last monitor result, over SSH
./green rehearse             # restore both stores from backup, boot, drill
```

## Credentials

All in the gitignored `.envrc.private`; the header of `colors.yml` lists them
with the scope each one needs. The state pair reaches no host. The storage
pair reaches the Neon and app hosts, the backup pair the Neon host and
ClickHouse node 0. `ENCRYPTION_KEY`, `SALT` and the initial user's password are
operator-held: keep copies outside these machines, because a Postgres backup
is readable only with the first two.

### One storage bucket for two tenants

Neon's layers and WAL and Langfuse's events and media share `langfuse-storage`
and its token, by decision. A compromised app host can therefore delete Neon
layers; the Postgres dumps in `langfuse-backup`, which that host cannot reach,
bound that loss. The hardening path is a third bucket and token: point
`langfuse-s3-bucket` at it, supply the pair as
`COLORS_PAR_LANGFUSE_STORAGE_R2_*`, a Neon-only pair as `COLORS_PAR_NEON_R2_*`,
and re-converge.

## After a create

```sh
ssh langfuse-vultr sudo langfuse-credential   # the project API keys
ssh langfuse-vultr sudo langfuse-status       # markers and containers
ssh langfuse-vultr-clickhouse-0 clickhouse-client --user admin --password "$(ssh langfuse-vultr-clickhouse-0 cat /etc/clickhouse-secrets/admin_password)"
```

Log in at `https://langfuse.bigconfig.online` as `claude@ululi.it` with
`COLORS_PAR_LANGFUSE_INIT_USER_PASSWORD`.

To render media in the UI, add a CORS rule to `langfuse-storage` for
`https://langfuse.bigconfig.online` (R2 dashboard or `wrangler r2 bucket cors
put`); the SDK path needs none.

## Recovery

| Failure | Recovers from | RPO |
|---|---|---|
| a ClickHouse replica | the other two | 0 |
| a Redis restart | the AOF | 0 |
| the Redis host | nothing automated; queued jobs are lost | the queue |
| the Neon host | the six-hourly Postgres dump | 6 h |
| the ClickHouse cluster | the nightly native backup, paired with the next dump | 24 h |
| the app host | `./green create` with the same operator-held keys | 0 |

`./green rehearse` proves the path and writes `.colors-recovery-verified`
beside `.colors-ready` under `langfuse-vultr/` in the storage bucket. The
pairing rule and its consistency semantics are in the package README.

## Deleting

```sh
COLORS_PAR_COMPUTE_PREVENT_DESTROY=false ./green delete
```

Removes the six machines, the firewall groups, the VPC, the DNS record, the
SSH config block and the machine keypair. Removes nothing in R2.
