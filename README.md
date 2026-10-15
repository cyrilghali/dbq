# dbq

Query a Postgres database from the terminal, by short source name, with no
client or MCP server to keep running. Every source resolves and connects
fresh on each call, so a VPN blip or a stale session never costs a restart.

```
dbq prod "select count(*) from users"
dbq staging '\d orders'
dbq --json local "select id, email from users limit 5"
echo "select now()" | dbq prod
dbq --history "users"          # search past queries
```

## Why

- **One binary, no daemon.** Nothing to keep alive, nothing that goes stale.
- **Credentials never touch argv.** Passwords come from the macOS keychain
  and are exported as `PG*` environment variables, never passed as CLI
  arguments.
- **Read-only by convention.** Any source not named `local-*` or `*-write`
  runs with `default_transaction_read_only=on` and a statement timeout.
- **A local audit trail.** Every query, successful or not, is appended as
  one JSON line to a local history file, searchable with `dbq --history`.
- **A concurrency cap.** Non-local sources are capped at a small number of
  simultaneous connections per source, so a burst of parallel callers can't
  pile onto one database.

## Install

```
git clone https://github.com/cyrilghali/dbq ~/.local/share/dbq
ln -s ~/.local/share/dbq/dbq /usr/local/bin/dbq   # or anywhere on your PATH
```

Requires `bash`, `python3`, `psql` (`brew install libpq`), and, on macOS,
`/usr/bin/lockf` (ships with the OS) for the concurrency cap. The keychain
integration is macOS-only (`security find-generic-password`).

## Configure

`dbq` reads four required values from a config file at
`${DBQ_CONFIG:-~/.config/dbq/config}` (see `example.conf`):

```
TEMPLATE="$HOME/.config/dbq/sources.toml"
PSQL="/opt/homebrew/opt/libpq/bin/psql"
TAILNET="example.ts.net"
KC_SERVICE="dbq"

# optional, shown at their defaults
# MAX_CONCURRENT=2
# QUEUE_WAIT_FACTOR=2
```

| Key | Meaning |
|---|---|
| `TEMPLATE` | Path to the TOML file listing your sources (below). |
| `PSQL` | Path to the `psql` binary. |
| `TAILNET` | A Tailscale tailnet name. If the current profile isn't on it, `dbq` switches to it before connecting to a non-local source. Only matters if you use Tailscale; point it at any name if you don't, and connections will just proceed. |
| `KC_SERVICE` | The macOS keychain service name holding your passwords. Also used as the prefix for the DSN placeholders below. |
| `MAX_CONCURRENT` | Concurrent-connection cap per non-local source (default 2; local sources are unlimited unless you set this). |
| `QUEUE_WAIT_FACTOR` | A query waiting for a free slot gives up after `QUEUE_WAIT_FACTOR × --timeout` seconds (600s if `--timeout 0`). |

### Source template

`TEMPLATE` is a small TOML file, one `[[sources]]` block per source:

```
[[sources]]
id = "local-main"
dsn = "postgresql://postgres:postgres@localhost:5432/main"

[[sources]]
id = "prod-app"
dsn = "postgresql://appuser:${DBQ_PASSWORD}@prod-db.internal:5432/main?sslmode=require"
```

A `${...}` placeholder in a `dsn` line is read from the macOS keychain: the
part after the `KC_SERVICE`-derived prefix (upper-cased, `KC_SERVICE` with
`-` turned into `_`) names the keychain item, lower-cased with `_` turned
back into `-`. With `KC_SERVICE="dbq"`, `${DBQ_PASSWORD}` reads the keychain
item named `password`:

```
security add-generic-password -s dbq -a password -w '<the password>'
```

A source named `local-*` skips the tailnet check, the read-only transaction,
the statement timeout and the concurrency cap: it's meant for a database on
your own machine.

A source named `*-write` is a remote database you mean to write to. It keeps
the tailnet check, the statement timeout and the concurrency cap, but skips
the read-only transaction. It answers only to its full name: `dbq staging-w`
never reaches `staging-write`, so a typo can't open a writable connection.

## Use

```
dbq                                   # list sources
dbq <source> "<sql>"                  # run a query
dbq <source> '\d <table>'             # psql backslash commands work
dbq --json <source> "<select ...>"    # aggregate one SELECT into a JSON array
dbq <source> --csv -f report.sql      # any psql flag passes through
echo "select now()" | dbq <source>    # SQL on stdin
dbq --timeout 300 <source> "<sql>"    # statement timeout in seconds
dbq --history [pattern]               # search past queries
dbq --history --source <name> --limit 5 --json
```

`<source>` can be any unique prefix (`prod-a` for `prod-app`). Run `dbq -h`
for the full flag reference.

## History

Every query appends one line to
`${XDG_STATE_HOME:-~/.local/state}/dbq/history.jsonl` (created `0700`/`0600`):
timestamp, source, the full SQL text, row count (when it could be
determined), duration, exit code, whether the output was truncated, and how
long the query waited for a concurrency slot. A history write failure never
fails the query itself.

## Test

```
./dbq-test
```

Needs a local Postgres reachable at `localhost:5432` as user `postgres`
with password `postgrespassword` and a database named `main`. Adjust the
throwaway source template at the top of `dbq-test` if your local setup
differs. The suite builds its own scratch config and history directory, so
it never touches your real `~/.config/dbq/config` or history file.
